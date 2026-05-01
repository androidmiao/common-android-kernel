---
type: analysis
question: "初次研究 common-android-mainline checkout 後，這個專案的用途、入口、建置方式與本地狀態是什麼？"
pages_consulted:
  - overview.md
  - concepts/gki.md
  - concepts/kconfig-and-build.md
  - android/drivers-android-overview.md
created: 2026-04-25
last_updated: 2026-04-25
---

# 專案初始導覽

## 結論

這個 checkout 是 Android Common Kernel (ACK) 的 `common-android-mainline` 工作區，不是單一 Git repository。外層由 Android `repo` manifest 管理，核心主體在 `common/`，建置工具在 `build/kernel/`，裝置與外部模組分散在 `devices/`、`common-modules/`、`external/`、`prebuilts/` 等子專案。

`common/Makefile` 顯示此核心版本為 Linux `6.19.0-rc8`，代號 `Baby Opossum Posse`（`common/Makefile:2-6`）。此分支的主要任務是提供 Android GKI 的共用核心基底：Google 維護通用 kernel image，OEM/SoC 差異透過 vendor modules、KMI、vendor hooks、device tree 與 defconfig fragments 接入。

## Checkout 結構

`.repo/manifests/default.xml` 是理解此工作區的最可靠入口。manifest 指定預設 revision 為 `main-kernel`，superproject revision 為 `common-android-mainline`（`.repo/manifests/default.xml:3-7`）。

關鍵子專案：

| 路徑 | manifest 專案 | 用途 |
|------|---------------|------|
| `common/` | `kernel/common`, revision `android-mainline` | Linux kernel 主體，包含 `arch/`、`drivers/`、`mm/`、`fs/`、`net/`、`kernel/`、`include/` 等 |
| `build/kernel/` | `kernel/build` | Kleaf/Bazel Android kernel 建置系統，並 link `tools/bazel` 與 `MODULE.bazel`（`.repo/manifests/default.xml:8-12`） |
| `common-modules/trusty/` | `kernel/common-modules/trusty` | Trusty 相關共用模組 |
| `common-modules/virtio-media/` | `platform/external/virtio-media` | virtio media 模組與範例 |
| `common-modules/virtual-device/` | `kernel/common-modules/virtual-device` | Cuttlefish/virtual device 相關模組與 fragments |
| `devices/google/raviole/` | `kernel/devices/google/raviole` | Pixel 6 / GS101 raviole device tree 與 GKI fragment |
| `kernel/tests/`, `test/dittosuite/`, `test/ltp/` | 多個測試專案 | kernel/network/LTP/dittosuite 測試支援（`.repo/manifests/default.xml:25-33`） |
| `prebuilts/` | clang、rust、build-tools、JDK、Tradefed 等 | hermetic build 所需工具鏈與測試工具 |

因此，平常改 kernel 程式碼通常會進 `common/`；改 Android kernel build 規則會進 `build/kernel/`；改裝置或模組整合會進 `devices/` 或 `common-modules/`。

## 建置入口

此 checkout 的主要建置系統是 Kleaf/Bazel。Kleaf 文件明確說明 Android Common Kernels 在 `common/` 下定義至少一個 kernel rule，最簡單建置命令是 `tools/bazel build //common:kernel`（`build/kernel/kleaf/docs/kleaf.md:21-29`）。Kleaf 也說明 `//common:kernel` 在 ACK/GKI 中通常是 `kernel_aarch64` 的 alias（`build/kernel/kleaf/docs/kleaf.md:48-52`）。

`common/BUILD.bazel` 的實際定義也符合這點：

- `kernel_aarch64` 是主要 ARM64 GKI 目標，使用 `arch/arm64/configs/gki_defconfig`，輸出 GKI artifacts，make goals 包含 `Image`、壓縮 image 與 `modules`（`common/BUILD.bazel:208-229`）。
- `kernel_aarch64_16k` 使用同一個 GKI defconfig，但設定 `page_size = "16k"`（`common/BUILD.bazel:243-265`）。
- `kernel_x86_64` 使用 `arch/x86/configs/gki_defconfig`（`common/BUILD.bazel:286-307`）。
- `kernel` alias 指向 `kernel_aarch64`，`kernel_dist` alias 指向 `kernel_aarch64_dist`（`common/BUILD.bazel:309-318`）。

常用命令：

```sh
tools/bazel build //common:kernel
tools/bazel build //common:kernel_aarch64
tools/bazel run //common:kernel_aarch64_dist
tools/bazel run //common:kernel_aarch64_config
```

## Android 特有層

ACK 相對上游 Linux 的核心差異集中在 GKI、Android 專屬驅動、vendor hooks、KABI/KMI 穩定性與 Android 預設配置。第一個值得閱讀的原始碼入口是 `common/drivers/android/`。

`common/drivers/android/Kconfig` 定義 Android driver 選單與主要選項：

| CONFIG | 用途 |
|--------|------|
| `ANDROID_BINDER_IPC` | Android Binder IPC C driver，Binder 是 Android process 間通訊與 remote method invocation 的核心（`common/drivers/android/Kconfig:4-15`） |
| `ANDROID_BINDER_IPC_RUST` | Rust 版 Binder，依賴 `RUST && MMU && !ANDROID_BINDER_IPC`，與 C 版互斥（`common/drivers/android/Kconfig:17-28`） |
| `ANDROID_BINDERFS` | Binderfs pseudo-filesystem，支援 per-IPC namespace 動態建立 binder devices（`common/drivers/android/Kconfig:30-40`） |
| `ANDROID_BINDER_DEVICES` | 預設 binder 裝置名稱為 `binder,hwbinder,vndbinder`（`common/drivers/android/Kconfig:42-52`） |
| `ANDROID_VENDOR_HOOKS` | 以 tracepoints 實作 vendor hooks，讓 vendor modules attach 到 `DECLARE_HOOK` 或 `DECLARE_RESTRICTED_HOOK`（`common/drivers/android/Kconfig:65-72`） |
| `ANDROID_DEBUG_KINFO` | 匯出 kallsyms、page directory pointer、UTS_RELEASE、build fingerprint 等 crash/debug 資訊（`common/drivers/android/Kconfig:74-83`） |
| `ANDROID_KABI_RESERVE` | GKI 結構體 ABI 保留填充（`common/drivers/android/Kconfig:85-100`） |
| `ANDROID_VENDOR_OEM_DATA` | vendor/OEM data padding，供 vendor hooks 與 vendor modules 使用（`common/drivers/android/Kconfig:102-120`） |

對後續研究而言，建議從這幾個層次進入：

1. GKI 與 Kleaf：`wiki/concepts/gki.md`、`wiki/concepts/kconfig-and-build.md`、`build/kernel/kleaf/docs/kleaf.md`
2. Android 專屬驅動：`common/drivers/android/`、`wiki/android/drivers-android-overview.md`
3. Vendor hooks 與 ABI：`include/trace/hooks/`、`include/linux/android_kabi.h`、`include/linux/android_vendor.h`
4. 主要 Linux 子系統：scheduler、memory management、networking、filesystems、block、security

## 本地狀態注意事項

外層 `common-android-mainline/` 不是 Git repository；它是由 `.repo/` 管理的多 repo checkout。各子目錄，例如 `common/` 與 `build/kernel/`，才是獨立 Git repository。

初次研究時觀察到：

- `build/kernel` 處於 detached HEAD，且沒有顯示未提交修改。
- `common` 處於 detached HEAD，且已有未提交修改。
- `common` 的未提交修改集中在 netfilter xtables/UAPI 相關檔案與一個 memory model litmus test：`include/uapi/linux/netfilter/xt_*.h`、`include/uapi/linux/netfilter_ipv4/ipt_*.h`、`include/uapi/linux/netfilter_ipv6/ip6t_HL.h`、`net/netfilter/xt_*.c`、`tools/memory-model/litmus-tests/Z6.0+pooncelock+poonceLock+pombonce.litmus`。

後續若要實作功能或修 bug，應先確認這些未提交修改是否屬於正在進行的工作，避免覆蓋或誤判 diff。

## 交叉參考

- [Architecture Overview](../overview.md)
- [GKI](../concepts/gki.md)
- [Kconfig 與 Build 系統](../concepts/kconfig-and-build.md)
- [drivers/android/ 目錄總覽](../android/drivers-android-overview.md)
- [Android vs Upstream 差異全面分析](android-vs-upstream.md)
