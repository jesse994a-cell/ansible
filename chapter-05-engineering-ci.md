# 第 5 章：工程化與 CI Integration

> **學習目標：** 將 Ansible 專案從「能跑就好」提升到「工程級品質」——導入 ansible-lint 程式碼規範、Molecule 自動化測試框架，最終透過 GitLab CI/CD 建立完整的端到端自動化流程。

---

## 5.1 理論說明：為什麼需要工程化？

### 5.1.1 沒有測試的 Ansible 專案的風險

```
沒有測試的典型問題：

  開發者修改了 role A
       │
       ▼
  直接推送到 main branch
       │
       ▼
  CI/CD 觸發，自動 apply 到正式環境
       │
       ▼
  role A 的改動影響了 role B（依賴衝突）
       │
       ▼
  生產服務中斷 🔥
```

### 5.1.2 工程化後的流程

```
開發者修改了 role A
       │
       ▼
  ansible-lint 檢查（語法、最佳實踐）
       │
       ▼
  Molecule 在 Docker 容器中測試（隔離）
       │
       ▼  通過
  Merge Request Review
       │
       ▼  approve
  merge 到 main
       │
       ▼
  GitLab CI --check（dry run，不實際變更）
       │
       ▼  確認無誤
  手動觸發 → 部署到正式環境
```

### 5.1.3 工具職責分工

| 工具 | 職責 | 類比 |
|------|------|------|
| **ansible-lint** | 靜態分析，檢查語法與最佳實踐 | ESLint / flake8 |
| **Molecule** | 動態測試，在隔離環境實際執行並驗證 | Jest / pytest |
| **GitLab CI** | 自動化觸發上述工具，整合到 PR/MR 流程 | GitHub Actions / Jenkins |

---

## 5.2 ansible-lint：程式碼品質守門員

### 5.2.1 安裝

```bash
# 已在第 1 章透過 pipx 安裝，確認版本
ansible-lint --version
# 預期：ansible-lint X.X.X using ansible-core X.X.X
```

### 5.2.2 設定檔 .ansible-lint

在專案根目錄建立 `.ansible-lint`，定義規則：

```yaml
# .ansible-lint
---
# 設定檔版本
profile: production  # 使用最嚴格的規則集（min/basic/moderate/safety/shared/production）

# 要忽略的路徑（不進行 lint 的目錄）
exclude_paths:
  - .cache/
  - .git/
  - molecule/

# 要跳過的規則 ID（謹慎使用，需附上理由）
skip_list:
  # - yaml[line-length]  # 範例：跳過行長度限制

# 要強制執行的額外規則（預設關閉）
enable_list:
  - no-changed-when         # 所有 command/shell task 都必須有 changed_when
  - no-free-form            # 禁止 free-form 模組寫法（強制使用 key=value 或 YAML 格式）

# 對特定規則使用警告而非錯誤
warn_list:
  - experimental
  - role-name

# 設定 loop 的 loop_var 命名規範
loop_var_prefix: ^(item|my_.+|loop_.+)$

# 任務名稱必須符合的格式（大寫開頭）
task_name_prefix: "{stem} | "

# 允許的 YAML 格式（使用 yamllint）
yamllint:
  extends: default
  rules:
    line-length:
      max: 160  # 放寬到 160 字元（Ansible YAML 常有較長的行）
      level: warning
    truthy:
      allowed-values: ["true", "false"]
```

### 5.2.3 常用 lint 指令

```bash
# 檢查整個專案
ansible-lint

# 檢查特定 playbook
ansible-lint playbooks/site.yml

# 檢查特定 role
ansible-lint roles/kernel_tuning/

# 列出所有可用規則
ansible-lint --list-rules

# 列出所有 tag（規則分類）
ansible-lint --list-tags

# 自動修復部分問題（如 YAML 格式）
ansible-lint --fix

# 輸出 JSON 格式（適合 CI 解析）
ansible-lint --format json

# 顯示詳細輸出
ansible-lint -v
```

### 5.2.4 常見 lint 錯誤與修正

**錯誤 1：command-instead-of-module**

```yaml
# ❌ 錯誤：用 command 執行有對應模組的操作
- name: Create directory
  ansible.builtin.command: mkdir -p /tmp/mydir

# ✅ 正確：使用 file 模組
- name: Create directory
  ansible.builtin.file:
    path: /tmp/mydir
    state: directory
    mode: '0755'
```

**錯誤 2：no-changed-when**

```yaml
# ❌ 錯誤：command task 沒有 changed_when
- name: Run script
  ansible.builtin.command: /usr/local/bin/myscript.sh

# ✅ 正確：明確定義何時算 changed
- name: Run script
  ansible.builtin.command: /usr/local/bin/myscript.sh
  register: script_result
  changed_when: "'changed' in script_result.stdout"
```

**錯誤 3：risky-file-permissions**

```yaml
# ❌ 錯誤：未指定檔案權限
- name: Create config file
  ansible.builtin.copy:
    content: "config content"
    dest: /etc/myapp.conf

# ✅ 正確：明確指定 mode
- name: Create config file
  ansible.builtin.copy:
    content: "config content"
    dest: /etc/myapp.conf
    owner: root
    group: root
    mode: '0644'
```

---

## 5.3 Molecule：自動化角色測試框架

### 5.3.1 Molecule 的運作原理

```
molecule test 執行流程：

  1. dependency    → 安裝 role 依賴
  2. lint          → 執行 ansible-lint
  3. cleanup       → 清理殘留環境（若存在）
  4. destroy       → 銷毀測試容器
  5. syntax        → 檢查 playbook 語法
  6. create        → 建立測試容器（Docker/Vagrant/EC2）
  7. prepare       → 準備環境（如安裝 python3）
  8. converge      → 執行 playbook（apply role）
  9. idempotence   → 再跑一次，確認 changed=0
  10. side_effect  → 測試 side effect（如重啟後）
  11. verify       → 執行驗證測試（Testinfra/Ansible）
  12. cleanup      → 清理
  13. destroy      → 銷毀容器
```

### 5.3.2 安裝 Molecule

```bash
# 安裝 Molecule 與 Docker driver
pipx inject ansible molecule molecule-plugins[docker]

# 確認安裝
molecule --version
```

### 5.3.3 為 kernel_tuning Role 建立 Molecule 測試

```bash
# 在 role 目錄內初始化 Molecule
cd roles/kernel_tuning
molecule init scenario --driver-name docker
```

生成的結構：

```
roles/kernel_tuning/
└── molecule/
    └── default/
        ├── molecule.yml     ← 測試環境設定
        ├── converge.yml     ← 執行被測 role 的 playbook
        ├── verify.yml       ← 驗證測試
        └── prepare.yml      ← 環境準備（選用）
```

### 5.3.4 molecule/default/molecule.yml

```yaml
# roles/kernel_tuning/molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    # 安裝 role 所需的 collections
    requirements-file: ../../../../requirements.yml

driver:
  name: docker

platforms:
  - name: ubuntu-2404-instance
    image: geerlingguy/docker-ubuntu2404-ansible:latest
    # 使用 privileged 模式，才能修改 sysctl（核心參數測試需要）
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: /lib/systemd/systemd  # 啟用 systemd

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml
  options:
    # 使用 verbose 輸出以便 CI 中 debug
    v: true
  env:
    # 在 CI 中啟用 Molecule 測試時使用 fake vault 密碼
    ANSIBLE_VAULT_PASSWORD: "molecule-test-password"

verifier:
  name: ansible   # 使用 Ansible 自身做驗證（也可用 testinfra）

scenario:
  test_sequence:
    - dependency
    - lint
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence   # 這是最關鍵的測試：確認 Idempotency
    - verify
    - destroy
```

### 5.3.5 molecule/default/converge.yml

```yaml
# roles/kernel_tuning/molecule/default/converge.yml
---
- name: Converge
  hosts: all
  gather_facts: true

  # 在 Molecule 測試中使用較寬鬆的測試參數（容器環境有限制）
  vars:
    # Docker 容器中不支援所有 sysctl（特別是網路相關），
    # 但 privileged 模式下大多數參數可以設定
    kernel_vm_swappiness: 10
    kernel_net_core_somaxconn: 65535
    kernel_tcp_tw_reuse: 1

  roles:
    - role: kernel_tuning
```

### 5.3.6 molecule/default/verify.yml（驗證測試）

```yaml
# roles/kernel_tuning/molecule/default/verify.yml
---
- name: Verify kernel_tuning role
  hosts: all
  gather_facts: true

  tasks:
    # ── 驗證 sysctl 設定已套用 ───────────────────────────────────────
    - name: Check net.core.somaxconn
      ansible.builtin.command:
        cmd: sysctl -n net.core.somaxconn
      register: somaxconn_val
      changed_when: false
      failed_when: somaxconn_val.stdout | int != 65535

    - name: Check net.ipv4.tcp_tw_reuse
      ansible.builtin.command:
        cmd: sysctl -n net.ipv4.tcp_tw_reuse
      register: tw_reuse_val
      changed_when: false
      failed_when: tw_reuse_val.stdout | int != 1

    - name: Check vm.swappiness
      ansible.builtin.command:
        cmd: sysctl -n vm.swappiness
      register: swappiness_val
      changed_when: false
      failed_when: swappiness_val.stdout | int != 10

    # ── 確認設定檔存在 ───────────────────────────────────────────────
    - name: Check sysctl config file exists
      ansible.builtin.stat:
        path: /etc/sysctl.d/99-kernel-tuning.conf
      register: sysctl_file
      failed_when: not sysctl_file.stat.exists

    - name: Verify sysctl file contains expected values
      ansible.builtin.command:
        cmd: grep "net.core.somaxconn" /etc/sysctl.d/99-kernel-tuning.conf
      register: grep_result
      changed_when: false
      failed_when: grep_result.rc != 0

    # ── 驗證結果摘要 ─────────────────────────────────────────────────
    - name: All verifications passed!
      ansible.builtin.debug:
        msg:
          - "✅ net.core.somaxconn = {{ somaxconn_val.stdout }}"
          - "✅ net.ipv4.tcp_tw_reuse = {{ tw_reuse_val.stdout }}"
          - "✅ vm.swappiness = {{ swappiness_val.stdout }}"
          - "✅ sysctl config file exists"
```

### 5.3.7 執行 Molecule 測試

```bash
# 完整測試流程（create → converge → idempotence → verify → destroy）
molecule test

# 只執行特定步驟（開發階段用）
molecule create     # 只建立容器
molecule converge   # 只執行 playbook（不 destroy）
molecule verify     # 只執行驗證
molecule destroy    # 銷毀容器

# 進入容器 debug
molecule login      # 進入測試容器的 shell

# 查看測試容器的 Ansible 輸出
molecule converge -- -v

# 測試 Idempotency（單獨執行）
molecule idempotence
```

---

## 5.4 requirements.yml：統一管理依賴

```yaml
# requirements.yml（專案根目錄）
---
collections:
  - name: ansible.posix
    version: ">=1.5.4"
  - name: community.docker
    version: ">=3.10.0"
  - name: community.general
    version: ">=8.0.0"
  - name: community.crypto
    version: ">=2.19.0"

roles:
  # 若使用第三方 role（如 geerlingguy.docker）
  # - name: geerlingguy.docker
  #   version: "7.0.0"
```

```bash
# 安裝所有依賴
ansible-galaxy install -r requirements.yml

# 安裝到指定路徑（CI 環境用）
ansible-galaxy collection install -r requirements.yml -p ./collections
```

---

## 5.5 完整 GitLab CI/CD Pipeline

### 5.5.1 .gitlab-ci.yml（完整版本）

```yaml
# .gitlab-ci.yml
---
stages:
  - validate      # 語法與 lint 檢查
  - test          # Molecule 自動化測試
  - dry-run       # Ansible dry-run（--check）
  - deploy        # 實際部署

# ── 全域變數 ─────────────────────────────────────────────────────────────
variables:
  ANSIBLE_FORCE_COLOR: "1"
  ANSIBLE_HOST_KEY_CHECKING: "False"
  PY_COLORS: "1"
  # CI 中使用較輕量的 callback
  ANSIBLE_STDOUT_CALLBACK: "yaml"

# ── 共用設定（YAML 錨點）────────────────────────────────────────────────
.ansible_base:
  image: python:3.12-slim
  before_script:
    # 安裝系統依賴
    - apt-get update -qq
    - apt-get install -y -qq git openssh-client rsync
    # 安裝 Ansible 與工具
    - pip install --quiet
        ansible
        ansible-lint
        molecule
        "molecule-plugins[docker]"
    # 安裝 Galaxy Collections
    - ansible-galaxy collection install -r requirements.yml -q
    # 設定 SSH Key（從 GitLab CI Variable 注入）
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - ssh-keyscan -H $PRODUCTION_HOSTS >> ~/.ssh/known_hosts 2>/dev/null || true
    # 設定 Vault 密碼
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/.vault_pass
    - chmod 600 /tmp/.vault_pass
  after_script:
    # 清理機密檔案
    - rm -f /tmp/.vault_pass ~/.ssh/id_ed25519

# ── Stage 1：Validate ────────────────────────────────────────────────────
ansible-lint:
  extends: .ansible_base
  stage: validate
  script:
    - ansible-lint --format pep8
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

syntax-check:
  extends: .ansible_base
  stage: validate
  script:
    - ansible-playbook playbooks/site.yml --syntax-check
    - ansible-playbook playbooks/kernel_tuning.yml --syntax-check
    - ansible-playbook playbooks/security_hardening.yml --syntax-check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ── Stage 2：Molecule 測試 ───────────────────────────────────────────────
molecule-kernel-tuning:
  extends: .ansible_base
  stage: test
  image: python:3.12-slim
  services:
    - name: docker:dind   # Docker-in-Docker，讓 Molecule 可以建立容器
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_TLS_VERIFY: "1"
    DOCKER_CERT_PATH: "$DOCKER_TLS_CERTDIR/client"
  before_script:
    - !reference [.ansible_base, before_script]
    - apt-get install -y -qq docker.io
  script:
    - cd roles/kernel_tuning
    - molecule test
  artifacts:
    when: always
    paths:
      - roles/kernel_tuning/.molecule/
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - roles/kernel_tuning/**/*  # 只在 kernel_tuning 有變更時才跑

molecule-security-hardening:
  extends: molecule-kernel-tuning
  script:
    - cd roles/security_hardening
    - molecule test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - roles/security_hardening/**/*

# ── Stage 3：Dry Run ─────────────────────────────────────────────────────
dry-run-staging:
  extends: .ansible_base
  stage: dry-run
  script:
    - >
      ansible-playbook playbooks/site.yml
      -i inventory/staging/
      --vault-password-file /tmp/.vault_pass
      --check
      --diff
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

dry-run-production:
  extends: .ansible_base
  stage: dry-run
  script:
    - >
      ansible-playbook playbooks/site.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      --check
      --diff
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  allow_failure: false  # dry-run 失敗則阻止部署

# ── Stage 4：部署 ────────────────────────────────────────────────────────
deploy-staging:
  extends: .ansible_base
  stage: deploy
  script:
    - >
      ansible-playbook playbooks/site.yml
      -i inventory/staging/
      --vault-password-file /tmp/.vault_pass
      -v
  environment:
    name: staging
    url: http://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: on_success  # staging 自動部署

deploy-production:
  extends: .ansible_base
  stage: deploy
  script:
    - >
      ansible-playbook playbooks/site.yml
      -i inventory/production/
      --vault-password-file /tmp/.vault_pass
      -v
  environment:
    name: production
    url: http://production.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual  # 生產環境需要人工觸發
  # 部署後 10 分鐘內可回滾
```

### 5.5.2 GitLab 設定 CI/CD Variables

在 GitLab Project → Settings → CI/CD → Variables 設定以下變數：

| Variable 名稱 | 類型 | 說明 | Protected | Masked |
|---------------|------|------|-----------|--------|
| `SSH_PRIVATE_KEY` | File | Ansible SSH 私鑰 | ✅ | ✅ |
| `ANSIBLE_VAULT_PASSWORD` | Variable | Vault 密碼 | ✅ | ✅ |
| `PRODUCTION_HOSTS` | Variable | 正式環境主機 IP（空格分隔）| ✅ | ❌ |

---

## 5.6 Ansible Callback Plugin：強化執行報告

```yaml
# ansible.cfg 中加入
[defaults]
# 顯示每個任務的執行時間
callbacks_enabled = timer, profile_tasks, profile_roles

# CI 中使用 JSON 輸出（方便解析）
# stdout_callback = json
```

```bash
# 產生 HTML 格式的執行報告
pip install ara

# 在 ansible.cfg 加入
[defaults]
callbacks_enabled = ara.plugins.callback.ara_default

# 查看報告
ara playbook list
```

---

## 5.7 完整端到端測試腳本

在 CI/CD 設定完成後，以下是完整的驗收測試：

```bash
#!/bin/bash
# scripts/e2e_test.sh - 端到端驗收測試腳本

set -euo pipefail

echo "=== 1. 語法與 Lint 檢查 ==="
ansible-lint playbooks/ roles/
ansible-playbook playbooks/site.yml --syntax-check

echo "=== 2. Molecule 測試（所有 Role）==="
for role in roles/*/; do
  if [ -d "${role}molecule" ]; then
    echo "Testing ${role}..."
    (cd "$role" && molecule test)
  fi
done

echo "=== 3. Staging 環境 Dry Run ==="
ansible-playbook playbooks/site.yml \
  -i inventory/staging/ \
  --vault-password-file /tmp/.vault_pass \
  --check --diff

echo "=== 4. Staging 環境部署 ==="
ansible-playbook playbooks/site.yml \
  -i inventory/staging/ \
  --vault-password-file /tmp/.vault_pass

echo "=== 5. 驗證 Staging 環境 ==="
ansible all -i inventory/staging/ -m ansible.builtin.ping
ansible all -i inventory/staging/ \
  -m ansible.builtin.command \
  -a "sysctl net.core.somaxconn"

echo "=== 6. Idempotency 驗證 ==="
ansible-playbook playbooks/site.yml \
  -i inventory/staging/ \
  --vault-password-file /tmp/.vault_pass 2>&1 | \
  grep -E "changed=([1-9])" && \
  echo "❌ Idempotency 失敗！有未預期的 changed" && exit 1 || \
  echo "✅ Idempotency 驗證通過（changed=0）"

echo ""
echo "✅ 所有端到端測試通過！可以安全部署到生產環境。"
```

---

## 5.8 Makefile：統一開發入口

```makefile
# Makefile - 提供統一的命令入口
.PHONY: help lint test dry-run deploy setup

VAULT_FILE ?= /tmp/.vault_pass
INVENTORY ?= inventory/production

help:  ## 顯示說明
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

setup:  ## 安裝所有依賴
	pipx install ansible ansible-lint
	pipx inject ansible molecule "molecule-plugins[docker]"
	ansible-galaxy install -r requirements.yml

lint:  ## 執行 ansible-lint
	ansible-lint

test:  ## 執行所有 Molecule 測試
	@for role in roles/*/; do \
		if [ -d "$${role}molecule" ]; then \
			echo "=== Testing $$role ==="; \
			(cd "$$role" && molecule test); \
		fi \
	done

test-role:  ## 測試特定 Role (ROLE=<name>)
	cd roles/$(ROLE) && molecule test

dry-run:  ## Dry run 到指定 Inventory
	ansible-playbook playbooks/site.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE) \
		--check --diff

deploy:  ## 部署到指定 Inventory
	ansible-playbook playbooks/site.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE)

deploy-kernel:  ## 只部署核心調優
	ansible-playbook playbooks/kernel_tuning.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE) \
		--tags kernel

deploy-hardening:  ## 只部署資安強化
	ansible-playbook playbooks/security_hardening.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE) \
		--tags hardening

idempotency-check:  ## 執行 Idempotency 檢查
	ansible-playbook playbooks/site.yml \
		-i $(INVENTORY) \
		--vault-password-file $(VAULT_FILE) 2>&1 | \
		grep -E "changed=([0-9]+)" | \
		grep -v "changed=0" && exit 1 || echo "✅ Idempotency OK"

vault-edit:  ## 編輯 Vault 加密檔
	ansible-vault edit vault/secrets.yml

docs:  ## 產生 role 文件（需安裝 antsibull-docs）
	antsibull-docs collection --use-current . docs/
```

使用範例：

```bash
# 查看所有可用命令
make help

# 完整測試
make lint && make test

# Dry run 到 staging
make dry-run INVENTORY=inventory/staging

# 部署到生產環境
make deploy INVENTORY=inventory/production VAULT_FILE=~/.vault_pass
```

---

## 5.9 驗證步驟：確認 CI/CD 流程正常

```bash
# 1. 提交一個小改動（如修改 README）
git add -A
git commit -m "ci: test pipeline trigger"
git push origin feature/test-pipeline

# 2. 建立 Merge Request，觀察 CI 執行
# 在 GitLab UI：Merge Requests → New MR

# 3. 確認以下 jobs 都通過：
# ✅ ansible-lint
# ✅ syntax-check
# ✅ molecule-kernel-tuning
# ✅ dry-run-staging

# 4. Merge 到 main 後，確認以下 jobs 執行：
# ✅ dry-run-production
# ✅ deploy-staging（自動）
# 🔵 deploy-production（等待手動觸發）

# 5. 觀察 staging 部署結果
ansible all -i inventory/staging/ -m ansible.builtin.ping

# 6. 手動觸發 production 部署（在 GitLab UI 中點擊 ▶ 按鈕）

# 7. 驗收：生產環境所有主機狀態正常
ansible all -i inventory/production/ -m ansible.builtin.setup \
  -a "filter=ansible_distribution*"
```

---

## 5.10 章節小結與完整課程回顧

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| ansible-lint 設定與執行 | ☐ 專案所有 role 通過 lint 檢查 |
| Molecule 測試框架 | ☐ 每個 role 有獨立 molecule 測試並全部通過 |
| Idempotency 自動驗證 | ☐ CI pipeline 中包含 idempotency 測試步驟 |
| GitLab CI/CD 完整 pipeline | ☐ MR 觸發 lint + test，merge 後自動部署 staging |
| 生產環境 Manual Gate | ☐ production 部署需要人工確認，防止意外 |
| Makefile 統一入口 | ☐ 團隊可用 `make lint/test/deploy` 操作 |

---

## 完整課程總結

恭喜你完成了所有 5 個章節！以下是你建立的完整系統：

```
┌────────────────────────────────────────────────────────────────┐
│              你已建立的 Ansible 自動化運維體系                  │
│                                                                │
│  第1章  環境與 Inventory ── 多環境分離、Vault 加密              │
│    ↓                                                           │
│  第2章  Docker + GitLab ─── 自動化部署、GitOps 閉環            │
│    ↓                                                           │
│  第3章  Kernel Tuning ───── 高併發調優、node_exporter 監控     │
│    ↓                                                           │
│  第4章  Security Hardening ─ SSH/UFW/Fail2Ban/Vault 整合       │
│    ↓                                                           │
│  第5章  工程化 CI/CD ────── Lint/Molecule/GitLab Pipeline       │
│                                                                │
│  最終產出：推送程式碼 → 自動測試 → 自動部署 → 監控驗證         │
└────────────────────────────────────────────────────────────────┘
```

**下一步學習建議：**

- **進階 Ansible：** 動態 Inventory（AWS/GCP Plugin）、Custom Module 開發
- **Terraform + Ansible：** 用 Terraform 建立基礎設施，Ansible 設定作業系統
- **監控完善：** 部署 Prometheus + Grafana，建立 Kernel Tuning 效果的儀表板
- **GitOps 進階：** 探索 ArgoCD（用於 Kubernetes）或 Flux 的類似 GitOps 概念

---

*← [返回總覽](./README.md) | [上一章](./chapter-04-security-hardening.md)*
