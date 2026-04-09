---
type: analysis
question: "ACK 各子系統的鎖定模式比較：從 spinlock 層次到 RCU 讀取路徑的設計選擇"
pages_consulted:
  - concepts/locking-primitives.md
  - entities/binder.md
  - subsystems/scheduler.md
  - subsystems/memory-management.md
  - subsystems/filesystems.md
  - subsystems/security.md
  - subsystems/ipc.md
  - subsystems/block-layer.md
  - subsystems/networking.md
  - data-structures/binder_proc.md
  - data-structures/sk_buff.md
created: 2026-04-09
---

# 跨子系統鎖定模式分析

## 總覽

核心的效能和正確性高度依賴鎖定策略的選擇。本分析比較 ACK 各主要子系統的鎖定模式，識別共通設計模式，並分析其對 Android 效能的影響。

## 鎖定模式分類

### 模式一：階層式 Spinlock（嚴格順序）

**代表**：Binder、IPC

Binder 使用三層 spinlock 層次結構，取鎖順序嚴格不可逆轉：

```
outer_lock (proc)
  → node->lock (per-node)
    → inner_lock (proc, 保護 todo/stack)
```

IPC 子系統類似地使用三層策略：

```
RCU 讀取鎖 (lock-free 查找)
  → ipc_ids.rwsem (ID 表修改)
    → kern_ipc_perm.lock (per-object 狀態)
```

**特點**：
- 函式命名標註鎖定狀態（Binder: `_olocked`/`_nlocked`/`_ilocked`）
- 層次結構消除死鎖可能性（只要遵守順序）
- 犧牲部分並行性以換取可預測性

**Android 影響**：Binder 的鎖定層次是高負載下的主要效能瓶頸。`inner_lock` 保護 todo 列表和 transaction_stack，在系統服務密集 IPC 時成為競爭熱點。

### 模式二：Per-CPU 結構 + RCU

**代表**：排程器、Block Layer、網路

排程器的 per-CPU run queue (`struct rq`) 配合 `rq->__lock` spinlock：

```
每個 CPU 一個 rq，各自獨立鎖定
負載均衡時才需要跨 CPU 取鎖（src_rq → dst_rq，按 CPU 編號排序）
RCU 讀取排程域 (sched_domain) 拓撲
```

Block layer 的 multi-queue 架構：

```
per-CPU software queue (blk_mq_ctx, spinlock)
  → per-HW-queue (blk_mq_hw_ctx)
    → sbitmap 原子 tag 分配
```

網路收包路徑：

```
NAPI softirq (per-CPU, 無鎖)
  → per-socket lock (bh_lock_sock)
  → RCU 路由/規則查找
```

**特點**：
- 熱路徑完全在 per-CPU 結構上操作，消除跨 CPU 鎖定競爭
- RCU 保護讀取密集型資料（路由表、排程域、裝置列表）
- 跨 CPU 操作（負載均衡、設備查找）使用 RCU 或有序鎖定

**Android 影響**：per-CPU 設計在大小核異構架構（big.LITTLE/DynamIQ）上表現良好，每個核心的 run queue 獨立運作，廠商透過 78 個 vendor hooks 調整跨核心任務遷移策略。

### 模式三：rwsem + 細粒度鎖

**代表**：記憶體管理、檔案系統

記憶體管理的鎖定層次：

```
mm->mmap_lock (rwsem, 保護 VMA 樹)
  → per-VMA lock (CONFIG_PER_VMA_LOCK, 細粒度 page fault)
    → page table lock (spinlock, 保護 PTE 修改)
      → LRU list lock (spinlock, 保護回收列表)
```

檔案系統的鎖定層次：

```
inode->i_rwsem (rwsem, 保護檔案操作)
  → dentry->d_lockref (lockref 原子操作)
    → dcache hash spinlock (保護 dentry cache)
    → address_space 樹 lock (保護 page cache)
```

**特點**：
- rwsem 允許多讀者並行（page fault 讀取路徑不互斥）
- 細粒度鎖（per-VMA、per-dentry）減少競爭範圍
- lockref 利用原子操作避免在熱路徑上取鎖

**Android 影響**：`CONFIG_PER_VMA_LOCK` 對 Android 多執行緒應用特別有利——同一行程的多個執行緒可以平行處理不同 VMA 的 page fault，而不必競爭全域 `mmap_lock`。

### 模式四：Static Call 派發

**代表**：LSM 框架、Vendor Hooks

LSM 的 273 個 hooks 使用 static call 機制：

```
security_file_open(file)
  → LSM_LOOP_UNROLL 展開 static call
    → selinux_file_open(file)    [直接呼叫，無間接跳轉]
    → landlock_file_open(file)   [直接呼叫]
```

Vendor hooks 的 restricted 型別也使用 static call：

```
trace_android_rvh_select_task_rq_fair(...)
  → static_call(...)  [單一廠商函式，直接呼叫]
```

**特點**：
- 未啟用時編譯為 NOP（`static_branch_unlikely()`）
- 啟用後為直接呼叫（非間接跳轉），對 CPU 分支預測器友好
- 消除了傳統函式指標陣列的間接呼叫開銷

**Android 影響**：static call 確保安全檢查和 vendor hooks 在熱路徑上的開銷最小化——這對 Android 的效能敏感場景（如 Binder IPC 每秒數千次觸發 SELinux 檢查）至關重要。

### 模式五：原子操作 + 無鎖演算法

**代表**：網路（sk_buff 參考計數）、記憶體（page 參考計數）

```
sk_buff: atomic_t users (skb_get/kfree_skb)
page: atomic_t _refcount (get_page/put_page)
sbitmap: atomic bitmap for tag allocation
```

**特點**：
- 單一計數器操作不需要鎖
- 適用於生命週期管理（建立/銷毀）
- 缺點：不適合保護複合操作

## 信號量特殊案例：IPC sem_array

IPC 信號量使用獨特的自適應鎖定策略：

```
if (sma->use_global_lock) {
    // 全域鎖定：保護整個 sem_array
    ipc_lock_object(&sma->sem_perm);
} else {
    // 細粒度鎖定：僅鎖定受影響的 sem
    spin_lock(&sem->lock);
}
```

`USE_GLOBAL_LOCK_HYSTERESIS = 10`：在細粒度和全域模式之間設定切換閾值，避免頻繁模式切換。

## 各子系統鎖定策略比較表

| 子系統 | 主要鎖定原語 | 讀取路徑 | 寫入路徑 | 競爭熱點 |
|--------|-------------|---------|---------|---------|
| **排程器** | per-CPU spinlock | RCU (sched_domain) | rq->__lock | 跨 CPU 負載均衡 |
| **記憶體管理** | rwsem + spinlock | per-VMA lock / RCU | mmap_lock write | mmap_lock (fork, munmap) |
| **Binder** | 三層 spinlock | — | inner_lock | todo 列表操作 |
| **檔案系統** | rwsem + lockref | RCU walk (dcache) | i_rwsem write | inode write 操作 |
| **網路** | per-CPU + per-socket | RCU (路由/規則) | bh_lock_sock | conntrack 更新 |
| **Block Layer** | per-CPU + sbitmap | — | blk_mq_ctx spinlock | tag 分配 (sbitmap) |
| **Security** | static call | RCU (SELinux policy) | AVC spinlock | AVC cache 競爭 |
| **IPC** | rwsem + spinlock | RCU (ID 查找) | kern_ipc_perm.lock | 信號量操作 |

## RT Preemption 影響

ACK 的 GKI defconfig 啟用 `CONFIG_PREEMPT=y`（完全搶佔），但不是 `PREEMPT_RT`。在完全搶佔模式下：

- Spinlock 仍然禁止搶佔（不同於 PREEMPT_RT 的可睡眠 spinlock）
- Softirq 可被搶佔
- `mutex` 和 `rwsem` 是可睡眠的

若未來 ACK 啟用 `PREEMPT_RT`，所有 spinlock 會自動轉換為 RT mutex，帶來完整的優先權繼承但增加上下文切換開銷。

## 設計模式總結

1. **分而治之**：per-CPU 結構（排程器、block layer、網路）消除跨核心競爭
2. **讀寫分離**：rwsem（mm、fs）允許讀者平行；RCU 提供零開銷讀取
3. **階層排序**：嚴格的鎖定層次（Binder、IPC）保證無死鎖
4. **自適應策略**：IPC 信號量根據工作負載動態切換鎖定粒度
5. **靜態派發**：LSM 和 vendor hooks 用 static call 消除間接呼叫開銷
6. **原子極簡**：參考計數（sk_buff、page）用原子操作而非鎖

## 交叉參考

- [鎖定原語](../concepts/locking-primitives.md) — 各鎖定原語實現細節
- [排程器](../subsystems/scheduler.md) — per-CPU run queue 鎖定
- [記憶體管理](../subsystems/memory-management.md) — mmap_lock 與 per-VMA lock
- [Binder](../entities/binder.md) — 三層 spinlock 架構
- [`binder_proc`](../data-structures/binder_proc.md) — Binder 鎖定欄位
- [`sk_buff`](../data-structures/sk_buff.md) — 網路封包參考計數
- [IPC](../subsystems/ipc.md) — 三層鎖定策略
- [安全子系統](../subsystems/security.md) — LSM static call 派發
- [Vendor Hooks](../concepts/vendor-hooks.md) — restricted hook static_call
