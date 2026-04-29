# 第 8 章：自動化稽核證據收集器（Evidence Collector）

> **學習目標：** 撰寫一套完整的 Evidence Collector Playbook，定期收集帳號審查、套件清單、備份驗證、SUID/777 權限掃描、SSH 設定快照等稽核素材，以 Markdown 格式產出後自動透過 `git commit` 推送到專屬的 `audit-evidence` 倉庫，每份報告含時間戳記，提供完整的稽核軌跡。

---

## 8.1 理論說明：什麼是稽核證據鏈？

### 8.1.1 ISO 27001 稽核員的思維

外部稽核員在驗證控制措施時，最核心的問題是：

> **「你怎麼證明這件事確實有在做，而不只是說說而已？」**

單憑口頭說明或一份政策文件是不夠的。ISO 27001 需要**持續性的操作記錄（Operational Evidence）**——包含時間、執行者、結果的具體記錄。

### 8.1.2 Git 作為稽核證據庫

將稽核報告 `git commit` 到獨立倉庫的優勢：

```
┌────────────────────────────────────────────────────────────────┐
│               Git 作為稽核證據庫的優勢                          │
│                                                                │
│  ✅ 每份報告有精確的時間戳記（commit timestamp）               │
│  ✅ 修改歷史不可竄改（Git hash 機制）                          │
│  ✅ 稽核員可直接瀏覽 GitLab UI，無需特殊工具                   │
│  ✅ diff 功能讓「變化」一目了然（新增帳號、套件異動）           │
│  ✅ 可設定 Protected Branch 防止事後篡改                       │
└────────────────────────────────────────────────────────────────┘
```

### 8.1.3 收集項目對應 ISO 27001 控制措施

| 稽核收集項目 | 對應 ISO 27001 | 說明 |
|-------------|----------------|------|
| 帳號清單審查 | A.5.16 身分管理 | 確認無未授權帳號存在 |
| sudoers 設定 | A.8.2 特殊存取權限 | 確認特權帳號受到管控 |
| 已安裝套件清單 | A.8.8 技術漏洞管理 | 追蹤軟體版本，識別已知 CVE |
| 備份驗證 | A.8.13 資訊備份 | 確認備份確實存在且可用 |
| SUID/SGID 檔案 | A.8.3 資訊存取限制 | 偵測異常提權路徑 |
| 777 權限檔案 | A.8.3 資訊存取限制 | 偵測過度開放的權限 |
| SSH 設定快照 | A.8.20 網路安全 | 確認 SSH 強化設定未被修改 |
| 開放服務清單 | A.8.21 網路服務安全 | 確認只有必要埠號開放 |
| 登入失敗記錄 | A.8.16 監控活動 | 識別暴力破解嘗試 |

---

## 8.2 Ansible Role 結構：evidence_collector

```bash
ansible-galaxy role init roles/evidence_collector
```

```
roles/evidence_collector/
├── defaults/
│   └── main.yml          ← 報告路徑、git 設定、收集項目開關
├── tasks/
│   ├── main.yml          ← 任務入口，依序呼叫各收集模組
│   ├── accounts.yml      ← 帳號與 sudo 審查
│   ├── packages.yml      ← 已安裝套件清單
│   ├── backups.yml       ← 備份驗證
│   ├── permissions.yml   ← SUID/SGID/777 掃描
│   ├── ssh_config.yml    ← SSH 設定快照
│   ├── network.yml       ← 開放服務與防火牆規則
│   ├── logins.yml        ← 登入記錄分析
│   ├── assemble.yml      ← 將所有片段組合成完整報告
│   └── git_commit.yml    ← 推送到 audit-evidence 倉庫
└── templates/
    └── report_header.j2  ← 報告標頭模板
```

### 8.2.1 defaults/main.yml

```yaml
# roles/evidence_collector/defaults/main.yml
---
# ── 報告設定 ──────────────────────────────────
evidence_report_dir: /opt/evidence-reports
evidence_timestamp: "{{ ansible_date_time.iso8601_basic_short }}"
evidence_date: "{{ ansible_date_time.date }}"

# ── audit-evidence Git 倉庫設定 ───────────────
# 這個倉庫獨立於 ansible 程式碼，專門存放稽核證據
audit_evidence_repo_url: "{{ vault_audit_evidence_repo_url }}"
audit_evidence_local_path: /opt/audit-evidence-repo
audit_evidence_branch: main

# Git commit 者身份（CI bot 帳號）
git_committer_name: "Ansible Audit Bot"
git_committer_email: "ansible-bot@{{ ansible_domain | default('example.com') }}"

# ── 收集項目開關（可選擇性關閉）──────────────
collect_accounts: true
collect_packages: true
collect_backups: true
collect_permissions: true
collect_ssh_config: true
collect_network: true
collect_login_history: true

# ── 備份驗證設定 ──────────────────────────────
backup_check_paths:
  - path: /backup
    min_size_mb: 1        # 最小合理大小（MB）
    max_age_hours: 25     # 超過 25 小時未更新視為異常（考慮排程偏移）
  - path: /opt/gitlab/data/backups
    min_size_mb: 100
    max_age_hours: 25

# ── 權限掃描排除路徑 ──────────────────────────
permission_scan_paths:
  - /usr
  - /etc
  - /var
  - /home
  - /root
  - /opt

permission_exclude_paths:
  - /proc
  - /sys
  - /dev
  - /run
  - /snap

# ── 危險 SUID 二進位白名單（正常系統存在的 SUID）─
suid_whitelist:
  - /usr/bin/sudo
  - /usr/bin/su
  - /usr/bin/passwd
  - /usr/bin/chfn
  - /usr/bin/chsh
  - /usr/bin/gpasswd
  - /usr/bin/newgrp
  - /usr/lib/openssh/ssh-keysign
  - /usr/bin/mount
  - /usr/bin/umount
  - /usr/bin/ping
```

---

## 8.3 Evidence Collector 核心實作

### 8.3.1 tasks/main.yml

```yaml
# roles/evidence_collector/tasks/main.yml
---
# ── 建立報告目錄 ─────────────────────────────────────────────────────────
- name: Ensure evidence report directory exists
  ansible.builtin.file:
    path: "{{ evidence_report_dir }}/{{ evidence_date }}"
    state: directory
    mode: '0750'
  tags: [evidence]

# ── 初始化本次報告的 fact 容器 ───────────────────────────────────────────
- name: Initialize evidence facts container
  ansible.builtin.set_fact:
    evidence_sections: {}   # 各 section 的 Markdown 內容
    evidence_findings: []   # 發現的異常項目清單
  tags: [evidence]

# ── 依序執行各收集模組 ───────────────────────────────────────────────────
- name: Collect account information
  ansible.builtin.include_tasks: accounts.yml
  when: collect_accounts | bool
  tags: [evidence, accounts]

- name: Collect package list
  ansible.builtin.include_tasks: packages.yml
  when: collect_packages | bool
  tags: [evidence, packages]

- name: Verify backups
  ansible.builtin.include_tasks: backups.yml
  when: collect_backups | bool
  tags: [evidence, backups]

- name: Scan file permissions
  ansible.builtin.include_tasks: permissions.yml
  when: collect_permissions | bool
  tags: [evidence, permissions]

- name: Snapshot SSH configuration
  ansible.builtin.include_tasks: ssh_config.yml
  when: collect_ssh_config | bool
  tags: [evidence, ssh]

- name: Collect network service info
  ansible.builtin.include_tasks: network.yml
  when: collect_network | bool
  tags: [evidence, network]

- name: Collect login history
  ansible.builtin.include_tasks: logins.yml
  when: collect_login_history | bool
  tags: [evidence, logins]

# ── 組合完整報告 ─────────────────────────────────────────────────────────
- name: Assemble final report
  ansible.builtin.include_tasks: assemble.yml
  tags: [evidence, assemble]

# ── 推送到 audit-evidence 倉庫 ───────────────────────────────────────────
- name: Commit report to audit-evidence repository
  ansible.builtin.include_tasks: git_commit.yml
  tags: [evidence, git]
```

### 8.3.2 tasks/accounts.yml

```yaml
# roles/evidence_collector/tasks/accounts.yml
---
# ── 取得所有本地使用者帳號 ───────────────────────────────────────────────
- name: Get all local users from /etc/passwd
  ansible.builtin.getent:
    database: passwd
  register: local_users

# ── 取得所有有效群組 ─────────────────────────────────────────────────────
- name: Get all local groups
  ansible.builtin.getent:
    database: group

# ── 取得 sudo 權限設定 ──────────────────────────────────────────────────
- name: Read sudoers configuration
  ansible.builtin.command:
    cmd: getent shadow
  register: shadow_info
  changed_when: false
  no_log: true  # 不在日誌中顯示密碼 hash

- name: Get sudo group members
  ansible.builtin.command:
    cmd: "getent group sudo"
  register: sudo_group
  changed_when: false

- name: List files in sudoers.d
  ansible.builtin.find:
    paths: /etc/sudoers.d
    file_type: file
  register: sudoers_d_files

- name: Read each sudoers.d file
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  loop: "{{ sudoers_d_files.files }}"
  register: sudoers_d_contents

# ── 識別有 shell 的非系統帳號（UID >= 1000）──────────────────────────────
- name: Filter interactive users (UID >= 1000)
  ansible.builtin.set_fact:
    interactive_users: >-
      {{
        ansible_facts.getent_passwd
        | dict2items
        | selectattr('value.2', 'ge', '1000')
        | selectattr('value.6', 'ne', '/usr/sbin/nologin')
        | selectattr('value.6', 'ne', '/bin/false')
        | list
      }}

# ── 識別有 UID 0 的帳號（除 root 外都是異常）────────────────────────────
- name: Check for accounts with UID 0 (besides root)
  ansible.builtin.set_fact:
    uid0_accounts: >-
      {{
        ansible_facts.getent_passwd
        | dict2items
        | selectattr('value.2', 'eq', '0')
        | rejectattr('key', 'eq', 'root')
        | list
      }}

# ── 識別有密碼的帳號（建議全部使用 SSH 金鑰）────────────────────────────
- name: Check for accounts with password set
  ansible.builtin.shell: |
    awk -F: '$2 !~ /^[!*]/ && $2 != "" {print $1}' /etc/shadow
  register: accounts_with_password
  changed_when: false
  no_log: true

# ── 組合 accounts section Markdown ──────────────────────────────────────
- name: Build accounts section
  ansible.builtin.set_fact:
    evidence_accounts_md: |
      ## 2. 帳號審查（ISO 27001 A.5.16）

      **收集時間：** {{ ansible_date_time.iso8601 }}

      ### 2.1 互動式使用者帳號（UID ≥ 1000）

      | 使用者 | UID | GID | 家目錄 | Shell |
      |--------|-----|-----|--------|-------|
      {% for user in interactive_users %}
      | `{{ user.key }}` | {{ user.value[2] }} | {{ user.value[3] }} | {{ user.value[5] }} | {{ user.value[6] }} |
      {% endfor %}

      ### 2.2 Sudo 群組成員

      ```
      {{ sudo_group.stdout }}
      ```

      ### 2.3 sudoers.d 設定檔

      {% for file_content in sudoers_d_contents.results %}
      **檔案：** `{{ file_content.item.path }}`
      ```
      {{ file_content.content | b64decode }}
      ```
      {% endfor %}

      ### 2.4 ⚠️ 異常帳號檢查

      {% if uid0_accounts | length > 0 %}
      **❌ 發現 UID=0 的非 root 帳號（高風險）：**
      {% for acct in uid0_accounts %}
      - `{{ acct.key }}`
      {% endfor %}
      {% else %}
      ✅ 無 UID=0 的異常帳號
      {% endif %}

      {% if accounts_with_password.stdout_lines | length > 0 %}
      **⚠️ 以下帳號設有密碼（建議改用 SSH 金鑰認證）：**
      {% for acct in accounts_with_password.stdout_lines %}
      - `{{ acct }}`
      {% endfor %}
      {% else %}
      ✅ 所有帳號均使用金鑰認證或已鎖定密碼
      {% endif %}

# ── 將異常加入 findings ──────────────────────────────────────────────────
- name: Add UID0 findings to evidence
  ansible.builtin.set_fact:
    evidence_findings: "{{ evidence_findings + ['[HIGH] 發現 UID=0 非 root 帳號: ' + item.key] }}"
  loop: "{{ uid0_accounts }}"
  when: uid0_accounts | length > 0
```

### 8.3.3 tasks/packages.yml

```yaml
# roles/evidence_collector/tasks/packages.yml
---
- name: Gather installed package facts
  ansible.builtin.package_facts:
    manager: apt

- name: Get apt security updates available
  ansible.builtin.shell: |
    apt-get -s upgrade 2>/dev/null \
      | grep "^Inst" \
      | grep -i security \
      | awk '{print $2, $3}' \
      | head -30
  register: security_updates
  changed_when: false

- name: Get last apt upgrade time
  ansible.builtin.stat:
    path: /var/log/dpkg.log
  register: dpkg_log_stat

- name: Read last few dpkg log entries
  ansible.builtin.shell: |
    grep " upgrade " /var/log/dpkg.log 2>/dev/null | tail -10
  register: last_upgrades
  changed_when: false

- name: Build packages section
  ansible.builtin.set_fact:
    evidence_packages_md: |
      ## 3. 套件清單與更新狀態（ISO 27001 A.8.8）

      **收集時間：** {{ ansible_date_time.iso8601 }}

      ### 3.1 已安裝套件數量

      - 總計：**{{ ansible_facts.packages | length }}** 個套件

      ### 3.2 關鍵套件版本

      | 套件 | 版本 |
      |------|------|
      {% for pkg in ['openssh-server', 'openssl', 'linux-image-generic', 'docker-ce', 'fail2ban', 'ufw'] %}
      {% if pkg in ansible_facts.packages %}
      | `{{ pkg }}` | {{ ansible_facts.packages[pkg][0].version }} |
      {% else %}
      | `{{ pkg }}` | _(未安裝)_ |
      {% endif %}
      {% endfor %}

      ### 3.3 待安裝安全更新

      {% if security_updates.stdout_lines | length > 0 %}
      **⚠️ 以下安全更新尚未套用：**
      ```
      {{ security_updates.stdout }}
      ```
      {% else %}
      ✅ 無待安裝的安全更新
      {% endif %}

      ### 3.4 最近 10 筆套件升級記錄

      ```
      {{ last_upgrades.stdout if last_upgrades.stdout else '無記錄' }}
      ```

- name: Add pending security updates to findings
  ansible.builtin.set_fact:
    evidence_findings: >-
      {{
        evidence_findings +
        ['[MEDIUM] 有 ' + (security_updates.stdout_lines | length | string) + ' 個安全更新待套用']
      }}
  when: security_updates.stdout_lines | length > 0
```

### 8.3.4 tasks/permissions.yml

```yaml
# roles/evidence_collector/tasks/permissions.yml
---
# ── 掃描 SUID/SGID 檔案 ──────────────────────────────────────────────────
- name: Find SUID files
  ansible.builtin.command:
    cmd: >
      find {{ permission_scan_paths | join(' ') }}
      -xdev -perm -4000 -type f
      {{ permission_exclude_paths | map('regex_replace', '^(.*)$', '-not -path \1/*') | join(' ') }}
      2>/dev/null
  register: suid_files
  changed_when: false

- name: Find SGID files
  ansible.builtin.command:
    cmd: >
      find {{ permission_scan_paths | join(' ') }}
      -xdev -perm -2000 -type f
      2>/dev/null
  register: sgid_files
  changed_when: false

# ── 掃描 777 權限檔案（高風險）──────────────────────────────────────────
- name: Find world-writable files (777)
  ansible.builtin.command:
    cmd: >
      find {{ permission_scan_paths | join(' ') }}
      -xdev -perm -0002 -type f
      -not -path /proc/*
      -not -path /sys/*
      2>/dev/null
  register: world_writable_files
  changed_when: false

# ── 找出未預期的 SUID（不在白名單中）───────────────────────────────────
- name: Identify unexpected SUID binaries
  ansible.builtin.set_fact:
    unexpected_suid: >-
      {{
        suid_files.stdout_lines
        | difference(suid_whitelist)
      }}

- name: Build permissions section
  ansible.builtin.set_fact:
    evidence_permissions_md: |
      ## 5. 檔案權限掃描（ISO 27001 A.8.3）

      **收集時間：** {{ ansible_date_time.iso8601 }}

      ### 5.1 SUID 檔案清單

      **總計 {{ suid_files.stdout_lines | length }} 個 SUID 檔案**

      {% if unexpected_suid | length > 0 %}
      **❌ 發現非預期的 SUID 二進位（需調查）：**
      {% for f in unexpected_suid %}
      - `{{ f }}`
      {% endfor %}
      {% else %}
      ✅ 所有 SUID 檔案均在白名單內
      {% endif %}

      <details>
      <summary>完整 SUID 清單（展開）</summary>

      ```
      {{ suid_files.stdout if suid_files.stdout else '無' }}
      ```
      </details>

      ### 5.2 SGID 檔案清單

      **總計 {{ sgid_files.stdout_lines | length }} 個 SGID 檔案**

      <details>
      <summary>完整 SGID 清單（展開）</summary>

      ```
      {{ sgid_files.stdout if sgid_files.stdout else '無' }}
      ```
      </details>

      ### 5.3 777 權限檔案（World-Writable）

      {% if world_writable_files.stdout_lines | length > 0 %}
      **❌ 發現 {{ world_writable_files.stdout_lines | length }} 個 777 權限檔案（高風險）：**
      ```
      {{ world_writable_files.stdout }}
      ```
      {% else %}
      ✅ 未發現 777 權限檔案
      {% endif %}

- name: Add unexpected SUID to findings
  ansible.builtin.set_fact:
    evidence_findings: >-
      {{
        evidence_findings +
        ['[HIGH] 發現非預期 SUID 檔案: ' + item]
      }}
  loop: "{{ unexpected_suid }}"
  when: unexpected_suid | length > 0

- name: Add world-writable findings
  ansible.builtin.set_fact:
    evidence_findings: >-
      {{
        evidence_findings +
        ['[HIGH] 發現 777 權限檔案: ' + item]
      }}
  loop: "{{ world_writable_files.stdout_lines }}"
  when: world_writable_files.stdout_lines | length > 0
```

### 8.3.5 tasks/assemble.yml

```yaml
# roles/evidence_collector/tasks/assemble.yml
---
- name: Build report header
  ansible.builtin.set_fact:
    evidence_header_md: |
      # 系統安全性稽核證據報告

      | 項目 | 內容 |
      |------|------|
      | **主機名稱** | `{{ inventory_hostname }}` |
      | **IP 位址** | `{{ ansible_default_ipv4.address }}` |
      | **作業系統** | {{ ansible_distribution }} {{ ansible_distribution_version }} |
      | **Kernel** | {{ ansible_kernel }} |
      | **收集時間** | {{ ansible_date_time.iso8601 }} |
      | **收集工具** | Ansible Evidence Collector v1.0 |
      | **執行帳號** | {{ ansible_user_id }} |

      ---

      ## 1. 執行摘要

      {% if evidence_findings | length == 0 %}
      ✅ **本次掃描未發現異常項目。**
      {% else %}
      ⚠️ **本次掃描發現 {{ evidence_findings | length }} 個需注意項目：**

      {% for finding in evidence_findings %}
      - {{ finding }}
      {% endfor %}
      {% endif %}

      ---

- name: Assemble full report
  ansible.builtin.copy:
    content: >-
      {{ evidence_header_md }}
      {{ evidence_accounts_md | default('') }}
      {{ evidence_packages_md | default('') }}
      {{ evidence_backups_md | default('') }}
      {{ evidence_permissions_md | default('') }}
      {{ evidence_ssh_md | default('') }}
      {{ evidence_network_md | default('') }}
      {{ evidence_logins_md | default('') }}
    dest: "{{ evidence_report_dir }}/{{ evidence_date }}/{{ inventory_hostname }}-evidence.md"
    mode: '0640'

- name: Show report location
  ansible.builtin.debug:
    msg: "報告已產出：{{ evidence_report_dir }}/{{ evidence_date }}/{{ inventory_hostname }}-evidence.md"
```

### 8.3.6 tasks/git_commit.yml

```yaml
# roles/evidence_collector/tasks/git_commit.yml
# 這個任務在 Control Node（delegate_to: localhost）執行
---
# ── 確認 git 已安裝在 Control Node ──────────────────────────────────────
- name: Ensure git is installed on control node
  ansible.builtin.package:
    name: git
    state: present
  delegate_to: localhost
  run_once: true

# ── Clone 或更新 audit-evidence 倉庫 ────────────────────────────────────
- name: Clone audit-evidence repository (if not exists)
  ansible.builtin.git:
    repo: "{{ audit_evidence_repo_url }}"
    dest: "{{ audit_evidence_local_path }}"
    version: "{{ audit_evidence_branch }}"
    accept_hostkey: true
    # 使用 deploy key（從 Vault 注入到 ~/.ssh/audit_evidence_deploy_key）
    key_file: "~/.ssh/audit_evidence_deploy_key"
    force: false
  delegate_to: localhost
  run_once: true

# ── 建立對應日期的目錄結構 ───────────────────────────────────────────────
- name: Create date directory in audit-evidence repo
  ansible.builtin.file:
    path: "{{ audit_evidence_local_path }}/{{ evidence_date }}"
    state: directory
    mode: '0755'
  delegate_to: localhost

# ── 複製報告到 git 工作目錄 ──────────────────────────────────────────────
- name: Copy evidence report to git repository
  ansible.builtin.copy:
    src: "{{ evidence_report_dir }}/{{ evidence_date }}/{{ inventory_hostname }}-evidence.md"
    dest: "{{ audit_evidence_local_path }}/{{ evidence_date }}/{{ inventory_hostname }}-evidence.md"
    remote_src: false  # src 在 Control Node（fetch 已在 assemble 後完成）
  delegate_to: localhost

# ── 更新 README index ─────────────────────────────────────────────────────
- name: Update or create monthly index file
  ansible.builtin.lineinfile:
    path: "{{ audit_evidence_local_path }}/{{ evidence_date }}/README.md"
    line: "- [{{ inventory_hostname }}](./{{ inventory_hostname }}-evidence.md) — {{ ansible_date_time.iso8601 }}"
    create: true
    mode: '0644'
  delegate_to: localhost

# ── Git 設定與 Commit ─────────────────────────────────────────────────────
- name: Configure git committer identity
  ansible.builtin.shell: |
    cd {{ audit_evidence_local_path }}
    git config user.name "{{ git_committer_name }}"
    git config user.email "{{ git_committer_email }}"
  delegate_to: localhost
  changed_when: true

- name: Stage and commit evidence report
  ansible.builtin.shell: |
    cd {{ audit_evidence_local_path }}
    git add {{ evidence_date }}/
    # 若無新變更則跳過（避免空 commit）
    if git diff --staged --quiet; then
      echo "NO_CHANGES"
    else
      git commit -m "evidence({{ evidence_date }}): {{ inventory_hostname }} auto-collected

      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      Collected: {{ ansible_date_time.iso8601 }}
      Findings: {{ evidence_findings | length }}
      Collector: Ansible Evidence Collector"
      echo "COMMITTED"
    fi
  register: git_commit_result
  delegate_to: localhost
  changed_when: "'COMMITTED' in git_commit_result.stdout"

- name: Push to audit-evidence repository
  ansible.builtin.shell: |
    cd {{ audit_evidence_local_path }}
    GIT_SSH_COMMAND="ssh -i ~/.ssh/audit_evidence_deploy_key -o StrictHostKeyChecking=no" \
    git push origin {{ audit_evidence_branch }}
  delegate_to: localhost
  when: "'COMMITTED' in git_commit_result.stdout"
  register: git_push_result

- name: Show commit result
  ansible.builtin.debug:
    msg: >-
      {{
        '✅ 報告已 commit 並推送到 audit-evidence 倉庫'
        if 'COMMITTED' in git_commit_result.stdout
        else '✅ 無變更，跳過 commit（本次結果與上次相同）'
      }}
```

---

## 8.4 完整 Evidence Collector Playbook

```yaml
# playbooks/evidence_collector.yml
---
- name: Collect security evidence from all hosts
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: true
  become: true
  serial: 5       # 每次處理 5 台，避免 Control Node 報告目錄競爭

  vars_files:
    - ../vault/secrets.yml

  vars:
    # 此次收集的標籤（用於 git commit message）
    evidence_collection_label: "{{ collection_label | default('scheduled') }}"

  pre_tasks:
    - name: Setup deploy key for audit-evidence repo
      ansible.builtin.copy:
        content: "{{ vault_audit_evidence_deploy_key }}"
        dest: "~/.ssh/audit_evidence_deploy_key"
        mode: '0600'
      delegate_to: localhost
      run_once: true
      no_log: true

  roles:
    - role: evidence_collector
      tags: [evidence]

  post_tasks:
    - name: Clean up deploy key
      ansible.builtin.file:
        path: "~/.ssh/audit_evidence_deploy_key"
        state: absent
      delegate_to: localhost
      run_once: true
```

---

## 8.5 Vault 需新增的機密

```yaml
# vault/secrets.yml 新增：
vault_audit_evidence_repo_url: "git@gitlab.example.com:security/audit-evidence-2026.git"
vault_audit_evidence_deploy_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  [audit-evidence 倉庫的 Deploy Key 私鑰]
  -----END OPENSSH PRIVATE KEY-----
```

> **建立 Deploy Key：**
> ```bash
> ssh-keygen -t ed25519 -C "ansible-audit-bot" -f ~/.ssh/audit_evidence_deploy_key
> # 將公鑰加入 GitLab audit-evidence 專案的 Deploy Keys（啟用 Write access）
> # 將私鑰存入 Vault
> ```

---

## 8.6 驗證步驟

```bash
# 1. 執行 Evidence Collector（對所有主機）
ansible-playbook playbooks/evidence_collector.yml \
  --vault-password-file ~/.vault_pass \
  -v

# 2. 查看產出的報告
cat /opt/evidence-reports/$(date +%Y-%m-%d)/web-01-evidence.md

# 3. 確認 git commit 已推送
cd /opt/audit-evidence-repo
git log --oneline -5
# 預期看到類似：
# a1b2c3d evidence(2026-04-29): web-01 auto-collected
# e5f6g7h evidence(2026-04-29): db-01 auto-collected

# 4. 查看兩次報告的 diff（模擬稽核員的視角）
git diff HEAD~2 HEAD -- 2026-04-29/web-01-evidence.md

# 5. 在 GitLab UI 中模擬稽核員查閱
# 進入 audit-evidence 專案 → Repository → 選擇日期目錄 → 點擊 .md 檔案

# 6. 測試只收集特定主機
ansible-playbook playbooks/evidence_collector.yml \
  --vault-password-file ~/.vault_pass \
  --limit web-01 \
  --tags evidence,accounts

# 7. 驗證異常偵測（手動建立 777 檔案測試）
ansible web-01 -m ansible.builtin.file \
  -a "path=/tmp/test-777.txt state=touch mode=0777"

ansible-playbook playbooks/evidence_collector.yml \
  --vault-password-file ~/.vault_pass \
  --limit web-01 --tags evidence,permissions

# 報告應包含：[HIGH] 發現 777 權限檔案: /tmp/test-777.txt

# 清理測試檔案
ansible web-01 -m ansible.builtin.file \
  -a "path=/tmp/test-777.txt state=absent"
```

---

## 8.7 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| 理解稽核證據鏈的重要性 | ☐ 能說明為何用 Git 儲存比 NFS 或 Email 更好 |
| getent/find/stat 模組收集系統資訊 | ☐ 能手動執行各收集任務並確認輸出 |
| Markdown 報告自動組合 | ☐ 報告包含標頭、各 section、異常摘要 |
| git commit 自動推送 | ☐ audit-evidence 倉庫有對應的 commit 記錄 |
| 異常偵測（findings）| ☐ 製造 777 檔案後報告能正確顯示 HIGH 警告 |
| Deploy Key 與 Vault 整合 | ☐ 私鑰在執行完後自動清除 |

**下一章：** [第 9 章 GitLab CI/CD 定期稽核流水線](./chapter-09-scheduled-audit-pipeline.md) — 將 CIS 掃描與 Evidence Collector 整合進 GitLab Scheduled Pipeline，建立完整的自動化稽核閉環。

---

*← [返回總覽](./README.md) | [上一章](./chapter-07-cis-benchmark.md)*
