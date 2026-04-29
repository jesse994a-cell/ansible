# 第 3 章：高併發核心參數調優（Kernel Tuning）

> **學習目標：** 深度理解 Linux TCP Stack 的運作原理，使用 `ansible.posix.sysctl` 模組自動化調整核心參數，並整合 `node_exporter` 監控，用數據驗證調優效果。

---

## 3.1 理論說明：高併發系統的瓶頸在哪裡？

在高併發場景（如 Web API Server、反向代理、資料庫），系統可能在達到 CPU 或記憶體上限之前，就因為**核心網路參數配置不當**而無法承接更多連線。

### 3.1.1 TCP 連線的生命週期

```
Client                              Server
  │                                   │
  │──── SYN ──────────────────────►   │  ← SYN 進入 backlog 佇列
  │                                   │    (net.ipv4.tcp_max_syn_backlog)
  │   ◄── SYN-ACK ────────────────    │
  │                                   │
  │──── ACK ──────────────────────►   │  ← 3-way handshake 完成
  │                                   │    進入 accept 佇列
  │                                   │    (net.core.somaxconn)
  │   [資料傳輸中]                     │
  │                                   │
  │──── FIN ──────────────────────►   │
  │   ◄── ACK ────────────────────    │
  │   ◄── FIN ────────────────────    │  ← 進入 TIME_WAIT 狀態
  │──── ACK ──────────────────────►   │    持續 2×MSL（約 60 秒）
  │                                   │
```

### 3.1.2 關鍵瓶頸解析

**問題一：SYN Backlog 溢出**

當每秒新連線數超過 `net.ipv4.tcp_max_syn_backlog`，新的 SYN 包會被丟棄，客戶端看到連線逾時。

**問題二：Accept Queue 溢出**

完成三次握手的連線等待應用程式呼叫 `accept()` 時，若佇列長度（`net.core.somaxconn`）不足，連線被重置。

**問題三：TIME_WAIT 狀態堆積**

每個短連線關閉後，Server 端進入 TIME_WAIT，佔用 socket（`(src_ip, src_port, dst_ip, dst_port)` 四元組）約 60 秒。高併發時 TIME_WAIT 可堆積到數萬個，耗盡本地埠號（`net.ipv4.ip_local_port_range` 預設僅約 28,000 個）。

**問題四：File Descriptor 上限**

Linux 的「一切皆檔案」哲學讓每個 socket 也消耗一個 fd（file descriptor）。預設的 `nofile` 上限（1024）在高併發下瞬間耗盡，導致 `Too many open files` 錯誤。

---

## 3.2 核心參數深度解說

### 3.2.1 TCP 接收佇列參數

| 參數 | 預設值 | 建議值 | 說明 |
|------|--------|--------|------|
| `net.core.somaxconn` | 4096 | 65535 | Accept queue 最大長度。應用程式 `listen()` 的 backlog 不能超過此值。 |
| `net.ipv4.tcp_max_syn_backlog` | 512 | 65535 | SYN queue 最大長度。接受但尚未完成 3-way handshake 的連線數。 |
| `net.core.netdev_max_backlog` | 1000 | 65535 | 網卡驅動層的接收佇列，當核心處理速度跟不上網卡時使用。 |

### 3.2.2 TIME_WAIT 優化

| 參數 | 預設值 | 建議值 | 說明 |
|------|--------|--------|------|
| `net.ipv4.tcp_tw_reuse` | 0 | 1 | **允許 TIME_WAIT socket 被新連線重用**（僅對 Outgoing 連線有效）。啟用前提：`net.ipv4.tcp_timestamps=1`。 |
| `net.ipv4.tcp_fin_timeout` | 60 | 30 | FIN_WAIT_2 狀態的最長等待時間（秒）。縮短可加速資源釋放。 |
| `net.ipv4.tcp_max_tw_buckets` | 16384 | 262144 | 系統允許同時存在的 TIME_WAIT socket 最大數量，超過的連線直接關閉。 |

> **關於 `tcp_tw_reuse` 的重要說明：**
>
> `tcp_tw_reuse` 允許核心將 TIME_WAIT 的 socket 重用於**新的出站連線**（如 Nginx upstream 連到後端），而非用於接受新的入站連線。它依賴 TCP Timestamps（RFC 1323）確保不會誤判，所以必須同時開啟 `tcp_timestamps`。
>
> ⚠️ `net.ipv4.tcp_tw_recycle` 在 Linux 4.12 已被移除，**不要使用**，它在 NAT 環境下會導致連線問題。

### 3.2.3 TCP 視窗與緩衝區

| 參數 | 預設值 | 建議值 | 說明 |
|------|--------|--------|------|
| `net.ipv4.tcp_window_scaling` | 1 | 1 | RFC 1323 視窗縮放，允許 TCP 視窗超過 64KB，對高延遲網路效能影響顯著。 |
| `net.core.rmem_max` | 212992 | 134217728 | Socket 接收緩衝區最大值（128MB）。 |
| `net.core.wmem_max` | 212992 | 134217728 | Socket 傳送緩衝區最大值（128MB）。 |
| `net.ipv4.tcp_rmem` | 4096 87380 6291456 | 4096 87380 67108864 | TCP 接收緩衝：最小/預設/最大（bytes）。 |
| `net.ipv4.tcp_wmem` | 4096 16384 4194304 | 4096 65536 67108864 | TCP 傳送緩衝：最小/預設/最大（bytes）。 |

### 3.2.4 TCP Fast Open（TFO）

TCP Fast Open 允許在 SYN 包中攜帶資料，減少一個 RTT（Round-Trip Time）的延遲，對 HTTP/HTTPS 短連線效果顯著。

```
傳統 TCP：    SYN → SYN-ACK → ACK → Request → Response  （3 RTT）
TCP Fast Open：SYN+Data → SYN-ACK+Data → ACK        （1 RTT）
```

| 參數 | 值 | 說明 |
|------|----|------|
| `net.ipv4.tcp_fastopen` | 3 | 0=關閉, 1=僅 Client, 2=僅 Server, 3=Client+Server |

### 3.2.5 埠號範圍與連線上限

| 參數 | 預設值 | 建議值 | 說明 |
|------|--------|--------|------|
| `net.ipv4.ip_local_port_range` | 32768 60999 | 1024 65535 | 本地出站連線可用的埠號範圍，擴大可容許更多並發出站連線。 |
| `net.ipv4.tcp_timestamps` | 1 | 1 | 保持開啟（`tcp_tw_reuse` 的前提）。 |
| `net.ipv4.tcp_syn_retries` | 6 | 3 | SYN 重試次數，減少可加速失敗連線的判定。 |
| `net.ipv4.tcp_synack_retries` | 5 | 3 | SYN-ACK 重試次數。 |

### 3.2.6 File Descriptor 上限

```bash
# 查看目前的 fd 上限
ulimit -n          # 當前 shell session 的軟限制（預設 1024）
cat /proc/sys/fs/file-max  # 系統層級的最大 fd 數
```

| 位置 | 設定方式 | 影響範圍 |
|------|----------|----------|
| `/proc/sys/fs/file-max` | `sysctl fs.file-max` | 整個系統 |
| `/etc/security/limits.conf` | `<user> soft/hard nofile <N>` | 特定使用者或群組 |
| `/etc/systemd/system/<svc>.service` | `LimitNOFILE=` | 特定 systemd 服務 |

---

## 3.3 Ansible Role 結構：kernel_tuning

```bash
ansible-galaxy role init roles/kernel_tuning
```

```
roles/kernel_tuning/
├── defaults/
│   └── main.yml         ← 所有 sysctl 參數的預設值
├── handlers/
│   └── main.yml         ← reload sysctl
├── tasks/
│   ├── main.yml         ← 任務入口
│   ├── sysctl.yml       ← TCP 核心參數調整
│   ├── limits.yml       ← limits.conf 調整
│   └── validate.yml     ← 驗證參數生效
└── templates/
    ├── 99-kernel-tuning.conf.j2   ← sysctl 設定模板
    └── limits.conf.j2             ← limits 設定模板
```

---

## 3.4 程式碼實作

### 3.4.1 defaults/main.yml

```yaml
# roles/kernel_tuning/defaults/main.yml
---
# ════════════════════════════════════════════════════════════════
# TCP 連線佇列
# ════════════════════════════════════════════════════════════════

# Accept Queue：完成三次握手後等待 accept() 的連線佇列
kernel_net_core_somaxconn: 65535

# SYN Queue：收到 SYN 但尚未完成握手的連線佇列
kernel_tcp_max_syn_backlog: 65535

# 網卡驅動層接收佇列
kernel_net_core_netdev_max_backlog: 65535

# ════════════════════════════════════════════════════════════════
# TIME_WAIT 優化
# ════════════════════════════════════════════════════════════════

# 允許出站連線重用 TIME_WAIT socket（需搭配 tcp_timestamps=1）
kernel_tcp_tw_reuse: 1

# FIN_WAIT_2 逾時時間（秒）
kernel_tcp_fin_timeout: 30

# 系統允許的最大 TIME_WAIT socket 數
kernel_tcp_max_tw_buckets: 262144

# ════════════════════════════════════════════════════════════════
# TCP 效能參數
# ════════════════════════════════════════════════════════════════

# 時間戳記（tw_reuse 的前提，保持 1）
kernel_tcp_timestamps: 1

# 視窗縮放（RFC 1323，高延遲網路必備）
kernel_tcp_window_scaling: 1

# TCP Fast Open（Client+Server 模式）
kernel_tcp_fastopen: 3

# SYN/SYN-ACK 重試次數（減少以加速失敗判定）
kernel_tcp_syn_retries: 3
kernel_tcp_synack_retries: 3

# Keepalive 設定（偵測死亡連線）
kernel_tcp_keepalive_time: 600      # 600s 後開始發 keepalive 探針
kernel_tcp_keepalive_intvl: 30      # 探針間隔 30s
kernel_tcp_keepalive_probes: 3      # 3 次無回應後判定連線斷開

# ════════════════════════════════════════════════════════════════
# Socket 緩衝區
# ════════════════════════════════════════════════════════════════

# 最大 socket 緩衝區（128 MB）
kernel_net_core_rmem_max: 134217728
kernel_net_core_wmem_max: 134217728

# TCP 接收緩衝區（最小/預設/最大，bytes）
kernel_tcp_rmem: "4096 87380 67108864"

# TCP 傳送緩衝區（最小/預設/最大，bytes）
kernel_tcp_wmem: "4096 65536 67108864"

# ════════════════════════════════════════════════════════════════
# 埠號範圍
# ════════════════════════════════════════════════════════════════

# 擴大出站連線可用的埠號範圍（原 32768-60999 共約 28k，擴大到約 64k）
kernel_ip_local_port_range: "1024 65535"

# ════════════════════════════════════════════════════════════════
# File Descriptor 上限
# ════════════════════════════════════════════════════════════════

# 系統層級最大 fd 數（1M）
kernel_fs_file_max: 1048576

# 使用者/服務層級 nofile 軟/硬限制
kernel_nofile_soft: 524288
kernel_nofile_hard: 1048576

# 套用 limits 的目標（* = 所有使用者）
kernel_limits_targets:
  - user: "*"
    type: soft
    item: nofile
    value: "{{ kernel_nofile_soft }}"
  - user: "*"
    type: hard
    item: nofile
    value: "{{ kernel_nofile_hard }}"
  # root 使用者也需要獨立設定
  - user: root
    type: soft
    item: nofile
    value: "{{ kernel_nofile_soft }}"
  - user: root
    type: hard
    item: nofile
    value: "{{ kernel_nofile_hard }}"

# ════════════════════════════════════════════════════════════════
# 虛擬記憶體優化
# ════════════════════════════════════════════════════════════════

# 降低 Swap 使用傾向（0=完全不 swap，100=積極 swap）
kernel_vm_swappiness: 10

# 髒頁面比例上限（超過則觸發寫入）
kernel_vm_dirty_ratio: 60
kernel_vm_dirty_background_ratio: 5
```

### 3.4.2 tasks/main.yml

```yaml
# roles/kernel_tuning/tasks/main.yml
---
- name: Apply sysctl kernel parameters
  ansible.builtin.import_tasks: sysctl.yml
  tags: [kernel, sysctl]

- name: Configure file descriptor limits
  ansible.builtin.import_tasks: limits.yml
  tags: [kernel, limits]

- name: Validate kernel parameters
  ansible.builtin.import_tasks: validate.yml
  tags: [kernel, validate]
```

### 3.4.3 tasks/sysctl.yml（核心實作）

```yaml
# roles/kernel_tuning/tasks/sysctl.yml
---
# ── TCP 連線佇列 ─────────────────────────────────────────────────────────
- name: "TCP Queue | Set net.core.somaxconn (accept queue)"
  ansible.posix.sysctl:
    name: net.core.somaxconn
    value: "{{ kernel_net_core_somaxconn }}"
    state: present
    # 統一寫入到 /etc/sysctl.d/ 下的獨立設定檔（不污染 /etc/sysctl.conf）
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    # reload=true：立即套用，同時持久化到設定檔
    reload: true
  notify: Reload sysctl

- name: "TCP Queue | Set net.ipv4.tcp_max_syn_backlog (SYN queue)"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_max_syn_backlog
    value: "{{ kernel_tcp_max_syn_backlog }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "TCP Queue | Set net.core.netdev_max_backlog (NIC driver queue)"
  ansible.posix.sysctl:
    name: net.core.netdev_max_backlog
    value: "{{ kernel_net_core_netdev_max_backlog }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── TIME_WAIT 優化 ───────────────────────────────────────────────────────
- name: "TIME_WAIT | Enable tcp_timestamps (required for tw_reuse)"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_timestamps
    value: "{{ kernel_tcp_timestamps }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "TIME_WAIT | Enable tcp_tw_reuse for outgoing connections"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_tw_reuse
    value: "{{ kernel_tcp_tw_reuse }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "TIME_WAIT | Reduce tcp_fin_timeout"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_fin_timeout
    value: "{{ kernel_tcp_fin_timeout }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "TIME_WAIT | Set tcp_max_tw_buckets"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_max_tw_buckets
    value: "{{ kernel_tcp_max_tw_buckets }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── TCP Fast Open ────────────────────────────────────────────────────────
- name: "TFO | Enable TCP Fast Open (client+server mode)"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_fastopen
    value: "{{ kernel_tcp_fastopen }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── 連線重試次數 ─────────────────────────────────────────────────────────
- name: "Retry | Set tcp_syn_retries"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_syn_retries
    value: "{{ kernel_tcp_syn_retries }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Retry | Set tcp_synack_retries"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_synack_retries
    value: "{{ kernel_tcp_synack_retries }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── TCP Keepalive ────────────────────────────────────────────────────────
- name: "Keepalive | Set tcp_keepalive_time"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_keepalive_time
    value: "{{ kernel_tcp_keepalive_time }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Keepalive | Set tcp_keepalive_intvl"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_keepalive_intvl
    value: "{{ kernel_tcp_keepalive_intvl }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Keepalive | Set tcp_keepalive_probes"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_keepalive_probes
    value: "{{ kernel_tcp_keepalive_probes }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── Socket 緩衝區 ────────────────────────────────────────────────────────
- name: "Buffer | Set net.core.rmem_max"
  ansible.posix.sysctl:
    name: net.core.rmem_max
    value: "{{ kernel_net_core_rmem_max }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Buffer | Set net.core.wmem_max"
  ansible.posix.sysctl:
    name: net.core.wmem_max
    value: "{{ kernel_net_core_wmem_max }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Buffer | Set net.ipv4.tcp_rmem"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_rmem
    value: "{{ kernel_tcp_rmem }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "Buffer | Set net.ipv4.tcp_wmem"
  ansible.posix.sysctl:
    name: net.ipv4.tcp_wmem
    value: "{{ kernel_tcp_wmem }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── 埠號範圍 ─────────────────────────────────────────────────────────────
- name: "Port Range | Expand ip_local_port_range"
  ansible.posix.sysctl:
    name: net.ipv4.ip_local_port_range
    value: "{{ kernel_ip_local_port_range }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── 系統 fd 上限 ─────────────────────────────────────────────────────────
- name: "FD | Set fs.file-max (system-wide fd limit)"
  ansible.posix.sysctl:
    name: fs.file-max
    value: "{{ kernel_fs_file_max }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── 虛擬記憶體 ───────────────────────────────────────────────────────────
- name: "VM | Set vm.swappiness"
  ansible.posix.sysctl:
    name: vm.swappiness
    value: "{{ kernel_vm_swappiness }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "VM | Set vm.dirty_ratio"
  ansible.posix.sysctl:
    name: vm.dirty_ratio
    value: "{{ kernel_vm_dirty_ratio }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

- name: "VM | Set vm.dirty_background_ratio"
  ansible.posix.sysctl:
    name: vm.dirty_background_ratio
    value: "{{ kernel_vm_dirty_background_ratio }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-kernel-tuning.conf
    reload: true

# ── 確認設定檔內容 ───────────────────────────────────────────────────────
- name: Show generated sysctl config file
  ansible.builtin.command:
    cmd: cat /etc/sysctl.d/99-kernel-tuning.conf
  register: sysctl_content
  changed_when: false

- name: Display sysctl config
  ansible.builtin.debug:
    var: sysctl_content.stdout_lines
```

### 3.4.4 tasks/limits.yml

```yaml
# roles/kernel_tuning/tasks/limits.yml
---
# ── 確保 pam_limits 模組已載入 ──────────────────────────────────────────
- name: Ensure pam_limits is configured in /etc/pam.d/common-session
  ansible.builtin.lineinfile:
    path: /etc/pam.d/common-session
    line: "session required pam_limits.so"
    state: present

# ── 修改 /etc/security/limits.conf ──────────────────────────────────────
- name: Configure /etc/security/limits.conf for nofile
  community.general.pam_limits:
    domain: "{{ item.user }}"
    limit_type: "{{ item.type }}"
    limit_item: "{{ item.item }}"
    value: "{{ item.value }}"
  loop: "{{ kernel_limits_targets }}"

# ── systemd 服務也需要獨立設定（limits.conf 不影響 systemd 服務）───────
- name: Ensure systemd DefaultLimitNOFILE is set
  ansible.builtin.lineinfile:
    path: /etc/systemd/system.conf
    regexp: "^#?DefaultLimitNOFILE="
    line: "DefaultLimitNOFILE={{ kernel_nofile_hard }}"
    state: present
  notify: Reload systemd

- name: Ensure systemd user DefaultLimitNOFILE is set
  ansible.builtin.lineinfile:
    path: /etc/systemd/user.conf
    regexp: "^#?DefaultLimitNOFILE="
    line: "DefaultLimitNOFILE={{ kernel_nofile_hard }}"
    state: present
  notify: Reload systemd
```

### 3.4.5 tasks/validate.yml

```yaml
# roles/kernel_tuning/tasks/validate.yml
---
# ── 驗證關鍵 sysctl 參數 ─────────────────────────────────────────────────
- name: Read back key sysctl values for validation
  ansible.builtin.command:
    cmd: "sysctl -n {{ item.key }}"
  register: sysctl_check
  changed_when: false
  failed_when: sysctl_check.stdout | int != item.expected_value | int
  loop:
    - { key: "net.core.somaxconn",            expected_value: "{{ kernel_net_core_somaxconn }}" }
    - { key: "net.ipv4.tcp_max_syn_backlog",  expected_value: "{{ kernel_tcp_max_syn_backlog }}" }
    - { key: "net.ipv4.tcp_tw_reuse",         expected_value: "{{ kernel_tcp_tw_reuse }}" }
    - { key: "net.ipv4.tcp_fin_timeout",      expected_value: "{{ kernel_tcp_fin_timeout }}" }
    - { key: "fs.file-max",                   expected_value: "{{ kernel_fs_file_max }}" }

- name: Validation passed - all sysctl values are correct
  ansible.builtin.debug:
    msg: "✅ 所有核心參數驗證通過！"
```

### 3.4.6 handlers/main.yml

```yaml
# roles/kernel_tuning/handlers/main.yml
---
- name: Reload sysctl
  ansible.builtin.command:
    cmd: sysctl --system
  changed_when: true

- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true
```

---

## 3.5 監控整合：node_exporter

調優完成後，如何用數據驗證效果？安裝 `node_exporter` 暴露 Linux 系統指標，搭配 Prometheus + Grafana 觀察調優前後的連線數變化。

### 3.5.1 建立 monitoring Role

```bash
ansible-galaxy role init roles/monitoring
```

```yaml
# roles/monitoring/defaults/main.yml
---
node_exporter_version: "1.8.0"
node_exporter_port: 9100
node_exporter_user: node_exporter
node_exporter_install_dir: /opt/node_exporter

# 開放 9100 port 的來源 IP（監控伺服器）
monitoring_allowed_src_ip: "192.168.56.0/24"
```

```yaml
# roles/monitoring/tasks/main.yml
---
# ── 建立 node_exporter 專用使用者（最小權限原則）────────────────────────
- name: Create node_exporter user (no shell, no home)
  ansible.builtin.user:
    name: "{{ node_exporter_user }}"
    shell: /usr/sbin/nologin
    system: true
    create_home: false
    state: present

# ── 下載並安裝 node_exporter ─────────────────────────────────────────────
- name: Create node_exporter install directory
  ansible.builtin.file:
    path: "{{ node_exporter_install_dir }}"
    state: directory
    owner: "{{ node_exporter_user }}"
    mode: '0755'

- name: Download node_exporter binary
  ansible.builtin.get_url:
    url: >-
      https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/
      node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz
    dest: /tmp/node_exporter.tar.gz
    mode: '0644'

- name: Extract node_exporter
  ansible.builtin.unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: "{{ node_exporter_install_dir }}"
    remote_src: true
    extra_opts: ["--strip-components=1"]
    owner: "{{ node_exporter_user }}"

- name: Create symlink to /usr/local/bin
  ansible.builtin.file:
    src: "{{ node_exporter_install_dir }}/node_exporter"
    dest: /usr/local/bin/node_exporter
    state: link

# ── 建立 systemd service ─────────────────────────────────────────────────
- name: Create node_exporter systemd service
  ansible.builtin.template:
    src: node_exporter.service.j2
    dest: /etc/systemd/system/node_exporter.service
    mode: '0644'
  notify:
    - Reload systemd
    - Restart node_exporter

- name: Enable and start node_exporter
  ansible.builtin.service:
    name: node_exporter
    state: started
    enabled: true
```

```jinja2
{# roles/monitoring/templates/node_exporter.service.j2 #}
[Unit]
Description=Prometheus Node Exporter
Documentation=https://github.com/prometheus/node_exporter
After=network-online.target

[Service]
User={{ node_exporter_user }}
Group={{ node_exporter_user }}
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --web.listen-address=":{{ node_exporter_port }}" \
    --collector.systemd \
    --collector.processes \
    --collector.netstat \
    --collector.sockstat \
    --collector.tcpstat
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 3.5.2 關鍵監控指標

調優後，在 Grafana 觀察以下指標：

```promql
# TIME_WAIT socket 數量（應在調優後下降）
node_sockstat_TCP_tw

# 當前 ESTABLISHED 連線數
node_sockstat_TCP_inuse

# SYN backlog 溢出計數（應為 0 或接近 0）
node_netstat_TcpExt_ListenDrops

# Accept queue 溢出計數
node_netstat_TcpExt_ListenOverflows

# 系統已開啟的 fd 數量
node_filefd_allocated

# fd 使用率（不應超過 80%）
node_filefd_allocated / node_filefd_maximum
```

---

## 3.6 完整 Playbook

```yaml
# playbooks/kernel_tuning.yml
---
- name: Apply kernel tuning to web and db servers
  hosts: backend        # 同時對 webservers 和 dbservers 套用
  gather_facts: true

  vars_files:
    - ../vault/secrets.yml

  roles:
    - role: kernel_tuning
      tags: [kernel]
    - role: monitoring
      tags: [monitoring]
```

---

## 3.7 驗證步驟

### 3.7.1 手動驗證核心參數

```bash
# 1. 確認 sysctl 設定已套用
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog net.ipv4.tcp_tw_reuse

# 預期輸出：
# net.core.somaxconn = 65535
# net.ipv4.tcp_max_syn_backlog = 65535
# net.ipv4.tcp_tw_reuse = 1

# 2. 確認設定檔已寫入（重開機後仍生效）
cat /etc/sysctl.d/99-kernel-tuning.conf

# 3. 確認 tcp_tw_reuse 前提（timestamps 必須為 1）
sysctl net.ipv4.tcp_timestamps
# 預期：net.ipv4.tcp_timestamps = 1

# 4. 確認 fd 上限
ulimit -n          # 軟限制，預期 524288
ulimit -Hn         # 硬限制，預期 1048576

# 5. 確認系統層級 fd 上限
sysctl fs.file-max
# 預期：fs.file-max = 1048576

# 6. 確認埠號範圍
cat /proc/sys/net/ipv4/ip_local_port_range
# 預期：1024  65535
```

### 3.7.2 模擬高併發壓力測試

```bash
# 安裝壓測工具
sudo apt install -y wrk apache2-utils

# 使用 wrk 壓測（若本機有 HTTP 服務）
wrk -t12 -c400 -d30s http://localhost/

# 觀察 TIME_WAIT 數量（壓測期間執行）
watch -n 1 'ss -s | grep -E "TIME-WAIT|ESTAB|closed"'

# 觀察 socket 統計
watch -n 1 'cat /proc/net/sockstat'

# 確認沒有 backlog 溢出
netstat -s | grep -E "listen|overflow|drop"
# 理想狀態：「times the listen queue of a socket overflowed: 0」
```

### 3.7.3 確認重開機後設定持久化

```bash
# 模擬重開機（在 VM 中）
sudo reboot

# 重連後驗證
sysctl net.core.somaxconn net.ipv4.tcp_tw_reuse
# 設定應持續存在，因為寫入了 /etc/sysctl.d/99-kernel-tuning.conf
```

---

## 3.8 常見問題排查

**Q：調整 `tcp_tw_reuse=1` 後是否有副作用？**

在 Server 端純接受連線的場景（如 Nginx listen 80），`tcp_tw_reuse` 影響的是**出站連線**（Nginx upstream 連後端），不影響接受入站連線。若 Server 作為 Client 大量短連線某服務（如頻繁呼叫 API），效果最明顯。

**Q：`somaxconn` 調大後，應用程式需要同步修改嗎？**

是的。Nginx、Node.js 等應用程式的 `listen()` backlog 參數不能超過 `somaxconn`，但 Nginx 預設的 backlog 通常已是 511（受舊預設限制）。調整後建議同步修改：

```nginx
# nginx.conf
server {
    listen 80 backlog=65535;
}
```

**Q：`ulimit -n` 顯示的是舊值？**

`/etc/security/limits.conf` 的變更**不影響已登入的 Session**，需要重新登入或重啟服務。對於 systemd 管理的服務，修改後需 `systemctl daemon-reload` 並重啟服務。

---

## 3.9 章節小結

| 學習重點 | 掌握程度確認 |
|----------|-------------|
| 理解 TCP 三次握手與佇列機制 | ☐ 能解釋 SYN queue 與 Accept queue 的差異 |
| `ansible.posix.sysctl` 模組使用 | ☐ 可調整並持久化核心參數 |
| TIME_WAIT 與 `tcp_tw_reuse` 深度理解 | ☐ 能說明為何需要 timestamps 前提 |
| File Descriptor 多層設定 | ☐ 同時設定 sysctl、limits.conf、systemd |
| node_exporter 安裝與指標查詢 | ☐ 可在 Prometheus 查詢 TIME_WAIT 數量變化 |
| 參數生效後手動驗證 | ☐ 重開機後設定持久化 |

**下一章：** [第 4 章 Ubuntu 系統資安強化](./chapter-04-security-hardening.md) — 系統化地強化 SSH、防火牆、自動更新與密鑰管理。

---

*← [返回總覽](./README.md) | [上一章](./chapter-02-docker-gitlab.md)*
