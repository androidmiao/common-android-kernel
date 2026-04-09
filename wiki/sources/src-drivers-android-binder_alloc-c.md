---
type: source
source_path: drivers/android/binder_alloc.c
lines: 1410
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../data-structures/binder_proc.md
  - ./src-drivers-android-binder_alloc-h.md
  - ./src-drivers-android-binder-c.md
---

# Source: `drivers/android/binder_alloc.c`

## Summary

Binder 交易緩衝區分配器的 C 實現。管理每個 binder 行程的虛擬位址空間，負責分配、釋放和回收交易緩衝區。使用紅黑樹管理空閒與已分配緩衝區，並整合 LRU shrinker 機制在記憶體壓力時回收未使用的頁面。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| `binder_alloc_new_buf()` | ~127-131（宣告於 .h） | 分配新的交易緩衝區 |
| `binder_alloc_free_buf()` | ~139-140（宣告於 .h） | 釋放交易緩衝區 |
| `binder_alloc_mmap_handler()` | ~141-142（宣告於 .h） | 處理 mmap 初始化 |
| `binder_alloc_buffer_size()` | 61-67 | 計算緩衝區大小（KUNIT 可見） |
| `binder_insert_free_buffer()` | 70 | 將緩衝區插入空閒紅黑樹 |
| `binder_alloc_prepare_to_free()` | ~136-138（宣告於 .h） | 準備釋放指定緩衝區 |
| `binder_alloc_deferred_release()` | ~143（宣告於 .h） | 延遲釋放所有分配 |
| `binder_alloc_free_page()` | ~124-126（宣告於 .h） | LRU shrinker 回呼 |
| `binder_alloc_copy_user_to_buffer()` | ~163-168（宣告於 .h） | 使用者空間到緩衝區的資料複製 |
| `binder_alloc_copy_to_buffer()` | ~170-174（宣告於 .h） | 核心到緩衝區的資料複製 |
| `binder_alloc_copy_from_buffer()` | ~176-180（宣告於 .h） | 緩衝區到核心的資料複製 |
| `binder_alloc_init()` | ~132（宣告於 .h） | 初始化分配器 |
| `binder_alloc_shrinker_init()` | ~133（宣告於 .h） | 全域 shrinker 初始化 |

## Notable Implementation Details

- **雙紅黑樹管理**：`free_buffers`（按大小排序）和 `allocated_buffers`（按位址排序）。
- **LRU 頁面回收**：全域 `binder_freelist`（`list_lru`）追蹤可回收頁面；`binder_shrinker_mdata` 結構儲存回收所需的中繼資料。
- **非同步空間限制**：`free_async_space` 限制為總 VA 空間的 1/2，防止 oneway 交易耗盡緩衝區。
- **Oneway spam 偵測**：`oneway_spam_detected` 標誌在非同步緩衝區用量超過閾值時觸發。
- **KUnit 整合**：`VISIBLE_IF_KUNIT` 巨集匯出內部函式供單元測試使用。
- **偵錯遮罩**：`binder_alloc_debug_mask` 模組參數控制分配器偵錯輸出。
- **Mmap 限制**：`binder_alloc_mmap_lock` 互斥鎖確保每個行程只能 mmap 一次；`FORBIDDEN_MMAP_FLAGS` 禁止 `VM_WRITE`。

## Open Questions

- Rust 版本的 `range_alloc` 是否已完全對等替換此分配器？
- 頁面回收的效能影響如何量化？
