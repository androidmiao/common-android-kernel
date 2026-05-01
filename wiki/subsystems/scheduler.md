---
type: subsystem
kernel_path: "kernel/sched/"
upstream: partial
android_patches: "vendor hooks and GKI scheduling configuration"
vendor_hooks:
  - include/trace/hooks/sched.h
related:
  - ../data-structures/task_struct.md
  - ../concepts/locking-primitives.md
  - ../concepts/vendor-hooks.md
  - ../concepts/bpf.md
  - ../concepts/gki.md
  - ../concepts/tracing-and-ftrace.md
  - ../analyses/vendor-hooks-distribution.md
  - ../analyses/locking-patterns.md
last_updated: 2026-04-25
---

# Scheduler 子系統深度分析

> **核心路徑**: `kernel/sched/`
> **核心版本**: Linux 6.19-rc8 (Android Common Kernel)
> **原始碼規模**: ~62,743 行 (含 .c 與 .h)
> **排程類別**: stop > deadline > rt > fair (CFS/EEVDF) > idle > ext (BPF)

---

## 目錄

1. [架構總覽](#1-架構總覽)
2. [核心資料結構](#2-核心資料結構)
3. [排程類別 (Scheduling Classes)](#3-排程類別)
4. [EEVDF 演算法 — CFS 的核心](#4-eevdf-演算法--cfs-的核心)
5. [即時排程器 (RT Scheduler)](#5-即時排程器)
6. [截止期限排程器 (Deadline Scheduler)](#6-截止期限排程器)
7. [sched_ext — BPF 可擴展排程](#7-sched_ext--bpf-可擴展排程)
8. [負載追蹤 (PELT)](#8-負載追蹤-pelt)
9. [負載均衡 (Load Balancing)](#9-負載均衡)
10. [排程域與拓撲 (Topology)](#10-排程域與拓撲)
11. [CPU 頻率調節 (schedutil)](#11-cpu-頻率調節-schedutil)
12. [核心排程 (Core Scheduling)](#12-核心排程)
13. [壓力阻塞資訊 (PSI)](#13-壓力阻塞資訊-psi)
14. [Android Vendor Hooks](#14-android-vendor-hooks)
15. [Feature Flags 與執行期調校](#15-feature-flags-與執行期調校)
16. [建置系統與編譯架構](#16-建置系統與編譯架構)
17. [Syscall 介面](#17-syscall-介面)
18. [統計與除錯](#18-統計與除錯)
19. [輔助機制](#19-輔助機制)
20. [檔案索引](#20-檔案索引)
21. [交叉參考](#21-交叉參考)

---

## 1. 架構總覽

Linux 排程器採用**模組化類別架構 (pluggable scheduling class)**。每個排程類別透過 `struct sched_class` 虛擬函式表提供統一介面，核心排程器依照**嚴格優先順序**逐一查詢各類別：

```
stop_sched_class          ← 最高優先 (CPU 停機、hotplug)
  ↓
dl_sched_class            ← SCHED_DEADLINE (EDF)
  ↓
rt_sched_class            ← SCHED_FIFO / SCHED_RR (固定優先級)
  ↓
fair_sched_class          ← SCHED_NORMAL / SCHED_BATCH (EEVDF)
  ↓
ext_sched_class           ← sched_ext (BPF 自訂排程)
  ↓
idle_sched_class          ← 最低優先 (CPU 閒置)
```

排程的核心入口是 `__schedule()` (位於 `core.c`)，它負責：取得 per-CPU run queue 鎖、呼叫 `pick_next_task()` 遍歷排程類別、執行 context switch。

### 排程觸發時機

排程器在以下情境被觸發：任務主動讓出 CPU (`schedule()`)、從中斷返回時檢查到 `TIF_NEED_RESCHED` 旗標、timer tick 中 `scheduler_tick()` 發現當前任務應被搶佔、以及任務被喚醒時 `try_to_wake_up()` 判斷需要搶佔當前執行者。

---

## 2. 核心資料結構

### 2.1 Run Queue (`struct rq`) — `sched.h:1128`

每個 CPU 都有一個 `struct rq`，是排程器最核心的結構：

```c
struct rq {
    raw_spinlock_t   __lock;          // per-CPU 排程鎖
    unsigned int     nr_running;       // 可執行任務總數

    struct cfs_rq    cfs;             // CFS 排程佇列
    struct rt_rq     rt;              // RT 排程佇列
    struct dl_rq     dl;              // Deadline 排程佇列
    struct scx_rq    scx;            // sched_ext 佇列

    struct task_struct *curr;         // 當前執行任務
    struct task_struct *donor;        // Proxy Execution 的捐贈者
    struct task_struct *idle;         // idle 任務

    u64              clock;           // rq 時脈
    u64              clock_task;      // 排除 IRQ 的任務時脈
    u64              clock_pelt;      // PELT 負載追蹤時脈

    struct sched_avg avg_rt;          // RT 類別平均負載
    struct sched_avg avg_dl;          // DL 類別平均負載
    struct sched_avg avg_irq;         // IRQ 平均時間
    struct sched_avg avg_hw;          // 硬體壓力平均

    struct root_domain *rd;           // 根域 (load balancing)
    struct sched_domain *sd;          // 當前排程域
    // ...
};
```

所有對 run queue 的修改都必須持有 `rq->__lock`，且通常在禁止 IRQ 的情境下操作。

### 2.2 CFS Run Queue (`struct cfs_rq`) — `sched.h:682`

```c
struct cfs_rq {
    struct load_weight    load;             // 佇列總權重
    unsigned int          nr_queued;        // 已排入任務數
    unsigned int          h_nr_runnable;    // 階層式可執行數

    s64                   avg_vruntime;     // 加權平均虛擬執行時間
    u64                   zero_vruntime;    // vruntime 基準參考點
    struct rb_root_cached tasks_timeline;   // 紅黑樹 (依 vruntime 排序)

    struct sched_entity   *curr;            // 當前執行實體
    struct sched_entity   *next;            // 下一個偏好實體 (buddy)

    struct sched_avg      avg;              // PELT 負載追蹤
    struct list_head      leaf_cfs_rq_list; // 群組排程階層列表
    // ...
};
```

`tasks_timeline` 紅黑樹是 CFS 的核心結構，所有可執行任務依 vruntime 排序，確保 O(log n) 的插入與刪除。

### 2.3 排程實體 (`struct sched_entity`)

每個任務在 CFS 中的代表：

```c
struct sched_entity {
    struct load_weight  load;       // 任務權重 (由 nice 值決定)
    struct rb_node      run_node;   // 紅黑樹節點
    u64                 vruntime;   // 虛擬執行時間
    u64                 deadline;   // EEVDF 截止期限
    u64                 slice;      // 配置的時間片段
    s64                 vlag;       // 虛擬延遲 (公平性指標)
    int                 on_rq;      // 是否在佇列中
    struct sched_avg    avg;        // PELT 負載追蹤
    struct cfs_rq       *my_q;      // 群組排程的子佇列
};
```

### 2.4 排程類別介面 (`struct sched_class`) — `sched.h:2468`

```c
struct sched_class {
    void (*enqueue_task)(struct rq *, struct task_struct *, int flags);
    void (*dequeue_task)(struct rq *, struct task_struct *, int flags);
    void (*yield_task)(struct rq *);
    void (*check_preempt_curr)(struct rq *, struct task_struct *, int flags);
    struct task_struct *(*pick_next_task)(struct rq *);
    void (*put_prev_task)(struct rq *, struct task_struct *);
    void (*set_next_task)(struct rq *, struct task_struct *, bool first);
    int  (*balance)(struct rq *, struct task_struct *, struct rq_flags *);
    int  (*select_task_rq)(struct task_struct *, int cpu, int flags);
    void (*migrate_task_rq)(struct task_struct *, int new_cpu);
    void (*task_tick)(struct rq *, struct task_struct *, int queued);
    // ...
};
```

### 2.5 RT Run Queue (`struct rt_rq`) — `sched.h:834`

```c
struct rt_rq {
    struct rt_prio_array active;              // 優先級位圖陣列 (100 級)
    unsigned int         rt_nr_running;       // 執行中 RT 任務數
    struct { int curr; int next; } highest_prio;  // 最高優先級追蹤
    u64                  rt_time;             // 已消耗 RT 時間
    u64                  rt_runtime;          // RT 時間預算
    struct plist_head    pushable_tasks;      // 可推送到其他 CPU 的任務
};
```

### 2.6 Deadline Run Queue (`struct dl_rq`) — `sched.h:869`

```c
struct dl_rq {
    struct rb_root_cached root;               // 紅黑樹 (依截止期限排序)
    u64                   running_bw;         // 執行中頻寬
    u64                   this_bw;            // 已分配總頻寬
    struct { u64 curr; u64 next; } earliest_dl;  // 最早截止期限
    struct rb_root_cached pushable_dl_tasks_root; // 可推送 DL 任務
};
```

---

## 3. 排程類別

### 3.1 類別優先順序與用途

| 排程類別 | 政策 | 用途 | 選取演算法 |
|---------|------|------|-----------|
| `stop_sched_class` | 內部 | CPU hotplug、stop_machine | 唯一任務，絕對最高 |
| `dl_sched_class` | SCHED_DEADLINE | 硬即時週期性任務 | EDF (Earliest Deadline First) |
| `rt_sched_class` | SCHED_FIFO / SCHED_RR | 軟即時、低延遲任務 | 固定優先級 (100 級) |
| `fair_sched_class` | SCHED_NORMAL / SCHED_BATCH | 一般使用者任務 | EEVDF |
| `ext_sched_class` | SCHED_EXT | BPF 自訂排程策略 | 使用者定義 |
| `idle_sched_class` | SCHED_IDLE | CPU 空閒時執行 | 永遠最低 |

### 3.2 任務選取流程

`pick_next_task()` 在 `core.c` 中遍歷排程類別鏈結：

```
for_each_class(class) {
    p = class->pick_next_task(rq);
    if (p) return p;
}
```

只要高優先級類別有可執行任務，低優先級類別就不會被選到。CFS 任務僅在沒有 RT 和 DL 任務時才有機會執行。

---

## 4. EEVDF 演算法 — CFS 的核心

### 4.1 演算法概述

EEVDF (Earliest Eligible Virtual Deadline First) 是 CFS 目前採用的公平排程演算法，取代了早期純 vruntime 最小值的選取方式。它結合了**虛擬執行時間 (vruntime)** 的公平性概念與**虛擬截止期限 (deadline)** 的延遲控制。

### 4.2 核心公式

**虛擬執行時間更新** — `update_curr()` (fair.c:1226)：

```
vruntime += delta_exec × (NICE_0_LOAD / weight)
```

高權重 (低 nice 值) 的任務，vruntime 增長較慢，因此能獲得更多實際 CPU 時間。

**虛擬截止期限計算** — `update_deadline()`：

```
deadline = vruntime + (slice / weight)
```

slice 預設為排程週期的一部分。權重越大，deadline 越遠，表示可以執行更久。

**資格判定** — `entity_eligible()`：

```
eligible = (vruntime >= avg_vruntime)  // 簡化表示
```

任務的 vruntime 必須不超前於加權平均虛擬時間，才有資格被選取。

### 4.3 任務選取 — `__pick_eevdf()` (fair.c:951)

核心選取邏輯在紅黑樹上進行堆搜尋：在所有**合格 (eligible)** 任務中，選取**截止期限最早**者。這確保每個任務在其預期的時間片段內都能得到服務。

### 4.4 任務放置 — `place_entity()` (fair.c:5144)

新任務或被喚醒的任務需要被放置到適當的 vruntime 位置。EEVDF 採用**延遲保留 (lag preservation)** 策略：記住任務離開時的虛擬延遲 (vlag)，在重新加入時恢復，確保不會因為睡眠而獲得不公平的優勢或劣勢。

### 4.5 紅黑樹組織

`tasks_timeline` 是一棵 `rb_root_cached` 紅黑樹，具有快取的最左節點 (vruntime 最小)。插入和刪除為 O(log n)，而找到最左節點為 O(1)。EEVDF 選取需要遍歷樹進行堆搜尋，但實際上受限於樹的深度，效能良好。

### 4.6 群組排程 (Hierarchical Group Scheduling)

當啟用 `CONFIG_FAIR_GROUP_SCHED` 時，CFS 支援階層式排程：

```
root_cfs_rq
  ├── task_group_A (cfs_rq)
  │     ├── task_1 (sched_entity)
  │     └── task_2 (sched_entity)
  └── task_group_B (cfs_rq)
        └── task_3 (sched_entity)
```

每個 `task_group` 在每個 CPU 上有獨立的 `cfs_rq`。群組本身作為 `sched_entity` 參與上層排程，其權重是子任務的聚合。這確保群組間的公平性，同時群組內部也保持公平。

### 4.7 CFS 頻寬控制 (Bandwidth Throttling)

`CONFIG_CFS_BANDWIDTH` 允許限制 task_group 的 CPU 使用量。透過 `period` 和 `quota` 參數，當群組在一個週期內超過配額時，整個群組會被暫停 (throttled)，直到下一個週期開始。

---

## 5. 即時排程器

### 5.1 設計原理

RT 排程器 (`rt.c`) 實作 SCHED_FIFO 和 SCHED_RR 政策，使用 100 個優先級 (0-99) 的位圖陣列進行 O(1) 任務選取。同優先級的 FIFO 任務按先進先出順序執行；RR 任務在每個時間量子後輪替。

### 5.2 頻寬強制 (Bandwidth Enforcement)

RT 任務受 `rt_bandwidth` 機制約束，預設允許 RT 任務每 1 秒中最多使用 0.95 秒 CPU 時間，確保系統不會被 RT 任務完全佔據。頻寬執行透過 `hrtimer` (`sched_rt_period_timer()`) 週期性檢查。

### 5.3 多 CPU 任務遷移

RT 排程器實作推拉 (push/pull) 機制：當某 CPU 有多個 RT 任務時，`push_rt_tasks()` 將較低優先級者推送到其他 CPU；當 CPU 變空閒時，`pull_rt_task()` 從其他 CPU 拉取任務。`cpupri` 結構提供 O(1) 的最佳 CPU 查找。

---

## 6. 截止期限排程器

### 6.1 EDF 演算法

Deadline 排程器 (`deadline.c`) 實作 Earliest Deadline First 演算法，以紅黑樹維護任務，按絕對截止期限排序。每個 DL 任務有三個參數：`runtime` (每週期所需時間)、`deadline` (相對截止期限)、`period` (任務週期)。

### 6.2 Constant Bandwidth Server (CBS)

CBS 機制確保每個 DL 任務不會超過其分配的頻寬。在每個週期開始時，`replenish_dl_entity()` 會補充任務的 runtime。如果任務在週期內耗盡 runtime，會被延遲到下一個週期。

### 6.3 接納控制 (Admission Control)

透過 `sched_dl_overflow()` 實作接納控制：新的 DL 任務只有在系統有足夠頻寬時才會被接受。頻寬追蹤由 `struct dl_bw` 在 root domain 層級管理。

### 6.4 0-Lag Time

`task_non_contending()` 和 `task_contending()` 管理非競爭態轉換。當 DL 任務阻塞時，啟動 `inactive_task_timer()`，在 0-lag 時間後回收頻寬，避免短暫阻塞就釋放頻寬造成接納控制的問題。

### 6.5 CPU 選擇

`cpudl` 結構是一個最大堆 (max-heap)，記錄每個 CPU 上最晚的截止期限。選擇 CPU 時，`cpudl_find()` 找出截止期限最晚 (最有餘裕) 的 CPU 來放置新任務，支援非對稱容量系統。

---

## 7. sched_ext — BPF 可擴展排程

### 7.1 概述

`sched_ext` (`ext.c`) 是一個革命性的框架，允許透過 BPF 程式實作自訂排程策略，無需修改核心或重新編譯。它作為一個完整的排程類別存在，位於 fair 和 idle 之間。

### 7.2 核心介面 (`sched_ext_ops`)

BPF 排程器透過以下回呼函式實作策略：

| 回呼 | 用途 |
|------|------|
| `select_cpu()` | 為喚醒的任務選擇 CPU |
| `enqueue()` | 將任務排入 BPF 排程器 |
| `dequeue()` | 從 BPF 排程器移除任務 |
| `dispatch()` | 將任務分派到 local DSQ |
| `tick()` | 每次 timer tick 的處理 |
| `runnable()` / `running()` / `stopping()` | 任務狀態轉換通知 |
| `yield()` | 讓出 CPU 的處理 |

### 7.3 分派佇列 (Dispatch Queues)

SCX 使用分派佇列 (DSQ) 作為任務的中繼站。每個 CPU 有一個 local DSQ，也可建立自訂 DSQ。BPF 程式透過 `scx_bpf_dsq_insert()` 將任務放入指定的 DSQ。預設最大批次大小為 32。

### 7.4 安全機制

SCX 內建看門狗 (watchdog)，偵測失控的 BPF 排程器。如果 BPF 排程器出錯，系統會優雅地回退 (fallback) 到 FIFO 排程，確保系統穩定性。退出類型包含 `SCX_EXIT_DONE`、`SCX_EXIT_ERROR`、`SCX_EXIT_STALL` 等。

### 7.5 空閒 CPU 追蹤 (ext_idle.c)

SCX 框架提供內建的空閒 CPU 追蹤機制，支援 NUMA 和 LLC 感知。`scx_pick_idle_cpu()` 會依照 LLC → NUMA 節點 → 全局的順序搜尋空閒 CPU，最佳化快取局部性。

---

## 8. 負載追蹤 (PELT)

### 8.1 演算法原理

PELT (Per-Entity Load Tracking) 在 `pelt.c` 中實作，使用**指數衰減移動平均**追蹤每個排程實體的負載：

```
load_sum = Σ (contribution_i × y^i)
```

其中 `y^32 = 0.5` (約 32ms 半衰期)，`LOAD_AVG_PERIOD = 32`，最大累積值 `LOAD_AVG_MAX = 47742`。

### 8.2 追蹤指標

PELT 對每個實體追蹤三個獨立指標：

| 指標 | 含義 | 用途 |
|------|------|------|
| `load_avg` | 權重化負載 | 負載均衡決策 |
| `runnable_avg` | 可執行時間比例 | 偵測過載 |
| `util_avg` | 實際 CPU 使用率 | 頻率調節 (schedutil) |

### 8.3 階層聚合

PELT 不僅追蹤個別任務，還追蹤 CFS 佇列、RT 佇列、DL 佇列、IRQ 和硬體壓力的負載。群組排程時，子任務的負載會向上聚合到父群組。

### 8.4 PELT 時脈 (`clock_pelt`)

`update_rq_clock_pelt()` 會根據 CPU 容量和頻率縮放 PELT 時脈。當 CPU 以半頻運行時，PELT 時脈會相應減半，確保負載估計與實際能力一致。

### 8.5 使用率估計 (Util Estimation)

`UTIL_EST` 功能在 `features.h` 中啟用，透過記住任務上次的使用率，在任務剛被喚醒時提供更準確的使用率估計，避免啟動延遲。

---

## 9. 負載均衡

### 9.1 多層級均衡架構

負載均衡 (`fair.c` 中 11000+ 行) 在排程域 (scheduling domain) 的階層結構上運作：

```
NUMA 節點層    ← 跨節點均衡 (最高成本)
  ↓
CPU 封裝層     ← 跨實體 CPU 均衡
  ↓
核心叢集層     ← 跨核心叢集均衡 (如 big.LITTLE)
  ↓
SMT 層         ← 同核心的超執行緒均衡 (最低成本)
```

### 9.2 均衡決策

`sched_balance_find_src_group()` (fair.c:11440) 分析各群組的負載狀態，以 `group_type` 分類：

| 群組類型 | 含義 | 動作 |
|---------|------|------|
| `group_has_spare` | 有閒置容量 | 可接受遷移 |
| `group_fully_busy` | 滿載但無過載 | 維持現狀 |
| `group_misfit_task` | 任務不適合當前 CPU 容量 | 遷移到大核心 |
| `group_asym_packing` | 非對稱封裝最佳化 | 優先使用高效能核心 |
| `group_overloaded` | 過載 | 必須遷移 |

### 9.3 均衡時機

均衡在以下情境觸發：`scheduler_tick()` 中的週期性檢查 (`rebalance_domains()`)、CPU 進入閒置時 (`sched_balance_newidle()`)、NO_HZ 模式下的閒置負載均衡器 (ILB)。

### 9.4 任務遷移

`load_balance()` 識別最繁忙的 CPU 後，透過 `detach_tasks()` 和 `attach_tasks()` 遷移任務。遷移會更新 PELT 負載追蹤，確保統計數據的一致性。特殊情況下使用 CPU stop machine 進行強制遷移。

### 9.5 Misfit 任務處理

在大小核 (big.LITTLE) 架構中，`update_misfit_status()` 偵測在小核心上執行但需要更多容量的任務，觸發遷移到大核心。這對 Android 裝置的效能至關重要。

---

## 10. 排程域與拓撲

### 10.1 排程域 (`struct sched_domain`)

排程域在 `topology.c` 中建構，每個 CPU 有一個多層排程域鏈：

```c
struct sched_domain {
    struct sched_domain  *parent;     // 上層域
    struct sched_group   *groups;     // 域內的 CPU 群組
    unsigned long        span[];      // 域涵蓋的 CPU 遮罩
    // 旗標控制均衡行為
    unsigned int         flags;       // SD_LOAD_BALANCE, SD_BALANCE_WAKE, ...
};
```

### 10.2 效能域 (Performance Domain)

`struct perf_domain` 整合能源模型 (Energy Model)，為**能源感知排程 (EAS)** 提供基礎。EAS 在選擇 CPU 時考慮功耗效率，在滿足效能需求的前提下選擇最節能的核心。

### 10.3 EAS 可行性

`sched_is_eas_possible()` 檢查是否滿足 EAS 的前提條件：系統必須有非對稱 CPU 容量、完整的能源模型、以及在閾值以下的 CPU 數量。

### 10.4 Root Domain

`struct root_domain` 是負載均衡的頂層結構，管理全域 CPU 遮罩、overload 狀態追蹤，以及 `cpupri` 和 `cpudl` 結構。

---

## 11. CPU 頻率調節 (schedutil)

### 11.1 架構

`cpufreq_schedutil.c` 實作排程器驅動的 CPU 頻率調節器 (governor)。它直接從排程器上下文中讀取 PELT 使用率，計算目標頻率：

```
target_freq = capacity_ref_freq × (util / max_capacity) × 1.25
```

1.25 的倍數確保在高使用率時有頻率餘裕。

### 11.2 更新路徑

`sugov_update_single()` (單政策 CPU) 和 `sugov_update_shared()` (共享政策) 被排程器在任務喚醒和 tick 時呼叫。支援兩種模式：快速切換 (在排程器上下文中直接切換) 和慢速切換 (透過 kthread 延遲切換)。

### 11.3 IO 等待提升

當偵測到任務剛從 IO 等待返回時，`sugov_iowait_boost()` 會暫時提升 CPU 頻率，以加速 IO 完成後的處理。提升幅度會指數增長，直到達到最大頻率。

### 11.4 速率限制

`sugov_should_update_freq()` 確保頻率更新不會過於頻繁，透過 `rate_limit_us` 參數控制最小更新間隔，避免頻率抖動。

---

## 12. 核心排程 (Core Scheduling)

### 12.1 目的

`core_sched.c` 實作核心排程 (Core Scheduling)，確保 SMT 兄弟核心上同時運行的任務屬於相同的安全群組。這主要用於防禦 Spectre 等側通道攻擊。

### 12.2 Cookie 機制

使用引用計數的 `sched_core_cookie` 標識群組。同一 cookie 的任務可以在 SMT 兄弟上同時執行。不同 cookie 的任務不能同時運行，系統會強制其中一個兄弟核心閒置 (forced idle)。

### 12.3 介面

透過 `prctl(PR_SCHED_CORE, ...)` 系統呼叫設定任務的 core scheduling cookie。`sched_core_share_pid()` 支援在程序間分享或建立 cookie。

---

## 13. 壓力阻塞資訊 (PSI)

### 13.1 概述

PSI (Pressure Stall Information) 在 `psi.c` 中實作，追蹤系統資源壓力。它回答的問題是：「有多少工作因為缺少 CPU / 記憶體 / IO 資源而被延遲？」

### 13.2 壓力等級

| 等級 | 含義 |
|------|------|
| SOME | 至少有一個任務在等待該資源 |
| FULL | 所有任務都在等待，CPU 實際空閒 (浪費) |

### 13.3 追蹤資源

PSI 追蹤三種資源：CPU (任務就緒但等待 CPU)、Memory (任務因記憶體回收而阻塞)、IO (任務等待 IO 完成)。

### 13.4 平均值與觸發器

提供 10 秒、60 秒、300 秒三個時間窗口的指數移動平均值，透過 `/proc/pressure/{cpu,memory,io}` 匯出到使用者空間。支援即時輪詢觸發器，允許使用者空間在壓力超過閾值時收到通知。

---

## 14. Android Vendor Hooks

### 14.1 機制

Android Common Kernel 透過 `vendor_hooks.c` 匯出 **78 個追蹤點 (tracepoint)**，允許廠商模組在不修改核心排程器原始碼的情況下注入自訂邏輯。

### 14.2 Hook 分佈

| 領域 | 主要 Hook | 數量 |
|------|----------|------|
| 任務選擇與放置 | `select_task_rq_fair`, `select_task_rq_rt` | ~10 |
| 入隊/出隊 | `enqueue_task`, `dequeue_task`, `enqueue_entity` | ~8 |
| 負載追蹤 | `update_load_avg_*`, `update_rt_rq_load_avg` | ~6 |
| 任務遷移 | `migrate_queued_task`, `can_migrate_task` | ~5 |
| 負載均衡 | `sched_balance_*`, `find_busiest_queue` | ~8 |
| 拓撲 | `build_sched_domains`, `update_topology_flags` | ~4 |
| 頻率 | `scheduler_tick` (governor 互動) | ~3 |
| 喚醒 | `try_to_wake_up`, `sched_fork` | ~6 |
| 搶佔 | `check_preempt_wakeup_fair` | ~2 |
| Misfit | `update_misfit_status` | ~2 |
| 容量 | `update_cpu_capacity` | ~2 |

### 14.3 典型用途

Android 廠商利用這些 hook 實作：大小核 (big.LITTLE / DynamIQ) 排程策略最佳化、前景/背景應用的差異化排程、任務提升 (task boost) 與熱管理整合、自訂的 CPU 選擇邏輯、以及遊戲模式等效能模式。

### 14.4 Hook 型別

`android_rvh_*` (Restricted Vendor Hook) 是最常見的型別，只允許 GPL 相容的模組掛載。這些 hook 透過 `EXPORT_TRACEPOINT_SYMBOL_GPL()` 匯出。

---

## 15. Feature Flags 與執行期調校

### 15.1 Feature 定義 (`features.h`)

排程器 feature flags 可透過 debugfs 在執行期切換，使用 `static_key` (jump label) 實作零成本的條件檢查：

**EEVDF 相關：**

| Feature | 預設 | 說明 |
|---------|------|------|
| `PLACE_LAG` | ON | 跨睡眠/喚醒保留 vlag |
| `PLACE_DEADLINE_INITIAL` | ON | 新任務給予半個 slice 的初始截止期限 |
| `PLACE_REL_DEADLINE` | ON | 遷移時保留相對虛擬截止期限 |
| `RUN_TO_PARITY` | ON | 抑制喚醒搶佔直到 vlag 歸零 |
| `PREEMPT_SHORT` | ON | 允許短 slice 任務搶佔 |

**任務放置與快取：**

| Feature | 預設 | 說明 |
|---------|------|------|
| `NEXT_BUDDY` | OFF | 排程最後喚醒的任務 (快取熱度) |
| `CACHE_HOT_BUDDY` | ON | 將 buddy 視為快取熱任務 |

**延遲出隊：**

| Feature | 預設 | 說明 |
|---------|------|------|
| `DELAY_DEQUEUE` | ON | 延遲出隊以消耗負延遲 |
| `DELAY_ZERO` | ON | 在出隊/喚醒時將延遲截斷為 0 |

**負載均衡：**

| Feature | 預設 | 說明 |
|---------|------|------|
| `SIS_UTIL` | ON | 限制 LLC 域掃描範圍 |
| `UTIL_EST` | ON | 使用估計使用率 |
| `TTWU_QUEUE` | ON | 透過 IPI 排隊遠端喚醒 |
| `WA_IDLE` / `WA_WEIGHT` / `WA_BIAS` | ON | 喚醒親和性調校 |

### 15.2 執行期介面

```
/sys/kernel/debug/sched/features    # 讀寫 feature flags
/sys/kernel/debug/sched/scaling     # 調整縮放模式
/sys/kernel/debug/sched/verbose     # 詳細除錯
```

---

## 16. 建置系統與編譯架構

### 16.1 編譯單元組織

排程器使用兩個聚合編譯單元來平衡建置平行度和標頭檔成本：

**`build_policy.c`** — 策略排程程式碼：

包含 `idle.c`, `rt.c`, `cpudeadline.c`, `pelt.c`, `cputime.c`, `deadline.c`, 以及條件性的 `ext.c`, `ext_idle.c`, `syscalls.c`。

**`build_utility.c`** — 工具程式碼：

包含 `clock.c`, `cpuacct.c`, `cpufreq.c`, `cpufreq_schedutil.c`, `debug.c`, `stats.c`, `loadavg.c`, `completion.c`, `swait.c`, `wait_bit.c`, `wait.c`, `cpupri.c`, `stop_task.c`, `topology.c`，以及條件性的 `core_sched.c`, `psi.c`, `membarrier.c`, `isolation.c`, `autogroup.c`。

### 16.2 條件編譯選項

| CONFIG 選項 | 功能 |
|------------|------|
| `CONFIG_FAIR_GROUP_SCHED` | 階層式 CFS 群組排程 |
| `CONFIG_CFS_BANDWIDTH` | CFS 頻寬限制 |
| `CONFIG_RT_GROUP_SCHED` | 階層式 RT 群組排程 |
| `CONFIG_SCHED_CLASS_EXT` | BPF 可擴展排程類別 |
| `CONFIG_NUMA_BALANCING` | NUMA 感知排程 |
| `CONFIG_SCHED_SMT` | SMT 最佳化 |
| `CONFIG_UCLAMP_TASK` | 使用率鉗位 |
| `CONFIG_SCHED_CORE` | 核心排程 (SMT 安全) |
| `CONFIG_PSI` | 壓力阻塞資訊 |
| `CONFIG_SCHEDSTATS` | 排程統計 |
| `CONFIG_SCHED_AUTOGROUP` | 自動群組排程 |
| `CONFIG_ANDROID_VENDOR_HOOKS` | Android 廠商 hook |

### 16.3 Makefile 特殊處理

Makefile 對排程器程式碼禁用 KCOV (避免非系統呼叫路徑的覆蓋率雜訊) 和 KCSAN (啟用 barrier 檢測但避免誤報)。某些架構 (x86, m68k) 需要 frame pointer 以支援 `__switch_to()` 的正確堆疊追蹤。

---

## 17. Syscall 介面

### 17.1 排程相關系統呼叫

| Syscall | 功能 |
|---------|------|
| `sched_setscheduler()` | 設定排程策略和參數 |
| `sched_getscheduler()` | 查詢排程策略 |
| `sched_setparam()` | 設定排程參數 |
| `sched_getparam()` | 查詢排程參數 |
| `sched_setattr()` | 設定擴展排程屬性 (含 deadline 參數) |
| `sched_getattr()` | 查詢擴展排程屬性 |
| `sched_get_priority_max()` | 查詢策略的最大優先級 |
| `sched_get_priority_min()` | 查詢策略的最小優先級 |
| `sched_yield()` | 主動讓出 CPU |

### 17.2 Nice 值與優先級

`set_user_nice()` 在 `syscalls.c` 中實作。Nice 值 (-20 到 +19) 透過 `PRIO_TO_WEIGHT[]` 表轉換為排程權重。Nice 0 對應權重 1024，每增加 1 個 nice 值約減少 10% CPU 時間。

### 17.3 uclamp (Utilization Clamping)

`sched_setattr()` 支援設定 `sched_util_min` 和 `sched_util_max`，允許使用者空間限制任務的使用率範圍，間接影響 CPU 頻率選擇。

---

## 18. 統計與除錯

### 18.1 排程統計 (`stats.c` / `stats.h`)

當啟用 `CONFIG_SCHEDSTATS` 時，提供豐富的統計資訊：

透過 `/proc/schedstat` (版本 17) 匯出每個 CPU 的統計，包含 yield 計數、排程計數、喚醒計數、執行延遲。每個排程域也有負載均衡統計，包含均衡嘗試次數、不平衡度、遷移計數等。

每個任務也追蹤等待時間、睡眠時間、IO 等待時間、阻塞時間、執行時間等統計。

### 18.2 Debug 介面 (`debug.c`)

`/sys/kernel/debug/sched/` 目錄提供 feature flag 控制、排程器調校旋鈕、以及各種排程器狀態的查看。

### 18.3 Load Average (`loadavg.c`)

`/proc/loadavg` 中的 1 分鐘、5 分鐘、15 分鐘負載平均值在 `loadavg.c` 中計算。使用指數加權移動平均，每 `LOAD_FREQ` (5 秒) 更新一次。NO_HZ 感知的雙緩衝機制確保在有大量閒置 CPU 時仍然準確。

---

## 19. 輔助機制

### 19.1 排程器時脈 (`clock.c`)

提供 `sched_clock()` (架構相關) 和 `local_clock()` / `cpu_clock()` (per-CPU)。處理 TSC 不穩定的情況，使用單調性過濾器防止時脈回退。

### 19.2 等待佇列 (`wait.c` / `swait.c`)

通用等待佇列 (`wait_queue_head`) 支援獨佔喚醒和優先級排序。簡單等待佇列 (`swait_queue_head`) 使用 raw spinlock，適用於 RT-safe 的場景，如 completion 機制。

### 19.3 Completion (`completion.c`)

封裝簡單等待佇列，提供一次性事件同步。`complete()` 發出完成信號，`wait_for_completion()` 等待完成，支援超時和可中斷變體。

### 19.4 Membarrier (`membarrier.c`)

`membarrier()` 系統呼叫提供跨 CPU 的記憶體順序保證。使用 IPI 機制確保特定程序的所有執行緒看到一致的記憶體狀態，支援核心同步和 rseq 同步模式。

### 19.5 CPU 隔離 (`isolation.c`)

透過 `nohz_full=` 和 `isolcpus=` 開機參數，將指定 CPU 從負載均衡、中斷管理、核心噪音 (timer/kthread) 中隔離，專用於延遲敏感的工作負載。

### 19.6 Autogroup (`autogroup.c`)

自動群組排程將同一終端會話的程序歸為一組，確保群組間的公平性。當使用者同時編譯核心和看影片時，編譯器的數十個程序不會壓倒影片播放器。透過 `sysctl_sched_autogroup_enabled` 控制啟停。

### 19.7 CPU 記帳 (`cpuacct.c`)

Cgroup CPU 記帳子系統，透過 `/sys/fs/cgroup/cpuacct/` 提供每個 cgroup 的 CPU 使用量分解 (user/system/total)，支援 per-CPU 層級的精細統計。

---

## 20. 檔案索引

| 檔案 | 行數 | 說明 |
|------|------|------|
| `core.c` | 10,986 | 排程器核心：context switch, `__schedule()`, `try_to_wake_up()` |
| `fair.c` | 14,102 | CFS/EEVDF 排程器與負載均衡 |
| `rt.c` | ~2,500 | RT 排程器 (FIFO/RR) |
| `deadline.c` | ~2,800 | Deadline 排程器 (EDF/CBS) |
| `ext.c` | ~4,500 | sched_ext (BPF 排程框架) |
| `ext_idle.c` | ~400 | sched_ext 空閒 CPU 追蹤 |
| `sched.h` | 4,033 | 核心資料結構與宣告 |
| `pelt.c` | ~500 | PELT 負載追蹤演算法 |
| `pelt.h` | ~300 | PELT 介面與 clock scaling |
| `topology.c` | ~2,800 | 排程域建構與 EAS |
| `cpufreq_schedutil.c` | ~900 | schedutil 頻率調節器 |
| `core_sched.c` | ~300 | 核心排程 (SMT 安全) |
| `psi.c` | ~1,500 | 壓力阻塞資訊 |
| `vendor_hooks.c` | ~200 | Android vendor hook 匯出 |
| `features.h` | ~100 | Feature flags 定義 |
| `loadavg.c` | ~400 | /proc/loadavg 計算 |
| `cpudeadline.c` | ~250 | DL 任務 CPU 選擇 (max-heap) |
| `cpupri.c` | ~300 | RT 任務 CPU 選擇 (priority bitmap) |
| `stats.c` | ~200 | /proc/schedstat 匯出 |
| `stats.h` | 344 | 排程統計巨集與追蹤點 |
| `debug.c` | ~600 | Debugfs 介面 |
| `clock.c` | ~500 | 排程器時脈 (TSC 處理) |
| `idle.c` | ~500 | 閒置排程類別與 cpuidle 整合 |
| `stop_task.c` | ~150 | Stop 排程類別 |
| `completion.c` | ~300 | Completion 同步機制 |
| `wait.c` | ~300 | 等待佇列 |
| `swait.c` | ~146 | 簡單等待佇列 |
| `wait_bit.c` | ~200 | 位元等待佇列 |
| `membarrier.c` | ~300 | membarrier() 系統呼叫 |
| `isolation.c` | ~277 | CPU 隔離 |
| `autogroup.c` | ~294 | 自動群組排程 |
| `cpuacct.c` | ~300 | Cgroup CPU 記帳 |
| `cputime.c` | ~300 | CPU 時間追蹤 |
| `syscalls.c` | ~300 | 排程相關系統呼叫 |
| `build_policy.c` | ~30 | 策略編譯單元聚合 |
| `build_utility.c` | ~50 | 工具編譯單元聚合 |
| `Makefile` | ~30 | 建置規則 |

---

## 21. 交叉參考

- [`task_struct`](../data-structures/task_struct.md) — scheduler 最重要的任務描述符，包含 policy、priority、CPU affinity、vendor data 等欄位
- [鎖定原語](../concepts/locking-primitives.md) — run queue lock、RT mutex、RCU 與 lockdep 背景
- [Vendor Hooks](../concepts/vendor-hooks.md) — Android 排程器 hooks 的宣告/派發機制
- [Vendor Hooks 跨子系統分佈分析](../analyses/vendor-hooks-distribution.md) — 排程器 78 個 hooks 佔 ACK vendor hooks 約六成的策略含義
- [跨子系統鎖定模式分析](../analyses/locking-patterns.md) — per-CPU run queue + RCU 的鎖定模式比較
- [BPF](../concepts/bpf.md) — sched_ext 與 BPF 可擴展排程的基礎
- [GKI](../concepts/gki.md) — GKI defconfig 與 Android kernel module 邊界
- [Memory Management](memory-management.md) — PSI、memory pressure 與 scheduler/cgroup 的交互

> 本文件由自動分析產生，基於 `kernel/sched/` 目錄下全部原始碼的逐檔解讀。
