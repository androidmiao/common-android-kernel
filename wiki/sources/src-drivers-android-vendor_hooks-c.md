---
type: source
source_path: drivers/android/vendor_hooks.c
lines: 93
ingested: 2026-04-09
related:
  - ../entities/vendor-hooks-driver.md
  - ../concepts/vendor-hooks.md
  - ../concepts/tracing-and-ftrace.md
---

# Source: `drivers/android/vendor_hooks.c`

## Summary

Android Vendor Hook 匯出驅動，負責建立 tracepoint 並將其匯出為 GPL 符號，供廠商模組掛載。本檔案是 GKI 廠商擴展框架的核心匯出點。透過 `#define CREATE_TRACE_POINTS` 搭配各 hook 標頭檔的 include，一次性建立所有 tracepoint 定義。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| （無傳統函式） | — | 本檔案無函式定義 |
| `CREATE_TRACE_POINTS` | 9 | 巨集定義，啟用 tracepoint 建立 |
| `EXPORT_TRACEPOINT_SYMBOL_GPL()` × 50 | 44-93 | 匯出 50 個 tracepoint 符號供模組使用 |

## Notable Implementation Details

- **Include 的 Hook 標頭檔**（27 個）：`cpuidle.h`、`mpam.h`、`wqlockup.h`、`debug.h`、`sysrqcrash.h`、`printk.h`、`epoch.h`、`cpufreq.h`、`ufshcd.h`、`cgroup.h`、`sys.h`、`iommu.h`、`net.h`、`pm_domain.h`、`cpuidle_psci.h`、`vmscan.h`、`avc.h`、`selinux.h`、`syscall_check.h`、`gic.h`、`gic_v3.h`、`remoteproc.h`、`reboot.h`、`timer.h`、`fpsimd.h`、`signal.h`、`vendor_hooks.h`
- **匯出符號按子系統分類**：
  - 信號：`android_vh_do_send_sig_info`
  - CPU 閒置：`android_vh_cpu_idle_enter/exit`
  - MPAM：`android_vh_mpam_set`
  - Workqueue：`android_vh_wq_lockup_pool`
  - 偵錯/SysRq：`android_vh_ipi_stop`、`android_vh_sysrq_crash`
  - Printk：`android_vh_printk_hotplug/caller_id/caller/ext_header`
  - 電源管理：`android_vh_show_suspend/resume_epoch_val`、`android_vh_allow_domain_state`、`android_vh_cpuidle_psci_enter/exit`
  - CPU 頻率：`android_vh_cpufreq_online`、`android_rvh_show_max_freq`
  - Cgroup：`android_rvh_cpu_cgroup_attach/online`、`android_vh_cgroup_attach`
  - UFS：`android_vh_ufs_fill_prdt/prepare_command/update_sysfs/send_command/compl_command/send_uic_command/send_tm_command/check_int_errors`、`android_rvh_ufs_reprogram_all_keys`
  - Syscall：`android_vh_syscall_prctl_finished`
  - IOMMU：`android_rvh_iommu_setup_dma_ops`、`android_vh_iommu_iovad_alloc/free_iova`
  - 網路：`android_vh_ptype_head`
  - 記憶體回收：`android_rvh_set_balance_anon_file_reclaim`
  - SELinux/AVC：`android_rvh_selinux_avc_insert/node_delete/node_replace/lookup`、`android_rvh_selinux_is_initialized`
  - 安全：`android_vh_check_mmap_file`、`android_vh_check_file_open`、`android_vh_check_bpf_syscall`
  - GIC：`android_vh_gic_set_affinity`
  - 遠端處理器：`android_vh_rproc_recovery/recovery_set`
  - 定時器：`android_vh_timer_calc_index`
  - FPSIMD：`android_vh_is_fpsimd_save`
  - 關機：`android_rvh_hw_protection_shutdown`
- **Restricted hooks**（`android_rvh_*` 前綴）：14 個，使用 `DECLARE_RESTRICTED_HOOK`，只允許一個回呼。
- **Regular hooks**（`android_vh_*` 前綴）：36 個，使用 `DECLARE_HOOK`，允許多個回呼。

## Open Questions

- 排程子系統的 78 個 vendor hooks 是否在其他位置匯出（如 `kernel/sched/` 目錄自行匯出）？
- 為何部分 hook 標頭檔（如 `sched.h`）未在此檔案 include？
