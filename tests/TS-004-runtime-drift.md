# TS-004 — Runtime Drift Detection

> **Objective**: Verify that MemGuard detects and handles runtime state drift (e.g., duplicate task IDs, stale cache, or bootstrap/lookup mismatch).

## Test Setup

1. Create a task via `commit_event(TaskCreated)` with `id: TASK-011`.
2. Manually or via external process, create a **second** task with the same ID in `memory/context.md` (simulating a merge conflict or manual edit).
3. Call `memguard_runtime_bootstrap()`.

## Expected Behavior

1. **Bootstrap Detects Anomaly**
   - `bootstrap()` returns `cache_status: "RebuildRequired"` or reports the duplicate.
2. **Agent Response**
   - The agent does **not** proceed with new state commits until the anomaly is resolved.
   - The agent calls `query_memory()` to verify the inconsistency.
3. **Resolution**
   - If the anomaly is unambiguous (e.g., clear duplicate), the agent informs the user and suggests manual cleanup or auto-deduplication.
   - If the anomaly might be intentional, the agent asks the user before acting.

## Pass Criteria

- [ ] `bootstrap()` detects the duplicate or stale state.
- [ ] Agent halts new commits until anomaly is resolved.
- [ ] Agent uses `query_memory()` to verify before acting.
- [ ] No silent overwrites or silent deletions occur.

## Related

- `SKILL.md` Section 3: Cache Status Handling (`RebuildRequired`, `Corrupted`)
- `SKILL.md` Section 12: Success Criteria (anomaly recommendation)
- `diagnostics/runtime-drift.md`: Duplicate task ID, stale index, bootstrap mismatch

## Fixture

See `tests/fixtures/duplicate-task-id/TS-004-duplicate-tasks.json` for a simulated duplicate task state.
