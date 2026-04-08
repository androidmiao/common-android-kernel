---
type: concept
scope: kernel-wide
related:
  - ../concepts/rcu.md
  - ../concepts/interrupt-handling.md
last_updated: 2026-04-08
---

# 鎖定原語

## 概述

Linux 核心提供多層次的同步原語，從最輕量的原子操作到可睡眠的互斥鎖，適用於不同的上下文需求。核心實現位於 `kernel/locking/` 目錄，包含 mutex、rwsem、spinlock、qspinlock、rtmutex、lockdep 等關鍵實現。鎖的選擇取決於臨界區是否可以睡眠、持有時間長短、讀寫比例等因素。

## 機制

### 自旋鎖（Spinlock）

**檔案**：`include/linux/spinlock.h`（620行）、`kernel/locking/spinlock.c`、`kernel/locking/qspinlock.c`

自旋鎖是最基本的不可睡眠鎖。持有自旋鎖時，核心禁用搶佔（可選禁用中斷）。

**初始化**（`spinlock.h:101-114`）：

```c
raw_spin_lock_init(lock)  // 除錯模式呼叫 lockdep
```

**操作變體**：

| 函數 | 行為 |
|------|------|
| `spin_lock()` | 禁用搶佔，取得鎖 |
| `spin_lock_irqsave()` | 禁用中斷 + 搶佔，取得鎖 |
| `spin_lock_bh()` | 禁用 bottom-half + 搶佔，取得鎖 |
| `spin_trylock()` | 非阻塞嘗試 |

**Qspinlock（隊列自旋鎖）**（`qspinlock.c:27-150`）：

基於 MCS（Mellor-Crummey Scott）鎖的壓縮版本，使用 32 位元表示：

```
(queue_tail, pending_bit, lock_value)

快速路徑：(0,0,0) → (0,0,1) → (0,0,0)
待決路徑：(0,1,1) → (0,1,0)
隊列路徑：(*,x,y) → (*,0,0) → (*,0,1) → (0,0,1)
```

Per-CPU 隊列（`qspinlock.c:80`）：

```c
static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[_Q_MAX_NODES]);
// _Q_MAX_NODES = 4（task, softirq, hardirq, nmi 上下文）
```

**記憶體屏障**（`spinlock.h:175-177`）：

```c
smp_mb__after_spinlock()  // 保證 ACQUIRE 語義並實現 RCsc 順序一致性鎖
```

### 互斥鎖（Mutex）

**檔案**：`include/linux/mutex.h`（259行）、`kernel/locking/mutex.c`

互斥鎖是可睡眠的鎖，只能在進程上下文中使用（不能在中斷或 softirq 中使用）。

**資料結構**（`mutex.h:78-124`，非 PREEMPT_RT）：

```c
struct mutex {
    atomic_long_t        owner;      // 所有者任務指標 + 標誌
    raw_spinlock_t       wait_lock;
    struct list_head     wait_list;
    struct optimistic_spin_queue osq; // CONFIG_MUTEX_SPIN_ON_OWNER
};
```

**所有者編碼**（`mutex.c:57-71`）：

```
owner = task_struct * | MUTEX_FLAG_HANDOFF | MUTEX_FLAG_PICKUP
```

最低位用於 handoff（握手）與 pickup（拾取）標誌。

**快速路徑**（`mutex.c:84-118`）：

```c
__mutex_trylock_common()
  → atomic_long_try_cmpxchg_acquire()  // 無鎖原子 CAS
```

**適應性自旋（OSQ）**：當 mutex 持有者正在 CPU 上執行時，等待者選擇自旋而非睡眠，減少上下文切換開銷。

### 讀寫信號量（RW Semaphore）

**檔案**：`include/linux/rwsem.h`（310行）、`kernel/locking/rwsem.c`

允許多讀者並行或單寫者獨占。

**資料結構**（`rwsem.h:48-67`）：

```c
struct rw_semaphore {
    atomic_long_t count;   // 讀者計數 + 寫入標誌
    atomic_long_t owner;   // 所有者 + READER_OWNED 標誌
    struct optimistic_spin_queue osq;
    raw_spinlock_t wait_lock;
    struct list_head wait_list;
};
```

**位元配置**（`rwsem.c:82-129`，64 位元）：

| 位元 | 意義 |
|------|------|
| 0 | `RWSEM_WRITER_LOCKED`（寫入持有）|
| 1 | `RWSEM_FLAG_WAITERS`（有等待者）|
| 2 | `RWSEM_FLAG_HANDOFF`（握手標誌）|
| 63 | `RWSEM_FLAG_READFAIL`（讀失敗）|
| 8-62 | 55 位讀者計數 |

**所有者追蹤**（`rwsem.c:37-66`）：

```c
#define RWSEM_READER_OWNED  (1UL << 0)   // 讀者持有提示
#define RWSEM_NONSPINNABLE  (1UL << 1)   // 禁止自旋
```

### 序列鎖（Seqlock）

**檔案**：`include/linux/seqlock.h`（1326行）、`include/linux/seqlock_types.h`（93行）

適用於讀多寫少、讀者容忍重試的場景。

**類型**：

1. **`seqcount_t`**（`seqlock_types.h:33-38`）— 原始計數器，需外部寫入保護
2. **`seqcount_LOCKNAME_t`**（62-72行）— 帶鎖的序列計數（spinlock、mutex、rwlock 等）
3. **`seqlock_t`**（84-91行）— 內嵌 spinlock 的完整序列鎖

**讀路徑**（`seqlock.h:157-177`）：

```c
do {
    seq = smp_load_acquire(&s->seqcount.sequence);
    // ... 讀取受保護資料 ...
} while (seq & 1 || seq != smp_load_acquire(&s->seqcount.sequence));
// 偶數 = 寫入完成，序列未變 = 資料一致
```

### RT 互斥鎖與優先權繼承

**檔案**：`kernel/locking/rtmutex.c`、`rtmutex_common.h`

RT mutex 提供優先權繼承——當高優先權任務等待低優先權任務持有的鎖時，低優先權任務被暫時提升至等待者的優先權，防止優先權反轉。

**所有者編碼**（`rtmutex.c:69-93`）：

```c
lock->owner:
  NULL | 0: 未鎖定，無等待者
  NULL | 1: 未鎖定，有等待者（頂部等待者將取得鎖）
  task | 0: 已鎖定（快速釋放可能）
  task | 1: 已鎖定且有等待者

#define RT_MUTEX_HAS_WAITERS  0x01  // 最低位
```

等待佇列使用紅黑樹，按優先權排序。在 `CONFIG_PREEMPT_RT` 下，mutex 和 rwsem 都基於 rtmutex 實現。

### Lockdep：死鎖檢測器

**檔案**：`include/linux/lockdep.h`（664行）、`kernel/locking/lockdep.c`（180070行）

Lockdep 在執行時記錄所有鎖獲取事件，檢測：

- 圓形依賴（AB-BA 死鎖）
- 鎖反向
- 硬中斷/軟中斷安全性違規

**核心結構**（`lockdep.h:48-83`）：

```c
struct lock_list {
    struct lock_class *class;     // 鎖類別
    struct lock_class *links_to;  // 依賴目標
    u16 distance;                 // BFS 距離
    u8 dep;                       // 依賴位元圖
};

struct lock_chain {
    unsigned int irq_context: 2;  // 硬/軟中斷上下文
    unsigned int depth: 6;        // 鏈深度
    u64 chain_key;                // 雜湊鍵
};
```

**等待類型**：

```c
enum lockdep_wait_type {
    LD_WAIT_INV = 0,    // 無效
    LD_WAIT_SPIN,       // 自旋鎖
    LD_WAIT_SLEEP,      // 睡眠鎖（mutex/rwsem）
};
```

## 使用模式

### 鎖選擇指南

| 場景 | 推薦鎖 | 原因 |
|------|--------|------|
| 中斷上下文 | `spin_lock_irqsave()` | 不可睡眠 |
| Softirq 上下文 | `spin_lock_bh()` | 不可睡眠，防止 deadlock |
| 短臨界區（進程） | `spin_lock()` | 低開銷 |
| 長臨界區（進程） | `mutex_lock()` | 可睡眠 |
| 讀多寫少（可睡眠）| `down_read()` / `down_write()` | 讀者並行 |
| 讀多寫少（不可睡眠）| seqlock | 讀者無鎖重試 |
| 讀多寫極少 | [RCU](rcu.md) | 讀者零開銷 |

## Android 相關性

鎖定系統本身無 Android 特定的 Vendor Hooks——Android 選擇依賴上游的 `CONFIG_PREEMPT` 和 `CONFIG_PREEMPT_RT` 配置。GKI defconfig 啟用 `CONFIG_PREEMPT=y`（第11行），提供完全可搶佔核心。

在 `CONFIG_PREEMPT_RT` 變體下，mutex 和 rwsem 都基於 rtmutex 實現，提供優先權繼承。

## 交叉參考

- [RCU](rcu.md) — 讀側零開銷的同步機制
- [中斷處理](interrupt-handling.md) — 影響鎖選擇的上下文約束
