---
type: entity
kernel_path: drivers/base/platform.c
config_option: (always built-in)
upstream: yes
related:
  - ../data-structures/device.md
  - ../data-structures/device_driver.md
  - ../data-structures/bus_type.md
  - ../concepts/driver-model.md
  - ../subsystems/driver-framework.md
  - ../concepts/gki.md
last_updated: 2026-04-09
---

# Platform Bus

## Overview

Platform Bus 是 Linux 核心中最廣泛使用的匯流排類型，專為 SoC 上的非可發現裝置設計（不像 PCI/USB 有硬體枚舉能力）。在 Android 設備的 ARM64 SoC 上，大多數片上周邊（UART、SPI 控制器、GPIO 控制器、時鐘控制器、電源管理 IC 等）都透過 Platform Bus 管理。裝置描述來自 Device Tree（DT），驅動透過 compatible 字串匹配。

實作位於 `drivers/base/platform.c`（1,567 行），標頭檔為 `include/linux/platform_device.h`。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/base/platform.c` | 1,567 | Platform bus 完整實作 |
| `include/linux/platform_device.h` | ~300 | 公開 API 與結構定義 |

## Implementation Details

### 核心資料結構

**`struct platform_device`**：嵌入 `struct device`，擴展資源管理能力。

```c
struct platform_device {
    const char      *name;          // 裝置名稱
    int             id;             // 裝置 ID (-1 = 無 ID, -2 = 自動分配)
    struct device   dev;            // 嵌入的通用裝置
    u32             num_resources;  // 資源數量
    struct resource *resource;      // I/O memory、IRQ 等資源
    const struct platform_device_id *id_entry;  // 匹配的 ID 條目
    const char      *driver_override;           // 強制綁定驅動名稱
};
```

**`struct platform_driver`**：嵌入 `struct device_driver`，擴展 platform 特定回呼。

```c
struct platform_driver {
    int  (*probe)(struct platform_device *);
    void (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int  (*suspend)(struct platform_device *, pm_message_t);
    int  (*resume)(struct platform_device *);
    struct device_driver driver;                // 嵌入的通用驅動
    const struct platform_device_id *id_table;  // 匹配 ID 表
    bool prevent_deferred_probe;                // 禁止延遲探測
    bool driver_managed_dma;                    // 驅動自管 DMA (VFIO)
};
```

### 匹配流程

`platform_match()` @ `platform.c:1376` 實作五級匹配優先順序：

1. **`driver_override`**：直接字串比較，最高優先（供使用者空間強制綁定）
2. **Device Tree**：`of_driver_match_device()` 比較 DT `compatible` 屬性與驅動的 `of_match_table`
3. **ACPI**：`acpi_driver_match_device()` 比較 ACPI HID/CID
4. **ID Table**：`platform_match_id()` 比較 `platform_device_id` 表
5. **名稱**：`strcmp(pdev->name, drv->name)` 字串匹配（最低優先）

在 Android 設備上，絕大多數匹配透過 Device Tree compatible 字串完成。

### 探測流程

`platform_probe()` @ `platform.c:1420`：
1. 呼叫 `dev_pm_domain_attach()` 連接電源域
2. 執行 `platform_driver->probe()` 或 `device_driver->probe()`
3. 處理 `-EPROBE_DEFER` 延遲探測

### 便利巨集

```c
// 消除模組 init/exit 樣板（最常用）
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)

// 內建驅動
#define builtin_platform_driver(__platform_driver) \
    builtin_driver(__platform_driver, platform_driver_register)

// 帶 init 資料的內建驅動
#define builtin_platform_driver_probe(__platform_driver, __platform_probe) ...
```

### 資源存取 API

| 函式 | 用途 |
|------|------|
| `platform_get_resource(pdev, type, num)` | 取得第 N 個指定類型資源 |
| `platform_get_irq(pdev, num)` | 取得第 N 個 IRQ 號碼 |
| `platform_get_irq_optional(pdev, num)` | 同上，不存在時不報錯 |
| `platform_get_irq_byname(pdev, name)` | 依名稱取得 IRQ |
| `devm_platform_ioremap_resource(pdev, idx)` | 映射第 N 個 IOMEM 資源（managed） |
| `devm_platform_get_and_ioremap_resource(pdev, idx, &res)` | 同上，回傳 resource 指標 |

### `platform_bus_type` 定義

```c
struct bus_type platform_bus_type = {
    .name       = "platform",
    .dev_groups = platform_dev_groups,
    .match      = platform_match,
    .uevent     = platform_uevent,
    .probe      = platform_probe,
    .remove     = platform_remove,
    .shutdown   = platform_shutdown,
    .dma_configure = platform_dma_configure,
    .dma_cleanup   = platform_dma_cleanup,
    .pm         = &platform_dev_pm_ops,
};  /* @ platform.c:1516 */
```

## Userspace Interface

sysfs 拓撲：
```
/sys/bus/platform/
├── devices/          # 符號連結到 /sys/devices/platform/ 下的裝置
│   ├── soc:serial@...
│   ├── soc:i2c@...
│   └── ...
├── drivers/
│   ├── serial-driver/
│   │   ├── bind          # 寫入裝置名稱以手動綁定
│   │   ├── unbind        # 寫入裝置名稱以手動解綁
│   │   └── ...
│   └── ...
├── drivers_autoprobe     # 0/1 控制自動探測
└── drivers_probe         # 寫入裝置名稱觸發探測
```

## Android-Specific Notes

Platform Bus 在 ACK 中完全與上游一致，無 Android 修補。Android SoC 廠商（Qualcomm、MediaTek、Samsung 等）的大量裝置驅動都以 `module_platform_driver()` 註冊，作為 GKI vendor 模組載入。Platform Bus 的穩定性是 GKI 模組相容性的基石。

## Cross-References

- [`struct device`](../data-structures/device.md) — platform_device 嵌入的通用裝置
- [`struct device_driver`](../data-structures/device_driver.md) — platform_driver 嵌入的通用驅動
- [`struct bus_type`](../data-structures/bus_type.md) — Platform Bus 是 bus_type 的一個實例
- [Driver Model](../concepts/driver-model.md) — 驅動模型概念
- [Driver Framework](../subsystems/driver-framework.md) — 子系統完整分析
- [GKI](../concepts/gki.md) — Generic Kernel Image 架構
