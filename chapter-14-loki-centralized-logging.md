# 第 14 章：集中式日誌 SIEM（Loki + Grafana）

> **工具定位：** **Ansible 負責部署整個 Loki 觀測棧**（Loki、Promtail、Grafana），並統一設定所有主機的 Promtail agent 將日誌推送到集中儲存。日誌的查詢、告警、視覺化由 Loki/Grafana 負責。
>
> **ISO 27001 對應：** A.8.15（日誌記錄）、A.8.16（監控活動）

---

## 14.1 理論說明：為什麼需要集中式日誌？

### 14.1.1 分散式日誌的問題

目前架構中，每台伺服器的日誌各自存在 `/var/log/auth.log`、`/var/log/syslog`。稽核員問：「3 月 15 日凌晨 2 點誰登入了 db-01？」你必須 SSH 進去用 `grep`，效率極差且容易漏看。

集中式日誌解決了：
- **跨主機關聯分析**（同一個 IP 攻擊多台主機）
- **長期保留**（本機 logrotate 可能已刪除）
- **稽核員可直接查詢**（Grafana UI，不需 SSH 權限）

### 14.1.2 Loki vs ELK 的選擇

| 特性 | Grafana Loki | ELK Stack |
|------|-------------|-----------|
| 記憶體消耗 | 低（~500MB）| 高（~4GB+）|
| 查詢語言 | LogQL（類似 PromQL）| Lucene |
| 索引方式 | 只索引 Label，不索引內容 | 全文索引 |
| 適合場景 | 中小規模（< 100 台）| 大規模、全文搜尋需求 |

本章選用 Loki，更符合 Ubuntu + Docker 的輕量化部署需求。

---

## 14.2 架構設計

```
每台受管主機                        日誌伺服器（gitlab_servers）
────────────                        ─────────────────────────
/var/log/auth.log  ─┐               ┌──────────┐
/var/log/syslog    ─┤  Promtail     │  Loki    │  ← 儲存
/var/log/nginx/    ─┘  (port 9080)  │  :3100   │
Docker container   ─►  ─────────►   └──────────┘
  logs              labels:              │
  (host, service,   host=web-01          ▼
   env=production)  service=nginx   ┌──────────┐
                                    │ Grafana  │  ← 查詢 + 告警
                                    │  :3000   │
                                    └──────────┘
```

---

## 14.3 Ansible Role：loki_stack

```bash
ansible-galaxy role init roles/loki_stack
```

```
roles/loki_stack/
├── defaults/main.yml
├── tasks/
│   ├── main.yml
│   ├── loki.yml          ← 部署 Loki 容器
│   ├── grafana.yml       ← 部署 Grafana 容器
│   └── promtail.yml      ← 在所有主機部署 Promtail agent
└── templates/
    ├── loki-config.yml.j2
    ├── promtail-config.yml.j2
    └── grafana-datasource.yml.j2
```

### 14.3.1 defaults/main.yml

```yaml
# roles/loki_stack/defaults/main.yml
---
loki_version: "2.9.5"
grafana_version: "10.4.0"
promtail_version: "2.9.5"

loki_data_path: /opt/loki/data
grafana_data_path: /opt/loki/grafana
loki_network: loki_net

loki_port: 3100
grafana_port: 3000

# 日誌保留天數（ISO 27001 建議最少 90 天）
loki_retention_days: 90

# Grafana 管理員帳號（密碼來自 Vault）
grafana_admin_user: admin
grafana_admin_password: "{{ vault_grafana_admin_password }}"

# 要收集的日誌來源
promtail_scrape_configs:
  - job_name: system
    paths:
      - /var/log/auth.log
      - /var/log/syslog
    labels:
      service: system
  - job_name: nginx
    paths:
      - /var/log/nginx/access.log
      - /var/log/nginx/error.log
    labels:
      service: nginx
  - job_name: docker
    # 收集所有 Docker 容器日誌
    pipeline_stages:
      - docker: {}
```

### 14.3.2 tasks/loki.yml

```yaml
# roles/loki_stack/tasks/loki.yml
---
- name: Create Loki data directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ loki_data_path }}"
    - "{{ loki_data_path }}/chunks"
    - "{{ loki_data_path }}/index"

- name: Deploy Loki configuration
  ansible.builtin.template:
    src: loki-config.yml.j2
    dest: /opt/loki/loki-config.yml
    mode: '0644'
  notify: Restart Loki

- name: Deploy Loki container
  community.docker.docker_container:
    name: loki
    image: "grafana/loki:{{ loki_version }}"
    state: started
    restart_policy: unless-stopped
    published_ports:
      - "127.0.0.1:{{ loki_port }}:3100"    # 只綁定本機，避免外部直接存取
    networks:
      - name: "{{ loki_network }}"
    volumes:
      - "{{ loki_data_path }}:/loki"
      - "/opt/loki/loki-config.yml:/etc/loki/local-config.yaml:ro"
    command: -config.file=/etc/loki/local-config.yaml
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3100/ready"]
      interval: 10s
      retries: 5
```

### 14.3.3 templates/loki-config.yml.j2

```yaml
{# roles/loki_stack/templates/loki-config.yml.j2 #}
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 1h
  max_chunk_age: 1h
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/index_cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem

limits_config:
  # 日誌保留政策
  retention_period: {{ loki_retention_days }}d
  # 每個 stream 的最大 log rate
  ingestion_rate_mb: 16

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: {{ loki_retention_days }}d
```

### 14.3.4 tasks/promtail.yml（部署到所有受管主機）

```yaml
# roles/loki_stack/tasks/promtail.yml
# 此 task 對所有主機（非只有 log server）執行
---
- name: Download Promtail binary
  ansible.builtin.get_url:
    url: >-
      https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/
      promtail-linux-amd64.zip
    dest: /tmp/promtail.zip
    mode: '0644'

- name: Extract Promtail
  ansible.builtin.unarchive:
    src: /tmp/promtail.zip
    dest: /usr/local/bin/
    remote_src: true

- name: Create Promtail config
  ansible.builtin.template:
    src: promtail-config.yml.j2
    dest: /etc/promtail/config.yml
    mode: '0644'
  notify: Restart Promtail

- name: Create Promtail systemd service
  ansible.builtin.copy:
    content: |
      [Unit]
      Description=Promtail log shipper
      After=network-online.target

      [Service]
      ExecStart=/usr/local/bin/promtail-linux-amd64 -config.file=/etc/promtail/config.yml
      Restart=always
      RestartSec=5s

      [Install]
      WantedBy=multi-user.target
    dest: /etc/systemd/system/promtail.service
    mode: '0644'
  notify:
    - Reload systemd
    - Restart Promtail

- name: Enable and start Promtail
  ansible.builtin.service:
    name: promtail
    state: started
    enabled: true
```

### 14.3.5 templates/promtail-config.yml.j2

```yaml
{# roles/loki_stack/templates/promtail-config.yml.j2 #}
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://{{ groups['gitlab_servers'][0] }}:{{ loki_port }}/loki/api/v1/push

scrape_configs:
{% for config in promtail_scrape_configs %}
  - job_name: {{ config.job_name }}
    static_configs:
      - targets:
          - localhost
        labels:
          job: {{ config.job_name }}
          host: {{ inventory_hostname }}
          env: {{ env | default('production') }}
{% if config.paths is defined %}
          __path__: "{%- for p in config.paths -%}{{ p }}{%- if not loop.last -%},{%- endif -%}{%- endfor -%}"
{% endif %}
{% if config.labels is defined %}
{% for k, v in config.labels.items() %}
          {{ k }}: {{ v }}
{% endfor %}
{% endif %}
{% endfor %}

  # 自動收集 Docker 容器日誌
  - job_name: docker_containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: container
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: service
    pipeline_stages:
      - docker: {}
```

---

## 14.4 ISO 27001 告警規則設定

在 Grafana 設定關鍵安全事件的告警規則（Ansible 可透過 API 自動建立）：

```yaml
# roles/loki_stack/tasks/grafana_alerts.yml
---
# 透過 Grafana API 建立告警規則
- name: Create SSH brute force alert rule
  ansible.builtin.uri:
    url: "http://localhost:{{ grafana_port }}/api/v1/provisioning/alert-rules"
    method: POST
    headers:
      Authorization: "Basic {{ (grafana_admin_user + ':' + grafana_admin_password) | b64encode }}"
      Content-Type: application/json
    body_format: json
    body:
      title: "SSH Brute Force Detection"
      ruleGroup: "security"
      # LogQL 查詢：10 分鐘內同一 IP 超過 10 次 SSH 失敗
      data:
        - refId: A
          queryType: logql
          expr: >-
            sum by (host) (
              count_over_time(
                {job="system"} |= "Failed password" [10m]
              )
            ) > 10
      condition: "A"
      annotations:
        summary: "可能的 SSH 暴力破解攻擊"
        description: "主機 {{ $labels.host }} 在 10 分鐘內有超過 10 次 SSH 失敗"
    status_code: [200, 201]
  delegate_to: localhost
  ignore_errors: true
```

---

## 14.5 常用 LogQL 查詢（稽核用）

```logql
# 查詢特定時間段的 sudo 使用記錄
{job="system", host="web-01"} |= "sudo" | pattern `<_> <user> : TTY=<_> ; PWD=<_> ; USER=<_> ; COMMAND=<cmd>`

# 查詢所有 SSH 成功登入
{job="system"} |= "Accepted publickey" | regexp `for (?P<user>\w+) from (?P<ip>[\d.]+)`

# 查詢登入失敗（過去 7 天）
{job="system"} |= "Failed password" | count_over_time([7d])

# 跨主機查詢特定 IP 的活動
{job="system"} |= "192.168.56.100"
```

---

## 14.6 Playbook

```yaml
# playbooks/deploy_loki.yml
---
- name: Deploy Loki + Grafana on log server
  hosts: gitlab_servers
  gather_facts: true
  become: true
  vars_files:
    - ../vault/secrets.yml
  roles:
    - role: loki_stack
      tags: [loki, grafana]

- name: Deploy Promtail agent on all hosts
  hosts: all
  gather_facts: true
  become: true
  vars_files:
    - ../vault/secrets.yml
  tasks:
    - name: Install Promtail
      ansible.builtin.include_role:
        name: loki_stack
        tasks_from: promtail
      tags: [promtail]
```

---

## 14.7 驗證步驟

```bash
# 1. 部署整個 Loki 棧
ansible-playbook playbooks/deploy_loki.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認 Loki 正常接收日誌
curl http://localhost:3100/ready
# 預期：ready

# 3. 查詢 Loki 確認有日誌進來
curl -G http://localhost:3100/loki/api/v1/query \
  --data-urlencode 'query={job="system"}' \
  --data-urlencode 'limit=5' | python3 -m json.tool

# 4. 開啟 Grafana（瀏覽器）
# http://<host>:3000 → Explore → Loki data source
# 輸入查詢：{job="system", host="web-01"} |= "ssh"

# 5. 確認 90 天保留政策設定
curl http://localhost:3100/config | grep retention

# 6. 模擬 SSH 失敗觸發告警
# 故意 SSH 用錯密碼 10 次
# 查看 Grafana Alerting 頁面是否有告警
```

---

## 14.8 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 部署 Loki/Grafana/Promtail、統一設定所有主機的日誌收集、建立 Grafana 告警規則 |
| **Loki 負責** | 日誌儲存與索引、90 天保留政策執行 |
| **Grafana 負責** | 跨主機查詢、告警通知、稽核員視覺化介面 |

*← [返回總覽](./README.md) | [上一章](./chapter-13-trivy-container-scanning.md)*
