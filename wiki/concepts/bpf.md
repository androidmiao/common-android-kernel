---
type: concept
scope: kernel-wide
related:
  - ../concepts/tracing-and-ftrace.md
  - ../concepts/vendor-hooks.md
last_updated: 2026-04-08
---

# BPF 子系統

## 概述

BPF（Berkeley Packet Filter）已從最初的封包過濾器演化為通用的核心內可程式化框架。它允許在核心中安全執行使用者定義的程式，用於網路過濾、追蹤、安全策略、排程（sched_ext）等。BPF 程式在載入時經過驗證器（verifier）的嚴格檢查，確保不會導致核心崩潰或安全漏洞。

核心實現位於 `kernel/bpf/`（71 個檔案），其中驗證器 `verifier.c` 是最大的單一檔案（25403行，769519 字節）。

## 機制

### BPF 程式類型

**定義**（`include/uapi/linux/bpf.h:1041-1075`）：

共 33 種程式類型，按用途分類：

**網路**：
- `BPF_PROG_TYPE_SOCKET_FILTER` — Socket 級過濾
- `BPF_PROG_TYPE_SCHED_CLS` / `SCHED_ACT` — 流量分類/動作
- `BPF_PROG_TYPE_XDP` — eXpress Data Path（高性能網路）
- `BPF_PROG_TYPE_SK_*` — Socket 操作
- `BPF_PROG_TYPE_LWT_*` — 輕量級隧道
- `BPF_PROG_TYPE_FLOW_DISSECTOR` — 流分析
- `BPF_PROG_TYPE_NETFILTER` — Netfilter 整合

**追蹤與監控**：
- `BPF_PROG_TYPE_KPROBE` — 核心探針
- `BPF_PROG_TYPE_TRACEPOINT` — 追蹤點
- `BPF_PROG_TYPE_RAW_TRACEPOINT` — 原始追蹤點
- `BPF_PROG_TYPE_PERF_EVENT` — 效能事件
- `BPF_PROG_TYPE_TRACING` — 通用追蹤

**安全與控制**：
- `BPF_PROG_TYPE_LSM` — Linux 安全模組
- `BPF_PROG_TYPE_CGROUP_*` — Cgroup 控制（SKB、SOCK、SOCKOPT、SYSCTL）

**排程**：
- `BPF_PROG_TYPE_STRUCT_OPS` — 結構體操作（用於 sched_ext）

### BPF 映射類型

**定義**（`include/uapi/linux/bpf.h:980-1031`）：

30 種映射類型，BPF 程式間和程式與使用者空間間共享資料的主要方式：

| 分類 | 類型 |
|------|------|
| **基本** | `HASH`、`ARRAY`、`PERCPU_HASH`、`PERCPU_ARRAY` |
| **LRU** | `LRU_HASH`、`LRU_PERCPU_HASH` |
| **特殊結構** | `LPM_TRIE`（最長前綴匹配）、`BLOOM_FILTER` |
| **嵌套** | `ARRAY_OF_MAPS`、`HASH_OF_MAPS` |
| **網路** | `DEVMAP`、`SOCKMAP`、`XSKMAP` |
| **隊列** | `QUEUE`、`STACK`、`RINGBUF`、`USER_RINGBUF` |
| **存儲** | `SK_STORAGE`、`TASK_STORAGE`、`CGRP_STORAGE` |
| **高級** | `STRUCT_OPS`、`ARENA` |

### 核心資料結構

**bpf_map**（`include/linux/bpf.h:295-338`）：

```c
struct bpf_map {
    const struct bpf_map_ops *ops;
    enum bpf_map_type map_type;
    u32 key_size, value_size, max_entries;
    struct btf *btf;                     // 類型資訊
    atomic64_t refcnt, usercnt, writecnt;
    bool frozen;                         // 寫一次保護
};
```

**BTF 欄位類型**（`bpf.h:194-212`）：

BPF 支持在映射值中嵌入核心物件：`BPF_SPIN_LOCK`、`BPF_TIMER`、`BPF_KPTR_*`（核心指標）、`BPF_LIST_HEAD`/`NODE`、`BPF_RB_ROOT`/`NODE`、`BPF_WORKQUEUE`。

### BPF 驗證器

**實現**（`kernel/bpf/verifier.c`，25403行）

驗證器是 BPF 安全性的基石，在程式載入時進行靜態分析。

**主要入口**（第25114行）：`bpf_check()`

**驗證步驟**：

1. **DAG 檢查** — 確保程式是有向無環圖
2. **循環檢測** — 拒絕後向邊（除非是有界循環）
3. **邊界檢查** — 驗證跳轉目標合法
4. **寄存器狀態追蹤** — 監控所有執行路徑的寄存器/棧狀態
5. **類型檢查** — 確保操作的類型安全
6. **記憶體存取驗證** — 防止越界讀寫

**寄存器模型**：
- R0：返回值
- R1-R5：參數傳遞
- R6-R9：callee-saved
- R10：棧幀指標（唯讀）

### BPF Helpers 與 kfuncs

**Helpers**（`net/core/filter.c:1737+`）：

核心提供的安全函數，BPF 程式透過 helper ID 呼叫：

- `bpf_skb_store_bytes` / `bpf_skb_load_bytes` — 封包資料操作
- `bpf_redirect` — 封包重定向
- `bpf_l3_csum_replace` / `bpf_l4_csum_replace` — 校驗和修改
- `bpf_map_lookup_elem` / `bpf_map_update_elem` — 映射操作

**kfuncs**（`kernel/bpf/cpumask.c` 等）：

新型核心函數介面，透過 BTF 匹配函數簽名：

- `bpf_cpumask_create()` / `bpf_cpumask_acquire()` / `bpf_cpumask_release()` — CPU 掩碼操作
- `bpf_cpumask_and()` / `bpf_cpumask_or()` — 位運算

### sched_ext：BPF 排程器

**實現**（`kernel/sched/ext.c`，211069 字節）

sched_ext 允許透過 BPF 程式實現自定義的排程策略。

**常數**（`kernel/sched/ext_internal.h:33-45`）：
- `SCX_DSP_DFL_MAX_BATCH=32` — 預設排程批次
- `SCX_WATCHDOG_MAX_TIMEOUT=30*HZ` — 看門狗超時

使用 `BPF_PROG_TYPE_STRUCT_OPS` 載入排程策略，透過 `sched_class` 結構體操作整合到核心排程器。

### BTF（BPF Type Format）

**實現**（`kernel/bpf/btf.c`，253332 字節）

BTF 提供核心資料結構的類型資訊，使 BPF 程式可以安全地存取核心結構體欄位。

BTF 種類（`btf.c:325-337`）：`INT`、`PTR`、`ARRAY`、`STRUCT`、`UNION`、`ENUM`、`FUNC`、`FUNC_PROTO`、`VAR`、`DATASEC`、`FLOAT`、`ENUM64` 等。

## 使用模式

### BPF 程式生命週期

```
使用者空間程式（C/Rust）
  → LLVM/clang 編譯為 BPF 字節碼
    → bpf() 系統呼叫載入
      → 驗證器檢查
        → JIT 編譯為原生機器碼
          → 附加到 hook 點（tracepoint、XDP、cgroup 等）
```

### 常見應用場景

| 場景 | 程式類型 | 映射類型 |
|------|---------|---------|
| 網路流量控制 | `SCHED_CLS` | `HASH`、`ARRAY` |
| 高性能封包處理 | `XDP` | `DEVMAP`、`PERCPU_ARRAY` |
| 系統追蹤 | `TRACEPOINT`、`KPROBE` | `RINGBUF`、`HASH` |
| 安全策略 | `LSM` | `HASH`、`ARRAY` |
| 自定義排程 | `STRUCT_OPS` | `ARRAY`、`TASK_STORAGE` |

## Android 相關性

### Android 中的 BPF 使用

- **流量控制**：`BPF_PROG_TYPE_SCHED_CLS` 用於 netd 的封包過濾與計費
- **Cgroup 控制**：`BPF_PROG_TYPE_CGROUP_*` 用於進程/socket 限制
- **安全策略**：`BPF_PROG_TYPE_LSM` 用於安全增強
- **sched_ext**：可能用於 Android 的排程優化

### GKI defconfig 中的 BPF 配置

| 配置 | 行號 | 用途 |
|------|------|------|
| `CONFIG_BPF_SYSCALL=y` | 6 | 啟用 bpf() 系統呼叫 |
| `CONFIG_BPF_JIT_ALWAYS_ON=y` | 8 | 強制 JIT 編譯 |
| `CONFIG_BPF_LSM=y` | 10 | BPF 安全模組 |
| `CONFIG_SCHED_CLASS_EXT=y` | 12 | sched_ext 支持 |

## 交叉參考

- [追蹤與 ftrace](tracing-and-ftrace.md) — BPF 追蹤程式使用的 tracepoint 基礎設施
- [Vendor Hooks](vendor-hooks.md) — 與 BPF tracepoint 程式的區別
- [Scheduler](../subsystems/scheduler.md) — sched_ext 的詳細分析
