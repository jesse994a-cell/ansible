# 第 13 章：容器映像掃描（Trivy）

> **工具定位：** Trivy 是掃描引擎，GitLab CI 是觸發平台。**Ansible 的角色**是將 Trivy 安裝到 GitLab Runner 環境、部署掃描結果收集服務，並將歷史報告納入 audit-evidence 管理。
>
> **ISO 27001 對應：** A.8.25（安全開發生命週期）、A.8.28（安全編碼）、A.8.8（技術漏洞管理）

---

## 13.1 理論說明：軟體供應鏈攻擊與容器映像風險

### 13.1.1 為什麼要掃描容器映像？

容器映像由**基礎映像層**疊加而成。即使你自己的程式碼沒有漏洞，底層的 `ubuntu:22.04` 或 `python:3.12-slim` 可能包含已知 CVE（Common Vulnerabilities and Exposures）。

```
你的 Dockerfile：
  FROM python:3.12-slim          ← 可能含有 CVE-2024-XXXX
  COPY ./app /app                ← 你的程式碼
  RUN pip install flask==2.3.0   ← flask 依賴可能有漏洞

Trivy 掃描範圍：
  ├── OS 層（Debian packages）
  ├── Language packages（pip, npm, gem）
  ├── Dockerfile 設定問題（如 root 執行）
  └── Secrets（誤 commit 的 API key）
```

### 13.1.2 ISO 27001 稽核場景

稽核員：「你們如何確保部署的容器沒有已知漏洞？」

**有 Trivy 的答覆：** 「每次 build 後 Trivy 自動掃描，CRITICAL CVE 阻擋合併。每週對 production 所有映像重新掃描，報告保存在 audit-evidence 倉庫，這是過去 12 個月的記錄。」

---

## 13.2 Ansible Role：trivy_setup

Ansible 負責在 GitLab Runner 主機上安裝並更新 Trivy。

```bash
ansible-galaxy role init roles/trivy_setup
```

### 13.2.1 defaults/main.yml

```yaml
# roles/trivy_setup/defaults/main.yml
---
trivy_version: "0.52.0"
trivy_install_dir: /usr/local/bin
trivy_db_dir: /var/lib/trivy

# 掃描政策
trivy_exit_code_critical: 1   # CRITICAL → CI fail
trivy_exit_code_high: 0       # HIGH → 警告但不 fail（可依政策調整）

# 忽略未修復的 CVE（實務上很多 CVE 短期內沒有 patch）
trivy_ignore_unfixed: true

# 掃描結果格式
trivy_output_format: "table"   # table / json / sarif
```

### 13.2.2 tasks/main.yml

```yaml
# roles/trivy_setup/tasks/main.yml
---
- name: Create Trivy directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ trivy_db_dir }}"

- name: Download Trivy binary
  ansible.builtin.get_url:
    url: >-
      https://github.com/aquasecurity/trivy/releases/download/v{{ trivy_version }}/
      trivy_{{ trivy_version }}_Linux-64bit.tar.gz
    dest: /tmp/trivy.tar.gz
    mode: '0644'

- name: Extract Trivy
  ansible.builtin.unarchive:
    src: /tmp/trivy.tar.gz
    dest: "{{ trivy_install_dir }}"
    remote_src: true
    include:
      - trivy
    mode: '0755'

- name: Verify Trivy installation
  ansible.builtin.command:
    cmd: trivy --version
  register: trivy_version_check
  changed_when: false

- name: Pre-download vulnerability database
  ansible.builtin.command:
    cmd: trivy image --download-db-only --cache-dir {{ trivy_db_dir }}
  changed_when: false
  # 更新 DB 可能失敗（網路問題），不阻斷後續
  ignore_errors: true

- name: Setup Trivy DB update cron
  ansible.builtin.cron:
    name: "trivy db update"
    hour: "1"
    minute: "0"
    # 每天更新漏洞資料庫
    job: "trivy image --download-db-only --cache-dir {{ trivy_db_dir }} >> /var/log/trivy-db-update.log 2>&1"
    state: present
```

---

## 13.3 GitLab CI 整合（核心掃描邏輯）

GitLab CI 是 Trivy 掃描的主要執行環境。以下是完整的 CI 設定：

```yaml
# .gitlab-ci.yml 新增 container-scan stage
stages:
  - build
  - container-scan    # 新增
  - deploy

# ── 每次 build 後立即掃描 ────────────────────────────────────────────────
trivy-scan-on-build:
  stage: container-scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    # 使用 GitLab Container Registry 的快取
    TRIVY_CACHE_DIR: ".trivycache/"
    GIT_STRATEGY: none
  cache:
    paths:
      - .trivycache/
  script:
    # 掃描剛 build 的映像
    - trivy image
        --cache-dir $TRIVY_CACHE_DIR
        --exit-code {{ trivy_exit_code_critical }}
        --severity CRITICAL
        {{ '--ignore-unfixed' if trivy_ignore_unfixed else '' }}
        --format table
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    # 同時輸出 JSON 供 artifact 保存
    - trivy image
        --cache-dir $TRIVY_CACHE_DIR
        --exit-code 0
        --severity CRITICAL,HIGH,MEDIUM
        --format json
        --output trivy-report.json
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts:
    when: always
    paths:
      - trivy-report.json
    expire_in: 1 year
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# ── 每週對 production 所有映像重新掃描 ──────────────────────────────────
# （基礎映像可能在本週出現新 CVE）
trivy-weekly-production-scan:
  stage: container-scan
  extends: .ansible_base
  script:
    - |
      # 取得目前 production 運行中的所有映像
      IMAGES=$(ansible all -i inventory/production/ \
        -m community.docker.docker_host_info \
        | grep "RepoTags" | grep -v "none" | awk '{print $2}')

      # 對每個映像執行掃描
      for IMAGE in $IMAGES; do
        echo "=== Scanning $IMAGE ==="
        trivy image \
          --exit-code 0 \
          --severity CRITICAL,HIGH \
          --format json \
          --output "trivy-prod-$(echo $IMAGE | tr '/:' '-').json" \
          $IMAGE
      done
    # 產生彙整報告
    - python3 scripts/merge_trivy_reports.py trivy-prod-*.json > trivy-weekly-summary.md
    # commit 到 audit-evidence
    - git clone $AUDIT_EVIDENCE_REPO_URL /tmp/audit-evidence
    - cp trivy-weekly-summary.md /tmp/audit-evidence/$(date +%Y-%m-%d)/trivy-scan.md
    - cd /tmp/audit-evidence
    - git add . && git commit -m "trivy($(date +%Y-%m-%d)): weekly production image scan"
    - git push
  artifacts:
    when: always
    paths:
      - trivy-*.json
      - trivy-weekly-summary.md
    expire_in: 1 year
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "weekly_audit"
```

---

## 13.4 彙整報告腳本

```python
# scripts/merge_trivy_reports.py
"""將多個 Trivy JSON 報告合併為 Markdown 摘要"""
import json, sys
from pathlib import Path

def parse_trivy_json(filepath):
    with open(filepath) as f:
        data = json.load(f)
    results = data.get("Results", [])
    vulns = []
    for r in results:
        for v in r.get("Vulnerabilities", []) or []:
            vulns.append({
                "image": data.get("ArtifactName", filepath),
                "pkg": v.get("PkgName"),
                "cve": v.get("VulnerabilityID"),
                "severity": v.get("Severity"),
                "installed": v.get("InstalledVersion"),
                "fixed": v.get("FixedVersion", "未修復"),
                "title": v.get("Title", "")[:80],
            })
    return vulns

all_vulns = []
for f in sys.argv[1:]:
    all_vulns.extend(parse_trivy_json(f))

critical = [v for v in all_vulns if v["severity"] == "CRITICAL"]
high     = [v for v in all_vulns if v["severity"] == "HIGH"]

print(f"# Trivy 容器映像掃描週報\n")
print(f"| 等級 | 數量 |")
print(f"|------|------|")
print(f"| CRITICAL | {len(critical)} |")
print(f"| HIGH | {len(high)} |")
print(f"\n## CRITICAL 漏洞\n")
if critical:
    print("| 映像 | 套件 | CVE | 已安裝版本 | 修復版本 |")
    print("|------|------|-----|-----------|---------|")
    for v in critical:
        print(f"| {v['image'][:40]} | {v['pkg']} | [{v['cve']}](https://nvd.nist.gov/vuln/detail/{v['cve']}) | {v['installed']} | {v['fixed']} |")
else:
    print("✅ 無 CRITICAL 漏洞")
```

---

## 13.5 驗證步驟

```bash
# 1. 安裝 Trivy 到 GitLab Runner 主機
ansible-playbook playbooks/site.yml \
  --vault-password-file ~/.vault_pass \
  --tags trivy

# 2. 手動執行掃描測試
trivy image ubuntu:24.04 --severity CRITICAL,HIGH

# 3. 確認 CI pipeline 中掃描 job 正常執行
# GitLab → CI/CD → Pipelines → 找最近的 pipeline → trivy-scan-on-build

# 4. 查看 JSON 報告
cat trivy-report.json | python3 -m json.tool | grep -E '"Severity"|"VulnerabilityID"' | head -20

# 5. 測試 CRITICAL CVE 阻擋合併（使用已知有漏洞的映像）
trivy image python:3.6-slim --severity CRITICAL --exit-code 1
# 預期：回傳 exit code 1，列出 CRITICAL CVE
```

---

## 13.6 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 在 Runner 主機安裝 Trivy、維護漏洞 DB 更新 cron |
| **GitLab CI 負責** | 每次 build 觸發掃描、CRITICAL blocking、報告 artifact 保存 |
| **audit-evidence 負責** | 長期保存每週掃描記錄，供稽核員查閱 |

*← [返回總覽](./README.md) | [上一章](./chapter-12-backup-dr-validation.md)*
