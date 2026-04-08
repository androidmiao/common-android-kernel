---
type: entity
kernel_path: kernel/cgroup/
config_option: CONFIG_CGROUPS
upstream: partial
related:
  - ../subsystems/scheduler.md
  - ../subsystems/memory-management.md
  - ../subsystems/block-layer.md
  - ../concepts/vendor-hooks.md
  - ../concepts/bpf.md
last_updated: 2026-04-09
---

# Cgroup Controllers — 行程資源群組控制器

## Overview

Control Groups（cgroups）是 Linux 核心的行程分組機制，用於限制、計量和隔離行程對系統資源（CPU、記憶體、I/O、PID 等）的使用。核心框架為上游 Linux `[upstream]`，但 ACK 加入了 3 個 Android vendor hooks `[android]` 用於 CPU cgroup 操作。

在 Android 中，cgroups 是至關重要的基礎設施：`ActivityManagerService` 使用 cgroups 管理前景/背景行程的 CPU 配額、記憶體限額和 I/O 優先權，`lmkd` 使用 memory cgroup 的 PSI 壓力資訊觸發 low memory killer。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| **核心框架** | | |
| `kernel/cgroup/cgroup.c` | 7,452 | Cgroup 核心：階層管理、任務遷移、BPF 整合 |
| `kernel/cgroup/cgroup-internal.h` | — | 內部資料結構 |
| `kernel/cgroup/rstat.c` | 762 | 運行統計聚合 |
| `kernel/cgroup/namespace.c` | 144 | Cgroup namespace 操作 |
| `kernel/cgroup/cgroup-v1.c` | 1,373 | Legacy cgroup v1 介面 |
| **控制器** | | |
| `kernel/cgroup/cpuset.c` | 4,552 | CPU/記憶體節點綁定、分區 |
| `kernel/cgroup/cpuset-v1.c` | 607 | Legacy CPUset v1 介面 |
| `kernel/cgroup/freezer.c` | 326 | 行程凍結/解凍（cgroup v2） |
| `kernel/cgroup/legacy_freezer.c` | 474 | Legacy 凍結器（cgroup v1） |
| `kernel/cgroup/pids.c` | 460 | PID 數量限制 |
| `kernel/cgroup/misc.c` | 478 | 雜項資源限制（SEV/TDX） |
| `kernel/cgroup/dmem.c` | 830 | 裝置記憶體限制（GPU 等） |
| `kernel/cgroup/rdma.c` | 612 | RDMA 資源限制 |
| `kernel/cgroup/debug.c` | 377 | 除錯介面 |
| **外部控制器** | | |
| `mm/memcontrol.c` | ~10,000+ | 記憶體 cgroup（記憶體限額/計量） |
| `block/blk-cgroup.c` | ~3,000+ | Block I/O cgroup（頻寬/IOPS） |
| **總計** | **~18,447** | （僅 kernel/cgroup/） |

## Implementation Details

### 核心框架（cgroup.c）

**核心資料結構：**
- **`struct cgroup_root`** — Cgroup 階層根節點
- **`struct cgroup`** — 個別 cgroup 節點
- **`struct css_set`** — 任務的子系統狀態集合
- **`struct cgrp_cset_link`** — Cgroup ↔ css_set 的 M:N 連結
- **`struct cgroup_subsys_state` (css)** — 每個子系統的基礎狀態

**鎖定機制：**
- `cgroup_mutex` — 階層修改的主鎖
- `css_set_lock` — 保護 `task->cgroups` 指標和 css_set 鏈
- `cgroup_idr_lock` — 保護 cgroup/css ID 釋放
- `cgroup_threadgroup_rwsem` — 保護 threadgroup 變更

**關鍵函式：**
- `cgroup_attach_task()` — 任務附加至 cgroup
- `cgroup_migrate()` — 任務遷移
- `cgroup_mkdir()` / `cgroup_rmdir()` — 建立/刪除 cgroup
- `cgroup_bpf_get()` / `cgroup_bpf_put()` — BPF 程式附加

### 控制器概覽

#### CPUset（cpuset.c — 4,552 行）

CPU 和記憶體節點綁定控制器。限制行程只能在指定的 CPU 和 NUMA 節點上執行。

- 支援分區（isolated partitions、remote partitions）
- Hotplug 處理與排程域重建
- Housekeeping CPU 隔離
- 2 個 Android vendor hooks：`android_rvh_cpu_cgroup_attach()`、`android_rvh_cpu_cgroup_online()`

#### Freezer（freezer.c — 326 行）

Cgroup v2 行程凍結/解凍控制器。

- `cgroup_freeze()` — 凍結/解凍 cgroup 階層
- `cgroup_enter_frozen()` / `cgroup_leave_frozen()` — 任務凍結狀態管理
- `CGRP_FREEZE` 旗標、`freezer.nr_frozen_tasks` 計數器
- `freezer.freeze_seq` seqcount 保護凍結狀態變更

#### PIDs（pids.c — 460 行）

PID 數量限制控制器，防止 fork bomb 攻擊。

- 階層式計費：`pids_try_charge()` / `pids_uncharge()`
- 檔案介面：`pids.max`、`pids.current`、`pids.peak`、`pids.events`
- `pids_can_fork()` — fork 前檢查是否超限

#### Memory Cgroup（mm/memcontrol.c — ~10,000+ 行）

記憶體資源限制與計量。

- 頁面計數器階層（`page_counter`）
- Socket / Kernel / BPF 記憶體計量
- PSI（Pressure Stall Information）壓力報告
- OOM 處理與回收

#### Block Cgroup（block/blk-cgroup.c — ~3,000+ 行）

Block I/O 資源限制。

- Throttle 策略（BPS/IOPS 限制）
- I/O 優先權
- iocost / iolatency QoS 策略

#### Device Memory（dmem.c — 830 行）

裝置記憶體（GPU、加速器）資源限制。

- 區域式架構（`struct dmem_cgroup_region`）
- 每 CSS 池（`struct dmem_cgroup_pool_state`）使用 `page_counter`
- RCU + spinlock 鎖定模型
- Android 意義：GPU 記憶體計量

#### Misc（misc.c — 478 行）

雜項資源限制：AMD SEV ASIDs、AMD SEV-ES ASIDs、Intel TDX Host HKIDs。

#### RDMA（rdma.c — 612 行）

RDMA 資源限制：HCA handles 和 HCA objects。

## Userspace Interface

### Cgroup v2（統一階層）
```
/sys/fs/cgroup/
├── cgroup.controllers       # 可用控制器
├── cgroup.subtree_control   # 啟用的控制器
├── system.slice/
│   ├── cpu.max              # CPU 配額
│   ├── memory.max           # 記憶體限額
│   ├── io.max               # I/O 限制
│   ├── pids.max             # PID 限制
│   └── cgroup.freeze        # 凍結控制
└── user.slice/
    └── ...
```

### Cgroup v1（Legacy，Android 仍大量使用）
```
/dev/cpuctl/                 # CPU 控制器
/dev/cpuset/                 # CPUset 控制器
/dev/memcg/                  # 記憶體控制器
/dev/blkio/                  # Block I/O 控制器
/dev/stune/                  # 排程調頻（Android 專屬）
```

## Android-Specific Notes

### Vendor Hooks

`include/trace/hooks/cgroup.h` 定義 3 個 hooks：

1. **`android_vh_cgroup_attach`** — 通用 cgroup attach hook，任務附加至任何 cgroup 時觸發
2. **`android_rvh_cpu_cgroup_attach`** — CPU cgroup attach restricted hook，可控制任務附加策略
3. **`android_rvh_cpu_cgroup_online`** — CPU cgroup online restricted hook，控制 cgroup 建立策略

Vendor hooks 呼叫位置：`cgroup.c:2756`（`trace_android_vh_cgroup_attach()`）。

### Android 使用模式

- **ActivityManagerService**：根據行程優先權（前景/背景/快取）將行程移至不同 cgroup，調整 CPU 配額和記憶體限額
- **lmkd**：監控 memory cgroup 的 PSI 壓力指標，觸發低記憶體殺手
- **EAS（Energy Aware Scheduling）**：透過 cpuset 和 CPU cgroup 控制大小核分配
- **Thermal throttling**：透過 vendor hooks 動態調整 CPU cgroup 行為

## Configuration

| 選項 | 說明 |
|------|------|
| `CONFIG_CGROUPS` | 啟用 cgroup 框架 |
| `CONFIG_CGROUP_CPUACCT` | CPU 使用計量 |
| `CONFIG_CGROUP_SCHED` | CPU 排程控制 |
| `CONFIG_CPUSETS` | CPUset 控制器 |
| `CONFIG_CGROUP_FREEZER` | 凍結控制器 |
| `CONFIG_CGROUP_PIDS` | PID 限制 |
| `CONFIG_MEMCG` | 記憶體 cgroup |
| `CONFIG_BLK_CGROUP` | Block I/O cgroup |
| `CONFIG_CGROUP_DMEM` | 裝置記憶體 cgroup |
| `CONFIG_PSI` | Pressure Stall Information |

## Cross-References

- [Scheduler 子系統](../subsystems/scheduler.md) — CPU cgroup 排程整合、78 個排程 vendor hooks
- [記憶體管理子系統](../subsystems/memory-management.md) — Memory cgroup、OOM、PSI
- [Block Layer 子系統](../subsystems/block-layer.md) — Block cgroup、I/O QoS
- [Vendor Hooks](../concepts/vendor-hooks.md) — 3 個 cgroup vendor hooks 定義
- [BPF](../concepts/bpf.md) — BPF-Cgroup 整合、socket/device/sysctl 控制
