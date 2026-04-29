# 第 11 章：帳號生命週期自動化

> **學習目標：** 使用 Ansible 建立完整的帳號生命週期管理流程——從新員工入職自動建立帳號、SSH 金鑰派送，到離職時的一鍵撤銷，以及每季自動產出存取審查報告（Access Review），涵蓋 ISO 27001 Annex A 5.16、5.18、8.2 的核心要求。

---

## 11.1 理論說明：帳號管理為何是 ISO 27001 稽核重點？

### 11.1.1 A.5.16 身分管理 / A.5.18 存取權利

ISO 27001 稽核員在驗證存取控制時，最常問的三個問題：

1. **「誰有權限存取哪些系統？有清單嗎？」** → 需要帳號清冊
2. **「前員工的帳號都停用了嗎？」** → 需要離職程序記錄
3. **「存取權限有定期審查嗎？多久一次？」** → 需要 Access Review 記錄

手動管理在人員少時尚可應付，但超過 10 台伺服器、20 名工程師後，「忘記停用前員工帳號」幾乎是必然會發生的問題——而這是稽核員最嚴重的扣分項之一。

### 11.1.2 本章採用的帳號管理模型

```
┌─────────────────────────────────────────────────────────────────┐
│                      帳號生命週期模型                             │
│                                                                 │
│  入職                    在職                    離職            │
│  ──────                 ──────                 ──────           │
│  建立帳號               定期審查                停用帳號          │
│  派送 SSH Key           （每季）                撤銷 SSH Key      │
│  設定 sudo 權限         異動申請                 移除 sudo 權限    │
│  加入正確群組           帳號異動記錄              鎖定帳號          │
│       │                     │                       │           │
│       └─────────────────────┴───────────────────────┘          │
│                             │                                   │
│                    全程 git commit                               │
│                    → audit-evidence                             │
└─────────────────────────────────────────────────────────────────┘
```

### 11.1.3 帳號資料來源設計

本章使用 **YAML 檔案作為「使用者目錄」**，納入 Git 版控：

```yaml
# inventory/users/all_users.yml（版控在 Ansible Repo 中）
users:
  - username: alice
    full_name: "Alice Chen"
    email: "alice@example.com"
    role: developer
    status: active         # active | offboarded
    ssh_pubkeys:
      - "ssh-ed25519 AAAA... alice@laptop"
    sudo_access: false
    groups: [developers]
    joined_date: "2025-03-01"
    offboarded_date: null

  - username: bob
    full_name: "Bob Lin"
    status: offboarded     # 已離職
    offboarded_date: "2026-01-15"
    ...
```

這個設計的優點：**每次人員異動都是一個 git commit，天然形成稽核軌跡。**

---

## 11.2 Ansible Role 結構：account_lifecycle

```bash
ansible-galaxy role init roles/account_lifecycle
```

```
roles/account_lifecycle/
├── defaults/
│   └── main.yml          ← 帳號政策預設值
├── tasks/
│   ├── main.yml          ← 入口，依模式執行
│   ├── provision.yml     ← 建立 / 更新帳號（入職）
│   ├── offboard.yml      ← 停用帳號（離職）
│   ├── ssh_keys.yml      ← SSH 金鑰管理
│   ├── sudo.yml          ← sudo 權限管理
│   └── review_report.yml ← 季度存取審查報告產出
└── templates/
    ├── sudoers_user.j2   ← 使用者 sudoers 片段
    └── access_review.md.j2 ← 存取審查報告模板
```

### 11.2.1 defaults/main.yml

```yaml
# roles/account_lifecycle/defaults/main.yml
---
# ── 帳號政策 ──────────────────────────────────
# 帳號 Shell（新帳號預設）
account_default_shell: /bin/bash

# 是否鎖定離職帳號（而非刪除，保留 UID 與 home 目錄供稽核）
account_offboard_lock: true

# 離職帳號的 home 目錄保留天數（0 = 永久保留）
account_offboard_home_retention_days: 90

# sudo 使用需密碼（timeout 5 分鐘）
account_sudo_password_required: true
account_sudo_timeout_minutes: 5

# ── 存取審查設定 ──────────────────────────────
access_review_report_dir: /opt/access-review-reports
access_review_inactive_days: 90    # 超過 90 天未登入列為「待審查」

# ── 使用者資料來源 ────────────────────────────
# 從 Ansible Repo 的 users 目錄讀取
users_file: "{{ playbook_dir }}/../inventory/users/all_users.yml"
```

### 11.2.2 tasks/main.yml

```yaml
# roles/account_lifecycle/tasks/main.yml
---
# ── 載入使用者資料 ───────────────────────────────────────────────────────
- name: Load users data from YAML file
  ansible.builtin.include_vars:
    file: "{{ users_file }}"
    name: user_directory

- name: Split active and offboarded users
  ansible.builtin.set_fact:
    active_users: >-
      {{ user_directory.users | selectattr('status', 'eq', 'active') | list }}
    offboarded_users: >-
      {{ user_directory.users | selectattr('status', 'eq', 'offboarded') | list }}

- name: Show user summary
  ansible.builtin.debug:
    msg:
      - "在職使用者：{{ active_users | length }} 人"
      - "已離職使用者：{{ offboarded_users | length }} 人"

# ── 依模式執行 ───────────────────────────────────────────────────────────
- name: Provision active user accounts
  ansible.builtin.import_tasks: provision.yml
  tags: [accounts, provision]

- name: Offboard departed user accounts
  ansible.builtin.import_tasks: offboard.yml
  tags: [accounts, offboard]

- name: Manage SSH keys
  ansible.builtin.import_tasks: ssh_keys.yml
  tags: [accounts, ssh]

- name: Manage sudo access
  ansible.builtin.import_tasks: sudo.yml
  tags: [accounts, sudo]
```

### 11.2.3 tasks/provision.yml

```yaml
# roles/account_lifecycle/tasks/provision.yml
---
- name: Create or update active user accounts
  ansible.builtin.user:
    name: "{{ item.username }}"
    comment: "{{ item.full_name }}"
    shell: "{{ item.shell | default(account_default_shell) }}"
    # groups 列表中第一個為主群組，其餘為附加群組
    groups: "{{ item.groups | default([]) }}"
    append: true
    # state=present：存在則更新，不存在則建立
    state: present
    # 禁用密碼登入（強制使用 SSH 金鑰）
    password: "!"
    # 若帳號已鎖定（前次 offboard），重新啟用
    password_lock: false
  loop: "{{ active_users }}"
  loop_control:
    label: "{{ item.username }}"

- name: Ensure home directory exists with correct permissions
  ansible.builtin.file:
    path: "/home/{{ item.username }}"
    state: directory
    owner: "{{ item.username }}"
    group: "{{ item.username }}"
    mode: '0750'    # 不允許其他人讀取 home 目錄
  loop: "{{ active_users }}"
  loop_control:
    label: "{{ item.username }}"

- name: Ensure .ssh directory exists
  ansible.builtin.file:
    path: "/home/{{ item.username }}/.ssh"
    state: directory
    owner: "{{ item.username }}"
    group: "{{ item.username }}"
    mode: '0700'
  loop: "{{ active_users }}"
  loop_control:
    label: "{{ item.username }}"
```

### 11.2.4 tasks/offboard.yml

```yaml
# roles/account_lifecycle/tasks/offboard.yml
---
# ── 鎖定離職帳號（不刪除，保留 UID 供日誌追蹤）──────────────────────────
- name: Lock offboarded user accounts
  ansible.builtin.user:
    name: "{{ item.username }}"
    # password_lock=true 等同 passwd -l，無法用密碼登入
    password_lock: true
    # 清空 shell 使無法互動式登入
    shell: /usr/sbin/nologin
    state: present   # 保留帳號，只鎖定
  loop: "{{ offboarded_users }}"
  loop_control:
    label: "{{ item.username }}"
  # 帳號不存在時跳過（可能從未在此主機建立）
  ignore_errors: true

# ── 撤銷所有 SSH 授權金鑰 ─────────────────────────────────────────────────
- name: Remove all SSH authorized keys for offboarded users
  ansible.builtin.file:
    path: "/home/{{ item.username }}/.ssh/authorized_keys"
    state: absent
  loop: "{{ offboarded_users }}"
  loop_control:
    label: "{{ item.username }}"
  ignore_errors: true

# ── 移除 sudo 權限 ───────────────────────────────────────────────────────
- name: Remove sudo access for offboarded users
  ansible.builtin.file:
    path: "/etc/sudoers.d/{{ item.username }}"
    state: absent
  loop: "{{ offboarded_users }}"
  loop_control:
    label: "{{ item.username }}"

# ── 終止該使用者現有的 session ─────────────────────────────────────────────
- name: Kill all active sessions for offboarded users
  ansible.builtin.shell: |
    pkill -KILL -u {{ item.username }} || true
  loop: "{{ offboarded_users }}"
  loop_control:
    label: "{{ item.username }}"
  changed_when: false
  ignore_errors: true

# ── 記錄離職操作 ─────────────────────────────────────────────────────────
- name: Log offboarding action
  ansible.builtin.lineinfile:
    path: /var/log/account_lifecycle.log
    line: >-
      {{ ansible_date_time.iso8601 }} | OFFBOARD |
      {{ item.username }} |
      offboarded_date={{ item.offboarded_date | default('unknown') }} |
      executed_by={{ ansible_user_id }} |
      host={{ inventory_hostname }}
    create: true
    mode: '0640'
  loop: "{{ offboarded_users }}"
  loop_control:
    label: "{{ item.username }}"
```

### 11.2.5 tasks/ssh_keys.yml

```yaml
# roles/account_lifecycle/tasks/ssh_keys.yml
---
# ── 為在職使用者部署 SSH 公鑰 ────────────────────────────────────────────
# exclusive=true：移除不在清單中的金鑰（防止後門）
# 這是最關鍵的設定——確保只有清單中的金鑰有效
- name: Deploy authorized SSH keys for active users
  ansible.posix.authorized_key:
    user: "{{ item.0.username }}"
    key: "{{ item.1 }}"
    # exclusive=true 代表這個使用者的 authorized_keys
    # 只會包含本次 loop 的所有金鑰
    state: present
  loop: "{{ active_users | subelements('ssh_pubkeys', skip_missing=True) }}"
  loop_control:
    label: "{{ item.0.username }}: {{ item.1[:30] }}..."
  no_log: true   # 不在 Ansible 輸出中顯示公鑰

# ── 使用 exclusive 模式確保清單以外的金鑰被移除 ──────────────────────────
# 注意：先用一般 loop 加入所有金鑰，再用 exclusive 確保沒有額外金鑰
- name: Enforce exclusive SSH key list (remove unlisted keys)
  ansible.posix.authorized_key:
    user: "{{ item.username }}"
    key: "{{ item.ssh_pubkeys | join('\n') }}"
    # exclusive=true 會清除所有不在 key 參數中的金鑰
    exclusive: true
    state: present
  loop: "{{ active_users | selectattr('ssh_pubkeys', 'defined') | list }}"
  loop_control:
    label: "{{ item.username }}"
  no_log: true
```

### 11.2.6 tasks/sudo.yml

```yaml
# roles/account_lifecycle/tasks/sudo.yml
---
# ── 為有 sudo 權限的使用者建立 sudoers 設定 ──────────────────────────────
- name: Create sudoers entry for users with sudo_access
  ansible.builtin.template:
    src: sudoers_user.j2
    dest: "/etc/sudoers.d/{{ item.username }}"
    owner: root
    group: root
    mode: '0440'
    # 語法驗證：錯誤的 sudoers 會鎖死整個 sudo
    validate: "/usr/sbin/visudo -cf %s"
  loop: "{{ active_users | selectattr('sudo_access', 'eq', true) | list }}"
  loop_control:
    label: "{{ item.username }}"

# ── 移除不再需要 sudo 的使用者設定 ──────────────────────────────────────
- name: Remove sudoers entry for users without sudo_access
  ansible.builtin.file:
    path: "/etc/sudoers.d/{{ item.username }}"
    state: absent
  loop: "{{ active_users | selectattr('sudo_access', 'eq', false) | list }}"
  loop_control:
    label: "{{ item.username }}"
```

### 11.2.7 templates/sudoers_user.j2

```jinja2
{# roles/account_lifecycle/templates/sudoers_user.j2 #}
# sudoers entry for {{ item.username }}
# 由 Ansible account_lifecycle 管理 — 請勿手動修改
# 最後更新：{{ ansible_date_time.iso8601 }}

# sudo 密碼逾時時間（{{ account_sudo_timeout_minutes }} 分鐘）
Defaults:{{ item.username }} timestamp_timeout={{ account_sudo_timeout_minutes }}

{% if item.sudo_commands is defined %}
# 限制只能執行指定指令
{{ item.username }} ALL=(ALL) {{ 'NOPASSWD:' if not account_sudo_password_required else '' }}{{ item.sudo_commands | join(', ') }}
{% else %}
# 允許所有指令（需密碼）
{{ item.username }} ALL=(ALL:ALL) {{ 'NOPASSWD:' if not account_sudo_password_required else '' }}ALL
{% endif %}
```

---

## 11.3 季度存取審查報告（Access Review）

這是 ISO 27001 A.5.18 最核心的要求——定期讓管理者確認每個人的存取權限是否仍然需要。

### 11.3.1 tasks/review_report.yml

```yaml
# roles/account_lifecycle/tasks/review_report.yml
---
# ── 取得最後登入時間 ─────────────────────────────────────────────────────
- name: Get last login times for all users
  ansible.builtin.command:
    cmd: lastlog --time {{ access_review_inactive_days }}
  register: inactive_users_raw
  changed_when: false

- name: Get full lastlog output
  ansible.builtin.command:
    cmd: lastlog
  register: full_lastlog
  changed_when: false

# ── 取得當前 SSH 連線 ────────────────────────────────────────────────────
- name: Get currently logged in users
  ansible.builtin.command:
    cmd: who
  register: current_sessions
  changed_when: false

# ── 取得 sudo 使用記錄 ───────────────────────────────────────────────────
- name: Get recent sudo usage from auth.log
  ansible.builtin.shell: |
    grep "sudo:" /var/log/auth.log 2>/dev/null | tail -50 \
      | grep -v "pam_unix" | grep "COMMAND"
  register: sudo_usage
  changed_when: false
  ignore_errors: true

# ── 分析每個帳號的當前狀態 ──────────────────────────────────────────────
- name: Check account lock status
  ansible.builtin.shell: |
    passwd -S {{ item.username }} 2>/dev/null | awk '{print $2}'
  register: account_status_raw
  loop: "{{ user_directory.users }}"
  loop_control:
    label: "{{ item.username }}"
  changed_when: false
  ignore_errors: true

# ── 取得每個使用者的 authorized_keys 數量 ───────────────────────────────
- name: Count SSH keys per user
  ansible.builtin.find:
    paths: "/home/{{ item.username }}/.ssh"
    patterns: "authorized_keys"
    file_type: file
  register: ssh_key_files
  loop: "{{ user_directory.users }}"
  loop_control:
    label: "{{ item.username }}"
  ignore_errors: true

# ── 產出存取審查報告 ─────────────────────────────────────────────────────
- name: Ensure access review report directory exists
  ansible.builtin.file:
    path: "{{ access_review_report_dir }}"
    state: directory
    mode: '0750'

- name: Generate Access Review report
  ansible.builtin.template:
    src: access_review.md.j2
    dest: >-
      {{ access_review_report_dir }}/access-review-{{ inventory_hostname }}-{{ ansible_date_time.date }}.md
    mode: '0640'

- name: Fetch report to control node
  ansible.builtin.fetch:
    src: >-
      {{ access_review_report_dir }}/access-review-{{ inventory_hostname }}-{{ ansible_date_time.date }}.md
    dest: "./access-review-reports/{{ ansible_date_time.date }}/{{ inventory_hostname }}.md"
    flat: true
```

### 11.3.2 templates/access_review.md.j2

```jinja2
{# roles/account_lifecycle/templates/access_review.md.j2 #}
# 存取權限審查報告（Access Review）

> **重要提示：** 請管理者審閱以下清單，確認每位使用者的存取權限仍然符合業務需要。
> 如有任何異動需求，請在 Ansible Repo 的 `inventory/users/all_users.yml` 提交 MR。

| 項目 | 內容 |
|------|------|
| **主機** | `{{ inventory_hostname }}` |
| **報告日期** | {{ ansible_date_time.date }} |
| **審查週期** | 季度（每 3 個月） |
| **產生者** | Ansible account_lifecycle role |
| **下次審查** | 請在 90 天內完成審查確認 |

---

## 1. 在職帳號清單（需管理者逐一確認）

| 使用者 | 姓名 | 角色 | 群組 | Sudo | SSH 金鑰數 | 狀態 | 動作 |
|--------|------|------|------|------|------------|------|------|
{% for user in active_users %}
| `{{ user.username }}` | {{ user.full_name }} | {{ user.role | default('N/A') }} | {{ user.groups | join(', ') }} | {{ '✅ 是' if user.sudo_access else '❌ 否' }} | {{ user.ssh_pubkeys | default([]) | length }} | 🟢 在職 | ☐ 確認 / ☐ 撤銷 |
{% endfor %}

> **管理者簽核：** ________________　**日期：** ________________

---

## 2. 近 {{ access_review_inactive_days }} 天未登入帳號（建議確認）

```
{{ inactive_users_raw.stdout if inactive_users_raw.stdout else '無未登入帳號' }}
```

---

## 3. 已離職帳號確認（應全部鎖定）

| 使用者 | 離職日期 | 帳號狀態 |
|--------|---------|---------|
{% for user in offboarded_users %}
| `{{ user.username }}` | {{ user.offboarded_date | default('未記錄') }} | 🔴 應已鎖定 |
{% endfor %}

---

## 4. 近期 Sudo 使用記錄

```
{{ sudo_usage.stdout | default('無記錄或 auth.log 不可讀') }}
```

---

## 5. 當前登入 Session

```
{{ current_sessions.stdout | default('無活躍 session') }}
```

---

## 審查確認記錄

| 審查者 | 職稱 | 確認日期 | 簽名 |
|--------|------|---------|------|
| | | | |

_本報告由 Ansible account_lifecycle role 自動產生。審查結果請存回 audit-evidence 倉庫。_
_產生時間：{{ ansible_date_time.iso8601 }}_
```

---

## 11.4 入職 / 離職 Playbook

```yaml
# playbooks/onboard_user.yml
---
# 使用方式：
# ansible-playbook playbooks/onboard_user.yml \
#   --vault-password-file ~/.vault_pass
# （會自動讀取 inventory/users/all_users.yml，
#   對 status=active 的使用者建立帳號）
- name: Provision user accounts (onboarding)
  hosts: all
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: account_lifecycle
      tags: [accounts, provision, ssh, sudo]
```

```yaml
# playbooks/offboard_user.yml
---
# 使用方式：
# 1. 先在 all_users.yml 中將使用者 status 改為 offboarded
# 2. git commit & push（觸發 CI/CD）
# 3. 或手動執行：
# ansible-playbook playbooks/offboard_user.yml \
#   --vault-password-file ~/.vault_pass
- name: Offboard departed users
  hosts: all
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: account_lifecycle
      tags: [accounts, offboard, ssh, sudo]
```

```yaml
# playbooks/access_review.yml
---
# 每季執行，產出存取審查報告
- name: Generate quarterly access review report
  hosts: all
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  tasks:
    - name: Load user data and generate access review
      ansible.builtin.include_role:
        name: account_lifecycle
        tasks_from: review_report
      tags: [accounts, review]

  post_tasks:
    - name: Commit access review to audit-evidence
      ansible.builtin.include_tasks: ../roles/evidence_collector/tasks/git_commit.yml
      vars:
        evidence_report_dir: "{{ access_review_report_dir }}"
      tags: [git]
```

---

## 11.5 GitLab CI/CD 整合

```yaml
# .gitlab-ci.yml 新增帳號管理相關 jobs

# 當 all_users.yml 有變更時自動觸發帳號同步
sync-user-accounts:
  stage: deploy
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/onboard_user.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
  rules:
    # 只在 users 檔案有變更時才執行
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - inventory/users/all_users.yml
      when: on_success

# 每季產出存取審查報告（3/6/9/12 月第一週一）
quarterly-access-review:
  stage: evidence-collect
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/access_review.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "quarterly_review"
```

> **Schedule 設定：** 在 GitLab CI/CD → Schedules 另建一個季度排程：
> `0 9 1 3,6,9,12 1`（每年 3、6、9、12 月第一個週一早上 9:00）

---

## 11.6 使用者目錄 YAML 範本

```yaml
# inventory/users/all_users.yml
---
users:
  # ── 在職員工 ─────────────────────────────────────────────────────────
  - username: alice
    full_name: "Alice Chen"
    email: "alice@example.com"
    role: senior-engineer
    status: active
    ssh_pubkeys:
      # 格式：公鑰全文，方便 Ansible 直接寫入 authorized_keys
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxx alice@macbook"
    sudo_access: true
    # 限制 sudo 只能執行指定指令（不設則允許所有）
    sudo_commands:
      - /usr/bin/docker
      - /usr/bin/systemctl restart nginx
    groups:
      - developers
      - docker
    joined_date: "2024-06-01"
    offboarded_date: null

  - username: bob
    full_name: "Bob Lin"
    email: "bob@example.com"
    role: developer
    status: active
    ssh_pubkeys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIyyyy bob@thinkpad"
    sudo_access: false
    groups:
      - developers
    joined_date: "2025-01-15"
    offboarded_date: null

  # ── 已離職員工（保留記錄，帳號已鎖定）──────────────────────────────
  - username: carol
    full_name: "Carol Wu"
    email: "carol@example.com"
    role: engineer
    status: offboarded
    ssh_pubkeys: []    # 離職後清空
    sudo_access: false
    groups: []
    joined_date: "2023-03-01"
    offboarded_date: "2026-02-28"
```

---

## 11.7 驗證步驟

```bash
# 1. 模擬入職：新增使用者到 all_users.yml，執行 onboard playbook
ansible-playbook playbooks/onboard_user.yml \
  --vault-password-file ~/.vault_pass \
  --check --diff   # 先 dry-run

ansible-playbook playbooks/onboard_user.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認帳號已建立
ansible all -m ansible.builtin.command -a "id alice"
# 預期：uid=1001(alice) gid=1001(alice) groups=...,docker

# 3. 確認 SSH 金鑰已部署
ansible all -m ansible.builtin.command \
  -a "cat /home/alice/.ssh/authorized_keys"

# 4. 確認 sudo 設定正確
ansible all -m ansible.builtin.command \
  -a "cat /etc/sudoers.d/alice"

# 5. 模擬離職：將 carol 的 status 改為 offboarded
# 編輯 inventory/users/all_users.yml
# git commit -m "offboard: carol 離職 2026-02-28"

ansible-playbook playbooks/offboard_user.yml \
  --vault-password-file ~/.vault_pass

# 6. 確認帳號已鎖定
ansible all -m ansible.builtin.command \
  -a "passwd -S carol"
# 預期：carol L (locked)

ansible all -m ansible.builtin.command \
  -a "cat /home/carol/.ssh/authorized_keys"
# 預期：cat: 檔案不存在（已清除）

# 7. 執行季度存取審查報告
ansible-playbook playbooks/access_review.yml \
  --vault-password-file ~/.vault_pass

cat access-review-reports/$(date +%Y-%m-%d)/web-01.md

# 8. 確認 git 記錄完整
git log --oneline -- inventory/users/all_users.yml
# 每次人員異動都有對應的 commit
```

---

## 11.8 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| YAML 使用者目錄作為 IaC 人員管理 | ☐ 人員異動以 git commit 記錄，可追蹤 |
| `ansible.builtin.user` 建立 / 鎖定帳號 | ☐ 離職帳號狀態為 L（locked）|
| `ansible.posix.authorized_key` 獨占模式 | ☐ exclusive=true 確保清單外金鑰自動移除 |
| sudoers 模板化管理 + validate | ☐ 語法錯誤不會被部署（validate 保護）|
| 季度存取審查報告產出 | ☐ 報告包含完整帳號清單供管理者簽核 |
| GitLab CI 自動同步（users 檔案變更觸發）| ☐ push all_users.yml → 自動執行帳號同步 |

**下一章：** [第 12 章 備份還原驗證與 DR 演練](./chapter-12-backup-dr-validation.md)

---

*← [返回總覽](./README.md) | [上一章](./chapter-10-ssl-certificate-management.md)*
