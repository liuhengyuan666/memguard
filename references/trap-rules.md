# Trap Recording Reference

> Load this reference when you encounter a non-trivial error that should be recorded for future agents.

## When to Record a Trap

Record a Trap (not an ADR) when **ALL** of the following are true:

1. You encountered an error that was unexpected
2. The error has a clear signature and solution
3. The error is likely to recur in the future

**Do NOT** record:
- One-off typos or formatting issues
- Lint warnings with obvious fixes
- Temporary environment problems
- Errors caused by known incomplete work

## Trap Structure

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

## Required vs Recommended Fields

| Field | Required | Purpose |
|-------|----------|---------|
| `error_signature` | ✅ Yes | Short, searchable identifier (e.g., "NPE in auth handler") |
| `context` | ✅ Yes | When and where the error occurred |
| `solution` | ✅ Yes | How to fix it |
| `root_cause` | Recommended | Why it happened — deeper analysis |
| `prevention` | Recommended | How to avoid recurrence |

## Committing a Trap

Use `memguard_runtime_commit_event` with:

```json
{
  "event_type": "TrapRecorded",
  "payload": {
    "error_signature": "...",
    "context": "...",
    "solution": "...",
    "root_cause": "...",
    "prevention": "..."
  }
}
```

**Backward compatibility**: `root_cause` and `prevention` default to empty string if omitted.
