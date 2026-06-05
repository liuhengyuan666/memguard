# Task Lookup Reference

> Load this reference when you need to query the status or location of a specific task.

## When to Call `task_lookup`

Call `memguard_runtime_task_lookup` (available in v0.5.0+) **before**:

- Creating a new task (check for duplicates)
- Updating a task (check current status and location)
- When a task "disappears" from bootstrap output

## Input

```json
{ "task_id": "TASK-011" }
```

## Result Classes

### Active

```json
{
  "found": true,
  "id": "TASK-004",
  "status": "InProgress",
  "location": "Active",
  "description": "Implement OAuth login"
}
```

Task is in `context.md` and can be updated.

### Archived (Done)

```json
{
  "found": true,
  "id": "TASK-001",
  "status": "Done",
  "location": "Archived",
  "archived_date": "2026-06-02",
  "description": "Implemented login feature"
}
```

Task completed. Do not recreate unless requirements have fundamentally changed.

### Archived (Superseded)

```json
{
  "found": true,
  "id": "TASK-011",
  "status": "Superseded",
  "location": "Archived",
  "archived_date": "2026-06-05",
  "description": "Old approach A",
  "superseded_by": {
    "reference": { "type": "Adr", "id": "ADR-053" },
    "reason": "Ground Truth generation redesigned"
  }
}
```

Task replaced by newer work. **Follow the chain**:
- If `reference.type` is `"Adr"` → read that ADR for replacement tasks
- If `reference.type` is `"Task"` → follow to the replacement task

### Archived (Cancelled)

```json
{
  "found": true,
  "id": "TASK-028",
  "status": "Cancelled",
  "location": "Archived",
  "archived_date": "2026-06-02",
  "description": "Electron Desktop Support"
}
```

Task abandoned. Do not recreate unless the decision to abandon is explicitly reversed.

### Not Found

```json
{
  "found": false,
  "id": "TASK-999",
  "message": "Task has never been created"
}
```

Safe to create. No historical record exists.

## Example Workflow

```
Agent needs to implement "login page"
→ task_lookup("TASK-015")
← { found: true, status: "Superseded", superseded_by: { type: "Adr", id: "ADR-053" } }
→ query_memory("ADR-053")
← ADR-053 replaced login page with OAuth integration (TASK-018)
→ Proceed with TASK-018
```

## Why This Matters

Without `task_lookup`, the only way to check a task's status is via `runtime_bootstrap`, which **filters out terminal tasks**. If you try to update a task that was already archived, you get:

```
Task not found: TASK-011
```

This error is ambiguous — it could mean the task never existed OR it was archived. `task_lookup` removes this ambiguity.
