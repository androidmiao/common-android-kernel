---
type: data-structure
defined_in: common/include/linux/mm_types.h:904
size_approx: variable; one instance per virtual memory area
lifecycle: allocated when mappings are created or split; freed after unmap/merge via VMA teardown paths
related:
  - mm_struct
  - page
  - maple_tree
  - file
  - memory-management
last_updated: 2026-04-25
---

# `struct vm_area_struct`

## Purpose

`struct vm_area_struct` describes one contiguous virtual address range in a process address space. VMAs are the units managed by `mmap()`, `munmap()`, page fault handling, file mappings, anonymous memory, stack growth, `mprotect()`, and many memory accounting paths.

In modern Linux, a process owns a set of VMAs through `mm_struct.mm_mt`, a Maple Tree keyed by virtual address range. Page fault handling starts from an address, finds the matching VMA, checks permissions and flags, then chooses the anonymous or file-backed fault path.

## Key Field Groups

| Group | Representative fields | Role |
|-------|-----------------------|------|
| Address range | `vm_start`, `vm_end` | Half-open virtual address range `[start, end)` |
| Access policy | `vm_flags` | Read/write/exec, shared/private, stack, huge page, lock and special mapping attributes |
| Backing object | `vm_file`, `vm_pgoff`, `anon_vma` | File-backed mapping metadata or anonymous reverse-mapping anchor |
| Operations | `vm_ops` | Filesystem or special mapping callbacks, especially fault handling |
| Ownership | `vm_mm` | Back pointer to the owning [`mm_struct`](mm_struct.md) |
| Synchronization | per-VMA lock fields, `mmap_lock` relationship | Supports finer-grained page fault concurrency when enabled |

## Lifecycle

VMAs are created by mapping operations such as `mmap()`, process image loading, stack setup, and shared-memory mappings. They may be split when only part of a range changes attributes, merged when adjacent compatible ranges become equivalent, and removed on `munmap()`, `execve()`, or process exit.

The VMA tree is protected primarily by `mm_struct.mmap_lock`, with newer kernels adding per-VMA locking to reduce contention on parallel page faults.

## Code Paths

Typical VMA lookup and use:

```text
page fault address
  -> find VMA in mm_struct.mm_mt
  -> validate vm_flags permissions
  -> dispatch to anonymous or file-backed fault path
```

Typical mapping update:

```text
mmap/munmap/mprotect
  -> take mmap_lock for write
  -> insert, split, merge, or remove vm_area_struct entries
  -> update page tables and accounting as needed
```

## Android Relevance

VMAs matter directly to Android performance because large apps and system services frequently fault many independent mappings in parallel. Per-VMA locking can reduce contention compared with forcing every page fault through the process-wide `mmap_lock`. Android-specific memory mechanisms such as Binder buffers, memfd/ashmem compatibility, DMA-BUF mappings, and app zygote mappings all rely on normal VMA and page fault machinery.

## Cross-References

- [Memory Management](../subsystems/memory-management.md)
- [`struct mm_struct`](mm_struct.md)
- [`struct page` / `struct folio`](page.md)
- [記憶體分配](../concepts/memory-allocation.md)
