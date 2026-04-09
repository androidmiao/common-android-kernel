---
type: source
source_path: drivers/android/binder/stats.rs
lines: 89
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-defs-rs.md
---

# Source: `drivers/android/binder/stats.rs`

## Summary

Rust Binder 的統計追蹤模組。全域 `GLOBAL_STATS` 使用原子計數器追蹤每種 BC_*/BR_* 命令的執行次數，供 binderfs 的 stats 檔案顯示。

## Key Functions

| Function | Line | Purpose |
|----------|------|---------|
| `BinderStats::new()` | 22 | 建立零初始化的統計結構（const fn） |
| `BinderStats::inc_bc()` | 32 | 原子增加 BC 命令計數 |
| `BinderStats::inc_br()` | 39 | 原子增加 BR 回覆計數 |
| `BinderStats::debug_print()` | 46 | 格式化輸出到 SeqFile |
| `command_string()` | 71 | 從 C 字串表取得命令名稱 |
| `return_string()` | 80 | 從 C 字串表取得回覆名稱 |

## Notable Implementation Details

- **純原子操作**：使用 `AtomicU32` 陣列和 `Relaxed` ordering，無需鎖。
- **外部 C 字串表**：透過 `extern "C"` 引用 C 側定義的 `binder_command_strings` 和 `binder_return_strings` 陣列。
- **ioctl 號碼索引**：使用 `kernel::ioctl::_IOC_NR()` 從 ioctl 命令號提取序號作為陣列索引。
- **邊界檢查**：`inc_bc()`/`inc_br()` 使用 `get()` 進行陣列邊界檢查，越界時靜默忽略。
- **全域靜態**：`GLOBAL_STATS` 為全域靜態變數，無需額外同步。
