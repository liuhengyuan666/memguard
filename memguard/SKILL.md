---
name: memguard
description: Structured memory management and behavioral runtime contract for AI agents. Provides long-term context continuity, decision anchoring, hallucination reduction, and controlled Explore → Execution workflows.
license: MIT
compatibility: opencode
metadata:
  version: 2.0.0
  author: Lhy
  tags:
    [
      memory,
      adr,
      context-management,
      hallucination-guard,
      workflow,
      agent-runtime,
    ]
---

# MemGuard — Agent Memory & Runtime Spec

> Trigger:
> Activate automatically when `memory/` exists in project root.
> If absent, initialize during first project interaction.

> Goal:
> Maintain stable long-term context, reduce hallucinations,
> preserve architectural decisions, and enforce structured
> agent operating behavior across sessions.

---

# 1. Core Runtime Principles

## 1.1 Memory As Context Anchor

Memory provides persistent operational context.

Agent MUST:

- reference prior decisions before proposing alternatives
- preserve architectural continuity
- avoid contradicting established project structure

Explicit user instructions MAY override memory.

When conflict exists:

1. surface the conflict
2. request confirmation if ambiguity exists
3. record override into memory if confirmed

---

## 1.2 Decision Continuity

Before proposing new technical or product decisions:

- inspect `memory/decisions.md`
- check for active/rejected/superseded entries
- avoid repeating previously rejected approaches

If proposing a previously rejected idea,
Agent MUST explain the meaningful difference.

---

## 1.3 Explicit Assumptions

All unverified assumptions MUST use:

```text
[ASSUMPTION: ...]
```

Implicit assumptions are forbidden.

---

## 1.4 Dual-Mode Operation

Agent operates in TWO modes only:

| Mode | Purpose |
|---|---|
| Explore | divergence, uncertainty reduction, solution analysis |
| Execution | deterministic implementation and delivery |

Agent MUST adapt output structure to current mode.

---

## 1.5 Operational Boundaries

Agent MUST NOT autonomously:

- perform project management
- manage CI/CD pipelines
- manage git hosting workflows
- perform external service integration
- invent infrastructure not present in project
- rewrite architecture without justification

Agent MAY provide plans/recommendations for these areas.

---

# 2. Runtime Modes

## 2.1 Explore Mode

Use when:

- requirements are ambiguous
- architecture is unresolved
- trade-offs remain unclear
- major uncertainty exists

Output MUST include:

1. problem framing
2. 2-5 candidate approaches
3. trade-offs
4. assumptions
5. validation strategy
6. decision points

---

## 2.2 Execution Mode

Use when:

- implementation path is sufficiently clear
- architecture is mostly stable
- requirements are actionable

Output MUST include:

1. current objective
2. task breakdown
3. dependencies
4. architectural constraints
5. execution plan

---

## 2.3 Mode Transition Rules

Transition Explore → Execution ONLY when:

- solution converges to 1-2 viable paths
- major uncertainty is validated
- MVP scope is sufficiently defined

Mode transitions SHOULD update `memory/context.md`.

---

# 3. Memory Structure

```text
memory/
├── context.md
├── decisions.md
├── product.md
├── tech.md
├── structure.md
├── glossary.md
├── history/
└── archive/
```

---

## File Responsibilities

| File | Purpose | Strategy |
|---|---|---|
| context.md | current operational state | overwrite |
| decisions.md | ADR-style decisions | append-only |
| product.md | stable business knowledge | overwrite |
| tech.md | stable technical architecture | overwrite |
| structure.md | current project structure | overwrite |
| glossary.md | terminology normalization | append |
| history/ | milestone summaries only | append |
| archive/ | compressed historical memory | overwrite |

---

# 4. Memory Loading Strategy

Agent MUST load memory selectively.

## Minimum Required

Always load:

- `context.md`
- active sections of `decisions.md`

---

## Conditional Loading

Load ONLY when relevant:

| File | Load When |
|---|---|
| product.md | business/domain reasoning |
| tech.md | implementation/architecture |
| structure.md | file/module operations |
| glossary.md | terminology ambiguity |
| history/ | historical reasoning |
| archive/ | long-term context recovery |

---

## Loading Priority

Prefer:

1. active context
2. active decisions
3. current structure
4. compressed summaries
5. historical detail

---

# 5. Memory Write Policy

Agent SHOULD minimize unnecessary memory writes.

Only stable, validated, reusable information
belongs in memory.

---

## 5.1 Write Triggers

| Event | Target |
|---|---|
| confirmed decision | decisions.md |
| phase/goal change | context.md |
| major architecture change | tech.md |
| major structure change | structure.md |
| terminology stabilization | glossary.md |
| milestone completion | history/ |

---

## 5.2 Forbidden Writes

DO NOT write:

- speculative ideas
- temporary thoughts
- unresolved exploration
- duplicated information
- transient debugging details
- verbose execution logs

---

## 5.3 Write Rules

| File | Allowed Action |
|---|---|
| decisions.md | append-only |
| glossary.md | append |
| context.md | overwrite |
| tech.md | overwrite |
| structure.md | overwrite |
| archive/ | overwrite |
| history/ | new milestone files only |

---

# 6. ADR Decision Format

`memory/decisions.md`

```markdown
## [YYYY-MM-DD] Decision Title

Status: active | superseded | deprecated | rejected

### Context
...

### Alternatives
...

### Decision
...

### Rationale
...

### Consequences
...
```

---

# 7. Context Format

`memory/context.md`

```markdown
# Current State

Mode:
Current Goal:
Current Phase:

# Active Tasks

- ...

# Constraints

- ...

# Risks

- ...
```

---

# 8. Hallucination Guardrails

Before major reasoning or implementation,
Agent MUST verify against memory.

---

## 8.1 Structure Verification

Before creating/modifying files:

- verify against `structure.md`
- avoid inventing nonexistent modules
- avoid assuming architecture

---

## 8.2 Decision Verification

Before architecture suggestions:

- inspect relevant decisions
- avoid contradicting active ADRs

---

## 8.3 Technology Verification

Before stack-specific implementation:

- verify against `tech.md`
- avoid hallucinating dependencies/frameworks

---

## 8.4 Assumption Visibility

Unverified claims MUST use:

```text
[ASSUMPTION: ...]
```

---

# 9. Memory Compression

To prevent context collapse,
memory MUST remain compact.

When memory grows excessively:

- summarize obsolete decisions
- archive deprecated context
- compress historical records
- remove duplicated terminology
- retain active operational knowledge

Prefer:

```text
active state > historical detail
```

---

# 10. Project Initialization

## New Project

If no codebase exists:

1. initialize memory structure
2. request product + technical context
3. enter Explore mode

---

## Existing Project

If codebase exists:

1. initialize memory structure
2. scan project structure
3. infer technical stack
4. generate initial `structure.md`
5. generate initial `tech.md`
6. request user confirmation
7. determine runtime mode

---

# 11. Runtime Self-Check

Before session completion,
Agent SHOULD verify:

- were new decisions made?
- did goals change?
- did architecture change?
- did terminology stabilize?
- should memory be compressed?
- are assumptions marked?
- does implementation violate ADRs?

---

# 12. Recommended Runtime Behavior

Preferred execution order:

```text
1. Load minimum memory
2. Detect mode
3. Expand relevant memory
4. Reason
5. Execute
6. Update memory selectively
7. Compress if necessary
```

---

# 13. Integration

Place this file in:

```text
.opencode/skills/
```

or equivalent OpenAgent skill directory.

Agent SHOULD automatically load and apply
this runtime contract during sessions.

---

# 14. Design Philosophy

MemGuard prioritizes:

- continuity over statelessness
- decisions over conversation history
- active context over historical detail
- structure over improvisation
- controlled autonomy over unrestricted generation

The system is designed to reduce:

- hallucinated architecture
- repeated decision cycles
- context drift
- memory bloat
- implementation inconsistency

while preserving:

- adaptability
- user override capability
- iterative architecture evolution
- long-term project coherence