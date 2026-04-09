---
type: source
source_path: drivers/android/binder_alloc.h
lines: 189
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ./src-drivers-android-binder_alloc-c.md
  - ./src-drivers-android-binder_internal-h.md
---

# Source: `drivers/android/binder_alloc.h`

## Summary

Binder 緩衝區分配器的標頭檔，定義 `struct binder_buffer`、`struct binder_alloc`、`struct binder_shrinker_mdata` 以及分配器 API 介面。

## Key Functions

| Declaration | Line | Purpose |
|-------------|------|---------|
| `binder_alloc_new_buf()` | 127-131 | 分配新緩衝區 |
| `binder_alloc_free_buf()` | 139-140 | 釋放緩衝區 |
| `binder_alloc_mmap_handler()` | 141-142 | mmap 處理 |
| `binder_alloc_init()` | 132 | 初始化分配器 |
| `binder_alloc_shrinker_init()` | 133 | Shrinker 初始化 |
| `binder_alloc_shrinker_exit()` | 134 | Shrinker 清理 |
| `binder_alloc_prepare_to_free()` | 136-138 | 準備釋放 |
| `binder_alloc_deferred_release()` | 143 | 延遲釋放 |
| `binder_alloc_copy_user_to_buffer()` | 163-168 | 使用者→緩衝區複製 |
| `binder_alloc_copy_to_buffer()` | 170-174 | 核心→緩衝區複製 |
| `binder_alloc_copy_from_buffer()` | 176-180 | 緩衝區→核心複製 |
| `binder_alloc_get_free_async_space()` | 156-161 | 取得非同步可用空間（inline） |

## Notable Implementation Details

- **`struct binder_buffer`** (:41)：交易緩衝區中繼資料，含 5 個位元欄位（`free`、`clear_on_free`、`allow_user_free`、`async_transaction`、`oneway_spam_suspect`）和 27-bit `debug_id`。
- **`struct binder_alloc`** (:107)：每行程分配器狀態，核心欄位：
  - `mutex`：保護分配器所有欄位
  - `mm`：行程的 mm_struct 指標
  - `vm_start` + `buffer_size`：VA 空間範圍
  - `free_buffers`/`allocated_buffers`：紅黑樹
  - `free_async_space`：非同步空間限額
  - `pages`/`pages_high`：頁面陣列與高水位
  - `freelist`：指向全域 LRU 列表
  - `mapped`：僅允許一次 mmap
- **`struct binder_shrinker_mdata`** (:66)：LRU 回收中繼資料（`lru` list_head、`alloc` 指標、`page_index`）。
- **`page_to_lru()` inline** (:72)：從 `struct page` 的 private 欄位取得 LRU 連結。
- **KUnit 條件匯出**：`__binder_alloc_init()` 和 `binder_alloc_buffer_size()` 僅在 `CONFIG_KUNIT` 啟用時匯出。

## Open Questions

- 分配器的鎖定粒度是否足夠（目前使用單一 mutex）？
