---
type: concept
scope: kernel-wide
related:
  - ../concepts/locking-primitives.md
  - ../concepts/interrupt-handling.md
  - ../concepts/memory-allocation.md
last_updated: 2026-04-08
---

# RCU（Read-Copy-Update）

## 概述

RCU 是 Linux 核心中最重要的同步機制之一，專為**讀多寫少**的場景設計。讀側幾乎零開銷（不需要原子操作、記憶體屏障或鎖），寫側則透過「複製後更新」的方式修改資料，等待所有讀者完成後再釋放舊版本。

核心實現位於 `kernel/rcu/` 目錄，以 Tree RCU（`tree.c`，167714 字節）為主要實現。

## 機制

### RCU 種類

**四種主要實現**（`kernel/rcu/Kconfig`）：

| 種類 | Kconfig | 適用場景 |
|------|---------|---------|
| **Tree RCU** | `CONFIG_TREE_RCU` | SMP 系統（預設）|
| **Preempt RCU** | `CONFIG_PREEMPT_RCU` | 可搶佔核心（RT 需求）|
| **Tiny RCU** | `CONFIG_TINY_RCU` | 單 CPU 嵌入式系統 |
| **SRCU** | `CONFIG_TREE_SRCU` | 需要在讀側睡眠的場景 |

另有 Tasks RCU（`kernel/rcu/tasks.h`）用於追蹤自願上下文切換。

### 核心 API

**定義**：`include/linux/rcupdate.h`（第40-225行）

| API | 用途 |
|-----|------|
| `rcu_read_lock()` / `rcu_read_unlock()` | 讀側臨界區 |
| `synchronize_rcu()` | 同步等待 Grace Period |
| `call_rcu(head, func)` | 非同步 callback 註冊 |
| `call_rcu_hurry(head, func)` | 緊急 callback（`CONFIG_RCU_LAZY`）|
| `rcu_dereference(ptr)` | RCU 保護的指標檢索 |
| `rcu_assign_pointer(ptr, val)` | RCU 保護的指標發佈 |

**讀側實現**（`rcupdate.h:70-108`）：
- `CONFIG_TREE_RCU`：`rcu_read_lock()` 等同 `preempt_disable()`
- `CONFIG_PREEMPT_RCU`：使用專門的 `__rcu_read_lock()` / `__rcu_read_unlock()`

### Grace Period 機制

Grace Period 是 RCU 的核心概念——它是一段時間窗口，保證在此窗口開始前進入讀側臨界區的所有讀者已經離開。

**全局狀態**（`kernel/rcu/tree.c:500-574`）：

- `rcu_state.gp_seq` — 全局 GP 序號（低2位為控制標誌）
- `rcu_state.gp_state` — 當前 GP 狀態（IDLE、WAIT_GPS 等）

**Quiescent State（QS）檢測**：

QS 是 CPU 報告「我已經不在任何讀側臨界區中」的事件。不同 RCU 種類的 QS 定義：

| 種類 | QS 定義 |
|------|---------|
| Tree RCU | 排程、preempt 禁用退出、CPU 空閒 |
| Preempt RCU | 自願上下文切換 |
| SRCU | `srcu_read_unlock()` |
| Tasks RCU | 自願上下文切換、空閒、用戶模式 |

QS 報告從葉節點逐層上升到根節點。

### RCU 樹階層結構

**定義**（`kernel/rcu/tree.h:41-139`）：

```c
struct rcu_node {
    raw_spinlock_t lock;           // 層級保護鎖
    unsigned long gp_seq;          // Grace Period 序號
    unsigned long qsmask;          // 需要 QS 的 CPU/節點遮罩
    unsigned long expmask;         // 加急 GP 遮罩
    struct list_head blkd_tasks;   // 阻塞任務列表
    struct rcu_node *parent;       // 父節點
    int grplo, grphi;              // 覆蓋的 CPU 範圍
    u8 level;                      // 樹層級
};
```

**Per-CPU 資料**（`tree.h:189-260`）：

```c
struct rcu_data {
    unsigned long gp_seq;              // 追蹤 GP 序號
    union rcu_noqs cpu_no_qs;          // CPU 無 QS 標誌
    struct rcu_node *mynode;           // 所屬葉節點
    struct rcu_segcblist cblist;       // 分段 callback 列表
    bool rcu_need_heavy_qs;            // 需要重型 QS
    bool rcu_urgent_qs;                // 需要輕型 QS
};
```

**Fanout 配置**（`Kconfig:156-200`）：

- `CONFIG_RCU_FANOUT`：分支因數（64位系統預設 64）
- `CONFIG_RCU_FANOUT_LEAF`：葉層分支因數（預設 16）

### Callback 分段列表

**定義**（`kernel/rcu/rcu_segcblist.h`）：

四個段按 GP 階段分類：

| 段 | 意義 |
|----|------|
| `RCU_DONE_TAIL` | 已完成的 callback |
| `RCU_WAIT_TAIL` | 等待 GP 完成的 callback |
| `RCU_NEXT_READY_TAIL` | 下一個 GP 的 callback |
| `RCU_NEXT_TAIL` | 最新註冊的 callback |

### NOCB（No-Callback）CPU 離載

**實現**（`kernel/rcu/tree_nocb.h:16-105`）：

將 callback 處理從特定 CPU 離載到專用 kthread，減少 CPU 的 RCU 負擔。

- `rcu_nocbs=<cpu-list>` 啟動參數指定離載 CPU
- `nocb_cb_kthread` — Callback 調用執行緒
- `nocb_gp_kthread` — Grace Period 執行緒

### 加急 Grace Period

**實現**（`kernel/rcu/tree_exp.h:20-68`）：

`synchronize_rcu_expedited()` 透過 IPI 強制所有 CPU 立即報告 QS，大幅縮短 GP 等待時間，但會中斷其他 CPU。

### RCU 受保護列表

**定義**（`include/linux/rculist.h`）：

| API | 行號 | 用途 |
|-----|------|------|
| `list_for_each_rcu()` | 50-53 | RCU 安全的列表遍歷 |
| `list_add_rcu()` | 125-128 | RCU 列表新增 |
| `list_add_tail_rcu()` | 146-149 | RCU 列表尾部新增 |
| `list_del_rcu()` | — | RCU 安全的刪除 |

使用 `rcu_assign_pointer()` 發佈和 `rcu_dereference()` 讀取，確保讀-寫順序正確。

## 使用模式

### 典型用法

```c
// 讀側
rcu_read_lock();
p = rcu_dereference(global_ptr);
// 使用 p...
rcu_read_unlock();

// 寫側
new = kmalloc(...);
*new = *old;                    // 複製舊資料
new->field = new_value;         // 修改
rcu_assign_pointer(global_ptr, new);  // 發佈新版本
synchronize_rcu();              // 等待所有讀者
kfree(old);                     // 釋放舊版本
```

### 何時使用 RCU

- 讀遠多於寫的資料結構（路由表、模組列表、PID 查詢等）
- 讀側不能容忍鎖爭用的高頻路徑
- 需要在中斷上下文中安全讀取的資料

## Android 相關性

GKI defconfig 中的 RCU 配置：

- `CONFIG_RCU_NOCB_CPU=y`（21行）— 啟用 NOCB 離載，將 RCU callback 處理從關鍵 CPU 移開
- `CONFIG_RCU_LAZY=y`（22行）— 延遲 callback 執行，節省電力

在 `PREEMPT_RT` 變體下，`CONFIG_PREEMPT_RCU` 自動選擇，且 `use_softirq` 預設禁用（`tree.c:115`）。

## 交叉參考

- [鎖定原語](locking-primitives.md) — RCU 與其他鎖的比較
- [中斷處理](interrupt-handling.md) — RCU softirq 的執行時機
- [記憶體分配](memory-allocation.md) — `SLAB_TYPESAFE_BY_RCU` 機制
