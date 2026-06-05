# Archive Format Reference

> Load this reference when you need to understand how archived tasks and ADRs are structured.

## Tasks Archive (`tasks_archive.md`)

### 3-Section Format

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

### Section Semantics

| Section | Meaning | Has Replacement |
|---------|---------|-----------------|
| `## Completed` | Task finished successfully | No |
| `## Superseded` | Task replaced by better solution | Yes — follow `Superseded by` |
| `## Cancelled` | Task abandoned | No |

**Why 3 sections**: Superseded and Cancelled have fundamentally different semantics. A Superseded task has an inheritance chain (follow the `superseded_by` reference). A Cancelled task is a dead end.

### Global Deduplication

Tasks are deduplicated by ID across **all sections**. If TASK-001 already exists in `## Completed`, it will not be duplicated in `## Superseded`.

The **earliest** occurrence is kept.

## Decisions Archive (`decisions_archive.md`)

Stale ADRs (status: `Superseded`, `Rejected`, `Archived`) are automatically migrated from `decisions.md` to `decisions_archive.md`.

Active ADRs (`Accepted`, `Proposed`) remain in `decisions.md`.

## Cleanup CLI

Use `memguard cleanup --dry-run` to scan for hygiene issues:

- Done tasks still in `context.md`
- Stale ADRs still in `decisions.md`
- Duplicate ADRs

**Safety**: `--dry-run` reports without modifying. Apply creates a backup in `.memguard/backups/YYYYMMDD-HHMMSS/`.
