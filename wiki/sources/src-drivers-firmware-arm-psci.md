---
type: source
source_path: drivers/firmware/psci/ + drivers/firmware/smccc/ + drivers/firmware/arm_ffa/
lines: 4114
ingested: 2026-04-10
---

# Source: ARM PSCI / SMCCC / FF-A

## Summary

三個密切相關的 ARM 標準韌體介面驅動：

**PSCI (Power State Coordination Interface)**：CPU 電源狀態管理（熱插拔、idle、系統掛起），2 個檔案 1,332 行。是所有 ARM64 Android 裝置的必要元件。

**SMCCC (Secure Monitor Call Calling Convention)**：定義 SMC/HVC 呼叫的標準化暫存器規範，3 個檔案 386 行。是所有安全世界通訊（PSCI、Qualcomm SCM、Samsung 等）的底層基礎。

**FF-A (Firmware Framework for A-profile)**：ARMv8-A 虛擬化環境中安全/普通世界 VM 間的通訊框架，4 個檔案 2,396 行。支援 Hafnium hypervisor 和 Android pKVM。

## Key Functions

### PSCI (`psci/`)

| 函式 | 檔案 | 用途 |
|------|------|------|
| `psci_cpu_on()` | psci.c | CPU 上線（熱插拔） |
| `psci_cpu_off()` | psci.c | CPU 下線 |
| `psci_cpu_suspend()` | psci.c | CPU 掛起到低功耗狀態 |
| `psci_system_suspend()` | psci.c | 整個系統掛起 |
| `psci_system_off()` / `psci_system_reset()` | psci.c | 關機/重啟 |
| `psci_checker_main()` | psci_checker.c | 運行時 PSCI 正確性驗證 |

### SMCCC (`smccc/`)

| 函式 | 檔案 | 用途 |
|------|------|------|
| `arm_smccc_smc()` | (asm) | 觸發 SMC 例外進入 EL3 |
| `arm_smccc_hvc()` | (asm) | 觸發 HVC 例外進入 EL2 |
| `smccc_soc_id_init()` | soc_id.c | SoC 識別碼查詢 |
| `kvm_arm_hyp_service_available()` | kvm_guest.c | KVM guest SMCCC 服務偵測 |

### FF-A (`arm_ffa/`)

| 函式 | 檔案 | 用途 |
|------|------|------|
| `ffa_partition_info_get()` | driver.c | 查詢安全分區 |
| `ffa_msg_send_direct_req()` | driver.c | 直接訊息傳送 |
| `ffa_mem_share()` | driver.c | 記憶體共享 |
| `ffa_bus_type` | bus.c | FF-A 裝置匯流排 |

## Notable Implementation Details

**PSCI 是系統基礎**：Android 裝置的每一次 CPU 熱插拔（big.LITTLE 調度）、每一次深度 idle（C-state 進入）、每一次系統掛起/恢復都經過 PSCI。它是 `arm64/kernel/psci.c`（架構層）和 `drivers/firmware/psci/psci.c`（韌體層）的協作。

**SMCCC 版本協商**：`smccc.c` 在啟動時查詢安全韌體支援的 SMCCC 版本（1.0、1.1、1.2 等），決定可用的功能集。這影響了 PSCI、FF-A、KVM 等所有上層協定的行為。

**FF-A RXTX 緩衝區**：FF-A 使用一對 RX/TX 共享記憶體頁面作為通訊緩衝區，由 hypervisor 在安全/普通世界間轉發。每個 VM 有獨立的 16-bit UUID 識別。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
- [ARM64 架構](../subsystems/arch-arm64.md)
