---
type: analysis
question: "Vendor hooks 在 ACK 各子系統的分佈、設計模式與擴展策略為何？"
pages_consulted:
  - concepts/vendor-hooks.md
  - entities/vendor-hooks-driver.md
  - subsystems/scheduler.md
  - subsystems/memory-management.md
  - subsystems/networking.md
  - subsystems/filesystems.md
  - subsystems/block-layer.md
  - subsystems/security.md
  - subsystems/ipc.md
  - entities/cgroup-controllers.md
created: 2026-04-09
---

# Vendor Hooks 跨子系統分佈分析

## 總覽

Android Common Kernel (ACK) 透過 vendor hook 框架讓 SoC 廠商在不修改核心原始碼的前提下擴展核心行為。本分析彙整所有已建檔子系統中的 vendor hook，揭示其分佈模式、設計選擇與架構含義。

## 全域統計

ACK 共定義 **130 個 vendor hooks**，分佈在 **29 個標頭檔** (`include/trace/hooks/`)，由 `drivers/android/vendor_hooks.c` 匯出其中 **50 個** `EXPORT_TRACEPOINT_SYMBOL_GPL`。

## 按子系統分佈

| 子系統 | Unrestricted (`android_vh_*`) | Restricted (`android_rvh_*`) | 合計 | 佔比 |
|--------|-------------------------------|------------------------------|------|------|
| **排程器** | 11 | 67 | **78** | 60.0% |
| **UFS 儲存** | 8 | 1 | **9** | 6.9% |
| **SELinux / AVC** | 0 | 5 | **5** | 3.8% |
| **Cgroup 控制器** | 1 | 2 | **3** | 2.3% |
| **IOMMU** | 2 | 1 | **3** | 2.3% |
| **Syscall 檢查** | 3 | 0 | **3** | 2.3% |
| **記憶體管理** | 1 | 1 | **2** | 1.5% |
| **網路** | ~1 | 0 | **~1** | 0.8% |
| **檔案系統** | 1 | 0 | **1** | 0.8% |
| **其他** (cpuidle, cpufreq, printk, power, gic 等) | ~25 | 0 | **~25** | 19.2% |
| **Block Layer** | 0 | 0 | **0** | 0% |
| **IPC (System V)** | 0 | 0 | **0** | 0% |

## 分佈模式分析

### 排程器獨佔六成

排程器子系統擁有 78 個 hooks（佔 60%），其中 **67 個為 restricted**，這反映了排程是 Android 廠商差異化的核心戰場。涵蓋的熱點包括任務遷移決策、CPU 選擇、負載均衡策略、RT 排程參數、以及 PELT 負載追蹤。restricted 型別確保每個 hook 僅有一個廠商實作，避免多廠商模組衝突導致排程不確定性。

### 安全子系統全部 restricted

Security 的 5 個 hooks 均為 `DECLARE_RESTRICTED_HOOK`，且全數集中在 SELinux AVC 快取操作（insert/delete/replace/lookup）和初始化狀態通知。這顯示 Google 對安全性採取最保守的擴展策略——廠商只能「觀察」安全決策，不能覆寫。

### Block Layer 與 IPC 零 hooks

Block layer 和 System V IPC 完全沒有 vendor hooks。Block layer 的廠商擴展在更低層的 UFS 控制器驅動（9 hooks）和 IOMMU（3 hooks）提供，而非在通用 block 層。System V IPC 在 Android 中幾乎不使用（Android 以 Binder 為主），因此無擴展需求。

### 記憶體管理極度克制

整個 mm/ 子系統僅 2 個 hooks：一個控制 anon/file 回收比率（restricted），一個檢查 mmap 操作（unrestricted）。考慮到 mm/ 有 140,000+ 行程式碼，這代表 Google 刻意限制廠商對記憶體管理核心邏輯的干預，優先透過 userspace（如 lmkd）或 BPF 程式來調校。

## 兩種 Hook 型別的設計考量

**Unrestricted (`DECLARE_HOOK`)**：基於 tracepoint，允許多個探針同時註冊。用於通知性質的事件（如 cgroup attach、file open 檢查、printk 攔截），多個廠商模組共存不會衝突。

**Restricted (`DECLARE_RESTRICTED_HOOK`)**：限制僅一個探針註冊，使用 `static_call` 派發。用於必須產出單一決策的場景（如排程 CPU 選擇、AVC 快取管理），多個實作會造成語義矛盾。static_call 同時提供效能優勢——未註冊時編譯為 NOP，註冊後為直接呼叫（無間接跳轉開銷）。

## 效能影響

未啟用的 hooks 使用 `static_branch_unlikely()` 編譯為 NOP 指令，對核心熱路徑零開銷。啟用後 restricted hooks 的 static_call 等同直接函式呼叫，unrestricted hooks 走標準 tracepoint 迭代路徑。

## 替代擴展機制

對於未提供 vendor hooks 的子系統，ACK 提供其他擴展方式：

- **sched_ext (BPF)**：排程器的 eBPF 擴展，允許用 BPF 程式定義完整排程策略，比 vendor hooks 更靈活
- **BPF LSM**：安全子系統的動態 BPF 安全策略，GKI defconfig 預設啟用
- **Netfilter / nf_tables**：網路子系統的封包處理擴展
- **Device Mapper**：Block layer 的 dm-default-key inline 加密

## 結論

Vendor hooks 的分佈反映了 Android 生態系中「哪裡需要硬體差異化」的策略地圖。排程器佔六成說明 SoC 廠商的核心價值主張在於效能調校（大小核排程、功耗管理）。安全子系統僅開放觀察性 hooks 體現了 Google 對安全邊界的嚴格管控。記憶體管理和 Block layer 的低/零 hook 數量則表明這些領域的擴展需求已被 BPF、userspace 或更低層驅動 hooks 滿足。

## 交叉參考

- [Vendor Hook 框架](../concepts/vendor-hooks.md)
- [Vendor Hooks 匯出驅動](../entities/vendor-hooks-driver.md)
- [排程器](../subsystems/scheduler.md) — 78 個 hooks 詳細列表
- [安全子系統](../subsystems/security.md) — 5 個 restricted hooks
- [GKI 架構](../concepts/gki.md) — KMI 穩定性與模組符號保護
- [Cgroup 控制器](../entities/cgroup-controllers.md) — 3 個 hooks
