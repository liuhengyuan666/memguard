# MemGuard Diagnostic — Payload Errors

> **When to load**: `memguard_runtime_commit_event` returns `-32602` (Invalid params) or a validation error.

## Common Error Signatures & Fixes

### Error: `-32602` — Missing required field

**Symptom**: The payload is missing a field that the schema requires.

**Most common causes**:

| Event | Common mistake | Correct field |
|-------|----------------|---------------|
| `TaskUpdated` | `status` | `new_status` |
| `TaskCreated` | missing `id` | `id` (required) |
| `AdrCommitted` | missing `context` or `decision` | both required |
| `TrapRecorded` | `error` | `error_signature` |

**Fix**:
1. Verify the exact field name from the table above.
2. Do **not** guess — copy the exact field name.
3. Retry once with the corrected payload.

### Error: `Invalid field type`

**Symptom**: A field exists but has the wrong type (e.g., string instead of number, boolean instead of string).

**Fix**:
1. Check the schema in the corresponding reference file (`task-lifecycle.md`, `adr-lifecycle.md`, `trap-rules.md`).
2. Correct the type and retry once.

### Error: `Unknown event_type`

**Symptom**: The `event_type` string does not match a known event.

**Valid event types**:
- `TaskCreated`
- `TaskUpdated`
- `AdrCommitted`
- `TrapRecorded`
- `PhaseChanged`

**Fix**:
1. Verify the exact spelling and case of the event type.
2. If the event type is not in the list above, it is not supported by the current MCP version.

## Recovery Protocol

1. **Do NOT guess** the correct schema.
2. Load this file (`diagnostics/payload-errors.md`) or the relevant reference.
3. Identify the exact field or type mismatch.
4. Retry **exactly once** with the corrected payload.
5. If it still fails, inform the user and stop retrying.

## Related

- `SKILL.md` Section 5.2: Error Recovery Rule
- `references/task-lifecycle.md`: Correct payload schema for tasks
- `references/adr-lifecycle.md`: Correct payload schema for ADRs
- `references/trap-rules.md`: Correct payload schema for traps
- `tests/TS-002-payload-recovery.md`: Regression test for this scenario
- `tests/fixtures/payload-error/TS-002-invalid-payload.json`: Reproducible invalid payload
