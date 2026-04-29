# 第 2 章：Docker 與 GitLab 容器化部署

> **學習目標：** 使用 Ansible `community.docker` 模組自動部署 GitLab CE（含 PostgreSQL、Redis），並建立 GitOps 閉環——當程式碼推送至 GitLab 時，自動觸發 Ansible 對目標主機進行配置。

---

## 2.1 理論說明：為什麼用 Ansible 管理 Docker？

### 2.1.1 容器編排的兩個層次

```
┌──────────────────────────────────────────────┐
│  Layer 2：容器層（Container Orchestration）   │
│  Docker Compose / Kubernetes                 │
│  管理：容器生命週期、網路、Volume             │
├──────────────────────────────────────────────┤
│  Layer 1：主機層（Infrastructure Automation） │
│  Ansible                                     │
│  管理：Docker 安裝、系統設定、防火牆、監控    │
└──────────────────────────────────────────────┘
```

Ansible 負責**主機層**的自動化：確保 Docker Engine 安裝正確、服務啟動、iptables 規則設定。容器的啟動與設定，也可以透過 Ansible 的 `community.docker` 模組統一管理，達到「一份 Playbook 從零到完整服務」的效果。

### 2.1.2 GitOps 概念

**GitOps** 的核心原則：**Git Repository 是唯一的事實來源（Single Source of Truth）**。

```
開發者 push code
       │
       ▼
┌─────────────┐    觸發    ┌───────────────┐    執行     ┌─────────────┐
│  GitLab     │ ────────► │  GitLab CI/CD │ ──────────► │  Ansible    │
│  Repository │           │  Pipeline     │             │  Playbook   │
└─────────────┘           └───────────────┘             └──────┬──────┘
                                                               │
                                                    SSH/API    ▼
                                                     ┌─────────────────┐
                                                     │  Target Servers │
                                                     │  (自動配置完成)  │
                                                     └─────────────────┘
```

本章的終極目標：建立這個閉環，讓「推送程式碼」等於「基礎設施自動更新」。

---

## 2.2 Ansible Role 結構：docker

```bash
# 建立 docker role
ansible-galaxy role init roles/docker
```

```
roles/docker/
├── defaults/
│   └── main.yml         ← Docker 版本、daemon 設定預設值
├── handlers/
│   └── main.yml         ← 重啟 Docker daemon
├── tasks/
│   ├── main.yml         ← 任務入口，include 子任務
│   ├── install.yml      ← 安裝 Docker Engine
│   └── configure.yml    ← 配置 daemon.json
└── templates/
    └── daemon.json.j2   ← Docker daemon 設定模板
```

### 2.2.1 defaults/main.yml

```yaml
# roles/docker/defaults/main.yml
---
# ── Docker 版本 ───────────────────────────────
# 指定版本確保跨主機一致性，避免 "latest" 帶來的不確定性
docker_version: "5:26.1.*"
docker_compose_version: "2.27.0"

# ── Docker daemon 設定 ────────────────────────
docker_daemon_config:
  # 使用 overlay2 儲存驅動（Ubuntu 24.04 最佳選擇）
  storage-driver: overlay2
  # 限制容器日誌大小，避免磁碟爆滿
  log-driver: json-file
  log-opts:
    max-size: "100m"
    max-file: "3"
  # 自訂 Docker 網段，避免與內部網路衝突
  bip: "172.17.0.1/16"
  # 啟用實驗功能（選用）
  experimental: false
  # 指標暴露（給 Prometheus 用）
  metrics-addr: "0.0.0.0:9323"
  experimental: true

# ── 允許管理 Docker 的使用者 ───────────────────
docker_users:
  - ubuntu
```

### 2.2.2 tasks/main.yml（入口）

```yaml
# roles/docker/tasks/main.yml
---
- name: Install Docker Engine
  ansible.builtin.import_tasks: install.yml
  tags: [docker, install]

- name: Configure Docker daemon
  ansible.builtin.import_tasks: configure.yml
  tags: [docker, configure]
```

### 2.2.3 tasks/install.yml

```yaml
# roles/docker/tasks/install.yml
---
# ── 移除舊版本 Docker（避免衝突）────────────────────────────────────────
- name: Remove conflicting Docker packages
  ansible.builtin.apt:
    name:
      - docker.io
      - docker-doc
      - docker-compose
      - docker-compose-v2
      - podman-docker
      - containerd
      - runc
    state: absent
  tags: [docker]

# ── 安裝 GPG 金鑰與 APT Repository ──────────────────────────────────────
- name: Create /etc/apt/keyrings directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Download Docker official GPG key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
    force: false  # 如果已存在則跳過

- name: Add Docker APT repository
  ansible.builtin.apt_repository:
    repo: >-
      deb [arch={{ ansible_architecture | replace('x86_64', 'amd64') }}
      signed-by=/etc/apt/keyrings/docker.asc]
      https://download.docker.com/linux/ubuntu
      {{ ansible_distribution_release }} stable
    state: present
    filename: docker

# ── 安裝 Docker Engine ───────────────────────────────────────────────────
- name: Install Docker Engine packages
  ansible.builtin.apt:
    name:
      - "docker-ce={{ docker_version }}"
      - "docker-ce-cli={{ docker_version }}"
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present
    update_cache: true
  notify: Restart Docker

# ── 啟動並設定開機自啟 ───────────────────────────────────────────────────
- name: Enable and start Docker service
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true

# ── 將使用者加入 docker 群組 ─────────────────────────────────────────────
- name: Add users to docker group
  ansible.builtin.user:
    name: "{{ item }}"
    groups: docker
    append: true  # append=true 不會移除使用者現有群組
  loop: "{{ docker_users }}"
```

### 2.2.4 tasks/configure.yml

```yaml
# roles/docker/tasks/configure.yml
---
- name: Configure Docker daemon (daemon.json)
  ansible.builtin.copy:
    content: "{{ docker_daemon_config | to_nice_json }}"
    dest: /etc/docker/daemon.json
    owner: root
    group: root
    mode: '0644'
  notify: Restart Docker
```

### 2.2.5 handlers/main.yml

```yaml
# roles/docker/handlers/main.yml
---
- name: Restart Docker
  ansible.builtin.service:
    name: docker
    state: restarted
```

---

## 2.3 Ansible Role 結構：gitlab

```bash
ansible-galaxy role init roles/gitlab
```

```
roles/gitlab/
├── defaults/
│   └── main.yml         ← GitLab 版本、埠號、Volume 路徑等
├── tasks/
│   ├── main.yml
│   ├── network.yml      ← 建立 Docker 網路
│   ├── volumes.yml      ← 建立 Volume 目錄
│   ├── postgres.yml     ← 部署 PostgreSQL
│   ├── redis.yml        ← 部署 Redis
│   └── gitlab.yml       ← 部署 GitLab CE
└── handlers/
    └── main.yml
```

### 2.3.1 defaults/main.yml

```yaml
# roles/gitlab/defaults/main.yml
---
# ── GitLab 版本 ───────────────────────────────
gitlab_image: "gitlab/gitlab-ce:17.0.1-ce.0"
gitlab_external_url: "http://{{ ansible_host }}"

# ── PostgreSQL ────────────────────────────────
postgres_image: "postgres:16-alpine"
postgres_db: gitlabhq_production
postgres_user: gitlab
# 實際密碼來自 Vault（vault_db_app_password）
postgres_password: "{{ vault_db_app_password }}"

# ── Redis ─────────────────────────────────────
redis_image: "redis:7-alpine"

# ── Docker 網路名稱 ───────────────────────────
gitlab_network: gitlab_net

# ── Volume 掛載路徑（主機端） ─────────────────
gitlab_data_path: /opt/gitlab
gitlab_config_path: "{{ gitlab_data_path }}/config"
gitlab_logs_path: "{{ gitlab_data_path }}/logs"
gitlab_data_dir: "{{ gitlab_data_path }}/data"
postgres_data_path: "{{ gitlab_data_path }}/postgresql"
redis_data_path: "{{ gitlab_data_path }}/redis"

# ── GitLab 埠號 ───────────────────────────────
gitlab_http_port: 80
gitlab_https_port: 443
gitlab_ssh_port: 2222

# ── GitLab 初始設定（機密來自 Vault）─────────
gitlab_root_password: "{{ vault_gitlab_root_password }}"
```

### 2.3.2 tasks/main.yml

```yaml
# roles/gitlab/tasks/main.yml
---
- name: Create Docker network
  ansible.builtin.import_tasks: network.yml
  tags: [gitlab, network]

- name: Create volume directories
  ansible.builtin.import_tasks: volumes.yml
  tags: [gitlab, volumes]

- name: Deploy PostgreSQL
  ansible.builtin.import_tasks: postgres.yml
  tags: [gitlab, postgres]

- name: Deploy Redis
  ansible.builtin.import_tasks: redis.yml
  tags: [gitlab, redis]

- name: Deploy GitLab CE
  ansible.builtin.import_tasks: gitlab.yml
  tags: [gitlab, app]
```

### 2.3.3 tasks/network.yml

```yaml
# roles/gitlab/tasks/network.yml
---
- name: Create dedicated Docker network for GitLab
  community.docker.docker_network:
    name: "{{ gitlab_network }}"
    # bridge 驅動，容器間可透過 service name 互通
    driver: bridge
    state: present
```

### 2.3.4 tasks/volumes.yml

```yaml
# roles/gitlab/tasks/volumes.yml
---
# 一次建立所有需要的目錄，loop 寫法更簡潔
- name: Create GitLab data directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - "{{ gitlab_config_path }}"
    - "{{ gitlab_logs_path }}"
    - "{{ gitlab_data_dir }}"
    - "{{ postgres_data_path }}"
    - "{{ redis_data_path }}"
```

### 2.3.5 tasks/postgres.yml

```yaml
# roles/gitlab/tasks/postgres.yml
---
- name: Deploy PostgreSQL container
  community.docker.docker_container:
    name: gitlab-postgres
    image: "{{ postgres_image }}"
    state: started
    restart_policy: unless-stopped  # 自動重啟，除非手動停止

    # 連接至 GitLab 專用網路
    networks:
      - name: "{{ gitlab_network }}"

    # Volume 掛載：主機目錄 → 容器內路徑
    volumes:
      - "{{ postgres_data_path }}:/var/lib/postgresql/data"

    # 環境變數（密碼來自 Vault，不以明文出現在 task 中）
    env:
      POSTGRES_DB: "{{ postgres_db }}"
      POSTGRES_USER: "{{ postgres_user }}"
      POSTGRES_PASSWORD: "{{ postgres_password }}"
      # PostgreSQL 效能調整
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"

    # 健康檢查：確保 PostgreSQL 真正可接受連線後才算 healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ postgres_user }} -d {{ postgres_db }}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

- name: Wait for PostgreSQL to be healthy
  community.docker.docker_container_info:
    name: gitlab-postgres
  register: postgres_info
  until: postgres_info.container.State.Health.Status == "healthy"
  retries: 12   # 最多等待 2 分鐘
  delay: 10
```

### 2.3.6 tasks/redis.yml

```yaml
# roles/gitlab/tasks/redis.yml
---
- name: Deploy Redis container
  community.docker.docker_container:
    name: gitlab-redis
    image: "{{ redis_image }}"
    state: started
    restart_policy: unless-stopped

    networks:
      - name: "{{ gitlab_network }}"

    volumes:
      - "{{ redis_data_path }}:/data"

    # Redis 開啟持久化（AOF）
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 2.3.7 tasks/gitlab.yml

```yaml
# roles/gitlab/tasks/gitlab.yml
---
- name: Deploy GitLab CE container
  community.docker.docker_container:
    name: gitlab-ce
    image: "{{ gitlab_image }}"
    state: started
    restart_policy: unless-stopped

    # 埠號映射：主機埠 → 容器埠
    published_ports:
      - "{{ gitlab_http_port }}:80"
      - "{{ gitlab_https_port }}:443"
      - "{{ gitlab_ssh_port }}:22"

    networks:
      - name: "{{ gitlab_network }}"

    volumes:
      - "{{ gitlab_config_path }}:/etc/gitlab"
      - "{{ gitlab_logs_path }}:/var/log/gitlab"
      - "{{ gitlab_data_dir }}:/var/opt/gitlab"

    env:
      # GitLab 對外 URL（影響 clone URL、郵件中的連結）
      GITLAB_OMNIBUS_CONFIG: |
        external_url '{{ gitlab_external_url }}'

        # 使用外部 PostgreSQL（停用內建）
        postgresql['enable'] = false
        gitlab_rails['db_adapter'] = 'postgresql'
        gitlab_rails['db_encoding'] = 'utf8'
        gitlab_rails['db_database'] = '{{ postgres_db }}'
        gitlab_rails['db_username'] = '{{ postgres_user }}'
        gitlab_rails['db_password'] = '{{ postgres_password }}'
        gitlab_rails['db_host'] = 'gitlab-postgres'
        gitlab_rails['db_port'] = 5432

        # 使用外部 Redis（停用內建）
        redis['enable'] = false
        gitlab_rails['redis_host'] = 'gitlab-redis'
        gitlab_rails['redis_port'] = 6379

        # 初始 root 密碼（首次啟動時設定）
        gitlab_rails['initial_root_password'] = '{{ gitlab_root_password }}'

        # 關閉 signup（安全考量）
        gitlab_rails['gitlab_signup_enabled'] = false

        # SSH 埠號提示
        gitlab_rails['gitlab_shell_ssh_port'] = {{ gitlab_ssh_port }}

    # GitLab 啟動較慢，設定較寬鬆的健康檢查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/-/health"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 300s  # GitLab 首次啟動需要約 5 分鐘

# 等待 GitLab 完全啟動
- name: Wait for GitLab to be healthy (up to 10 minutes)
  community.docker.docker_container_info:
    name: gitlab-ce
  register: gitlab_info
  until: gitlab_info.container.State.Health.Status == "healthy"
  retries: 20
  delay: 30

- name: Show GitLab access information
  ansible.builtin.debug:
    msg:
      - "GitLab CE 部署完成！"
      - "URL: {{ gitlab_external_url }}"
      - "帳號: root"
      - "密碼: （已透過 Vault 設定）"
      - "SSH Port: {{ gitlab_ssh_port }}"
```

---

## 2.4 GitLab Runner 自動化註冊

### 2.4.1 Runner 角色設計

```bash
ansible-galaxy role init roles/gitlab_runner
```

```yaml
# roles/gitlab_runner/defaults/main.yml
---
gitlab_runner_image: "gitlab/gitlab-runner:v17.0.0"

# Runner 設定（Token 來自 Vault）
gitlab_runner_registration_token: "{{ vault_gitlab_runner_token }}"
gitlab_runner_url: "{{ gitlab_external_url }}"
gitlab_runner_name: "{{ inventory_hostname }}-docker-runner"
gitlab_runner_tags:
  - docker
  - ubuntu-24.04
  - production

# Runner executor
gitlab_runner_executor: docker
gitlab_runner_docker_image: "ubuntu:24.04"

# Runner 並發數
gitlab_runner_concurrent: 4
```

```yaml
# roles/gitlab_runner/tasks/main.yml
---
# ── 部署 Runner 容器 ─────────────────────────────────────────────────────
- name: Create GitLab Runner config directory
  ansible.builtin.file:
    path: /etc/gitlab-runner
    state: directory
    mode: '0700'

- name: Deploy GitLab Runner container
  community.docker.docker_container:
    name: gitlab-runner
    image: "{{ gitlab_runner_image }}"
    state: started
    restart_policy: unless-stopped
    # 需要掛載 Docker socket 以執行 Docker-in-Docker
    volumes:
      - /etc/gitlab-runner:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock

# ── 自動註冊 Runner ──────────────────────────────────────────────────────
- name: Check if runner is already registered
  ansible.builtin.stat:
    path: /etc/gitlab-runner/config.toml
  register: runner_config

- name: Register GitLab Runner
  # 只在尚未註冊時才執行（Idempotency）
  when: not runner_config.stat.exists or runner_config.stat.size == 0
  community.docker.docker_container_exec:
    container: gitlab-runner
    command: >
      gitlab-runner register
      --non-interactive
      --url "{{ gitlab_runner_url }}"
      --registration-token "{{ gitlab_runner_registration_token }}"
      --description "{{ gitlab_runner_name }}"
      --executor "{{ gitlab_runner_executor }}"
      --docker-image "{{ gitlab_runner_docker_image }}"
      --tag-list "{{ gitlab_runner_tags | join(',') }}"
      --run-untagged=false
      --locked=false
```

---

## 2.5 GitOps 閉環：.gitlab-ci.yml

這是 GitOps 的核心——當 Ansible Playbook 有任何變更推送到 GitLab，CI/CD Pipeline 自動對目標主機重新套用配置：

```yaml
# .gitlab-ci.yml
---
stages:
  - lint         # 程式碼品質檢查
  - dry-run      # 模擬執行（不實際變更）
  - deploy       # 實際部署

variables:
  # Ansible Vault 密碼從 GitLab CI/CD Variables 注入（加密儲存）
  ANSIBLE_VAULT_PASSWORD: $VAULT_PASSWORD
  # 關閉 SSH host key checking（CI 環境）
  ANSIBLE_HOST_KEY_CHECKING: "False"

# 所有 job 的共用 before_script
.ansible_setup: &ansible_setup
  before_script:
    - apt-get update -qq && apt-get install -y -qq python3-pip
    - pip3 install ansible ansible-lint --quiet
    - ansible-galaxy collection install ansible.posix community.docker community.general -q
    # 從 CI Variable 建立 Vault 密碼檔
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/.vault_pass
    - chmod 600 /tmp/.vault_pass
    # 建立 SSH 私鑰（從 CI Variable 注入）
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/ansible_ed25519
    - chmod 600 ~/.ssh/ansible_ed25519

# ── Stage 1：Lint 檢查 ───────────────────────────────────────────────────
ansible-lint:
  stage: lint
  image: python:3.12-slim
  <<: *ansible_setup
  script:
    - ansible-lint playbooks/
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# ── Stage 2：Dry Run ─────────────────────────────────────────────────────
dry-run:
  stage: dry-run
  image: python:3.12-slim
  <<: *ansible_setup
  script:
    - >
      ansible-playbook playbooks/site.yml
      --vault-password-file /tmp/.vault_pass
      --check
      --diff
      -v
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# ── Stage 3：正式部署 ────────────────────────────────────────────────────
deploy-production:
  stage: deploy
  image: python:3.12-slim
  <<: *ansible_setup
  script:
    - >
      ansible-playbook playbooks/site.yml
      --vault-password-file /tmp/.vault_pass
      -v
  # 只在 main branch 且 tag 格式正確時才自動部署
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # 需要人工確認才執行（避免意外部署）
  environment:
    name: production
```

> **重要：** 在 GitLab Project Settings → CI/CD → Variables 中設定：
> - `VAULT_PASSWORD`：Ansible Vault 密碼（標記為 Protected + Masked）
> - `SSH_PRIVATE_KEY`：Ansible 控制節點的 SSH 私鑰（標記為 Protected + Masked）

---

## 2.6 完整部署 Playbook

```yaml
# playbooks/deploy_gitlab.yml
---
- name: Install Docker on GitLab server
  hosts: gitlab_servers
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: common
      tags: [common]
    - role: docker
      tags: [docker]

- name: Deploy GitLab CE stack
  hosts: gitlab_servers

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: gitlab
      tags: [gitlab]

- name: Deploy GitLab Runner on web servers
  hosts: webservers
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: gitlab_runner
      tags: [runner]
```

---

## 2.7 驗證步驟

```bash
# 1. 執行部署（首次需要較長時間）
ansible-playbook playbooks/deploy_gitlab.yml \
  --vault-password-file ~/.vault_pass \
  -v

# 2. 確認容器狀態
ansible gitlab_servers -m ansible.builtin.command \
  -a "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

# 3. 查看 GitLab 啟動日誌
ansible gitlab_servers -m ansible.builtin.command \
  -a "docker logs --tail=50 gitlab-ce"

# 4. 檢查 GitLab 健康狀態
ansible gitlab_servers -m ansible.builtin.uri \
  -a "url=http://localhost/-/health return_content=yes"
# 預期：{"status": "ok"}

# 5. 確認 PostgreSQL 連線
ansible gitlab_servers -m ansible.builtin.command \
  -a "docker exec gitlab-postgres pg_isready -U gitlab -d gitlabhq_production"
# 預期：localhost:5432 - accepting connections

# 6. 確認 Redis 連線
ansible gitlab_servers -m ansible.builtin.command \
  -a "docker exec gitlab-redis redis-cli ping"
# 預期：PONG

# 7. 確認 Runner 已註冊（在 GitLab UI 中查看）
# Admin Area → CI/CD → Runners
```

---

## 2.8 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| community.docker 模組使用 | ☐ 可用 `docker_container` 部署容器 |
| 多容器依賴管理（健康檢查等待）| ☐ PostgreSQL healthy 後再啟動 GitLab |
| Ansible Vault 整合 | ☐ 密碼透過 Vault 注入，不出現在 task 明文中 |
| GitLab Runner 自動註冊 | ☐ Runner 上線並出現在 GitLab Admin 頁面 |
| GitOps 閉環設計 | ☐ 理解 `.gitlab-ci.yml` 如何觸發 Ansible |
| Idempotency 驗證 | ☐ 重跑 playbook 所有容器狀態不變，changed=0 |

**下一章：** [第 3 章 高併發核心參數調優](./chapter-03-kernel-tuning.md) — 深度解說 TCP Stack 優化與 Linux Kernel 參數調整。

---

*← [返回總覽](./README.md) | [上一章](./chapter-01-environment-inventory.md)*
