# MemGuard V4 Core Design Snapshot

> **Status**: Active — current production architecture  
> **Date**: 2025–2026  
> **MCP Version**: v0.5.0  
> **Skill Version**: v4.2.0  
> **Purpose**: Document V4 architecture decisions for evolutionary review and V0.5.0+ planning

---

## V4 Architecture Overview

V4 evolved from V3's "Git-Native Memory Logger" to **"Memory Lifecycle Manager"**:

- All state persisted in Markdown (unchanged from V3)
- Human-readable, Git-diff friendly (unchanged from V3)
- **Task state machine**: 6 statuses with terminal states and auto-archival
- **ADR state machine**: 5 statuses with transition validation
- **Validation Framework**: Pre-mutation validation with 7 validators
- **Archive governance**: 3-section format (Completed/Superseded/Cancelled) with global dedup
- **Reference linking**: Tasks can reference ADRs or other Tasks for causal chains
- **Inverted Search Index**: O(1) term pre-filtering + scoring

V4 solved the **"memory rot"** problem: without lifecycle management, `context.md` grew to 50+ tasks, Done tasks accumulated indefinitely, and ADRs had no structured state transitions.

---

## V4 Data Model

### TaskStatus (6 variants)

```rust
enum TaskStatus {
    Todo,
    InProgress,
    Done,
    Blocked,       // ← V4.0: blocked by external dependency
    Superseded,    // ← V4.1: replaced by better solution
    Cancelled,     // ← V4.1: abandoned with no replacement
}
```

**Active states** (appear in bootstrap `active_tasks`):
- `Todo` — Not yet started
- `InProgress` — Currently being worked on
- `Blocked` — Blocked by external dependency or prerequisite task

**Terminal states** (immutable, auto-archived):
- `Done` — Completed successfully → archived to `## Completed`
- `Superseded` — Replaced by better solution → archived to `## Superseded`
- `Cancelled` — Abandoned with no replacement → archived to `## Cancelled`

**Why 6 statuses**: V3 only had Todo/InProgress/Done. V4.0 added `Blocked` to express external dependencies without creating duplicate tasks. V4.1 added `Superseded` and `Cancelled` to distinguish "completed" from "abandoned" and "replaced", enabling Decision Traceability.

### ADR / AdrStatus (5 variants)

```rust
enum AdrStatus {
    Proposed,
    Accepted,
    Superseded,
    Rejected,   // ← V4: explicit rejection (not just "not active")
    Archived,   // ← V4: historical record
}
```

**Valid transitions**:
```
Proposed → Accepted | Rejected
Accepted → Superseded | Archived
Rejected → Proposed (resubmission with material changes)
Superseded → * (terminal)
Archived → * (terminal)
```

**Backward compatibility**: `"active"` (V3 string) deserializes to `Accepted`, serializes to `"Accepted"`.

### SupersededInfo + Reference

```rust
struct SupersededInfo {
    reference: Reference,    // Task or ADR that caused the supersession
    reason: String,          // Human-readable explanation
}

enum Reference {
    Task(String),   // e.g., Reference::Task("TASK-015")
    Adr(String),    // e.g., Reference::Adr("ADR-053")
}
```

**Design rationale**: `Reference` is the generic entity pointer. When an ADR replaces an old approach, the old tasks are marked `Superseded` with `reference = Adr("ADR-XXX")` and a reason. Future agents can follow the chain: old task → ADR → new tasks.

**Serialization**: `{"type":"Adr","id":"ADR-053"}` — tagged serde for extensibility.

### Trap (V4 extended)

```rust
struct Trap {
    error_signature: String,
    context: String,
    solution: String,
    root_cause: String,     // ← V4: why it happened
    prevention: String,     // ← V4: how to avoid in future
}
```

**Why root_cause + prevention**: V3 Traps were "bug reports" without institutional depth. V4 makes Traps reusable knowledge by requiring root cause analysis and prevention guidelines.

---

## V4 Archive Format

### tasks_archive.md (3-section format, V4.1)

```markdown
# Archived Tasks

## Completed
### 2026-06-02
- [Done] [TASK-001] Implemented login feature

## Superseded
### 2026-06-02
- [Superseded] [TASK-011] Old approach A
  Superseded by: ADR-053
  Reason: Ground Truth generation redesigned

## Cancelled
### 2026-06-02
- [Cancelled] [TASK-028] Electron Desktop Support
```

**Key decisions**:
- **3 top-level sections**: Completed / Superseded / Cancelled have different semantics. Superseded has an inheritance chain (follow `superseded_by`). Cancelled has no replacement.
- **Date subsections**: Grouped by archival date for chronological scanning.
- **Global dedup by task ID**: If TASK-001 already exists in any section, it won't be duplicated. Keeps earliest occurrence.

### decisions_archive.md

Stale ADRs (Superseded/Rejected/Archived) auto-migrated from `decisions.md`.

---

## V4 Validation Framework

**7 validators** (registered in `ValidatorRegistry`, short-circuit on first error):

| Validator | Detects |
|-----------|---------|
| `EmptyTaskId` | `TaskCreated` with empty `id` |
| `DuplicateTaskId` | `TaskCreated` with ID already in `active_tasks` |
| `AdrActiveConflict` | Same ID `Accepted` ADR exists with different content |
| `AdrRejectedRepeat` | Same content as existing `Rejected` ADR |
| `AdrInvalidTransition` | ADR state transition not in `valid_transitions()` |
| `TaskTerminalTransition` | ← V4.1: Transition FROM terminal state (Done/Superseded/Cancelled) |
| `SupersededByRequired` | ← V4.1: `TaskUpdated` to `Superseded` missing `superseded_by` |

**Why pre-validation**: All validators run BEFORE acquiring write locks. Pure functions, no side effects. If validation fails, no state is mutated and no flush is triggered.

---

## V4 Search (Inverted Index)

V3 used brute-force O(N) search. V4 introduced `SearchIndex`:

```rust
struct SearchIndex {
    terms: HashMap<String, Vec<(EntryType, usize)>>,
}
```

**Build**: Tokenize ADR (title/context/decision/tags) and Trap (5 fields) → map term → (type, idx).

**Search**: Tokenize query → lookup terms → union candidates → `score_match_v3()` scoring → sort.

**Fallback**: If query tokens all miss, brute-force scan (V3 behavior).

**Performance**: 500 items build < 200ms; common term query < 50ms.

---

## V4 Cleanup CLI

`memguard cleanup` — manual data quality tool:

**Pipeline**: Scan → Analyze → Report → Confirm → Apply → Rebuild Cache

**Detects**:
- Done tasks still in `context.md` → archive to `tasks_archive.md`
- Stale ADRs in `decisions.md` → migrate to `decisions_archive.md`
- Duplicate ADRs → keep high-priority, mark others Superseded

**Safety**:
- `--dry-run`: report only
- Auto backup to `.memguard/backups/YYYYMMDD-HHMMSS/`
- Interactive confirmation
- Idempotent: second run → "No issues found"
- Cache rebuild: immediate `runtime_state.json` + `search_index.json` rewrite

---

## V4 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **6 Task statuses** | V3's 3 statuses couldn't express "blocked" or "replaced". Terminal states prevent infinite accumulation. |
| **Auto-archive on terminal** | Done/Superseded/Cancelled tasks are removed from `active_tasks` and appended to `tasks_archive.md`. Keeps `context.md` focused. |
| **Superseded ≠ Cancelled** | Superseded has an inheritance chain (follow `superseded_by` to replacement). Cancelled is a dead end. Different reasoning paths. |
| **`Reference` enum for linking** | Generic `{type, id}` pointer. Extensible for future entity types without schema migration. |
| **Validation pre-mutation** | Validators run before write locks. Fail-fast prevents corrupt state from ever reaching disk. |
| **3-section archive** | Semantically distinct sections make archive human-scannable and machine-parseable. |
| **Backward compat for "active"** | `"active"` deserializes to `Accepted`. Old V3 ADRs work without manual migration. |
| **Terminal state immutability** | Once Done/Superseded/Cancelled, a task cannot transition back. Prevents accidental resurrection of dead work. |

---

## V4 Limitations (Motivation for V0.5.0+)

### 1. No Task Lookup

**Problem**: `runtime_bootstrap` filters out terminal tasks. `runtime_query_memory` only searches ADRs and Traps — NOT tasks. Agent cannot distinguish:

```text
"TASK-011 not found"  ← 从未存在？还是已归档？还是被废弃？
```

**Impact**: Agent forced to violate SOP by directly editing `memory/*.md`.

**V0.5.0 solution**: `task_lookup` MCP tool + `TaskIndex` (O(1) ID lookup across active + archive).

### 2. No Entity History

**Problem**: Agent sees `TASK-011: Superseded` but cannot answer:
```text
"Why Superseded?"
"When was it changed?"
"Who/what caused the change?"
```

**Impact**: Agent loses causal reasoning ability for historical decisions.

**V0.5.0 solution**: `task_lookup` returns `superseded_by` + `archived_date`.
**V0.6.0 solution**: Event sourcing with `task_events.md` append-only log.

### 3. Skill is Monolithic

**Problem**: `SKILL.md` is ~490 lines (~5000-7000 tokens). Agent loads all lifecycle rules, archive formats, and validation details every session — even when only doing Bootstrap + Query.

**Impact**: Context window pressure. Agent may ignore important rules because they're buried in a wall of text.

**V0.5.0 solution**: Progressive Disclosure — core `SKILL.md` (~1000 tokens) as Router, detailed rules in `references/*.md` loaded on-demand.

### 4. Reference is Hardcoded

**Problem**: `Reference` enum only supports `Task` and `Adr`. Future entity types (Trap, Decision, Regime) require enum extension.

**Impact**: Breaking change every time a new entity type is introduced.

**V0.5.0 solution**: Generalize `Reference` to `EntityRef` with open `entity_type` string/enum.

---

## V4 → V0.5.0 Migration Notes

- **No breaking changes**: V0.5.0 is additive (TaskLookup tool + TaskIndex). Existing `memory/*.md` files work unchanged.
- **TaskIndex rebuilds on bootstrap**: If `.memguard/` cache is stale or missing, `TaskIndex` is rebuilt from `memory/*.md` scan.
- **Skill restructuring**: Old monolithic `SKILL.md` will be replaced by Router + references/. Agents using remote URL will auto-load new structure.

---

## File Index (V4 Era)

| File | Lines | Responsibility |
|------|-------|---------------|
| `src/models.rs` | ~374 | Domain structs: `TaskStatus` (6), `AdrStatus` (5), `Reference`, `SupersededInfo`, `RuntimeEvent` |
| `src/engine/state_manager.rs` | ~1,877 | State machine, debounced flush, ADR transitions, 7 validators, bootstrap |
| `src/engine/projection.rs` | ~1,291 | Markdown ↔ Rust, 3-section archive, global dedup, phase canonicalization |
| `src/engine/validator.rs` | ~180 | `Validator` trait + `ValidatorRegistry` |
| `src/engine/validators/*.rs` | ~7×100 | 7 concrete validators |
| `src/mcp/server.rs` | ~1,048 | MCP JSON-RPC, 3 tools, initialize auto-correction |
| `src/search/index.rs` | ~580 | `SearchIndex` + inverted index |
| `src/search/scorer.rs` | ~180 | `score_match_v3` + `ngram_jaccard` |
| `src/cli/cleanup.rs` | ~695 | Manual cleanup CLI with 14 integration tests |
| `memguard/SKILL.md` | ~490 | Agent SOP (monolithic, ~5000-7000 tokens when loaded) |

---

*Document Version*: 1.0  
*Last Updated*: 2026-06-06  
*Author*: Lhy  
*Next Review*: V0.5.0 Entity Integrity implementation
