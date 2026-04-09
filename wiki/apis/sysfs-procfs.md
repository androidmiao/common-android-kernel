---
type: api
direction: kernel-to-user
related:
  - ../subsystems/filesystems.md
  - ../concepts/module-system.md
  - ../concepts/vendor-hooks.md
  - ../subsystems/memory-management.md
last_updated: 2026-04-09
---

# sysfs / procfs 介面

## Overview

sysfs 和 procfs 是 Linux 核心透過虛擬檔案系統暴露資訊給使用者空間的兩大機制。sysfs（掛載於 `/sys`）圍繞 kobject 裝置模型，以結構化階層呈現裝置、驅動和子系統資訊；procfs（掛載於 `/proc`）提供行程資訊和核心參數的存取介面。兩者合計構成核心最廣泛使用的資訊暴露 API。

## Interface Definition

---

## Part I: sysfs

### 核心實現

**位置：** `fs/sysfs/`

| 檔案 | 行數 | 用途 |
|------|------|------|
| `file.c` | 20,838 | 檔案操作（讀寫屬性） |
| `dir.c` | 4,181 | 目錄操作 |
| `group.c` | 16,615 | 屬性群組管理 |
| `mount.c` | 2,543 | 掛載操作 |
| `symlink.c` | 4,960 | 符號連結操作 |

### Kobject 基礎設施

定義於 `include/linux/kobject.h:64-82`：

```c
struct kobject {
    const char              *name;
    struct list_head         entry;
    struct kobject          *parent;
    struct kset             *kset;
    const struct kobj_type  *ktype;
    struct kernfs_node      *sd;       // sysfs 目錄節點
    struct kref              kref;
    // ...
};
```

- **`struct kset`**（lines 168-173）：kobject 集合，帶 uevent 操作
- **`struct kobj_type`**（lines 116-123）：類型描述符，定義 `sysfs_ops` 和預設屬性群組

### sysfs_ops 回呼

定義於 `include/linux/sysfs.h:392`：

```c
struct sysfs_ops {
    ssize_t (*show)(struct kobject *, struct attribute *, char *);
    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

讀取使用 seq_file 介面；寫入限制為 `PAGE_SIZE`（通常 4,096 bytes）。

### 屬性巨集

定義於 `include/linux/device.h:139-265`：

| 巨集 | 權限 | 方向 | 說明 |
|------|------|------|------|
| `DEVICE_ATTR(_name, _mode, _show, _store)` | 自訂 | R/W | 完整自訂（line 157） |
| `DEVICE_ATTR_RW(_name)` | 0644 | R/W | 讀寫屬性（line 180） |
| `DEVICE_ATTR_RO(_name)` | 0444 | R | 唯讀屬性（line 198） |
| `DEVICE_ATTR_WO(_name)` | 0200 | W | 唯寫屬性（line 216） |
| `DEVICE_ATTR_ADMIN_RW(_name)` | 0600 | R/W | 管理員讀寫（line 189） |
| `DEVICE_ATTR_ADMIN_RO(_name)` | 0400 | R | 管理員唯讀（line 207） |

**型別輔助巨集**（lines 228-265）：
- `DEVICE_ULONG_ATTR` — 背靠 `unsigned long`，自動 show/store
- `DEVICE_INT_ATTR` — 背靠 `int`
- `DEVICE_BOOL_ATTR` — 背靠 `bool`
- `DEVICE_STRING_ATTR_RO` — 字串屬性（唯讀）

**其他層級屬性巨集**：

| 巨集 | 標頭 | 說明 |
|------|------|------|
| `BUS_ATTR_RW/RO/WO` | `include/linux/device/bus.h:122-127` | 匯流排層級 |
| `DRIVER_ATTR_RW/RO/WO` | `include/linux/device/driver.h:151-156` | 驅動層級 |
| `CLASS_ATTR_RW/RO/WO` | `include/linux/device/class.h:175-180` | 類別層級 |

### 屬性群組

`include/linux/sysfs.h:105-123`：

```c
struct attribute_group {
    const char              *name;          // 子目錄名稱（可選）
    umode_t (*is_visible)(struct kobject *, struct attribute *, int);
    umode_t (*is_bin_visible)(struct kobject *, struct bin_attribute *, int);
    struct attribute        **attrs;
    struct bin_attribute    **bin_attrs;
};
```

- `is_visible()` 回呼可根據 kobject 動態顯示/隱藏屬性
- `SYSFS_PREALLOC` 旗標（line 125）：預分配 sysfs 資源
- `SYSFS_GROUP_INVISIBLE` 旗標（line 126）：隱藏整個群組目錄

### sysfs 階層結構

| 路徑 | 內容 |
|------|------|
| `/sys/class/` | 裝置類別（net、block、input 等） |
| `/sys/devices/` | 裝置拓撲樹 |
| `/sys/bus/` | 匯流排類型（pci、usb、platform 等） |
| `/sys/kernel/` | 核心參數 |
| `/sys/module/` | 已載入模組及其參數 |
| `/sys/fs/` | 檔案系統參數 |

---

## Part II: procfs

### 核心實現

**位置：** `fs/proc/`

| 檔案 | 行數 | 用途 |
|------|------|------|
| `base.c` | 98,726 | `/proc/[pid]/*` 行程資訊 |
| `proc_sysctl.c` | 45,285 | `/proc/sys/` sysctl 整合 |
| `generic.c` | 20,001 | 核心 proc 項目建立/管理 |
| `inode.c` | 17,029 | VFS inode 操作 |
| `fd.c` | 9,168 | 檔案描述符資訊 |
| `meminfo.c` | 181 | `/proc/meminfo` |

### proc_ops 結構

定義於 `include/linux/proc_fs.h:35-51`：

```c
struct proc_ops {
    unsigned int  proc_flags;
    int         (*proc_open)(struct inode *, struct file *);
    ssize_t     (*proc_read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t     (*proc_read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t     (*proc_write)(struct file *, const char __user *, size_t, loff_t *);
    loff_t      (*proc_lseek)(struct file *, loff_t, int);
    int         (*proc_release)(struct inode *, struct file *);
    __poll_t    (*proc_poll)(struct file *, struct poll_table_struct *);
    long        (*proc_ioctl)(struct file *, unsigned int, unsigned long);
    int         (*proc_mmap)(struct file *, struct vm_area_struct *);
};
```

**proc_flags**（lines 22-33）：
- `PROC_ENTRY_PERMANENT` — 項目永不移除
- `PROC_ENTRY_proc_read_iter` — 使用 iter 讀取
- `PROC_ENTRY_FORCE_LOOKUP` — 每次存取強制查詢

### 項目建立 API

定義於 `fs/proc/generic.c:418-677`：

| API | 簽名 | 用途 |
|-----|------|------|
| `proc_create()` | `(name, mode, parent, proc_ops)` @ line 600 | 建立常規檔案 |
| `proc_create_data()` | `(name, mode, parent, proc_ops, data)` @ line 586 | 帶私有資料 |
| `proc_create_seq()` | `(name, mode, parent, seq_ops)` @ line 102（巨集） | 使用 seq_file |
| `proc_create_seq_private()` | `(name, mode, parent, seq_ops, state_size, data)` @ line 634 | seq_file 帶狀態 |
| `proc_create_single()` | `(name, mode, parent, show)` @ line 107（巨集） | 最簡 API |
| `proc_create_single_data()` | `(name, mode, parent, show, data)` @ line 665 | 帶私有資料 |
| `proc_mkdir()` | `(name, parent)` @ line 543 | 建立子目錄 |
| `proc_symlink()` | `(name, parent, target)` @ line 484 | 建立符號連結 |
| `proc_create_mount_point()` | `(name)` @ line 550 | 空掛載點 |

### seq_file 介面

定義於 `include/linux/seq_file.h`，是 procfs 最常用的讀取機制：

```c
struct seq_operations {
    void * (*start)(struct seq_file *m, loff_t *pos);
    void   (*stop)(struct seq_file *m, void *v);
    void * (*next)(struct seq_file *m, void *v, loff_t *pos);
    int    (*show)(struct seq_file *m, void *v);
};
```

**輸出輔助函式**：`seq_printf()`、`seq_puts()`、`seq_putc()`、`seq_put_decimal_ull()`、`seq_write()`、`seq_escape_mem()`。

### proc_dir_entry 結構

定義於 `fs/proc/internal.h:31-67`：

- `atomic_t in_use` — 模組引用計數
- `struct proc_ops *proc_ops` — 檔案操作
- `struct seq_operations *seq_ops` — 序列檔案操作（可選）
- `void *data` — 私有資料
- `umode_t mode` — 檔案權限
- `struct rb_tree subdir` — 子目錄紅黑樹

### 主要 /proc 階層

| 路徑 | 實現檔案 | 內容 |
|------|----------|------|
| `/proc/stat` | `kernel/time/timer_list.c` | CPU 統計 |
| `/proc/meminfo` | `fs/proc/meminfo.c` (181行) | 記憶體資訊 |
| `/proc/[pid]/` | `fs/proc/base.c` (98,726行) | 行程資訊 |
| `/proc/[pid]/maps` | `fs/proc/task_mmu.c` | 記憶體映射 |
| `/proc/[pid]/fd/` | `fs/proc/fd.c` | 檔案描述符 |
| `/proc/sys/` | `fs/proc/proc_sysctl.c` | sysctl 參數 |

## Semantics

### 權限模型

**sysfs：**
- 檔案權限由巨集定義（0444、0644、0600 等）
- 透過 `is_visible()` 回呼動態控制存取
- 每個屬性限制讀寫大小為 `PAGE_SIZE`

**procfs：**
- 檔案權限由 `mode` 參數指定
- 預設擁有者為 root (uid 0, gid 0)
- `proc_set_user()` @ `include/linux/proc_fs.h:117` 可變更擁有者
- 存取由 inode 權限控制

### sysfs vs procfs 選擇指南

| 特性 | sysfs | procfs |
|------|-------|--------|
| 設計目的 | 裝置/驅動模型 | 行程/核心參數 |
| 結構 | kobject 階層 | 自由結構 |
| 每檔案限制 | PAGE_SIZE | 無限制（seq_file） |
| 建立方式 | 屬性巨集（自動） | proc_create API（手動） |
| 適用場景 | 裝置屬性 | 統計、行程資訊、sysctl |

## Android-Specific Notes

### Android sysfs 使用 [android]

- **Trusty TEE 驅動**（`common-modules/trusty/drivers/trusty/trusty.c`）：透過 `DEVICE_ATTR` 暴露 Trusty 裝置屬性，權限 0660
- **模組參數**：Android 模組廣泛使用 `module_param()` 巨集在 `/sys/module/` 下暴露可調參數

### Android procfs 整合 [android]

- **cpufreq times 追蹤**：`include/linux/cpufreq_times.h`，在 `/proc/[pid]/` 下加入 CPU 頻率時間統計
- **LSM hook 整合**：擴充 `/proc/[pid]/` 項目的安全檢查
- **擴充記憶體映射資訊**：smaps 包含每區域 RSS 詳細統計

## Cross-References

- [Filesystems](../subsystems/filesystems.md) — VFS 層與 sysfs/procfs 的整合
- [Module 系統](../concepts/module-system.md) — `/sys/module/` 下的模組參數暴露
- [Memory Management](../subsystems/memory-management.md) — `/proc/meminfo` 和 `/proc/[pid]/maps`
- [ioctl 介面](ioctl-interfaces.md) — 另一種核心-使用者空間通訊
- [Netlink](netlink.md) — Socket 式核心-使用者空間通訊
- [Syscalls](syscalls.md) — 系統呼叫介面
