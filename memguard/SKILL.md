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
  version: 4.1.0
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

# MemGuard v4 — Agent SOP (Standard Operating Procedure)

> **You have been equipped with the `memguard` MCP server (3 tools).
> This SOP tells you exactly WHEN and HOW to use them.**
>
> **MCP tools**: `memguard_runtime_bootstrap` | `memguard_runtime_query_memory`
> | `memguard_runtime_commit_event`
>
> **Design philosophy**: continuity > statelessness · decisions > conversation
> history · active context > historical detail

---

## 1. Memory As Context Anchor

Memory provides persistent operational context across sessions.

Agent **MUST**:
- Reference prior decisions before proposing alternatives
- Preserve architectural continuity
- Avoid contradicting established project structure

### Knowledge Source Priority (Hard Rule)

When reasoning, proposing solutions, or launching search agents, the following
priority order is absolute. A lower-priority source **MUST NOT** override a
higher-priority source without explicit user confirmation.

```
1. User instruction          (explicit override)
2. Active ADRs               (Accepted / Proposed)
3. Traps                     (recorded failures & solutions)
4. Search results            (external docs, examples, libraries)
5. Model internal knowledge  (training data, generic patterns)
```

**Consequences**:
- If an active ADR says "use Tencent datasource", search results recommending
  EastMoney **MUST NOT** override the ADR without user confirmation.
- If a Trap records "do not use feature X because of bug Y", model knowledge
  saying "feature X is standard" **MUST NOT** override the Trap.
- Before launching any `explore` / `librarian` / `oracle` agent for a task that
  touches architecture, ADRs, or previously modified modules, **MUST** first
  call `memguard_runtime_query_memory()` to load relevant project memory.

Explicit user instructions **MAY** override memory. When conflict exists:
1. Surface the conflict explicitly
2. Request confirmation if ambiguity exists
3. Record the override into memory if confirmed

---

## 2. Session Start Protocol

**MUST** call `memguard_runtime_bootstrap()` as the **very first action** in any
session, after context loss, or on first interaction with a new project.

Do **NOT** generate any code or propose any architecture before the bootstrap
result is received. Read and acknowledge:
- `current_phase` — the project's current operational phase
- `constraints` — architectural constraints that **MUST** be respected
- `latest_adr` — the most recent architecture decision
- `adr_count` / `trap_count` — if `adr_count` is 0 and you are making nontrivial
  design decisions this session, the project needs its first ADRs
- `active_tasks` — tasks being tracked (read last; decisions are more important)

**MUST announce** the bootstrap summary in this exact format:
```
Memory context loaded:
- Phase: {current_phase}
- ADRs: {adr_count}
- Traps: {trap_count}
- Active Tasks: {active_tasks.length}
```
Do **NOT** omit `trap_count` even if it is zero. A project with zero traps is
information — it means no reusable failure memory has been recorded yet.

---

## 3. Pre-Decision / Pre-Coding Check

**MUST** call `memguard_runtime_query_memory(query_intent="...")` **before**:
- Proposing a new architecture decision
- Modifying any core module (auth, database driver, routing, data model)
- Introducing a new library, framework, or external dependency
- Revisiting a topic that was discussed in prior sessions
- Writing code that touches an area covered by an existing ADR

Pass your intent as a concise natural-language description (e.g.,
`"authentication token validation"`, `"database migration strategy"`).

If the query returns a superseded/rejected ADR relevant to your proposal:
**MUST** explain the material difference before proceeding. Do **NOT**
re-propose a previously rejected approach without this explanation.

---

## 4. State Commit Triggers

**MUST** call `memguard_runtime_commit_event` immediately when:

| Event | event_type | payload | When |
|-------|-----------|---------|------|
| Architecture decision | `AdrCommitted` | `{ id, title, status, context, decision, tags }` | **After any technology selection, API design decision, architectural tradeoff, or fallback strategy.** Even if uncertain, commit as `Proposed`. If the project has `adr_count: 0`, every nontrivial design decision this session should produce an ADR. |
| New task created | `TaskCreated` | `{ id, description }` | **After decomposing work into trackable tasks.** Tasks are always created as `Todo` regardless of any `status` provided in the payload — use `TaskUpdated` to transition afterward. Task IDs should follow `TASK-XXX` format (e.g., `TASK-001`). Duplicate IDs are rejected. |
| Task status change | `TaskUpdated` | `{ task_id, new_status, superseded_by? }` | Any transition among active states (Todo↔InProgress↔Blocked). Terminal transitions (→Done/Superseded/Cancelled) are one-way. `superseded_by` is **required** when `new_status` is `Superseded`. |
| Bug or error with fix | `TrapRecorded` | `{ error_signature, context, solution }` | Non-trivial bugs where the fix is reusable knowledge |
| Phase transition | `PhaseChanged` | `{ new_phase }` | Switching between Explore/Execution modes, or between planning/implementation/verification |

### ADR Status Lifecycle

```
Proposed → Accepted → Superseded
                        Deprecated
                        Rejected
```

- **Proposed**: Under discussion, not yet committed
- **Accepted**: Active and binding — **MUST** be followed
- **Superseded**: Replaced by a newer ADR — reference the superseding ADR id
- **Deprecated**: No longer applicable but kept for historical record
- **Rejected**: Explicitly rejected — **MUST NOT** be re-proposed without
  explaining the material difference

### Phase canonical names (MUST use short identifiers)

| Phase | Meaning |
|-------|---------|
| `explore` | Divergence, uncertainty reduction, solution analysis |
| `plan` | Architecture design, task decomposition |
| `implement` | Deterministic implementation |
| `verify` | Testing, review, validation |
| `complete` | Delivered and verified |

**MUST NOT** use free-text phase names longer than 32 characters.

---

## 5. Explore ↔ Execution Mode Switching

Agent operates in two modes only:

| Mode | Purpose | Output structure |
|------|---------|-----------------|
| **Explore** | Divergence, uncertainty reduction, solution analysis | Problem framing + 2-5 candidate approaches + tradeoffs + assumptions + validation strategy |
| **Execution** | Deterministic implementation and delivery | Current objective + task breakdown + dependencies + constraints + execution plan |

**Switch Explore → Execution** ONLY when **ALL 3** conditions are met:
1. Solution has converged to 1-2 viable paths
2. Major uncertainty has been validated
3. MVP scope is sufficiently defined

**Switch Execution → Explore** when:
1. An unvalidated assumption is discovered
2. A previously accepted ADR is found to be invalid
3. A new major constraint is introduced mid-implementation

**MUST** call `memguard_runtime_commit_event` with `PhaseChanged` on every
mode transition.

---

## 6. Hallucination Guardrails (4-Layer Validation)

Before any major reasoning or implementation, **MUST** verify against memory:

### Layer 1 — Structure Verification
Before creating or modifying files:
- Verify against the project structure described in bootstrap's `constraints`
- Avoid inventing nonexistent modules or assuming architecture

### Layer 2 — Decision Verification
Before proposing architecture:
- Inspect relevant ADRs via `memguard_runtime_query_memory`
- Avoid contradicting any active ADR (`status: Accepted`)
- Reference ADRs with `[ADR-XXX]` notation to enable traceability

### Layer 3 — Technology Verification
Before stack-specific implementation:
- Verify tech constraints from `memguard_runtime_bootstrap`
- Avoid hallucinating dependencies, frameworks, or APIs not present in the project

### Layer 4 — Assumption Visibility
- All unverified claims **MUST** use `[ASSUMPTION: ...]`
- All external knowledge **MUST** cite the source
- Implicit assumptions are **forbidden**

---

## 7. Memory Write Policy

Agent **SHOULD** minimize unnecessary memory writes. Only stable, validated,
reusable information belongs in memory.

### Write Triggers
| Event | Commit as |
|-------|-----------|
| Confirmed architecture decision | `AdrCommitted` |
| Phase/goal change | `PhaseChanged` |
| Major architecture change | `AdrCommitted` |
| Non-trivial bug fix with reusable solution | `TrapRecorded` |
| New task created | `TaskCreated` |
| Task status change | `TaskUpdated` |

### Forbidden Writes
**MUST NOT** commit these as events:
- Speculative ideas (unvalidated)
- Temporary thoughts
- Unresolved exploration
- Duplicated information
- Transient debugging details
- Verbose execution logs
- Lint fixes, typo corrections, variable renames, formatting changes

### File Access
**MUST NOT** read or write `memory/*.md` files directly. All memory operations
go through the MCP tools: `memguard_runtime_bootstrap`,
`memguard_runtime_query_memory`, `memguard_runtime_commit_event`.

`.memguard/*.json` is a runtime cache derived from `memory/*.md`. It is
automatically managed — do **NOT** read or edit it.

---

## 8. Memory Compression

To prevent context collapse, memory **MUST** remain compact. When memory grows
excessively:
- Summarize obsolete ADRs (`status: superseded` and not referenced in >3 sessions)
- Archive deprecated context
- Remove duplicated terminology
- Retain active operational knowledge

Prefer: `active state > historical detail`

---

## 9. Session Self-Check

Before ending any session, **MUST** verify ALL of the following.
**ADR creation is the highest-priority check** — a session that made nontrivial
decisions without committing ADRs is incomplete:

- [ ] **Were new architecture decisions made?** → `memguard_runtime_commit_event { AdrCommitted }` — this is the single most important persistence action
- [ ] Were new tasks created? → `memguard_runtime_commit_event { TaskCreated }`
- [ ] Did goals or phase change? → `memguard_runtime_commit_event { PhaseChanged }`
- [ ] Were non-trivial errors encountered? → `memguard_runtime_commit_event { TrapRecorded }`
- [ ] Did any tasks change status? → `memguard_runtime_commit_event { TaskUpdated }`
- [ ] Are all `[ASSUMPTION: ...]` markers either validated or marked for next session?
- [ ] Does any implementation contradict an active ADR?
- [ ] Should any ADRs be superseded or deprecated?
- [ ] Is memory still compact? Should stale decisions be summarized?

**Do NOT consider the session complete until all items are checked.**

---

## 10. Recommended Execution Order

```
1. memguard_runtime_bootstrap()           → load current state
2. memguard_runtime_query_memory(...)     → check for relevant decisions/traps
3. Determine mode (Explore vs Execution)  → apply switching rules from §5
4. Commit any new decisions               → if a nontrivial design choice was
                                             made, commit an AdrCommitted event
                                             IMMEDIATELY (before writing code)
5. Write code or explore solutions        → apply guardrails from §6
6. memguard_runtime_commit_event(...)     → persist remaining decisions, tasks,
                                             traps, phase changes
7. Session self-check                     → verify §9 checklist
```

---

## 11. Constraints & Boundaries

Agent **MUST NOT** autonomously:
- Perform project management operations
- Manage CI/CD pipelines
- Manage Git hosting workflows (force-push, change branch protection)
- Integrate external services not present in the project
- Invent infrastructure not established in the project memory
- Rewrite architecture without justification and an ADR

Agent **MAY** provide plans and recommendations for these areas.

---

## 12. Integration

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

---

## Appendix: MCP Tool Quick Reference

| Tool | Call When | Key Parameter |
|------|-----------|---------------|
| `memguard_runtime_bootstrap` | Session start, context loss | `project_root` (optional) |
| `memguard_runtime_query_memory` | Before decisions, coding, proposing | `query_intent` (required) |
| `memguard_runtime_commit_event` | After decisions, task updates, traps, phase changes | `event_type` + `payload` (required) |

---

## ADR Lifecycle

ADR statuses: Proposed → Accepted → Superseded/Archived.

When changing an existing accepted decision:

DO NOT create a parallel active ADR.

Instead:
1. Query existing ADR
2. Create new ADR with updated decision
3. Mark old ADR as Superseded

Valid transitions:
- Proposed → Accepted
- Proposed → Rejected
- Accepted → Superseded
- Accepted → Archived
- Rejected → Proposed (resubmission)
- Superseded → * (terminal)
- Archived → * (terminal)

## Task Lifecycle

```
Todo
  ↓
InProgress
  ↓
Blocked
  ↓
Done        ──→ Archived to ## Completed
Superseded  ──→ Archived to ## Superseded
Cancelled   ──→ Archived to ## Cancelled
```

### Active States (appear in bootstrap `active_tasks`)
- `Todo` — Not yet started
- `InProgress` — Currently being worked on
- `Blocked` — Blocked by external dependency or prerequisite task

### Terminal States (immutable, removed from `active_tasks`, archived)
- `Done` — Task completed successfully
- `Superseded` — Task abandoned because a better solution exists
- `Cancelled` — Task abandoned with no replacement

### Terminal State Rules

**`Done`**:
- Automatically archived to `tasks_archive.md` under `## Completed`
- Do NOT keep Done tasks in `context.md`

**`Superseded`** (MUST provide `superseded_by`):
- Automatically archived to `tasks_archive.md` under `## Superseded`
- **MUST** include `superseded_by` field with `reference` and `reason`
- `reference.type`: `"Adr"` or `"Task"`
- `reference.id`: the replacing ADR or Task ID
- Example:
  ```json
  {
    "task_id": "TASK-011",
    "new_status": "Superseded",
    "superseded_by": {
      "reference": { "type": "Adr", "id": "ADR-053" },
      "reason": "Ground Truth generation redesigned"
    }
  }
  ```

**`Cancelled`**:
- Automatically archived to `tasks_archive.md` under `## Cancelled`
- No `superseded_by` required
- Use when a task is simply abandoned (e.g., user decides not to do it)

### Critical Rule

When an ADR replaces an existing implementation plan:
- **Do NOT** mark old tasks as `Done`
- **MUST** mark them as `Superseded` and reference the replacing ADR

This preserves Decision Traceability — future Agents can follow the causal chain from the old task to the new ADR to the new tasks.

### Archive Format

```markdown
# Tasks Archive

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

### Blocked Usage

Use `Blocked` status when:
- A task is blocked by an external dependency
- A task cannot proceed until another task completes

### State Transition Validation

The following transitions are **forbidden**:
- `Done` → any other status
- `Superseded` → any other status
- `Cancelled` → any other status

## Trap Recording Rules

Record a Trap (not an ADR) when:
- You encountered an error that was unexpected
- The error has a clear signature and solution
- The error is likely to recur

Trap structure:
```markdown
## Trap: {error_signature}

### Context
{when/where it happened}

### Root Cause
{why it happened}

### Solution
{how to fix it}

### Prevention
{how to avoid it in the future}
```

Required fields: error_signature, context, solution
Recommended fields: root_cause, prevention
