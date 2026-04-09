---
type: api
direction: user-to-kernel
related:
  - ../subsystems/scheduler.md
  - ../subsystems/memory-management.md
  - ../subsystems/filesystems.md
  - ../subsystems/ipc.md
  - ../subsystems/networking.md
  - ../concepts/bpf.md
last_updated: 2026-04-09
---

# 系統呼叫介面

## Overview

系統呼叫（syscall）是使用者空間進入核心態的唯一合法途徑。ACK（Linux 6.19-rc8）在 ARM64 架構上定義了 **471 個**系統呼叫（`__NR_syscalls = 471`），編號 0–470。系統呼叫表定義於通用標頭 `include/uapi/asm-generic/unistd.h`，ARM64 入口實現於 `arch/arm64/kernel/syscall.c`。

## Interface Definition

### 系統呼叫表

**定義：** `include/uapi/asm-generic/unistd.h`（line 864：`#define __NR_syscalls 471`）

使用巨集模式定義：
- `__SYSCALL(nr, handler)` — 基本系統呼叫
- `__SC_COMP(nr, native, compat)` — 含 32-bit 相容版本
- `__SC_3264(nr, 32bit, 64bit)` — 32/64-bit 不同實現

**宣告：** `include/linux/syscalls.h`（463+ 個 `asmlinkage long` 函式原型）

### ARM64 呼叫慣例

| 暫存器 | 用途 |
|--------|------|
| X0–X5 | 引數 1–6 |
| X8 | 系統呼叫編號（64-bit 模式） |
| X7 | 系統呼叫編號（32-bit 相容模式） |
| X0 | 回傳值 |

`regs->orig_x0` 保存原始第一引數，供重啟語意使用。

### ARM64 入口流程

#### 1. 異常進入

使用者空間執行 `svc #0` 指令觸發同步異常，由 `arch/arm64/kernel/entry.S` 中的 `el0_svc` 處理。

#### 2. 系統呼叫派發

**`do_el0_svc()`** @ `arch/arm64/kernel/syscall.c:149-151`：

```
el0_svc (entry.S)
  → do_el0_svc()
    → el0_svc_common()  @ line 73-147
      ├── 驗證 syscall_nr < __NR_syscalls
      ├── syscall_trace_enter() — 追蹤/稽核
      ├── add_random_kstack_offset() — 堆疊隨機化
      ├── MTE async fault 檢測 (lines 99-107)
      └── sys_call_table[syscall_nr](regs)
```

**32-bit 相容：** `do_el0_svc_compat()` @ lines 154-159，使用 `compat_sys_call_table`，從 X7 取得系統呼叫編號。

#### 3. 系統呼叫表

**`arch/arm64/kernel/sys.c:54-62`**：

```c
extern const syscall_fn_t sys_call_table[__NR_syscalls];
// syscall_fn_t = long (*)(const struct pt_regs *)
```

表透過 `#include <asm/syscall_table_64.h>` 初始化。

#### 4. Wrapper 機制

**`arch/arm64/include/asm/syscall_wrapper.h`**：

`__SYSCALL_DEFINEx` 巨集（lines 49-65）將暫存器引數從 `struct pt_regs` 解包為函式參數：
- `SC_ARM64_REGS_TO_ARGS`（lines 13-16）：映射 X0–X5 到引數
- 產生 `__arm64_sys_*()` wrapper 函式

### 系統呼叫定義模式

```c
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{
    // 實現
}
```

`SYSCALL_DEFINE[0-6]` 巨集展開為帶 wrapper 的函式定義，處理型別轉換和追蹤標記。

### ARM64 專屬系統呼叫

定義於 `arch/arm64/kernel/sys.c`：

- `SYSCALL_DEFINE6(mmap, ...)` @ line 21 — ARM64 記憶體映射
- `SYSCALL_DEFINE1(arm64_personality, ...)` @ line 31 — ARM64 personality 檢查

## Semantics

### 系統呼叫分類

| 類別 | 示例 | 涵蓋範圍 |
|------|------|----------|
| 行程管理 | `fork`、`clone`、`execve`、`exit`、`wait4` | 行程建立/終止/同步 |
| 檔案 I/O | `open`、`read`、`write`、`close`、`lseek` | 檔案操作 |
| 記憶體管理 | `mmap`、`mprotect`、`brk`、`madvise` | 虛擬記憶體 |
| 網路 | `socket`、`bind`、`connect`、`sendmsg`、`recvmsg` | 網路通訊 |
| IPC | `pipe2`、`eventfd2`、`epoll_*`、`signalfd4` | 行程間通訊 |
| 信號 | `rt_sigaction`、`rt_sigprocmask`、`kill` | 信號處理 |
| 排程 | `sched_setscheduler`、`sched_yield`、`nanosleep` | 排程控制 |
| 安全 | `prctl`、`seccomp`、`capget/capset` | 安全控制 |
| 裝置控制 | `ioctl` | 裝置特定操作 |
| 檔案系統 | `mount`、`umount2`、`statfs`、`sync` | 檔案系統管理 |
| 擴充屬性 | `setxattr`、`getxattr`、`listxattr`、`removexattr` | 檔案擴充屬性 |
| I/O 多工 | `epoll_create1`、`epoll_ctl`、`epoll_pwait` | 事件驅動 I/O |
| io_uring | `io_uring_setup`、`io_uring_enter`、`io_uring_register` | 非同步 I/O |
| BPF | `bpf` | BPF 程式/映射管理 |
| inotify | `inotify_init1`、`inotify_add_watch`、`inotify_rm_watch` | 檔案系統監控 |

### 錯誤處理

系統呼叫回傳負值表示錯誤（`-errno`），C 庫將其轉換為設定 `errno` 並回傳 -1。

### 追蹤與稽核

`el0_svc_common()` 中整合了：
- `syscall_trace_enter/exit()` — ptrace/seccomp 攔截
- Stack randomization（`add_random_kstack_offset()`）
- MTE（Memory Tagging Extension）async fault 偵測

## Usage Examples

**使用者空間**（透過 C 庫）：
```c
int fd = open("/dev/binder", O_RDWR);           // __NR_openat
void *p = mmap(NULL, sz, PROT_READ, ...);        // __NR_mmap
int ret = ioctl(fd, BINDER_VERSION, &ver);       // __NR_ioctl
```

**核心內新增系統呼叫**：
```c
// include/uapi/asm-generic/unistd.h
#define __NR_newsyscall 470
__SYSCALL(__NR_newsyscall, sys_newsyscall)

// kernel/newsyscall.c
SYSCALL_DEFINE2(newsyscall, int, arg1, long, arg2)
{
    return do_something(arg1, arg2);
}
```

## Android-Specific Notes

### 無 Android 專屬系統呼叫 [upstream]

ACK 沒有新增 Android 專屬的系統呼叫。Android 的核心擴充主要透過以下機制而非新 syscall：
- **ioctl**：Binder IPC（`drivers/android/binder.c`）
- **Vendor hooks**：`include/trace/hooks/` tracepoint 框架
- **sysfs/procfs**：裝置和行程資訊暴露
- **BPF**：網路和安全策略

### Vendor Hook 整合

`kernel/bpf/syscall.c` 中使用 `trace_android_vh_check_bpf_syscall`，允許廠商在 BPF 系統呼叫進入點進行攔截，這是 ACK 中唯一直接與系統呼叫路徑整合的 vendor hook。

## Cross-References

- [Scheduler](../subsystems/scheduler.md) — `sched_*` 系統呼叫的實現
- [Memory Management](../subsystems/memory-management.md) — `mmap`、`mprotect` 等記憶體系統呼叫
- [Filesystems](../subsystems/filesystems.md) — 檔案 I/O 系統呼叫的 VFS 派發
- [IPC](../subsystems/ipc.md) — System V IPC 系統呼叫
- [Networking](../subsystems/networking.md) — Socket 系統呼叫
- [BPF](../concepts/bpf.md) — `bpf()` 系統呼叫
- [ioctl 介面](ioctl-interfaces.md) — `ioctl()` 系統呼叫的詳細分析
- [Netlink](netlink.md) — 透過 `socket()` 系統呼叫建立
- [sysfs/procfs](sysfs-procfs.md) — 透過 `read()`/`write()` 存取
