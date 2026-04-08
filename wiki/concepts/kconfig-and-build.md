---
type: concept
scope: kernel-wide
related:
  - ../concepts/gki.md
  - ../concepts/module-system.md
last_updated: 2026-04-08
---

# Kconfig 與 Build 系統

## 概述

Android Common Kernel 使用兩套構建系統：**Kleaf/Bazel**（現代，主要）與**傳統 build.config**（已廢棄）。Kconfig 保持 Linux 標準階層結構，但新增了 GKI 特化層（`init/Kconfig.gki`）以支持模組化構建。整個構建流程已從 shell-based 的 `build.config.*` 遷移到確定性的 Bazel 規則。

## 機制

### Kconfig 階層

**頂級 Kconfig**（`Kconfig`，38行）：

主選單 "Linux/$(ARCH) $(KERNELVERSION) Kernel Configuration"，按標準 Linux 方式包含各子系統的 Kconfig 源文件（init、fs、drivers、security 等）。第37行使用 `$(KCONFIG_EXT_PREFIX)Kconfig.ext` 用於外部專案擴展。

**外部擴展**（`Kconfig.ext`，4行）：佔位符文件，當 `KCONFIG_EXT_PREFIX` 未定義時使用，設計用於裝置特定的配置。

**GKI 隱藏配置**（`init/Kconfig.gki`，336行）：

為模組化構建預先啟用只在模組中使用的配置選項。主要配置組：

| 配置組 | 行號 | 用途 |
|--------|------|------|
| `GKI_HIDDEN_DRM_CONFIGS` | 1-19 | DRM/顯示相關依賴 |
| `GKI_HIDDEN_CRYPTO_CONFIGS` | 40-46 | 加密引擎 |
| `GKI_HIDDEN_SND_CONFIGS` | 48-55 | 音頻系統 |
| `GKI_HIDDEN_SND_SOC_CONFIGS` | 57-72 | SoC 音頻 |
| `GKI_HIDDEN_QCOM_CONFIGS` | 119-130 | 高通特定 |
| `GKI_HIDDEN_MEDIA_CONFIGS` | 132-149 | 媒體控制器 |
| `GKI_HIDDEN_USB_CONFIGS` | 179-188 | USB PHY |
| `GKI_HIDDEN_XFRM_OFFLOAD_CONFIGS` | 285-290 | IPsec 卸載 |

集合點 `GKI_HACKS_TO_FIX`（第299-335行）一次選擇所有隱藏配置——這是臨時措施，註釋指出「應重做為上游解決方案」。

### GKI Defconfig

**ARM64 主配置**：`arch/arm64/configs/gki_defconfig`（785行）

關鍵配置分類：

**核心與排程**：`CONFIG_PREEMPT=y`（11）、`CONFIG_SCHED_CLASS_EXT=y`（12）、`CONFIG_UCLAMP_TASK=y`（27）

**BPF**：`CONFIG_BPF_SYSCALL=y`（6）、`CONFIG_BPF_JIT_ALWAYS_ON=y`（8）、`CONFIG_BPF_LSM=y`（10）

**RCU**：`CONFIG_RCU_NOCB_CPU=y`（21）、`CONFIG_RCU_LAZY=y`（22）

**Rust**：`CONFIG_RUST=y`（54）

**模組**：`CONFIG_MODULES=y`（100）、`CONFIG_MODULE_SIG=y`（105）、`CONFIG_MODULE_SIG_PROTECT=y`（106）

**安全**：`CONFIG_SHADOW_CALL_STACK=y`（98）、`CONFIG_CFI=y`（99）

**Android**：`CONFIG_ANDROID_BINDER_IPC=y`（630）、`CONFIG_ANDROID_VENDOR_HOOKS=y`（633）、`CONFIG_GKI_HACKS_TO_FIX=y`（113）

**Defconfig Fragment 機制**：透過 Bazel 的 `pre_defconfig_fragments` / `post_defconfig_fragments` 在基礎 defconfig 上疊加配置片段。現有片段位於 `build/kernel/kleaf/impl/defconfig/`：

- `arm64_4k_defconfig` — 4K 頁面大小
- `arm64_16k_defconfig` — 16K 頁面大小
- `arm64_64k_defconfig` — 64K 頁面大小
- 以及 debug、kasan、kcov、rust 等功能變體

### Bazel/Kleaf 構建系統

**架構層次**：

```
build/kernel/kleaf/
├── kernel.bzl              # 公共規則出口（143行）
├── common_kernels.bzl      # GKI kernel 通用功能
├── constants.bzl           # 常數定義
└── impl/                   # 實現層
    ├── kernel_build.bzl    # kernel_build 規則
    ├── gki_artifacts.bzl   # GKI 構件生成
    ├── defconfig/          # Defconfig 片段
    └── abi/                # ABI 相關規則
```

**公共規則**（`kernel.bzl`）：

- `kernel_build` — 主核心構建規則
- `kernel_module` — 核心模組規則
- `kernel_abi` — ABI 監控
- `gki_artifacts` — GKI 構件生成
- `kernel_images` — 映像打包
- `system_dlkm_image` — 系統 DLKM 映像

**核心 Bazel 目標**（`BUILD.bazel:208-307`）：

```python
common_kernel(
    name = "kernel_aarch64",
    outs = DEFAULT_GKI_OUTS,
    arch = "arm64",
    build_gki_artifacts = True,
    defconfig = "arch/arm64/configs/gki_defconfig",
    make_goals = ["Image", "Image.lz4", "modules"],
    module_implicit_outs = get_gki_modules_list("arm64"),
)
```

定義三個主要目標：`kernel_aarch64`（4K 頁面）、`kernel_aarch64_16k`（16K）、`kernel_x86_64`。

**GKI 構件生成**（`gki_artifacts.bzl:27-148`）：

1. 使用 `mkbootimg` 生成啟動映像
2. 支持 lz4、gz、lzma 壓縮
3. 生成 `gki-info.txt` 元數據
4. AVB（Android Verified Boot）簽名

**模組清單管理**（`bazel/modules_private.bzl`，200+ 行）：

- `_COMMON_GKI_MODULES_LIST`：91 個通用模組（11-91行）
- 架構特定列表：ARM64、x86_64、ARM、i386
- `get_gki_modules_list()` / `get_gki_modules_superset()` — 查詢函數

### 構建常數

`build.config.constants`（6行）：

```
BRANCH="android-mainline"
CLANG_VERSION="r584948b"
RUSTC_VERSION="1.91.1"
```

### 傳統 Build Config（已廢棄）

以下文件已廢棄，顯示錯誤信息指導用戶使用 Bazel：

- `build.config.common` — 建議設置 `kernel_build.kcflags`
- `build.config.gki` — 建議設置 `kernel_build.defconfig`
- `build.config.gki.aarch64` — 建議設置 `kernel_build.make_goals`

### 構建流程

```
kernel_build() 規則：
1. 接收 defconfig 路徑
2. 應用 pre_defconfig_fragments（可選）
3. 運行 Makefile 生成 .config
4. 應用 post_defconfig_fragments（check_defconfig）
5. 編譯核心：make_goals = ["Image", "Image.lz4", "modules"]
6. 收集模組（module_implicit_outs）
7. 簽署模組（MODULE_SIG=y）
8. 打包 system_dlkm_image
9. 生成 GKI 構件（boot.img）
```

## 使用模式

### 裝置特定的配置方式

廠商透過 defconfig fragment 疊加自己的配置，例如 `common-modules/trusty/trusty_defconfig.fragment`：

```
CONFIG_TRUSTY=m
CONFIG_TRUSTY_CRASH_IS_PANIC=y
CONFIG_TRUSTY_SMC_TRANSPORT=m
```

### 受保護模組名稱

`common_kernel_protected_module_names` 規則（`common_kernels.bzl:648-656`）從模組列表生成排序的受保護模組清單，用於 GKI 符號保護。

## Android 相關性

| 特性 | 實現 |
|------|------|
| 模組化 | 從 build.config 遷移到 Bazel `kernel_build` 規則 |
| ABI 追蹤 | `kernel_abi` 規則 + KMI 符號列表 |
| 受保護模組 | `common_kernel_protected_module_names` 規則 |
| DLKM 打包 | `system_dlkm_image` 規則 |
| Defconfig 管理 | Fragment 疊加而非完整重複 |
| 跨架構 | arm64、arm、x86_64、i386 支援 |
| 頁面大小變體 | 4K、16K、64K 選項 |
| Hermetic 構建 | Kleaf hermetic_tools 與 toolchain 管理 |

## 交叉參考

- [GKI](gki.md) — Kconfig/Build 系統服務的核心架構
- [Module 系統](module-system.md) — 模組的構建與簽署
