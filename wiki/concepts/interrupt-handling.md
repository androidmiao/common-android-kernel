---
type: concept
scope: kernel-wide
related:
  - ../concepts/locking-primitives.md
  - ../concepts/rcu.md
last_updated: 2026-04-08
---

# 中斷處理

## 概述

Linux 核心的中斷處理分為**上半部（top-half）**和**下半部（bottom-half）**。上半部在硬體中斷上下文中快速執行，下半部（softirq、tasklet、workqueue）延遲處理較耗時的工作。核心實現位於 `kernel/irq/`（IRQ 子系統）和 `kernel/softirq.c`（softirq/tasklet）。

## 機制

### 硬體中斷流程（Top-Half）

**中斷描述符**（`kernel/irq/irqdesc.c`）：

每個 IRQ 線路由一個 `irq_desc` 結構表示（`irqdesc.c:115-140` 初始化）：

- `irq_common_data` — 共享 IRQ 資料（CPU 親和力、MSI 資訊）
- `irq_data` — 架構相關資料（硬體中斷號、`irq_chip`）
- `handle_irq` — 流程處理器函數指標
- `kstat_irqs` — 按 CPU 的中斷統計

**IRQ 資料結構**（`include/linux/irq.h:177-188`）：

```c
struct irq_data {
    unsigned int  irq;      // Linux 中斷號
    unsigned long hwirq;    // 硬體中斷號（域相關）
    struct irq_chip *chip;  // 中斷控制器操作
    struct irq_domain *domain;  // IRQ 域
};
```

**中斷服務路徑**（`kernel/irq/handle.c:185-227`）：

```
硬體中斷 → CPU 異常處理（GIC/APIC）
  → irq_desc->handle_irq()  // 流程處理器
    → __handle_irq_event_percpu()
      → action->handler()   // 已安裝的硬中斷處理器
      → 若返回 IRQ_WAKE_THREAD → 喚醒執行緒化處理器
```

**流程處理器類型**：

| 函數 | 觸發方式 | 適用場景 |
|------|---------|---------|
| `handle_edge_irq()` | 邊界觸發 | GPIO、按鍵 |
| `handle_level_irq()` | 電平觸發 | 傳統 PCI |
| `handle_fasteoi_irq()` | 快速 EOI | GIC 優化 |
| `handle_percpu_irq()` | Per-CPU | 計時器、IPI |

### IRQ 動作結構（irqaction）

**定義**（`include/linux/interrupt.h:123-140`）：

```c
struct irqaction {
    irq_handler_t  handler;    // 硬中斷處理器
    irq_handler_t  thread_fn;  // 執行緒化處理器
    struct task_struct *thread; // 專用中斷執行緒
    unsigned int   flags;      // IRQF_* 標誌
    const char    *name;
};
```

### IRQ 域映射

**定義**（`include/linux/irqdomain.h:68-125`）：

IRQ 域負責將硬體中斷號（hwirq）映射到 Linux 中斷號（virq）。

```c
struct irq_domain_ops {
    int (*map)(struct irq_domain *, unsigned int virq, irq_hw_number_t hw);
    int (*xlate)(...);        // 解析設備樹規範
    int (*activate)(...);     // 啟用硬體向量
    void (*deactivate)(...);
};
```

**IRQ 親和力**（`irqdesc.c:55-113`）：

每個 `irq_desc` 包含 `affinity`、`effective_affinity`、`pending_mask`，控制中斷在哪個 CPU 上執行。

### 執行緒化中斷（Threaded IRQs）

**API**（`include/linux/interrupt.h:155-176`）：

`request_threaded_irq()` 將中斷處理分為：
- 硬中斷處理器（快速，在中斷上下文）
- 執行緒函數（在專用核心執行緒中，可睡眠）

**強制執行緒化**（`kernel/irq/manage.c:27-36`）：

```c
force_irqthreads_key  // 靜態分支，強制所有中斷執行緒化
```

**IRQF_ONESHOT**（`handle.c:61-145`）：

設定此標誌時，硬中斷處理器禁用中斷線路，直到執行緒函數完成後才重新啟用——防止中斷風暴。

### 下半部機制（Bottom-Half）

#### Softirq

**實現**（`kernel/softirq.c:62-85`）：

10 種 softirq，優先順序由高到低：

| 索引 | 名稱 | 用途 |
|------|------|------|
| 0 | `HI_SOFTIRQ` | 高優先權 tasklet |
| 1 | `TIMER_SOFTIRQ` | 計時器 |
| 2 | `NET_TX_SOFTIRQ` | 網路發送 |
| 3 | `NET_RX_SOFTIRQ` | 網路接收 |
| 4 | `BLOCK_SOFTIRQ` | 區塊 I/O |
| 5 | `IRQ_POLL_SOFTIRQ` | IRQ 輪詢 |
| 6 | `TASKLET_SOFTIRQ` | 一般 tasklet |
| 7 | `SCHED_SOFTIRQ` | 排程 |
| 8 | `HRTIMER_SOFTIRQ` | 高精度計時器 |
| 9 | `RCU_SOFTIRQ` | RCU |

執行時機：硬中斷退出時透過 `do_softirq()` 或 `ksoftirqd` 執行緒。

#### Tasklet

**結構**（`include/linux/interrupt.h:722-733`）：

```c
struct tasklet_struct {
    struct tasklet_struct *next;
    unsigned long state;     // TASKLET_STATE_SCHED, TASKLET_STATE_RUN
    atomic_t count;          // 禁用計數
    bool use_callback;
    union {
        void (*func)(unsigned long data);
        void (*callback)(struct tasklet_struct *t);
    };
    unsigned long data;
};
```

Tasklet 特性：在單一 CPU 上序列化執行，不同 tasklet 可在不同 CPU 並行。Per-CPU 隊列位於 `softirq.c:843-900`。

#### Workqueue

工作佇列在進程上下文中的核心執行緒（kworker）中執行，可以睡眠。適用於需要互斥鎖或長時間操作的下半部工作。

## 使用模式

### 下半部選擇指南

| 機制 | 上下文 | 可睡眠 | 適用場景 |
|------|--------|--------|---------|
| Softirq | 中斷上下文 | 否 | 高頻、效能關鍵（網路、排程）|
| Tasklet | 中斷上下文 | 否 | 中等頻率，需序列化 |
| Workqueue | 進程上下文 | 是 | 需睡眠鎖或長時間操作 |
| Threaded IRQ | 進程上下文 | 是 | 硬體中斷的延遲處理 |

## Android 相關性

**Vendor Hook**：`include/trace/hooks/gic.h` 提供 `android_vh_gic_set_affinity` hook，允許廠商自定義 GIC 中斷親和力設定。

GKI defconfig 啟用 `CONFIG_PREEMPT=y`，搭配執行緒化中斷可改善即時響應性。

## 交叉參考

- [鎖定原語](locking-primitives.md) — 不同中斷上下文對鎖的約束
- [RCU](rcu.md) — Softirq 中的 RCU callback 處理
