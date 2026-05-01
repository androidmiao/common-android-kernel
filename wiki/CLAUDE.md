# Android Common Kernel Wiki — Schema & Workflows

> **Target codebase:** Android Common Kernel (ACK) — `common-android-mainline`, Linux 6.19-rc8
> **Purpose:** Incrementally build a persistent, interlinked knowledge base that captures the software architecture, implementation details, design patterns, and Android-specific extensions of the ACK.

---

## 1. Guiding Principles

1. **Source code is the ground truth.** The wiki summarises what the code *does*, not what documentation *says* it does. Always cite file paths and line ranges.
2. **Accumulate, don't re-derive.** Every analysis session should leave behind wiki pages that future sessions can build on.
3. **Cross-reference aggressively.** A page about the Binder driver should link to the page about `struct binder_proc`, the page about the IPC subsystem concept, and the page about vendor hooks — and vice versa.
4. **Distinguish upstream Linux from Android additions.** The ACK is a fork; always note whether a component is upstream Linux, an Android-specific patch (`ANDROID:` tag), a backport (`BACKPORT:`/`FROMGIT:`), or a GKI/vendor-hook extension.

---

## 2. Directory Layout

```
wiki/
├── CLAUDE.md            # This file — schema & workflows
├── index.md             # Master catalogue of every wiki page
├── log.md               # Chronological record of all operations
├── overview.md          # High-level architecture map of the entire kernel
│
├── subsystems/          # One page per kernel subsystem
│   ├── scheduler.md
│   ├── memory-management.md
│   ├── networking.md
│   ├── filesystems.md
│   ├── block-layer.md
│   ├── ipc.md
│   ├── security.md
│   └── ...
│
├── concepts/            # Cross-cutting design concepts & mechanisms
│   ├── gki.md                    # Generic Kernel Image architecture
│   ├── vendor-hooks.md           # Vendor hook framework
│   ├── kconfig-and-build.md      # Build system, defconfigs, Bazel, Kleaf
│   ├── module-system.md          # Loadable modules, symbol exports
│   ├── locking-primitives.md
│   ├── interrupt-handling.md
│   ├── memory-allocation.md
│   ├── rcu.md
│   ├── tracing-and-ftrace.md
│   ├── bpf.md
│   ├── rust-in-kernel.md
│   └── ...
│
├── entities/            # Specific, notable components
│   ├── binder.md
│   ├── binderfs.md
│   ├── ashmem.md  (if present)
│   ├── ion-dmabuf.md
│   ├── cgroup-controllers.md
│   └── ...
│
├── data-structures/     # Key kernel data structures
│   ├── task_struct.md
│   ├── mm_struct.md
│   ├── sk_buff.md
│   ├── binder_proc.md
│   ├── inode.md
│   ├── page.md
│   └── ...
│
├── apis/                # Internal kernel APIs & syscall interfaces
│   ├── ioctl-interfaces.md
│   ├── netlink.md
│   ├── sysfs-procfs.md
│   ├── kfuncs-bpf.md
│   └── ...
│
├── android/             # Android-specific topics
│   ├── gki-modules-list.md
│   ├── abi-stability.md
│   ├── debug-kinfo.md
│   ├── vendor-hook-catalogue.md
│   ├── android-patches-policy.md
│   └── ...
│
├── analyses/            # Filed query results, comparisons, deep-dives
│   ├── binder-transaction-flow.md
│   ├── scheduler-vs-upstream.md
│   └── ...
│
└── sources/             # Summary pages for ingested source files/docs
    ├── src-drivers-android-binder-c.md
    └── ...
```

---

## 3. Page Templates

### 3.1 Subsystem Page (`subsystems/*.md`)

```markdown
---
type: subsystem
kernel_path: <top-level directory, e.g. "kernel/sched/">
upstream: <yes | partial | no>
android_patches: <count or "none">
vendor_hooks: <list of relevant hook headers>
related: [<links to concept, entity, data-structure pages>]
last_updated: <ISO date>
---

# <Subsystem Name>

## Purpose
<One paragraph: what this subsystem does in the kernel.>

## Directory Map
<Table or tree showing key files/subdirectories with one-line descriptions.>

## Architecture
<Prose description of the high-level design. Diagrams encouraged (Mermaid).>

## Key Data Structures
<Bulleted list linking to data-structures/ pages.>

## Key Code Paths
<Numbered walk-throughs of important flows, citing file:line.>

## Android-Specific Changes
<What the ACK adds or modifies vs. upstream.>

## Vendor Hooks
<Which hooks exist in this subsystem, what they allow vendors to override.>

## Configuration
<Relevant Kconfig options and their effects.>

## Cross-References
<Links to related pages.>
```

### 3.2 Concept Page (`concepts/*.md`)

```markdown
---
type: concept
scope: <kernel-wide | subsystem-specific>
related: [...]
last_updated: <ISO date>
---

# <Concept Name>

## Overview
<What the concept is and why it exists.>

## Mechanism
<How it works in detail. Code references.>

## Usage Patterns
<How other subsystems use this concept. Examples from the codebase.>

## Android Relevance
<Why this matters for Android / GKI / vendor integration.>

## Cross-References
```

### 3.3 Entity Page (`entities/*.md`)

```markdown
---
type: entity
kernel_path: <primary source location>
config_option: <Kconfig symbol, e.g. CONFIG_ANDROID_BINDER_IPC>
upstream: <yes | no | partial>
related: [...]
last_updated: <ISO date>
---

# <Entity Name>

## Overview
<What it is, its history, why it exists.>

## Source Layout
<File listing with one-line descriptions.>

## Implementation Details
<Deep-dive. Data flow, state machines, error handling.>

## Userspace Interface
<ioctl, sysfs, procfs, netlink — whatever applies.>

## Android-Specific Notes

## Cross-References
```

### 3.4 Data Structure Page (`data-structures/*.md`)

```markdown
---
type: data-structure
defined_in: <header file path>
size_approx: <bytes or "variable">
lifecycle: <how it's allocated and freed>
related: [...]
last_updated: <ISO date>
---

# `struct <name>`

## Purpose
<One paragraph.>

## Definition
<Abbreviated struct definition with key fields annotated. NOT a full copy — highlight the architecturally important fields.>

## Field Groups
<Group fields by function: identity, state, linkage, accounting, etc.>

## Lifecycle
<Creation → usage → destruction. Who allocates, who holds references, how it's freed.>

## Key Operations
<Functions that operate on this structure. file:line citations.>

## Relationships
<What other structures it embeds or points to. A small relationship diagram (Mermaid) if useful.>

## Cross-References
```

### 3.5 API Page (`apis/*.md`)

```markdown
---
type: api
direction: <kernel-to-user | kernel-internal | user-to-kernel>
related: [...]
last_updated: <ISO date>
---

# <API Name>

## Overview

## Interface Definition
<Function signatures, ioctl numbers, netlink families, etc.>

## Semantics
<Detailed behavioural description. Error conditions.>

## Usage Examples
<Where in the kernel (or userspace) this API is called.>

## Cross-References
```

### 3.6 Source Summary Page (`sources/*.md`)

```markdown
---
type: source
source_path: <path relative to kernel root>
lines: <line count>
ingested: <ISO date>
---

# Source: `<filename>`

## Summary
<What this file does.>

## Key Functions
<Table: function name | line range | purpose>

## Notable Implementation Details

## Open Questions
<Things that need further investigation.>
```

### 3.7 Analysis Page (`analyses/*.md`)

```markdown
---
type: analysis
question: <the question that motivated this analysis>
pages_consulted: [<list of wiki pages read>]
created: <ISO date>
---

# <Analysis Title>

<Free-form deep-dive. Should be self-contained and citable from other pages.>
```

---

## 4. Operations

### 4.1 Ingest (Source File → Wiki)

**Trigger:** User asks to analyse a specific kernel source file or directory.

**Flow:**

1. **Read** the target source file(s). For large files (>500 lines), read in sections.
2. **Discuss** key observations with the user — what the code does, interesting design choices, non-obvious details.
3. **Create or update** a source summary page in `sources/`.
4. **Create or update** all affected pages:
   - Subsystem page if one doesn't exist for this area
   - Entity pages for notable components found
   - Data structure pages for important structs
   - Concept pages for cross-cutting patterns observed
5. **Update** `index.md` — add new pages, update summaries of modified pages.
6. **Append** to `log.md`.

**File reading strategy for large files:**
- Start with the file header and includes (first ~50 lines) for context
- Read the struct/type definitions
- Identify the main entry points (init, probe, open, ioctl handlers)
- Read hot paths in detail
- Skim helper functions, noting their signatures and purpose
- For files >2000 lines, prioritise architecturally significant sections

### 4.2 Ingest (Directory / Subsystem → Wiki)

**Trigger:** User asks to analyse an entire subsystem or directory.

**Flow:**

1. **List** all files in the directory. Categorise by role (core, helpers, platform-specific, tests, headers).
2. **Read Kconfig** and **Makefile** first — they reveal the subsystem's structure and configuration space.
3. **Read key headers** to understand the data structures and API surface.
4. **Read core implementation files**, following the strategy above.
5. **Build the subsystem page** incrementally as you read.
6. **Create entity, data-structure, concept pages** as needed.
7. **Update** index and log.

### 4.3 Query

**Trigger:** User asks a question about the kernel.

**Flow:**

1. **Read `index.md`** to find relevant wiki pages.
2. **Read** those pages.
3. If existing pages are insufficient, **read source code** directly to fill the gap.
4. **Synthesise** an answer with citations to both wiki pages and source files.
5. If the answer is substantial and reusable, **file it** as an analysis page in `analyses/`.
6. Update index and log if new pages were created.

### 4.4 Lint

**Trigger:** User asks for a health check, or periodically after many ingests.

**Checklist:**

- [ ] Orphan pages (in wiki but not linked from anywhere)
- [ ] Dead links (links to pages that don't exist)
- [ ] Stale pages (last_updated is old; source code may have changed)
- [ ] Missing pages (concepts/entities mentioned but without their own page)
- [ ] Inconsistencies (two pages making contradictory claims)
- [ ] Thin pages (pages with only a stub or placeholder content)
- [ ] Coverage gaps (major subsystems or files not yet ingested)
- [ ] Missing vendor hook documentation
- [ ] Cross-reference completeness (are bidirectional links in place?)

---

## 5. Conventions

### 5.1 Naming
- File names: lowercase, hyphen-separated. e.g. `memory-management.md`, `task_struct.md` (underscores for C identifiers).
- Use the kernel's own terminology. Don't rename concepts.

### 5.2 Code References
- Always use `file:line` format: `kernel/sched/core.c:4521`
- For ranges: `kernel/sched/core.c:4521-4580`
- For functions: `` `schedule()` @ kernel/sched/core.c:6723 ``

### 5.3 Links
- Use relative markdown links from the current wiki page, e.g. `[Binder](entities/binder.md)` from `wiki/CLAUDE.md`
- Cross-references should be bidirectional: if A links to B, B should link back to A.

### 5.4 Upstream vs. Android
- Tag sections or claims with `[upstream]` or `[android]` when the distinction matters.
- In frontmatter, `upstream: yes` means the component exists in mainline Linux; `upstream: no` means it's Android-only; `upstream: partial` means the ACK version differs from upstream.

### 5.5 Diagrams
- Use Mermaid syntax for inline diagrams.
- Common diagram types: flowcharts for code paths, class diagrams for struct relationships, sequence diagrams for IPC flows.

### 5.6 Frontmatter
- Every page MUST have YAML frontmatter with at least `type`, `related`, and `last_updated`.
- `related` is a list of relative paths to other wiki pages.

---

## 6. Prioritised Exploration Order

When starting from scratch, ingest in this order to build foundational knowledge first:

1. **Build system & GKI architecture** — `concepts/gki.md`, `concepts/kconfig-and-build.md`
   - Files: `Makefile`, `Kconfig`, `build.config.*`, `arch/arm64/configs/gki_defconfig`, `gki/`, `modules.bzl`, `bazel/`
2. **Android-specific components** — `android/` directory in wiki
   - Files: `drivers/android/`, `include/trace/hooks/`, vendor hooks
3. **Process management & scheduling** — `subsystems/scheduler.md`
   - Files: `kernel/sched/`, `include/linux/sched.h`, `kernel/fork.c`
4. **Memory management** — `subsystems/memory-management.md`
   - Files: `mm/`, `include/linux/mm.h`, `include/linux/mm_types.h`
5. **IPC (including Binder)** — `subsystems/ipc.md`, `entities/binder.md`
   - Files: `ipc/`, `drivers/android/binder.c`, `include/uapi/linux/android/binder.h`
6. **Filesystems** — `subsystems/filesystems.md`
   - Files: `fs/`, VFS layer, key filesystem implementations
7. **Networking** — `subsystems/networking.md`
   - Files: `net/core/`, `net/ipv4/`, `net/ipv6/`, `include/net/`
8. **Security** — `subsystems/security.md`
   - Files: `security/`, SELinux, capabilities
9. **Device driver framework** — `concepts/driver-model.md`
   - Files: `drivers/base/`, `include/linux/device.h`
10. **Block layer & I/O** — `subsystems/block-layer.md`
    - Files: `block/`, `io_uring/`

---

## 7. Special Considerations for ACK Analysis

### 7.1 Vendor Hook Analysis
The vendor hook framework (`include/trace/hooks/`) is central to Android GKI. When encountering vendor hooks:
- Document the hook's location (which function, which file)
- Document what data is exposed to the hook
- Explain what a vendor module could do with this hook
- Link back to the subsystem page

### 7.2 ABI Stability
GKI enforces ABI stability for kernel modules. When analysing:
- Note which symbols are exported (`EXPORT_SYMBOL_GPL`)
- Note KMI (Kernel Module Interface) implications
- Reference `android/abi_gki_aarch64*` files if present

### 7.3 Rust in Kernel
The ACK includes Rust support (`rust/`). When encountering Rust code:
- Document the Rust-C binding boundaries
- Note safety abstractions
- Link to `concepts/rust-in-kernel.md`

### 7.4 Scale Management
The kernel has 36,000+ `.c` files and 26,000+ `.h` files. To manage this:
- Don't try to ingest everything. Focus on what the user cares about.
- Use Kconfig/Makefile to understand what's compiled for a given config.
- Use `grep` / `Grep` tool to find call sites, definitions, and cross-references.
- The `index.md` file is the primary navigation tool; keep it well-organised.

---

## 8. Index & Log Formats

### index.md

```markdown
# Wiki Index

## Overview
- [Architecture Overview](overview.md) — High-level map of the entire kernel

## Subsystems
- [Scheduler](subsystems/scheduler.md) — Process scheduling, CFS, RT, deadline
- ...

## Concepts
- [GKI](concepts/gki.md) — Generic Kernel Image architecture
- ...

## Entities
- [Binder](entities/binder.md) — Android IPC mechanism
- ...

## Data Structures
- [`task_struct`](data-structures/task_struct.md) — Process/thread descriptor
- ...

## APIs
- ...

## Android-Specific
- ...

## Analyses
- ...

## Sources
- ...
```

### log.md

```markdown
# Wiki Log

## [YYYY-MM-DD] <operation> | <subject>
<Brief description of what was done. Pages created/updated.>
```

---

## 9. Session Startup Checklist

At the start of each session:

1. Read this file (`CLAUDE.md`) to recall the schema.
2. Read `index.md` to understand current wiki state.
3. Read `log.md` (last 10 entries) to understand recent activity.
4. Ask the user what they'd like to explore or ingest.
5. Proceed with the appropriate operation.

---

## 10. Mirroring to GitHub Repository

The wiki is mirrored to a separate GitHub-tracked repository for version
control and sharing. The mainline working tree is the **source of truth**;
the GitHub mirror is downstream.

### Paths

- **Source (truth):** `/Users/alex.miao/android-kernel/common-android-mainline/wiki/`
- **Mirror (GitHub):** `/Users/alex.miao/Documents/Documents - Alexmiao MacBook M4 Pro/GitHub/common-android-kernel/wiki/`

### Sync command

When the user asks to sync new/modified wiki content to the GitHub repo, run:

```bash
rsync -av --update --exclude='.DS_Store' \
  "/Users/alex.miao/android-kernel/common-android-mainline/wiki/" \
  "/Users/alex.miao/Documents/Documents - Alexmiao MacBook M4 Pro/GitHub/common-android-kernel/wiki/"
```

### Policy

- `--update`: only overwrite when the source file is newer (preserves any
  GitHub-side edits that happen to be newer; this should be rare).
- `--exclude='.DS_Store'`: never propagate macOS metadata.
- **No `--delete`**: files removed from the source are NOT auto-removed from
  the mirror. If the user wants pruning, they must explicitly request it.
- After syncing, verify file counts match (excluding `.DS_Store`):
  ```bash
  find <SRC> -type f ! -name '.DS_Store' | wc -l
  find <DST> -type f ! -name '.DS_Store' | wc -l
  ```

### Required Cowork access

The GitHub folder is not mounted by default in Cowork sessions. Before
syncing, request access with:

```
request_cowork_directory(path="/Users/alex.miao/Documents/Documents - Alexmiao MacBook M4 Pro/GitHub/common-android-kernel")
```

### When to sync

- After any meaningful wiki change (new pages, updated pages, log entries).
- Whenever the user explicitly requests it (e.g. "同步到 GitHub repo").
- Not automatically on every edit — wait for an explicit request or a
  natural checkpoint (end of an analysis session).
