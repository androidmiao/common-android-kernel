---
type: data-structure
defined_in: common/include/linux/mm_types.h:79
size_approx: 64 bytes (on 64-bit systems)
lifecycle: allocated by alloc_pages() / __folio_alloc_noprof(), freed by free_pages() / folio_put()
related: [folio, address_space, page_flags, page_ref, page_alloc, lru_list, slab, buddy_system]
last_updated: 2026-04-09
---

# `struct page` / `struct folio`

## Purpose

`struct page` 是 Linux 内核中代表物理内存中单个基础页面的核心数据结构。系统中的每一个物理页都有一个关联的 `struct page`,用于跟踪该页的使用状态、引用计数、映射关系以及其他元数据。该结构是内存管理的基础,被页分配器、页缓存、直接内存访问(DMA)、SLUB 分配器和许多其他内核子系统使用。

`struct folio` 是一个较新的抽象,它代表了一个逻辑上连续的字节集合(可由多个物理页组成),大小为 2 的幂次。一个 folio 在物理、虚拟和逻辑上都是连续的,大小至少为 PAGE_SIZE。folio 是内核向统一的大页面和小页面处理方向演进的一部分,逐步替代基于页的 API。

## Definition

### struct page

```c
struct page {
    memdesc_flags_t flags;           // 原子标志位,可异步更新(page-flags.h)
    union {
        struct {                     // 页缓存和匿名页面
            union {
                struct list_head lru;
                struct list_head buddy_list;
                struct list_head pcp_list;
                struct llist_node pcp_llist;
            };
            struct address_space *mapping;
            union {
                pgoff_t __folio_index;   // 映射内的偏移量
                unsigned long share;     // fsdax 的共享计数
            };
            unsigned long private;   // 映射私有数据或缓冲区头
        };
        struct {                     // page_pool (网络栈)
            unsigned long pp_magic;
            struct page_pool *pp;
            unsigned long _pp_mapping_pad;
            unsigned long dma_addr;
            atomic_long_t pp_ref_count;
        };
        struct {                     // 复合页尾页
            unsigned long compound_head;
        };
        struct {                     // ZONE_DEVICE 页
            void *_unused_pgmap_compound_head;
            void *zone_device_data;
        };
        struct rcu_head rcu_head;
    };

    union {                          // 页类型或映射计数 (4 字节)
        unsigned int page_type;
        atomic_t _mapcount;
    };

    atomic_t _refcount;              // 引用计数(绝不直接使用!)

#ifdef CONFIG_MEMCG
    unsigned long memcg_data;
#elif defined(CONFIG_SLAB_OBJ_EXT)
    unsigned long _unused_slab_obj_exts;
#endif

#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;                   // 内核虚拟地址(highmem)
#endif

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
    int _last_cpupid;
#endif

#ifdef CONFIG_KMSAN
    struct page *kmsan_shadow;
    struct page *kmsan_origin;
#endif
} _struct_page_alignment;
```

### struct folio

```c
struct folio {
    union {
        struct {
            memdesc_flags_t flags;
            union {
                struct list_head lru;
                struct {
                    void *__filler;
                    unsigned int mlock_count;
                };
                struct dev_pagemap *pgmap;
            };
            struct address_space *mapping;
            union {
                pgoff_t index;
                unsigned long share;
            };
            union {
                void *private;
                swp_entry_t swap;
            };
            atomic_t _mapcount;
            atomic_t _refcount;
#ifdef CONFIG_MEMCG
            unsigned long memcg_data;
#endif
#if defined(WANT_PAGE_VIRTUAL)
            void *virtual;
#endif
#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
            int _last_cpupid;
#endif
        };
        struct page page;
    };

    union {
        struct {
            unsigned long _flags_1;
            unsigned long _head_1;
            union {
                struct {
                    atomic_t _large_mapcount;
                    atomic_t _nr_pages_mapped;
#ifdef CONFIG_64BIT
                    atomic_t _entire_mapcount;
                    atomic_t _pincount;
#endif
                    mm_id_mapcount_t _mm_id_mapcount[2];
                    union {
                        mm_id_t _mm_id[2];
                        unsigned long _mm_ids;
                    };
                };
                unsigned long _usable_1[4];
            };
            atomic_t _mapcount_1;
            atomic_t _refcount_1;
#ifdef NR_PAGES_IN_LARGE_FOLIO
            unsigned int _nr_pages;
#endif
        };
        struct page __page_1;
    };

    union {
        struct {
            unsigned long _flags_2;
            unsigned long _head_2;
            struct list_head _deferred_list;
#ifndef CONFIG_64BIT
            atomic_t _entire_mapcount;
            atomic_t _pincount;
#endif
        };
        struct page __page_2;
    };

    union {
        struct {
            unsigned long _flags_3;
            unsigned long _head_3;
            void *_hugetlb_subpool;
            void *_hugetlb_cgroup;
            void *_hugetlb_cgroup_rsvd;
            void *_hugetlb_hwpoison;
        };
        struct page __page_3;
    };
};
```

## Field Groups

### struct page 字段分组

#### 标志位和状态 (Flags and Status)
- **flags**: 原子标志位集合,见 page-flags.h,表示页的各种状态(脏、锁定、可回收等)

#### 链表管理 (List Management)
- **lru**: 页面回收列表,受 lruvec->lru_lock 保护,也可被页所有者用作通用列表
- **buddy_list**: 伙伴分配器中的空闲页链表
- **pcp_list/pcp_llist**: 每个 CPU 缓存链表

#### 映射与缓存 (Mapping and Caching)
- **mapping**: 指向 address_space(页缓存)或 anon_vma(匿名内存)
- **__folio_index**: 页在映射中的偏移量(以页为单位)
- **share**: DAX 映射的共享计数

#### 私有数据 (Private Data)
- **private**: 映射私有数据(缓冲区头、SwapEntry 等),含义取决于标志位

#### 复合页支持 (Compound Page Support)
- **page_type**: 类型化 folio 的页类型,或映射计数的映射容量

#### 引用计数 (Reference Counting)
- **_mapcount**: 页在页表中被直接引用的次数(初始化为 -1)
- **_refcount**: 页的总引用计数(见 page_ref.h),绝不直接使用

#### 内存控制组与对象扩展 (Memory Cgroup and Object Extensions)
- **memcg_data**: 关联的 memory cgroup 数据
- **_unused_slab_obj_exts**: SLAB 对象扩展占位符

#### 其他 (Others)
- **virtual**: 内核直接映射中的虚拟地址(highmem 架构)
- **_last_cpupid**: 最后访问该页的 CPU 和进程ID(NUMA 相关)
- **kmsan_shadow/kmsan_origin**: KMSAN 检测器元数据(仅 CONFIG_KMSAN)

### struct folio 额外字段分组

#### 大页面映射跟踪 (Large Page Mapping Tracking)
- **_large_mapcount**: 大 folio 中所有页的映射计数之和
- **_nr_pages_mapped**: 已映射页数
- **_entire_mapcount**: 整个 folio 作为整体映射的次数(CONFIG_64BIT)
- **_pincount**: DMA pin 计数(CONFIG_64BIT)

#### 反向映射优化 (Reverse Mapping Optimization)
- **_mm_id[2]**: 最多两个虚拟内存空间ID(RMAP 优化)
- **_mm_ids**: 作为单一无符号长整形访问 MM ID
- **_mm_id_mapcount[2]**: 各虚拟内存空间的映射计数

#### 大页面特定 (Large Page Specific)
- **_deferred_list**: 在内存压力下待分割的 folio
- **_nr_pages**: folio 中的页数
- **_hugetlb_subpool**: HugeTLB 子池指针
- **_hugetlb_cgroup**: HugeTLB cgroup 指针
- **_hugetlb_cgroup_rsvd**: HugeTLB cgroup 保留计数
- **_hugetlb_hwpoison**: 硬件中毒页列表头

## Lifecycle

### 分配 (Allocation)

#### struct page 的初始化
1. **启动时初始化**:
   - 在 `init_section_delay_populated()` 中,所有物理页都获得 `struct page` 元数据
   - 初始引用计数设置为 1(见 page_ref.h:115 的 `init_page_count()`)

2. **页分配器分配**:
   - **函数**: `__alloc_pages_noprof()`(mm/page_alloc.c)
   - 从伙伴分配器获取空闲页块
   - 调用 `post_alloc_hook()` 进行初始化
   - 引用计数初始化为 1

3. **Folio 分配**:
   - **函数**: `__folio_alloc_noprof()`(mm/page_alloc.c:5281)
   - 分配 2^order 个连续页
   - 初始化 folio 特定字段(_nr_pages、_large_mapcount 等)
   - 返回指向头页的 folio

#### Folio 初始化详情
- 所有尾页的 _mapcount 和 _refcount 被标记为 -1(表示其为复合页尾页)
- _entire_mapcount 初始化为 -1
- _nr_pages_mapped 初始化为 0
- 锁标志(MM_IDS_SHARED_BIT 等)初始化为 0

### 释放 (Deallocation)

#### struct page 释放流程
1. **引用计数递减**:
   - `put_page()` 或 `folio_put()` 递减 _refcount
   - 当计数达到 0 时触发释放

2. **清理**:
   - `free_pages_prepare()`(mm/page_alloc.c:1343) 检查:
     - 页未被锁定、引用或脏化
     - 清除 PG_uptodate、PG_dirty 等标志
     - 调用 `page_pool_free_pages()` 或同步回收

3. **返回伙伴分配器**:
   - `__free_pages_ok()`(mm/page_alloc.c:1603) 将页返回伙伴系统
   - 合并相邻空闲块(coalescing)
   - 页可被重新分配

#### Folio 释放流程
- `folio_put()` 通过 folio 的头页调用 `put_page()`
- 整个 folio(所有尾页)被原子地释放
- 尾页元数据不需要单独清理(重新初始化时覆盖)

## Key Operations

### 引用计数操作 (Reference Count Operations)

**在 page_ref.h 中定义**:

- `page_ref_count(page)`: 获取引用计数
- `folio_ref_count(folio)`: 获取 folio 引用计数
- `page_ref_add(page, nr)`: 增加引用计数
- `folio_ref_add(folio, nr)`: 增加 folio 引用计数
- `page_ref_sub(page, nr)`: 减少引用计数
- `folio_ref_sub(folio, nr)`: 减少 folio 引用计数
- `page_ref_freeze(page, count)`: 冻结引用计数(用于迁移)
- `page_ref_unfreeze(page)`: 解冻引用计数

### 页-Folio 转换操作

- `page_folio(page)`: 从任何页(头页或尾页)获取 folio
- `folio_page(folio, n)`: 获取 folio 中的第 n 个页
- `folio_file_page(folio, index)`: 获取映射到特定偏移量的页

### 内存顺序操作 (Memory Order Operations)

- `folio_try_get_rcu()`: RCU 保护下的原子引用获取
- `folio_get_nontail(page)`: 对非尾页的原子获取

### 标志检查操作 (Flag Checking)

- `PageDirty()`, `PageLocked()`, `PageUptodate()` 等(page-flags.h)
- `folio_test_dirty()`, `folio_test_locked()` 等(folio 等效物)

### 页回收和迁移

- `get_page_unless_zero(page)`: 在引用计数不为 0 时获取页
- `page_ref_freeze()`: 冻结引用计数以进行迁移
- `page_ref_unfreeze()`: 迁移后解冻

## Relationships

```
struct page
    ├─ flags ──> page-flags.h (定义标志位含义)
    ├─ mapping ──> address_space (页缓存或 anon_vma)
    ├─ _refcount ──> page_ref.h (引用计数操作)
    ├─ lru ──> lru_list (页面回收列表)
    ├─ buddy_list ──> buddy allocator (伙伴分配)
    └─ folio ──> struct folio (现代大页面抽象)

struct folio
    ├─ page (与 struct page 的内存布局对齐)
    ├─ _mm_id ──> reverse mapping (反向映射追踪)
    ├─ _large_mapcount ──> large folio mapping
    └─ _deferred_list ──> folio_split() (分割待办)
```

### 关键子系统

- **页分配器**(mm/page_alloc.c): alloc_pages(), __folio_alloc_noprof(), free_pages()
- **页缓存**(mm/filemap.c): add_to_page_cache_lru(), folio_add_lru()
- **反向映射**(mm/rmap.c): page_add_anon_rmap(), page_remove_rmap()
- **SLUB 分配器**(mm/slub.c): 使用 page->freelist 和 page->inuse
- **网络栈**(net/core/page_pool.c): page_pool 结构
- **DMA 处理**: 需要追踪 page->dma_addr

## The page-to-folio Transition

### 历史背景

Linux 内核历来依赖 `struct page` 代表内存管理的基本单位。然而,随着现代硬件支持更大的页面(如 2MB 的 THPs),基于单一页面的 API 变得繁琐且低效。

### Folio 的优势

1. **统一接口**: 单一 folio API 处理小页面(4KB)和大页面(2MB+)
2. **减少代码复杂性**: 无需分别处理头页和尾页
3. **性能改进**: 更少的间接引用,更好的缓存局部性
4. **内存布局优化**: folio 专用字段(_large_mapcount 等)避免了页之间的对齐浪费

### 兼容性层

`folio-compat.c` 提供向后兼容函数:
- `page_folio(page)`: 任何页返回其 folio
- 旧 page-based API 内部调用 folio API

### 迁移策略

**驱动程序作者应该**:
- 新代码使用 folio API(folio_alloc(), folio_put(), 等)
- 逐步将旧 page API 替换为 folio 等效物
- 使用 `page_folio(page)` 从旧代码获取 folio

## Android-Specific Changes

### ACK 中的应用

Android Common Kernel (ACK) 基于上游 Linux 内核,未在 `struct page` 或 `struct folio` 的定义中进行任何特殊修改。但是,以下 Android 特定子系统与页面管理交互:

1. **Ion 内存分配器**:
   - 使用页分配器进行内存分配,追踪通过自定义元数据而非页标志

2. **Secure Page Pool**:
   - Trusty 虚拟化监视程序使用页映射进行共享内存通信
   - 在 common-modules/trusty 中引用 struct page

3. **KGSL GPU 驱动**:
   - 分配大块连续内存供 GPU 使用
   - 管理与 GPU 地址空间的映射

4. **内存压力回应**:
   - Android 的 Low Memory Killer 监视内存压力
   - 依赖页回收(mm/page_reclaim.c 和 vmscan.c)

### 页面标志的 Android 用途

虽然 page->flags 定义在上游中,但 Android 驱动程序可能使用:
- `PagePrivate()` 标志进行驱动程序特定的元数据
- `page->private` 指针存储驱动程序状态
- `page->mapping` 用于大页面映射追踪

## Cross-References

### 内核头文件
- `include/linux/mm_types.h:79` - struct page 定义
- `include/linux/mm_types.h:401` - struct folio 定义
- `include/linux/page-flags.h` - 页标志位定义和测试宏
- `include/linux/page_ref.h` - 引用计数操作
- `include/linux/page.h` - 页面操作的公共 API

### 内核源文件
- `mm/page_alloc.c` - 页分配器实现
- `mm/folio-compat.c:13-86` - 页到 folio 兼容层
- `mm/filemap.c` - 页缓存实现
- `mm/rmap.c` - 反向映射
- `mm/vmscan.c` - 页回收

### 追踪和调试
- `include/trace/events/page_ref.h` - 引用计数追踪点
- `mm/debug_page_ref.c` - 引用计数调试

### 相关概念
- **address_space**: 页缓存容器
- **folio**: 现代页面抽象
- **buddy allocator**: 页面分配和释放机制
- **page flags**: 页面状态标志位集合
- **reference counting**: 内存生命周期管理

---

**注**: 本文档基于 Linux 内核源代码和 ACK 的分析。folio API 仍在演进,一些子系统仍在从页 API 迁移到 folio API。
