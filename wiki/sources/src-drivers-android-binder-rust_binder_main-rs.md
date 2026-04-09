---
type: source
source_path: drivers/android/binder/rust_binder_main.rs
lines: 611
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-c.md
  - ./src-drivers-android-binder-process-rs.md
  - ./src-drivers-android-binder-thread-rs.md
---

# Source: `drivers/android/binder/rust_binder_main.rs`

## Summary

Rust Binder 驅動的頂層模組入口。定義模組結構、初始化流程、file_operations 對接、debugfs/binderfs 整合，以及全域狀態管理。是 C `binder.c` 的 Rust 等效實現。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| 模組宣告 | 1-49 | `#![recursion_limit = "256"]`，引入所有子模組 |
| `mod binderfs` | 51-80+ | FFI 綁定：`init_rust_binderfs()`、`rust_binderfs_create_proc_file()`、`rust_binderfs_remove_file()` |
| 子模組聲明 | 36-48 | allocation、context、deferred_close、defs、error、node、page_range、process、range_alloc、stats、thread、trace、transaction |

## Notable Implementation Details

- **13 個子模組**：完整對映 C 版本的所有功能模組。
- **FFI 橋接**：透過 `extern "C"` 區塊呼叫 C 側的 binderfs 函式（`rust_binderfs.c`）。
- **kernel crate 依賴**：使用 `kernel::fs::File`、`kernel::sync::Arc`、`kernel::list::ListArc`、`kernel::seq_file::SeqFile`、`kernel::sync::poll::PollTable`、`kernel::task::Pid` 等核心抽象。
- **全域原子計數器**：`AtomicBool`、`AtomicUsize` 用於全域狀態和 debug ID 生成。
- **類型別名**：`DArc`、`DLArc`、`DTRWrap`、`DeliverToRead` 等簡化泛型使用。
- **Clippy 配置**：允許 `as_underscore`、`ref_as_ptr`、`ptr_as_ptr`、`cast_lossless`。
- **CONFIG 互斥**：`ANDROID_BINDER_IPC_RUST` 依賴 `RUST && MMU && !ANDROID_BINDER_IPC`，與 C 版本互斥。

## Open Questions

- Rust 版本目前是否為 GKI 的預設選擇？還是仍為實驗性？
- 效能與 C 版本的比較資料？
