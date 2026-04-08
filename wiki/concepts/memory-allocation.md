---
type: concept
scope: kernel-wide
related:
  - ../concepts/locking-primitives.md
  - ../concepts/rcu.md
last_updated: 2026-04-08
---

# 記憶體分配

## 概述

Linux 核心提供多層次的記憶體分配器，從底層的頁面分配器（Buddy System）到高層的 slab 分配器（SLUB），以及用於虛擬連續記憶體的 vmalloc 和用於 DMA 裝置的 CMA。每個分配器針對不同的使用場景進行了優化。

## 機制

### 頁面分配器與 Buddy 系統

**實現**：`mm/page_alloc.c`（7727行）

Buddy 系統將記憶體組織為 2^0、2^1、...、2^N 頁大小的塊。相鄰的同大小空閒塊可合併為更大的塊，分配時較大的塊可拆分為較小的塊。

**Per-CPU 頁面快取（PCP）**（`page_alloc.c:94-202`）：

```c
struct per_cpu_pages {
    struct list_head lists[];  // 本地頁面快取
    // 當頁面數超過 high water mark → 批量釋放到 buddy 系統
};
```

加速單頁分配/釋放，避免全域鎖爭用。使用 `pcp_spin_trylock()` / `pcp_spin_unlock()` 保護。

**區域（Zone）**：`ZONE_DMA`、`ZONE_NORMAL`、`ZONE_HIGHMEM`、`ZONE_MOVABLE`。

### GFP 標誌

**定義**：`include/linux/gfp.h`

GFP（Get Free Pages）標誌控制分配行為：

| 分類 | 標誌 | 意義 |
|------|------|------|
| **常用組合** | `GFP_KERNEL` | 可阻塞、可回收，最常用 |
| | `GFP_ATOMIC` | 不可阻塞（中斷上下文）|
| | `GFP_NOWAIT` | 不等待 |
| **行為** | `__GFP_DIRECT_RECLAIM` | 允許直接記憶體回收 |
| | `__GFP_KSWAPD_RECLAIM` | 喚醒 kswapd |
| | `__GFP_FS` | 允許檔案系統操作 |
| | `__GFP_IO` | 允許 I/O 操作 |
| **區域** | `__GFP_DMA` / `__GFP_DMA32` | DMA 區域 |
| | `__GFP_MOVABLE` | 可移動頁面 |
| **其他** | `__GFP_ZERO` | 清零記憶體 |
| | `__GFP_ACCOUNT` | memcg 計算 |

**快速查表**（`gfp.h:124-133`）：`GFP_ZONE_TABLE` 將 GFP 標誌快速對應到區域。

**檢查函數**（`gfp.h:38-60`）：
- `gfpflags_allow_blocking()` — 是否允許阻塞
- `gfpflags_allow_spinning()` — 是否允許自旋鎖

### SLUB 分配器

**實現**：`mm/slub.c`（10143行）

SLUB 是 Linux 的預設 slab 分配器，為固定大小的核心物件（`kmalloc`）提供高效能快取。

**架構**：

```
kmem_cache
├── cpu_slab (per-CPU)     → 快速路徑：從凍結 slab 分配（無需加鎖）
├── node->partial           → 部分使用的 slab 列表
└── node->full              → 完全使用的 slab 列表
```

**鎖定策略**（`slub.c:54-150`）：

1. `slab_mutex` — 全域互斥鎖（保護快取創建/銷毀）
2. `node->list_lock` — 自旋鎖（保護節點級別的 slab 列表）
3. `kmem_cache->cpu_slab->lock` — 局部鎖（保護 per-CPU 狀態）

**kmalloc API**（`include/linux/slab.h`）：

| 函數 | 用途 |
|------|------|
| `kmalloc(size, flags)` | 分配核心記憶體 |
| `kzalloc(size, flags)` | 分配並清零 |
| `kmalloc_array(n, size, flags)` | 陣列分配（溢位檢查）|
| `kcalloc(n, size, flags)` | 陣列分配並清零 |
| `kfree(ptr)` | 釋放 |

**Slab 標誌**（`slab.h:25-64`）：

- `SLAB_HWCACHE_ALIGN` — 按快取行對齊
- `SLAB_TYPESAFE_BY_RCU` — 使用 RCU 延遲頁面釋放
- `SLAB_POISON`、`SLAB_RED_ZONE` — 除錯特性
- `SLAB_NO_MERGE` — 防止與其他快取合併

### vmalloc

**實現**：`mm/vmalloc.c`

用於分配**虛擬連續**但**物理不連續**的記憶體，適合大塊分配（> PAGE_SIZE）。

核心機制（`vmalloc.c:94-150`）：
- 使用核心頁表映射不連續的物理頁面
- `vmap_pte_range()` 設定 PTE 級別的頁表項
- `vmap_try_huge_pmd()` 巨大頁優化（`CONFIG_HAVE_ARCH_HUGE_VMALLOC`）
- 每個 CPU 有 `vfree_deferred` 工作隊列用於延遲釋放

### CMA（連續記憶體分配器）

**實現**：`mm/cma.c`

用於 DMA 裝置需要的物理連續記憶體（常用於視訊/多媒體）。

**結構**（`cma.c:59-98`）：

```c
struct cma {
    unsigned long base_pfn;     // 起始物理頁框號
    unsigned long count;        // 區域頁面總數
    unsigned long *bitmap;      // 追蹤已分配頁面
    struct mutex lock;
};
```

CMA 區域在啟動時保留，平時可被可移動頁面使用，需要時透過遷移回收。

### DMA 池

**實現**：`mm/dmapool.c`

用於小型 DMA 記憶體塊（< 1 頁）：

```c
struct dma_pool {
    struct list_head page_list;   // 已分配的頁面
    size_t allocation;            // 塊大小
    spinlock_t lock;              // 單一自旋鎖保護
};
```

### Per-CPU 分配器

**實現**：`mm/percpu.c`

- 使用 `DEFINE_PER_CPU()` 巨集聲明 per-CPU 變數
- 透過 `this_cpu_ptr()` 存取當前 CPU 的副本
- 避免快取行競爭，提升擴展性

## 使用模式

### 分配器選擇指南

| 需求 | 分配器 | API |
|------|--------|-----|
| 小型核心物件（< PAGE_SIZE）| SLUB | `kmalloc()` / `kzalloc()` |
| 大塊虛擬連續記憶體 | vmalloc | `vmalloc()` / `vfree()` |
| 物理頁面 | Buddy | `alloc_pages()` / `__get_free_pages()` |
| DMA 連續記憶體 | CMA | `cma_alloc()` / `dma_alloc_coherent()` |
| 小型 DMA 塊 | DMA 池 | `dma_pool_alloc()` |
| Per-CPU 資料 | percpu | `alloc_percpu()` |

## Android 相關性

**Vendor Hook**（`include/trace/hooks/mm.h`）：定義了 3 個受限制 hook，但目前已被註釋掉：

- `android_rvh_set_skip_swapcache_flags`
- `android_rvh_set_gfp_zone_flags`
- `android_rvh_set_readahead_gfp_mask`

**Vmscan Hook**（`include/trace/hooks/vmscan.h`）：1 個啟用的受限制 hook，用於控制匿名/檔案頁回收平衡。

GKI defconfig 中的記憶體相關配置：

- `CONFIG_TRANSPARENT_HUGEPAGE=y`（124行）
- `CONFIG_CMA=y`（127行）
- `CONFIG_LRU_GEN=y`（134行）— LRU 生成追蹤

## 交叉參考

- [鎖定原語](locking-primitives.md) — SLUB 的鎖定策略
- [RCU](rcu.md) — `SLAB_TYPESAFE_BY_RCU` 機制
