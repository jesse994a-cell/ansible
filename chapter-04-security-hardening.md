# 第 4 章：Ubuntu 系統資安強化（Security Hardening）

> **學習目標：** 使用 Ansible 系統化地強化 Ubuntu 24.04 的安全基線，涵蓋 SSH 強化、UFW 防火牆、Fail2Ban、自動安全更新，並深度整合 Ansible Vault 管理所有機密資料。

---

## 4.1 理論說明：為什麼需要系統資安強化？

Ubuntu 24.04 的預設安裝是「通用型」設定，在安全與易用性之間取平衡。作為伺服器，需要進一步收緊：

### 4.1.1 常見攻擊面

```
┌──────────────────────────────────────────────────────────┐
│                   典型攻擊路徑                            │
│                                                          │
│  Internet                                                │
│     │                                                    │
│     ├── SSH Brute Force ──────► Port 22 (weak password) │
│     │                                                    │
│     ├── Port Scanning ─────────► 開放不必要的服務埠      │
│     │                                                    │
│     ├── Exploit known CVEs ────► 未更新的系統套件         │
│     │                                                    │
│     └── Privilege Escalation ──► sudo 設定過於寬鬆       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 4.1.2 強化原則

1. **最小攻擊面（Minimize Attack Surface）：** 關閉所有非必要的服務與埠號
2. **最小權限（Least Privilege）：** 每個服務使用獨立的低權限帳號
3. **縱深防禦（Defense in Depth）：** 多層防護（SSH 金鑰 + UFW + Fail2Ban）
4. **自動化合規（Compliance as Code）：** 將資安政策以 Ansible 程式碼表達，可審計可重現

---

## 4.2 Ansible Role 結構：security_hardening

```bash
ansible-galaxy role init roles/security_hardening
```

```
roles/security_hardening/
├── defaults/
│   └── main.yml              ← 所有安全設定的可調整預設值
├── handlers/
│   └── main.yml              ← 重啟 SSH、UFW reload 等
├── tasks/
│   ├── main.yml              ← 任務入口
│   ├── ssh.yml               ← SSH 伺服器強化
│   ├── ufw.yml               ← UFW 防火牆規則
│   ├── fail2ban.yml          ← Fail2Ban 入侵防禦
│   ├── auto_updates.yml      ← 自動安全更新
│   ├── users.yml             ← 使用者與 sudo 管理
│   └── audit.yml             ← auditd 安全審計
├── templates/
│   ├── sshd_config.j2        ← SSH 伺服器設定
│   ├── fail2ban_jail.j2      ← Fail2Ban 規則
│   └── 50unattended-upgrades.j2
└── files/
    └── sudoers_hardened      ← 強化的 sudoers 片段
```

---

## 4.3 程式碼實作

### 4.3.1 defaults/main.yml

```yaml
# roles/security_hardening/defaults/main.yml
---
# ════════════════════════════════════════════════════════════════
# SSH 強化設定
# ════════════════════════════════════════════════════════════════

# SSH 監聽埠號（更改預設 22 以減少 Brute Force 攻擊）
ssh_port: 22  # 正式環境建議改為非標準埠，如 2222

# 禁止 root 直接 SSH 登入（強制透過一般帳號 + sudo）
ssh_permit_root_login: "no"

# 禁止密碼登入，強制使用金鑰認證
ssh_password_authentication: "no"

# 強制使用 Ed25519 或 ECDSA（禁用弱演算法）
ssh_allow_algorithms:
  kex:
    - curve25519-sha256
    - curve25519-sha256@libssh.org
    - ecdh-sha2-nistp256
    - ecdh-sha2-nistp384
  cipher:
    - chacha20-poly1305@openssh.com
    - aes256-gcm@openssh.com
    - aes128-gcm@openssh.com
    - aes256-ctr
  mac:
    - hmac-sha2-256-etm@openssh.com
    - hmac-sha2-512-etm@openssh.com
    - umac-128-etm@openssh.com

# 連線逾時設定
ssh_client_alive_interval: 300     # 每 5 分鐘探測一次
ssh_client_alive_count_max: 2      # 2 次無回應則斷線（等同 10 分鐘逾時）

# 最大認證嘗試次數（防止 Brute Force）
ssh_max_auth_tries: 3

# 限制可 SSH 登入的使用者或群組
ssh_allow_users: []     # 空=不限制（可設定為 ["ubuntu", "deploy"]）
ssh_allow_groups:
  - sudo              # 只允許 sudo 群組的使用者登入

# 禁用 SSH 的危險功能
ssh_x11_forwarding: "no"
ssh_allow_tcp_forwarding: "no"
ssh_gateway_ports: "no"
ssh_permit_tunnel: "no"

# ════════════════════════════════════════════════════════════════
# UFW 防火牆
# ════════════════════════════════════════════════════════════════

# 預設政策
ufw_default_incoming: deny    # 拒絕所有入站（再逐一允許需要的埠）
ufw_default_outgoing: allow   # 允許所有出站

# 允許的入站規則
ufw_allow_rules:
  - { port: "{{ ssh_port }}", proto: tcp, comment: "SSH" }
  - { port: 80, proto: tcp, comment: "HTTP" }
  - { port: 443, proto: tcp, comment: "HTTPS" }

# ════════════════════════════════════════════════════════════════
# Fail2Ban
# ════════════════════════════════════════════════════════════════

# 封鎖時間（秒）：-1 = 永久封鎖
fail2ban_bantime: 3600       # 封鎖 1 小時

# 觀察時間視窗（秒）
fail2ban_findtime: 600       # 10 分鐘內

# 允許失敗次數上限
fail2ban_maxretry: 3

# 白名單 IP（不會被封鎖）
fail2ban_ignoreip:
  - "127.0.0.1/8"
  - "::1"
  - "192.168.56.0/24"  # 內部網段

# ════════════════════════════════════════════════════════════════
# 自動更新
# ════════════════════════════════════════════════════════════════

# 是否啟用自動套用安全更新
auto_updates_enabled: true

# 更新後是否自動重開機（僅在需要 kernel 更新時）
auto_updates_reboot: false

# 自動重開機時間（若 auto_updates_reboot=true）
auto_updates_reboot_time: "02:00"

# ════════════════════════════════════════════════════════════════
# 使用者管理
# ════════════════════════════════════════════════════════════════

# 部署帳號（Ansible 使用的帳號）
deploy_user: ubuntu
deploy_user_groups:
  - sudo

# 要移除的危險帳號
remove_users:
  - games
  - news
  - uucp

# ════════════════════════════════════════════════════════════════
# Ansible Vault 管理的機密（變數名以 vault_ 前綴）
# ════════════════════════════════════════════════════════════════

# 部署帳號的 authorized_keys（公鑰從 Vault 注入）
deploy_user_ssh_pubkeys: "{{ vault_deploy_ssh_pubkeys | default([]) }}"
```

### 4.3.2 tasks/main.yml

```yaml
# roles/security_hardening/tasks/main.yml
---
- name: Harden user accounts
  ansible.builtin.import_tasks: users.yml
  tags: [hardening, users]

- name: Harden SSH server
  ansible.builtin.import_tasks: ssh.yml
  tags: [hardening, ssh]

- name: Configure UFW firewall
  ansible.builtin.import_tasks: ufw.yml
  tags: [hardening, ufw]

- name: Configure Fail2Ban
  ansible.builtin.import_tasks: fail2ban.yml
  tags: [hardening, fail2ban]

- name: Configure automatic security updates
  ansible.builtin.import_tasks: auto_updates.yml
  tags: [hardening, updates]

- name: Configure auditd security logging
  ansible.builtin.import_tasks: audit.yml
  tags: [hardening, audit]
```

### 4.3.3 tasks/ssh.yml

```yaml
# roles/security_hardening/tasks/ssh.yml
---
# ── 確保 openssh-server 已安裝 ──────────────────────────────────────────
- name: Install openssh-server
  ansible.builtin.apt:
    name: openssh-server
    state: present
    update_cache: true

# ── 部署 SSH 公鑰（透過 Vault 管理）────────────────────────────────────
- name: Deploy authorized SSH public keys for deploy user
  ansible.posix.authorized_key:
    user: "{{ deploy_user }}"
    key: "{{ item }}"
    # exclusive=true：移除所有未在此列表中的金鑰（防止後門）
    exclusive: false
    state: present
  loop: "{{ deploy_user_ssh_pubkeys }}"
  when: deploy_user_ssh_pubkeys | length > 0
  no_log: true  # 不在 Ansible 輸出中顯示公鑰內容

# ── 套用強化的 sshd_config ──────────────────────────────────────────────
- name: Apply hardened sshd_config from template
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: '0600'
    validate: "/usr/sbin/sshd -t -f %s"  # 部署前先驗證語法，防止鎖死 SSH
  notify: Restart SSH

# ── 確保 sshd service 正在運行 ──────────────────────────────────────────
- name: Ensure SSH service is enabled and running
  ansible.builtin.service:
    name: ssh
    state: started
    enabled: true

# ── 驗證 SSH 金鑰認證可以正常連線（保護機制）────────────────────────────
# 注意：應在套用新設定後立即測試連線，確保不會鎖死
- name: Verify SSH connection test (check mode only)
  ansible.builtin.wait_for:
    host: "{{ ansible_host }}"
    port: "{{ ssh_port }}"
    timeout: 10
    state: started
  changed_when: false
  delegate_to: localhost  # 從 Control Node 測試
```

### 4.3.4 templates/sshd_config.j2

```jinja2
{# roles/security_hardening/templates/sshd_config.j2 #}
# /etc/ssh/sshd_config
# 由 Ansible 管理 — 請勿手動修改
# 最後更新：{{ ansible_date_time.iso8601 }} by {{ ansible_user_id }}

# ── 基本設定 ─────────────────────────────────────────────────────
Port {{ ssh_port }}
AddressFamily inet
ListenAddress 0.0.0.0

# ── 認證設定 ─────────────────────────────────────────────────────
# 禁止 root 直接登入
PermitRootLogin {{ ssh_permit_root_login }}

# 強制金鑰認證，禁止密碼登入
PasswordAuthentication {{ ssh_password_authentication }}
PubkeyAuthentication yes

# 禁用其他認證方式
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

# 最大認證嘗試次數
MaxAuthTries {{ ssh_max_auth_tries }}

# ── 加密演算法（現代化安全標準）──────────────────────────────────
# 金鑰交換演算法（移除 DH group1/group14，保留橢圓曲線）
KexAlgorithms {{ ssh_allow_algorithms.kex | join(',') }}

# 加密演算法（移除弱 CBC 模式）
Ciphers {{ ssh_allow_algorithms.cipher | join(',') }}

# MAC 演算法（使用 ETM = Encrypt-then-MAC）
MACs {{ ssh_allow_algorithms.mac | join(',') }}

# Host Key 演算法
HostKeyAlgorithms ssh-ed25519,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,rsa-sha2-512,rsa-sha2-256

# ── 連線控制 ─────────────────────────────────────────────────────
# 連線逾時探測（偵測死連線）
ClientAliveInterval {{ ssh_client_alive_interval }}
ClientAliveCountMax {{ ssh_client_alive_count_max }}

# 登入逾時（未完成認證則斷開）
LoginGraceTime 30

# 最大並發連線數（尚未認證）
MaxStartups 10:30:60

# 最大 session 數（每個連線）
MaxSessions 4

{% if ssh_allow_groups | length > 0 %}
# 限制可登入的群組
AllowGroups {{ ssh_allow_groups | join(' ') }}
{% endif %}

{% if ssh_allow_users | length > 0 %}
# 限制可登入的使用者
AllowUsers {{ ssh_allow_users | join(' ') }}
{% endif %}

# ── 危險功能禁用 ──────────────────────────────────────────────────
# 禁止 X11 轉發
X11Forwarding {{ ssh_x11_forwarding }}

# 禁止 TCP 轉發（Port Forwarding）
AllowTcpForwarding {{ ssh_allow_tcp_forwarding }}

# 禁止 Agent 轉發
AllowAgentForwarding no

# 禁止 GatewayPorts
GatewayPorts {{ ssh_gateway_ports }}

# 禁止 TUN 隧道
PermitTunnel {{ ssh_permit_tunnel }}

# 禁止使用者設定 authorized_keys 的替代路徑（防止繞過）
AuthorizedKeysFile .ssh/authorized_keys

# ── 日誌設定 ─────────────────────────────────────────────────────
# 詳細記錄（含認證資訊）
LogLevel VERBOSE
SyslogFacility AUTH

# ── SFTP 設定 ─────────────────────────────────────────────────────
Subsystem sftp /usr/lib/openssh/sftp-server
```

### 4.3.5 tasks/ufw.yml

```yaml
# roles/security_hardening/tasks/ufw.yml
---
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present

# ── 重置為預設政策（確保乾淨狀態）──────────────────────────────────────
- name: Reset UFW to defaults (disable first to avoid lockout)
  community.general.ufw:
    state: reset

# ── 設定預設政策 ─────────────────────────────────────────────────────────
- name: Set UFW default incoming policy to deny
  community.general.ufw:
    default: deny
    direction: incoming

- name: Set UFW default outgoing policy to allow
  community.general.ufw:
    default: allow
    direction: outgoing

# ── 設定允許的入站規則 ───────────────────────────────────────────────────
- name: Allow specific ports
  community.general.ufw:
    rule: allow
    port: "{{ item.port | string }}"
    proto: "{{ item.proto }}"
    comment: "{{ item.comment }}"
  loop: "{{ ufw_allow_rules }}"

# ── 允許已建立的連線（避免中斷現有 SSH）────────────────────────────────
- name: Allow established connections
  community.general.ufw:
    rule: allow
    proto: tcp
    from_ip: any
    to_ip: any
    # UFW 預設已包含此規則，此步驟為明確確認

# ── 限制 SSH（對同一 IP 有速率限制，防 Brute Force）─────────────────────
- name: Rate-limit SSH connections
  community.general.ufw:
    rule: limit
    port: "{{ ssh_port | string }}"
    proto: tcp
    comment: "SSH rate limit"

# ── 啟用 UFW ─────────────────────────────────────────────────────────────
- name: Enable UFW
  community.general.ufw:
    state: enabled
  # 注意：啟用 UFW 前必須確保 SSH 埠已在規則中，否則會鎖死連線

# ── 確認 UFW 狀態 ────────────────────────────────────────────────────────
- name: Show UFW status
  ansible.builtin.command:
    cmd: ufw status verbose
  register: ufw_status
  changed_when: false

- name: Display UFW rules
  ansible.builtin.debug:
    var: ufw_status.stdout_lines
```

### 4.3.6 tasks/fail2ban.yml

```yaml
# roles/security_hardening/tasks/fail2ban.yml
---
- name: Install Fail2Ban
  ansible.builtin.apt:
    name:
      - fail2ban
      - python3-systemd  # 讓 Fail2Ban 讀取 systemd journal
    state: present

# ── 部署 jail 設定（使用 jail.local 覆寫，不修改 jail.conf）──────────
- name: Deploy Fail2Ban jail configuration
  ansible.builtin.template:
    src: fail2ban_jail.j2
    dest: /etc/fail2ban/jail.local
    owner: root
    group: root
    mode: '0644'
  notify: Restart Fail2Ban

# ── 確保 Fail2Ban 服務啟動 ──────────────────────────────────────────────
- name: Enable and start Fail2Ban
  ansible.builtin.service:
    name: fail2ban
    state: started
    enabled: true
```

### 4.3.7 templates/fail2ban_jail.j2

```jinja2
{# roles/security_hardening/templates/fail2ban_jail.j2 #}
# /etc/fail2ban/jail.local
# 由 Ansible 管理 — 請勿手動修改

[DEFAULT]
# 白名單 IP（不會被封鎖）
ignoreip = {{ fail2ban_ignoreip | join(' ') }}

# 封鎖時間（-1 = 永久）
bantime  = {{ fail2ban_bantime }}

# 觀察時間視窗
findtime = {{ fail2ban_findtime }}

# 最大失敗次數
maxretry = {{ fail2ban_maxretry }}

# 使用 systemd journal 作為後端（適合 Ubuntu 24.04）
backend = systemd

# 封鎖動作：使用 UFW
banaction = ufw
banaction_allports = ufw

# 傳送 email 通知（選用，需設定 sendmail）
# destemail = {{ admin_email | default('root@localhost') }}
# sender = fail2ban@{{ inventory_hostname }}
# action = %(action_mwl)s

[sshd]
enabled  = true
port     = {{ ssh_port }}
filter   = sshd
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400  # SSH 封鎖 24 小時（比預設更嚴格）

[nginx-http-auth]
enabled  = false  # 若部署 Nginx 可啟用

[nginx-limit-req]
enabled  = false

[nginx-botsearch]
enabled  = false
```

### 4.3.8 tasks/auto_updates.yml

```yaml
# roles/security_hardening/tasks/auto_updates.yml
---
- name: Install unattended-upgrades
  ansible.builtin.apt:
    name:
      - unattended-upgrades
      - apt-listchanges
    state: present

- name: Configure unattended-upgrades
  ansible.builtin.template:
    src: 50unattended-upgrades.j2
    dest: /etc/apt/apt.conf.d/50unattended-upgrades
    owner: root
    group: root
    mode: '0644'

# 啟用自動更新的觸發條件
- name: Configure auto-upgrade trigger
  ansible.builtin.copy:
    content: |
      APT::Periodic::Update-Package-Lists "1";
      APT::Periodic::Unattended-Upgrade "1";
      APT::Periodic::AutocleanInterval "7";
    dest: /etc/apt/apt.conf.d/20auto-upgrades
    mode: '0644'

- name: Enable unattended-upgrades service
  ansible.builtin.service:
    name: unattended-upgrades
    state: started
    enabled: "{{ auto_updates_enabled }}"
```

### 4.3.9 templates/50unattended-upgrades.j2

```jinja2
{# roles/security_hardening/templates/50unattended-upgrades.j2 #}
// /etc/apt/apt.conf.d/50unattended-upgrades
// 由 Ansible 管理

Unattended-Upgrade::Allowed-Origins {
    // Ubuntu 安全更新（必須啟用）
    "${distro_id}:${distro_codename}-security";
    // Ubuntu ESM（Extended Security Maintenance）
    "Ubuntu ESM Apps:${distro_codename}-apps-security";
};

// 排除不自動更新的套件（避免版本升級導致服務中斷）
Unattended-Upgrade::Package-Blacklist {
    // "docker-ce";
    // "docker-ce-cli";
};

// 更新後自動移除不需要的套件
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 更新失敗時傳送 email
Unattended-Upgrade::Mail "{{ admin_email | default('root') }}";
Unattended-Upgrade::MailReport "on-change";

// 是否允許自動重開機（僅在 kernel 更新需要時）
Unattended-Upgrade::Automatic-Reboot "{{ auto_updates_reboot | lower }}";
Unattended-Upgrade::Automatic-Reboot-Time "{{ auto_updates_reboot_time }}";

// 詳細日誌
Unattended-Upgrade::Verbose "true";
Unattended-Upgrade::Debug "false";
```

### 4.3.10 tasks/users.yml

```yaml
# roles/security_hardening/tasks/users.yml
---
# ── 移除不必要的系統帳號 ─────────────────────────────────────────────────
- name: Remove unnecessary system accounts
  ansible.builtin.user:
    name: "{{ item }}"
    state: absent
    remove: true
  loop: "{{ remove_users }}"
  ignore_errors: true

# ── 確保 deploy 使用者存在且在正確群組 ──────────────────────────────────
- name: Ensure deploy user exists with correct groups
  ansible.builtin.user:
    name: "{{ deploy_user }}"
    groups: "{{ deploy_user_groups }}"
    append: true
    shell: /bin/bash
    state: present

# ── 強化 sudo 設定 ───────────────────────────────────────────────────────
- name: Ensure sudoers.d directory exists
  ansible.builtin.file:
    path: /etc/sudoers.d
    state: directory
    mode: '0750'

- name: Require password for sudo (disable NOPASSWD for deploy user)
  ansible.builtin.copy:
    content: |
      # Require sudo password for deploy user
      # Generated by Ansible - do not edit manually
      Defaults:{{ deploy_user }} timestamp_timeout=5
    dest: /etc/sudoers.d/deploy_hardened
    owner: root
    group: root
    mode: '0440'
    validate: "/usr/sbin/visudo -cf %s"  # 驗證語法，防止鎖死 sudo

# ── 確保密碼政策設定 ─────────────────────────────────────────────────────
- name: Configure PAM password policy
  ansible.builtin.apt:
    name: libpam-pwquality
    state: present

- name: Set password quality requirements
  ansible.builtin.lineinfile:
    path: /etc/security/pwquality.conf
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: '^#?minlen',   line: 'minlen = 14' }
    - { regexp: '^#?dcredit',  line: 'dcredit = -1' }
    - { regexp: '^#?ucredit',  line: 'ucredit = -1' }
    - { regexp: '^#?lcredit',  line: 'lcredit = -1' }
    - { regexp: '^#?ocredit',  line: 'ocredit = -1' }
```

### 4.3.11 tasks/audit.yml

```yaml
# roles/security_hardening/tasks/audit.yml
---
- name: Install auditd
  ansible.builtin.apt:
    name:
      - auditd
      - audispd-plugins
    state: present

- name: Configure audit rules for security events
  ansible.builtin.copy:
    content: |
      # /etc/audit/rules.d/99-hardening.rules
      # 由 Ansible 管理

      # 監控 /etc/passwd 和 /etc/shadow 的修改
      -w /etc/passwd -p wa -k user_modification
      -w /etc/shadow -p wa -k user_modification
      -w /etc/group -p wa -k user_modification
      -w /etc/sudoers -p wa -k sudo_modification
      -w /etc/sudoers.d/ -p wa -k sudo_modification

      # 監控 SSH 設定修改
      -w /etc/ssh/sshd_config -p wa -k ssh_config

      # 監控認證相關事件
      -w /var/log/auth.log -p wa -k auth_log

      # 監控 sysctl 修改
      -w /etc/sysctl.conf -p wa -k sysctl_change
      -w /etc/sysctl.d/ -p wa -k sysctl_change

      # 監控特權指令使用
      -a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

      # 確保 audit 規則不可刪除（需重開機才能移除）
      -e 2
    dest: /etc/audit/rules.d/99-hardening.rules
    owner: root
    group: root
    mode: '0640'
  notify: Restart auditd

- name: Enable and start auditd
  ansible.builtin.service:
    name: auditd
    state: started
    enabled: true
```

### 4.3.12 handlers/main.yml

```yaml
# roles/security_hardening/handlers/main.yml
---
- name: Restart SSH
  ansible.builtin.service:
    name: ssh
    state: restarted

- name: Reload UFW
  community.general.ufw:
    state: reloaded

- name: Restart Fail2Ban
  ansible.builtin.service:
    name: fail2ban
    state: restarted

- name: Restart auditd
  ansible.builtin.service:
    name: auditd
    state: restarted
```

---

## 4.4 Ansible Vault 深度應用：SSH 金鑰管理

### 4.4.1 問題場景

在企業環境，SSH 公鑰也需要版本控制，但私鑰絕對不能明文儲存。以下示範如何用 Vault 管理 authorized_keys 的內容：

```bash
# Step 1：產生 Ed25519 金鑰對（Deploy 帳號專用）
ssh-keygen -t ed25519 -C "ansible-deploy@$(hostname)" -f ~/.ssh/ansible_ed25519

# Step 2：查看公鑰
cat ~/.ssh/ansible_ed25519.pub
# 輸出：ssh-ed25519 AAAAC3Nz... ansible-deploy@control-node

# Step 3：將公鑰加入 Vault
ansible-vault edit vault/secrets.yml
```

在 vault/secrets.yml 中加入：

```yaml
# vault/secrets.yml（加密前內容）
vault_deploy_ssh_pubkeys:
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxx ansible-deploy@control-node"
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIyyyyyyyyyyyyyyyyyy ops-team@laptop"
```

### 4.4.2 Vault 在 CI/CD 環境的最佳實踐

```
┌────────────────────────────────────────────────────────────────┐
│                  Vault 密碼管理矩陣                             │
│                                                                │
│  環境        │  Vault 密碼儲存位置         │  存取方式          │
│  ─────────── │  ──────────────────────── │  ──────────────── │
│  本機開發    │  ~/.vault_pass（gitignore）│  --vault-pw-file  │
│  CI/CD       │  GitLab CI Variable        │  環境變數          │
│  正式環境    │  HashiCorp Vault / AWS KMS │  動態取得          │
└────────────────────────────────────────────────────────────────┘
```

```bash
# GitLab CI/CD 中的 Vault 整合
# 在 .gitlab-ci.yml 中：
deploy:
  script:
    # 從 GitLab CI Variable 建立 vault_pass 檔
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/.vault_pass
    - chmod 600 /tmp/.vault_pass
    - ansible-playbook playbooks/security_hardening.yml \
        --vault-password-file /tmp/.vault_pass
    # 清理（使用 trap 確保即使失敗也會清除）
    - rm -f /tmp/.vault_pass
```

---

## 4.5 完整 Playbook

```yaml
# playbooks/security_hardening.yml
---
- name: Apply security hardening to all servers
  hosts: all
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  # 安全強化的執行順序很重要：
  # 1. 先確保 SSH 金鑰可用，才能禁用密碼登入
  # 2. 先設定 UFW 規則，才能啟用防火牆
  pre_tasks:
    - name: Verify SSH key authentication works before disabling passwords
      ansible.builtin.ping:
      register: ping_test
      failed_when: ping_test is failed

    - name: Safety check - ensure we can connect before hardening
      ansible.builtin.debug:
        msg: "SSH 連線正常，可以安全執行強化步驟"

  roles:
    - role: security_hardening
      tags: [hardening]

  post_tasks:
    - name: Final verification - ensure services are running
      ansible.builtin.service_facts:

    - name: Verify critical services are running
      ansible.builtin.assert:
        that:
          - ansible_facts.services['ssh.service'].state == 'running'
          - ansible_facts.services['ufw.service'].state == 'running'
          - ansible_facts.services['fail2ban.service'].state == 'running'
        fail_msg: "某些關鍵服務未在運行！請立即檢查。"
        success_msg: "所有安全服務運行正常。"
```

---

## 4.6 驗證步驟

```bash
# 1. 確認 SSH 設定語法正確
ansible all -m ansible.builtin.command -a "sshd -t"
# 預期：無輸出（語法正確）

# 2. 確認 root 登入已禁用
ansible all -m ansible.builtin.command \
  -a "grep 'PermitRootLogin' /etc/ssh/sshd_config"
# 預期：PermitRootLogin no

# 3. 確認密碼認證已禁用
ansible all -m ansible.builtin.command \
  -a "grep 'PasswordAuthentication' /etc/ssh/sshd_config"
# 預期：PasswordAuthentication no

# 4. 確認 UFW 狀態
ansible all -m ansible.builtin.command -a "ufw status numbered"
# 預期：看到 SSH(22)、HTTP(80)、HTTPS(443) 的 ALLOW 規則

# 5. 確認 Fail2Ban 狀態
ansible all -m ansible.builtin.command -a "fail2ban-client status"
# 預期：看到 sshd jail 已啟動

# 6. 測試 Fail2Ban 是否運作（手動觸發）
# 在 Control Node 上：
ssh -o PasswordAuthentication=yes wrong-user@<target-ip>
# 重複 3 次後，你的 IP 應該被封鎖 5 分鐘
fail2ban-client status sshd
# 應看到 Banned IP list 中有你的 IP

# 解除封鎖
ansible all -m ansible.builtin.command \
  -a "fail2ban-client set sshd unbanip <your-ip>"

# 7. 確認自動更新設定
ansible all -m ansible.builtin.command \
  -a "unattended-upgrade --dry-run --debug 2>&1 | head -20"

# 8. 確認 auditd 正在記錄
ansible all -m ansible.builtin.command \
  -a "auditctl -l"
# 預期：看到 -w /etc/passwd -p wa -k user_modification 等規則

# 9. 測試強化後的 SSH 連線（在本機測試）
ssh -i ~/.ssh/ansible_ed25519 ubuntu@<target-ip>
# 應成功連線

# 10. 測試密碼登入已被拒絕
ssh -o PasswordAuthentication=yes ubuntu@<target-ip>
# 應看到：Permission denied (publickey)
```

---

## 4.7 資安合規檢查：ansible-audit 自動掃描

```bash
# 安裝 ansible-hardening 的審計工具（基於 OpenSCAP）
pip install ansible-hardening

# 或使用 Lynis（系統安全審計工具）
ansible all -m ansible.builtin.apt \
  -a "name=lynis state=present"

ansible all -m ansible.builtin.command \
  -a "lynis audit system --quiet --no-log" \
  > lynis_report.txt 2>&1
```

---

## 4.8 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| SSH 強化（金鑰、演算法、禁 root）| ☐ 密碼登入被拒，金鑰登入正常 |
| UFW 防火牆規則設定 | ☐ 只有指定埠號開放，其餘全拒 |
| Fail2Ban 自動封鎖 | ☐ 多次失敗 SSH 後 IP 被封鎖 |
| 自動安全更新 | ☐ unattended-upgrades 服務正在運行 |
| Vault 管理 SSH 公鑰 | ☐ 公鑰透過 Vault 注入，不以明文出現在 Git |
| sshd_config validate 安全機制 | ☐ 理解 `validate` 如何防止設定錯誤鎖死伺服器 |

**下一章：** [第 5 章 工程化與 CI Integration](./chapter-05-engineering-ci.md) — 完成端到端的自動化流程：Lint、Molecule 測試、GitLab CI 自動派送。

---

*← [返回總覽](./README.md) | [上一章](./chapter-03-kernel-tuning.md)*
