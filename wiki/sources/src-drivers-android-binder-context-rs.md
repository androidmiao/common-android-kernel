---
type: source
source_path: drivers/android/binder/context.rs
lines: 180
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-process-rs.md
---

# Source: `drivers/android/binder/context.rs`

## Summary

Rust Binder 的 `Context` 類型，代表一個 binder 上下文（對應一個 binder 裝置如 `/dev/binder`、`/dev/hwbinder`）。管理 context manager 節點和所有使用此上下文的行程列表。

## Key Functions

| Function | Line | Purpose |
|----------|------|---------|
| `Context::new()` | 68 | 建立新上下文並加入全域列表 |
| `Context::deregister()` | 92 | 從全域列表移除 |
| `Context::register_process()` | 98 | 註冊行程到上下文 |
| `Context::deregister_process()` | 106 | 反註冊行程 |
| `Context::set_manager_node()` | 115 | 設定 context manager（含 SELinux 檢查） |
| `Context::unset_manager_node()` | 136 | 取消 context manager |
| `Context::get_manager_node()` | 141 | 取得 manager 節點引用 |
| `Context::for_each_proc()` | 151 | 迭代所有行程 |
| `Context::get_all_procs()` | 161 | 取得所有行程列表 |
| `Context::get_procs_with_pid()` | 172 | 按 PID 過濾行程 |
| `get_all_contexts()` | 28 | 取得所有上下文列表 |

## Notable Implementation Details

- **全域鎖**：`CONTEXTS` 使用 `kernel::sync::global_lock!` 巨集，是一個 `Mutex<ContextList>`，在模組初始化時 init。
- **Manager 結構**：`node: Option<NodeRef>`（context manager 節點）+ `uid: Option<Kuid>`（UID 限制，首次設定後鎖定）+ `all_procs: List<Process>`。
- **SELinux 整合**：`set_manager_node()` 呼叫 `security::binder_set_context_mgr()` 進行安全檢查。
- **UID 一致性**：若 context manager 已設定過，新的設定必須使用相同的 EUID。
- **ListArc 模式**：`Context` 實現 `ListArcSafe` 和 `ListItem`，可安全地加入和移除列表。
- **Pin-Init**：使用 `try_pin_init!` 巨集進行固定初始化。

## Open Questions

- 多上下文場景（binder/hwbinder/vndbinder）的隔離是否充分？
