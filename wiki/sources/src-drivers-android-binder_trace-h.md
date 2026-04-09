---
type: source
source_path: drivers/android/binder_trace.h
lines: 472
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/tracing-and-ftrace.md
  - ./src-drivers-android-binder-c.md
---

# Source: `drivers/android/binder_trace.h`

## Summary

Binder 驅動的 trace event 定義標頭檔。使用 `TRACE_EVENT`/`DEFINE_EVENT`/`DECLARE_EVENT_CLASS` 巨集定義約 20 個 tracepoint，供 ftrace 追蹤 binder 操作。

## Key Functions

| Tracepoint | Line | Purpose |
|-----------|------|---------|
| `binder_ioctl` | 22 | 追蹤 ioctl 呼叫（cmd + arg） |
| `binder_ioctl_done` | — | ioctl 完成 |
| `binder_write_done` | — | thread_write 完成 |
| `binder_read_done` | — | thread_read 完成 |
| `binder_lock` / `binder_locked` / `binder_unlock` | — | 鎖定追蹤 |
| `binder_transaction` | — | 交易開始 |
| `binder_transaction_received` | — | 交易接收 |
| `binder_command` | — | BC_* 命令 |
| `binder_return` | — | BR_* 回覆 |
| `binder_alloc_buf` | — | 緩衝區分配 |
| `binder_free_buf` | — | 緩衝區釋放 |
| `binder_alloc_page_*` | — | 頁面分配/釋放 |
| `binder_update_page_range` | — | 頁面範圍更新 |

## Notable Implementation Details

- **TRACE_SYSTEM**：定義為 `binder`，所有 tracepoint 歸入 `binder` 系統。
- **Event Class 模式**：`binder_function_return_class` 基礎類別被多個完成事件共用。
- **前向宣告**：引用但不 include `binder_buffer`、`binder_node`、`binder_proc` 等結構。
- **Multi-read 防護**：標準 `TRACE_HEADER_MULTI_READ` 保護。
