# 第 16 章：密碼與機密輪換（HashiCorp Vault）

> **工具定位：** HashiCorp Vault 是獨立的機密管理平台。**Ansible 負責部署 Vault、初始化、設定 Policy，並撰寫 Playbook 透過 Vault API 執行自動化密碼輪換**（資料庫密碼、SSH CA 金鑰、API Token）。
>
> **ISO 27001 對應：** A.5.17（鑑別資訊的使用）— 機密必須定期輪換，且有存取記錄。

---

## 16.1 理論說明：Ansible Vault vs HashiCorp Vault

這兩個工具名稱相似，功能完全不同：

| | Ansible Vault | HashiCorp Vault |
|--|--------------|----------------|
| 用途 | 加密 Ansible 變數檔案 | 動態機密管理平台 |
| 運作方式 | 靜態加密（AES-256）| 動態產生、短期有效的機密 |
| 適合場景 | 小型環境，機密數量少 | 中大型環境，需要機密輪換、稽核日誌 |
| 稽核能力 | 有變更記錄（git）| 完整的存取日誌 + 動態租約 |

**兩者可以並存**：Ansible Vault 儲存 Vault 的 Root Token，HashiCorp Vault 負責其他所有機密的動態輪換。

### 16.1.1 動態機密的概念

```
傳統做法（靜態密碼）：
  DB_PASSWORD = "mypassword"  ← 永遠不變，一旦洩漏就完了

HashiCorp Vault 動態機密：
  應用程式 → 向 Vault 請求 DB 帳號
  Vault → 在 PostgreSQL 動態建立新帳號（有效期 1 小時）
  PostgreSQL → 1 小時後帳號自動失效
  → 就算攻擊者取得帳號，1 小時後也無效了
```

---

## 16.2 Ansible Role：hashicorp_vault

```bash
ansible-galaxy role init roles/hashicorp_vault
```

### 16.2.1 defaults/main.yml

```yaml
# roles/hashicorp_vault/defaults/main.yml
---
vault_version: "1.17.0"
vault_data_path: /opt/vault
vault_config_path: /etc/vault.d
vault_port: 8200
vault_network: vault_net

# Vault 初始化設定（Shamir Secret Sharing）
vault_key_shares: 5       # 分成 5 份金鑰
vault_key_threshold: 3    # 需要 3 份才能 unseal

# 資料庫動態機密設定
vault_db_lease_duration: "1h"      # 動態帳號有效期
vault_db_max_lease_duration: "24h"

# 密碼輪換 Policy（靜態機密輪換間隔）
vault_rotation_period: "720h"  # 30 天
```

### 16.2.2 tasks/main.yml

```yaml
# roles/hashicorp_vault/tasks/main.yml
---
- name: Create Vault directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0750'
  loop:
    - "{{ vault_data_path }}"
    - "{{ vault_config_path }}"

- name: Deploy Vault configuration
  ansible.builtin.template:
    src: vault-config.hcl.j2
    dest: "{{ vault_config_path }}/vault.hcl"
    mode: '0640'
  notify: Restart Vault

- name: Deploy Vault container
  community.docker.docker_container:
    name: hashicorp-vault
    image: "hashicorp/vault:{{ vault_version }}"
    state: started
    restart_policy: unless-stopped
    capabilities:
      - IPC_LOCK    # 防止 memory swap（機密安全需求）
    published_ports:
      - "{{ vault_port }}:8200"
    networks:
      - name: "{{ vault_network }}"
    volumes:
      - "{{ vault_data_path }}:/vault/data"
      - "{{ vault_config_path }}:/vault/config:ro"
    env:
      VAULT_ADDR: "http://0.0.0.0:8200"
    command: server
    healthcheck:
      test: ["CMD", "vault", "status"]
      interval: 10s
      retries: 5

# ── Vault 初始化（只在第一次執行）──────────────────────────────────────
- name: Check if Vault is already initialized
  ansible.builtin.uri:
    url: "http://localhost:{{ vault_port }}/v1/sys/init"
    method: GET
    return_content: true
  register: vault_init_status
  delegate_to: localhost

- name: Initialize Vault (first time only)
  ansible.builtin.uri:
    url: "http://localhost:{{ vault_port }}/v1/sys/init"
    method: PUT
    body_format: json
    body:
      secret_shares: "{{ vault_key_shares }}"
      secret_threshold: "{{ vault_key_threshold }}"
  register: vault_init_result
  when: not vault_init_status.json.initialized
  delegate_to: localhost

# ⚠️ 重要：初始化結果包含 Root Token 和 Unseal Keys
# 必須安全地分發給不同的金鑰保管人
- name: Save Vault init result to local secure file
  ansible.builtin.copy:
    content: "{{ vault_init_result.json | to_nice_json }}"
    dest: "/tmp/vault-init-{{ ansible_date_time.date }}.json"
    mode: '0600'
  when:
    - vault_init_result is defined
    - not vault_init_result.skipped | default(false)
  delegate_to: localhost

- name: IMPORTANT - Save these keys securely!
  ansible.builtin.debug:
    msg:
      - "⚠️  Vault 已初始化！"
      - "Unseal Keys 和 Root Token 已儲存至 /tmp/vault-init-*.json"
      - "請立即將這些金鑰分發給不同的金鑰保管人，並從此機器刪除"
      - "Root Token 請存入 Ansible Vault（ironic，但這是最佳實踐）"
  when:
    - vault_init_result is defined
    - not vault_init_result.skipped | default(false)
```

### 16.2.3 自動密碼輪換 Playbook

```yaml
# playbooks/vault_rotate_secrets.yml
---
# 透過 Vault API 執行靜態機密輪換
# 適用於無法使用動態機密的場景（如 SMTP 密碼）
- name: Rotate static secrets in HashiCorp Vault
  hosts: localhost
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml    # 包含 vault_root_token

  vars:
    vault_addr: "http://{{ groups['gitlab_servers'][0] }}:8200"

  tasks:
    # ── 產生新的隨機密碼 ────────────────────────────────────────────────
    - name: Generate new database password
      ansible.builtin.set_fact:
        new_db_password: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits,punctuation') }}"

    # ── 更新 Vault 中的機密 ─────────────────────────────────────────────
    - name: Update database password in Vault
      ansible.builtin.uri:
        url: "{{ vault_addr }}/v1/secret/data/production/database"
        method: POST
        headers:
          X-Vault-Token: "{{ vault_root_token }}"
          Content-Type: application/json
        body_format: json
        body:
          data:
            db_password: "{{ new_db_password }}"
            rotated_at: "{{ ansible_date_time.iso8601 }}"
            rotated_by: "ansible-rotation-bot"
      register: vault_update_result

    # ── 同步更新實際資料庫密碼 ──────────────────────────────────────────
    - name: Apply new password to PostgreSQL
      community.docker.docker_container_exec:
        container: gitlab-postgres
        command: >
          psql -U postgres -c
          "ALTER USER gitlab PASSWORD '{{ new_db_password }}';"
      delegate_to: "{{ groups['gitlab_servers'][0] }}"
      no_log: true

    # ── 更新 GitLab 設定中的 DB 密碼 ──────────────────────────────────
    - name: Update GitLab configuration with new DB password
      community.docker.docker_container_exec:
        container: gitlab-ce
        command: >
          sed -i "s/db_password = .*/db_password = '{{ new_db_password }}'/"
          /etc/gitlab/gitlab.rb
      delegate_to: "{{ groups['gitlab_servers'][0] }}"
      no_log: true

    - name: Reconfigure GitLab to apply new password
      community.docker.docker_container_exec:
        container: gitlab-ce
        command: gitlab-ctl reconfigure
      delegate_to: "{{ groups['gitlab_servers'][0] }}"

    # ── 記錄輪換事件 ────────────────────────────────────────────────────
    - name: Log rotation event to audit-evidence
      ansible.builtin.lineinfile:
        path: "{{ playbook_dir }}/../audit-evidence-repo/secret-rotation-log.md"
        line: >-
          | {{ ansible_date_time.iso8601 }} | database/gitlab | 成功 |
          {{ ansible_user_id }} | Ansible 自動輪換 |
        create: true
      ignore_errors: true
```

### 16.2.4 PostgreSQL 動態機密設定

```yaml
# roles/hashicorp_vault/tasks/configure_db_secrets.yml
# 設定 Vault 的 Database Secrets Engine
---
- name: Enable database secrets engine
  ansible.builtin.uri:
    url: "http://localhost:{{ vault_port }}/v1/sys/mounts/database"
    method: POST
    headers:
      X-Vault-Token: "{{ vault_root_token }}"
    body_format: json
    body:
      type: database
    status_code: [200, 204]
  delegate_to: localhost

- name: Configure PostgreSQL connection in Vault
  ansible.builtin.uri:
    url: "http://localhost:{{ vault_port }}/v1/database/config/postgresql"
    method: POST
    headers:
      X-Vault-Token: "{{ vault_root_token }}"
    body_format: json
    body:
      plugin_name: postgresql-database-plugin
      allowed_roles: "readonly,readwrite"
      connection_url: >-
        postgresql://{{'{{'}}username{{'}}'}}:{{'{{'}}password{{'}}'}}@gitlab-postgres:5432/gitlabhq_production
      username: "vault_admin"
      password: "{{ vault_db_admin_password }}"
  delegate_to: localhost

- name: Create readonly role in Vault
  ansible.builtin.uri:
    url: "http://localhost:{{ vault_port }}/v1/database/roles/readonly"
    method: POST
    headers:
      X-Vault-Token: "{{ vault_root_token }}"
    body_format: json
    body:
      db_name: postgresql
      creation_statements:
        - "CREATE ROLE \"{{'{{'}}name{{'}}'}}\" WITH LOGIN PASSWORD '{{'{{'}}password{{'}}'}}' VALID UNTIL '{{'{{'}}expiration{{'}}'}}';"
        - "GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{'{{'}}name{{'}}'}}\";"
      default_ttl: "{{ vault_db_lease_duration }}"
      max_ttl: "{{ vault_db_max_lease_duration }}"
  delegate_to: localhost
```

---

## 16.3 驗證步驟

```bash
# 1. 部署 HashiCorp Vault
ansible-playbook playbooks/deploy_vault.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認 Vault 狀態
curl http://localhost:8200/v1/sys/health | python3 -m json.tool
# 預期：{"initialized": true, "sealed": false, ...}

# 3. 測試動態機密申請（需先完成 DB secrets engine 設定）
curl -H "X-Vault-Token: <token>" \
  http://localhost:8200/v1/database/creds/readonly
# 預期：{"data": {"username": "v-readonly-xxxx", "password": "..."}}

# 4. 執行密碼輪換
ansible-playbook playbooks/vault_rotate_secrets.yml \
  --vault-password-file ~/.vault_pass

# 5. 確認輪換記錄
cat audit-evidence-repo/secret-rotation-log.md
```

---

## 16.4 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 部署 Vault 容器、初始化設定、撰寫輪換 Playbook、記錄輪換事件 |
| **HashiCorp Vault 負責** | 機密儲存加密、動態機密產生、存取日誌、Policy 管理 |
| **適用場景** | 超過 10 個需要輪換的機密，或有動態機密需求的環境 |

*← [返回總覽](./README.md) | [上一章](./chapter-15-openvas-vulnerability-scanning.md)*
