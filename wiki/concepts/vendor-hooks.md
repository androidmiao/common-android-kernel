---
type: concept
scope: kernel-wide
related:
  - ../concepts/gki.md
  - ../concepts/tracing-and-ftrace.md
  - ../concepts/module-system.md
  - ../subsystems/scheduler.md
last_updated: 2026-04-08
---

# Vendor Hook 框架

## 概述

Vendor Hook 框架是 Android GKI 架構中允許廠商模組修改核心行為的機制。它基於 Linux tracepoint 基礎設施構建，但針對 Android 的需求進行了定制——提供受限制的 hook 以確保安全性和穩定性。廠商不需要直接修改核心原始碼，而是透過註冊 hook 回調函數，在核心的特定執行點插入自定義邏輯。

整個框架定義了 **130 個 hook**，分布在 **29 個標頭檔案**中，涵蓋排程、記憶體、安全、電源管理、存儲等子系統。

## 機制

### 核心巨集與基礎設施

**關鍵檔案**：
- `drivers/android/vendor_hooks.c`（94行）— Hook 點初始化，tracepoint 符號匯出
- `include/trace/hooks/vendor_hooks.h`（131行）— 核心巨集定義
- `kernel/tracepoint.c` — Tracepoint 管理與 hook 註冊實現

**兩種 Hook 類型**：

#### DECLARE_HOOK（無限制 Hook）

```c
#define DECLARE_HOOK DECLARE_TRACE_EVENT   // vendor_hooks.h:16
```

- 命名約定：`android_vh_*`（VH = Vendor Hook）
- 可由多個探針（probe）同時註冊
- 無條件執行——每次到達 hook 點都會呼叫
- 適用於觀察性用途（監控、日誌）

#### DECLARE_RESTRICTED_HOOK（受限制 Hook）

```c
#define DECLARE_RESTRICTED_HOOK(name, proto, args, cond)   // vendor_hooks.h:62-63
```

- 命名約定：`android_rvh_*`（RVH = Restricted Vendor Hook）
- **限制為最多一個已註冊探針**
- 包含第四個參數 `cond`（條件），控制 hook 是否執行
- 使用 `static_call` 機制實現直接呼叫，避免間接跳轉開銷
- **不可取消註冊**（`vendor_hooks.h:115` 註解）
- 適用於性能敏感的修改（排程、記憶體）

### Hook 定義實現

**`__DEFINE_HOOK_EXT` 巨集**（`vendor_hooks.h:22-52`）負責：

1. 建立 tracepoint 字串表項
2. 聲明 static call key
3. 定義 tracepoint 迭代器函數
4. 建立 `struct tracepoint` 結構體，包含：
   - `.name` — hook 名稱
   - `.key` — 靜態鑰匙（零開銷條件檢查）
   - `.static_call_key` — 優化的直接呼叫
   - `.funcs` — 已註冊的探針陣列

**Tracepoint 迭代器**（`vendor_hooks.h:38-51`）：

```c
int __traceiter_##_name(void *__data, proto) {
    struct tracepoint_func *it_func_ptr;
    it_func_ptr = (&__tracepoint_##_name)->funcs;
    do {
        __data = (it_func_ptr)->data;
        ((void(*)(void *, proto))(it_func_ptr->func))(__data, args);
    } while (READ_ONCE((++it_func_ptr)->func));
    return 0;
}
```

### 受限制 Hook 的呼叫路徑

**條件檢查**（`vendor_hooks.h:83-91`）：

```c
if (static_branch_unlikely(&__tracepoint_##name.key))
    DO_RESTRICTED_HOOK(name, TP_ARGS(args), TP_CONDITION(cond));
```

- `static_branch_unlikely()` — 當沒有探針註冊時，編譯器將此路徑優化為 NOP
- `DO_RESTRICTED_HOOK` — 先檢查 `cond`，再透過 `static_call` 直接呼叫探針

**靜態呼叫優化**（`vendor_hooks.h:71-79`）：

```c
#define __DO_RESTRICTED_HOOK_CALL(name, args) \
    do { \
        struct tracepoint_func *it_func_ptr; \
        it_func_ptr = (&__tracepoint_##name)->funcs; \
        if (it_func_ptr) { \
            __data = (it_func_ptr)->data; \
            static_call(tp_func_##name)(__data, args); \
        } \
    } while (0)
```

### 註冊機制

**`android_rvh_probe_register()`**（`kernel/tracepoint.c`）：

```c
int android_rvh_probe_register(struct tracepoint *tp, void *probe, void *data) {
    // 驗證：如果已啟用靜態鑰匙，不允許同時傳遞 data
    if (WARN_ON(static_key_enabled(&tp->key) && data))
        return -EINVAL;
    mutex_lock(&tracepoints_mutex);
    ret = android_rvh_add_func(tp, &tp_func);
    mutex_unlock(&tracepoints_mutex);
    return ret;
}
EXPORT_SYMBOL_GPL(android_rvh_probe_register);
```

所有 hook 透過 `EXPORT_TRACEPOINT_SYMBOL_GPL()` 匯出（`vendor_hooks.c:44-93`），使外部模組可以註冊。

### 禁用模式

當 `CONFIG_TRACEPOINTS` 或 `CONFIG_ANDROID_VENDOR_HOOKS` 未啟用時（`vendor_hooks.h:125-130`）：

```c
#define DECLARE_HOOK DECLARE_EVENT_NOP   // Hook 變成 NOP，零代碼開銷
```

## Hook 完整分類

### 按子系統統計

| 子系統 | 標頭檔案 | 無限制 | 受限制 | 總計 |
|--------|----------|--------|--------|------|
| 排程 | `sched.h` | 11 | 67 | **78** |
| 存儲 (UFS) | `ufshcd.h` | 8 | 1 | **9** |
| 安全 (AVC) | `avc.h` | 0 | 4 | **4** |
| Syscall 檢查 | `syscall_check.h` | 3 | 0 | **3** |
| Cgroup | `cgroup.h` | 1 | 2 | **3** |
| IOMMU | `iommu.h` | 2 | 1 | **3** |
| Printk | `printk.h` | 4 | 0 | **4** |
| CPU Idle | `cpuidle.h` | 2 | 0 | **2** |
| CPU Idle PSCI | `cpuidle_psci.h` | 2 | 0 | **2** |
| CPUfreq | `cpufreq.h` | 1 | 1 | **2** |
| Remoteproc | `remoteproc.h` | 2 | 0 | **2** |
| Epoch | `epoch.h` | 2 | 0 | **2** |
| SELinux | `selinux.h` | 0 | 1 | **1** |
| 記憶體 (vmscan) | `vmscan.h` | 0 | 1 | **1** |
| 網路 | `net.h` | 1 | 0 | **1** |
| GIC | `gic.h` | 1 | 0 | **1** |
| GIC v3 | `gic_v3.h` | 0 | 1 | **1** |
| Timer | `timer.h` | 1 | 0 | **1** |
| Signal | `signal.h` | 1 | 0 | **1** |
| Debug | `debug.h` | 1 | 0 | **1** |
| Sys | `sys.h` | 1 | 0 | **1** |
| SysRq Crash | `sysrqcrash.h` | 1 | 0 | **1** |
| PM Domain | `pm_domain.h` | 1 | 0 | **1** |
| Reboot | `reboot.h` | 0 | 1 | **1** |
| MPAM | `mpam.h` | 1 | 0 | **1** |
| WQ Lockup | `wqlockup.h` | 1 | 0 | **1** |
| FPSIMD | `fpsimd.h` | 1 | 0 | **1** |
| MM | `mm.h` | 0 | 3 | **3**（已註釋） |
| **總計** | **29 檔案** | **56** | **74** | **130** |

### 排程 Hook 重點範例

排程子系統擁有最多的 hook（78 個），涵蓋：

- `android_rvh_select_task_rq_fair` — CPU 選擇優化
- `android_rvh_enqueue_task` / `android_rvh_dequeue_task` — 任務佇列管理
- `android_rvh_schedule` — 排程決策點
- `android_rvh_place_entity` — CFS 實體放置
- `android_rvh_cpu_overutilized` — CPU 過載判定

## 使用模式

### 廠商模組註冊範例

```c
// 廠商模組中的 hook 註冊
#include <trace/hooks/sched.h>

static void my_enqueue_hook(void *data, struct rq *rq,
                            struct task_struct *p, int flags) {
    // 自定義入隊邏輯
}

static int __init my_module_init(void) {
    return register_trace_android_rvh_enqueue_task(
        my_enqueue_hook, NULL);
}
```

### 與 Tracepoint 的關係

Vendor Hook 是 Linux tracepoint 基礎設施的 Android 擴展。繼承結構：

1. **基礎**：`struct tracepoint`（`include/linux/tracepoint-defs.h:39-48`）
2. **中層**：Trace events（`TRACE_EVENT` 巨集）
3. **頂層**：Vendor Hooks（`DECLARE_HOOK` = `DECLARE_TRACE_EVENT`）

## Android 相關性

### 為何選擇 Hook 而非直接修補

| 比較面向 | Hook Framework | 直接修補 | 動態 kprobes |
|----------|---------------|---------|-------------|
| 維護成本 | 低（清晰擴展點）| 高（核心修改）| 中（脆弱符號）|
| 安全性 | 高（受限制）| 低（無限制）| 中（需特殊權限）|
| 性能 | 最佳（靜態呼叫）| 最佳（內聯）| 差（動態開銷）|
| 多廠商支持 | 是 | 否（衝突）| 是（複雜）|
| ABI 穩定性 | 高（固定介面）| 低（隨版變化）| 中（符號依賴）|

### 性能特性

- **禁用時**：`static_branch_unlikely()` 將 hook 路徑編譯為 NOP，**零開銷**
- **啟用時**：受限制 hook 使用 `static_call`，等同直接函數呼叫，無間接跳轉
- **無限制 hook**：使用標準 tracepoint 迭代器，支持多探針但開銷稍高

## 交叉參考

- [GKI](gki.md) — Vendor Hook 存在的架構基礎
- [Tracing 與 ftrace](tracing-and-ftrace.md) — Hook 底層使用的 tracepoint 基礎設施
- [Module 系統](module-system.md) — 廠商模組如何載入與存取 hook
- [Scheduler](../subsystems/scheduler.md) — 包含 78 個 Vendor Hook 的詳細分析
