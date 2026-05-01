---
type: source
source_path: samples/Kconfig
lines: 334
ingested: 2026-04-09
---

# Source: `samples/Kconfig`

## Summary

頂層 Kconfig 選單，控制 `samples/` 目錄下所有範例程式碼的編譯。以 `menuconfig SAMPLES` 為根節點，包含 ~35 個獨立配置選項，涵蓋核心模組範例、使用者空間工具、追蹤基礎設施範例、安全機制範例、以及 Rust 核心驅動範例。另外透過 `source` 引入 `samples/rust/Kconfig`（15 個 Rust 範例）和 `samples/damon/Kconfig`（3 個 DAMON 範例）。

## Key Configuration Options

| 配置選項 | 類型 | 依賴 | 用途 |
|---------|------|------|------|
| `SAMPLES` | menuconfig | — | 總開關，啟用範例程式碼編譯 |
| `SAMPLE_TRACE_EVENTS` | tristate | `EVENT_TRACING && m` | Trace event 範例模組 |
| `SAMPLE_TRACE_CUSTOM_EVENTS` | tristate | `EVENT_TRACING && m` | 自定義 trace event 範例 |
| `SAMPLE_TRACE_PRINTK` | tristate | `EVENT_TRACING && m` | trace_printk 格式測試 |
| `SAMPLE_FTRACE_DIRECT` | tristate | `DYNAMIC_FTRACE_WITH_DIRECT_CALLS && m` | register_ftrace_direct() 範例 |
| `SAMPLE_FTRACE_DIRECT_MULTI` | tristate | `DYNAMIC_FTRACE_WITH_DIRECT_CALLS && m` | 多 IP ftrace direct 範例 |
| `SAMPLE_FTRACE_OPS` | tristate | `FUNCTION_TRACER` | 自定義 ftrace ops 計時範例 |
| `SAMPLE_TRACE_ARRAY` | tristate | `EVENT_TRACING && m` | Ftrace instance 核心存取 |
| `SAMPLE_KOBJECT` | tristate | — | kobject/kset/ktype 使用範例 |
| `SAMPLE_KPROBES` | tristate | `KPROBES && m` | kprobe 範例模組 |
| `SAMPLE_KRETPROBES` | tristate | `SAMPLE_KPROBES && KRETPROBES` | kretprobe 範例 |
| `SAMPLE_HW_BREAKPOINT` | tristate | `HAVE_HW_BREAKPOINT && m` | 硬體斷點範例 |
| `SAMPLE_FPROBE` | tristate | `FPROBE && m` | fprobe 範例（支援 symbol 參數）|
| `SAMPLE_KFIFO` | tristate | `m` | kfifo API 使用範例 |
| `SAMPLE_KDB` | tristate | `KGDB_KDB && m` | kdb 自定義命令範例 |
| `SAMPLE_QMI_CLIENT` | tristate | `ARCH_QCOM && NET && m` | QMI/QRTR 通訊範例 |
| `SAMPLE_RPMSG_CLIENT` | tristate | `RPMSG && m` | rpmsg AMP 通訊範例 |
| `SAMPLE_LIVEPATCH` | tristate | `LIVEPATCH && m` | 即時修補範例 |
| `SAMPLE_CONFIGFS` | tristate | `CONFIGFS_FS && m` | configfs 介面範例 |
| `SAMPLE_CONNECTOR` | tristate | `CONNECTOR && HEADERS_INSTALL && m` | connector 核心/使用者空間通訊 |
| `SAMPLE_FANOTIFY_ERROR` | bool | `FANOTIFY && CC_CAN_LINK` | fanotify 檔案系統錯誤監控 |
| `SAMPLE_SECCOMP` | bool | `SECCOMP_FILTER && CC_CAN_LINK` | seccomp BPF 過濾器範例 |
| `SAMPLE_LANDLOCK` | bool | `CC_CAN_LINK && HEADERS_INSTALL` | Landlock 沙箱管理器範例 |
| `SAMPLE_ANDROID_BINDERFS` | bool | `CC_CAN_LINK && HEADERS_INSTALL` | Android binderfs 使用範例 |
| `SAMPLE_VFS` | bool | `CC_CAN_LINK && HEADERS_INSTALL` | 新 VFS 系統呼叫範例 (mount API, statx) |
| `SAMPLE_CORESIGHT_SYSCFG` | tristate | `CORESIGHT && m` | CoreSight 配置載入範例 |
| `SAMPLE_KMEMLEAK` | tristate | `DEBUG_KMEMLEAK && m` | 記憶體洩漏檢測器測試 |
| `SAMPLE_CGROUP` | bool | `CGROUPS && CC_CAN_LINK` | cgroup API 使用範例 |
| `SAMPLE_CHECK_EXEC` | bool | `CC_CAN_LINK && HEADERS_INSTALL` | SECBIT_EXEC 安全位元範例 |
| `SAMPLE_HUNG_TASK` | tristate | `DETECT_HUNG_TASK && DEBUG_FS` | hung task 偵測觸發範例 |
| `SAMPLE_VFIO_MDEV_MTTY` | tristate | `VFIO` | VFIO 虛擬 tty mdev 範例 |
| `SAMPLE_VFIO_MDEV_MDPY` | tristate | `VFIO` | VFIO 虛擬顯示 mdev 範例 |
| `SAMPLE_VFIO_MDEV_MBOCHS` | tristate | `VFIO` | VFIO Bochs 虛擬顯示 mdev 範例 |
| `SAMPLE_DAMON_WSSE` | bool | `DAMON && DAMON_VADDR` | DAMON 工作集大小估算 |
| `SAMPLE_DAMON_PRCL` | bool | `DAMON && DAMON_VADDR` | DAMON 主動記憶體回收 |
| `SAMPLE_DAMON_MTIER` | bool | `DAMON && DAMON_PADDR` | DAMON 記憶體階層化 |
| `SAMPLE_TSM_MR` | tristate | — (select TSM_MEASUREMENTS) | TSM 測量暫存器模擬 |
| `SAMPLES_RUST` | menuconfig | `RUST` | Rust 範例總開關 |

## Notable Implementation Details

- **兩類編譯目標：** `obj-$(CONFIG_*)` 用於核心模組（.ko），`subdir-$(CONFIG_*)` 用於使用者空間程式
- **Android 唯一範例：** `SAMPLE_ANDROID_BINDERFS` 是唯一直接標記為 Android 的範例
- **架構限制：** `SAMPLE_QMI_CLIENT` 僅限 `ARCH_QCOM`；`SAMPLE_FTRACE_DIRECT` 需要 `HAVE_SAMPLE_FTRACE_DIRECT`（x86/arm64）
- **子 Kconfig 引入：** `samples/rust/Kconfig` 和 `samples/damon/Kconfig` 透過 `source` 引入

## Open Questions

- GKI defconfig 是否啟用任何 SAMPLE_* 選項？（通常不會，這些僅供開發/測試用）
- Rust 範例是否在 Android CI 中定期編譯測試？

## Cross-References

- [Samples 子系統](../subsystems/samples.md)
- [追蹤與 ftrace](../concepts/tracing-and-ftrace.md)
- [BPF](../concepts/bpf.md)
- [Rust in Kernel](../concepts/rust-in-kernel.md)
- [Binderfs](../entities/binderfs.md)
- [Vendor Hooks](../concepts/vendor-hooks.md)
