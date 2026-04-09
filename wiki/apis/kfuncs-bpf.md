---
type: api
direction: kernel-internal
related:
  - ../concepts/bpf.md
  - ../concepts/tracing-and-ftrace.md
  - ../subsystems/networking.md
  - ../subsystems/security.md
last_updated: 2026-04-09
---

# BPF kfuncs 介面

## Overview

BPF kfuncs（kernel functions）是 BPF 程式呼叫核心函式的新一代機制，取代傳統的 BPF helper functions。kfuncs 透過 BTF（BPF Type Format）自動描述函式簽名，無需為每個函式手動定義 `bpf_func_proto`。ACK 中共有 **45 個** kfunc 集合（`BTF_KFUNCS_START` 定義），分布在 **31 個**檔案中，另有 **157 個**傳統 BPF helper proto 和 **104 個** `BPF_CALL_N` 實現。

## Interface Definition

### 註冊基礎設施

定義於 `include/linux/btf.h`：

**kfunc 標記巨集**（line 89）：
```c
#define __bpf_kfunc __used __retain __noclone noinline
```

**kfunc 集合結構**（lines 121-125）：
```c
struct btf_kfunc_id_set {
    struct module *owner;
    struct btf_id_set8 *set;
    bool (*filter)(const struct bpf_prog *prog, u32 kfunc_id);
};
```

**註冊 API**（lines 582-583）：
```c
int register_btf_kfunc_id_set(enum bpf_prog_type prog_type,
                               const struct btf_kfunc_id_set *s);
```

### kfunc 屬性旗標

定義於 `include/linux/btf.h:18-81`：

| 旗標 | 行號 | 說明 |
|------|------|------|
| `KF_ACQUIRE` | 18 | 取得函式（回傳需 release 的參考） |
| `KF_RELEASE` | 19 | 釋放函式（釋放 acquire 取得的參考） |
| `KF_RET_NULL` | 20 | 可回傳 null 指標 |
| `KF_TRUSTED_ARGS` | 69 | 只接受受信任指標 |
| `KF_SLEEPABLE` | 70 | 函式可能休眠 |
| `KF_DESTRUCTIVE` | 71 | 執行破壞性操作 |
| `KF_RCU` | 72 | RCU 或受信任指標引數 |
| `KF_ITER_NEW` | 74 | 迭代器建立函式 |
| `KF_ITER_NEXT` | 75 | 迭代器推進函式 |
| `KF_ITER_DESTROY` | 76 | 迭代器銷毀函式 |
| `KF_RCU_PROTECTED` | 77 | RCU 保護 |
| `KF_FASTCALL` | 78 | FastCall 協定 |
| `KF_ARENA_RET` | 79 | 回傳 arena 指標 |
| `KF_ARENA_ARG1/ARG2` | 80-81 | Arena 指標引數 |

### 主要 kfunc 集合

#### 1. 通用核心 kfuncs — `kernel/bpf/helpers.c`

**generic_btf_ids**（lines 4394-4440，47 個 kfuncs）：

核心物件生命週期管理：
- `bpf_obj_new_impl` / `bpf_obj_drop_impl` — 物件分配/釋放（KF_ACQUIRE / KF_RELEASE）
- `bpf_percpu_obj_new_impl` / `bpf_percpu_obj_drop_impl` — Per-CPU 物件
- `bpf_refcount_acquire_impl` — 增加參考計數（KF_ACQUIRE）

行程/cgroup 操作：
- `bpf_task_acquire` / `bpf_task_release` — 取得/釋放 task 參考
- `bpf_task_from_pid` — 由 PID 查找 task（KF_ACQUIRE | KF_RET_NULL）
- `bpf_cgroup_acquire` / `bpf_cgroup_release` — cgroup 參考管理
- `bpf_cgroup_from_id` — 由 ID 查找 cgroup

資料結構操作：
- `bpf_list_push_front/back` / `bpf_list_pop_front/back` — 鏈結串列
- `bpf_rbtree_add` / `bpf_rbtree_remove` / `bpf_rbtree_first` — 紅黑樹

**common_btf_ids**（lines 4456-4544，88+ 個 kfuncs）：

迭代器操作（KF_ITER_NEW/NEXT/DESTROY）：
- `bpf_iter_num_new/next/destroy` — 數值迭代器
- `bpf_iter_task_new/next/destroy` — 行程迭代器
- `bpf_iter_css_task_new/next/destroy` — CSS 行程迭代器

動態指標（dynptr）操作：
- `bpf_dynptr_adjust` / `bpf_dynptr_clone` / `bpf_dynptr_is_null`
- `bpf_dynptr_is_rdonly` / `bpf_dynptr_size` / `bpf_dynptr_slice`

字串操作（KF_FASTCALL）：
- `bpf_str_cmp` — 字串比較

#### 2. CPU Mask kfuncs — `kernel/bpf/cpumask.c`

（lines 477-504，28 個 kfuncs）

- `bpf_cpumask_create` / `bpf_cpumask_release` / `bpf_cpumask_acquire` — 生命週期
- `bpf_cpumask_set_cpu` / `bpf_cpumask_clear_cpu` / `bpf_cpumask_test_cpu` — 位元操作
- `bpf_cpumask_and` / `bpf_cpumask_or` / `bpf_cpumask_xor` — 集合運算
- `bpf_cpumask_any_distribute` / `bpf_cpumask_any_and_distribute` — CPU 選擇

註冊目標：`BPF_PROG_TYPE_TRACING`

#### 3. Arena kfuncs — `kernel/bpf/arena.c`

（lines 620-624，3 個 kfuncs）

- `bpf_arena_alloc_pages` — 分配 arena 頁面（KF_ARENA_RET | KF_SLEEPABLE）
- `bpf_arena_free_pages` — 釋放 arena 頁面（KF_ARENA_ARG2 | KF_SLEEPABLE）
- `bpf_arena_reserve_pages` — 保留 arena 頁面

#### 4. Crypto kfuncs — `kernel/bpf/crypto.c`

（lines 348-362，兩個集合共 5 個 kfuncs）

初始化集合：
- `bpf_crypto_ctx_create` / `bpf_crypto_ctx_release` / `bpf_crypto_ctx_acquire`

操作集合：
- `bpf_crypto_decrypt` / `bpf_crypto_encrypt`

#### 5. 網路 kfuncs — `net/core/filter.c`

（lines 12442-12465，6 個集合）

- `bpf_kfunc_check_set_skb` — Socket buffer 操作
- `bpf_kfunc_check_set_xdp` — XDP 操作
- `bpf_kfunc_check_set_sock_addr` — Socket 位址操作
- `bpf_kfunc_check_set_tcp_reqsk` — TCP request socket
- `bpf_kfunc_check_set_sock_ops` — Socket 操作
- 多個程式類型註冊：SCHED_CLS、XDP、CGROUP_SKB 等

#### 6. TCP 壅塞控制 kfuncs — `net/ipv4/bpf_tcp_ca.c`

（lines 191-197，5 個 kfuncs）

- `tcp_reno_ssthresh` — Reno 慢啟動閾值
- `tcp_reno_cong_avoid` — Reno 壅塞避免
- `tcp_slow_start` — 慢啟動演算法
- `tcp_cong_avoid_ai` — 加法增加

#### 7. 其他 kfunc 集合

| 檔案 | kfunc 數 | 用途 |
|------|----------|------|
| `kernel/bpf/rqspinlock.c` | ~5 | 佇列化自旋鎖 |
| `kernel/bpf/map_iter.c` | ~3 | Map 迭代器 |
| `net/core/xdp.c` | ~5 | XDP metadata |
| `net/netfilter/` | ~10 | Netfilter 整合 |
| `drivers/hid/bpf/` | ~5 | HID 裝置 |
| `kernel/trace/` | ~5 | 追蹤 |
| `fs/` | ~5 | 檔案系統 |

### 傳統 BPF Helper Functions

kfuncs 之前的機制，仍廣泛使用：

**定義模式**：
```c
BPF_CALL_N(bpf_helper_name, type1, arg1, type2, arg2, ...)

const struct bpf_func_proto bpf_helper_name_proto = {
    .func      = bpf_helper_name,
    .gpl_only  = true,
    .ret_type  = RET_INTEGER,
    .arg1_type = ARG_PTR_TO_CTX,
    // ...
};
```

**統計**：
- 104 個 `BPF_CALL_N` 呼叫（kernel/bpf/ 中各種 arity）
- 157 個 `const struct bpf_func_proto` 定義
- 主要集中在 `kernel/bpf/helpers.c`、`net/core/filter.c`

## Semantics

### kfunc vs Helper 的差異

| 特性 | kfunc | Helper |
|------|-------|--------|
| 型別描述 | BTF 自動推導 | 手動 bpf_func_proto |
| 註冊方式 | btf_kfunc_id_set | 程式類型的 func_proto 列表 |
| 可擴充性 | 模組可註冊新 kfunc | 需修改核心 |
| 驗證器整合 | 透過 BTF 旗標 | 透過 arg_type/ret_type |
| 生命週期語意 | KF_ACQUIRE/KF_RELEASE | ARG_PTR_TO_ALLOC_MEM 等 |

### 驗證器檢查

BPF 驗證器（`kernel/bpf/verifier.c`）對 kfunc 呼叫進行：
- 引數型別匹配（透過 BTF）
- 取得/釋放配對追蹤
- 休眠能力檢查（KF_SLEEPABLE vs 程式類型）
- RCU 保護驗證
- 迭代器狀態機驗證（new → next* → destroy）

## Android-Specific Notes

### Android BPF 整合 [android]

- `kernel/bpf/syscall.c` 使用 `trace_android_vh_check_bpf_syscall` vendor hook，允許廠商在 BPF 系統呼叫層面進行攔截
- Android 網路和安全子系統廣泛使用 BPF 程式，kfuncs 提供更靈活的核心功能存取

### 無 Android 專屬 kfuncs

目前 ACK 中未發現 Android 專屬的 kfunc 註冊。Android 的 BPF 擴充主要透過 vendor hooks 而非 kfuncs。

## Cross-References

- [BPF](../concepts/bpf.md) — BPF 子系統全面分析（33 種程式類型、30 種映射類型）
- [追蹤與 ftrace](../concepts/tracing-and-ftrace.md) — kfunc 與追蹤基礎設施的關係
- [Networking](../subsystems/networking.md) — 網路 kfuncs 的應用場景
- [Security](../subsystems/security.md) — BPF LSM 與安全 kfuncs
- [Vendor Hooks](../concepts/vendor-hooks.md) — Android 的替代擴充機制
- [Syscalls](syscalls.md) — BPF 系統呼叫入口
