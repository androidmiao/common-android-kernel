---
type: android
scope: kernel-wide
related:
  - "../concepts/vendor-hooks.md"
  - "../entities/vendor-hooks-driver.md"
  - "drivers-android-overview.md"
last_updated: 2026-04-09
---

# Vendor Hook 完整目錄

## 概要

Android Common Kernel 透過 `include/trace/hooks/` 目錄下的 29 個標頭檔宣告所有 vendor hooks。這些 hooks 基於 tracepoint 機制，允許廠商模組在不修改核心程式碼的情況下擴展核心行為。本頁提供所有 hook 標頭檔的完整分類與統計。

## Hook 標頭檔統計

| 標頭檔 | Hook 數量 | 子系統 | 類型 |
|--------|-----------|--------|------|
| `sched.h` | 78 | 排程器 | 混合（VH + RVH） |
| `vendor_hooks.h` | 10 | 框架基礎設施 | 巨集定義 |
| `ufshcd.h` | 9 | UFS 儲存 | VH + RVH |
| `printk.h` | 4 | 核心列印 | VH |
| `avc.h` | 4 | SELinux AVC | RVH |
| `syscall_check.h` | 3 | 系統呼叫 | VH |
| `mm.h` | 3 | 記憶體管理 | VH |
| `iommu.h` | 3 | IOMMU | VH + RVH |
| `cgroup.h` | 3 | Cgroup | VH + RVH |
| `remoteproc.h` | 2 | 遠端處理器 | VH |
| `epoch.h` | 2 | Suspend/Resume | VH |
| `cpuidle_psci.h` | 2 | PSCI 省電 | VH |
| `cpuidle.h` | 2 | CPU 閒置 | VH |
| `cpufreq.h` | 2 | CPU 頻率 | VH + RVH |
| `wqlockup.h` | 1 | Workqueue | VH |
| `vmscan.h` | 1 | 頁面回收 | RVH |
| `timer.h` | 1 | 計時器 | VH |
| `sysrqcrash.h` | 1 | SysRq Crash | VH |
| `sys.h` | 1 | 系統呼叫 | VH |
| `signal.h` | 1 | 信號 | VH |
| `selinux.h` | 1 | SELinux | RVH |
| `reboot.h` | 1 | 重開機 | RVH |
| `pm_domain.h` | 1 | 電源域 | VH |
| `net.h` | 1 | 網路 | VH |
| `mpam.h` | 1 | MPAM | VH |
| `gic_v3.h` | 1 | GICv3 中斷 | VH |
| `gic.h` | 1 | GIC 中斷 | VH |
| `fpsimd.h` | 1 | FPSIMD | VH |
| `debug.h` | 1 | 除錯 | VH |
| **總計** | **~141** | **15+ 子系統** | |

## Hook 類型說明

### DECLARE_HOOK（VH — Vendor Hook）
- 前綴：`android_vh_`
- 允許多個廠商模組同時註冊
- 可以在運行時動態註冊/取消
- 適用於觀察性或可疊加的擴展

### DECLARE_RESTRICTED_HOOK（RVH — Restricted Vendor Hook）
- 前綴：`android_rvh_`
- 僅允許**一個**模組註冊（使用 static_call 優化）
- 一旦註冊不可更改
- 適用於需要獨佔控制的關鍵路徑（如排程決策、安全檢查）

## 按子系統分類

### 排程器（78 hooks — 最多）
`trace/hooks/sched.h` 提供了 Android 核心中最密集的 hook 覆蓋，涵蓋：
- 任務遷移與負載均衡
- CPU 頻率調整（schedutil）
- PELT 負載追蹤
- RT/Deadline 排程決策
- Cgroup 附加
- 能量感知排程（EAS）

### 安全（5 restricted hooks）
- `avc.h`：AVC 快取操作（insert、delete、replace、lookup）
- `selinux.h`：SELinux 初始化狀態
- 全部為 restricted hook，確保安全關鍵路徑的獨佔控制

### 儲存（9 hooks）
- `ufshcd.h`：UFS 控制器操作（PRDT 填充、命令準備/發送/完成、UICommand、金鑰重程式設計）

### 記憶體管理（4 hooks）
- `mm.h`：mmap 檔案檢查、檔案開啟檢查、BPF 系統呼叫檢查
- `vmscan.h`：匿名/檔案頁面回收平衡

## vendor_hooks.c 匯出的符號

`drivers/android/vendor_hooks.c` 使用 `EXPORT_TRACEPOINT_SYMBOL_GPL` 匯出約 50 個 tracepoint 符號。這些是「bare tracehook」——沒有對應的 trace event，純粹供外部模組 probe。

關鍵匯出（按子系統）：

**CPU/電源管理：**
- `android_vh_cpu_idle_enter` / `android_vh_cpu_idle_exit`
- `android_vh_cpufreq_online`
- `android_vh_cpuidle_psci_enter` / `android_vh_cpuidle_psci_exit`
- `android_vh_allow_domain_state`

**Cgroup：**
- `android_rvh_cpu_cgroup_attach` / `android_rvh_cpu_cgroup_online`
- `android_vh_cgroup_attach`

**安全：**
- `android_rvh_selinux_avc_insert` / `android_rvh_selinux_avc_node_delete`
- `android_rvh_selinux_avc_node_replace` / `android_rvh_selinux_avc_lookup`
- `android_rvh_selinux_is_initialized`
- `android_vh_check_mmap_file` / `android_vh_check_file_open`
- `android_vh_check_bpf_syscall`

**儲存（UFS）：**
- `android_vh_ufs_fill_prdt` / `android_vh_ufs_prepare_command`
- `android_vh_ufs_send_command` / `android_vh_ufs_compl_command`
- `android_rvh_ufs_reprogram_all_keys`

**其他：**
- `android_vh_do_send_sig_info`（信號）
- `android_vh_ptype_head`（網路）
- `android_rvh_iommu_setup_dma_ops`（IOMMU）
- `android_vh_gic_set_affinity`（中斷）
- `android_rvh_hw_protection_shutdown`（重開機）
- `android_vh_rproc_recovery` / `android_vh_rproc_recovery_set`（遠端處理器）

## 排程器 Hook 未在此檔匯出的原因

注意：`trace/hooks/sched.h` 宣告的 78 個排程器 hooks 並**不**在 `vendor_hooks.c` 中匯出。排程器 hooks 透過排程器子系統自身的程式碼路徑（`kernel/sched/`）直接呼叫，且其 tracepoint 在各自的原始碼檔案中定義。`vendor_hooks.c` 只負責匯出那些定義在自身但沒有其他程式碼路徑會直接使用的「bare」tracepoint hooks。

## 交叉參考

- [Vendor Hooks 框架](../concepts/vendor-hooks.md) — 設計原理與機制
- [Vendor Hooks Driver](../entities/vendor-hooks-driver.md) — 匯出驅動實作細節
- [drivers/android/ 總覽](drivers-android-overview.md) — 目錄總覽
- [排程器](../subsystems/scheduler.md) — 78 個排程器 hooks 的完整分析
- [安全](../subsystems/security.md) — 安全相關 restricted hooks
