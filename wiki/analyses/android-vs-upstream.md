---
type: analysis
question: "ACK 相對於上游 Linux 的所有修改和擴展的全面比較"
pages_consulted:
  - subsystems/scheduler.md
  - subsystems/memory-management.md
  - subsystems/networking.md
  - subsystems/filesystems.md
  - subsystems/block-layer.md
  - subsystems/security.md
  - subsystems/ipc.md
  - concepts/gki.md
  - concepts/vendor-hooks.md
  - concepts/rust-in-kernel.md
  - entities/binder.md
  - entities/ashmem.md
  - entities/dmabuf-heaps.md
  - entities/debug-kinfo.md
created: 2026-04-09
---

# Android vs Upstream：ACK 差異全面分析

## 總覽

Android Common Kernel (ACK) 基於 Linux 6.19-rc8，目標是在維持最大上游相容性的同時，為 Android 裝置提供必要的平台功能。本分析系統性地比較 ACK 與上游 Linux 的差異，按修改程度分類。

## 差異分類框架

我們將 ACK 的修改分為四個層級：

1. **Android 專屬元件** — 上游完全不存在的程式碼
2. **Android 補丁** — 對上游程式碼的修改或擴展
3. **配置差異** — 相同程式碼但不同的 Kconfig 啟用選擇
4. **完全上游** — 無任何 Android 修改

## 1. Android 專屬元件

這些元件在上游 Linux 中完全不存在：

### 核心 IPC — Binder (`drivers/android/`)

- `binder.c` (7,374 行) + Rust 實現 (~7,400 行)：Android 的核心跨行程通訊機制
- `binderfs.c` (786 行)：動態 Binder 裝置管理檔案系統
- 三裝置架構（binder/hwbinder/vndbinder）、凍結/解凍、優先權繼承
- **上游狀態**：從未合入上游，是 Android staging 畢業後的獨立維護元件

### Vendor Hook 框架 (`include/trace/hooks/`, `drivers/android/vendor_hooks.c`)

- 130 個 hooks、29 個標頭檔、93 行匯出驅動
- 基於 tracepoint 基礎設施的擴展框架
- **上游狀態**：Android 專屬設計，上游有 tracepoint 但無 vendor hook 概念

### 增量檔案系統 — IncFS (`fs/incfs/`, 21 檔案)

- 支援 Android 應用程式增量安裝（Play Store streaming install）
- **上游狀態**：從未提交上游

### Debug Kinfo (`drivers/staging/android/debug_kinfo.c`)

- 185 行核心 + 69 行標頭：匯出 kallsyms 位址和段佈局供 bootloader crash dump
- **上游狀態**：Android staging 元件

### Memfd-Ashmem 相容層 (`mm/memfd-ashmem-shim.c`, 214 行)

- 將舊版 ashmem ioctl 轉換為 memfd 呼叫
- **上游狀態**：Android 相容性膠層

### GKI 基礎設施

- `gki/` 目錄：模組符號保護清單、ABI 定義
- `kernel/module/gki_module.c`：KMI 符號存取控制
- `init/Kconfig.gki`：隱藏配置（預先啟用驅動相依選項）
- **上游狀態**：Android 專屬構建系統元件

## 2. Android 補丁（對上游程式碼的修改）

### 記憶體管理

| 修改 | 位置 | 說明 |
|------|------|------|
| MGLRU 預設啟用 | `mm/vmscan.c` | 上游預設關閉，ACK 預設開啟 Multi-Gen LRU |
| Memfd LUO | `mm/memfd.c` | Live Update Orchestrator，kexec 期間保留 memfd |
| Vendor hooks (2) | `mm/vmscan.c:2579`, `mm/util.c:592` | 回收比率控制、mmap 安全檢查 |

### 檔案系統

| 修改 | 位置 | 說明 |
|------|------|------|
| FUSE passthrough | `fs/fuse/backing.c` | 繞過 userspace daemon 直接 I/O，取代 sdcardfs |
| fscrypt Android 相容 | `fs/crypto/keyring.c:782-788` | 舊版金鑰推導模式支援 |
| f2fs SQLite 調校 | `fs/f2fs/file.c:3336` | 原子寫入語意支援 |
| Vendor hook (1) | `fs/open.c:939` | `vfs_open()` 檔案開啟攔截 |

### Block Layer

| 修改 | 位置 | 說明 |
|------|------|------|
| DM Default Key | `bio.c:274` | `bi_skip_dm_default_key` 欄位，inline 加密 fallback |

### 安全

| 修改 | 位置 | 說明 |
|------|------|------|
| Binder LSM hooks (4) | `selinux/hooks.c:7312-7315` | Binder IPC 授權 hooks |
| Vendor hooks (5) | `avc.h`, `selinux.h` | AVC 快取監控（restricted） |

### 排程器

| 修改 | 位置 | 說明 |
|------|------|------|
| Vendor hooks (78) | `kernel/sched/` 多處 | 任務遷移、CPU 選擇、負載均衡等擴展點 |
| `task_struct` 擴展 | `include/linux/sched.h` | 96 bytes vendor/OEM data 欄位 |

### 網路

| 修改 | 位置 | 說明 |
|------|------|------|
| QRTR 支援 | `net/qrtr/` (2,612 行) | Qualcomm IPC Router（上游已合入但 Android 額外維護） |
| Vendor hooks (~1) | `include/trace/hooks/net.h` | 封包處理攔截 |

### Cgroup

| 修改 | 位置 | 說明 |
|------|------|------|
| Vendor hooks (3) | `kernel/cgroup/`, `kernel/sched/` | cgroup_attach、cpu_cgroup_attach、cpu_cgroup_online |

## 3. 配置差異（GKI defconfig vs 上游 defconfig）

ACK 的 `gki_defconfig` 與上游典型配置有顯著差異：

**Android 啟用但上游通常關閉**：CONFIG_ANDROID_BINDER_IPC、CONFIG_ANDROID_VENDOR_HOOKS、CONFIG_MODULE_SIG_PROTECT、CONFIG_GENDWARFKSYMS、CONFIG_LRU_GEN（MGLRU）、CONFIG_RUST（Rust 核心支援）

**安全強化（GKI 全部啟用）**：CONFIG_CFI_CLANG、CONFIG_SHADOW_CALL_STACK、CONFIG_INIT_ON_ALLOC_DEFAULT_ON、CONFIG_INIT_STACK_ALL_ZERO、CONFIG_FORTIFY_SOURCE、CONFIG_SLAB_FREELIST_HARDENED、CONFIG_HARDENED_USERCOPY

**LSM 堆疊**：`landlock,lockdown,yama,loadpin,safesetid,selinux,smack,tomoyo,apparmor,ipe,bpf` — 比大多數上游發行版啟用更多 LSM

## 4. 完全上游（無修改）

以下子系統在 ACK 中與上游 Linux 完全一致：

- **System V IPC** (`ipc/`) — 9,889 行，零 Android 修改、零 vendor hooks
- **Block Layer 核心** (`block/`) — 64,528 行，僅一個 bio 欄位新增
- **鎖定原語** — qspinlock、mutex、rwsem 等完全上游
- **中斷處理** — IRQ 框架完全上游
- **RCU** — Tree RCU/SRCU 完全上游
- **追蹤/ftrace** — tracepoint 基礎設施完全上游（vendor hooks 建立在其上但不修改它）

## 修改量化

| 類別 | 估計修改行數 | 佔核心總程式碼比例 |
|------|-------------|-------------------|
| Android 專屬元件 | ~20,000 行 | < 0.1% |
| Android 補丁 | ~5,000 行 | < 0.03% |
| Vendor hook 插入點 | ~1,000 行 | < 0.005% |
| **合計** | **~26,000 行** | **< 0.15%** |

（對比核心總量：36,348 個 .c 檔案、數千萬行程式碼）

## 設計哲學

ACK 的修改策略遵循幾個明確原則：

1. **最小侵入**：Android 專屬程式碼集中在 `drivers/android/` 和獨立檔案，極少修改上游熱路徑
2. **可觀察而非可覆寫**：安全相關 hooks 僅提供觀察能力，不允許廠商改變安全決策
3. **擴展點優於分支**：vendor hooks 和 BPF 讓廠商在不 fork 核心的前提下差異化
4. **漸進廢棄**：ashmem → memfd、Ion → DMA-BUF heaps、sdcardfs → FUSE passthrough
5. **Rust 安全替代**：Binder 的 Rust 重新實現代表記憶體安全的長期方向

## 交叉參考

- [GKI 架構](../concepts/gki.md) — KMI 穩定性機制
- [Vendor Hook 框架](../concepts/vendor-hooks.md) — Hook 設計與實現
- [Vendor Hooks 分佈分析](vendor-hooks-distribution.md) — Hook 統計詳情
- [Binder](../entities/binder.md) — 最大的 Android 專屬元件
- [Ashmem](../entities/ashmem.md) — 廢棄過渡範例
- [Rust in Kernel](../concepts/rust-in-kernel.md) — Rust Binder 替代實現
