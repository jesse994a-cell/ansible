# Ansible 自動化運維實戰教材
## 面向 Ubuntu 24.04 的 DevOps/SRE 工程師完整課程

> **適用對象：** 具備 Linux 基礎，希望透過 Ansible 深入自動化運維的技術人員。  
> **作業系統：** Ubuntu 24.04 LTS (Noble Numbat)  
> **Ansible 版本：** 2.16+（搭配 ansible-core 2.16）

---

## 學習路徑總覽

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Ansible 自動化運維課程                            │
│                                                                      │
│  第1章          第2章          第3章                                   │
│  環境初始化  →  Docker/GitLab →  核心調優                              │
│  & Inventory    CI/CD + GitOps  Kernel Tuning                        │
│                                                                      │
│                    ↓                                                 │
│              第4章          第5章                                     │
│              資安強化    →  工程化 CI/CD                               │
│              Hardening      Lint + Molecule                          │
│                                                                      │
│  ══════════════ ISO 27001 合規自動化（第 6-9 章）══════════════        │
│                                                                      │
│  第6章           第7章           第8章           第9章                 │
│  IT 資產管理  →  CIS Benchmark →  稽核證據收集 →  定期稽核流水線        │
│  Snipe-IT        合規掃描         Evidence         Scheduled CI/CD   │
│  (A.5.9)         (A.8.8/8.9)      Collector        52週閉環           │
│                                                                      │
│  ══════════════ 進階合規自動化（第 10-12 章）══════════════            │
│                                                                      │
│  第10章              第11章              第12章                        │
│  SSL 憑證監控    →   帳號生命週期    →   備份 DR 演練                   │
│  到期告警 + 續期     入離職自動化         RTO/RPO 實測                  │
│  (A.8.24)            (A.5.16/5.18)       (A.8.13)                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 課程大綱與章節索引

### 第一部分：自動化運維基礎（第 1-5 章）

| 階段 | 章節 | 核心主題 | 產出目標 |
|------|------|----------|----------|
| 第一階段 | [第 1 章](./chapter-01-environment-inventory.md) | Ansible 基礎架構 | 多環境 Inventory 與 Vault 基礎 |
| 第二階段 | [第 2 章](./chapter-02-docker-gitlab.md) | Docker & GitLab CI/CD | 自動化部署 GitLab + GitOps 閉環 |
| 第三階段 | [第 3 章](./chapter-03-kernel-tuning.md) | 高併發核心參數調優 | 系統效能強化 Role |
| 第四階段 | [第 4 章](./chapter-04-security-hardening.md) | 系統資安強化 | 資安防護中心 Role |
| 第五階段 | [第 5 章](./chapter-05-engineering-ci.md) | 工程化與 CI Integration | 端到端自動化流程 |

### 第二部分：ISO 27001 合規自動化（第 6-9 章）

| 階段 | 章節 | 核心主題 | ISO 27001 控制措施 | 產出目標 |
|------|------|----------|-------------------|----------|
| 第六階段 | [第 6 章](./chapter-06-asset-management.md) | IT 資產管理（Snipe-IT）| A.5.9 | 自動維護的資產清冊 |
| 第七階段 | [第 7 章](./chapter-07-cis-benchmark.md) | CIS Benchmark 合規掃描 | A.8.8 / A.8.9 | 合規率報告 + 自動修復 |
| 第八階段 | [第 8 章](./chapter-08-evidence-collector.md) | 稽核證據收集器 | A.5.16 / A.8.3 / A.8.13 | 每週稽核 Markdown 報告 |
| 第九階段 | [第 9 章](./chapter-09-scheduled-audit-pipeline.md) | 定期稽核流水線 | A.8.16（持續監控）| 52 週完整稽核時間線 |

### 第三部分：進階合規自動化（第 10-12 章）

| 階段 | 章節 | 核心主題 | ISO 27001 控制措施 | 產出目標 |
|------|------|----------|-------------------|----------|
| 第十階段 | [第 10 章](./chapter-10-ssl-certificate-management.md) | SSL 憑證監控與自動續期 | A.8.24 | 憑證到期告警 + certbot 自動續期 |
| 第十一階段 | [第 11 章](./chapter-11-account-lifecycle.md) | 帳號生命週期自動化 | A.5.16 / A.5.18 / A.8.2 | 入離職流程 + 季度存取審查報告 |
| 第十二階段 | [第 12 章](./chapter-12-backup-dr-validation.md) | 備份還原驗證與 DR 演練 | A.8.13 | RTO/RPO 實測記錄 + 月度 DR 演練 |

### 第四部分：工具整合擴展（第 13-20 章）

> **定位說明：** 本部分的章節以 Ansible 作為**部署與整合的橋樑**，核心功能由各專業工具提供。適合在完成前三部分後，依需求選讀。

| 章節 | 核心工具 | Ansible 的角色 | ISO 27001 |
|------|---------|--------------|-----------|
| [第 13 章](./chapter-13-trivy-container-scanning.md) | Trivy + GitLab CI | 安裝 Binary、更新漏洞 DB | A.8.25 / A.8.28 |
| [第 14 章](./chapter-14-loki-centralized-logging.md) | Loki + Grafana + Promtail | 部署整個觀測棧、統一設定 | A.8.15 / A.8.16 |
| [第 15 章](./chapter-15-openvas-vulnerability-scanning.md) | OpenVAS / GVM | 部署容器、API 觸發掃描 | A.8.8 |
| [第 16 章](./chapter-16-hashicorp-vault-secret-rotation.md) | HashiCorp Vault | 部署、初始化、API 輪換 | A.5.17 |
| [第 17 章](./chapter-17-gitleaks-secret-scanning.md) | GitLeaks | 安裝 Binary、pre-commit hook | A.8.10 / A.5.33 |
| [第 18 章](./chapter-18-change-management.md) | GitLab MR Rules | 透過 API 設定 Protected Branch | A.8.32 |
| [第 19 章](./chapter-19-eramba-grc.md) | Eramba GRC | 部署容器、API 匯入資產 | A.5.19 / A.5.22 |
| [第 20 章](./chapter-20-gophish-awareness-training.md) | GoPhish | 部署容器、API 建立演練 | A.6.3 |

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
│   ├── gitlab_runner/             # GitLab Runner 自動註冊
│   ├── kernel_tuning/             # 第3章：核心調優 Role
│   ├── security_hardening/        # 第4章：資安強化 Role
│   ├── monitoring/                # 監控 Role（node_exporter）
│   ├── snipeit/                   # 第6章：Snipe-IT 資產管理
│   ├── asset_register/            # 第6章：資產自動註冊
│   ├── UBUNTU24-CIS/              # 第7章：CIS Benchmark（ansible-lockdown）
│   ├── cis_audit/                 # 第7章：CIS 掃描包裝器
│   ├── evidence_collector/        # 第8章：稽核證據收集器
│   ├── cert_monitor/              # 第10章：SSL 憑證監控
│   ├── certbot/                   # 第10章：Let's Encrypt 自動續期
│   ├── account_lifecycle/         # 第11章：帳號生命週期管理
│   └── backup_validator/          # 第12章：備份還原驗證
├── playbooks/
│   ├── site.yml                   # 主入口 Playbook
│   ├── deploy_gitlab.yml
│   ├── kernel_tuning.yml
│   ├── security_hardening.yml
│   ├── deploy_snipeit.yml         # 第6章
│   ├── cis_audit.yml              # 第7章：CIS 掃描
│   ├── cis_remediate.yml          # 第7章：CIS 修復
│   ├── evidence_collector.yml     # 第8章：稽核證據收集
│   ├── cert_monitor.yml           # 第10章：憑證掃描
│   ├── onboard_user.yml           # 第11章：入職帳號建立
│   ├── offboard_user.yml          # 第11章：離職帳號撤銷
│   ├── access_review.yml          # 第11章：季度存取審查
│   ├── backup.yml                 # 第12章：備份執行
│   └── dr_validation.yml          # 第12章：DR 演練驗證
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

*教材版本：v4.0 | 更新日期：2026-04 | 適用 Ansible Core 2.16+ | 共 20 章*
