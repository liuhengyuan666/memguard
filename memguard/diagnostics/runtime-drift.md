# MemGuard Diagnostic — Runtime Drift

> **When to load**: `bootstrap()` returns unexpected state, duplicate IDs, stale cache, or `bootstrap` ≠ `task_lookup` results.

## Drift Patterns

### Pattern 1: Duplicate Task ID

**Symptom**: `bootstrap()` or `task_lookup()` reveals two tasks with the same ID (e.g., `TASK-011`).

**Root cause**: Manual edit of `memory/context.md`, merge conflict, or concurrent write without MCP serialization.

**Detection**:
```
bootstrap() → cache_status: "RebuildRequired"
            → anomaly: "duplicate_task_id"
            → duplicate_id: "TASK-011"
```

**Fix**:
1. Call `query_memory()` to verify the exact duplication.
2. If unambiguous (e.g., two `TASK-011` entries), inform the user.
3. Do **not** auto-delete without user confirmation.
4. Suggest: rename one to a new ID, or mark the newer one as superseded.

### Pattern 2: Stale Cache (`memory_mtime > cache_mtime`)

**Symptom**: `bootstrap()` returns `cache_status: "RebuildRequired"` even though `memory/*.md` was recently edited.

**Root cause**: `.memguard/runtime_state.json` was not rebuilt after an external edit to `memory/*.md`.

**Fix**:
1. Do **not** commit new state until the cache is rebuilt.
2. Inform the user: "MemGuard cache is stale, waiting for rebuild."
3. The MCP runtime will rebuild the cache automatically on the next `bootstrap()` or `commit_event()` call.
4. After rebuild, rerun `bootstrap()` to confirm `cache_status: "Healthy"`.

### Pattern 3: Bootstrap ≠ Lookup Mismatch

**Symptom**: `bootstrap()` says a task is `Done`, but `task_lookup()` says it is `InProgress`.

**Root cause**: Cache corruption or partial write failure.

**Fix**:
1. Call `query_memory()` to verify the true state from `memory/*.md`.
2. Treat `memory/*.md` as the source of truth; the cache is a derived view.
3. If the mismatch persists, treat the cache as potentially corrupted and avoid new commits until resolved.

## Prevention

- Always use `memguard_runtime_commit_event()` to write state. Never edit `memory/*.md` directly.
- If manual editing is unavoidable, rerun `bootstrap()` immediately after.

## Related

- `SKILL.md` Section 3: Cache Status Handling
- `SKILL.md` Section 12: Success Criteria (anomaly recommendation)
- `tests/TS-004-runtime-drift.md`: Regression test for this scenario
- `tests/fixtures/duplicate-task-id/TS-004-duplicate-tasks.json`: Reproducible duplicate state
