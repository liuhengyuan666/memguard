# TS-003 — Bootstrap Rerun After Context Loss

> **Objective**: Verify that a MemGuard session correctly handles `memguard_runtime_bootstrap()` rerun after context loss or compression.

## Test Setup

1. Start a session, perform `bootstrap()`, and commit several state changes.
2. Simulate context loss (e.g., start a new session, or manually clear the agent's context).
3. The agent attempts to modify state (e.g., create a task or commit an ADR).

## Expected Behavior

1. **Context Loss Detection**
   - The agent detects that its shadow summary is missing or stale.
2. **Mandatory Rerun**
   - Before any `commit_event`, the agent calls `memguard_runtime_bootstrap()`.
3. **Idempotent Bootstrap**
   - The second `bootstrap()` returns the same state as the first (assuming no external changes to `memory/*.md`).
4. **State Continuity**
   - After rerun, the agent resumes from the correct baseline and does not overwrite or duplicate existing state.

## Pass Criteria

- [ ] `bootstrap()` is called before any state change after context loss.
- [ ] Rerun returns the same state as the original session.
- [ ] No duplicate tasks or ADRs are created after the rerun.
- [ ] Agent announces the rerun summary in the standard format.

## Related

- `SKILL.md` Section 3: Session Start Protocol
- `SKILL.md` Section 7.2: Bootstrap Shadow Summary
- `diagnostics/runtime-drift.md`: Bootstrap vs. lookup inconsistency

## Fixture

See `tests/fixtures/cache-corruption/TS-003-stale-state.json` for a simulated stale bootstrap response.
