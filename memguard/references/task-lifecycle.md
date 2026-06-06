# Task Lifecycle Reference

> Load this reference when you need to create, update, or transition a task.

## Task Statuses

### Active States (appear in bootstrap `active_tasks`)

| Status | Meaning |
|--------|---------|
| `Todo` | Not yet started |
| `InProgress` | Currently being worked on |
| `Blocked` | Blocked by external dependency or prerequisite task |

### Terminal States (immutable, auto-archived)

| Status | Meaning | Archive Section |
|--------|---------|-----------------|
| `Done` | Completed successfully | `## Completed` |
| `Superseded` | Replaced by better solution | `## Superseded` |
| `Cancelled` | Abandoned with no replacement | `## Cancelled` |

**Critical rule**: Once a task reaches a terminal state, it **CANNOT** transition back. This prevents accidental resurrection of dead work.

## State Transitions

```
Todo
  ↓
InProgress ←→ Blocked
  ↓
Done        ──→ Archived to ## Completed
Superseded  ──→ Archived to ## Superseded
Cancelled   ──→ Archived to ## Cancelled
```

**Forbidden transitions**:
- `Done` → any other status
- `Superseded` → any other status
- `Cancelled` → any other status

## Superseded Tasks

When an ADR replaces an existing implementation plan:

- **Do NOT** mark old tasks as `Done`
- **MUST** mark them as `Superseded` and reference the replacing ADR

Example payload:

```json
{
  "event_type": "TaskUpdated",
  "payload": {
    "task_id": "TASK-011",
    "new_status": "Superseded",
    "superseded_by": {
      "reference": { "type": "Adr", "id": "ADR-053" },
      "reason": "Ground Truth generation redesigned"
    }
  }
}
```

This preserves Decision Traceability — future Agents can follow the causal chain from the old task to the new ADR to the new tasks.

## Creating a Task

Use `memguard_runtime_commit_event` with:

```json
{
  "event_type": "TaskCreated",
  "payload": {
    "id": "TASK-XXX",
    "description": "..."
  }
}
```

**Note**: New tasks are always created as `Todo` regardless of any `status` provided.

## Committing a Status Change

Use `memguard_runtime_commit_event` with:

```json
{
  "event_type": "TaskUpdated",
  "payload": {
    "task_id": "TASK-XXX",
    "new_status": "InProgress"
  }
}
```

For `Superseded`, `superseded_by` is **required**.
