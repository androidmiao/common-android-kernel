---
type: source
source_path: drivers/android/binder/page_range.rs
lines: 731
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder_alloc-c.md
  - ./src-drivers-android-binder-allocation-rs.md
---

# Source: `drivers/android/binder/page_range.rs`

## Summary

Rust Binder 的頁面範圍管理模組，處理 VMA shrinker 整合的頁面生命週期。對應 C 版本 `binder_alloc.c` 的底層頁面分配/回收邏輯。管理 shrinker 註冊和 LRU 列表，在記憶體壓力時回收未使用的 binder 頁面。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct Shrinker` | 42-45 | 核心 shrinker 的 Rust 包裝（opaque 指標 + list_lru） |
| `Shrinker::new()` | 59 | 建立 shrinker（const unsafe fn） |
| `struct ShrinkablePageRange` | — | 可回收的頁面範圍 |

## Notable Implementation Details

- **鎖定順序**：mmap lock → spinlock → lru spinlock（嚴格順序）；shrinker 因反向鎖定使用 trylock。
- **Opaque FFI**：`kernel::types::Opaque` 包裝 C 的 `struct shrinker` 指標和 `struct list_lru`。
- **Send + Sync**：手動實現 `Send` 和 `Sync` trait（unsafe impl），因 shrinker 和 list_lru 是執行緒安全的。
- **頁面管理**：使用 `kernel::page::{Page, PAGE_SHIFT, PAGE_SIZE}` 進行頁面級操作。
- **MM 整合**：`kernel::mm::{virt, Mm, MmWithUser}` 處理虛擬記憶體映射。
- **使用者空間資料讀取**：`kernel::uaccess::UserSliceReader` 進行安全的 user-kernel 資料傳輸。
- **page_range_helper.c**：C 輔助函式（85 行）提供直接操作 `struct list_lru` 等無法在 Rust 中安全表達的操作。

## Open Questions

- Shrinker 回呼中 trylock 失敗的頻率？是否影響記憶體回收效率？
