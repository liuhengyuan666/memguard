# TS-001 — Long Session Compression

> **Objective**: Verify that MemGuard bootstrap shadow summary survives context window compression and does not hallucinate state.

## Test Setup

- Start a long-running session with MemGuard active.
- Perform at least 10 `commit_event` operations (tasks, ADRs, traps).
- Let the session grow until the context window compresses (or simulate compression by truncating earlier context).

## Expected Behavior

1. **Bootstrap Shadow Summary Intact**
   - After compression, `current_phase`, `constraints`, `latest_adr`, `active_task_count` remain accurate.
2. **No State Hallucination**
   - The agent does not re-create tasks that already exist.
   - The agent does not re-propose ADRs already in `decisions.md`.
3. **Recovery Protocol**
   - If the shadow summary is lost, the agent **must** call `memguard_runtime_bootstrap()` again before any further `commit_event`.
   - After rerun, the agent acknowledges the rerun with the standard format:
     ```
     Memory context reloaded:
     - Phase: {current_phase}
     - ADRs: {adr_count}
     - Traps: {trap_count}
     - Active Tasks: {active_tasks.length}
     ```

## Pass Criteria

- [ ] Shadow summary survives at least one compression cycle.
- [ ] No duplicate task IDs created after compression.
- [ ] If shadow summary is lost, `bootstrap()` is rerun before any state change.

## Related

- `SKILL.md` Section 7.2: Bootstrap Shadow Summary
- `diagnostics/runtime-drift.md`: Compression-induced state drift
