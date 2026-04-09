---
type: api
direction: user-to-kernel
related:
  - ../entities/binder.md
  - ../entities/binderfs.md
  - ../subsystems/filesystems.md
  - ../subsystems/networking.md
  - ../concepts/vendor-hooks.md
last_updated: 2026-04-09
---

# ioctl 介面

## Overview

ioctl（input/output control）是 Linux 核心最重要的使用者空間-核心空間通訊機制之一。透過單一系統呼叫 `ioctl(fd, cmd, arg)` 對已開啟的檔案描述符執行裝置或檔案系統特定的操作。ACK 中定義了約 **1,834 個** ioctl 命令，分布在 **132 個** UAPI 標頭檔中，涵蓋超過 **40 個** magic number 群組。

## Interface Definition

### 命令編碼格式

ioctl 命令是一個 32-bit 整數，由四個欄位組成（定義於 `include/uapi/asm-generic/ioctl.h`）：

| 欄位 | 位元 | 說明 |
|------|------|------|
| direction | 2 bits (30-31) | `_IOC_NONE`(0)、`_IOC_WRITE`(1)、`_IOC_READ`(2) |
| size | 14 bits (16-29) | 參數結構大小，最大 16KB-1 |
| type | 8 bits (8-15) | Magic number，標識命令群組 |
| number | 8 bits (0-7) | 群組內命令編號 |

### 定義巨集

```c
_IO(type, nr)              // 無資料傳輸
_IOR(type, nr, argtype)    // 核心 → 使用者空間（讀取）
_IOW(type, nr, argtype)    // 使用者空間 → 核心（寫入）
_IOWR(type, nr, argtype)   // 雙向傳輸
```

### VFS 派發機制

主要入口函式位於 `fs/ioctl.c`：

1. **`SYSCALL_DEFINE3(ioctl)`** @ `fs/ioctl.c:583` — 系統呼叫入口，執行安全檢查後派發
2. **`do_vfs_ioctl()`** @ `fs/ioctl.c:492` — 處理通用 VFS ioctl（11 個內建命令）：
   - 檔案控制：`FIOCLEX`、`FIONCLEX`、`FIONBIO`、`FIOASYNC`、`FIOQSIZE`
   - 檔案系統操作：`FIFREEZE`、`FITHAW`、`FS_IOC_FIEMAP`、`FIGETBSZ`
   - 檔案複製：`FICLONE`、`FICLONERANGE`、`FIDEDUPERANGE`
   - 屬性管理：`FS_IOC_GETFLAGS`、`FS_IOC_SETFLAGS`
3. **`vfs_ioctl()`** @ `fs/ioctl.c:44` — 呼叫檔案系統或驅動的 `->unlocked_ioctl` handler

### 32/64-bit 相容性

`CONFIG_COMPAT` 啟用時，核心提供 `compat_ptr_ioctl()` @ `fs/ioctl.c:629` 作為通用相容層，自動轉換指標大小。各驅動可透過 `f_op->compat_ioctl` 提供自訂相容處理。

## Semantics

### Android 關鍵 ioctl 介面

#### Binder IPC（最關鍵的 Android 介面）[android]

**Magic number:** `'b'`（0x62）
**標頭：** `include/uapi/linux/android/binder.h`
**處理函式：** `binder_ioctl()` @ `drivers/android/binder.c:5965`

| 命令 | 巨集 | 方向 | 資料結構 | 用途 |
|------|------|------|----------|------|
| `BINDER_WRITE_READ` | `_IOWR('b', 1)` | 雙向 | `binder_write_read` | 核心 IPC 讀寫操作 |
| `BINDER_SET_MAX_THREADS` | `_IOW('b', 5)` | 寫入 | `__u32` | 設定最大執行緒數 |
| `BINDER_SET_CONTEXT_MGR` | `_IOW('b', 7)` | 寫入 | `__s32` | 註冊為 context manager |
| `BINDER_THREAD_EXIT` | `_IOW('b', 8)` | 寫入 | `__s32` | 執行緒退出通知 |
| `BINDER_VERSION` | `_IOWR('b', 9)` | 雙向 | `binder_version` | 查詢協定版本 |
| `BINDER_SET_CONTEXT_MGR_EXT` | `_IOW('b', 13)` | 寫入 | `flat_binder_object` | 擴充 context manager |
| `BINDER_FREEZE` | `_IOW('b', 14)` | 寫入 | `binder_freeze_info` | 凍結行程 |
| `BINDER_GET_FROZEN_INFO` | `_IOWR('b', 15)` | 雙向 | `binder_frozen_status_info` | 查詢凍結狀態 |
| `BINDER_ENABLE_ONEWAY_SPAM_DETECTION` | `_IOW('b', 16)` | 寫入 | `__u32` | 啟用單向呼叫垃圾偵測 |
| `BINDER_GET_EXTENDED_ERROR` | `_IOWR('b', 17)` | 雙向 | `binder_extended_error` | 取得擴充錯誤資訊 |

Binder 還定義了兩組子協定命令：BC_*（magic `'c'`，21 個使用者→核心命令）和 BR_*（magic `'r'`，23 個核心→使用者回應），透過 `BINDER_WRITE_READ` 的讀寫緩衝區傳遞。

#### Binderfs 控制 [android]

**Magic number:** `'b'`（0x62）
**標頭：** `include/uapi/linux/android/binderfs.h`
**處理函式：** `binder_ctl_ioctl()` @ `drivers/android/binderfs.c`

- `BINDER_CTL_ADD` | `_IOWR('b', 1)` | `struct binderfs_device` — 動態建立 binder 裝置節點

#### Input/Event（evdev）[upstream]

**Magic number:** `'E'`（18 個命令）
**標頭：** `include/uapi/linux/input.h`

- `EVIOCGVERSION` — 取得驅動版本
- `EVIOCGID` — 取得裝置 ID
- `EVIOCGNAME(len)` — 取得裝置名稱
- `EVIOCGBIT()` — 查詢事件能力位元
- `EVIOCGRAB` — 獨佔裝置
- `EVIOCSCLOCKID` — 設定時鐘來源

#### Video4Linux2（V4L2）[upstream]

**Magic number:** `'V'`（**116+ 命令**，最大的 ioctl 群組）
**標頭：** `include/uapi/linux/videodev2.h`

涵蓋攝影擷取、影片播放、覆疊、編解碼、調諧器、音訊、串流等功能。核心命令包含 `VIDIOC_QUERYCAP`、`VIDIOC_G_FMT`/`VIDIOC_S_FMT`、`VIDIOC_REQBUFS`、`VIDIOC_QBUF`/`VIDIOC_DQBUF`、`VIDIOC_STREAMON`/`VIDIOC_STREAMOFF`。

#### DRM/Graphics [upstream]

**標頭：** `include/drm/drm_ioctl.h`、`drivers/gpu/drm/`

透過 `DRM_IOCTL_DEF_DRV()` 巨集定義驅動特定命令，支援 KMS（Kernel Mode Setting）、GEM（Graphics Execution Manager）緩衝區管理、驗證/主控機制。旗標系統包含 `AUTH`、`MASTER`、`ROOT_ONLY`、`RENDER_ALLOW`。

#### TTY/Terminal [upstream]

**Magic number:** `'T'`（30 個命令）
**標頭：** `include/uapi/asm-generic/ioctls.h`

- `TCGETS`/`TCSETS` — 終端設定
- `TIOCGWINSZ`/`TIOCSWINSZ` — 視窗大小
- `TIOCMGET`/`TIOCMSET` — 數據機控制

### 命令群組統計

| Magic | 字母 | 命令數 | 主要用途 |
|-------|------|--------|----------|
| `'V'` | V | 116 | Video4Linux2 媒體 |
| `'o'` | o | 85 | 各式裝置控制 |
| `'a'` | a | 69 | 音訊/聲音 |
| `'p'` | p | 64 | 各式協定 |
| `'U'` | U | 55 | USB 裝置/gadget |
| `'f'` | f | 38 | 檔案操作/framebuffer |
| `'i'` | i | 32 | 輸入裝置 |
| `'T'` | T | 30 | TTY/終端 |
| `'E'` | E | 18 | Input events |
| `'b'` | b | 7 | Binder/binderfs |

## Usage Examples

**使用者空間呼叫 Binder ioctl：**
```c
struct binder_write_read bwr = { ... };
ioctl(binder_fd, BINDER_WRITE_READ, &bwr);
```

**核心內驅動註冊 ioctl handler：**
```c
static const struct file_operations binder_fops = {
    .unlocked_ioctl = binder_ioctl,
    .compat_ioctl = compat_ptr_ioctl,
};
```

## Android-Specific Notes

ACK 中的 Android 專屬 ioctl 介面集中在 3 個檔案：`drivers/android/binder.c`（212KB）、`include/uapi/linux/android/binder.h`、`include/uapi/linux/android/binderfs.h`。Binder 的 55+ 命令（13 ioctl + 21 BC + 23 BR）構成 Android IPC 的完整通訊協定。

## Cross-References

- [Binder](../entities/binder.md) — Binder ioctl 的完整實現分析
- [Binderfs](../entities/binderfs.md) — Binderfs 控制介面
- [Filesystems](../subsystems/filesystems.md) — VFS ioctl 派發機制
- [sysfs/procfs](sysfs-procfs.md) — 另一種核心-使用者空間介面
- [Syscalls](syscalls.md) — ioctl 系統呼叫入口
