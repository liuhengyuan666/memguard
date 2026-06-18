---
name: memguard
description: |
  **MEMORY MANAGEMENT SOP** — Structured project memory with ADR-based decisions,
  trap tracking, and controlled Explore↔Execution workflows. Activates when the
  user asks about project decisions, task status, memory, session context, or
  mentions a previously discussed topic. WHEN: Architectural memory, ADR tracking,
  task lifecycle management, project state continuity, cross-session context
  recovery. INVOKES: memguard_runtime_bootstrap (session start),
  memguard_runtime_query_memory (before decisions/writing code),
  memguard_runtime_commit_event (after decisions/task updates).
license: MIT
compatibility: opencode
metadata:
  version: 4.3.0
  author: Lhy
  requires-mcp: ["memguard"]
  tags:
    [
      memory,
      adr,
      context-management,
      hallucination-guard,
      workflow,
      agent-sop,
      mcp,
    ]
---

# MemGuard Skill — Agent SOP Router

> **You have been equipped with the `memguard` MCP server (3 tools).
> This SOP tells you WHEN and HOW to use them.**
>
> **MCP tools**: `memguard_runtime_bootstrap` | `memguard_runtime_query_memory`
> | `memguard_runtime_commit_event`
>
> **Detailed rules** (ADR lifecycle, Task lifecycle, Trap format, Archive format,
> Task lookup) are in `references/*.md` — load on demand when needed.
>
> **Design philosophy**: continuity > statelessness · decisions > conversation
> history · active context > historical detail

---

## 0. Activation

If this document is visible in the current context, MemGuard is already active.
**The activation decision has already been made. Do not re-evaluate whether
MemGuard should be used.**

**Do NOT call `skill("memguard")`.** Proceed directly with the
Bootstrap → Query → Commit workflow below.

This SOP and the MCP tools are the instruction layer and the execution layer,
respectively. Both are already available.

---

## 1. Quick Trigger Guide

Match the user's intent to the minimum MemGuard workflow:

| User asks about | Required action |
| --------------- | --------------- |
| Existing task or task status | `bootstrap()` + `task_lookup()` |
| Architecture, design, or technology choice | `bootstrap()` + `query_memory()` |
| Refactor or core module change | `bootstrap()` + `query_memory()` |
| New feature or library introduction | `bootstrap()` + `query_memory()` |
| Bug or error with a reusable fix | `bootstrap()` + `query_memory()` → `commit_event(TrapRecorded)` |
| Status update on current phase | `bootstrap()` |
| ADR discussion or decision reversal | `bootstrap()` + `query_memory()` |

When in doubt, call `bootstrap()` first. It is cheap and idempotent.

---

## 2. Knowledge Source Priority (Hard Rule)

```
1. User instruction          (explicit override)
2. Active ADRs               (Accepted / Proposed)
3. Traps                     (recorded failures & solutions)
4. Search results            (external docs, examples, libraries)
5. Model internal knowledge  (training data, generic patterns)
```

**Active ADRs are binding**. If an active ADR says "use Tencent datasource",
search results recommending EastMoney **MUST NOT** override the ADR without user
confirmation. Before launching any `explore` / `librarian` / `oracle` agent for
architecture-related work, **MUST** first call `memguard_runtime_query_memory()`.

---

## 3. Session Start Protocol

**MUST** call `memguard_runtime_bootstrap()` as the **very first action** in any
session, after context loss, or on first interaction with a new project.

Do **NOT** generate any code or propose any architecture before the bootstrap
result is received. Read and acknowledge:
- `current_phase` — the project's current operational phase
- `constraints` — architectural constraints that **MUST** be respected
- `latest_adr` — the most recent architecture decision
- `adr_count` / `trap_count` / `memory_health` — project maturity signals
- `cache_status` — cache health signal (`Healthy`, `RebuildRequired`, `Corrupted`)
- `active_tasks` — tasks being tracked (read last; decisions are more important)

### Cache Status Handling

If `cache_status` is present in the bootstrap response and is not `Healthy`:

- `RebuildRequired` — cache is stale but recoverable. Avoid committing new
  state changes until the cache is rebuilt (inform the user).
- `Corrupted` — cache may be inconsistent. Treat memory as uncertain; do not
  modify tasks or ADRs. Continue in read-only mode using session context and
  inform the user that MemGuard needs repair.

Do **NOT** read `memory/*.md` or `.memguard/*.json` directly to bypass the MCP.

**MUST announce** the bootstrap summary in this exact format:
```
Memory context loaded:
- Phase: {current_phase}
- ADRs: {adr_count} (active/superseded/archived if memory_health present)
- Traps: {trap_count}
- Active Tasks: {active_tasks.length}
```

---

## 4. Pre-Decision / Pre-Coding Check

**MUST** call `memguard_runtime_query_memory(query_intent="...")` **before**:
- Proposing a new architecture decision
- Modifying any core module (auth, database driver, routing, data model)
- Introducing a new library, framework, or external dependency
- Revisiting a topic that was discussed in prior sessions

### ADR Consultation Contract

After calling `query_memory()` and before committing a new ADR:

1. Review any relevant active, superseded, or rejected ADRs returned.
2. In your rationale, explicitly explain whether the new proposal:
   - **Follows** existing ADRs (no conflict)
   - **Extends** existing ADRs (additive change)
   - **Supersedes** an existing ADR (requires marking the old one as Superseded)
3. If the query returns a `Rejected` ADR with content similar to your proposal,
   explain the **material difference** that justifies revisiting it.

A new `AdrCommitted` event without this relationship explanation is incomplete.

---

## 5. State Commit Triggers

**MUST** call `memguard_runtime_commit_event` immediately when:

| Event | event_type | When |
|-------|-----------|------|
| Architecture decision | `AdrCommitted` | After any technology selection, API design, architectural tradeoff, or fallback strategy. Even if uncertain, commit as `Proposed`. |
| New task created | `TaskCreated` | After decomposing work into trackable tasks. Tasks always start as `Todo`. |
| Task status change | `TaskUpdated` | Any transition. Terminal transitions (→Done/Superseded/Cancelled) are one-way. `superseded_by` **required** for `Superseded`. |
| Bug or error with fix | `TrapRecorded` | Non-trivial bugs where the fix is reusable knowledge |
| Phase transition | `PhaseChanged` | Switching between Explore/Execution modes |

**Detailed payload schemas** → see `references/adr-lifecycle.md`, `references/task-lifecycle.md`, `references/trap-rules.md`.

### Commit Receipt (SHOULD)

After a successful `commit_event`, you SHOULD briefly acknowledge the commit
in your response (e.g., `⪢ MemGuard: TaskUpdated TASK-018`). This improves
observability during debugging but is not required in user-facing output if
it would add noise.

---

## 5.1 MCP Payload Cheat Sheet

When constructing `commit_event` payloads, these fields are the most commonly
missed. Copy the exact field names:

| Event | Required fields in `payload` |
|-------|------------------------------|
| `TaskCreated` | `id` |
| `TaskUpdated` | `task_id`, `new_status` |
| `AdrCommitted` | `id`, `status` |
| `TrapRecorded` | `error_signature`, `solution` |

Do not guess field names. If you are unsure, load the corresponding reference.

---

## 5.2 Error Recovery Rule

If `memguard_runtime_commit_event` returns a validation error (`-32602` or
similar):

1. **Do NOT guess the payload schema.**
2. Load the corresponding reference file (see Section 10).
3. Retry **exactly once** using the documented schema.
4. If validation still fails, inform the user and stop retrying.

---

## 6. Task Integrity (v0.5.0+)

Before creating or updating a task:

1. Call `memguard_runtime_task_lookup({"task_id": "TASK-XXX"})`
2. If `found: true` and `location: "Archived"`:
   - `status: "Superseded"` → follow `superseded_by.reference` to replacement
   - `status: "Done"` → do not recreate unless requirements changed
   - `status: "Cancelled"` → do not recreate unless decision reversed
3. If `found: false` → safe to create

**Never** update a task without checking its location first. Terminal tasks cannot
be modified.

**Detailed lookup guide** → `references/task-lookup.md`.

---

## 7. Write Policy

**MUST NOT** read or write `memory/*.md` files directly. All memory operations
go through MCP tools.

**MUST NOT** commit these as events:
- Speculative ideas (unvalidated)
- Temporary thoughts
- Unresolved exploration
- Duplicated information
- Transient debugging details
- Lint fixes, typo corrections, variable renames, formatting changes

`.memguard/*.json` is a runtime cache derived from `memory/*.md`. It is
automatically managed — do **NOT** read or edit it.

---

## 7.1 Completion Checkpoint

Before responding with work completion, phase completion, progress summary
involving completed work, or task handoff:

Silently verify:
- [ ] Any task status changed → `commit_event(TaskUpdated)` if not done
- [ ] Any new task created → `commit_event(TaskCreated)` if not done
- [ ] Any architecture decision made → `commit_event(AdrCommitted)` if not done
- [ ] Any non-trivial bug fixed → `commit_event(TrapRecorded)` if not done

Do NOT conclude without these checks.

---

## 7.2 Bootstrap Shadow Summary

After `bootstrap()`, maintain an internal shadow summary:

- `current_phase`: ...
- `constraints`: ...
- `latest_adr`: ...
- `active_task_count`: N

If context compression occurs and the summary is unavailable, **MUST rerun
`bootstrap()`**. It is cheap and idempotent.

---

## 8. Execution Order

```
1. memguard_runtime_bootstrap()           → load current state
2. memguard_runtime_query_memory(...)     → check for relevant decisions/traps
3. Determine mode (Explore vs Execution)
4. Commit any new decisions               → if nontrivial design choice made,
                                             commit AdrCommitted IMMEDIATELY
5. Write code or explore solutions
6. memguard_runtime_commit_event(...)     → persist remaining decisions, tasks,
                                             traps, phase changes
7. Session self-check                     → verify all commits completed
```

---

## 9. Exception Handling

If any MemGuard MCP tool fails (timeout, error response, or unavailable):

1. **Inform the user** that memory synchronization is temporarily unavailable.
2. **Continue the task** using session context and any memory already loaded.
3. **Treat memory state as potentially stale** — do not assume the absence of
   an ADR or task means it does not exist.
4. **Retry memory synchronization later** if appropriate (e.g., before the
   next significant decision or at session end).

Memory continuity is preferred. Task execution should not halt solely because
MemGuard is unavailable.

---

## 10. Reference Routing Rules

Loading the correct reference before acting is not optional guidance — it is a
rule. Use this decision tree:

| If you are doing this... | Then you MUST load... |
| ------------------------- | --------------------- |
| Creating or transitioning an ADR | `references/adr-lifecycle.md` |
| Creating or transitioning a task | `references/task-lifecycle.md` |
| Using `task_lookup` or interpreting lookup results | `references/task-lookup.md` |
| Recording a trap | `references/trap-rules.md` |
| Installing or registering MemGuard | `references/deployment.md` |

**SHOULD load** when reasoning about archived entities or running cleanup:
`references/archive-format.md`.

Failure to load the relevant reference before committing an event is a
compliance violation.

### Quick Reference Index

| File | Contains |
|------|----------|
| `references/adr-lifecycle.md` | ADR state machine & transitions |
| `references/task-lifecycle.md` | Task 6-status lifecycle |
| `references/trap-rules.md` | Trap recording guidelines |
| `references/archive-format.md` | Archive structure & cleanup |
| `references/task-lookup.md` | Task lookup usage guide |
| `references/deployment.md` | Installation & registration |

---

## 11. Compliance

The MemGuard workflow is part of the project architecture.

When memory exists:

- Bootstrap before planning.
- Query before creating entities.
- Commit after decisions.

Skipping these steps may result in duplicate tasks, conflicting ADRs, or
inconsistent project state. The goal is not rule-following — it is keeping the
memory graph accurate so future sessions start from a reliable baseline.

---

## 12. Success Criteria

A compliant session usually contains:

1. `memguard_runtime_bootstrap()` — at session start.
2. `memguard_runtime_query_memory(...)` — before architecture decisions or
   revisiting prior discussions.
3. `memguard_runtime_task_lookup({"task_id": "TASK-XXX"})` — when discussing
   or changing a task.
4. `memguard_runtime_commit_event(...)` — when ADRs, tasks, traps, or phase
   state change.

If a session ends without items 2–4 being called where relevant, treat it as a
sign that memory may be out of sync with the work performed.

**Recommended action on anomalies**: If `bootstrap()` reveals duplicate task IDs,
inconsistent statuses, or unexpected active tasks, you SHOULD call
`query_memory()` to verify the inconsistency before proceeding. If the anomaly
is unclear or might reflect an intentional user state, ask the user before
diving deeper.

---

## 13. Integration

For installation and registration instructions, see
`references/deployment.md`. Deployment details do not belong in the Router.
