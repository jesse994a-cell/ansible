# 第 10 章：SSL 憑證監控與自動續期

> **學習目標：** 使用 `community.crypto` 模組定期掃描所有伺服器的 SSL/TLS 憑證到期日，在到期前 30/14/7 天自動發出分級告警，並整合 certbot 完成 Let's Encrypt 憑證的自動續期。所有憑證狀態記錄進 audit-evidence 倉庫，作為 ISO 27001 Annex A 8.24 的持續性合規證據。

---

## 10.1 理論說明：憑證管理為何是稽核重點？

### 10.1.1 ISO 27001 A.8.24 的要求

「使用加密技術」控制措施要求組織必須：
- 定義加密金鑰與憑證的**生命週期管理政策**
- 確保憑證在**到期前**完成更新
- 保存憑證管理的**操作記錄**

稽核員最常發現的問題不是「沒有 HTTPS」，而是：
1. **憑證悄悄過期**造成服務中斷（沒有監控機制）
2. **無法說明誰負責、何時更新**（無記錄）
3. **自簽憑證在生產環境使用**但無管控（無清冊）

### 10.1.2 憑證風險矩陣

```
到期狀態         風險等級    建議行動
─────────────────────────────────────────────────────
已過期           🔴 CRITICAL  立即更新，服務可能已中斷
7 天內到期       🔴 HIGH      今日內完成更新
14 天內到期      🟡 MEDIUM    本週內完成更新
30 天內到期      🟢 LOW       排程更新，列入本月工作
30 天以上        ✅ OK        正常，繼續監控
```

### 10.1.3 本章架構

```
┌─────────────────────────────────────────────────────────────┐
│                    SSL 憑證管理自動化                          │
│                                                             │
│  Scheduled Pipeline（每週）                                  │
│       │                                                     │
│       ▼                                                     │
│  cert_monitor role                                          │
│  ├── 掃描目標：                                              │
│  │   ├── 本機 /etc/ssl 憑證檔案                              │
│  │   ├── 遠端 HTTPS endpoints（含第三方）                    │
│  │   └── Docker 容器內的憑證                                 │
│  │                                                         │
│  ├── community.crypto.x509_certificate_info                 │
│  │   → 讀取到期日、Subject、Issuer、SAN                      │
│  │                                                         │
│  ├── 分級告警                                               │
│  │   → Slack / Email                                        │
│  │                                                         │
│  └── 報告 git commit → audit-evidence                       │
│                                                             │
│  certbot role（需要時觸發）                                  │
│  → Let's Encrypt 自動續期                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 10.2 Ansible Role 結構：cert_monitor

```bash
ansible-galaxy role init roles/cert_monitor
```

```
roles/cert_monitor/
├── defaults/
│   └── main.yml          ← 監控目標、告警閾值、通知設定
├── tasks/
│   ├── main.yml
│   ├── scan_files.yml    ← 掃描本機憑證檔案
│   ├── scan_remote.yml   ← 掃描遠端 HTTPS endpoints
│   ├── evaluate.yml      ← 計算到期天數、分級
│   ├── alert.yml         ← 發送分級告警
│   └── report.yml        ← 產出 Markdown 報告
└── templates/
    └── cert_report.md.j2 ← 報告模板
```

### 10.2.1 defaults/main.yml

```yaml
# roles/cert_monitor/defaults/main.yml
---
# ── 本機憑證掃描路徑 ──────────────────────────
cert_scan_paths:
  - /etc/ssl/certs
  - /etc/nginx/ssl
  - /etc/letsencrypt/live
  - /opt/gitlab/config/ssl

# 只掃描這些副檔名
cert_file_extensions:
  - "*.crt"
  - "*.pem"
  - "*.cer"

# ── 遠端 HTTPS endpoint 監控清單 ─────────────
# 除了本機憑證，也可以監控任意 HTTPS 網址
cert_remote_endpoints:
  - name: "GitLab"
    host: "{{ ansible_host }}"
    port: 443
  - name: "Snipe-IT"
    host: "{{ ansible_host }}"
    port: 8443
  # 加入第三方服務（即使不是自己管的也要監控）
  # - name: "Payment Gateway"
  #   host: "payment.example.com"
  #   port: 443

# ── 告警閾值（天數）──────────────────────────
cert_alert_critical_days: 7    # 7 天內到期 → CRITICAL
cert_alert_warning_days: 14    # 14 天內到期 → WARNING
cert_alert_info_days: 30       # 30 天內到期 → INFO

# ── 通知設定 ──────────────────────────────────
cert_slack_webhook: "{{ vault_slack_webhook_url | default('') }}"
cert_notify_email: "{{ admin_email | default('') }}"

# ── 報告設定 ──────────────────────────────────
cert_report_dir: /opt/cert-reports
cert_report_filename: "cert-status-{{ inventory_hostname }}-{{ ansible_date_time.date }}.md"
```

### 10.2.2 tasks/main.yml

```yaml
# roles/cert_monitor/tasks/main.yml
---
- name: Initialize cert findings list
  ansible.builtin.set_fact:
    cert_findings: []        # 所有憑證資訊
    cert_alerts: []          # 需告警的憑證

- name: Scan local certificate files
  ansible.builtin.import_tasks: scan_files.yml
  tags: [cert, scan_files]

- name: Scan remote HTTPS endpoints
  ansible.builtin.import_tasks: scan_remote.yml
  tags: [cert, scan_remote]

- name: Evaluate expiry and set alert levels
  ansible.builtin.import_tasks: evaluate.yml
  tags: [cert, evaluate]

- name: Send alerts if needed
  ansible.builtin.import_tasks: alert.yml
  when: cert_alerts | length > 0
  tags: [cert, alert]

- name: Generate certificate status report
  ansible.builtin.import_tasks: report.yml
  tags: [cert, report]
```

### 10.2.3 tasks/scan_files.yml

```yaml
# roles/cert_monitor/tasks/scan_files.yml
---
# ── 找出所有憑證檔案 ─────────────────────────────────────────────────────
- name: Find certificate files in configured paths
  ansible.builtin.find:
    paths: "{{ cert_scan_paths }}"
    patterns: "{{ cert_file_extensions }}"
    recurse: true
    file_type: file
  register: found_cert_files
  # 路徑不存在時不報錯
  ignore_errors: true

# ── 讀取每個憑證的詳細資訊 ──────────────────────────────────────────────
- name: Read certificate info for each file
  community.crypto.x509_certificate_info:
    path: "{{ item.path }}"
    # valid_at：檢查憑證在指定時間是否有效（用於提前驗證）
    valid_at:
      now: "+0d"
      in_30_days: "+30d"
  register: cert_file_info
  loop: "{{ found_cert_files.files | default([]) }}"
  loop_control:
    label: "{{ item.path }}"
  # 部分 .pem 可能是私鑰或 CA bundle，讀取失敗時跳過
  ignore_errors: true

# ── 整理成統一格式 ───────────────────────────────────────────────────────
- name: Append file cert info to findings
  ansible.builtin.set_fact:
    cert_findings: >-
      {{
        cert_findings + [{
          'source': 'file',
          'path': item.item.path,
          'name': item.item.path | basename,
          'subject': item.subject.commonName | default('N/A'),
          'issuer': item.issuer.organizationName | default('Unknown'),
          'not_after': item.not_after,
          'sans': item.subject_alt_name | default([]) | join(', '),
          'valid_now': item.valid_at.now,
          'valid_30d': item.valid_at.in_30_days,
          'serial': item.serial_number | default('N/A')
        }]
      }}
  loop: "{{ cert_file_info.results | default([]) }}"
  loop_control:
    label: "{{ item.item.path | default('unknown') }}"
  when:
    - not item.failed | default(false)
    - item.not_after is defined
```

### 10.2.4 tasks/scan_remote.yml

```yaml
# roles/cert_monitor/tasks/scan_remote.yml
---
# ── 透過 openssl 指令取得遠端憑證資訊 ───────────────────────────────────
- name: Fetch remote certificate via openssl
  ansible.builtin.shell: |
    echo | openssl s_client \
      -connect {{ item.host }}:{{ item.port }} \
      -servername {{ item.host }} \
      2>/dev/null | openssl x509 -noout \
      -subject -issuer -enddate -serial 2>/dev/null
  register: remote_cert_raw
  loop: "{{ cert_remote_endpoints }}"
  loop_control:
    label: "{{ item.name }} ({{ item.host }}:{{ item.port }})"
  changed_when: false
  # 遠端主機可能不可達，忽略錯誤繼續其他主機
  ignore_errors: true
  delegate_to: localhost   # 從 Control Node 發起連線

# ── 解析 openssl 輸出 ────────────────────────────────────────────────────
- name: Parse remote certificate info
  ansible.builtin.set_fact:
    cert_findings: >-
      {{
        cert_findings + [{
          'source': 'remote',
          'name': item.item.name,
          'path': item.item.host + ':' + (item.item.port | string),
          'subject': (item.stdout | regex_search('CN\s*=\s*([^\n,/]+)', '\1') | first | default('N/A')),
          'issuer':  (item.stdout | regex_search('O\s*=\s*([^\n,/]+)', '\1') | first | default('Unknown')),
          'not_after': (item.stdout | regex_search('notAfter=(.+)', '\1') | first | default('N/A')),
          'sans': 'N/A',
          'valid_now': not item.failed,
          'serial': (item.stdout | regex_search('serial=([A-Fa-f0-9]+)', '\1') | first | default('N/A'))
        }]
      }}
  loop: "{{ remote_cert_raw.results }}"
  loop_control:
    label: "{{ item.item.name }}"
  when: not item.failed | default(false)

# ── 記錄無法連線的 endpoint ──────────────────────────────────────────────
- name: Record unreachable endpoints as findings
  ansible.builtin.set_fact:
    cert_alerts: >-
      {{
        cert_alerts + [{
          'name': item.item.name,
          'path': item.item.host + ':' + (item.item.port | string),
          'level': 'CRITICAL',
          'days_remaining': -1,
          'message': 'endpoint 無法連線，憑證狀態未知'
        }]
      }}
  loop: "{{ remote_cert_raw.results }}"
  when: item.failed | default(false)
```

### 10.2.5 tasks/evaluate.yml

```yaml
# roles/cert_monitor/tasks/evaluate.yml
---
# ── 計算每個憑證距到期的天數，並分級 ────────────────────────────────────
- name: Calculate days until expiry and assign alert level
  ansible.builtin.set_fact:
    cert_alerts: >-
      {{
        cert_alerts + (
          [] if days_remaining > cert_alert_info_days
          else [{
            'name': item.name,
            'path': item.path,
            'subject': item.subject,
            'issuer': item.issuer,
            'not_after': item.not_after,
            'days_remaining': days_remaining,
            'level':
              'CRITICAL' if days_remaining <= 0 else
              'CRITICAL' if days_remaining <= cert_alert_critical_days else
              'WARNING'  if days_remaining <= cert_alert_warning_days else
              'INFO',
            'message':
              '憑證已過期！' if days_remaining <= 0 else
              (days_remaining | string) + ' 天後到期'
          }]
        )
      }}
  vars:
    # 計算到期天數：將 not_after 字串轉換成天數差
    days_remaining: >-
      {{
        (
          (item.not_after | to_datetime('%b %d %H:%M:%S %Y %Z'))
          - (ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ'))
        ).days
        | int
      }}
  loop: "{{ cert_findings }}"
  loop_control:
    label: "{{ item.name }}"
  # not_after 解析失敗時跳過
  ignore_errors: true
```

### 10.2.6 tasks/alert.yml

```yaml
# roles/cert_monitor/tasks/alert.yml
---
# ── 顯示告警摘要 ─────────────────────────────────────────────────────────
- name: Display certificate alerts
  ansible.builtin.debug:
    msg: >-
      [{{ item.level }}] {{ item.name }}
      ({{ item.path }}): {{ item.message }}
  loop: "{{ cert_alerts }}"

# ── Slack 告警 ───────────────────────────────────────────────────────────
- name: Send Slack notification for expiring certificates
  ansible.builtin.uri:
    url: "{{ cert_slack_webhook }}"
    method: POST
    body_format: json
    body:
      text: "🔐 *SSL 憑證到期告警* — {{ inventory_hostname }}"
      attachments:
        - color: >-
            {{
              'danger'  if cert_alerts | selectattr('level', 'eq', 'CRITICAL') | list | length > 0
              else 'warning'
            }}
          fields: >-
            {{
              cert_alerts | map(attribute='name') | zip(
                cert_alerts | map(attribute='message')
              ) | map('join', ': ') | list
              | map('community.general.dict_to_list') | list
            }}
          footer: "Ansible cert_monitor | {{ ansible_date_time.iso8601 }}"
    status_code: [200]
  when:
    - cert_slack_webhook | length > 0
    - cert_alerts | length > 0
  delegate_to: localhost
  # 告警發送失敗不應中斷整個 playbook
  ignore_errors: true

# ── 如有 CRITICAL 項目，在 Ansible 輸出中標記為警告（不 fail）────────────
- name: Log CRITICAL certificate alerts
  ansible.builtin.debug:
    msg: "⚠️  CRITICAL: {{ item.name }} — {{ item.message }}"
  loop: "{{ cert_alerts | selectattr('level', 'eq', 'CRITICAL') | list }}"
```

### 10.2.7 tasks/report.yml

```yaml
# roles/cert_monitor/tasks/report.yml
---
- name: Ensure report directory exists
  ansible.builtin.file:
    path: "{{ cert_report_dir }}"
    state: directory
    mode: '0750'

- name: Generate certificate status Markdown report
  ansible.builtin.template:
    src: cert_report.md.j2
    dest: "{{ cert_report_dir }}/{{ cert_report_filename }}"
    mode: '0640'

- name: Fetch report to control node
  ansible.builtin.fetch:
    src: "{{ cert_report_dir }}/{{ cert_report_filename }}"
    dest: "./cert-reports/{{ ansible_date_time.date }}/{{ inventory_hostname }}.md"
    flat: true
```

### 10.2.8 templates/cert_report.md.j2

```jinja2
{# roles/cert_monitor/templates/cert_report.md.j2 #}
# SSL/TLS 憑證狀態報告

| 項目 | 內容 |
|------|------|
| **主機** | `{{ inventory_hostname }}` |
| **掃描時間** | {{ ansible_date_time.iso8601 }} |
| **憑證總數** | {{ cert_findings | length }} |
| **需告警數** | {{ cert_alerts | length }} |

---

## 告警項目

{% if cert_alerts | length == 0 %}
✅ 所有憑證狀態正常，30 天內無到期風險。
{% else %}
| 等級 | 名稱 | 路徑 | 說明 |
|------|------|------|------|
{% for alert in cert_alerts | sort(attribute='days_remaining') %}
| {{ '🔴 CRITICAL' if alert.level == 'CRITICAL' else '🟡 WARNING' if alert.level == 'WARNING' else '🟢 INFO' }} | {{ alert.name }} | `{{ alert.path }}` | {{ alert.message }} |
{% endfor %}
{% endif %}

---

## 完整憑證清單

| 名稱 | Subject (CN) | 發行機構 | 到期日 | 狀態 |
|------|-------------|---------|--------|------|
{% for cert in cert_findings %}
{% set days = ((cert.not_after | to_datetime('%b %d %H:%M:%S %Y %Z')) - (ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ'))).days | int %}
| {{ cert.name }} | `{{ cert.subject }}` | {{ cert.issuer }} | {{ cert.not_after }} | {{ '🔴 過期' if days <= 0 else '🔴 ' + days|string + 'd' if days <= 7 else '🟡 ' + days|string + 'd' if days <= 30 else '✅ ' + days|string + 'd' }} |
{% endfor %}

---
_由 Ansible cert_monitor role 自動產生 — {{ ansible_date_time.iso8601 }}_
```

---

## 10.3 certbot 自動續期 Role

```bash
ansible-galaxy role init roles/certbot
```

### 10.3.1 defaults/main.yml

```yaml
# roles/certbot/defaults/main.yml
---
# ── 憑證申請設定 ──────────────────────────────
certbot_email: "{{ admin_email }}"

# 要申請/續期的域名清單
certbot_domains: []
# 範例：
# certbot_domains:
#   - domain: "gitlab.example.com"
#     webroot: "/var/www/html"    # webroot 驗證方式
#   - domain: "snipeit.example.com"
#     webroot: "/var/www/html"

# 驗證方式：webroot（需已有 HTTP 服務）或 standalone（Certbot 自建）
certbot_authenticator: webroot

# Nginx reload handler 名稱（續期後觸發）
certbot_post_hook: "systemctl reload nginx || true"

# 憑證儲存路徑（Let's Encrypt 預設）
certbot_cert_path: "/etc/letsencrypt/live"

# 自動續期 cron 設定
certbot_auto_renew: true
certbot_auto_renew_hour: "3"
certbot_auto_renew_minute: "30"
```

### 10.3.2 tasks/main.yml

```yaml
# roles/certbot/tasks/main.yml
---
# ── 安裝 certbot ─────────────────────────────────────────────────────────
- name: Install certbot
  ansible.builtin.apt:
    name:
      - certbot
      - python3-certbot-nginx   # Nginx plugin
    state: present
    update_cache: true

# ── 申請或續期憑證 ───────────────────────────────────────────────────────
- name: Obtain or renew certificates for configured domains
  ansible.builtin.command:
    cmd: >
      certbot certonly
      --non-interactive
      --agree-tos
      --email {{ certbot_email }}
      --{{ certbot_authenticator }}
      {% if certbot_authenticator == 'webroot' %}
      --webroot-path {{ item.webroot | default('/var/www/html') }}
      {% endif %}
      -d {{ item.domain }}
      --deploy-hook "{{ certbot_post_hook }}"
  loop: "{{ certbot_domains }}"
  loop_control:
    label: "{{ item.domain }}"
  register: certbot_result
  # certonly 在憑證已是最新時回傳 1，需要標記為 not changed
  changed_when: "'Congratulations' in certbot_result.stdout"
  failed_when:
    - certbot_result.rc != 0
    - "'Certificate not yet due for renewal' not in certbot_result.stderr"
  when: certbot_domains | length > 0

# ── 設定 cron 定期自動續期 ──────────────────────────────────────────────
- name: Setup certbot auto-renewal cron job
  ansible.builtin.cron:
    name: "certbot auto renew"
    hour: "{{ certbot_auto_renew_hour }}"
    minute: "{{ certbot_auto_renew_minute }}"
    # 每天嘗試，certbot 會自己判斷是否需要續期（< 30 天到期才更新）
    job: >
      certbot renew --quiet
      --deploy-hook "{{ certbot_post_hook }}"
      >> /var/log/certbot-renew.log 2>&1
    state: "{{ 'present' if certbot_auto_renew else 'absent' }}"
  when: certbot_auto_renew | bool
```

---

## 10.4 整合進 Scheduled Pipeline

```yaml
# .gitlab-ci.yml 新增 cert-monitor stage
cert-monitor-weekly:
  stage: evidence-collect      # 整合進既有的 evidence-collect stage
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/cert_monitor.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
  artifacts:
    when: always
    paths:
      - cert-reports/
    expire_in: 1 year
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

```yaml
# playbooks/cert_monitor.yml
---
- name: Monitor SSL/TLS certificate expiry
  hosts: all
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: cert_monitor
      tags: [cert, monitor]

  post_tasks:
    - name: Commit cert reports to audit-evidence
      ansible.builtin.include_tasks: ../roles/evidence_collector/tasks/git_commit.yml
      vars:
        evidence_report_dir: /opt/cert-reports
        audit_evidence_local_path: /opt/audit-evidence-repo
      tags: [cert, git]
```

---

## 10.5 驗證步驟

```bash
# 1. 手動執行憑證掃描
ansible-playbook playbooks/cert_monitor.yml \
  --vault-password-file ~/.vault_pass -v

# 2. 確認報告產出
cat cert-reports/$(date +%Y-%m-%d)/web-01.md

# 3. 測試即將到期的告警（建立測試憑證）
# 產生一個 3 天後到期的自簽憑證來測試告警
openssl req -x509 -nodes -days 3 \
  -newkey rsa:2048 \
  -keyout /tmp/test-expiring.key \
  -out /tmp/test-expiring.crt \
  -subj "/CN=test.expiring.example.com"

# 複製到目標主機測試
ansible web-01 -m ansible.builtin.copy \
  -a "src=/tmp/test-expiring.crt dest=/etc/ssl/certs/test-expiring.crt"

# 重跑掃描，應看到 CRITICAL 告警
ansible-playbook playbooks/cert_monitor.yml \
  --vault-password-file ~/.vault_pass --limit web-01

# 清理測試憑證
ansible web-01 -m ansible.builtin.file \
  -a "path=/etc/ssl/certs/test-expiring.crt state=absent"

# 4. 確認 certbot 自動續期設定
ansible all -m ansible.builtin.cron \
  -a "name='certbot auto renew' state=present" --check

# 5. 確認報告已推送到 audit-evidence
cd /opt/audit-evidence-repo
git log --oneline -3
```

---

## 10.6 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| `community.crypto.x509_certificate_info` 讀取憑證資訊 | ☐ 可取得 not_after、subject、issuer |
| 本機檔案與遠端 endpoint 雙管齊下掃描 | ☐ 兩種來源都出現在報告中 |
| 天數計算與分級（CRITICAL/WARNING/INFO）| ☐ 3 天到期憑證觸發 CRITICAL 告警 |
| Slack Webhook 告警整合 | ☐ 告警訊息出現在 Slack 頻道 |
| certbot 自動續期 + cron 設定 | ☐ cron job 存在且時間正確 |
| 報告進入 audit-evidence 倉庫 | ☐ 每週有憑證狀態快照可供稽核查閱 |

**下一章：** [第 11 章 帳號生命週期自動化](./chapter-11-account-lifecycle.md)

---

*← [返回總覽](./README.md) | [上一章](./chapter-09-scheduled-audit-pipeline.md)*
