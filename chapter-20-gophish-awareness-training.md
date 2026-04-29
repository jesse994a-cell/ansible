# 第 20 章：員工意識訓練追蹤（GoPhish）

> **工具定位：** GoPhish 是釣魚演練平台，LMS 是訓練管理系統。**Ansible 負責部署 GoPhish**、設定演練排程，並產出訓練完成率與釣魚點擊率報告，作為 ISO 27001 持續性員工意識訓練的稽核證據。
>
> **ISO 27001 對應：** A.6.3（資訊安全意識、教育和培訓）— 所有人員必須接受適合其職責的安全意識訓練，並有記錄可查。

---

## 20.1 理論說明：為什麼員工意識訓練需要自動化？

技術上再嚴密的防護，都可能因為一封釣魚信而被突破。ISO 27001 A.6.3 不只要求「辦過訓練」，還要求：

- **全員完成率記錄**（可以查到誰完成了、誰沒有）
- **效果量測**（訓練後點擊率是否下降）
- **定期執行**（不是辦一次就結束）

GoPhish 提供的釣魚模擬演練，讓「員工對釣魚信的辨識率」從主觀自評，變成客觀可量測的數字。

### 20.1.1 訓練效果的量化指標

```
指標                  意義                    ISO 27001 用途
────────────────────────────────────────────────────────────
釣魚點擊率          點擊惡意連結的員工比例    A.6.3 訓練效果量測
訓練完成率          完成意識訓練的員工比例    A.6.3 訓練涵蓋率
報告率              舉報可疑信件的員工比例    A.6.3 積極參與程度
趨勢                歷次演練的改善幅度        A.6.3 持續改善證據
```

---

## 20.2 Ansible Role：gophish

```bash
ansible-galaxy role init roles/gophish
```

### 20.2.1 defaults/main.yml

```yaml
# roles/gophish/defaults/main.yml
---
gophish_version: "0.12.1"
gophish_data_path: /opt/gophish
gophish_port: 3333          # Admin UI（僅限內網存取）
gophish_phishing_port: 8880  # 釣魚頁面接收埠

# Admin 帳號（密碼來自 Vault）
gophish_admin_password: "{{ vault_gophish_admin_password }}"

# 釣魚演練設定
gophish_smtp_host: "smtp.example.com"
gophish_smtp_port: 587
gophish_smtp_user: "phishing-test@example.com"
gophish_smtp_password: "{{ vault_gophish_smtp_password }}"
gophish_from_address: "IT Security <security@example.com>"

# 演練頻率（每季）
gophish_campaign_interval: "quarterly"

# 員工清單來源（重用第 11 章的使用者目錄）
gophish_users_file: "{{ playbook_dir }}/../inventory/users/all_users.yml"
```

### 20.2.2 tasks/main.yml

```yaml
# roles/gophish/tasks/main.yml
---
- name: Create GoPhish data directory
  ansible.builtin.file:
    path: "{{ gophish_data_path }}"
    state: directory
    mode: '0750'

- name: Deploy GoPhish configuration
  ansible.builtin.template:
    src: gophish-config.json.j2
    dest: "{{ gophish_data_path }}/config.json"
    mode: '0640'
  notify: Restart GoPhish

- name: Deploy GoPhish container
  community.docker.docker_container:
    name: gophish
    image: "gophish/gophish:{{ gophish_version }}"
    state: started
    restart_policy: unless-stopped
    published_ports:
      # Admin UI 只對內網開放
      - "127.0.0.1:{{ gophish_port }}:3333"
      # 釣魚頁面可對外（需有對應 DNS）
      - "{{ gophish_phishing_port }}:80"
    volumes:
      - "{{ gophish_data_path }}:/app/data"
      - "{{ gophish_data_path }}/config.json:/app/config.json:ro"
    healthcheck:
      test: ["CMD", "curl", "-fk", "https://localhost:3333/"]
      interval: 10s
      retries: 5

- name: Wait for GoPhish to be ready
  ansible.builtin.wait_for:
    host: localhost
    port: "{{ gophish_port }}"
    delay: 5
    timeout: 60
```

### 20.2.3 templates/gophish-config.json.j2

```jinja2
{
  "admin_server": {
    "listen_url": "0.0.0.0:3333",
    "use_tls": true,
    "cert_path": "/app/data/gophish.crt",
    "key_path": "/app/data/gophish.key"
  },
  "phish_server": {
    "listen_url": "0.0.0.0:80",
    "use_tls": false
  },
  "db_name": "sqlite3",
  "db_path": "/app/data/gophish.db",
  "migrations_prefix": "db/db_",
  "contact_address": "{{ gophish_admin_password }}",
  "logging": {
    "filename": "/app/data/gophish.log",
    "level": "info"
  }
}
```

---

## 20.3 GoPhish API 自動建立演練活動

```yaml
# roles/gophish/tasks/create_campaign.yml
---
# ── 取得 API Token ───────────────────────────────────────────────────────
- name: Get GoPhish API key
  ansible.builtin.uri:
    url: "https://localhost:{{ gophish_port }}/api/login"
    method: POST
    validate_certs: false
    body_format: json
    body:
      username: admin
      password: "{{ gophish_admin_password }}"
    return_content: true
  register: gophish_auth
  delegate_to: localhost

# ── 建立目標群組（從 all_users.yml 讀取）────────────────────────────────
- name: Load active users
  ansible.builtin.include_vars:
    file: "{{ gophish_users_file }}"
    name: user_data

- name: Create target group from active users
  ansible.builtin.uri:
    url: "https://localhost:{{ gophish_port }}/api/groups/"
    method: POST
    validate_certs: false
    headers:
      Authorization: "{{ gophish_auth.json.data.api_key }}"
    body_format: json
    body:
      name: "All Staff - {{ ansible_date_time.date }}"
      targets: >-
        {{
          user_data.users
          | selectattr('status', 'eq', 'active')
          | map(attribute='email')
          | map('community.general.dict', 'email', ?)
          | list
        }}
    status_code: [200, 201]
  delegate_to: localhost

# ── 建立釣魚演練活動 ─────────────────────────────────────────────────────
- name: Create phishing campaign
  ansible.builtin.uri:
    url: "https://localhost:{{ gophish_port }}/api/campaigns/"
    method: POST
    validate_certs: false
    headers:
      Authorization: "{{ gophish_auth.json.data.api_key }}"
    body_format: json
    body:
      name: "Q{{ ansible_date_time.month | int // 4 + 1 }} {{ ansible_date_time.year }} Security Awareness Test"
      template:
        name: "Standard IT Phishing Template"
      landing_page:
        name: "IT Login Page Clone"
      url: "http://{{ ansible_host }}:{{ gophish_phishing_port }}"
      smtp:
        name: "Company SMTP"
      launch_date: "{{ ansible_date_time.iso8601 }}"
      send_by_date: null    # 立即發送
      groups:
        - name: "All Staff - {{ ansible_date_time.date }}"
    status_code: [200, 201]
  delegate_to: localhost
```

---

## 20.4 演練結果報告

```yaml
# roles/gophish/tasks/collect_results.yml
# 演練結束後（建議等 2 週）執行
---
- name: Get campaign results from GoPhish API
  ansible.builtin.uri:
    url: "https://localhost:{{ gophish_port }}/api/campaigns/{{ campaign_id }}/results"
    method: GET
    validate_certs: false
    headers:
      Authorization: "{{ gophish_auth.json.data.api_key }}"
    return_content: true
  register: campaign_results
  delegate_to: localhost

- name: Calculate key metrics
  ansible.builtin.set_fact:
    total_targets: "{{ campaign_results.json.results | length }}"
    clicked: >-
      {{ campaign_results.json.results | selectattr('status', 'eq', 'Clicked Link') | list | length }}
    submitted: >-
      {{ campaign_results.json.results | selectattr('status', 'eq', 'Submitted Data') | list | length }}
    reported: >-
      {{ campaign_results.json.results | selectattr('status', 'eq', 'Email Reported') | list | length }}

- name: Generate awareness training report
  ansible.builtin.copy:
    content: |
      # 員工安全意識訓練報告

      **演練日期：** {{ ansible_date_time.date }}
      **演練類型：** 釣魚郵件模擬

      ## 關鍵指標

      | 指標 | 數值 | 目標 |
      |------|------|------|
      | 目標人數 | {{ total_targets }} | — |
      | 釣魚點擊率 | {{ (clicked | int / total_targets | int * 100) | round(1) }}% | < 5% |
      | 資訊提交率 | {{ (submitted | int / total_targets | int * 100) | round(1) }}% | < 2% |
      | 主動舉報率 | {{ (reported | int / total_targets | int * 100) | round(1) }}% | > 20% |

      ## 結論與後續行動

      {% if (clicked | int / total_targets | int) > 0.1 %}
      ⚠️ **點擊率超過 10%，建議加強訓練並於 30 天內再次演練。**
      {% else %}
      ✅ 本次演練表現符合預期，繼續維持季度演練頻率。
      {% endif %}

      _本報告已匯入稽核證據系統_
    dest: "./phishing-report-{{ ansible_date_time.date }}.md"
    mode: '0644'
  delegate_to: localhost
```

---

## 20.5 訓練完成率追蹤（LMS 整合）

```yaml
# 若有使用 LMS（如 Moodle、TalentLMS）
# 透過 Ansible 呼叫 LMS API 取得訓練完成率
- name: Get training completion from LMS API
  ansible.builtin.uri:
    url: "{{ lms_api_url }}/api/v1/courses/{{ security_course_id }}/completion"
    method: GET
    headers:
      Authorization: "Bearer {{ vault_lms_api_token }}"
    return_content: true
  register: lms_completion
  delegate_to: localhost
  ignore_errors: true

- name: Calculate completion rate
  ansible.builtin.set_fact:
    completion_rate: >-
      {{
        (lms_completion.json.completed | int /
         lms_completion.json.enrolled | int * 100)
        | round(1)
        if lms_completion is not failed
        else 'N/A'
      }}
```

---

## 20.6 驗證步驟

```bash
# 1. 部署 GoPhish
ansible-playbook playbooks/deploy_gophish.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認 Admin UI 可存取（僅限本機）
curl -kI https://localhost:3333/

# 3. 建立測試演練（只包含測試信箱）
ansible-playbook playbooks/phishing_campaign.yml \
  --vault-password-file ~/.vault_pass \
  -e "test_mode=true"

# 4. 等待 2 週後收集結果
ansible-playbook playbooks/phishing_results.yml \
  --vault-password-file ~/.vault_pass \
  -e "campaign_id=1"

# 5. 查看報告
cat phishing-report-$(date +%Y-%m-%d).md

# 6. 確認報告已進入 audit-evidence 倉庫
cd /opt/audit-evidence-repo
git log --oneline -3
```

---

## 20.7 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 部署 GoPhish、透過 API 建立演練活動、收集結果、產出訓練報告 |
| **GoPhish 負責** | 釣魚郵件發送、點擊追蹤、資訊提交偵測 |
| **LMS 負責**（選用）| 線上安全意識課程管理、完成率追蹤 |
| **稽核用途** | 每季演練報告 + 趨勢圖，證明員工意識訓練持續進行 |

---

## 完整課程（第 13-20 章）定位說明

這 8 章的共同特點是：**Ansible 是部署與整合的橋樑，核心功能由專業工具提供**。

| 章節 | Ansible 貢獻 | 主要工具 |
|------|-------------|---------|
| 第 13 章 Trivy | 安裝 Binary、更新 DB Cron | GitLab CI |
| 第 14 章 Loki | 部署整個觀測棧、統一 Promtail 設定 | Loki / Grafana |
| 第 15 章 OpenVAS | 部署容器、API 觸發掃描、收集報告 | OpenVAS / GVM |
| 第 16 章 Vault | 部署、初始化、API 呼叫輪換 | HashiCorp Vault |
| 第 17 章 GitLeaks | 安裝 Binary、pre-commit hook 設定 | GitLeaks / GitLab CI |
| 第 18 章 Change Mgmt | 透過 GitLab API 設定 Protected Branch、MR 規則 | GitLab |
| 第 19 章 Eramba | 部署容器、API 匯入資產與供應商資料 | Eramba GRC |
| 第 20 章 GoPhish | 部署容器、API 建立活動、收集結果 | GoPhish |

*← [返回總覽](./README.md) | [上一章](./chapter-19-eramba-grc.md)*
