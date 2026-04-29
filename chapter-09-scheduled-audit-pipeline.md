# 第 9 章：GitLab CI/CD 定期稽核流水線

> **學習目標：** 將第 7 章的 CIS Benchmark 掃描與第 8 章的 Evidence Collector 整合進 GitLab Scheduled Pipeline，建立每週自動執行的稽核閉環。同時概覽 Eramba GRC 平台的 Docker 部署，作為管理 ISO 27001 文件與風險的集中平台。

---

## 9.1 理論說明：稽核自動化的閉環設計

### 9.1.1 完整稽核自動化架構

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ISO 27001 稽核自動化閉環                           │
│                                                                     │
│  ┌──────────────┐   週期觸發    ┌──────────────────────────────┐   │
│  │  GitLab      │ ────────────► │  Scheduled Pipeline          │   │
│  │  Schedule    │               │                              │   │
│  │  (每週一     │               │  Stage 1: evidence_collect   │   │
│  │   02:00)     │               │  ↓ 收集帳號/套件/權限/備份   │   │
│  └──────────────┘               │                              │   │
│                                  │  Stage 2: cis_audit          │   │
│                                  │  ↓ CIS Level 1 合規掃描      │   │
│                                  │                              │   │
│                                  │  Stage 3: report_commit      │   │
│                                  │  ↓ Markdown 報告 git commit  │   │
│                                  │                              │   │
│                                  │  Stage 4: notify             │   │
│                                  │  ↓ 通知 (Slack / Email)      │   │
│                                  └──────────────────────────────┘   │
│                                           │                         │
│                          ┌────────────────┼────────────────┐        │
│                          ▼                ▼                ▼        │
│                  audit-evidence    Eramba GRC         Grafana        │
│                  Repository        Dashboard          Dashboard      │
│                  (稽核證據庫)      (ISMS 管理)        (趨勢圖)       │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.1.2 為什麼要「定期」而非「按需」？

ISO 27001 稽核員最喜歡看到的不是「我們有做過一次掃描」，而是**「我們有持續性的監控機制，這是過去 12 個月每週的合規記錄」**。

Git commit history 的時間線就是最有力的持續性證明：

```
audit-evidence-2026/
├── 2026-01-06/    ← 1 月第 1 週
│   ├── web-01-evidence.md
│   └── db-01-evidence.md
├── 2026-01-13/    ← 1 月第 2 週
│   ├── web-01-evidence.md
│   └── db-01-evidence.md
... （52 週完整記錄）
└── 2026-12-28/
```

---

## 9.2 GitLab Scheduled Pipeline 設定

### 9.2.1 建立 Schedule（GitLab UI）

1. 進入 GitLab 專案 → **CI/CD** → **Schedules**
2. 點擊 **New Schedule**
3. 填入設定：

| 欄位 | 值 |
|------|----|
| Description | Weekly Security Audit |
| Interval Pattern | `0 2 * * 1`（每週一 02:00 UTC+8 需調整） |
| Cron Timezone | Asia/Taipei |
| Target Branch | main |
| Variables | `SCHEDULE_TYPE = weekly_audit` |

> **Cron 表達式說明：** `0 2 * * 1` = 每週一 02:00
> - `0` = 分鐘 0
> - `2` = 小時 2（UTC 時間，若 GitLab 在 UTC+8 則設為 18 代表 02:00 台灣時間）
> - `*` = 每天
> - `*` = 每月
> - `1` = 週一

### 9.2.2 完整 .gitlab-ci.yml（整合版）

```yaml
# .gitlab-ci.yml - 完整整合版（涵蓋第 5 章 + 第 9 章新增 stages）
---
stages:
  - validate
  - test
  - cis-audit
  - evidence-collect
  - report-commit
  - notify
  - dry-run
  - deploy

variables:
  ANSIBLE_FORCE_COLOR: "1"
  ANSIBLE_HOST_KEY_CHECKING: "False"
  ANSIBLE_STDOUT_CALLBACK: "yaml"

# ── 共用基礎設定 ─────────────────────────────────────────────────────────
.ansible_base:
  image: python:3.12-slim
  before_script:
    - apt-get update -qq
    - apt-get install -y -qq git openssh-client rsync
    - pip install --quiet ansible ansible-lint
    - ansible-galaxy collection install -r requirements.yml -q
    - ansible-galaxy role install -r requirements.yml -q
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/.vault_pass
    - chmod 600 /tmp/.vault_pass
  after_script:
    - rm -f /tmp/.vault_pass ~/.ssh/id_ed25519 ~/.ssh/audit_evidence_deploy_key

# ════════════════════════════════════════════════════════════════════════
# 定期稽核 Stages（只在 Schedule 觸發時執行）
# ════════════════════════════════════════════════════════════════════════

# ── Stage: evidence-collect ──────────────────────────────────────────────
evidence-collect-all:
  stage: evidence-collect
  extends: .ansible_base
  before_script:
    - !reference [.ansible_base, before_script]
    # 設定 audit-evidence repo 的 Deploy Key
    - echo "$AUDIT_EVIDENCE_DEPLOY_KEY" | tr -d '\r' > ~/.ssh/audit_evidence_deploy_key
    - chmod 600 ~/.ssh/audit_evidence_deploy_key
    - ssh-keyscan gitlab.example.com >> ~/.ssh/known_hosts 2>/dev/null
  script:
    - >
      ansible-playbook playbooks/evidence_collector.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -e "collection_label=scheduled_weekly"
      -v
  artifacts:
    when: always
    paths:
      - audit-reports/
    expire_in: 1 year
  rules:
    # 只在 Schedule 觸發且類型為 weekly_audit 時執行
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"
    # 也允許手動在 main branch 觸發
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

# ── Stage: cis-audit ─────────────────────────────────────────────────────
cis-audit-production:
  stage: cis-audit
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/cis_audit.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -e "target_hosts=all cis_level=1"
      -v
  artifacts:
    when: always
    paths:
      - audit-reports/
    expire_in: 1 year
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

# ── Stage: report-commit ─────────────────────────────────────────────────
# 整合報告：生成當週總覽，並推送到 audit-evidence repo
weekly-report-summary:
  stage: report-commit
  extends: .ansible_base
  before_script:
    - !reference [.ansible_base, before_script]
    - echo "$AUDIT_EVIDENCE_DEPLOY_KEY" | tr -d '\r' > ~/.ssh/audit_evidence_deploy_key
    - chmod 600 ~/.ssh/audit_evidence_deploy_key
    - ssh-keyscan gitlab.example.com >> ~/.ssh/known_hosts 2>/dev/null
  script:
    # 產生當週摘要報告
    - python3 scripts/generate_weekly_summary.py audit-reports/ > weekly-summary.md
    # 推送到 audit-evidence 倉庫
    - git clone --depth=1 $AUDIT_EVIDENCE_REPO_URL /tmp/audit-evidence-repo
    - cp weekly-summary.md /tmp/audit-evidence-repo/$(date +%Y-%m-%d)/WEEKLY-SUMMARY.md
    - cd /tmp/audit-evidence-repo
    - git config user.name "Ansible Audit Bot"
    - git config user.email "ansible-bot@example.com"
    - git add .
    - |
      if ! git diff --staged --quiet; then
        git commit -m "weekly-summary($(date +%Y-%m-%d)): automated security audit report

        Pipeline: $CI_PIPELINE_URL
        Triggered: $CI_PIPELINE_SOURCE
        Branch: $CI_COMMIT_BRANCH"
        GIT_SSH_COMMAND="ssh -i ~/.ssh/audit_evidence_deploy_key" \
          git push origin main
      fi
  needs:
    - evidence-collect-all
    - cis-audit-production
  artifacts:
    paths:
      - weekly-summary.md
    expire_in: 1 year
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"

# ── Stage: notify ────────────────────────────────────────────────────────
notify-audit-complete:
  stage: notify
  image: alpine:latest
  before_script:
    - apk add -q curl
  script:
    # Slack 通知（若有設定 Webhook）
    - |
      if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -s -X POST "$SLACK_WEBHOOK_URL" \
          -H 'Content-type: application/json' \
          -d "{
            \"text\": \"✅ *週期安全稽核完成*\",
            \"attachments\": [{
              \"color\": \"good\",
              \"fields\": [
                {\"title\": \"執行時間\", \"value\": \"$(date)\", \"short\": true},
                {\"title\": \"Pipeline\", \"value\": \"$CI_PIPELINE_URL\", \"short\": true},
                {\"title\": \"報告位置\", \"value\": \"$AUDIT_EVIDENCE_REPO_URL\", \"short\": false}
              ]
            }]
          }"
      fi
    # Email 通知（透過 GitLab Pipeline Email）
    - echo "稽核流水線執行完畢，報告已推送到 audit-evidence 倉庫。"
  needs:
    - weekly-report-summary
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"
      when: always   # 即使前面有失敗也要通知

# ════════════════════════════════════════════════════════════════════════
# 原有 Deploy Stages（非 Schedule 時使用）
# ════════════════════════════════════════════════════════════════════════
deploy-production:
  stage: deploy
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/site.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -v
  environment:
    name: production
  rules:
    # 只在非 Schedule 的 main branch 才出現
    - if: $CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE != "schedule"
      when: manual
```

---

## 9.3 週報摘要產生腳本

```python
#!/usr/bin/env python3
# scripts/generate_weekly_summary.py
"""
讀取 audit-reports/ 目錄下的所有 .md 報告，
產生一份 WEEKLY-SUMMARY.md 彙整所有主機的稽核結果。
"""

import os
import sys
import re
from datetime import datetime
from pathlib import Path


def parse_evidence_report(filepath: Path) -> dict:
    """解析 Evidence Collector 產出的 Markdown 報告，提取關鍵摘要資訊。"""
    content = filepath.read_text(encoding="utf-8")
    hostname = filepath.stem.replace("-evidence", "")

    # 提取主機資訊
    ip_match = re.search(r"\*\*IP 位址\*\* \| `([^`]+)`", content)
    os_match = re.search(r"\*\*作業系統\*\* \| ([^\n]+)", content)
    time_match = re.search(r"\*\*收集時間\*\* \| ([^\n]+)", content)

    # 提取 findings（異常項目）
    findings = re.findall(r"- (\[(?:HIGH|MEDIUM|LOW)\] [^\n]+)", content)

    return {
        "hostname": hostname,
        "ip": ip_match.group(1) if ip_match else "N/A",
        "os": os_match.group(1).strip() if os_match else "N/A",
        "collected_at": time_match.group(1).strip() if time_match else "N/A",
        "findings": findings,
        "findings_high": [f for f in findings if f.startswith("[HIGH]")],
        "findings_medium": [f for f in findings if f.startswith("[MEDIUM]")],
        "status": "⚠️ 異常" if any(f.startswith("[HIGH]") for f in findings) else "✅ 正常",
    }


def generate_summary(reports_dir: str) -> str:
    reports_path = Path(reports_dir)
    today = datetime.now().strftime("%Y-%m-%d")

    # 找所有報告
    report_files = list(reports_path.rglob("*-evidence.md"))
    if not report_files:
        return f"# 週報摘要 {today}\n\n無報告檔案。\n"

    parsed = [parse_evidence_report(f) for f in report_files]
    total = len(parsed)
    normal = sum(1 for p in parsed if p["status"].startswith("✅"))
    abnormal = total - normal

    lines = [
        f"# 週期安全稽核週報 — {today}",
        "",
        "## 執行摘要",
        "",
        f"| 指標 | 數值 |",
        f"|------|------|",
        f"| 掃描主機總數 | {total} |",
        f"| 狀態正常 | {normal} |",
        f"| 有異常項目 | {abnormal} |",
        f"| 報告生成時間 | {datetime.now().isoformat()} |",
        "",
        "## 各主機狀態",
        "",
        "| 主機 | IP | 作業系統 | 狀態 | HIGH 異常數 |",
        "|------|----|----------|------|------------|",
    ]

    for p in sorted(parsed, key=lambda x: x["hostname"]):
        lines.append(
            f"| `{p['hostname']}` | {p['ip']} | {p['os']} "
            f"| {p['status']} | {len(p['findings_high'])} |"
        )

    lines += ["", "## 異常項目彙整", ""]
    high_findings_exist = any(p["findings_high"] for p in parsed)
    if high_findings_exist:
        lines.append("### 🔴 HIGH 等級（需立即處理）")
        lines.append("")
        for p in parsed:
            for f in p["findings_high"]:
                lines.append(f"- **{p['hostname']}**：{f}")
    else:
        lines.append("✅ 本週無 HIGH 等級異常項目。")

    medium_findings = [(p["hostname"], f) for p in parsed for f in p["findings_medium"]]
    if medium_findings:
        lines += ["", "### 🟡 MEDIUM 等級（需於本週處理）", ""]
        for hostname, finding in medium_findings:
            lines.append(f"- **{hostname}**：{finding}")

    lines += [
        "",
        "---",
        f"_本報告由 Ansible Evidence Collector 自動產生 — {datetime.now().isoformat()}_",
    ]

    return "\n".join(lines)


if __name__ == "__main__":
    reports_dir = sys.argv[1] if len(sys.argv) > 1 else "./audit-reports"
    print(generate_summary(reports_dir))
```

---

## 9.4 Eramba GRC 平台概覽

Eramba 是專為 ISO 27001 設計的 GRC（治理、風險、合規）平台，管理 ISMS 所需的文件與流程。

### 9.4.1 Docker 部署

```yaml
# roles/eramba/tasks/main.yml（節錄核心部分）
---
- name: Deploy Eramba GRC platform
  community.docker.docker_container:
    name: eramba
    image: eramba/community:latest
    state: started
    restart_policy: unless-stopped
    published_ports:
      - "8443:443"
    networks:
      - name: eramba_net
    volumes:
      - eramba_data:/var/www/eramba/app/tmp
    env:
      DB_HOST: eramba-mysql
      DB_NAME: eramba
      DB_USER: eramba
      DB_PASS: "{{ vault_eramba_db_password }}"
      APP_URL: "https://{{ ansible_host }}:8443"
```

### 9.4.2 Eramba 與 Ansible 的分工

```
┌─────────────────────────────────────────────────────────────────┐
│  工具分工：Ansible + Eramba 協作架構                             │
│                                                                 │
│  Ansible 負責（技術層）：                                        │
│  ├── 自動部署與設定（第 1-5 章）                                 │
│  ├── CIS Benchmark 掃描（第 7 章）                               │
│  ├── 資產自動註冊 → Snipe-IT（第 6 章）                         │
│  └── 稽核證據收集（第 8 章）→ git push                          │
│                          │                                      │
│                          │ 匯入報告 / API 整合                  │
│                          ▼                                      │
│  Eramba 負責（管理層）：                                         │
│  ├── 資產風險評估（Risk Register）                               │
│  ├── 適用性聲明書（Statement of Applicability, SoA）            │
│  ├── 內部稽核排程管理                                            │
│  ├── 不符合項目追蹤（Non-Conformance Tracking）                  │
│  └── ISO 27001 控制措施進度追蹤                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.4.3 API 整合範例：從 Ansible 上傳稽核結果到 Eramba

```yaml
# roles/evidence_collector/tasks/eramba_upload.yml
# 選用：若已部署 Eramba，可將稽核結果直接推送
---
- name: Upload evidence to Eramba via API
  ansible.builtin.uri:
    url: "{{ eramba_base_url }}/api/v1/audits/{{ eramba_audit_id }}/evidences"
    method: POST
    headers:
      X-API-Key: "{{ vault_eramba_api_key }}"
      Content-Type: "application/json"
    body_format: json
    body:
      title: "CIS Benchmark Scan - {{ inventory_hostname }} - {{ ansible_date_time.date }}"
      description: "自動化 CIS Level 1 合規掃描結果"
      date: "{{ ansible_date_time.date }}"
      # 報告內容以 base64 編碼上傳
      file_content: "{{ lookup('file', evidence_report_path) | b64encode }}"
      file_name: "{{ inventory_hostname }}-cis-audit.md"
    status_code: [200, 201]
  delegate_to: localhost
  when: eramba_base_url is defined
```

---

## 9.5 稽核完整性設計：防竄改機制

### 9.5.1 Protected Branch 設定

```bash
# 在 GitLab UI 設定：
# audit-evidence 專案 → Settings → Repository → Protected Branches
# Branch: main
# Allowed to push: No one（只有 CI/CD bot 透過 Deploy Token 可推送）
# Allowed to merge: Maintainers only
```

### 9.5.2 GPG 簽名（進階）

```bash
# 為 CI bot 建立 GPG 金鑰，所有 commit 都有 GPG 簽名
gpg --batch --gen-key << EOF
Key-Type: ECDSA
Key-Curve: nistp256
Subkey-Type: ECDH
Subkey-Curve: nistp256
Name-Real: Ansible Audit Bot
Name-Email: ansible-bot@example.com
Expire-Date: 1y
%no-protection
%commit
EOF

# 匯出公鑰，加入 GitLab 帳號
gpg --armor --export ansible-bot@example.com

# 在 .gitlab-ci.yml 中加入簽名步驟
# git commit -S -m "..."
```

---

## 9.6 驗證步驟：確認完整閉環運作

```bash
# 1. 手動觸發一次完整的稽核流水線
# GitLab → CI/CD → Schedules → Run Schedule

# 或直接用 API 觸發
curl -X POST \
  --header "PRIVATE-TOKEN: <your-token>" \
  "https://gitlab.example.com/api/v4/projects/<project-id>/pipeline/schedules/<schedule-id>/play"

# 2. 監控 Pipeline 執行狀況
# GitLab → CI/CD → Pipelines → 找到剛觸發的 Pipeline

# 3. 確認 evidence-collect stage 完成
ansible-playbook playbooks/evidence_collector.yml \
  --vault-password-file ~/.vault_pass --check

# 4. 確認 cis-audit stage 完成
ansible-playbook playbooks/cis_audit.yml \
  --vault-password-file ~/.vault_pass --check

# 5. 確認報告已推送到 audit-evidence 倉庫
git clone git@gitlab.example.com:security/audit-evidence-2026.git /tmp/check-evidence
ls /tmp/check-evidence/$(date +%Y-%m-%d)/
# 預期看到：web-01-evidence.md db-01-evidence.md WEEKLY-SUMMARY.md

# 6. 驗證 commit 的完整性（時間線連續）
cd /tmp/check-evidence
git log --oneline --graph
# 預期：每週一都有一個 commit，無間斷

# 7. 確認 Protected Branch 防止手動竄改
git push origin main --force
# 預期：錯誤：protected branch cannot be force-pushed

# 8. 模擬稽核員查閱報告
# 開啟瀏覽器 → GitLab audit-evidence 專案
# Repository → 選擇日期目錄 → 點選 WEEKLY-SUMMARY.md
# 稽核員可以直接在瀏覽器看到格式化的 Markdown 報告
```

---

## 9.7 監控儀表板：Grafana 整合（選用）

若已有 Prometheus + Grafana，可將稽核數據視覺化：

```python
# scripts/export_audit_metrics.py
# 解析 audit-evidence 歷史，產出 Prometheus metrics

from pathlib import Path
import re

metrics = []
for report in Path("audit-evidence-repo").rglob("*-evidence.md"):
    content = report.read_text()
    hostname = report.stem.replace("-evidence", "")
    date = report.parent.name

    high_count = len(re.findall(r"\[HIGH\]", content))
    medium_count = len(re.findall(r"\[MEDIUM\]", content))

    metrics.append(
        f'ansible_audit_findings{{host="{hostname}",severity="HIGH",date="{date}"}} {high_count}'
    )
    metrics.append(
        f'ansible_audit_findings{{host="{hostname}",severity="MEDIUM",date="{date}"}} {medium_count}'
    )

print("\n".join(metrics))
```

```yaml
# Grafana Dashboard 重要 Panel：
# 1. 各主機 HIGH 異常數趨勢（折線圖，觀察是否有惡化趨勢）
# 2. CIS 合規率週趨勢（目標：逐週提升）
# 3. 未套用安全更新的主機數（目標：0）
# 4. 本週異常主機清單（表格，快速識別需關注的主機）
```

---

## 9.8 完整課程（第 6-9 章）小結

| 章節 | 核心產出 | ISO 27001 對應 |
|------|----------|----------------|
| 第 6 章 Snipe-IT | 自動維護的資產清冊 CSV | Annex A 5.9 |
| 第 7 章 CIS Benchmark | 合規率報告 + 自動修復 | A.8.8, A.8.9 |
| 第 8 章 Evidence Collector | 每週稽核 Markdown 報告 | A.5.16, A.8.3, A.8.13 |
| 第 9 章 Scheduled Pipeline | 52 週完整稽核時間線 | A.8.16（持續監控） |

### 最終驗收清單

```
稽核自動化系統驗收：

□ Snipe-IT 正常運行，所有伺服器已自動建立資產記錄
□ CIS Benchmark 掃描可正常執行，產出 JSON + Markdown 報告
□ cis_exceptions.yml 記錄所有例外並有業務理由
□ Evidence Collector 可正常執行並產出所有 8 項稽核收集
□ 報告已正確 git commit 到 audit-evidence 倉庫
□ GitLab Scheduled Pipeline 每週自動執行（無需人工干預）
□ audit-evidence 倉庫已設定 Protected Branch
□ Eramba 已部署（或有替代的 GRC 平台）並與 Ansible 整合
□ 可產出一份「完整 52 週稽核記錄」給外部稽核員審閱
```

---

*← [返回總覽](./README.md) | [上一章](./chapter-08-evidence-collector.md)*
