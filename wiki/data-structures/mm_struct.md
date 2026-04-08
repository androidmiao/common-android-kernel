---
type: data-structure
defined_in: common/include/linux/mm_types.h:1075
size_approx: variable (base ~2-3KB, with dynamic flexible_array)
lifecycle: allocated via mm_alloc() or dup_mm(), freed via mmput() or mmdrop()
related: [task_struct, vm_area_struct, page, maple_tree, cpumask_t]
last_updated: 2026-04-09
---

# `struct mm_struct`

## 目的

`struct mm_struct` 是 Linux 核心内存管理的中枢数据结构。每个用户态进程都拥有一个 `mm_struct` 实例，用于管理该进程的虚拟地址空间、页表、内存区域、以及与内存相关的统计信息。这个结构充当了虚拟内存管理的"进程地址空间描述符"，是进程内存隔离、页表管理、和 NUMA 感知的基础。

## 定义

以下是 `struct mm_struct` 的关键字段简化视图（完整定义见 common/include/linux/mm_types.h:1075）：

```c
struct mm_struct {
    struct {
        /* 引用计数和高速缓存行对齐 */
        atomic_t mm_count;          /* 引用计数（高速缓存行对齐） */
        
        struct maple_tree mm_mt;    /* VMA 的 Maple Tree */
        unsigned long mmap_base;    /* mmap 区域基址 */
        unsigned long mmap_legacy_base;  /* 向下增长 mmap 的基址 */
        unsigned long task_size;    /* 任务 VM 空间大小 */
        pgd_t *pgd;                 /* 页全局目录指针 */
        
        atomic_t membarrier_state;  /* membarrier 行为标志 */
        atomic_t mm_users;          /* 用户计数 */
        
        struct mm_mm_cid mm_cid;    /* 调度器 MM CID 存储 */
        atomic_long_t pgtables_bytes; /* 所有页表字节大小 */
        int map_count;              /* VMA 数量 */
        
        spinlock_t page_table_lock; /* 保护页表和计数器 */
        struct rw_semaphore mmap_lock; /* 保护 VMA 和页表映射 */
        struct list_head mmlist;    /* 全局 mm 链表 */
        
        seqcount_t mm_lock_seq;     /* VMA 读写锁序列号 */
        seqcount_t write_protect_seq; /* 写保护序列号 */
        
        spinlock_t arg_lock;        /* 保护参数/环境地址 */
        unsigned long start_code, end_code;   /* 代码段范围 */
        unsigned long start_data, end_data;   /* 数据段范围 */
        unsigned long start_brk, brk, start_stack; /* 堆栈/堆 */
        unsigned long arg_start, arg_end, env_start, env_end;
        unsigned long saved_auxv[AT_VECTOR_SIZE]; /* ELF 辅助向量 */
        
        struct percpu_counter rss_stat[NR_MM_COUNTERS]; /* RSS 统计 */
        
        unsigned long hiwater_rss;  /* RSS 高水位 */
        unsigned long hiwater_vm;   /* VM 高水位 */
        unsigned long total_vm;     /* 映射的总页数 */
        unsigned long locked_vm;    /* mlock 页数 */
        atomic64_t pinned_vm;       /* 引脚页数 */
        unsigned long data_vm;      /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
        unsigned long exec_vm;      /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
        unsigned long stack_vm;     /* VM_STACK */
        
        struct linux_binfmt *binfmt; /* ELF/a.out 二进制格式 */
        mm_context_t context;       /* 架构特定的 MM 上下文 */
        mm_flags_t flags;           /* MM 标志位图 */
        
        struct kioctx_table *ioctx_table; /* AIO 上下文 */
        struct task_struct *owner;  /* mm 的规范所有者 */
        struct user_namespace *user_ns; /* 用户命名空间 */
        struct file *exe_file;      /* /proc/[pid]/exe 文件 */
        
        struct mmu_notifier_subscriptions *notifier_subscriptions;
        
        unsigned long numa_next_scan;  /* NUMA 扫描时间戳 */
        unsigned long numa_scan_offset; /* NUMA 扫描偏移量 */
        int numa_scan_seq;          /* NUMA 扫描序列号 */
        
        atomic_t tlb_flush_pending; /* TLB 刷新待处理 */
        
        struct iommu_mm_data *iommu_mm; /* IOMMU 数据（可选） */
        unsigned long ksm_merging_pages; /* KSM 合并页数 */
        atomic_long_t hugetlb_usage; /* HugeTLB 使用计数 */
        
        struct work_struct async_put_work; /* 异步释放工作队列 */
        
        mm_id_t mm_id;              /* 全局 MM 唯一标识符 */
        
    } __randomize_layout;
    
    char flexible_array[] __aligned(__alignof__(unsigned long));
    /* 动态数组：cpumask_t (cpu_vm_mask) + mm_cpus_allowed + cidmask */
};
```

## 字段分组

### 标识和引用计数
- **mm_count**: 原始引用计数。当降到 0 时，结构体被释放。由 mmgrab()/mmdrop() 修改。
- **mm_users**: 用户引用计数（包括用户空间）。由 mmget()/mmput() 修改。当降到 0 时，触发 __mmput()。

### 虚拟地址空间管理
- **mm_mt**: Maple Tree，存储该进程的所有 VMA（虚拟内存区域）。
- **mmap_base, mmap_legacy_base**: mmap 区域的基址（向上或向下增长的变体）。
- **task_size**: 用户空间虚拟地址空间的总大小。
- **pgd**: 指向该进程的页表的顶级（L1）目录指针。

### 页表保护和锁定
- **mmap_lock**: 读写信号量，保护 VMA 树、页表映射和大多数 mm 字段。
- **page_table_lock**: 自旋锁，保护页表和计数器的细粒度访问。
- **write_protect_seq**: 序列号，用于跟踪写保护操作（特别是 fork 期间的 COW）。

### 内存核算
- **total_vm, data_vm, exec_vm, stack_vm**: 分类的虚拟页计数。
- **locked_vm**: mlock() 锁定的页数。
- **pinned_vm**: 引脚页数（引脚数永久递增）。
- **rss_stat[]**: 每 CPU 的 RSS（驻留集大小）计数器数组。
- **hiwater_rss, hiwater_vm**: RSS 和 VM 使用的高水位标记。

### 地址布局
- **start_code, end_code**: 代码段的虚拟地址范围。
- **start_data, end_data**: 初始化数据段的范围。
- **start_brk, brk, start_stack**: 堆的起始和当前位置，以及栈的起始位置。
- **arg_start/end, env_start/end**: 命令行参数和环境字符串的地址。

### 调度和 CPU 亲和性
- **mm_cid**: 每 CPU 的 MM 上下文 ID（用于调度器）。
- **flexible_array**: 动态数组，包含：
  - `cpumask_t` - CPU VM 掩码（标记哪些 CPU 正在使用此 mm）。
  - `mm_cpus_allowed` - 所有线程允许的 CPU 的并集（CONFIG_SCHED_MM_CID）。
  - `cidmask` - 上下文 ID 位图（CONFIG_SCHED_MM_CID）。

### 进程二进制和命名空间
- **binfmt**: 指向 linux_binfmt 结构，用于加载 ELF/a.out 二进制格式。
- **user_ns**: 用户命名空间指针，用于 UID/GID 隔离。
- **exe_file**: 指向进程可执行文件的 RCU 保护指针。

### 特定架构和 NUMA
- **context**: 架构特定的内存管理上下文（TLB 管理、ASN 等）。
- **numa_next_scan, numa_scan_offset, numa_scan_seq**: NUMA 启发式故障的扫描参数。

### 内存通知和 MMU 钩子
- **notifier_subscriptions**: MMU 通知者（用于驱动程序和模块注册 mmunmap 回调）。
- **iommu_mm**: IOMMU 特定的 mm 数据（CONFIG_IOMMU_MM_DATA）。

### 内存合并和 HugeTLB
- **ksm_merging_pages, ksm_rmap_items, ksm_zero_pages**: KSM（内核同页合并）统计。
- **hugetlb_usage**: HugeTLB 页面使用计数。

### 其他
- **flags**: 位图标志（MM_DUMPABLE、MM_VM_MOVABLE 等），由 mm_flags_* 辅助函数访问。
- **lru_gen**: LRU 代代扫描的 mm 列表和位图（CONFIG_LRU_GEN_WALKS_MMU）。
- **mm_id**: 全局唯一的 MM ID，用于 rmap（CONFIG_MM_ID）。

## 生命周期

### 分配

1. **mm_alloc()** (kernel/fork.c:1164)
   - 调用 allocate_mm() 分配 mm_struct。
   - 用零初始化。
   - 调用 mm_init() 进行初始化（见下文）。

2. **mm_init()** (kernel/fork.c:1082)
   - 初始化 Maple Tree（mm_mt）和外部锁定。
   - 设置引用计数：mm_users=1，mm_count=1。
   - 初始化锁和信号量（page_table_lock、mmap_lock、arg_lock）。
   - 初始化 cpumask、AIO、TLB 挂起计数器。
   - 调用 mm_alloc_pgd() 分配页全局目录。
   - 调用 mm_alloc_id() 分配 MM ID。
   - 调用 init_new_context() 进行架构特定的初始化。
   - 调用 mm_alloc_cid() 进行调度器 CID 初始化。
   - 初始化 per-CPU RSS 计数器。

3. **dup_mm()** (kernel/fork.c:1526)
   - 在 fork() 期间使用，复制现有的 mm_struct。
   - 调用 allocate_mm() 并使用 memcpy() 复制原 mm。
   - 调用 mm_init() 进行初始化。
   - 调用 dup_mmap() 复制所有 VMA 和页表映射。
   - 更新高水位（hiwater_rss、hiwater_vm）。

### 使用

- **mmget()** / **mmget_not_zero()**: 获取对 mm 的引用（增加 mm_users）。
- **mmput()**: 释放引用（减少 mm_users）。当 mm_users 降到 0 时，触发 __mmput()。
- **mmgrab()** / **mmdrop()**: 操作 mm_count。
- **mmput_async()**: 异步释放（使用工作队列）。

### 释放

1. **__mmput()** (kernel/fork.c:1177)
   - 验证 mm_users == 0。
   - 清理 uprobe 状态。
   - 调用 exit_aio() 关闭异步 I/O 上下文。
   - 调用 ksm_exit() 清理 KSM。
   - 调用 khugepaged_exit() 和 exit_mmap() 释放所有 VMA 和页表。
   - 从全局 mmlist 中移除。
   - 释放 binfmt 模块。
   - 调用 mmdrop() 减少 mm_count。

2. **mmdrop()** (kernel/fork.c)
   - 减少 mm_count。
   - 当 mm_count 降到 0 时，调用 free_mm() 释放结构体本身。

## 关键操作

| 操作 | 文件:行号 | 说明 |
|------|----------|------|
| mm_alloc | kernel/fork.c:1164 | 分配并初始化新的 mm_struct |
| mm_init | kernel/fork.c:1082 | 初始化 mm_struct 的所有字段 |
| dup_mm | kernel/fork.c:1526 | 复制 mm 用于进程 fork |
| mmput | kernel/fork.c:1203 | 减少 mm_users 并可能触发释放 |
| mmput_async | kernel/fork.c:1221 | 异步释放 mm |
| __mmput | kernel/fork.c:1177 | 执行 mm 的清理和释放 |
| mmgrab | include/linux/sched/mm.h | 增加 mm_count |
| mmdrop | include/linux/sched/mm.h | 减少 mm_count |
| mmget | include/linux/sched/mm.h | 增加 mm_users（若 mm_users > 0） |
| mmget_not_zero | include/linux/sched/mm.h | 原子性增加 mm_users（如果非零） |
| exit_mmap | mm/mmap.c | 释放所有 VMA 和页表映射 |
| dup_mmap | mm/dup_mmap.c | 复制 VMA 树和页表映射 |
| expand_stack | mm/mmap.c | 扩展栈 VMA |
| find_vma | mm/mmap.c | 在 Maple Tree 中查找 VMA |

## 关系

### 嵌入和指向的结构

```
mm_struct
├── maple_tree (mm_mt)               → vm_area_struct nodes
├── task_struct* owner               → 规范用户/所有者
├── linux_binfmt* binfmt            → ELF 格式定义
├── mm_context_t context            → 架构特定上下文
├── file* exe_file (RCU)            → 可执行文件
├── iommu_mm_data* iommu_mm         → IOMMU 数据
├── mmu_notifier_subscriptions*     → 注册的通知者
├── kioctx_table* ioctx_table       → AIO 上下文
├── user_namespace*                 → UID 隔离命名空间
├── percpu_counter rss_stat[]       → 每 CPU RSS 计数
└── flexible_array                  → cpumask + MM CID 数据
    ├── cpumask_t (cpu_vm_mask)
    ├── cpumask_t mm_cpus_allowed
    └── unsigned long* cidmask
```

### 反向关系

- **task_struct**: 每个进程的 task_struct 有两个 mm 指针：
  - `mm`: 用户空间进程的 mm（NULL 表示内核线程）
  - `active_mm`: 当前活动的 mm（用于上下文切换）

- **vm_area_struct**: 每个 VMA 都通过 Maple Tree 链接到其 mm_struct。

- **page**: 通过 rmap（reverse mapping）和 MM ID 系统链接到 mm。

## Android 特定的更改

### IOMMU 支持
- **iommu_mm** 字段（CONFIG_IOMMU_MM_DATA）：用于 IOMMU 驱动程序存储每 mm 的 IOMMU 状态。在 ACK 中添加，用于加速 IOMMU 转换和隔离。

### MM ID 系统
- **mm_id** 字段（CONFIG_MM_ID）：全局唯一的 MM 标识符，用于改进 rmap 和反向映射性能。ACK 中引入，用于高效的 mm 识别。

### 双重计数模型
- **mm_count / mm_users** 架构：虽然来自上游，但在 ACK 中经过优化以支持高内存压力场景和异步释放路径（mmput_async）。

### futex 私有哈希（CONFIG_FUTEX_PRIVATE_HASH）
- **futex_hash_lock, futex_phash, futex_atomic** 等字段：ACK 特定的优化，用于加速私有 futex 操作。

### 无厂商钩子
- ACK 中的 mm_struct 没有显式的"vendor hook"字段。厂商特定的扩展通过：
  - 额外的 CONFIG 选项（如 CONFIG_IOMMU_MM_DATA）
  - 内部字段利用（如 iommu_mm、lru_gen）
  - 模块 API（通过 mmu_notifier、IOMMU 框架）

## 交叉参考

- **include/linux/mm.h**: 核心内存管理声明
- **include/linux/sched/mm.h**: 进程 mm 访问宏和函数
- **include/linux/mm_types.h**: 本结构体定义（common/include/linux/mm_types.h:1075）
- **kernel/fork.c**: 分配、初始化、复制、释放实现
- **mm/mmap.c**: VMA 管理和 Maple Tree 操作
- **mm/dup_mmap.c**: fork() 期间的 VMA 和页表复制
- **mm/exit_mmap.c**: 进程退出时的 mm 清理
- **drivers/iommu/**: IOMMU 集成（使用 iommu_mm）
- **include/linux/mmu_notifier.h**: MMU 通知注册

### 大小估计

典型的 64 位系统上：
- **基本 mm_struct 大小**：约 2-3 KB（包含所有固定字段）
- **flexible_array 大小**：
  - cpumask_t: 8 字节（NR_CPUS 位）→ 通常 8-128 字节
  - mm_cpus_allowed: 同上 → 8-128 字节
  - cidmask: 同上 → 8-128 字节
  - **总计**: ~24 字节（单 socket）到 ~512 字节（大型系统）
- **总大小范围**: ~2 KB - ~4 KB（单 socket 到 256+ CPU 系统）

