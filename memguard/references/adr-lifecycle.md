# ADR Lifecycle Reference

> Load this reference when you need to create, update, or transition an ADR.

## ADR Statuses

```
Proposed → Accepted → Superseded
             ↓
         Rejected
             ↓
         Archived
```

| Status | Meaning |
|--------|---------|
| `Proposed` | Under discussion, not yet committed. Can transition to `Accepted` or `Rejected`. |
| `Accepted` | Active and binding. **MUST** be followed. Can transition to `Superseded` or `Archived`. |
| `Superseded` | Replaced by a newer ADR. Terminal state. Reference the superseding ADR id. |
| `Rejected` | Explicitly rejected. Can be resubmitted as `Proposed` with material changes. |
| `Archived` | Historical record, no longer applicable. Terminal state. |

## Valid Transitions

- `Proposed` → `Accepted` | `Rejected`
- `Accepted` → `Superseded` | `Archived`
- `Rejected` → `Proposed` (resubmission)
- `Superseded` → * (terminal — no further transitions)
- `Archived` → * (terminal — no further transitions)

## When Changing an Accepted ADR

**DO NOT** create a parallel active ADR. Instead:

1. Query the existing ADR via `memguard_runtime_query_memory`
2. Create a new ADR with the updated decision
3. Mark the old ADR as `Superseded`

This preserves the decision chain: old ADR → new ADR → implementation.

## Committing an ADR

Use `memguard_runtime_commit_event` with:

```json
{
  "event_type": "AdrCommitted",
  "payload": {
    "id": "ADR-XXX",
    "title": "...",
    "status": "Proposed",
    "context": "...",
    "decision": "...",
    "tags": ["..."]
  }
}
```

**Backward compatibility**: `"active"` deserializes to `Accepted`.

## Warning: Rejected Repeat

If you propose an ADR with the **same content** as a previously `Rejected` ADR, the commit will fail. You **MUST** explain the material difference.
