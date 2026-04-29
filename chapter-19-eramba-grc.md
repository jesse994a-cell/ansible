# 第 19 章：供應商安全評估與 GRC（Eramba）

> **工具定位：** Eramba 是獨立的 GRC 平台。**Ansible 負責部署 Eramba 容器**，並透過 Eramba API 將前面各章自動化收集的資料（資產、稽核報告）匯入，作為 ISO 27001 管理層文件的集中平台。
>
> **ISO 27001 對應：** A.5.19（供應商關係中的資訊安全）、A.5.20（供應商協議中的資訊安全）、A.5.22（供應商服務的監控、審查及變更管理）

---

## 19.1 理論說明：GRC 平台在 ISO 27001 中的角色

GRC（Governance, Risk, Compliance）平台是 ISMS 的管理層，管理前面各章技術工具**無法涵蓋的文件與流程**：

```
技術層（Ansible 自動化）        管理層（Eramba GRC）
─────────────────────────      ──────────────────────────
✅ 資產清冊（Snipe-IT）   →    ✅ 資產風險評估
✅ CIS 合規報告           →    ✅ 不符合項目追蹤（NC Tracker）
✅ Evidence Collector    →    ✅ 稽核記錄管理
✅ DR 演練報告            →    ✅ 業務連續性計畫（BCP）
                               ✅ 供應商安全評估（本章核心）
                               ✅ 適用性聲明書（SoA）
                               ✅ 風險處理計畫（RTP）
```

---

## 19.2 Ansible Role：eramba

```bash
ansible-galaxy role init roles/eramba
```

### 19.2.1 defaults/main.yml

```yaml
# roles/eramba/defaults/main.yml
---
eramba_image: "eramba/community:latest"
eramba_data_path: /opt/eramba
eramba_port: 8443
eramba_network: eramba_net

eramba_db_name: eramba
eramba_db_user: eramba
eramba_db_password: "{{ vault_eramba_db_password }}"

eramba_app_url: "https://{{ ansible_host }}:{{ eramba_port }}"

# 供應商清單（從外部 YAML 讀取）
eramba_vendors_file: "{{ playbook_dir }}/../inventory/vendors/vendors.yml"
```

### 19.2.2 tasks/main.yml（部署 Eramba）

```yaml
# roles/eramba/tasks/main.yml
---
- name: Create Eramba data directories
  ansible.builtin.file:
    path: "{{ eramba_data_path }}"
    state: directory
    mode: '0755'

- name: Create Eramba Docker network
  community.docker.docker_network:
    name: "{{ eramba_network }}"
    state: present

- name: Deploy Eramba MySQL
  community.docker.docker_container:
    name: eramba-mysql
    image: "mysql:8.0"
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ eramba_network }}"
    volumes:
      - "{{ eramba_data_path }}/mysql:/var/lib/mysql"
    env:
      MYSQL_ROOT_PASSWORD: "{{ vault_mysql_root_password }}"
      MYSQL_DATABASE: "{{ eramba_db_name }}"
      MYSQL_USER: "{{ eramba_db_user }}"
      MYSQL_PASSWORD: "{{ eramba_db_password }}"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 10

- name: Deploy Eramba application
  community.docker.docker_container:
    name: eramba-app
    image: "{{ eramba_image }}"
    state: started
    restart_policy: unless-stopped
    published_ports:
      - "{{ eramba_port }}:443"
    networks:
      - name: "{{ eramba_network }}"
    volumes:
      - "{{ eramba_data_path }}/app:/var/www/eramba/app/tmp"
    env:
      DB_HOST: eramba-mysql
      DB_NAME: "{{ eramba_db_name }}"
      DB_USER: "{{ eramba_db_user }}"
      DB_PASS: "{{ eramba_db_password }}"
      APP_URL: "{{ eramba_app_url }}"
```

---

## 19.3 供應商安全評估自動化

### 19.3.1 供應商資料 YAML 格式

```yaml
# inventory/vendors/vendors.yml
---
vendors:
  - name: "AWS"
    type: "IaaS Cloud Provider"
    criticality: high
    services:
      - EC2
      - S3
      - RDS
    compliance_certs:
      - ISO 27001
      - SOC 2 Type II
    cert_verify_url: "https://aws.amazon.com/compliance/iso-27001-faqs/"
    annual_review_date: "2026-12-01"
    risk_owner: "infra-team"
    notes: "主要雲端基礎設施供應商"

  - name: "GitLab"
    type: "SaaS / Self-hosted"
    criticality: high
    services:
      - Source Code Management
      - CI/CD
    compliance_certs:
      - ISO 27001
      - SOC 2 Type II
    annual_review_date: "2026-06-01"
    risk_owner: "dev-team"
```

### 19.3.2 自動匯入供應商資料到 Eramba

```yaml
# roles/eramba/tasks/import_vendors.yml
---
- name: Load vendor data
  ansible.builtin.include_vars:
    file: "{{ eramba_vendors_file }}"
    name: vendor_data

- name: Get Eramba auth token
  ansible.builtin.uri:
    url: "https://localhost:{{ eramba_port }}/api/v1/auth/token"
    method: POST
    validate_certs: false
    body_format: json
    body:
      username: "{{ vault_eramba_admin_user }}"
      password: "{{ vault_eramba_admin_password }}"
    return_content: true
  register: eramba_auth
  delegate_to: localhost

- name: Create or update vendor records in Eramba
  ansible.builtin.uri:
    url: "https://localhost:{{ eramba_port }}/api/v1/third-party-risks"
    method: POST
    validate_certs: false
    headers:
      Authorization: "Bearer {{ eramba_auth.json.token }}"
    body_format: json
    body:
      name: "{{ item.name }}"
      status: "active"
      criticality: "{{ item.criticality }}"
      description: "{{ item.notes | default('') }}"
      compliance_certifications: "{{ item.compliance_certs | join(', ') }}"
      review_date: "{{ item.annual_review_date }}"
    status_code: [200, 201]
  loop: "{{ vendor_data.vendors }}"
  loop_control:
    label: "{{ item.name }}"
  delegate_to: localhost
```

### 19.3.3 自動產出供應商年度評估報告

```yaml
# playbooks/vendor_assessment.yml
---
- name: Generate annual vendor assessment report
  hosts: localhost
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  tasks:
    - name: Load vendor list
      ansible.builtin.include_vars:
        file: ../inventory/vendors/vendors.yml
        name: vendor_data

    - name: Check compliance certs (fetch from public URLs)
      ansible.builtin.uri:
        url: "{{ item.cert_verify_url }}"
        method: GET
        return_content: false
        status_code: [200]
      register: cert_check
      loop: "{{ vendor_data.vendors | selectattr('cert_verify_url', 'defined') | list }}"
      loop_control:
        label: "{{ item.name }}"
      ignore_errors: true

    - name: Generate vendor assessment Markdown
      ansible.builtin.template:
        src: vendor_assessment.md.j2
        dest: "./vendor-assessment-{{ ansible_date_time.date }}.md"
        mode: '0644'

    - name: Commit to audit-evidence
      ansible.builtin.include_tasks:
        file: ../roles/evidence_collector/tasks/git_commit.yml
      vars:
        evidence_report_dir: "."
        audit_evidence_local_path: /opt/audit-evidence-repo
```

---

## 19.4 驗證步驟

```bash
# 1. 部署 Eramba
ansible-playbook playbooks/deploy_eramba.yml \
  --vault-password-file ~/.vault_pass

# 2. 確認 Eramba Web UI 可存取
curl -kI https://localhost:8443/
# 預期：HTTP 200 or 302 (redirect to login)

# 3. 匯入供應商資料
ansible-playbook playbooks/vendor_assessment.yml \
  --vault-password-file ~/.vault_pass

# 4. 確認報告已產出
cat vendor-assessment-$(date +%Y-%m-%d).md

# 5. 在 Eramba UI 確認供應商已匯入
# https://localhost:8443 → Third Party Risk → Vendors
```

---

## 19.5 章節小結

| 角色分工 | 說明 |
|----------|------|
| **Ansible 負責** | 部署 Eramba 容器、透過 API 匯入資產與供應商資料、產出年度評估報告 |
| **Eramba 負責** | ISO 27001 控制措施進度追蹤、SoA、風險登記、供應商評估管理 |
| **適用場景** | 需要完整 ISO 27001 ISMS 文件管理、有專職 ISMS 人員的組織 |

*← [返回總覽](./README.md) | [上一章](./chapter-18-change-management.md)*
