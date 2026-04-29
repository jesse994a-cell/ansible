# 第 12 章：備份還原驗證與 DR 演練

> **學習目標：** 使用 Ansible 建立完整的備份還原驗證流程——從定期備份的存在性確認，到在隔離環境實際執行 restore 並驗證資料完整性，最後記錄 RTO（Recovery Time Objective）與 RPO（Recovery Point Objective）實測值，作為 ISO 27001 Annex A 8.13 的稽核證據。

---

## 12.1 理論說明：備份驗證 vs. 備份檢查

### 12.1.1 兩者的本質差異

ISO 27001 稽核員最常問的問題：**「你的備份有測試過可以還原嗎？」**

```
備份「檢查」（第 8 章的做法）：
  ✅ 備份檔案存在
  ✅ 檔案大小 > 100MB
  ✅ 備份時間 < 25 小時前
  ❌ 無法回答：「資料還原後服務能正常運作嗎？」

備份「驗證」（本章的做法）：
  ✅ 備份檔案存在（繼承自第 8 章）
  ✅ 在隔離環境實際 restore
  ✅ 應用程式啟動並通過健康檢查
  ✅ 資料完整性校驗（行數、checksum）
  ✅ 記錄實際 RTO（從備份到服務恢復的時間）
```

### 12.1.2 RTO / RPO 的定義與實測

```
RPO（Recovery Point Objective）資料遺失容忍量
─────────────────────────────────────────────
「最多可以接受遺失多少時間的資料？」

政策定義：24 小時
實測方式：確認最後一份備份的時間戳記
          └→ 上次備份距今的時間差 = 實際 RPO

RTO（Recovery Time Objective）服務恢復時間
─────────────────────────────────────────────
「從災難發生到服務恢復，最多允許多久？」

政策定義：4 小時
實測方式：計時 restore playbook 的執行時間
          └→ 計時開始 → restore → 健康檢查通過 = 實際 RTO
```

### 12.1.3 本章架構：隔離環境驗證

```
┌─────────────────────────────────────────────────────────────────┐
│                    備份還原驗證架構                               │
│                                                                 │
│  生產環境                        DR 隔離環境（Docker）           │
│  ─────────                      ─────────────────────          │
│  GitLab DB 每日備份              gitlab-restore-test            │
│  → /backup/gitlab-YYYYMMDD.tar  ← 同一台 host 的獨立容器        │
│                                                                 │
│  Ansible backup_validator role                                  │
│  Step 1：確認備份檔案存在（RPO 計算）                            │
│  Step 2：建立隔離容器（PostgreSQL / Redis）                      │
│  Step 3：還原備份到隔離容器                                      │
│  Step 4：執行完整性驗證（資料行數、checksum）                    │
│  Step 5：計算並記錄 RTO                                         │
│  Step 6：清理隔離容器                                           │
│  Step 7：產出 DR 演練報告 → audit-evidence                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12.2 Ansible Role 結構：backup_validator

```bash
ansible-galaxy role init roles/backup_validator
```

```
roles/backup_validator/
├── defaults/
│   └── main.yml          ← 備份路徑、RTO/RPO 政策值
├── tasks/
│   ├── main.yml
│   ├── check_backups.yml   ← 備份檔案存在性確認（RPO）
│   ├── restore_gitlab.yml  ← GitLab 備份還原驗證
│   ├── restore_db.yml      ← PostgreSQL 備份還原驗證
│   ├── verify_integrity.yml ← 資料完整性驗證
│   ├── cleanup.yml         ← 清理隔離環境
│   └── report.yml          ← 產出 DR 演練報告
└── templates/
    └── dr_report.md.j2     ← DR 報告模板
```

### 12.2.1 defaults/main.yml

```yaml
# roles/backup_validator/defaults/main.yml
---
# ── 備份路徑設定 ──────────────────────────────
backup_locations:
  - name: "GitLab"
    path: /opt/gitlab/data/backups
    pattern: "*_gitlab_backup.tar"
    min_size_mb: 100
    max_age_hours: 26      # 每日備份，容忍 2 小時誤差
    restore_type: gitlab

  - name: "PostgreSQL"
    path: /backup/postgresql
    pattern: "*.sql.gz"
    min_size_mb: 10
    max_age_hours: 26
    restore_type: postgresql

  - name: "Application Config"
    path: /backup/configs
    pattern: "*.tar.gz"
    min_size_mb: 1
    max_age_hours: 168     # 每週備份（7*24=168h）
    restore_type: files

# ── RTO / RPO 政策值（來自 BCP 文件）────────────
# 這些是目標值，實測後與之比對
dr_rto_target_minutes: 240    # 政策：4 小時內恢復
dr_rpo_target_hours: 24       # 政策：最多遺失 24 小時資料

# ── 隔離環境設定 ──────────────────────────────
dr_test_network: dr_validation_net
dr_test_postgres_container: dr-postgres-test
dr_test_gitlab_container: dr-gitlab-test

# ── 報告設定 ──────────────────────────────────
dr_report_dir: /opt/dr-reports
dr_report_filename: "dr-validation-{{ inventory_hostname }}-{{ ansible_date_time.date }}.md"
```

### 12.2.2 tasks/main.yml

```yaml
# roles/backup_validator/tasks/main.yml
---
# ── 初始化計時器與結果容器 ───────────────────────────────────────────────
- name: Initialize DR validation tracking
  ansible.builtin.set_fact:
    dr_start_time: "{{ ansible_date_time.epoch }}"
    dr_results: []          # 各備份的驗證結果
    dr_alerts: []           # 失敗或警告項目

- name: Check backup file existence and RPO
  ansible.builtin.import_tasks: check_backups.yml
  tags: [dr, check]

- name: Restore and validate GitLab backup
  ansible.builtin.import_tasks: restore_gitlab.yml
  tags: [dr, restore, gitlab]

- name: Restore and validate PostgreSQL backup
  ansible.builtin.import_tasks: restore_db.yml
  tags: [dr, restore, db]

- name: Verify data integrity
  ansible.builtin.import_tasks: verify_integrity.yml
  tags: [dr, verify]

- name: Clean up test environment
  ansible.builtin.import_tasks: cleanup.yml
  tags: [dr, cleanup]
  # 即使驗證失敗也要清理（always）
  always: true

- name: Generate DR validation report
  ansible.builtin.import_tasks: report.yml
  tags: [dr, report]
```

### 12.2.3 tasks/check_backups.yml

```yaml
# roles/backup_validator/tasks/check_backups.yml
---
- name: Check each backup location
  block:
    - name: Find latest backup file
      ansible.builtin.find:
        paths: "{{ item.path }}"
        patterns: "{{ item.pattern }}"
        age: "-{{ item.max_age_hours }}h"    # 最近 N 小時內
        size: "{{ item.min_size_mb }}m"      # 最小大小
        file_type: file
      register: backup_find_result

    - name: Get backup file stats
      ansible.builtin.stat:
        path: "{{ (backup_find_result.files | sort(attribute='mtime') | last).path }}"
      register: latest_backup_stat
      when: backup_find_result.files | length > 0

    - name: Calculate RPO (hours since last backup)
      ansible.builtin.set_fact:
        actual_rpo_hours: >-
          {{
            ((ansible_date_time.epoch | int) - (latest_backup_stat.stat.mtime | int)) / 3600
            | round(1)
          }}
      when: backup_find_result.files | length > 0

    - name: Record backup check result
      ansible.builtin.set_fact:
        dr_results: >-
          {{
            dr_results + [{
              'name': item.name,
              'type': 'backup_check',
              'status': 'PASS' if backup_find_result.files | length > 0 else 'FAIL',
              'latest_file': (backup_find_result.files | sort(attribute='mtime') | last).path
                             if backup_find_result.files | length > 0 else 'NOT FOUND',
              'actual_rpo_hours': actual_rpo_hours | default('N/A'),
              'rpo_target_hours': dr_rpo_target_hours,
              'rpo_ok': (actual_rpo_hours | default(999) | float) <= dr_rpo_target_hours
                        if backup_find_result.files | length > 0 else false
            }]
          }}

    - name: Add alert for missing or old backup
      ansible.builtin.set_fact:
        dr_alerts: >-
          {{
            dr_alerts + [{
              'level': 'CRITICAL',
              'name': item.name,
              'message': '找不到 ' + (item.max_age_hours | string) + ' 小時內的備份'
            }]
          }}
      when: backup_find_result.files | length == 0

  loop: "{{ backup_locations }}"
  loop_control:
    label: "{{ item.name }}"
```

### 12.2.4 tasks/restore_db.yml

```yaml
# roles/backup_validator/tasks/restore_db.yml
---
# ── 建立隔離的 PostgreSQL 測試容器 ──────────────────────────────────────
- name: Create DR test network
  community.docker.docker_network:
    name: "{{ dr_test_network }}"
    state: present

- name: Find latest PostgreSQL backup
  ansible.builtin.find:
    paths: "{{ (backup_locations | selectattr('restore_type', 'eq', 'postgresql') | first).path }}"
    patterns: "*.sql.gz"
    file_type: file
  register: pg_backups

- name: Set latest PostgreSQL backup path
  ansible.builtin.set_fact:
    latest_pg_backup: "{{ (pg_backups.files | sort(attribute='mtime') | last).path }}"
  when: pg_backups.files | length > 0

- name: Start isolated PostgreSQL container for restore test
  community.docker.docker_container:
    name: "{{ dr_test_postgres_container }}"
    image: "postgres:16-alpine"
    state: started
    networks:
      - name: "{{ dr_test_network }}"
    env:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "testuser"]
      interval: 5s
      timeout: 3s
      retries: 10
  register: pg_test_container

- name: Wait for test PostgreSQL to be ready
  community.docker.docker_container_info:
    name: "{{ dr_test_postgres_container }}"
  register: pg_test_info
  until: pg_test_info.container.State.Health.Status == "healthy"
  retries: 12
  delay: 5

# ── 記錄 RTO 計時開始 ────────────────────────────────────────────────────
- name: Record restore start time
  ansible.builtin.set_fact:
    restore_start_epoch: "{{ ansible_date_time.epoch }}"

# ── 還原備份到隔離容器 ───────────────────────────────────────────────────
- name: Copy backup file into container
  ansible.builtin.command:
    cmd: >
      docker cp {{ latest_pg_backup }}
      {{ dr_test_postgres_container }}:/tmp/restore.sql.gz
  changed_when: true
  when: pg_backups.files | length > 0

- name: Restore PostgreSQL backup in isolated container
  community.docker.docker_container_exec:
    container: "{{ dr_test_postgres_container }}"
    command: >
      sh -c "gunzip -c /tmp/restore.sql.gz
             | psql -U testuser -d testdb 2>&1"
  register: restore_output
  when: pg_backups.files | length > 0

# ── 記錄 RTO ─────────────────────────────────────────────────────────────
- name: Calculate actual RTO for PostgreSQL restore
  ansible.builtin.set_fact:
    pg_rto_minutes: >-
      {{
        ((ansible_date_time.epoch | int) - (restore_start_epoch | int)) / 60
        | round(1)
      }}

- name: Record PostgreSQL restore result
  ansible.builtin.set_fact:
    dr_results: >-
      {{
        dr_results + [{
          'name': 'PostgreSQL Restore',
          'type': 'restore',
          'status': 'PASS' if not restore_output.failed | default(false) else 'FAIL',
          'rto_minutes': pg_rto_minutes,
          'rto_target_minutes': dr_rto_target_minutes,
          'rto_ok': (pg_rto_minutes | float) <= dr_rto_target_minutes,
          'backup_file': latest_pg_backup | default('N/A'),
          'restore_output': restore_output.stdout_lines[:5] | default([])
        }]
      }}
  when: pg_backups.files | length > 0
```

### 12.2.5 tasks/restore_gitlab.yml

```yaml
# roles/backup_validator/tasks/restore_gitlab.yml
---
# GitLab 的 restore 流程比 PostgreSQL 複雜，
# 使用 gitlab-backup restore 指令
- name: Find latest GitLab backup
  ansible.builtin.find:
    paths: "{{ (backup_locations | selectattr('restore_type', 'eq', 'gitlab') | first).path }}"
    patterns: "*_gitlab_backup.tar"
    file_type: file
  register: gitlab_backups

- name: Set latest GitLab backup
  ansible.builtin.set_fact:
    latest_gitlab_backup: "{{ (gitlab_backups.files | sort(attribute='mtime') | last).path }}"
    latest_gitlab_backup_name: "{{ (gitlab_backups.files | sort(attribute='mtime') | last).path | basename }}"
  when: gitlab_backups.files | length > 0

# ── 啟動 GitLab 隔離測試環境 ────────────────────────────────────────────
- name: Record restore start time for GitLab
  ansible.builtin.set_fact:
    gitlab_restore_start: "{{ ansible_date_time.epoch }}"

- name: Start isolated GitLab test environment
  community.docker.docker_container:
    name: "{{ dr_test_gitlab_container }}"
    image: "gitlab/gitlab-ce:17.0.1-ce.0"
    state: started
    networks:
      - name: "{{ dr_test_network }}"
    volumes:
      # 掛載備份目錄（read-only）
      - "{{ (backup_locations | selectattr('restore_type', 'eq', 'gitlab') | first).path }}:/var/opt/gitlab/backups:ro"
    env:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost'
        postgresql['enable'] = true
        redis['enable'] = true
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/-/health"]
      interval: 30s
      timeout: 10s
      retries: 20
      start_period: 180s

- name: Wait for test GitLab to initialize
  community.docker.docker_container_info:
    name: "{{ dr_test_gitlab_container }}"
  register: gitlab_test_info
  until: gitlab_test_info.container.State.Health.Status == "healthy"
  retries: 25
  delay: 30

# ── 執行 GitLab restore ──────────────────────────────────────────────────
- name: Stop GitLab services before restore
  community.docker.docker_container_exec:
    container: "{{ dr_test_gitlab_container }}"
    command: gitlab-ctl stop puma
  register: stop_puma

- name: Stop Sidekiq
  community.docker.docker_container_exec:
    container: "{{ dr_test_gitlab_container }}"
    command: gitlab-ctl stop sidekiq

- name: Run GitLab backup restore
  community.docker.docker_container_exec:
    container: "{{ dr_test_gitlab_container }}"
    # BACKUP= 參數：去掉 _gitlab_backup.tar 後綴的部分
    command: >
      gitlab-backup restore BACKUP={{
        latest_gitlab_backup_name
        | regex_replace('_gitlab_backup\\.tar$', '')
      }} force=yes
  register: gitlab_restore_output
  when: gitlab_backups.files | length > 0

- name: Restart GitLab after restore
  community.docker.docker_container_exec:
    container: "{{ dr_test_gitlab_container }}"
    command: gitlab-ctl restart

# ── 計算 RTO ─────────────────────────────────────────────────────────────
- name: Calculate GitLab restore RTO
  ansible.builtin.set_fact:
    gitlab_rto_minutes: >-
      {{
        ((ansible_date_time.epoch | int) - (gitlab_restore_start | int)) / 60
        | round(1)
      }}

- name: Record GitLab restore result
  ansible.builtin.set_fact:
    dr_results: >-
      {{
        dr_results + [{
          'name': 'GitLab CE Restore',
          'type': 'restore',
          'status': 'PASS' if not gitlab_restore_output.failed | default(false) else 'FAIL',
          'rto_minutes': gitlab_rto_minutes,
          'rto_target_minutes': dr_rto_target_minutes,
          'rto_ok': (gitlab_rto_minutes | float) <= dr_rto_target_minutes,
          'backup_file': latest_gitlab_backup | default('N/A')
        }]
      }}
```

### 12.2.6 tasks/verify_integrity.yml

```yaml
# roles/backup_validator/tasks/verify_integrity.yml
---
# ── PostgreSQL 資料完整性驗證 ────────────────────────────────────────────
- name: Count tables in restored PostgreSQL database
  community.docker.docker_container_exec:
    container: "{{ dr_test_postgres_container }}"
    command: >
      psql -U testuser -d testdb -t -c
      "SELECT COUNT(*) FROM information_schema.tables
       WHERE table_schema='public';"
  register: pg_table_count
  ignore_errors: true

- name: Count total rows across all tables
  community.docker.docker_container_exec:
    container: "{{ dr_test_postgres_container }}"
    command: >
      psql -U testuser -d testdb -t -c
      "SELECT SUM(n_live_tup) FROM pg_stat_user_tables;"
  register: pg_row_count
  ignore_errors: true

- name: Record PostgreSQL integrity result
  ansible.builtin.set_fact:
    dr_results: >-
      {{
        dr_results + [{
          'name': 'PostgreSQL Data Integrity',
          'type': 'integrity',
          'status': 'PASS' if (pg_table_count.stdout | default('0') | trim | int) > 0 else 'WARN',
          'tables': pg_table_count.stdout | default('0') | trim,
          'rows': pg_row_count.stdout | default('0') | trim,
          'note': '資料表數量 > 0 表示還原成功'
        }]
      }}

# ── GitLab 健康檢查驗證 ──────────────────────────────────────────────────
- name: Verify GitLab health after restore
  ansible.builtin.command:
    cmd: >
      docker exec {{ dr_test_gitlab_container }}
      curl -sf http://localhost/-/health
  register: gitlab_health
  changed_when: false
  ignore_errors: true

- name: Check GitLab readiness
  ansible.builtin.command:
    cmd: >
      docker exec {{ dr_test_gitlab_container }}
      curl -sf http://localhost/-/readiness?all=1
  register: gitlab_readiness
  changed_when: false
  ignore_errors: true

- name: Record GitLab integrity result
  ansible.builtin.set_fact:
    dr_results: >-
      {{
        dr_results + [{
          'name': 'GitLab Health Check',
          'type': 'integrity',
          'status': 'PASS' if gitlab_health.rc == 0 else 'FAIL',
          'health_response': gitlab_health.stdout | default('N/A'),
          'readiness': 'OK' if gitlab_readiness.rc == 0 else 'FAIL'
        }]
      }}
```

### 12.2.7 tasks/cleanup.yml

```yaml
# roles/backup_validator/tasks/cleanup.yml
---
# 無論成功失敗都清理隔離環境
- name: Remove DR test PostgreSQL container
  community.docker.docker_container:
    name: "{{ dr_test_postgres_container }}"
    state: absent
    force_kill: true
  ignore_errors: true

- name: Remove DR test GitLab container
  community.docker.docker_container:
    name: "{{ dr_test_gitlab_container }}"
    state: absent
    force_kill: true
  ignore_errors: true

- name: Remove DR test network
  community.docker.docker_network:
    name: "{{ dr_test_network }}"
    state: absent
  ignore_errors: true

- name: Log cleanup completion
  ansible.builtin.debug:
    msg: "✅ DR 隔離測試環境已清理完畢"
```

### 12.2.8 tasks/report.yml + templates/dr_report.md.j2

```yaml
# roles/backup_validator/tasks/report.yml
---
- name: Calculate overall DR validation status
  ansible.builtin.set_fact:
    dr_overall_status: >-
      {{
        'FAIL' if dr_results | selectattr('status', 'eq', 'FAIL') | list | length > 0
        else 'WARN' if dr_results | selectattr('status', 'eq', 'WARN') | list | length > 0
        else 'PASS'
      }}
    dr_total_elapsed_minutes: >-
      {{
        ((ansible_date_time.epoch | int) - (dr_start_time | int)) / 60
        | round(1)
      }}

- name: Ensure DR report directory exists
  ansible.builtin.file:
    path: "{{ dr_report_dir }}"
    state: directory
    mode: '0750'

- name: Generate DR validation report
  ansible.builtin.template:
    src: dr_report.md.j2
    dest: "{{ dr_report_dir }}/{{ dr_report_filename }}"
    mode: '0640'

- name: Fetch report to control node
  ansible.builtin.fetch:
    src: "{{ dr_report_dir }}/{{ dr_report_filename }}"
    dest: "./dr-reports/{{ ansible_date_time.date }}/{{ inventory_hostname }}.md"
    flat: true
```

```jinja2
{# roles/backup_validator/templates/dr_report.md.j2 #}
# DR 演練報告（Disaster Recovery Validation）

| 項目 | 內容 |
|------|------|
| **主機** | `{{ inventory_hostname }}` |
| **演練日期** | {{ ansible_date_time.date }} |
| **開始時間** | {{ ansible_date_time.iso8601 }} |
| **總耗時** | {{ dr_total_elapsed_minutes }} 分鐘 |
| **整體結果** | {{ '✅ PASS' if dr_overall_status == 'PASS' else '❌ FAIL' if dr_overall_status == 'FAIL' else '⚠️ WARN' }} |
| **執行者** | {{ ansible_user_id }}（Ansible 自動化）|

---

## RTO / RPO 實測對比

| 指標 | 政策目標 | 實測值 | 結果 |
|------|---------|--------|------|
{% for r in dr_results | selectattr('type', 'eq', 'restore') | list %}
| {{ r.name }} RTO | {{ r.rto_target_minutes }} 分鐘 | {{ r.rto_minutes }} 分鐘 | {{ '✅ 達標' if r.rto_ok else '❌ 超標' }} |
{% endfor %}
{% for r in dr_results | selectattr('type', 'eq', 'backup_check') | list %}
| {{ r.name }} RPO | {{ r.rpo_target_hours }} 小時 | {{ r.actual_rpo_hours }} 小時 | {{ '✅ 達標' if r.rpo_ok else '❌ 超標' }} |
{% endfor %}

---

## 各項驗證結果

| 項目 | 類型 | 狀態 | 備註 |
|------|------|------|------|
{% for r in dr_results %}
| {{ r.name }} | {{ r.type }} | {{ '✅ PASS' if r.status == 'PASS' else '❌ FAIL' if r.status == 'FAIL' else '⚠️ WARN' }} | {{ r.note | default(r.message | default('')) }} |
{% endfor %}

---

## 資料完整性

{% for r in dr_results | selectattr('type', 'eq', 'integrity') | list %}
### {{ r.name }}
- **狀態：** {{ r.status }}
{% if r.tables is defined %}
- **還原資料表數：** {{ r.tables }}
- **總資料行數：** {{ r.rows }}
{% endif %}
{% if r.health_response is defined %}
- **Health Check：** {{ r.health_response }}
{% endif %}
{% endfor %}

---

## 告警項目

{% if dr_alerts | length == 0 %}
✅ 本次演練無告警項目。
{% else %}
{% for alert in dr_alerts %}
- **[{{ alert.level }}]** {{ alert.name }}：{{ alert.message }}
{% endfor %}
{% endif %}

---

## 後續行動

{% if dr_overall_status != 'PASS' %}
⚠️ **本次 DR 演練有失敗或警告項目，請於 5 個工作天內完成改善：**

{% for r in dr_results | selectattr('status', 'ne', 'PASS') | list %}
- [ ] **{{ r.name }}**：{{ r.note | default('請調查失敗原因') }}
{% endfor %}
{% else %}
✅ 本次 DR 演練全部通過，無需立即行動。下次演練：{{ ansible_date_time.date }}（建議 90 天後）
{% endif %}

---
_本報告由 Ansible backup_validator role 自動產生_
_演練結束時間：{{ ansible_date_time.iso8601 }}_
```

---

## 12.3 備份 Playbook：自動備份 GitLab 與 PostgreSQL

```yaml
# playbooks/backup.yml
---
- name: Perform scheduled backups
  hosts: gitlab_servers
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  tasks:
    # ── GitLab 備份 ──────────────────────────────────────────────────────
    - name: Trigger GitLab backup
      community.docker.docker_container_exec:
        container: gitlab-ce
        command: gitlab-backup create STRATEGY=copy
      register: gitlab_backup_result
      changed_when: true

    - name: Verify GitLab backup was created
      ansible.builtin.find:
        paths: /opt/gitlab/data/backups
        patterns: "*_gitlab_backup.tar"
        age: "-1h"
      register: new_backup
      failed_when: new_backup.files | length == 0

    # ── PostgreSQL 備份 ───────────────────────────────────────────────────
    - name: Perform PostgreSQL dump
      community.docker.docker_container_exec:
        container: gitlab-postgres
        command: >
          sh -c "pg_dump -U gitlab gitlabhq_production
                 | gzip > /tmp/gitlab-db-{{ ansible_date_time.date }}.sql.gz"
      changed_when: true

    - name: Copy PostgreSQL dump out of container
      ansible.builtin.command:
        cmd: >
          docker cp
          gitlab-postgres:/tmp/gitlab-db-{{ ansible_date_time.date }}.sql.gz
          /backup/postgresql/
      changed_when: true

    # ── 備份保留政策（保留最近 7 份）────────────────────────────────────
    - name: Remove old PostgreSQL backups (keep 7 days)
      ansible.builtin.find:
        paths: /backup/postgresql
        patterns: "*.sql.gz"
        age: "7d"
        file_type: file
      register: old_backups

    - name: Delete old backups
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
```

---

## 12.4 DR 演練 Playbook 與 GitLab CI/CD 整合

```yaml
# playbooks/dr_validation.yml
---
- name: Perform DR validation exercise
  hosts: gitlab_servers
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: backup_validator
      tags: [dr, validate]

  post_tasks:
    - name: Commit DR report to audit-evidence
      ansible.builtin.include_tasks:
        file: ../roles/evidence_collector/tasks/git_commit.yml
      vars:
        evidence_report_dir: "{{ dr_report_dir }}"
      tags: [dr, git]
```

```yaml
# .gitlab-ci.yml 新增 DR 演練排程
monthly-dr-validation:
  stage: evidence-collect
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/dr_validation.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -v
  artifacts:
    when: always
    paths:
      - dr-reports/
    expire_in: 2 years    # DR 記錄保留更長（稽核追溯需求）
  rules:
    # 每月第一個週日 03:00 執行（避免影響業務）
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "monthly_dr"
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  # DR 演練允許一定失敗率（但會記錄）
  allow_failure: false
```

> **Schedule 設定：** `0 3 * * 0`（每週日 03:00，或改為 `0 3 1 * *` 每月 1 日）

---

## 12.5 驗證步驟

```bash
# 1. 先確認有備份存在
ls -lah /opt/gitlab/data/backups/
ls -lah /backup/postgresql/

# 2. 手動執行 DR 驗證（加 -v 觀察詳細過程）
ansible-playbook playbooks/dr_validation.yml \
  --vault-password-file ~/.vault_pass \
  -v

# 3. 查看報告
cat dr-reports/$(date +%Y-%m-%d)/gitlab-01.md

# 4. 確認 RTO 實測值符合政策（4 小時內）
grep "RTO" dr-reports/$(date +%Y-%m-%d)/gitlab-01.md

# 5. 確認隔離容器已清理（不應有殘留）
docker ps -a | grep "dr-"
# 預期：無輸出

# 6. 確認報告已推送到 audit-evidence
cd /opt/audit-evidence-repo && git log --oneline -3

# 7. 模擬備份損毀，確認告警機制正常
# 暫時移除最新備份
mv /backup/postgresql/*.sql.gz /tmp/

# 執行驗證
ansible-playbook playbooks/dr_validation.yml \
  --vault-password-file ~/.vault_pass

# 報告應顯示 CRITICAL 告警
# 還原備份
mv /tmp/*.sql.gz /backup/postgresql/
```

---

## 12.6 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| 備份「存在性」vs「還原驗證」的差異 | ☐ 能說明兩者對稽核員的不同意義 |
| 隔離 Docker 容器作為 DR 測試環境 | ☐ 還原後容器不影響生產服務 |
| RTO 自動計時與記錄 | ☐ 報告顯示實際 RTO 並與政策目標對比 |
| RPO 計算（最後備份距今時間）| ☐ 可識別超過 RPO 目標的備份 |
| `always: true` 確保清理 task 一定執行 | ☐ 即使驗證失敗，隔離容器也被清除 |
| 月度 DR 演練排程 | ☐ GitLab Schedule 正確設定，每月自動執行 |

---

## 完整 ISO 27001 自動化教材總結

| 章節 | 核心產出 | ISO 27001 |
|------|----------|-----------|
| 第 1-5 章 | 自動化運維基礎 | 技術基礎設施 |
| 第 6 章 | 資產清冊（Snipe-IT）| A.5.9 |
| 第 7 章 | CIS 合規報告 | A.8.8 / A.8.9 |
| 第 8 章 | 週期稽核證據 | A.5.16 / A.8.3 |
| 第 9 章 | 52 週稽核時間線 | A.8.16 |
| 第 10 章 | 憑證到期監控 | A.8.24 |
| 第 11 章 | 帳號生命週期 | A.5.16 / A.5.18 |
| 第 12 章 | DR 演練記錄 | A.8.13 |

---

*← [返回總覽](./README.md) | [上一章](./chapter-11-account-lifecycle.md)*
