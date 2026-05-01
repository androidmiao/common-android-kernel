---
type: source
source_path: drivers/firmware/arm_scmi/
lines: 19481
ingested: 2026-04-10
---

# Source: `drivers/firmware/arm_scmi/`

## Summary

ARM System Control and Management Interface (SCMI) 協定驅動，是現代 ARM SoC 上與系統控制處理器 (SCP) 通訊的標準化框架。提供電源域、時脈、DVFS 效能、感測器、重設、voltage 等管理功能。這是 ARM 生態系統中最重要的韌體介面之一，31 個 .c/.h 檔案共 19,481 行。

## Key Functions

| 函式/檔案 | 行數 | 用途 |
|-----------|------|------|
| `driver.c` | ~3,500 | 核心驅動：訊息傳輸、協定協商、xfer 管理 |
| `perf.c` | ~1,200 | DVFS/效能域管理、快取頻率表 |
| `clock.c` | ~800 | 時脈速率查詢與設定 |
| `power.c` | ~500 | 電源域狀態管理 |
| `sensors.c` | ~900 | 感測器資料收集與閾值通知 |
| `notify.c` | ~800 | 事件通知框架 |
| `bus.c` | ~400 | SCMI bus 實現（device-driver 匹配） |
| `raw_mode.c` | ~600 | Debug/測試用原始訊息注入 |
| `transports/mailbox.c` | ~300 | Mailbox 傳輸後端 |
| `transports/optee.c` | ~400 | OP-TEE 傳輸後端 |
| `transports/smc.c` | ~250 | SMC 直接呼叫傳輸後端 |
| `transports/virtio.c` | ~600 | Virtio 傳輸後端（虛擬化） |
| `vendors/imx/` | ~400 | i.MX 廠商擴展（BBM、CPU、LMM、MISC） |

## Notable Implementation Details

**多傳輸後端架構**：SCMI 支援四種傳輸機制（mailbox、OP-TEE、SMC、virtio），透過 DT/ACPI 在啟動時選擇。每種傳輸實現 `scmi_transport_ops` 介面，核心驅動 (`driver.c`) 對傳輸層透明。

**協定分層設計**：每個管理功能（perf、clock、power 等）作為獨立的 SCMI 協定實現，透過 `scmi_protocol_register()` 動態註冊。協定版本協商確保前後相容。

**通知子系統**：`notify.c` 實現完整的事件訂閱框架，允許上層驅動（如 thermal、cpufreq）接收感測器閾值、效能變更等非同步通知。

**廠商擴展**：`vendors/imx/` 展示了 SCMI 的擴展機制 — 廠商可在標準協定之上定義私有訊息 ID 和功能。

**Kconfig 選項**：`ARM_SCMI_PROTOCOL`（核心）、`ARM_SCMI_RAW_MODE_SUPPORT`（debug）、`ARM_SCMI_DEBUG_COUNTERS`（計數器）、`ARM_SCMI_QUIRKS`（SoC workarounds）、`ARM_SCMI_POWER_CONTROL`（系統電源控制）。

## Open Questions

- SCMI 的 virtio 傳輸在 Android 虛擬化場景（pKVM）中的使用範圍？
- i.MX 廠商擴展是否有其他 SoC 廠商的類似實現未在樹中？

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
- [Driver Model](../concepts/driver-model.md)
- [中斷處理](../concepts/interrupt-handling.md)
