# 第 17 章：機密掃描（GitLeaks）

> **工具定位：** GitLeaks 是 GitLab CI 的一個 job，Ansible 的角色是在所有開發者機器上部署 **pre-commit hook**（本地端預防），並設定 GitLab Runner 的執行環境。
>
> **ISO 27001 對應：** A.8.10（刪除資訊）、A.5.33（資訊系統過程中的隱私保護）— 防止 API Key、密碼、Token 意外進入版本控制系統。

---

## 17.1 理論說明：為什麼機密洩漏如此常見？

開發者在測試時容易把機密直接寫進程式碼：

```python
# ❌ 最常見的錯誤 - 直接 hardcode
DB_PASSWORD = "mypassword123"
AWS_KEY = "AKIAIOSFODNN7EXAMPLE"
SLACK_WEBHOOK = "https://hooks.slack.com/services/T00/B00/xxx"
```

即使後來用 `git rm` 刪除，**Git history 依然保留**，任何 clone 此 repo 的人都可以找到。

### 17.1.1 防護層次

```
防護層 1（本機）：pre-commit hook
  → commit 時立即掃描，開發者立即知道，最早防堵

防護層 2（CI）：GitLab CI Secret Detection job
  → push 時在 CI 掃描，第二道防線

防護層 3（定期）：掃描整個 Git history
  → 每月對所有 repo 執行 gitleaks，找出歷史遺留問題
```

---

## 17.2 Ansible Role：gitleaks_setup

```bash
ansible-galaxy role init roles/gitleaks_setup
```

### 17.2.1 defaults/main.yml

```yaml
# roles/gitleaks_setup/defaults/main.yml
---
gitleaks_version: "8.18.2"
gitleaks_install_dir: /usr/local/bin
pre_commit_version: "3.7.0"

# 要安裝 pre-commit hook 的使用者（開發者帳號）
gitleaks_dev_users: "{{ active_users | selectattr('role', 'in', ['developer', 'senior-engineer']) | list }}"

# 自訂規則設定路徑
gitleaks_config_path: /etc/gitleaks/.gitleaks.toml
```

### 17.2.2 tasks/main.yml（在開發者機器上執行）

```yaml
# roles/gitleaks_setup/tasks/main.yml
---
# ── 安裝 GitLeaks ────────────────────────────────────────────────────────
- name: Download GitLeaks binary
  ansible.builtin.get_url:
    url: >-
      https://github.com/gitleaks/gitleaks/releases/download/v{{ gitleaks_version }}/
      gitleaks_{{ gitleaks_version }}_linux_x64.tar.gz
    dest: /tmp/gitleaks.tar.gz
    mode: '0644'

- name: Install GitLeaks
  ansible.builtin.unarchive:
    src: /tmp/gitleaks.tar.gz
    dest: "{{ gitleaks_install_dir }}"
    remote_src: true
    include:
      - gitleaks
    mode: '0755'

# ── 部署自訂規則設定 ─────────────────────────────────────────────────────
- name: Create gitleaks config directory
  ansible.builtin.file:
    path: /etc/gitleaks
    state: directory
    mode: '0755'

- name: Deploy custom gitleaks rules
  ansible.builtin.copy:
    content: |
      # .gitleaks.toml - 由 Ansible 管理
      title = "Company GitLeaks Rules"

      # 繼承預設規則
      [extend]
      useDefault = true

      # 公司自訂規則：偵測內部 API Token 格式
      [[rules]]
      id = "company-api-token"
      description = "Company Internal API Token"
      regex = '''cycraft-[a-zA-Z0-9]{32}'''
      tags = ["api", "company"]

      # 忽略已知的測試用假金鑰
      [allowlist]
      description = "Allowlisted test values"
      regexes = [
        '''EXAMPLE_KEY_DO_NOT_USE''',
        '''test_token_placeholder'''
      ]
      paths = [
        '''tests/fixtures/''',
        '''docs/examples/'''
      ]
    dest: "{{ gitleaks_config_path }}"
    mode: '0644'

# ── 安裝 pre-commit 並設定 hook ──────────────────────────────────────────
- name: Install pre-commit for each developer
  ansible.builtin.pip:
    name: pre-commit
    version: "{{ pre_commit_version }}"
    state: present

- name: Create global pre-commit config
  ansible.builtin.copy:
    content: |
      repos:
        - repo: https://github.com/gitleaks/gitleaks
          rev: v{{ gitleaks_version }}
          hooks:
            - id: gitleaks
              args: ["--config={{ gitleaks_config_path }}"]
    dest: /etc/pre-commit-config.yaml
    mode: '0644'

- name: Install pre-commit hook for each developer's global git config
  ansible.builtin.command:
    cmd: >
      sudo -u {{ item.username }}
      git config --global core.hooksPath /etc/git-hooks
  loop: "{{ gitleaks_dev_users }}"
  loop_control:
    label: "{{ item.username }}"
  changed_when: true

- name: Create global git hooks directory
  ansible.builtin.file:
    path: /etc/git-hooks
    state: directory
    mode: '0755'

- name: Create pre-commit hook script
  ansible.builtin.copy:
    content: |
      #!/bin/bash
      # Global pre-commit hook - 由 Ansible 管理
      gitleaks protect \
        --config {{ gitleaks_config_path }} \
        --staged \
        --verbose
    dest: /etc/git-hooks/pre-commit
    mode: '0755'
```

---

## 17.3 GitLab CI 整合

GitLab Ultimate 版內建 Secret Detection，Community 版使用 GitLeaks job：

```yaml
# .gitlab-ci.yml
stages:
  - secret-scan   # 在最前面，越早發現越好
  - validate
  - ...

# ── 每次 push 掃描最新 commit ────────────────────────────────────────────
secret-detection:
  stage: secret-scan
  image:
    name: "zricethezav/gitleaks:v{{ gitleaks_version }}"
    entrypoint: [""]
  script:
    - gitleaks detect
        --config .gitleaks.toml
        --source .
        --log-opts "HEAD~1..HEAD"
        --verbose
        --exit-code 1
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  allow_failure: false   # 發現機密 → 阻擋合併

# ── 每月掃描整個 Git history ─────────────────────────────────────────────
secret-scan-full-history:
  stage: secret-scan
  image: "zricethezav/gitleaks:v{{ gitleaks_version }}"
  script:
    - gitleaks detect
        --config .gitleaks.toml
        --source .
        --verbose
        --report-format json
        --report-path gitleaks-history-report.json
        --exit-code 0    # 歷史掃描不 blocking，但記錄
  artifacts:
    when: always
    paths:
      - gitleaks-history-report.json
    expire_in: 1 year
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "monthly_dr"
```

---

## 17.4 驗證步驟

```bash
# 1. 安裝 GitLeaks 到開發者機器
ansible-playbook playbooks/site.yml \
  --vault-password-file ~/.vault_pass \
  --tags gitleaks \
  --limit webservers   # 或指定開發者機器

# 2. 測試 pre-commit hook（故意加入假 API key）
cd /tmp/test-repo && git init
echo 'API_KEY = "AKIAIOSFODNN7EXAMPLE"' > test.py
git add test.py
git commit -m "test"   # 應被 pre-commit hook 阻擋

# 3. 確認 GitLeaks binary 安裝正確
gitleaks version

# 4. 手動掃描現有 repo
gitleaks detect --config /etc/gitleaks/.gitleaks.toml \
  --source /path/to/repo --verbose

# 5. 查看 CI pipeline 中的掃描結果
# GitLab → MR → Security → Secret Detection
```

---

## 17.5 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 安裝 GitLeaks binary、部署自訂規則、設定開發者 pre-commit hook |
| **GitLab CI 負責** | 每次 push 觸發掃描、MR 阻擋、歷史掃描記錄 |
| **pre-commit 負責** | 在本地端 commit 時第一時間阻擋 |

*← [返回總覽](./README.md) | [上一章](./chapter-16-hashicorp-vault-secret-rotation.md)*
