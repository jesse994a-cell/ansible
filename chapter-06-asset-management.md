# 第 6 章：IT 資產管理自動化（Snipe-IT）

> **學習目標：** 使用 Ansible 自動部署 Snipe-IT 資產管理平台，並在每台伺服器初始化時自動呼叫 API 將硬體資訊（CPU、RAM、IP、序號）寫入資產清冊，產出符合 ISO 27001 Annex A 5.9 要求的資產清冊。

---

## 6.1 理論說明：為什麼 ISO 27001 需要資產清冊？

### 6.1.1 Annex A 5.9 的要求

ISO 27001:2022 Annex A 5.9「資訊及其他相關資產的清冊」要求組織必須：

- **識別**所有與資訊處理相關的資產
- **維護**資產清冊，包括擁有者、位置、狀態
- **定期審查**確保清冊與實際環境一致

稽核員最常問的問題：**「你怎麼確保資產清冊是最新的？」**

手動維護 Excel 的問題顯而易見——人員變動、設備異動往往來不及更新。解法是讓「基礎設施即程式碼」的流程本身產生資產清冊：**每次 Ansible 初始化一台主機，自動在資產管理系統建立或更新對應記錄。**

### 6.1.2 Snipe-IT 架構概覽

```
┌─────────────────────────────────────────────────────────────┐
│                    資產管理自動化架構                          │
│                                                             │
│  Ansible Playbook（主機初始化時觸發）                        │
│       │                                                     │
│       │  REST API (JSON)                                    │
│       ▼                                                     │
│  ┌──────────────┐    ┌──────────────┐   ┌───────────────┐  │
│  │  Snipe-IT    │◄──►│  MySQL 8.0   │   │  Redis Cache  │  │
│  │  (Laravel)   │    │              │   │               │  │
│  │  Port 80     │    └──────────────┘   └───────────────┘  │
│  └──────────────┘                                           │
│       │                                                     │
│       ▼                                                     │
│  匯出 CSV/PDF → ISO 27001 資產清冊證據                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.2 Ansible Role 結構：snipeit

```bash
ansible-galaxy role init roles/snipeit
```

```
roles/snipeit/
├── defaults/
│   └── main.yml          ← Snipe-IT 版本、DB 設定、API 金鑰
├── tasks/
│   ├── main.yml
│   ├── network.yml       ← Docker 網路
│   ├── database.yml      ← MySQL 容器
│   ├── redis.yml         ← Redis 容器
│   ├── app.yml           ← Snipe-IT 應用容器
│   └── init.yml          ← 初次設定（建立 API Token）
└── handlers/
    └── main.yml
```

### 6.2.1 defaults/main.yml

```yaml
# roles/snipeit/defaults/main.yml
---
# ── 版本設定 ──────────────────────────────────
snipeit_image: "snipe/snipe-it:v7.0.13"
mysql_image: "mysql:8.0"
redis_image: "redis:7-alpine"

# ── Docker 網路 ───────────────────────────────
snipeit_network: snipeit_net

# ── 資料路徑 ──────────────────────────────────
snipeit_data_path: /opt/snipeit
snipeit_uploads_path: "{{ snipeit_data_path }}/uploads"
snipeit_logs_path: "{{ snipeit_data_path }}/logs"
mysql_data_path: "{{ snipeit_data_path }}/mysql"

# ── 埠號 ──────────────────────────────────────
snipeit_port: 8088        # 避免與 GitLab 的 80 衝突

# ── 資料庫（機密來自 Vault）───────────────────
snipeit_db_name: snipeit
snipeit_db_user: snipeit
snipeit_db_password: "{{ vault_snipeit_db_password }}"
mysql_root_password: "{{ vault_mysql_root_password }}"

# ── 應用設定 ──────────────────────────────────
snipeit_app_url: "http://{{ ansible_host }}:{{ snipeit_port }}"
snipeit_app_key: "{{ vault_snipeit_app_key }}"    # base64:32 bytes
snipeit_app_locale: "zh-TW"
snipeit_timezone: "Asia/Taipei"
snipeit_mail_from: "snipeit@{{ ansible_domain | default('example.com') }}"

# ── API 存取（機密來自 Vault）─────────────────
# 此 Token 由 Snipe-IT 初始化後手動產生，再存入 Vault
snipeit_api_token: "{{ vault_snipeit_api_token }}"

# ── 資產分類 ID（在 Snipe-IT 中預先建立）──────
snipeit_category_server: 1
snipeit_category_network: 2
snipeit_manufacturer_custom: 3    # 虛擬機/自建伺服器

# ── 資產狀態 ID ───────────────────────────────
snipeit_status_deployed: 1        # 已部署
snipeit_status_pending: 2         # 待部署
```

### 6.2.2 tasks/main.yml

```yaml
# roles/snipeit/tasks/main.yml
---
- name: Create Docker network
  ansible.builtin.import_tasks: network.yml
  tags: [snipeit, network]

- name: Deploy MySQL
  ansible.builtin.import_tasks: database.yml
  tags: [snipeit, database]

- name: Deploy Redis
  ansible.builtin.import_tasks: redis.yml
  tags: [snipeit, redis]

- name: Deploy Snipe-IT application
  ansible.builtin.import_tasks: app.yml
  tags: [snipeit, app]
```

### 6.2.3 tasks/database.yml

```yaml
# roles/snipeit/tasks/database.yml
---
- name: Create MySQL data directory
  ansible.builtin.file:
    path: "{{ mysql_data_path }}"
    state: directory
    mode: '0750'

- name: Deploy MySQL container
  community.docker.docker_container:
    name: snipeit-mysql
    image: "{{ mysql_image }}"
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ snipeit_network }}"
    volumes:
      - "{{ mysql_data_path }}:/var/lib/mysql"
    env:
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
      MYSQL_DATABASE: "{{ snipeit_db_name }}"
      MYSQL_USER: "{{ snipeit_db_user }}"
      MYSQL_PASSWORD: "{{ snipeit_db_password }}"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost",
             "-u", "root", "--password={{ mysql_root_password }}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 60s

- name: Wait for MySQL to be healthy
  community.docker.docker_container_info:
    name: snipeit-mysql
  register: mysql_info
  until: mysql_info.container.State.Health.Status == "healthy"
  retries: 15
  delay: 10
```

### 6.2.4 tasks/app.yml

```yaml
# roles/snipeit/tasks/app.yml
---
- name: Create Snipe-IT data directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ snipeit_uploads_path }}"
    - "{{ snipeit_logs_path }}"

- name: Deploy Snipe-IT container
  community.docker.docker_container:
    name: snipeit-app
    image: "{{ snipeit_image }}"
    state: started
    restart_policy: unless-stopped
    published_ports:
      - "{{ snipeit_port }}:80"
    networks:
      - name: "{{ snipeit_network }}"
    volumes:
      - "{{ snipeit_uploads_path }}:/var/lib/snipeit/uploads"
      - "{{ snipeit_logs_path }}:/var/lib/snipeit/logs"
    env:
      # 應用設定
      APP_ENV: production
      APP_DEBUG: "false"
      APP_KEY: "{{ snipeit_app_key }}"
      APP_URL: "{{ snipeit_app_url }}"
      APP_LOCALE: "{{ snipeit_app_locale }}"
      APP_TIMEZONE: "{{ snipeit_timezone }}"

      # 資料庫連線（透過 Docker 網路，用 service name 連線）
      DB_HOST: snipeit-mysql
      DB_PORT: "3306"
      DB_DATABASE: "{{ snipeit_db_name }}"
      DB_USERNAME: "{{ snipeit_db_user }}"
      DB_PASSWORD: "{{ snipeit_db_password }}"

      # Redis
      REDIS_HOST: snipeit-redis
      REDIS_PORT: "6379"
      CACHE_DRIVER: redis
      SESSION_DRIVER: redis
      QUEUE_DRIVER: redis

      # 郵件設定（選用）
      MAIL_DRIVER: log    # 開發用，正式環境改為 smtp
      MAIL_FROM_ADDR: "{{ snipeit_mail_from }}"

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 120s

- name: Wait for Snipe-IT to be ready
  community.docker.docker_container_info:
    name: snipeit-app
  register: snipeit_info
  until: snipeit_info.container.State.Health.Status == "healthy"
  retries: 15
  delay: 20

- name: Run initial database migration
  community.docker.docker_container_exec:
    container: snipeit-app
    command: php artisan migrate --force
  register: migrate_result
  changed_when: "'Nothing to migrate' not in migrate_result.stdout"
```

---

## 6.3 資產自動註冊 Role：asset_register

這是本章的核心——每台伺服器初始化時，自動將自己的硬體資訊推送到 Snipe-IT。

```bash
ansible-galaxy role init roles/asset_register
```

```
roles/asset_register/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── gather_facts.yml    ← 收集主機硬體資訊
│   ├── lookup_asset.yml    ← 查詢資產是否已存在
│   ├── create_asset.yml    ← 建立新資產
│   └── update_asset.yml    ← 更新既有資產
└── vars/
    └── main.yml            ← API endpoint 常數
```

### 6.3.1 defaults/main.yml

```yaml
# roles/asset_register/defaults/main.yml
---
# Snipe-IT 伺服器位置（通常是同一台 gitlab_server）
snipeit_base_url: "http://{{ groups['gitlab_servers'][0] }}:8088/api/v1"
snipeit_api_token: "{{ vault_snipeit_api_token }}"

# 資產分類對應（依主機所屬群組自動判斷）
asset_category_map:
  webservers: 1
  dbservers: 1
  gitlab_servers: 1
  default: 1

# 資產模型 ID（在 Snipe-IT 中預建「Generic Server」模型）
snipeit_model_id_server: 1
snipeit_model_id_vm: 2

# 是否在資產備註中加入 Ansible Playbook 執行資訊
asset_include_ansible_metadata: true
```

### 6.3.2 tasks/main.yml

```yaml
# roles/asset_register/tasks/main.yml
---
- name: Gather hardware facts
  ansible.builtin.import_tasks: gather_facts.yml
  tags: [asset, facts]

- name: Look up existing asset in Snipe-IT
  ansible.builtin.import_tasks: lookup_asset.yml
  tags: [asset, lookup]

- name: Create new asset record
  ansible.builtin.import_tasks: create_asset.yml
  when: asset_existing_id is not defined or asset_existing_id == ""
  tags: [asset, create]

- name: Update existing asset record
  ansible.builtin.import_tasks: update_asset.yml
  when: asset_existing_id is defined and asset_existing_id != ""
  tags: [asset, update]
```

### 6.3.3 tasks/gather_facts.yml

```yaml
# roles/asset_register/tasks/gather_facts.yml
---
# ── 確保 facts 已收集 ────────────────────────────────────────────────────
- name: Gather full system facts
  ansible.builtin.setup:
    gather_subset:
      - hardware        # CPU、RAM
      - network         # IP、MAC
      - virtual         # 是否為 VM

# ── 取得序號（DMI Table）────────────────────────────────────────────────
- name: Get system serial number
  ansible.builtin.command:
    cmd: dmidecode -s system-serial-number
  register: raw_serial
  changed_when: false
  failed_when: false   # 部分 VM 可能無法取得序號

- name: Get system manufacturer
  ansible.builtin.command:
    cmd: dmidecode -s system-manufacturer
  register: raw_manufacturer
  changed_when: false
  failed_when: false

- name: Get system product name
  ansible.builtin.command:
    cmd: dmidecode -s system-product-name
  register: raw_product
  changed_when: false
  failed_when: false

# ── 整理資產資訊為統一格式 ───────────────────────────────────────────────
- name: Set asset_info dictionary
  ansible.builtin.set_fact:
    asset_info:
      # 唯一識別名稱（Ansible inventory 名稱）
      name: "{{ inventory_hostname }}"
      # 資產標籤（用作 serial 搜尋依據）
      asset_tag: "{{ inventory_hostname }}"
      # 序號（VM 可能為 "Not Specified"）
      serial: >-
        {{
          raw_serial.stdout | default('') | trim
          if raw_serial.stdout | default('') | trim not in ['', 'Not Specified', 'To Be Filled By O.E.M.']
          else ('VM-' + ansible_machine_id[:12])
        }}
      # IP 位址
      ip_address: "{{ ansible_default_ipv4.address | default(ansible_host) }}"
      # MAC 位址
      mac_address: "{{ ansible_default_ipv4.macaddress | default('N/A') }}"
      # CPU 資訊
      cpu_cores: "{{ ansible_processor_vcpus | default(ansible_processor_cores) }}"
      cpu_model: "{{ ansible_processor[2] | default('Unknown') }}"
      # 記憶體（GB，四捨五入）
      ram_gb: "{{ (ansible_memtotal_mb / 1024) | round(1) }}"
      # OS 資訊
      os_name: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
      os_kernel: "{{ ansible_kernel }}"
      # 硬體製造商
      manufacturer: "{{ raw_manufacturer.stdout | default('Unknown') | trim }}"
      product_name: "{{ raw_product.stdout | default('Unknown') | trim }}"
      # 是否為虛擬機
      is_virtual: "{{ ansible_virtualization_role == 'guest' }}"
      # 備註：Ansible 管理資訊
      notes: >-
        最後由 Ansible 更新：{{ ansible_date_time.iso8601 }}
        | Inventory Group: {{ group_names | join(', ') }}
        | Ansible Version: {{ ansible_version.full }}

- name: Display gathered asset info
  ansible.builtin.debug:
    var: asset_info
```

### 6.3.4 tasks/lookup_asset.yml

```yaml
# roles/asset_register/tasks/lookup_asset.yml
---
# ── 以 asset_tag（= inventory_hostname）搜尋既有資產 ────────────────────
- name: Search for existing asset by asset_tag
  ansible.builtin.uri:
    url: "{{ snipeit_base_url }}/hardware?asset_tag={{ asset_info.asset_tag }}&limit=1"
    method: GET
    headers:
      Authorization: "Bearer {{ snipeit_api_token }}"
      Accept: application/json
    status_code: [200]
    return_content: true
  register: asset_search_result
  delegate_to: localhost   # API 呼叫從 Control Node 發出

- name: Extract existing asset ID if found
  ansible.builtin.set_fact:
    asset_existing_id: >-
      {{
        asset_search_result.json.rows[0].id
        if (asset_search_result.json.total | int) > 0
        else ""
      }}

- name: Show lookup result
  ansible.builtin.debug:
    msg: >-
      {{
        '找到既有資產 ID: ' + asset_existing_id | string
        if asset_existing_id != ""
        else '未找到既有資產，將建立新記錄'
      }}
```

### 6.3.5 tasks/create_asset.yml

```yaml
# roles/asset_register/tasks/create_asset.yml
---
- name: Create new asset in Snipe-IT
  ansible.builtin.uri:
    url: "{{ snipeit_base_url }}/hardware"
    method: POST
    headers:
      Authorization: "Bearer {{ snipeit_api_token }}"
      Accept: application/json
      Content-Type: application/json
    body_format: json
    body:
      name: "{{ asset_info.name }}"
      asset_tag: "{{ asset_info.asset_tag }}"
      serial: "{{ asset_info.serial }}"
      model_id: >-
        {{
          snipeit_model_id_vm
          if asset_info.is_virtual
          else snipeit_model_id_server
        }}
      status_id: "{{ snipeit_status_deployed }}"
      # 自訂欄位（需在 Snipe-IT 中預先建立對應 Fieldset）
      _snipeit_ip_address_1: "{{ asset_info.ip_address }}"
      _snipeit_cpu_2: "{{ asset_info.cpu_model }} ({{ asset_info.cpu_cores }} vCPU)"
      _snipeit_ram_3: "{{ asset_info.ram_gb }} GB"
      _snipeit_os_4: "{{ asset_info.os_name }}"
      notes: "{{ asset_info.notes }}"
    status_code: [200, 201]
    return_content: true
  register: create_result
  delegate_to: localhost

- name: Verify asset was created successfully
  ansible.builtin.assert:
    that:
      - create_result.json.status == "success"
    fail_msg: "Snipe-IT 建立資產失敗：{{ create_result.json.messages | default('Unknown error') }}"

- name: Show created asset ID
  ansible.builtin.debug:
    msg: "✅ 資產已建立，ID: {{ create_result.json.payload.id }}"
```

### 6.3.6 tasks/update_asset.yml

```yaml
# roles/asset_register/tasks/update_asset.yml
---
- name: Update existing asset in Snipe-IT
  ansible.builtin.uri:
    url: "{{ snipeit_base_url }}/hardware/{{ asset_existing_id }}"
    method: PATCH    # PATCH = 只更新指定欄位，不覆蓋其他資料
    headers:
      Authorization: "Bearer {{ snipeit_api_token }}"
      Accept: application/json
      Content-Type: application/json
    body_format: json
    body:
      # 更新可能變動的資訊（IP 可能會變）
      _snipeit_ip_address_1: "{{ asset_info.ip_address }}"
      _snipeit_cpu_2: "{{ asset_info.cpu_model }} ({{ asset_info.cpu_cores }} vCPU)"
      _snipeit_ram_3: "{{ asset_info.ram_gb }} GB"
      _snipeit_os_4: "{{ asset_info.os_name }}"
      notes: "{{ asset_info.notes }}"
    status_code: [200]
    return_content: true
  register: update_result
  delegate_to: localhost

- name: Verify asset was updated successfully
  ansible.builtin.assert:
    that:
      - update_result.json.status == "success"
    fail_msg: "Snipe-IT 更新資產失敗：{{ update_result.json.messages | default('Unknown error') }}"

- name: Show updated asset
  ansible.builtin.debug:
    msg: "✅ 資產已更新，ID: {{ asset_existing_id }}"
```

---

## 6.4 網路設備資產發現（SNMP）

對於 Switch/Router 等無法安裝 Ansible agent 的設備，使用 SNMP 自動取得基本資訊：

```yaml
# playbooks/discover_network_assets.yml
---
- name: Discover network device assets via SNMP
  hosts: localhost        # 從 Control Node 發起 SNMP 查詢
  gather_facts: false

  vars_files:
    - ../vault/secrets.yml

  vars:
    # 要掃描的網段（SNMP community string 來自 Vault）
    snmp_community: "{{ vault_snmp_community }}"
    network_devices:
      - name: "core-switch-01"
        ip: "192.168.56.254"
        device_type: "switch"
      - name: "firewall-01"
        ip: "192.168.56.1"
        device_type: "firewall"

  tasks:
    - name: Install required Python SNMP library
      ansible.builtin.pip:
        name: pysnmp
        state: present

    - name: Query SNMP sysDescr (OID 1.3.6.1.2.1.1.1.0)
      community.general.snmp_facts:
        host: "{{ item.ip }}"
        version: v2c
        community: "{{ snmp_community }}"
      register: snmp_results
      loop: "{{ network_devices }}"
      ignore_errors: true   # 部分設備可能不支援 SNMP

    - name: Register network device to Snipe-IT
      ansible.builtin.uri:
        url: "{{ snipeit_base_url }}/hardware"
        method: POST
        headers:
          Authorization: "Bearer {{ vault_snipeit_api_token }}"
          Accept: application/json
          Content-Type: application/json
        body_format: json
        body:
          name: "{{ item.item.name }}"
          asset_tag: "NET-{{ item.item.name }}"
          status_id: 1
          model_id: 3    # Network Device 模型
          notes: >-
            SNMP sysDescr: {{ item.ansible_sysdescr | default('N/A') }}
            | IP: {{ item.item.ip }}
            | 發現時間: {{ ansible_date_time.iso8601 }}
      loop: "{{ snmp_results.results }}"
      when: not item.failed | default(false)
      delegate_to: localhost
```

---

## 6.5 通用 URI 模組範本（可套用其他 CMDB）

如果你的組織已有其他 CMDB（如 ServiceNow、iTop），以下是通用的 API 呼叫範本：

```yaml
# roles/asset_register/tasks/generic_cmdb_template.yml
---
# ════════════════════════════════════════════════════════════════
# 通用 CMDB API 整合範本
# 只需修改 cmdb_base_url、headers、body 格式即可套用
# ════════════════════════════════════════════════════════════════

- name: "[CMDB] Upsert asset record"
  ansible.builtin.uri:
    # 修改點 1：替換為你的 CMDB endpoint
    url: "{{ cmdb_base_url }}/assets/{{ asset_info.asset_tag }}"
    # 修改點 2：PUT = upsert（不存在則建立，存在則更新）
    method: PUT
    headers:
      # 修改點 3：依 CMDB 的認證方式調整
      Authorization: "Bearer {{ cmdb_api_token }}"
      Content-Type: "application/json"
      Accept: "application/json"
    body_format: json
    # 修改點 4：依 CMDB 的資料模型調整欄位名稱
    body:
      identifier: "{{ asset_info.asset_tag }}"
      display_name: "{{ asset_info.name }}"
      serial_number: "{{ asset_info.serial }}"
      ip_address: "{{ asset_info.ip_address }}"
      cpu_info: "{{ asset_info.cpu_model }}"
      memory_gb: "{{ asset_info.ram_gb }}"
      operating_system: "{{ asset_info.os_name }}"
      last_updated: "{{ ansible_date_time.iso8601 }}"
      managed_by: "ansible"
      environment: "{{ env | default('production') }}"
    status_code: [200, 201, 204]
    # 修改點 5：timeout 依 CMDB 回應速度調整
    timeout: 30
  register: cmdb_result
  delegate_to: localhost
  retries: 3        # 失敗最多重試 3 次（網路不穩時）
  delay: 5
```

---

## 6.6 整合進 site.yml

```yaml
# playbooks/site.yml 加入 asset_register role
---
- name: Apply common configuration and register assets
  hosts: all
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: common
      tags: [common]
    - role: asset_register   # ← 每次初始化自動更新資產清冊
      tags: [asset]
```

---

## 6.7 驗證步驟

```bash
# 1. 部署 Snipe-IT
ansible-playbook playbooks/deploy_snipeit.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認 Snipe-IT 正常運行
ansible gitlab_servers -m ansible.builtin.uri \
  -a "url=http://localhost:8088/health return_content=yes"
# 預期：{"status": "ok"}

# 3. 執行資產註冊（對所有主機）
ansible-playbook playbooks/site.yml \
  --vault-password-file ~/.vault_pass \
  --tags asset

# 4. 手動查詢 Snipe-IT API 確認資產已建立
curl -s -H "Authorization: Bearer <your-token>" \
  http://<snipeit-host>:8088/api/v1/hardware \
  | python3 -m json.tool | grep -E '"name"|"serial"|"ip_address"'

# 5. 驗證 Idempotency（重跑應更新而非重複建立）
ansible-playbook playbooks/site.yml \
  --vault-password-file ~/.vault_pass \
  --tags asset
# 應看到 update_asset task 執行而非 create_asset

# 6. 從 Snipe-IT 匯出資產清冊
# Admin UI → Reports → Asset List → Export to CSV
# 這份 CSV 就是 ISO 27001 稽核所需的資產清冊
```

---

## 6.8 Vault 設定補充

在 `vault/secrets.yml` 需加入：

```yaml
# vault/secrets.yml（加密前內容，新增以下欄位）
vault_snipeit_db_password: "SnipeDB@Secure2026"
vault_mysql_root_password: "MysqlR00t!Secure"
vault_snipeit_app_key: "base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
vault_snipeit_api_token: "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9..."
vault_snmp_community: "public"  # 正式環境請改為強密碼
```

> **取得 Snipe-IT APP_KEY：** 首次部署後，進入容器執行 `php artisan key:generate --show`，將輸出值存入 Vault。
>
> **取得 API Token：** 登入 Snipe-IT → 右上角個人圖示 → Manage API Keys → Create Token，將 token 存入 Vault。

---

## 6.9 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| Snipe-IT Docker 部署 | ☐ 瀏覽器可開啟 Snipe-IT 管理介面 |
| ansible.builtin.uri 呼叫 REST API | ☐ 理解 GET/POST/PATCH 的差異與使用時機 |
| 主機硬體 Facts 收集 | ☐ 能取得 CPU、RAM、序號、IP 等資訊 |
| 資產 Upsert 邏輯（先查後建/更新）| ☐ 重複執行不會產生重複資產記錄 |
| SNMP 網路設備發現 | ☐ 能查詢 Switch/Router 的 sysDescr |
| 通用 CMDB 範本 | ☐ 理解如何將範本套用至其他 CMDB 系統 |

**下一章：** [第 7 章 CIS Benchmark 合規掃描](./chapter-07-cis-benchmark.md) — 使用 Ansible Lockdown 對 Ubuntu 24.04 進行 CIS 標準掃描與自動修復。

---

*← [返回總覽](./README.md) | [上一章](./chapter-05-engineering-ci.md)*
