# 第 18 章：異動管理（GitLab Change Management）

> **工具定位：** 異動管理的核心機制是 GitLab 的 Protected Branches、Approval Rules、MR Templates。**Ansible 的角色是透過 GitLab API 自動設定這些規則**，確保每個 Project 都有一致的異動管控設定，同時提供 Ansible Playbook 作為「變更執行的標準動作」。
>
> **ISO 27001 對應：** A.8.32（變更管理）— 所有對資訊處理設施的變更都必須受控管理。

---

## 18.1 理論說明：為什麼 GitOps 本身就是異動管理？

ISO 27001 A.8.32 要求的異動管理包含：

1. **變更申請** — 清楚描述要改什麼、為什麼
2. **影響評估** — 評估對業務的影響
3. **審查核准** — 有授權人員審查並核准
4. **回滾計畫** — 萬一失敗如何回復
5. **變更記錄** — 完整的執行紀錄

在 GitLab 的 GitOps 框架下，**每個 MR（Merge Request）本身就是一份異動管理記錄**：
- MR 描述 = 變更申請 + 影響評估
- MR Review = 審查核准
- Git revert = 回滾計畫
- CI/CD 日誌 = 執行記錄
- Git commit = 完整審計軌跡

---

## 18.2 Ansible 透過 GitLab API 設定異動管理規則

```bash
ansible-galaxy role init roles/gitlab_change_mgmt
```

### 18.2.1 defaults/main.yml

```yaml
# roles/gitlab_change_mgmt/defaults/main.yml
---
gitlab_api_url: "http://{{ groups['gitlab_servers'][0] }}/api/v4"
gitlab_api_token: "{{ vault_gitlab_api_token }}"

# 需要設定異動管理的 Projects
gitlab_protected_projects:
  - project_id: 1          # ansible-course
    project_name: "ansible-course"

# Protected Branch 設定
change_mgmt_default_branch: "main"
change_mgmt_approvals_required: 2   # 需要 2 人 Approve（高風險環境）

# MR 模板（所有 Project 統一使用）
change_mgmt_mr_template_path: ".gitlab/merge_request_templates/change_request.md"
```

### 18.2.2 tasks/main.yml

```yaml
# roles/gitlab_change_mgmt/tasks/main.yml
---
- name: Configure Protected Branches
  ansible.builtin.include_tasks: protected_branches.yml
  loop: "{{ gitlab_protected_projects }}"
  loop_control:
    loop_var: project

- name: Configure Approval Rules
  ansible.builtin.include_tasks: approval_rules.yml
  loop: "{{ gitlab_protected_projects }}"
  loop_control:
    loop_var: project

- name: Deploy MR Template to each project
  ansible.builtin.include_tasks: mr_template.yml
  loop: "{{ gitlab_protected_projects }}"
  loop_control:
    loop_var: project
```

### 18.2.3 tasks/protected_branches.yml

```yaml
# roles/gitlab_change_mgmt/tasks/protected_branches.yml
---
- name: Protect main branch (no direct push)
  ansible.builtin.uri:
    url: "{{ gitlab_api_url }}/projects/{{ project.project_id }}/protected_branches"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_api_token }}"
    body_format: json
    body:
      name: "{{ change_mgmt_default_branch }}"
      push_access_level: 0          # 0 = No one（強制透過 MR）
      merge_access_level: 40        # 40 = Maintainers
      allow_force_push: false
      code_owner_approval_required: true
    status_code: [200, 201, 409]   # 409 = 已存在，不視為錯誤
  delegate_to: localhost

- name: Protect release/* branches
  ansible.builtin.uri:
    url: "{{ gitlab_api_url }}/projects/{{ project.project_id }}/protected_branches"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_api_token }}"
    body_format: json
    body:
      name: "release/*"
      push_access_level: 30         # 30 = Developers+
      merge_access_level: 40
      allow_force_push: false
    status_code: [200, 201, 409]
  delegate_to: localhost
```

### 18.2.4 tasks/approval_rules.yml

```yaml
# roles/gitlab_change_mgmt/tasks/approval_rules.yml
---
- name: Set MR approval requirement
  ansible.builtin.uri:
    url: "{{ gitlab_api_url }}/projects/{{ project.project_id }}/approvals"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_api_token }}"
    body_format: json
    body:
      approvals_before_merge: "{{ change_mgmt_approvals_required }}"
      # 不允許作者自己 approve 自己的 MR
      merge_requests_author_approval: false
      # 需要 Code Owner 審查（若有設定 CODEOWNERS 檔案）
      require_code_owner_approval: true
      # Reset approval when new commit is pushed（避免 approve 後再改代碼）
      reset_approvals_on_push: true
    status_code: [200, 201]
  delegate_to: localhost
```

### 18.2.5 MR 模板：變更申請單

```yaml
# roles/gitlab_change_mgmt/tasks/mr_template.yml
---
- name: Create MR template directory via API
  ansible.builtin.uri:
    url: "{{ gitlab_api_url }}/projects/{{ project.project_id }}/repository/files/.gitlab%2Fmerge_request_templates%2Fchange_request.md"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_api_token }}"
    body_format: json
    body:
      branch: "{{ change_mgmt_default_branch }}"
      content: "{{ lookup('template', 'mr_change_request.md.j2') | b64encode }}"
      encoding: base64
      commit_message: "ci: add standard change request MR template"
      author_name: "Ansible Change Mgmt Bot"
    status_code: [200, 201, 400]  # 400 = 檔案已存在
  delegate_to: localhost
```

```jinja2
{# roles/gitlab_change_mgmt/templates/mr_change_request.md.j2 #}
## 異動申請單（Change Request）

> 本 MR 採用公司標準異動管理流程。合併前請確認所有核取項目已完成。

### 1. 變更說明

**變更類型：** <!-- 選擇：新功能 / Bug Fix / 安全更新 / 設定變更 / 基礎設施 -->

**變更摘要：**
<!-- 用一句話描述此次變更 -->

**詳細說明：**
<!-- 為什麼需要這個變更？解決了什麼問題？ -->

---

### 2. 影響範圍評估

**影響的系統 / 服務：**
- [ ] Web 伺服器（web-01, web-02）
- [ ] 資料庫（db-01）
- [ ] GitLab
- [ ] 其他：___________

**預計影響程度：**
- [ ] 無服務中斷（Hot change）
- [ ] 需要短暫重啟（計畫停機：___ 分鐘）
- [ ] 重大影響（需要變更視窗）

---

### 3. 測試驗證

**已完成的測試：**
- [ ] 在 staging 環境驗證
- [ ] Molecule 測試通過
- [ ] Ansible dry-run（--check --diff）已執行

**驗證結果摘要：**
```
# 貼上 ansible-playbook --check 的相關輸出
```

---

### 4. 回滾計畫

**回滾方式：**
<!-- 若變更失敗，如何快速還原？ -->

**回滾指令：**
```bash
# git revert <commit-hash>
# 或其他回滾步驟
```

**回滾時間估計：** ___ 分鐘

---

### 5. 審查核准

- [ ] 技術審查完成（由 Reviewer 勾選）
- [ ] 影響範圍確認無誤
- [ ] 回滾計畫可行
- [ ] CI/CD 所有 job 通過

<!-- 審查者請在 Approve 前確認以上清單 -->
