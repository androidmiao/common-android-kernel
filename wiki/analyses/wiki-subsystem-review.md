---
type: analysis
question: "既有 Linux subsystem wiki 分析的品質、缺口與優先整理順序如何？"
pages_consulted:
  - index.md
  - subsystems/scheduler.md
  - subsystems/memory-management.md
  - subsystems/networking.md
  - subsystems/filesystems.md
  - subsystems/block-layer.md
  - subsystems/security.md
  - subsystems/ipc.md
  - subsystems/driver-framework.md
  - analyses/vendor-hooks-distribution.md
  - analyses/locking-patterns.md
  - analyses/security-hardening-strategy.md
  - analyses/memory-android-extensions.md
created: 2026-04-25
last_updated: 2026-04-25
---

# Subsystem Wiki 健康檢查

## 總評

`wiki/subsystems/` 已經形成可用的 Android Common Kernel 架構導覽層，而不是零散筆記。主要 Linux 子系統如 scheduler、memory management、networking、filesystems、block layer、security、IPC、driver framework 都已有獨立頁面；多數頁面包含 purpose、directory map、architecture、key data structures、code paths、Android-specific changes 與 cross references。

`wiki/analyses/` 的橫向分析也有價值，尤其是 vendor hooks 分佈、跨子系統鎖定模式、安全強化策略與 Android 記憶體擴展。這些頁面已經能回答「ACK 與 upstream Linux 差在哪裡」、「Android/GKI 為什麼這樣設計」這類跨頁問題。

## 強項

| 面向 | 評價 |
|------|------|
| 覆蓋範圍 | 核心子系統、架構、driver framework、firmware、headers、samples 均有頁面 |
| Android 視角 | 大多頁面會標出 GKI、vendor hooks、Android-specific patches、defconfig 關聯 |
| 架構導覽 | 多數頁面有 Mermaid 圖、目錄地圖、資料結構、主要流程 |
| 橫向分析 | `analyses/` 已能跨 subsystem 比較 vendor hooks、locking、安全與記憶體策略 |
| 可讀性 | 中文說明清楚，適合當作後續研究入口 |

## 主要缺口

### 1. 格式一致性

`scheduler.md` 與 `init.md` 是早期長文筆記風格，原本缺少 YAML frontmatter；`scheduler.md` 也缺少明確 cross-reference 區塊。這會讓 index、metadata 查詢與未來自動化檢查較難一致處理。

### 2. 壞連結

`memory-management.md` 原本連到不存在的 `../data-structures/folio.md` 與 `../data-structures/vm_area_struct.md`。其中 `folio` 已涵蓋在 `page.md`，而 `vm_area_struct` 值得補成獨立資料結構頁。

### 3. Survey 型頁面的證據密度較低

`arch-*`、`drivers-overview.md`、`firmware.md`、`include-headers.md`、`samples.md` 這類總覽頁較偏 survey，原本 source line references 較少。它們可讀性高，但若作為長期 wiki，應補上 evidence snapshot，讓讀者知道哪些結論對應哪些原始碼入口。

### 4. 統計與配置可能變 stale

頁面包含很多行數、檔案數、hook 數與 defconfig 選項。這些資訊對導覽有用，但隨 checkout 更新可能過期，需要定期用簡單工具檢查。

## 優先整理順序

1. **Wiki hygiene**：補齊 `scheduler.md` / `init.md` frontmatter，修掉壞連結。
2. **Cross-links**：補 `scheduler.md` 與 `task_struct`、locking、vendor hooks、BPF、GKI 等頁的連結。
3. **Evidence tables**：對 survey 型頁面補 evidence snapshot，每個大結論至少連到幾個 source path/line。
4. **Consistency checks**：跑 markdown link 檢查、frontmatter 檢查、dashboard count 檢查，留下結果。

## 交叉參考

- [Wiki Index](../index.md)
- [Scheduler](../subsystems/scheduler.md)
- [Init](../subsystems/init.md)
- [Memory Management](../subsystems/memory-management.md)
- [Vendor Hooks 跨子系統分佈分析](vendor-hooks-distribution.md)
- [跨子系統鎖定模式分析](locking-patterns.md)
