---
type: data-structure
defined_in: common/drivers/android/binder_internal.h:444
size_approx: 400-600 bytes (variable, includes embedded dynamic structures)
lifecycle: allocated in binder_open(), freed by binder_free_proc()
related: [binder_thread, binder_node, binder_ref, binder_context, binder_alloc]
last_updated: 2026-04-09
---

# `struct binder_proc`

## Purpose

`struct binder_proc` 是 Android Binder IPC 驱动中代表单个 Binder 进程的核心数据结构。每当应用程序或系统服务打开 `/dev/binder` 设备文件时,都会创建一个 `binder_proc` 实例。该结构维护进程的所有 Binder 相关状态,包括线程管理、节点管理、引用管理、事务处理和内存分配。

本结构是 Android 特定的,完全专为 Binder IPC 机制设计,未出现在通用 Linux 内核中。

## Definition

```c
struct binder_proc {
    struct hlist_node proc_node;           // 全局 binder_procs 哈希链表的节点
    struct rb_root threads;                // Binder 线程的红黑树 (thread->rb_node)
    struct rb_root nodes;                  // Binder 节点的红黑树,按 node->ptr 排序
    struct rb_root refs_by_desc;           // 引用的红黑树,按引用描述符排序
    struct rb_root refs_by_node;           // 引用的红黑树,按所指向节点排序
    struct list_head waiting_threads;      // 当前等待工作的线程列表
    int pid;                               // 进程的 PID (group_leader)
    struct task_struct *tsk;               // 进程的 task_struct 指针
    const struct cred *cred;               // 文件打开时的 cred 结构
    struct hlist_node deferred_work_node;  // 延迟工作队列的节点
    int deferred_work;                     // 延迟工作的位掩码
    int outstanding_txns;                  // 待处理事务计数
    bool is_dead;                          // 标志进程已死亡
    bool is_frozen;                        // 标志进程已冻结(无法处理事务)
    bool sync_recv;                        // 进程收到同步事务的标志
    bool async_recv;                       // 进程收到异步事务的标志
    wait_queue_head_t freeze_wait;         // 等待所有待处理事务完成的等待队列
    struct dbitmap dmap;                   // 动态位图,用于管理可用的引用描述符
    struct list_head todo;                 // 此进程的待做工作列表
    struct binder_stats stats;             // 每进程 Binder 统计数据
    struct list_head delivered_death;      // 已传递的死亡通知列表
    struct list_head delivered_freeze;     // 已传递的冻结通知列表
    u32 max_threads;                       // Binder 线程的最大数量限制
    int requested_threads;                 // 请求但尚未启动的线程数
    int requested_threads_started;         // 已启动的 Binder 线程数
    int tmp_ref;                           // 临时引用计数(保持进程活跃)
    struct binder_priority default_priority; // 默认调度优先级
    struct dentry *debugfs_entry;          // debugfs 节点指针
    struct binder_alloc alloc;             // Binder 内存分配器状态
    struct binder_context *context;        // 此进程的 Binder 上下文
    spinlock_t inner_lock;                 // 内锁(可嵌套于 outer_lock)
    spinlock_t outer_lock;                 // 外锁(锁顺序: 外 > 节点 > 内)
    struct dentry *binderfs_entry;         // binderfs 文件系统中的进程日志入口
    bool oneway_spam_detection_enabled;    // 是否启用单向调用垃圾邮件检测
};
```

## Field Groups

### 链表与树结构 (List and Tree Structures)
- **proc_node**: 用于全局 `binder_procs` 哈希列表
- **threads**: 红黑树,存储该进程中的所有 `binder_thread` 结构
- **nodes**: 红黑树,存储此进程拥有的所有 `binder_node` 结构
- **refs_by_desc**: 红黑树,按引用描述符索引该进程的所有引用
- **refs_by_node**: 红黑树,按所指向节点索引该进程的所有引用
- **waiting_threads**: 列表,存储当前等待 IPC 工作的线程
- **todo**: 待处理工作项(通知、事务完成)的列表
- **delivered_death**: 已传递的死亡通知列表
- **delivered_freeze**: 已传递的冻结通知列表
- **deferred_work_node**: 全局延迟工作列表中的节点

### 进程身份识别 (Process Identity)
- **pid**: 进程的 PID,由 `current->group_leader->pid` 初始化
- **tsk**: 指向进程 group_leader 的 `task_struct`
- **cred**: 进程在打开设备文件时的安全凭证

### 事务与工作管理 (Transaction and Work Management)
- **outstanding_txns**: 未完成事务的计数器,用于冻结/解冻流程同步
- **deferred_work**: 位掩码,标记待处理的延迟工作类型
- **tmp_ref**: 临时引用计数,防止在活跃事务处理期间释放进程
- **max_threads**: 该进程可创建的最大 Binder 线程数
- **requested_threads**: 已请求但尚未启动的线程数(0 或 1)
- **requested_threads_started**: 已启动的 Binder 线程数

### 进程生命周期状态 (Lifecycle State)
- **is_dead**: 当 `binder_release()` 被调用后设置为 true
- **is_frozen**: 表示进程被冻结,无法处理新的 Binder 调用
- **sync_recv**: 表示自冻结以来收到过同步事务
- **async_recv**: 表示自冻结以来收到过异步事务
- **freeze_wait**: 等待队列,用于等待所有待处理事务完成

### 内存与资源管理 (Memory and Resource Management)
- **alloc**: 每进程的 Binder 内存分配器状态(共享内存映射等)
- **dmap**: 动态位图,管理引用描述符的分配
- **stats**: 原子统计计数器,无需锁保护

### 调度与优先级 (Scheduling and Priority)
- **default_priority**: 进程中所有 Binder 调用继承的默认调度优先级

### 调试与日志 (Debugging and Logging)
- **debugfs_entry**: 指向 `/sys/kernel/debug/binder` 中的 debugfs 条目
- **binderfs_entry**: 指向 `binderfs` 中的进程特定日志文件

### 并发控制 (Concurrency Control)
- **inner_lock**: 自旋锁,保护对 `threads`、`nodes` 和事务相关的访问
- **outer_lock**: 自旋锁,保护对 `refs_by_desc` 和 `refs_by_node` 的访问

### Android 特定特性 (Android-Specific Features)
- **context**: 指向该进程所属的 `binder_context` (可能被多个 pid 共享)
- **oneway_spam_detection_enabled**: 控制单向调用是否进行垃圾邮件检测

## Lifecycle

### 分配 (Allocation)

`binder_proc` 在文件操作的 `binder_open()` 期间分配,位置在 `binder.c:6245`:

1. **初始化阶段**:
   ```c
   proc = kzalloc(sizeof(*proc), GFP_KERNEL);  // 零初始化内存
   ```

2. **成员初始化**:
   - `dbitmap_init(&proc->dmap)` - 初始化引用描述符位图
   - `spin_lock_init()` 初始化内部和外部自旋锁
   - `get_task_struct()` 获取对 group_leader 的引用
   - `get_cred()` 获取文件打开时的凭证
   - `INIT_LIST_HEAD()` 初始化所有列表
   - `init_waitqueue_head(&proc->freeze_wait)` 初始化冻结等待队列
   - `binder_alloc_init()` 初始化内存分配器状态

3. **注册**:
   - 进程添加到全局 `binder_procs` 哈希列表
   - 为新 PID 创建 debugfs 和 binderfs 条目

### 释放 (Deallocation)

释放是两阶段过程,涉及 `binder_release()` 和 `binder_deferred_release()`,位置在 `binder.c:6380-6461`:

**第一阶段 - binder_release()** (在进程关闭 `/dev/binder` 文件描述符时):
- 移除 debugfs 条目
- 移除 binderfs 条目
- 向延迟工作队列添加 `BINDER_DEFERRED_RELEASE` 工作

**第二阶段 - binder_deferred_release()** (由工作队列异步执行):
1. 从全局 `binder_procs` 列表中移除进程
2. 清除上下文管理器节点(如果适用)
3. 设置 `proc->is_dead = true`
4. 逐个释放所有 `binder_thread` 结构
5. 逐个释放所有 `binder_node` 结构
6. 逐个释放所有 `binder_ref` 结构
7. 释放所有待做工作项
8. 调用 `binder_proc_dec_tmpref(proc)` 完成最后清理

**完全释放条件**:
```c
if (proc->is_dead && RB_EMPTY_ROOT(&proc->threads) && !proc->tmp_ref) {
    binder_free_proc(proc);  // 最终调用 kfree()
}
```

进程必须满足以下条件才能真正释放:
- `is_dead` 为 true (已调用 `binder_release()`)
- 没有活跃线程 (线程树为空)
- `tmp_ref` 为零 (没有待处理事务引用进程)

## Key Operations

### 临时引用管理 (Temporary Reference Management)

**binder_proc_dec_tmpref()** `binder.c:1727`:
- 递减 `proc->tmp_ref` 计数器
- 当条件满足时触发进程释放
- 在事务完成和线程释放时调用

临时引用用于防止在处理事务期间进程被释放。当创建事务或线程被释放时增加 `tmp_ref`,完成后递减。

### 冻结支持 (Freeze Support)

Android 特定特性,允许系统冻结进程以实现应用待命:

- **BINDER_FREEZE ioctl** `binder.c:6073`:
  - 标记进程 `is_frozen = true`
  - 等待所有待处理事务完成
  - 唤醒冻结等待队列中的等待者

- **冻结期间的事务跟踪**:
  - `sync_recv`: 标记是否收到同步事务
  - `async_recv`: 标记是否收到异步事务
  - `outstanding_txns`: 计数待处理事务

- **解冻**:
  - 重置 `is_frozen`、`sync_recv`、`async_recv` 标志
  - 允许新事务进行

### 工作队列管理 (Work Queue Management)

进程维护多个工作队列:

1. **proc->todo**: 进程级工作(事务通知、完成)
2. **proc->delivered_death**: 死亡通知
3. **proc->delivered_freeze**: 冻结通知

线程定期轮询这些队列以获取待处理工作。

### 内存分配 (Memory Allocation)

`proc->alloc` 结构化管理:
- 进程间共享内存映射 (`binder_mmap()`)
- 事务数据缓冲区分配/释放
- 缓冲区池管理

### 引用描述符管理 (Reference Descriptor Management)

`proc->dmap` 动态位图管理:
- 为进程本地引用分配唯一的 32 位描述符
- 支持动态增长的引用数量
- 快速查找描述符与内核引用对象的映射

## Relationships

### 父结构 (Parent Structures)
- **binder_context**: 进程所属的 Binder 上下文(通常一个,但多进程可共享)
- **binder_device**: 进程打开的设备(保持对设备的引用计数)

### 子结构 (Child Structures)
- **binder_thread**: 一对多关系,线程存储在 `threads` 红黑树中
- **binder_node**: 一对多关系,节点存储在 `nodes` 红黑树中
- **binder_ref**: 一对多关系,引用存储在 `refs_by_desc` 和 `refs_by_node` 中
- **binder_alloc**: 一对一,嵌入式结构,管理共享内存
- **binder_stats**: 一对一,嵌入式统计数据

### 关键链接 (Key Links)
- 每个 `binder_thread` 通过 `thread->proc` 指回其进程
- 每个 `binder_node` 通过 `node->proc` 指回其所有者进程
- 每个 `binder_ref` 通过 `ref->proc` 指向包含它的进程(源)和 `ref->node->proc` 指向目标进程

### 全局关系 (Global Relationships)
- 所有活跃进程存储在 `binder_procs` 全局哈希列表中(由 `binder_procs_lock` 保护)
- 延迟释放的进程存储在 `binder_deferred_list` 中(由 `binder_deferred_lock` 保护)

## Android-Specific Changes

本结构完全是 Android 特定的,包含多个 Android-only 特性:

### 1. 冻结机制 (Freezing Mechanism)
- **字段**: `is_frozen`, `sync_recv`, `async_recv`, `freeze_wait`
- **目的**: 支持 Android 应用待命/冻结,减少电耗
- **IPC 命令**: `BINDER_FREEZE` 和 `BINDER_GET_FROZEN_INFO`
- **集成**: 深度集成到事务处理和线程管理中

### 2. 单向调用垃圾邮件检测
- **字段**: `oneway_spam_detection_enabled`
- **目的**: 防止应用使用单向调用造成拒绝服务攻击
- **机制**: 跟踪单向调用速率,超过阈值时失败

### 3. Binderfs 集成
- **字段**: `binderfs_entry`
- **功能**: 在 `binderfs` 文件系统中为每个进程创建日志文件
- **用途**: 提供细粒度的 Binder 事务日志

### 4. 多进程上下文支持
- **字段**: `context` (指向 `binder_context`)
- **特性**: 单个系统可有多个 Binder 驱动实例,每个有自己的命名空间
- **用途**: 支持多个应用特定的 Binder 域

### 5. 死亡通知和冻结通知
- **字段**: `delivered_death`, `delivered_freeze`
- **功能**: 与目标进程解关联时通知客户端进程
- **Android 用途**: 用于生命周期管理和应用待命

## Cross-References

### 相关内核文件
- **binder_internal.h** (定义): `common/drivers/android/binder_internal.h:444`
- **binder.c** (实现):
  - `binder_open()`: `common/drivers/android/binder.c:6245`
  - `binder_release()`: `common/drivers/android/binder.c:6380`
  - `binder_deferred_release()`: `common/drivers/android/binder.c:6461`
  - `binder_proc_dec_tmpref()`: `common/drivers/android/binder.c:1727`
- **binder_alloc.h**: 内存分配器定义
- **dbitmap.h**: 动态位图实现

### 相关数据结构
- `struct binder_thread`: Binder 线程表示
- `struct binder_node`: Binder 对象/节点表示
- `struct binder_ref`: 进程间引用
- `struct binder_transaction`: IPC 事务
- `struct binder_context`: Binder 上下文/命名空间

### 锁定规则
- **锁顺序**: `outer_lock` > `node->lock` > `inner_lock`
- **保护说明**:
  - `inner_lock`: 线程、节点、待做工作、事务堆栈
  - `outer_lock`: 引用树(按描述符和节点)
  - `binder_procs_lock`: 全局进程列表
  - `binder_deferred_lock`: 延迟工作列表

### 统计信息
- **stats.obj_created[BINDER_STAT_PROC]**: 创建的进程数
- **stats.obj_deleted[BINDER_STAT_PROC]**: 删除的进程数
- 每个进程维护的 `binder_stats` 提供事务、线程、节点的计数

### 调试接口
- **debugfs**: `/sys/kernel/debug/binder/[pid]` - 进程状态导出
- **binderfs**: `[binderfs_mount]/[context]/[pid]` - 进程日志文件
- **BINDER_DEBUG_OPEN_CLOSE**: 调试标志用于追踪生命周期事件
