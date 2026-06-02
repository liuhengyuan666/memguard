# MemGuard V3 Design Snapshot

> **Status**: Archived — superseded by V4  
> **Date**: 2024–2025 (pre-V4 era)  
> **Purpose**: Preserve V3 architecture decisions for evolutionary review

---

## V3 Architecture Overview

V3 was the first production-ready version of MemGuard. Its core philosophy was **"Git-Native Memory Logger"**:

- All state persisted in Markdown
- Human-readable, Git-diff friendly
- Minimal schema — just 3 file types: `context.md`, `decisions.md`, `traps.md`
- Simple concurrency via `Arc<RwLock<T>>`
- 500ms debounced flush to disk

---

## V3 Data Model

### TaskStatus
```rust
enum TaskStatus {
    Todo,
    InProgress,
    Done,
}
```

**Limitation**: No way to express "blocked by external dependency". Agents would leave tasks in `InProgress` indefinitely or create duplicate tasks.

### ADR
```rust
struct ADR {
    id: String,
    title: String,
    status: String,        // "active" or "superseded" — free-form string
    context: String,
    decision: String,
    tags: Vec<String>,
}
```

**Limitation**: `status` was a string. No validation, no state machine, no way to express "rejected then resubmitted". The only operational distinction was "active" (shown in bootstrap) vs everything else (ignored).

### Trap
```rust
struct Trap {
    error_signature: String,
    context: String,
    solution: String,
}
```

**Limitation**: No structured root cause analysis, no prevention guidelines. Traps were essentially "bug reports" without the depth needed for institutional knowledge.

### RuntimeState
```rust
struct RuntimeState {
    current_phase: String,
    active_tasks: Vec<Task>,      // ALL tasks, including Done
    constraints: Vec<String>,
}
```

**Limitation**: `active_tasks` included Done tasks. Over time, `context.md` grew to 50+ tasks, making bootstrap output noisy and reducing signal-to-noise ratio.

---

## V3 Search

V3 search was **brute-force**:

```rust
for adr in decisions {
    score = score_match_v3(query, adr.title, adr.context, ...)
}
for trap in traps {
    score = score_match_v3(query, trap.error_signature, ...)
}
sort by score
take top N
```

**Characteristics**:
- O(N) per query
- No caching or indexing
- `score_match_v3` used exact word boundary + 3-gram Jaccard fuzzy matching
- Good enough for < 50 ADRs
- Slowed noticeably at > 100 ADRs

**Why it was sufficient for V3**: V3 targeted small projects (tens of ADRs, tens of tasks). The design trade-off was simplicity over performance.

---

## V3 Validation

V3 validation was **inline and hard-coded**:

```rust
// In state_manager.rs, inside apply_event
if let TaskCreated(task) = event {
    if task.id.is_empty() { return Err(...); }
    if state.active_tasks.iter().any(|t| t.id == task.id) { return Err(...); }
}
```

**Characteristics**:
- Validation logic mixed with state mutation
- Not reusable or testable in isolation
- No way for users to add custom validators
- Error messages were ad-hoc strings

**Why it was sufficient for V3**: Only 2 validation rules existed (empty ID, duplicate ID). The cost of extracting a framework outweighed the benefit.

---

## V3 Directory Structure

```
[Project Root]
├── memory/
│   ├── context.md
│   ├── decisions.md
│   └── traps.md
│
└── .memguard/
    ├── runtime_state.json
    └── search_index.json
```

**No archive mechanism**. Done tasks accumulated in `context.md`. Stale ADRs accumulated in `decisions.md`. There was no concept of "active" vs "historical" memory.

**Why it was sufficient for V3**: V3 was designed for short-lived projects (weeks to months). Memory growth was assumed to be slow enough that accumulation wouldn't be a problem.

---

## V3 → V4 Trigger Events

The following real-world observations triggered the V4 redesign:

1. **Project `hy_quant_analysis_system`** accumulated 33 Done tasks in `context.md`, making bootstrap output 80% noise
2. **ADR-032 / ADR-033 duplicate**: Two ADRs with identical titles but different IDs coexisted as "active" because there was no mechanism to detect or resolve the conflict
3. **Search latency**: Querying "vue" on a project with 24 ADRs took ~200ms (acceptable but approaching threshold)
4. **No migration path**: Once data quality degraded, the only fix was manual editing of Markdown files
5. **Agent confusion**: Without `Blocked` status, agents would repeatedly attempt tasks that were blocked by external dependencies

---

## Design Decisions That Survived V3 → V4

The following V3 decisions were intentionally preserved in V4:

| Decision | Rationale | V4 Status |
|---|---|---|
| Markdown as source of truth | Git diff friendly, human readable | ✅ Preserved |
| `Arc<RwLock<T>>` concurrency | Simple, proven, no deadlocks | ✅ Preserved |
| 500ms debounced flush | I/O batching without complexity | ✅ Preserved |
| Phase canonicalization | Prevents "planning" vs "plan" confusion | ✅ Preserved |
| `initialize` handshake auto-correction | Solves "exe in wrong directory" problem | ✅ Preserved |
| `score_match_v3` algorithm | Hybrid exact + fuzzy works well | ✅ Preserved (extracted to `search/scorer.rs`) |

---

## Design Decisions That Were Replaced

| V3 Decision | V4 Replacement | Rationale |
|---|---|---|
| Free-form ADR status string | `AdrStatus` enum (5 variants) | Prevent invalid states, enable state machine |
| 3 task statuses | 4 task statuses (+ `Blocked`) | Express blocked-by-dependency clearly |
| 3 Trap fields | 5 Trap fields (+ root_cause, prevention) | Deeper institutional knowledge |
| Inline validation | `ValidatorRegistry` + 5 validators | Testable, extensible, consistent errors |
| Brute-force search | `SearchIndex` + inverted index | O(1) pre-filter for scale |
| No archive mechanism | `tasks_archive.md` + `decisions_archive.md` | Separate active from historical memory |
| No migration tool | `memguard cleanup` CLI | Manual governance without auto-rewrite risk |

---

## Lessons Learned

1. **Simplicity scales until it doesn't**. V3's "just 3 files" design was elegant for small projects but created pain at scale. V4 adds structure (archive files, enums, state machines) without sacrificing the core Git-Native philosophy.

2. **Manual migration > auto-migration**. Early V4 designs considered auto-rewriting `memory/*.md` on MCP startup. This was rejected because: (a) silent data modification violates the "source of truth" principle, (b) agents rely on predictable file contents, (c) backup/rollback becomes complex. The `memguard cleanup` CLI tool was the compromise: manual trigger, preview, confirm, apply.

3. **Validation should be a framework, not inline**. V3's inline validation worked for 2 rules but wouldn't scale to the 5+ rules needed for ADR state machines. Extracting `trait Validator` early prevented technical debt.

4. **Search indexing should be decoupled from scoring**. V3 tightly coupled `score_match_v3` with the query loop. V4's separation of `SearchIndex` (pre-filter) and `scorer` (precision) allows future index implementations (e.g., persistent disk index) without changing the scoring algorithm.

---

## File Reference

This snapshot was created from:
- `memguard-mcp/architecture.md` (V3 version, pre-2026-06-02)
- `memguard-mcp/README.md` (V3 version, pre-2026-06-02)
- `memguard/SKILL.md` (V3 version, pre-2026-06-02)

The actual V3 source code is preserved in git tags: `v0.1.0` through `v0.2.3`.
