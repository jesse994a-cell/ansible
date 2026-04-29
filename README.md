# Ansible 自動化運維實戰教材
## 面向 Ubuntu 24.04 的 DevOps/SRE 工程師完整課程

> **適用對象：** 具備 Linux 基礎，希望透過 Ansible 深入自動化運維的技術人員。  
> **作業系統：** Ubuntu 24.04 LTS (Noble Numbat)  
> **Ansible 版本：** 2.16+（搭配 ansible-core 2.16）

---

## 學習路徑總覽

```
┌─────────────────────────────────────────────────────────────┐
│                   Ansible 自動化運維課程                      │
│                                                             │
│  第1章          第2章          第3章                          │
│  環境初始化  →  Docker/GitLab →  核心調優                     │
│  & Inventory    CI/CD + GitOps  Kernel Tuning               │
│                                                             │
│                    ↓                                        │
│              第4章          第5章                            │
│              資安強化    →  工程化 CI/CD                      │
│              Hardening      Lint + Molecule                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 課程大綱與章節索引

| 階段 | 章節 | 核心主題 | 產出目標 |
|------|------|----------|----------|
| 第一階段 | [第 1 章](./chapter-01-environment-inventory.md) | Ansible 基礎架構 | 多環境 Inventory 與 Vault 基礎 |
| 第二階段 | [第 2 章](./chapter-02-docker-gitlab.md) | Docker & GitLab CI/CD | 自動化部署 GitLab + GitOps 閉環 |
| 第三階段 | [第 3 章](./chapter-03-kernel-tuning.md) | 高併發核心參數調優 | 系統效能強化 Role |
| 第四階段 | [第 4 章](./chapter-04-security-hardening.md) | 系統資安強化 | 資安防護中心 Role |
| 第五階段 | [第 5 章](./chapter-05-engineering-ci.md) | 工程化與 CI Integration | 端到端自動化流程 |

---

## 前置知識要求

- **Linux 基礎：** 熟悉 shell 操作、檔案權限、systemd 服務管理
- **網路概念：** 瞭解 TCP/IP、防火牆基本原理
- **版本控制：** 熟悉 Git 基本操作
- **容器基礎（選修）：** 有 Docker 使用經驗者學習第 2 章更順暢

---

## 學習環境建議

### 最低硬體需求

| 角色 | CPU | RAM | Disk |
|------|-----|-----|------|
| Ansible Control Node | 2 vCPU | 4 GB | 20 GB |
| Target Node (×2) | 2 vCPU | 4 GB | 40 GB |
| GitLab Server | 4 vCPU | 8 GB | 50 GB |

### 建議實驗架構

```
┌─────────────────────────────────────────────────┐
│                  實驗環境拓樸                     │
│                                                  │
│  [Control Node]          [Target Nodes]          │
│  Ubuntu 24.04            Ubuntu 24.04            │
│  Ansible 2.16+     SSH   ┌──────────────┐        │
│  192.168.56.10   ──────► │ web-01       │        │
│                          │ 192.168.56.11│        │
│                          └──────────────┘        │
│                          ┌──────────────┐        │
│                   SSH    │ db-01        │        │
│                  ──────► │ 192.168.56.12│        │
│                          └──────────────┘        │
└─────────────────────────────────────────────────┘
```

---

## 目錄結構規劃（完整課程）

本教材最終會建立如下的 Ansible 專案結構：

```
ansible-course/
├── ansible.cfg                    # Ansible 全域設定
├── inventory/
│   ├── production/
│   │   ├── hosts.yml              # 正式環境主機清單
│   │   ├── group_vars/
│   │   │   ├── all.yml            # 共用變數
│   │   │   ├── webservers.yml     # Web 群組變數
│   │   │   └── dbservers.yml      # DB 群組變數
│   │   └── host_vars/
│   │       ├── web-01.yml
│   │       └── db-01.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── roles/
│   ├── common/                    # 基礎初始化 Role
│   ├── docker/                    # Docker 安裝 Role
│   ├── gitlab/                    # GitLab 部署 Role
│   ├── kernel_tuning/             # 第3章：核心調優 Role
│   ├── security_hardening/        # 第4章：資安強化 Role
│   └── monitoring/                # 監控 Role（node_exporter）
├── playbooks/
│   ├── site.yml                   # 主入口 Playbook
│   ├── deploy_gitlab.yml
│   ├── kernel_tuning.yml
│   └── security_hardening.yml
├── vault/
│   └── secrets.yml                # Ansible Vault 加密機密
├── .gitlab-ci.yml                 # GitLab CI/CD Pipeline
├── .ansible-lint                  # Lint 設定
└── molecule/                      # Molecule 測試框架
    └── default/
        ├── molecule.yml
        ├── converge.yml
        └── verify.yml
```

---

## 工具安裝速查

```bash
# 安裝 Ansible（Ubuntu 24.04）
sudo apt update && sudo apt install -y pipx
pipx install ansible
pipx inject ansible ansible-lint

# 安裝 Ansible Galaxy Collections
ansible-galaxy collection install \
  community.docker \
  ansible.posix \
  community.general

# 驗證安裝
ansible --version
ansible-galaxy collection list
```

---

## 章節學習建議

1. **循序漸進：** 建議依章節順序學習，每章都有前章依賴。
2. **動手實作：** 每個 Role 都應在實際環境中跑過，不要只讀程式碼。
3. **Idempotency 驗證：** 每次 Playbook 跑完後，再跑一次確認 `changed=0`。
4. **Git 版控：** 所有 Ansible 程式碼應納入版控，養成 IaC 習慣。

---

*教材版本：v1.0 | 更新日期：2026-04 | 適用 Ansible Core 2.16+*
