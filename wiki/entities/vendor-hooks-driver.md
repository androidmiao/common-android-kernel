---
type: entity
kernel_path: drivers/android/vendor_hooks.c
config_option: CONFIG_ANDROID_VENDOR_HOOKS
upstream: no
related:
  - ../concepts/vendor-hooks.md
  - ../concepts/gki.md
  - ../concepts/module-system.md
last_updated: 2026-04-09
---

# Vendor Hooks Driver — 廠商 Hook 匯出驅動

## Overview

`vendor_hooks.c` 是 Android Vendor Hook 框架的核心匯出模組，將所有 Android tracepoint hooks 透過 `EXPORT_TRACEPOINT_SYMBOL_GPL` 匯出為 GPL 符號，使廠商核心模組能夠註冊回調函式。它不實現任何邏輯，純粹作為符號匯出的中心點 `[android]`。

Hook 的宣告和定義分散在 `include/trace/hooks/` 目錄的 29 個標頭檔中（詳見 [Vendor Hooks 概念頁](../concepts/vendor-hooks.md)），而此驅動負責將它們統一匯出。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/android/vendor_hooks.c` | 93 | Tracepoint 符號匯出 |

## Implementation Details

### 結構

1. **標頭引入**（行 13-38）— 包含所有 `include/trace/hooks/*.h` 標頭檔
2. **符號匯出**（行 44-93）— 50 個 `EXPORT_TRACEPOINT_SYMBOL_GPL()` 呼叫

### 匯出的 Tracepoints（50 個）

按子系統分類：

| 子系統 | Hooks | 說明 |
|--------|-------|------|
| **信號** | `android_vh_do_send_sig_info` | 信號發送 |
| **CPU idle** | `android_vh_cpu_idle_enter`, `_exit`, `_psci_enter`, `_psci_exit` | CPU 閒置狀態 |
| **CPU 頻率** | `android_vh_cpufreq_online` | 頻率上線 |
| **記憶體** | `android_rvh_set_balance_anon_file_reclaim` | Anon/file 回收平衡 |
| **Cgroup** | `android_vh_cgroup_attach`, `android_rvh_cpu_cgroup_attach`, `_online` | Cgroup 操作 |
| **UFS** | `android_vh_ufs_fill_prdt`, `_reprogram_all_keys`, `_prepare_command` 等 | UFS 儲存 (6 hooks) |
| **Syscall** | `android_vh_syscall_prctl_finished`, `_check_bpf_syscall` | 系統呼叫攔截 |
| **網路** | `android_vh_ptype_head` | 封包類型 |
| **SELinux** | `android_rvh_selinux_avc_insert/delete/replace/lookup`, `_is_initialized` | AVC 快取 (5 hooks) |
| **安全** | `android_vh_check_mmap_file`, `_check_file_open` | 檔案存取檢查 |
| **電源** | `android_vh_allow_domain_state`, `_hw_protection_shutdown` | PM 域/關機 |
| **Epoch** | `android_vh_show_suspend/resume_epoch_val` | 休眠/喚醒時間戳 |
| **Printk** | `android_vh_printk_hotplug/caller_id/caller/ext_header` | 日誌自訂 (4 hooks) |
| **Crash** | `android_vh_sysrq_crash`, `_ipi_stop` | 崩潰處理 |
| **Workqueue** | `android_vh_wq_lockup_pool` | 工作佇列鎖死檢測 |
| **MPAM** | `android_vh_mpam_set` | ARM MPAM 設定 |
| **IOMMU** | `android_rvh_iommu_setup_dma_ops`, `_iovad_alloc/free_iova` | IOMMU 操作 (3 hooks) |
| **GIC** | `android_vh_gic_set_affinity` | 中斷親和性 |
| **Remoteproc** | `android_vh_rproc_recovery`, `_set` | 遠端處理器恢復 |
| **Timer** | `android_vh_timer_calc_index` | 計時器索引 |
| **FP/SIMD** | `android_vh_is_fpsimd_save` | 浮點/SIMD 儲存 |

所有匯出均為 `EXPORT_TRACEPOINT_SYMBOL_GPL`（GPL-only，防止專有模組使用）。

## Userspace Interface

無直接使用者空間介面。廠商核心模組透過 `register_trace_android_vh_*()` / `register_trace_android_rvh_*()` API 註冊回調。

## Android-Specific Notes

- 完全 Android 專屬 `[android]`，不存在於上游 Linux
- 此檔案僅匯出符號，實際 hook 邏輯在各子系統原始碼中的 `trace_android_*()` 呼叫點
- `android_vh_*` 為普通 hooks，`android_rvh_*` 為 restricted hooks（不可同時多個廠商註冊）
- 此處列出 50 個 hooks，加上排程子系統的 78 個（透過 `sched/` 目錄自行匯出）和其他子系統直接匯出的，全核心共約 130 個 vendor hooks

## Cross-References

- [Vendor Hooks 概念](../concepts/vendor-hooks.md) — 框架設計、DECLARE_HOOK 機制、完整統計
- [GKI 架構](../concepts/gki.md) — Vendor hooks 在 GKI 模組化中的角色
- [Module 系統](../concepts/module-system.md) — EXPORT_TRACEPOINT_SYMBOL_GPL 機制
