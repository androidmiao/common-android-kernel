---
type: concept
scope: kernel-wide
related:
  - ../concepts/vendor-hooks.md
  - ../concepts/bpf.md
last_updated: 2026-04-08
---

# 追蹤與 ftrace

## 概述

Linux 核心追蹤子系統提供運行時可觀察性——追蹤函數呼叫、事件、延遲和效能瓶頸。核心元件包括 **ftrace**（函數追蹤器）、**tracepoints**（靜態追蹤點）、**kprobes**（動態探針）和 **ring buffer**（環形緩衝區）。Android 的 Vendor Hook 框架正是建立在 tracepoint 基礎設施之上。

核心實現位於 `kernel/trace/` 目錄（80+ 個檔案），其中 `trace.c`（297297 字節）、`ftrace.c`（223916 字節）和 `ring_buffer.c`（212754 字節）是最大的三個檔案。

## 機制

### Tracepoint 基礎設施

**核心結構**（`include/linux/tracepoint-defs.h:39-48`）：

```c
struct tracepoint {
    const char *name;                      // Tracepoint 名稱
    struct static_key_false key;           // 靜態分支（禁用時零開銷）
    struct static_call_key *static_call_key;
    void *static_call_tramp;               // 靜態呼叫蹦床
    void *iterator;                        // TP 迭代器
    void *probestub;                       // Probe 存根
    struct tracepoint_func __rcu *funcs;   // 已註冊探針列表
    struct tracepoint_ext *ext;            // 擴展（reg/unreg 函數）
};
```

**Tracepoint 函數**（`tracepoint-defs.h:26-30`）：

```c
struct tracepoint_func {
    void *func;    // 探針函數
    void *data;    // 私有資料
    int prio;      // 優先級（預設 TRACEPOINT_DEFAULT_PRIO=10）
};
```

**API**（`include/linux/tracepoint.h:36-55`）：

- `tracepoint_probe_register()` / `tracepoint_probe_register_prio()` — 註冊探針
- `tracepoint_probe_unregister()` — 註銷探針
- `for_each_kernel_tracepoint()` — 列舉所有 TP

**同步保證**（`tracepoint.h:114-131`）：

註銷時使用 `synchronize_rcu_tasks_trace()` + `synchronize_rcu()` 確保沒有探針正在執行中。

### Trace Events 系統

**實現**（`kernel/trace/trace_events.c`）

Trace events 是 tracepoints 的高級封裝，提供結構化的事件格式。

**TRACE_EVENT 巨集**（`include/trace/trace_events.h:39-46`）將事件定義展開為：

1. **結構定義**（第61-68行）：

```c
struct trace_event_raw_<name> {
    struct trace_entry ent;     // 通用頭部（type, flags, preempt_count, pid）
    <type> <field>;             // 使用者定義欄位
    char __data[];              // 動態資料
};
```

2. **資料偏移**（第128-131行）：動態欄位偏移結構

**事件欄位 API**（`trace_events.c:115-190`）：
- `trace_define_field()` — 註冊事件欄位
- `trace_find_event_field()` — 查找欄位

### ftrace 函數追蹤器

**實現**（`kernel/trace/ftrace.c`）

ftrace 使用 `gcc -pg` 編譯的函數呼叫前綴，在運行時動態修補 NOP 指令為追蹤呼叫。

**全局狀態**（`ftrace.c:95-126`）：

- `ftrace_enabled`（95行）— 全局開關
- `function_trace_op`（99行）— 當前追蹤操作
- `ftrace_ops_list`（125行）— 已註冊的 ftrace 操作列表
- `ftrace_trace_function`（126行）— 當前追蹤函數指標

**動態追蹤**（`CONFIG_DYNAMIC_FTRACE`，第74行）：

- 啟動時所有追蹤點為 NOP
- 啟用特定函數追蹤時，動態修補 NOP → 追蹤呼叫
- Hash 表管理追蹤的函數：`FTRACE_HASH_DEFAULT_BITS=10`（71行）

### Ring Buffer

**實現**（`kernel/trace/ring_buffer.c`）

Per-CPU 環形緩衝區，存儲所有追蹤事件。

**架構**（`ring_buffer.c:102-151`）：

```
讀者頁面 <-> 環形緩衝區
     ↓
  +---+   +---+   +---+
  |   |-->|   |-->|   |
  +---+   +---+   +---+
    ^               |
    +---------------+
```

**元數據**（`ring_buffer.c:50-64`）：

```c
struct ring_buffer_cpu_meta {
    unsigned long first_buffer;
    unsigned long head_buffer;
    unsigned long commit_buffer;
    __u32 subbuf_size;        // 子緩衝區大小
    __u32 nr_subbufs;
};
```

**事件頭部格式**（第69-85行）：
- `type_len`：5位
- `time_delta`：27位
- 特殊類型：`RINGBUF_TYPE_PADDING`、`TIME_EXTEND`、`TIME_STAMP`

### Kprobes

**實現**（`kernel/kprobes.c`）

動態探針，可在任意核心函數的任意位置插入追蹤程式碼。

**Hash 表**（`kprobes.c:49-62`）：
- `KPROBE_TABLE_SIZE = 64`
- 受 `kprobe_mutex` 保護

**指令緩存**（`kprobes.c:90-134`）：

```c
struct kprobe_insn_page {
    kprobe_opcode_t *insns;              // 指令槽頁面
    struct kprobe_insn_cache *cache;
    int nused;                           // 已使用槽數
    char slot_used[];                    // 3態：CLEAN/DIRTY/USED
};
```

使用 `execmem_alloc(EXECMEM_KPROBES, PAGE_SIZE)` 分配可執行記憶體（`kprobes.c:145-195`）。

### 與 Vendor Hooks 的關係

Android Vendor Hook 框架直接繼承 tracepoint 基礎設施：

```
Tracepoint infrastructure （struct tracepoint）
  ↓
Trace Events （TRACE_EVENT 巨集）
  ↓
Vendor Hooks （DECLARE_HOOK = DECLARE_TRACE_EVENT）
```

啟用條件（`vendor_hooks.h:14`）：

```c
#if defined(CONFIG_TRACEPOINTS) && defined(CONFIG_ANDROID_VENDOR_HOOKS)
```

Vendor Hook 使用與 tracepoint 相同的：
- `static_key_false` 實現零開銷禁用
- `static_call` 實現直接呼叫
- RCU 同步保護探針註冊/註銷

## 使用模式

### 追蹤工具鏈

| 工具 | 介面 | 用途 |
|------|------|------|
| `trace-cmd` | tracefs | 函數追蹤、事件追蹤 |
| `perf` | perf_event | 效能分析、採樣 |
| `bpftrace` | BPF | 動態追蹤腳本 |
| `atrace` / `systrace` | tracefs | Android 系統追蹤 |

### tracefs 介面

透過 `/sys/kernel/tracing/`（或 `/sys/kernel/debug/tracing/`）：

- `available_tracers` — 可用的追蹤器列表
- `current_tracer` — 當前啟用的追蹤器
- `trace` — 追蹤輸出
- `events/` — 事件目錄（按子系統分類）
- `set_ftrace_filter` — 函數過濾

## Android 相關性

### Atrace / Systrace 整合

Android 的 atrace 透過 tracefs 介面控制核心追蹤：

1. 啟用特定事件類別（sched、freq、idle 等）
2. 核心事件寫入 ring buffer
3. atrace 讀取 ring buffer 生成 trace 檔案
4. Perfetto 或 systrace 進行可視化分析

### 關鍵配置

GKI defconfig 啟用的追蹤相關配置：

- `CONFIG_TRACEPOINTS` — tracepoint 基礎設施
- `CONFIG_FTRACE` — ftrace 函數追蹤
- `CONFIG_ANDROID_VENDOR_HOOKS` — Vendor Hook 框架
- BPF 追蹤整合：`CONFIG_BPF_EVENTS`

## 交叉參考

- [Vendor Hooks](vendor-hooks.md) — 基於 tracepoint 的 Android 擴展機制
- [BPF](bpf.md) — BPF 追蹤程式（kprobe、tracepoint 類型）
