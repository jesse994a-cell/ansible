# 第 7 章：CIS Benchmark 合規掃描（Ansible Lockdown）

> **學習目標：** 使用 Ansible Lockdown 的 UBUNTU24-CIS role，對 Ubuntu 24.04 執行 CIS Benchmark Level 1/2 的掃描與自動修復。掌握 audit-only 模式（產出差距報告）與 remediate 模式（自動修復），並整合進 GitLab CI 的 manual gate 流程。

---

## 7.1 理論說明：什麼是 CIS Benchmark？

### 7.1.1 CIS 與 ISO 27001 的關係

**CIS（Center for Internet Security）Benchmark** 是全球公認最嚴格的系統安全配置標準，由資安社群持續維護。ISO 27001 的稽核員在驗證「Annex A 8.8 技術漏洞管理」與「8.9 組態管理」時，**引用 CIS Benchmark 合規報告是最有力的技術證據**。

```
ISO 27001 稽核要求                 CIS Benchmark 提供的證據
─────────────────────────────────────────────────────────────
A.8.8 技術漏洞管理          ←  掃描報告顯示哪些 CVE/設定已處理
A.8.9 組態管理              ←  報告顯示系統設定符合 CIS Level 1/2
A.8.2 特殊存取權限管理      ←  報告涵蓋 sudo/suid 設定檢查
A.8.3 資訊存取限制          ←  報告涵蓋 filesystem permission 檢查
```

### 7.1.2 CIS Level 1 vs Level 2

| 等級 | 適用場景 | 影響程度 | 主要控制項 |
|------|----------|----------|----------|
| **Level 1** | 所有伺服器（基礎線） | 低，不影響正常功能 | SSH 設定、密碼政策、日誌記錄 |
| **Level 2** | 高安全需求環境 | 中，部分功能受限 | SELinux/AppArmor、kernel 參數、掛載選項 |

### 7.1.3 Ansible Lockdown 專案架構

```
GitHub: ansible-lockdown/UBUNTU24-CIS
       │
       ├── tasks/
       │   ├── main.yml          ← 根據 ubuntu24cis_section* 開關決定執行哪些 section
       │   ├── section1/         ← 1.x 系統初始設定（分割槽、套件）
       │   ├── section2/         ← 2.x 服務設定
       │   ├── section3/         ← 3.x 網路設定
       │   ├── section4/         ← 4.x 日誌與稽核
       │   ├── section5/         ← 5.x 存取控制
       │   └── section6/         ← 6.x 系統維護
       │
       ├── vars/
       │   └── main.yml          ← 所有 CIS 控制項的開關變數
       │
       └── AUDIT/                ← GOSS 審計引擎的設定與測試
           ├── goss.yml
           └── goss_vars/
```

---

## 7.2 安裝 Ansible Lockdown

### 7.2.1 透過 ansible-galaxy 安裝

```bash
# 方法一：直接從 GitHub 安裝（取得最新版本）
ansible-galaxy role install git+https://github.com/ansible-lockdown/UBUNTU24-CIS.git,main \
  -p roles/ \
  --role-file requirements.yml

# 方法二：加入 requirements.yml（推薦，版本可控）
```

在 `requirements.yml` 中加入：

```yaml
# requirements.yml（在現有內容後新增）
roles:
  - name: UBUNTU24-CIS
    src: https://github.com/ansible-lockdown/UBUNTU24-CIS
    version: main     # 建議改為特定 tag，如 "v1.0.0"
    scm: git
```

```bash
# 安裝所有 requirements
ansible-galaxy role install -r requirements.yml -p roles/

# 驗證安裝
ls roles/UBUNTU24-CIS/
```

### 7.2.2 安裝 GOSS（審計引擎）

```bash
# GOSS 是一個輕量的 YAML-driven 系統驗證工具
# Ansible Lockdown 的 AUDIT 模式會在目標主機上執行 GOSS

# 手動下載 GOSS（Ansible 會自動處理，這裡是手動驗證）
curl -fsSL https://github.com/goss-org/goss/releases/latest/download/goss-linux-amd64 \
  -o /usr/local/bin/goss
chmod +x /usr/local/bin/goss
goss --version
```

---

## 7.3 Audit 模式：掃描並產出差距報告

Audit 模式**只掃描，不修改系統**，適合：
- 了解現有系統的合規狀態
- 為稽核員產出「現況差距分析」報告
- 在修復前後對比結果

### 7.3.1 建立 cis_audit Role 包裝器

```bash
ansible-galaxy role init roles/cis_audit
```

```yaml
# roles/cis_audit/defaults/main.yml
---
# ── GOSS 設定 ────────────────────────────────
goss_version: "0.4.7"
goss_install_dir: /usr/local/bin
goss_audit_dir: /var/lib/goss_audit

# ── 報告設定 ─────────────────────────────────
audit_report_dir: /opt/audit-reports
audit_report_format: json       # json 或 documentation
audit_timestamp: "{{ ansible_date_time.iso8601_basic_short }}"
audit_report_filename: "cis-audit-{{ inventory_hostname }}-{{ audit_timestamp }}.json"

# ── CIS Level 設定 ────────────────────────────
cis_level: 1                   # 1 或 2（Level 2 更嚴格）

# ── 要套用的 Section（可選擇性關閉不適用的項目）
ubuntu24cis_section1: true     # 系統初始設定
ubuntu24cis_section2: true     # 服務
ubuntu24cis_section3: true     # 網路
ubuntu24cis_section4: true     # 日誌與稽核
ubuntu24cis_section5: true     # 存取控制
ubuntu24cis_section6: true     # 系統維護
```

### 7.3.2 Audit Playbook

```yaml
# playbooks/cis_audit.yml
---
- name: CIS Ubuntu 24.04 Benchmark - Audit Mode
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml

  vars:
    # ── Audit 模式關鍵開關 ──────────────────────
    # setup_audit=true：部署 GOSS 並執行掃描
    setup_audit: true
    # run_audit=true：執行 GOSS 掃描
    run_audit: true
    # remediation=false：純掃描，不修改系統
    remediation: false

    # ── 掃描的 CIS Level ────────────────────────
    cis_level: "{{ cis_level | default(1) }}"

    # ── GOSS 輸出格式 ────────────────────────────
    audit_format: json

    # ── 報告輸出路徑 ─────────────────────────────
    audit_out_dir: "{{ audit_report_dir | default('/opt/audit-reports') }}"

  roles:
    - role: UBUNTU24-CIS
      tags: [cis, audit]

  post_tasks:
    # ── 將報告抓回 Control Node ─────────────────────────────────────────
    - name: Create local report directory on control node
      ansible.builtin.file:
        path: "./audit-reports/{{ ansible_date_time.date }}"
        state: directory
        mode: '0755'
      delegate_to: localhost
      run_once: false

    - name: Fetch audit report to control node
      ansible.builtin.fetch:
        src: "{{ audit_out_dir }}/{{ audit_report_filename | default('audit.json') }}"
        dest: "./audit-reports/{{ ansible_date_time.date }}/{{ inventory_hostname }}.json"
        flat: true
      ignore_errors: true

    - name: Convert JSON report to Markdown summary
      ansible.builtin.shell: |
        python3 << 'PYEOF'
        import json, sys
        from datetime import datetime

        report_file = "{{ audit_out_dir }}/{{ audit_report_filename | default('audit.json') }}"
        try:
            with open(report_file) as f:
                data = json.load(f)
        except:
            print("無法讀取報告檔案")
            sys.exit(1)

        total     = data.get("summary", {}).get("test-count", 0)
        passed    = data.get("summary", {}).get("summary-line", "").count("Successful")
        failed    = data.get("summary", {}).get("failed", 0)
        skipped   = data.get("summary", {}).get("skipped", 0)
        pass_rate = round((total - failed) / total * 100, 1) if total > 0 else 0

        md = f"""# CIS Ubuntu 24.04 Benchmark 掃描報告

        **主機：** {{ inventory_hostname }}
        **掃描時間：** {datetime.now().isoformat()}
        **CIS Level：** {{ cis_level }}
        **Ansible 執行者：** {{ ansible_user_id }}

        ## 總覽

        | 指標 | 數值 |
        |------|------|
        | 總控制項數 | {total} |
        | 通過 | {total - failed} |
        | 失敗 | {failed} |
        | 略過 | {skipped} |
        | **合規率** | **{pass_rate}%** |

        ## 失敗項目清單
        """

        results = data.get("results", {})
        failures = [k for k, v in results.items() if not v.get("successful", True)]
        if failures:
            for f in failures[:50]:  # 最多顯示 50 項
                md += f"\n- `{f}`"
        else:
            md += "\n✅ 所有控制項均通過！"

        print(md)
        PYEOF
      register: md_report
      changed_when: false
      ignore_errors: true

    - name: Save Markdown report
      ansible.builtin.copy:
        content: "{{ md_report.stdout }}"
        dest: "{{ audit_out_dir }}/cis-summary-{{ inventory_hostname }}-{{ audit_timestamp }}.md"
      when: md_report.rc == 0

    - name: Fetch Markdown report to control node
      ansible.builtin.fetch:
        src: "{{ audit_out_dir }}/cis-summary-{{ inventory_hostname }}-{{ audit_timestamp }}.md"
        dest: "./audit-reports/{{ ansible_date_time.date }}/{{ inventory_hostname }}.md"
        flat: true
      ignore_errors: true
```

---

## 7.4 Remediate 模式：自動修復不符項目

Remediate 模式會實際修改系統設定以符合 CIS 標準。**建議只在新機器初始化時自動執行，對既有機器謹慎評估影響後再執行。**

### 7.4.1 理解 CIS 控制項開關

UBUNTU24-CIS role 的每個控制項都有對應的布林變數，可以個別關閉：

```yaml
# 範例：關閉對環境有衝突的控制項
ubuntu24cis_rule_1_1_1_1: true    # 掛載 /tmp 為獨立分割槽（預設 true）
ubuntu24cis_rule_2_1_1: true      # 禁用 xinetd（預設 true）

# 常見需要關閉的項目（視環境而定）
ubuntu24cis_rule_3_3_9: false     # IPv6 禁用（若環境需要 IPv6 則關閉）
ubuntu24cis_ipv6_required: true   # 保留 IPv6（與上面搭配）
```

### 7.4.2 Remediate Playbook

```yaml
# playbooks/cis_remediate.yml
---
- name: CIS Ubuntu 24.04 Benchmark - Remediate Mode
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: true
  become: true

  vars_files:
    - ../vault/secrets.yml
    - ./cis_exceptions.yml    # 環境專屬的例外清單

  vars:
    # ── Remediate 模式 ──────────────────────────
    setup_audit: true          # 修復前先掃描，取得 baseline
    run_audit: true
    remediation: true          # ← 關鍵！開啟修復模式

    cis_level: 1               # 先從 Level 1 開始

    # ── 必須調整的項目（避免服務中斷）──────────
    # 若環境是 Docker host，部分網路設定需要保留
    ubuntu24cis_rule_3_3_1: "{{ not is_docker_host | default(false) }}"
    ubuntu24cis_rule_3_3_2: "{{ not is_docker_host | default(false) }}"

    # SSH Port 若已改為非 22，需告知 role
    ubuntu24cis_sshd_port: "{{ ssh_port | default(22) }}"

  roles:
    - role: UBUNTU24-CIS
      tags: [cis, remediate]

  post_tasks:
    - name: Run post-remediation audit
      ansible.builtin.include_role:
        name: UBUNTU24-CIS
      vars:
        remediation: false
        run_audit: true
        setup_audit: false    # GOSS 已安裝，跳過
      tags: [cis, post-audit]

    - name: Display remediation result
      ansible.builtin.debug:
        msg:
          - "修復完成！建議重新執行 audit 模式確認合規率提升。"
          - "注意：部分項目（如 kernel 參數）需要重開機才生效。"
```

### 7.4.3 cis_exceptions.yml：環境例外清單

```yaml
# playbooks/cis_exceptions.yml
# 記錄環境中刻意不符合的 CIS 項目，並說明業務理由
# 這份文件本身也是 ISO 27001 稽核的「風險接受」證據
---

# ── Section 1：系統初始設定 ──────────────────────────────────
# CIS 1.1.1.1：建議 /tmp 獨立分割槽
# 業務理由：現有 VM 範本不支援額外分割槽，已透過 tmpfs 掛載實現隔離
ubuntu24cis_rule_1_1_1_1: false

# ── Section 3：網路設定 ──────────────────────────────────────
# CIS 3.3.9：建議禁用 IPv6
# 業務理由：內部監控系統使用 IPv6，禁用會導致服務中斷
ubuntu24cis_rule_3_3_9: false
ubuntu24cis_ipv6_required: true

# ── Section 4：日誌設定 ──────────────────────────────────────
# 無例外

# ── Section 5：存取控制 ──────────────────────────────────────
# 無例外

# ═══════════════════════════════════════════════════════════
# 例外追蹤記錄（供稽核員查閱）
# ═══════════════════════════════════════════════════════════
# 每個例外必須包含：
#   - 違反的控制項編號
#   - 業務理由
#   - 補償控制措施
#   - 負責人
#   - 複審日期
cis_exceptions_log:
  - rule: "1.1.1.1"
    reason: "VM 範本架構限制，無法新增分割槽"
    compensating_control: "使用 noexec,nosuid 掛載 tmpfs /tmp，效果等同"
    owner: "infra-team"
    review_date: "2027-01-01"
  - rule: "3.3.9"
    reason: "監控系統依賴 IPv6"
    compensating_control: "防火牆已封鎖外部 IPv6 連線，僅允許內網"
    owner: "network-team"
    review_date: "2027-01-01"
```

---

## 7.5 整合進 GitLab CI：Manual Gate

```yaml
# .gitlab-ci.yml 新增 CIS 相關 stages
stages:
  - validate
  - test
  - cis-audit       # 新增：CIS 掃描
  - cis-remediate   # 新增：CIS 修復（需人工確認）
  - dry-run
  - deploy

# ── CIS Audit：定期或手動觸發 ────────────────────────────────────────────
cis-audit-staging:
  stage: cis-audit
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/cis_audit.yml
      -i inventory/staging/
      --vault-password-file /tmp/.vault_pass
      -e "target_hosts=all cis_level=1"
  artifacts:
    # 將掃描報告作為 CI artifacts 保存
    when: always
    paths:
      - audit-reports/
    expire_in: 1 year   # 保留 1 年（符合稽核保存期限要求）
  rules:
    # 每週一 02:00 自動執行
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "cis_audit"
    # 也允許手動觸發
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual

# ── CIS Remediate：必須人工確認才執行 ───────────────────────────────────
cis-remediate-staging:
  stage: cis-remediate
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/cis_remediate.yml
      -i inventory/staging/
      --vault-password-file /tmp/.vault_pass
      -e "target_hosts=all cis_level=1"
  rules:
    # 嚴格限制：只允許在 main branch 手動觸發
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: staging/cis-hardening

cis-remediate-production:
  stage: cis-remediate
  extends: .ansible_base
  script:
    - >
      ansible-playbook playbooks/cis_remediate.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -e "target_hosts=all cis_level=1"
      --limit "{{ TARGET_HOST | default('') }}"  # 可限制特定主機
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  # 允許部分失敗（例如已是合規狀態的主機）
  allow_failure: false
  environment:
    name: production/cis-hardening
```

---

## 7.6 解讀掃描報告

### 7.6.1 JSON 報告結構

```json
{
  "results": {
    "1.1.1.1 Ensure /tmp is a separate partition": {
      "successful": false,
      "summary-line": "Count: 1, Failed: 1, Skipped: 0",
      "test-count": 1,
      "failed-count": 1
    },
    "1.3.1 Ensure AIDE is installed": {
      "successful": true,
      "summary-line": "Count: 1, Failed: 0, Skipped: 0"
    }
  },
  "summary": {
    "test-count": 245,
    "failed": 23,
    "skipped": 12
  }
}
```

### 7.6.2 解讀合規率

```
合規率 = (總控制項 - 失敗項目) / 總控制項 × 100%

Level 1 目標：≥ 80%（初期），最終目標 ≥ 95%
Level 2 目標：≥ 70%（初期），最終目標 ≥ 85%

常見失敗原因：
- 1.x 分割槽相關：VM 範本限制，需加入例外清單
- 2.x 服務相關：Docker 需要部分服務，需加入例外清單
- 5.x 密碼政策：PAM 設定，remediate 模式可自動修復
```

---

## 7.7 驗證步驟

```bash
# 1. 首次執行 Audit（了解現況）
ansible-playbook playbooks/cis_audit.yml \
  --vault-password-file ~/.vault_pass \
  -e "target_hosts=web-01 cis_level=1" \
  -v

# 2. 查看掃描結果
cat audit-reports/$(date +%Y-%m-%d)/web-01.md

# 3. 執行 Remediation（從 Level 1 開始）
ansible-playbook playbooks/cis_remediate.yml \
  --vault-password-file ~/.vault_pass \
  -e "target_hosts=web-01" \
  --check --diff   # 先用 dry-run 確認影響範圍

# 4. 確認無問題後執行實際修復
ansible-playbook playbooks/cis_remediate.yml \
  --vault-password-file ~/.vault_pass \
  -e "target_hosts=web-01"

# 5. 修復後重跑 Audit 確認合規率提升
ansible-playbook playbooks/cis_audit.yml \
  --vault-password-file ~/.vault_pass \
  -e "target_hosts=web-01"

# 比較前後差異
diff \
  audit-reports/before/web-01.json \
  audit-reports/after/web-01.json

# 6. 確認報告已作為 CI artifact 保存
# GitLab UI：Job → Browse Artifacts → audit-reports/
```

---

## 7.8 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| 理解 CIS Level 1/2 差異 | ☐ 能說明兩者適用場景與影響範圍 |
| Ansible Lockdown 安裝 | ☐ `roles/UBUNTU24-CIS/` 目錄存在 |
| Audit 模式掃描並產出報告 | ☐ 產出 JSON + Markdown 報告 |
| 解讀合規率與失敗項目 | ☐ 能識別哪些失敗可以接受（業務理由） |
| 例外清單（cis_exceptions.yml）| ☐ 每個例外有業務理由與補償控制措施 |
| GitLab CI Manual Gate 整合 | ☐ Remediate job 需要人工確認才執行 |

**下一章：** [第 8 章 自動化稽核證據收集器](./chapter-08-evidence-collector.md) — 撰寫完整的 Evidence Collector Playbook，自動收集帳號、套件、備份、權限等稽核素材。

---

*← [返回總覽](./README.md) | [上一章](./chapter-06-asset-management.md)*
