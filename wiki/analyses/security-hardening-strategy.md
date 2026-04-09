---
type: analysis
question: "ACK 的安全強化策略：從 LSM 堆疊到編譯器保護的多層防禦體系"
pages_consulted:
  - subsystems/security.md
  - concepts/gki.md
  - concepts/bpf.md
  - concepts/vendor-hooks.md
  - entities/binder.md
created: 2026-04-09
---

# ACK 安全強化策略分析

## 總覽

Android Common Kernel 在上游 Linux 安全機制的基礎上，通過 GKI defconfig 啟用了一套全面的多層安全防禦體系。本分析從攻擊面防禦的角度，系統性地審視 ACK 的安全強化措施。

## 安全防禦層次

### 第一層：強制存取控制 (MAC)

**SELinux** 作為 Android 的主要 MAC 引擎（25,424 行、217 hooks），在以下關鍵路徑執行存取控制：

- **檔案存取**：`selinux_inode_permission()` — 每次檔案操作
- **行程操作**：`selinux_task_*()` — 行程建立、信號、ptrace
- **網路存取**：`selinux_socket_*()` — socket 建立、綁定、連線
- **Binder IPC**：4 個專用 hooks — `binder_set_context_mgr`、`binder_transaction`、`binder_transfer_binder`、`binder_transfer_file`
- **裝置存取**：`selinux_inode_getattr()`、`selinux_file_open()`

**效能最佳化**：AVC（Access Vector Cache）使用 hash table + LRU 快取存取決策，避免每次都查詢完整 policy database。5 個 Android restricted vendor hooks 允許廠商監控 AVC 行為但不能覆寫決策。

### 第二層：補充 LSM 模組

GKI 啟用的 LSM 堆疊順序：`landlock,lockdown,yama,loadpin,safesetid,selinux,smack,tomoyo,apparmor,ipe,bpf`

| LSM | Hooks 數 | 用途 | Android 重要性 |
|-----|---------|------|---------------|
| **SELinux** | 217 | 主要 MAC | 核心安全基礎 |
| **Landlock** | 34 | 非特權沙箱 | 應用程式自限制 |
| **SafeSetID** | 4 | UID/GID 轉換控制 | 系統服務權限隔離 |
| **BPF LSM** | 動態 | eBPF 安全策略 | 動態安全規則 |
| **Yama** | 4 | ptrace 限制 | 防止行程注入 |
| **LoadPin** | 3 | 載入來源限制 | 確保模組/韌體來源 |
| **Lockdown** | 1 | 核心完整性 | 防止運行時修改核心 |

### 第三層：編譯器安全功能

ACK 的 GKI defconfig 啟用了全套編譯器級安全保護：

**控制流完整性 (CFI)**
- `CONFIG_CFI_CLANG=y`：防止間接呼叫被重導向到任意地址。LLVM 在編譯期間為每個間接呼叫位點插入類型檢查，執行時違規觸發 panic。

**Shadow Call Stack (SCS)**
- `CONFIG_SHADOW_CALL_STACK=y`：在獨立的 shadow stack 中保存返回地址，防止 ROP（Return-Oriented Programming）攻擊。利用 ARM64 的 x18 暫存器指向 shadow stack。

**堆疊初始化**
- `CONFIG_INIT_STACK_ALL_ZERO=y`：所有堆疊變數自動初始化為零，消除未初始化記憶體洩漏。

**Source Fortification**
- `CONFIG_FORTIFY_SOURCE=y`：加強版 `memcpy()`/`strcpy()` 等函式，在編譯期和執行時檢測緩衝區溢出。

**Hardened Usercopy**
- `CONFIG_HARDENED_USERCOPY=y`：`copy_to_user()`/`copy_from_user()` 驗證源/目標位址合法性，防止核心記憶體洩漏。

### 第四層：記憶體分配器強化

**堆積保護**
- `CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y`：所有 `kmalloc()`/`alloc_pages()` 分配的記憶體自動清零
- `CONFIG_SLAB_FREELIST_HARDENED=y`：SLUB freelist 指標加密（XOR with per-cache random key），防止 use-after-free 攻擊利用 freelist 指標
- `CONFIG_SLAB_FREELIST_RANDOM=y`：freelist 隨機化初始順序，增加攻擊不確定性

**結構體隨機化**
- `CONFIG_RANDSTRUCT=y`（若啟用）：編譯期隨機化關鍵結構體的欄位排列，防止依賴特定記憶體偏移的攻擊

### 第五層：模組與完整性保護

**GKI 模組簽署**
- `CONFIG_MODULE_SIG=y`：所有 GKI 模組必須經 Google 簽署
- `CONFIG_MODULE_SIG_PROTECT=y`：簽署模組可存取完整 KMI，未簽署模組僅能使用 `gki_module_unprotected.h` 中列出的符號

**KMI 符號保護**
- `gki_module.c` 在模組載入時 (`module/main.c:1284-1292`) 用 `bsearch()` 查詢受保護符號表
- 未授權模組嘗試使用受保護符號會導致載入失敗

**完整性度量**
- `CONFIG_IMA=y`（Integrity Measurement Architecture）：度量核心載入的每個檔案
- `CONFIG_EVM=y`（Extended Verification Module）：保護檔案安全屬性免受竄改

### 第六層：BPF 安全

**BPF 驗證器** (25,403 行) 在程式載入時執行靜態分析：

- 禁止無界迴圈（DAG 檢查）
- 驗證所有記憶體存取在合法範圍內
- 追蹤暫存器類型與值域（abstract interpretation）
- 檢查指標算術合法性
- 限制程式複雜度（指令數、分支深度）

**BPF LSM** 允許動態附加 eBPF 程式到 LSM hooks，實現可程式化的安全策略。

## 攻擊面分析

| 攻擊類型 | 防禦機制 |
|----------|---------|
| 緩衝區溢出 | FORTIFY_SOURCE、HARDENED_USERCOPY |
| 控制流劫持 | CFI、SCS |
| ROP/JOP | Shadow Call Stack、CFI |
| 未初始化記憶體 | INIT_STACK_ALL_ZERO、INIT_ON_ALLOC |
| Use-After-Free | SLAB_FREELIST_HARDENED、INIT_ON_ALLOC |
| 權限提升 | SELinux、Capabilities、SafeSetID |
| 行程注入 | Yama ptrace 限制 |
| 惡意模組 | Module signature、KMI protection |
| IPC 濫用 | Binder SELinux hooks、凍結/垃圾郵件偵測 |
| 結構體偏移攻擊 | RANDSTRUCT |

## Vendor Hook 對安全的影響

安全相關的 5 個 vendor hooks 均為 **restricted** 型別，且僅提供觀察能力：

- AVC 快取操作通知（insert/delete/replace/lookup）：廠商可用於安全審計或自訂快取策略
- SELinux 初始化通知：廠商可在 SELinux 就緒後執行裝置特定的安全初始化

**設計意圖**：廠商不能透過 vendor hooks 繞過或弱化任何安全檢查。

## 與上游 Linux 的安全差異

ACK 的安全強化在以下方面超越典型上游配置：

1. **更多 LSM 同時啟用**：多數上游發行版僅啟用一個主要 MAC（SELinux 或 AppArmor），ACK 同時啟用 7+
2. **全套編譯器保護**：CFI + SCS + 堆疊清零在桌面 Linux 中尚未普及
3. **GKI 模組保護**：上游無等效的符號存取控制機制
4. **Binder 專用安全 hooks**：上游不存在的 IPC 安全層

## 交叉參考

- [安全子系統](../subsystems/security.md) — LSM 框架與 SELinux 詳細分析
- [GKI 架構](../concepts/gki.md) — 模組簽署與 KMI 保護
- [BPF](../concepts/bpf.md) — 驗證器與 BPF LSM
- [Binder](../entities/binder.md) — Binder 安全機制
- [Vendor Hooks 分佈](vendor-hooks-distribution.md) — 安全 hooks 的保守策略
- [Android vs Upstream](android-vs-upstream.md) — 配置差異
