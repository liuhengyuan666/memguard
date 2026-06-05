---
name: memguard
description: |
  **MEMORY MANAGEMENT SOP** — Structured project memory with ADR-based decisions,
  trap tracking, and controlled Explore↔Execution workflows. Activates when the
  user asks about project decisions, task status, memory, session context, or
  mentions a previously discussed topic. WHEN: "remember this", "record this
  decision", "what did we decide about", "check memory", "update task status",
  "start new session", "bootstrap memory", "project memory", "what's the current
  phase", "any open tasks". INVOKES: memguard_runtime_bootstrap (session start),
  memguard_runtime_query_memory (before decisions/writing code), 
  memguard_runtime_commit_event (after decisions/task updates).
license: MIT
compatibility: opencode
metadata:
  version: 4.2.0
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

## 1. Knowledge Source Priority (Hard Rule)

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

## 2. Session Start Protocol

**MUST** call `memguard_runtime_bootstrap()` as the **very first action** in any
session, after context loss, or on first interaction with a new project.

Do **NOT** generate any code or propose any architecture before the bootstrap
result is received. Read and acknowledge:
- `current_phase` — the project's current operational phase
- `constraints` — architectural constraints that **MUST** be respected
- `latest_adr` — the most recent architecture decision
- `adr_count` / `trap_count` / `memory_health` — project maturity signals
- `active_tasks` — tasks being tracked (read last; decisions are more important)

**MUST announce** the bootstrap summary in this exact format:
```
Memory context loaded:
- Phase: {current_phase}
- ADRs: {adr_count} (active/superseded/archived if memory_health present)
- Traps: {trap_count}
- Active Tasks: {active_tasks.length}
```

---

## 3. Pre-Decision / Pre-Coding Check

**MUST** call `memguard_runtime_query_memory(query_intent="...")` **before**:
- Proposing a new architecture decision
- Modifying any core module (auth, database driver, routing, data model)
- Introducing a new library, framework, or external dependency
- Revisiting a topic that was discussed in prior sessions

If the query returns a superseded/rejected ADR relevant to your proposal:
**MUST** explain the material difference before proceeding.

---

## 4. State Commit Triggers

**MUST** call `memguard_runtime_commit_event` immediately when:

| Event | event_type | When |
|-------|-----------|------|
| Architecture decision | `AdrCommitted` | After any technology selection, API design, architectural tradeoff, or fallback strategy. Even if uncertain, commit as `Proposed`. |
| New task created | `TaskCreated` | After decomposing work into trackable tasks. Tasks always start as `Todo`. |
| Task status change | `TaskUpdated` | Any transition. Terminal transitions (→Done/Superseded/Cancelled) are one-way. `superseded_by` **required** for `Superseded`. |
| Bug or error with fix | `TrapRecorded` | Non-trivial bugs where the fix is reusable knowledge |
| Phase transition | `PhaseChanged` | Switching between Explore/Execution modes |

**Detailed payload schemas** → see `references/adr-lifecycle.md`, `references/task-lifecycle.md`, `references/trap-rules.md`.

---

## 5. Task Integrity (v0.5.0+)

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

## 6. Write Policy

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

## 7. Execution Order

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

## 8. References (Load On Demand)

| File | When to Load |
|------|-------------|
| `references/adr-lifecycle.md` | Creating or transitioning an ADR |
| `references/task-lifecycle.md` | Creating or transitioning a task |
| `references/trap-rules.md` | Recording a trap |
| `references/archive-format.md` | Understanding archive structure or running cleanup |
| `references/task-lookup.md` | Using `task_lookup` tool or interpreting lookup results |

---

## 9. Integration

Place this file in `.opencode/skills/memguard/` (per project) or
`~/.config/opencode/skills/memguard/` (global). Alternatively, add the
remote URL to your `opencode.json`:

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/main/"
    ]
  }
}
```

The MCP server must be registered separately in `opencode.json`. See
`opencode.json.example` for a complete dual-layer configuration template.
