# 第 1 章：環境初始化與 Inventory 設計

> **學習目標：** 在 Ubuntu 24.04 建立完整的 Ansible 控制環境，掌握 Inventory 多環境管理，並初步認識 Ansible Vault 加密機制。

---

## 1.1 理論說明：為什麼 Inventory 設計至關重要？

Ansible 的核心哲學是「**Infrastructure as Code（IaC）**」——將基礎設施的狀態以程式碼描述，納入版本控制，使環境可重現、可審計。

### 1.1.1 Inventory 的本質

Inventory 是 Ansible 的「地圖」，定義了：

- **哪些主機**需要被管理（IP、FQDN）
- **主機隸屬於哪些群組**（webservers、dbservers）
- **每個群組或主機的變數**（連線帳號、port、環境特定參數）

不良的 Inventory 設計會導致：
- 正式環境與測試環境的變數混用（高風險）
- 機密資訊（密碼、Token）以明文存放（資安漏洞）
- 難以維護的 hard-coded 主機清單

### 1.1.2 靜態 vs. 動態 Inventory

| 類型 | 適用場景 | 說明 |
|------|----------|------|
| 靜態 Inventory | 固定 IP 的實體機、VM | YAML 或 INI 格式手寫 |
| 動態 Inventory | AWS EC2、GCP、vSphere | 透過 Plugin 即時查詢雲端 API |

本章以**靜態 Inventory** 為主，為後續章節奠定基礎。

---

## 1.2 安裝 Ansible 2.16+（Ubuntu 24.04）

Ubuntu 24.04 官方套件庫的 Ansible 版本較舊，建議使用 `pipx` 安裝，確保隔離性與版本可控。

### 1.2.1 安裝步驟

```bash
# Step 1：更新套件清單並安裝 pipx
sudo apt update && sudo apt install -y pipx python3-pip

# Step 2：確保 pipx 路徑在 PATH 中
pipx ensurepath
source ~/.bashrc  # 或重新開啟 terminal

# Step 3：安裝 Ansible（包含 ansible-core 2.16+）
pipx install ansible

# Step 4：安裝常用開發工具
pipx inject ansible ansible-lint
pipx inject ansible paramiko  # 備用 SSH 連線後端

# Step 5：驗證安裝版本
ansible --version
```

預期輸出：

```
ansible [core 2.16.x]
  config file = /etc/ansible/ansible.cfg
  configured module search path = [...]
  ansible python module location = ...
  ansible collection location = ~/.ansible/collections/...
  executable location = ~/.local/bin/ansible
  python version = 3.12.x
```

### 1.2.2 安裝必要 Galaxy Collections

```bash
# 安裝本教材所需的 Collection
ansible-galaxy collection install \
  ansible.posix \
  community.docker \
  community.general \
  community.crypto

# 驗證安裝
ansible-galaxy collection list
```

---

## 1.3 建立專案目錄結構

```bash
# 建立課程專案目錄
mkdir -p ~/ansible-course
cd ~/ansible-course

# 建立完整目錄樹
mkdir -p \
  inventory/production/group_vars \
  inventory/production/host_vars \
  inventory/staging/group_vars \
  inventory/staging/host_vars \
  roles \
  playbooks \
  vault
```

### 1.3.1 ansible.cfg 設定

`ansible.cfg` 是 Ansible 的全域設定檔，定義預設行為，避免每次指令都需要帶大量參數。

```bash
cat > ~/ansible-course/ansible.cfg << 'EOF'
[defaults]
# Inventory 預設路徑
inventory          = ./inventory/production

# Roles 搜尋路徑
roles_path         = ./roles

# 預設連線使用者（可被 host_vars 覆寫）
remote_user        = ubuntu

# SSH private key 路徑
private_key_file   = ~/.ssh/ansible_ed25519

# 平行執行的主機數量（fork 數）
forks              = 10

# 關閉 host key checking（僅限開發環境！正式環境應啟用）
host_key_checking  = False

# 顯示執行時間統計
callback_whitelist = timer, profile_tasks

# 日誌路徑（正式環境建議開啟）
#log_path          = ./ansible.log

# 提升 stdout 可讀性
stdout_callback    = yaml

[privilege_escalation]
# 預設使用 sudo 提升權限
become             = True
become_method      = sudo
become_user        = root
# 若目標主機的 sudo 需要密碼，設為 True 並執行時加 -K
become_ask_pass    = False

[ssh_connection]
# SSH 連線複用，大幅加速多主機操作
ssh_args           = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining         = True

[inventory]
# 讓 Ansible 忽略不符合格式的 inventory 檔案（避免 .swp 等暫存檔報錯）
ignore_extensions  = .pyc, .pyo, .swp, .bak, ~, .rpm, .md
EOF
```

---

## 1.4 Inventory 設計：多環境管理

### 1.4.1 目錄式 Inventory（推薦做法）

```
inventory/
├── production/           ← 正式環境
│   ├── hosts.yml         ← 主機清單
│   ├── group_vars/
│   │   ├── all.yml       ← 所有主機共用變數
│   │   ├── webservers.yml
│   │   └── dbservers.yml
│   └── host_vars/
│       ├── web-01.yml    ← web-01 專屬變數
│       └── db-01.yml
└── staging/              ← 測試環境
    ├── hosts.yml
    └── group_vars/
        └── all.yml
```

### 1.4.2 hosts.yml（YAML 格式）

```yaml
# inventory/production/hosts.yml
---
all:
  children:
    webservers:
      hosts:
        web-01:
          ansible_host: 192.168.56.11
        web-02:
          ansible_host: 192.168.56.12
      vars:
        # 群組層級的變數（可被 host_vars 覆寫）
        http_port: 80
        https_port: 443

    dbservers:
      hosts:
        db-01:
          ansible_host: 192.168.56.20
        db-02:
          ansible_host: 192.168.56.21
      vars:
        db_port: 5432

    # 巢狀群組：所有後端服務
    backend:
      children:
        webservers:
        dbservers:

    # GitLab 伺服器獨立分組
    gitlab_servers:
      hosts:
        gitlab-01:
          ansible_host: 192.168.56.30
```

### 1.4.3 group_vars/all.yml（共用變數）

```yaml
# inventory/production/group_vars/all.yml
---
# ── 連線設定 ────────────────────────────────
ansible_user: ubuntu
ansible_python_interpreter: /usr/bin/python3

# ── 環境標籤 ────────────────────────────────
env: production
datacenter: tw-tpe-01

# ── NTP 設定 ────────────────────────────────
ntp_servers:
  - time.google.com
  - time.cloudflare.com

# ── 套件鏡像來源 ────────────────────────────
apt_mirror: "http://tw.archive.ubuntu.com/ubuntu"

# ── 時區 ────────────────────────────────────
timezone: Asia/Taipei

# ── 系統管理員信箱 ───────────────────────────
admin_email: ops@example.com
```

### 1.4.4 group_vars/webservers.yml（群組專屬變數）

```yaml
# inventory/production/group_vars/webservers.yml
---
# Web 伺服器群組專屬設定
nginx_worker_processes: auto
nginx_worker_connections: 4096
nginx_keepalive_timeout: 65

# 日誌等級
nginx_log_level: warn

# 是否啟用 SSL
enable_ssl: true
ssl_cert_path: /etc/ssl/certs/server.crt
ssl_key_path: /etc/ssl/private/server.key
```

### 1.4.5 host_vars/web-01.yml（主機專屬變數）

```yaml
# inventory/production/host_vars/web-01.yml
---
# 此主機為主要入口，啟用額外功能
is_primary: true

# 覆寫群組變數的 worker_connections
nginx_worker_connections: 8192

# 此主機的 Floating IP（用於 HAProxy 或 VIP）
vip_address: 192.168.56.100
```

---

## 1.5 Ansible Vault：加密機密資料

### 1.5.1 為什麼需要 Vault？

在 Ansible 管理的基礎設施中，不可避免地需要處理機密資料：

- 資料庫密碼（`db_password`）
- GitLab 初始 root 密碼（`gitlab_initial_root_password`）
- API Token、SSL 私鑰
- SSH 密碼（若未使用金鑰認證）

**直接以明文存入 Git Repository 是嚴重的資安風險。** Ansible Vault 提供對稱加密（AES-256），讓機密資料可以安全納入版控。

### 1.5.2 建立 Vault 加密檔案

```bash
# 建立新的 Vault 加密檔案（互動式輸入密碼）
ansible-vault create vault/secrets.yml

# 輸入並確認 Vault 密碼後，編輯器會開啟，填入：
```

```yaml
# vault/secrets.yml（加密前的明文內容）
---
# ── 資料庫機密 ───────────────────────────────
vault_db_root_password: "SuperSecretDB#2026"
vault_db_app_password: "AppPassword@Secure"

# ── GitLab 機密 ──────────────────────────────
vault_gitlab_root_password: "GitLabR00t!Pass"
vault_gitlab_runner_token: "glrt-xxxxxxxxxxxxxxxxxx"

# ── SSH 相關 ─────────────────────────────────
vault_ssh_deploy_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  [私鑰內容]
  -----END OPENSSH PRIVATE KEY-----
```

### 1.5.3 Vault 常用指令

```bash
# 加密現有的明文檔案
ansible-vault encrypt vault/secrets.yml

# 解密（僅限本機操作，不建議長期解密狀態）
ansible-vault decrypt vault/secrets.yml

# 查看加密檔案內容（不解密到硬碟）
ansible-vault view vault/secrets.yml

# 編輯加密檔案（直接在加密狀態下編輯）
ansible-vault edit vault/secrets.yml

# 重新加密（更換 Vault 密碼）
ansible-vault rekey vault/secrets.yml

# 加密單一字串（適合嵌入 YAML 中）
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
```

### 1.5.4 在 group_vars 中使用 Vault 變數

最佳實踐是將明文變數與 Vault 加密變數**分開放在不同檔案**，但透過命名慣例關聯：

```
group_vars/
└── all/
    ├── vars.yml       ← 明文變數，引用 vault_ 前綴變數
    └── vault.yml      ← Vault 加密，所有 vault_ 前綴變數
```

```yaml
# group_vars/all/vars.yml（明文，可以安全 commit）
---
# 使用 vault_ 前綴的變數（實際值在 vault.yml 中加密）
db_root_password: "{{ vault_db_root_password }}"
db_app_password: "{{ vault_db_app_password }}"
gitlab_root_password: "{{ vault_gitlab_root_password }}"
```

```bash
# group_vars/all/vault.yml（使用 vault 加密此檔案）
ansible-vault encrypt group_vars/all/vault.yml
```

### 1.5.5 執行 Playbook 時提供 Vault 密碼

```bash
# 方法一：互動式輸入（適合本機開發）
ansible-playbook playbooks/site.yml --ask-vault-pass

# 方法二：密碼檔案（適合 CI/CD，密碼檔不納入版控）
echo "my-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook playbooks/site.yml --vault-password-file ~/.vault_pass

# 方法三：環境變數（適合 GitLab CI/CD）
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook playbooks/site.yml
```

> ⚠️ **注意：** 密碼檔案（`.vault_pass`）絕對不能納入 Git 版控，務必加入 `.gitignore`。

---

## 1.6 建立 Common Role（基礎初始化）

每台伺服器都應套用的基礎設定，統一放入 `common` Role。

### 1.6.1 Role 目錄結構

```bash
# 使用 ansible-galaxy 建立 Role 骨架
ansible-galaxy role init roles/common
```

生成的結構：

```
roles/common/
├── defaults/
│   └── main.yml      ← 預設變數（優先級最低，易被覆寫）
├── handlers/
│   └── main.yml      ← Handler（事件觸發的任務）
├── tasks/
│   └── main.yml      ← 主要任務列表
├── templates/
│   └── ...           ← Jinja2 模板檔案（.j2）
├── files/
│   └── ...           ← 靜態檔案
├── vars/
│   └── main.yml      ← 不易被覆寫的 Role 內部變數
└── meta/
    └── main.yml      ← Role 元資訊與依賴
```

### 1.6.2 defaults/main.yml

```yaml
# roles/common/defaults/main.yml
---
# ── 時區設定 ─────────────────────────────────
common_timezone: "Asia/Taipei"

# ── 要安裝的基礎套件 ──────────────────────────
common_packages:
  - curl
  - wget
  - vim
  - git
  - htop
  - iotop
  - net-tools
  - dnsutils
  - unzip
  - jq
  - python3-pip
  - ca-certificates
  - gnupg
  - lsb-release

# ── 要停用的服務 ──────────────────────────────
common_disabled_services:
  - snapd          # 在伺服器環境通常不需要
  - apport         # 崩潰報告服務，降低 overhead

# ── NTP 設定 ────────────────────────────────
common_ntp_servers:
  - time.google.com
  - time.cloudflare.com

# ── swap 設定 ────────────────────────────────
common_swappiness: 10  # 降低 swap 使用傾向（Server 最佳化）
```

### 1.6.3 tasks/main.yml

```yaml
# roles/common/tasks/main.yml
---
# ── 系統更新 ─────────────────────────────────────────────────────────────
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600  # 1小時內若已更新，跳過重複更新
  tags: [packages]

- name: Upgrade all packages to latest version
  ansible.builtin.apt:
    upgrade: dist
    autoremove: true
    autoclean: true
  tags: [packages]

# ── 安裝基礎套件 ─────────────────────────────────────────────────────────
- name: Install common packages
  ansible.builtin.apt:
    name: "{{ common_packages }}"
    state: present
  tags: [packages]

# ── 時區設定 ─────────────────────────────────────────────────────────────
- name: Set system timezone
  community.general.timezone:
    name: "{{ common_timezone }}"
  notify: Restart cron  # 更改時區後重啟 cron，避免排程時間錯誤
  tags: [timezone]

# ── NTP 時間同步 ─────────────────────────────────────────────────────────
- name: Ensure chrony is installed
  ansible.builtin.apt:
    name: chrony
    state: present
  tags: [ntp]

- name: Configure chrony NTP servers
  ansible.builtin.template:
    src: chrony.conf.j2
    dest: /etc/chrony/chrony.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart chrony
  tags: [ntp]

- name: Ensure chrony service is enabled and started
  ansible.builtin.service:
    name: chrony
    state: started
    enabled: true
  tags: [ntp]

# ── 停用不必要的服務 ─────────────────────────────────────────────────────
- name: Disable unnecessary services
  ansible.builtin.service:
    name: "{{ item }}"
    state: stopped
    enabled: false
  loop: "{{ common_disabled_services }}"
  # ignore_errors 確保服務不存在時不報錯
  ignore_errors: true
  tags: [services]

# ── Swap 調整 ────────────────────────────────────────────────────────────
- name: Set vm.swappiness to reduce swap usage
  ansible.posix.sysctl:
    name: vm.swappiness
    value: "{{ common_swappiness }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-common.conf
    reload: true
  tags: [kernel]

# ── 主機名稱設定 ─────────────────────────────────────────────────────────
- name: Set hostname
  ansible.builtin.hostname:
    name: "{{ inventory_hostname }}"
    use: systemd
  tags: [hostname]

- name: Update /etc/hosts with current hostname
  ansible.builtin.lineinfile:
    path: /etc/hosts
    regexp: "^127\\.0\\.1\\.1"
    line: "127.0.1.1 {{ inventory_hostname }}"
    state: present
  tags: [hostname]
```

### 1.6.4 handlers/main.yml

```yaml
# roles/common/handlers/main.yml
---
# Handler 只在被 notify 時執行，且在 play 結束時統一觸發一次（避免重複重啟）

- name: Restart cron
  ansible.builtin.service:
    name: cron
    state: restarted

- name: Restart chrony
  ansible.builtin.service:
    name: chrony
    state: restarted
```

### 1.6.5 templates/chrony.conf.j2

```jinja2
{# roles/common/templates/chrony.conf.j2 #}
# chrony.conf - 由 Ansible 管理，請勿手動修改
# 最後更新：{{ ansible_date_time.iso8601 }}

{% for server in common_ntp_servers %}
pool {{ server }} iburst
{% endfor %}

# 時鐘漂移補償檔案
driftfile /var/lib/chrony/drift

# 允許系統時鐘大幅調整（僅在服務啟動前3個更新）
makestep 1.0 3

# 啟用 RTC 時鐘同步
rtcsync

# 日誌目錄
logdir /var/log/chrony
```

---

## 1.7 第一個 Playbook：初始化所有主機

```yaml
# playbooks/site.yml
---
# ════════════════════════════════════════════════════
# site.yml - 主入口 Playbook
# 用途：對所有主機套用基礎設定
# 執行：ansible-playbook playbooks/site.yml
# ════════════════════════════════════════════════════

- name: Apply common configuration to all hosts
  hosts: all
  gather_facts: true   # 收集目標主機的系統資訊（OS、IP 等）

  # 引入 Vault 加密的機密變數
  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: common
      tags: [common]
```

---

## 1.8 Ad-hoc 指令：快速驗證

Ad-hoc 指令讓你無需寫 Playbook，直接對主機執行單一任務，非常適合快速除錯。

```bash
# 測試連線（所有主機 ping）
ansible all -m ansible.builtin.ping

# 查看所有主機的 OS 資訊
ansible all -m ansible.builtin.setup -a "filter=ansible_distribution*"

# 查看磁碟使用量
ansible webservers -m ansible.builtin.command -a "df -h"

# 在特定群組執行 shell 指令
ansible dbservers -m ansible.builtin.shell -a "systemctl status postgresql"

# 指定 inventory 檔案（覆寫 ansible.cfg 設定）
ansible all -i inventory/staging/hosts.yml -m ping

# 執行 playbook 並顯示詳細輸出
ansible-playbook playbooks/site.yml -v    # verbose
ansible-playbook playbooks/site.yml -vvv  # 超詳細（含 SSH debug）

# 模擬執行（Dry Run，不實際變更）
ansible-playbook playbooks/site.yml --check --diff
```

---

## 1.9 驗證步驟

執行完 common Role 後，透過以下指令確認配置生效：

```bash
# 1. 確認時區
ansible all -m ansible.builtin.command -a "timedatectl show --property=Timezone"
# 預期輸出：Timezone=Asia/Taipei

# 2. 確認 NTP 同步
ansible all -m ansible.builtin.command -a "chronyc tracking"
# 預期輸出包含 System time 和 Last offset

# 3. 確認 swappiness 值
ansible all -m ansible.builtin.command -a "cat /proc/sys/vm/swappiness"
# 預期輸出：10

# 4. 確認已安裝套件
ansible all -m ansible.builtin.package_facts
ansible all -m ansible.builtin.debug -a "msg={{ ansible_facts.packages['curl'] }}"

# 5. 確認 Idempotency（再跑一次，確認 changed=0）
ansible-playbook playbooks/site.yml
# 預期：所有 task 顯示 ok，changed=0
```

---

## 1.10 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| 安裝 Ansible 2.16+ via pipx | ☐ 可執行 `ansible --version` 顯示正確版本 |
| 建立多環境 Inventory 結構 | ☐ production/staging 分開，變數繼承正確 |
| group_vars / host_vars 應用 | ☐ 群組變數可被主機變數覆寫 |
| Ansible Vault 加密機密 | ☐ 可建立加密檔，並在 playbook 中正確引用 |
| Common Role 實作 | ☐ 套用 common role 後所有主機時區、NTP、套件一致 |
| Ad-hoc 指令熟悉 | ☐ 可快速查詢主機狀態，無需寫 Playbook |

**下一章：** [第 2 章 Docker 與 GitLab 容器化部署](./chapter-02-docker-gitlab.md) — 我們將使用 Ansible 自動化部署 GitLab CE 與 CI/CD 環境。

---

*← [返回總覽](./README.md)*
