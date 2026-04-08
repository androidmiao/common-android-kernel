---
type: entity
kernel_path: drivers/dma-buf/heaps/, drivers/dma-buf/dma-heap.c
config_option: CONFIG_DMABUF_HEAPS, CONFIG_DMABUF_HEAPS_SYSTEM, CONFIG_DMABUF_HEAPS_CMA
upstream: yes
related:
  - ../entities/ashmem.md
  - ../subsystems/memory-management.md
  - ../concepts/memory-allocation.md
last_updated: 2026-04-09
---

# DMA-BUF Heaps — 使用者空間 DMA 緩衝區分配

## Overview

DMA-BUF Heaps 是現代化的核心記憶體分配框架，提供使用者空間 API 用於分配可在多個裝置間共享的 DMA 緩衝區。它是 Android 舊版 Ion 分配器的上游替代品 `[upstream]`，採用模組化設計：框架層提供裝置介面與 heap 註冊機制，後端 heap 提供實際分配策略。

在 Android 中，DMA-BUF Heaps 主要用於 camera、display、GPU 等多媒體子系統的記憶體共享。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/dma-buf/dma-heap.c` | 414 | 框架層：heap 註冊、字元裝置、ioctl |
| `drivers/dma-buf/heaps/system_heap.c` | 556 | System Heap：buddy 分配器後端 |
| `drivers/dma-buf/heaps/cma_heap.c` | 448 | CMA Heap：連續記憶體分配後端 |
| `drivers/dma-buf/heaps/Kconfig` | — | 配置選項 |

## Implementation Details

### 框架層（dma-heap.c）

**核心資料結構：**
- **`struct dma_heap`**（dma-heap.c:38-47）— 代表一個 heap：字元裝置、refcount、操作函式表
- **全域 heap 列表**（dma-heap.c:49）— 所有已註冊的 heap
- **xarray**（dma-heap.c:53）— minor number → heap 映射

**關鍵函式：**

| 函式 | 行號 | 用途 |
|------|------|------|
| `dma_heap_add()` | :302-390 | 註冊新 heap，建立字元裝置 |
| `dma_heap_buffer_alloc()` | :79-98 | 呼叫 heap->ops->allocate() |
| `dma_heap_ioctl_allocate()` | :139-157 | 處理 `DMA_HEAP_IOCTL_ALLOC` |
| `dma_heap_open()` | :122-137 | 透過 xarray 查找 minor 對應的 heap |
| `dma_heap_init()` | :397-413 | 分配 chrdev 區域、建立 class |

### System Heap（system_heap.c）

從 buddy allocator（`alloc_pages()`）分配頁面，支援 cached 和 uncached 兩種模式。

**核心資料結構：**
- **`struct system_heap_buffer`**（system_heap.c:27-37）— 已分配緩衝區：DMA attachment 列表、sg_table、vmap 狀態
- **`struct dma_heap_attachment`**（system_heap.c:39-46）— 每裝置 DMA 映射狀態

**分配策略：**
- Order 陣列 `{8, 4, 0}` = 1MB、64KB、4KB 頁面（system_heap.c:59）
- 優先分配高 order 頁面以減少 IOMMU page table 項目，失敗時降級
- SWIOTLB bounce buffer 偵測：首次 `map_dma_buf()` 時檢查是否發生 bouncing

**關鍵函式：**

| 函式 | 行號 | 用途 |
|------|------|------|
| `system_heap_do_allocate()` | :392-490 | 核心分配邏輯 |
| `system_heap_attach()` | :102-130 | 建立 sg_table |
| `system_heap_map_dma_buf()` | :146-170 | 映射至裝置 |
| `system_heap_vmap()` | :301-327 | 建立核心虛擬映射 |

**初始化**（system_heap.c:526-554）：建立兩個 heap — `"system"`（cached）和 `"system-uncached"`。

### CMA Heap（cma_heap.c）

從 CMA（Contiguous Memory Allocator）分配實體連續記憶體。

**核心資料結構：**
- **`struct cma_heap`**（cma_heap.c:48-51）— 包裝 CMA 分配器指標
- **`struct cma_heap_buffer`**（cma_heap.c:53-63）— CMA 頁面、頁面陣列、頁面計數

**關鍵函式：**

| 函式 | 行號 | 用途 |
|------|------|------|
| `cma_heap_allocate()` | :299-387 | 透過 `cma_alloc()` 分配連續頁面 |
| `cma_heap_vm_fault()` | :187-196 | VM fault handler（pfn 映射） |
| `add_cma_heaps()` | :418-442 | 註冊 default 和 reserved CMA 區域 |

### 與舊版 Ion 的關係

| 特性 | Ion（已廢棄） | DMA-BUF Heaps |
|------|-------------|---------------|
| 架構 | 單體核心分配器 | 模組化框架 + 可插拔 heap |
| API | 自訂 Ion ioctl | 標準 DMA-BUF ioctl |
| 資料結構 | Ion 自訂結構 | 標準 sg_table + dma_buf |
| System Heap | Ion system heap | `system_heap.c`（buddy） |
| Carveout Heap | Ion carveout heap | `cma_heap.c`（CMA） |
| Uncached | 不支援 | `"system-uncached"` heap |

## Userspace Interface

- **裝置**：`/dev/dma_heap/system`、`/dev/dma_heap/system-uncached`、`/dev/dma_heap/<cma-name>`
- **分配**：`ioctl(heap_fd, DMA_HEAP_IOCTL_ALLOC, &alloc_data)` 回傳 dma_buf fd
- **使用**：取得的 fd 可透過 `mmap()` 映射至使用者空間，或傳遞給其他裝置驅動

## Android-Specific Notes

DMA-BUF Heaps 為上游 Linux 元件 `[upstream]`，但在 Android 中扮演核心角色：

- **多媒體記憶體共享**：Camera HAL → GPU → Display 的零複製 buffer 傳遞
- **取代 Ion**：Android 12+ 遷移至 DMA-BUF Heaps
- **Uncached heap**：`"system-uncached"` 為 Android 新增特性，避免 CPU cache 同步開銷
- **Vendor heap**：廠商可註冊自訂 heap（如 Samsung SLSI heap、Qualcomm secure heap）

## Configuration

| 選項 | 預設 | 說明 |
|------|------|------|
| `CONFIG_DMABUF_HEAPS` | Y | DMA-BUF Heaps 框架 |
| `CONFIG_DMABUF_HEAPS_SYSTEM` | Y | Buddy 分配器後端 |
| `CONFIG_DMABUF_HEAPS_CMA` | — | CMA 後端（依賴 `DMA_CMA`） |

## Cross-References

- [Ashmem](ashmem.md) — 舊版共享記憶體（已廢棄），DMA-BUF Heaps 取代 Ion 但未直接取代 Ashmem
- [記憶體管理子系統](../subsystems/memory-management.md) — Buddy 和 CMA 分配器基礎
- [記憶體分配概念](../concepts/memory-allocation.md) — GFP 標誌、CMA、DMA 池
