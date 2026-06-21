# TS-000 — MemGuard v4.4.0 Release Acceptance Test

> **Purpose**: Regression suite to verify core MemGuard functionality before any release or after any structural change.
>
> **When to run**: Before tagging a new version, after modifying `SKILL.md`, `index.json`, or `memory/*.md`, or after any long session where state may have drifted.

---

## Validation Checklist

### Bootstrap & State Loading

- [ ] `memguard_runtime_bootstrap()` completes without error.
- [ ] Bootstrap response includes:
  - `current_phase`
  - `adr_count`
  - `trap_count`
  - `active_tasks` (array or count)
  - `cache_status` (should be `"Healthy"` in normal conditions)
- [ ] Agent announces the bootstrap summary in the standard format:
  ```
  Memory context loaded:
  - Phase: {current_phase}
  - ADRs: {adr_count}
  - Traps: {trap_count}
  - Active Tasks: {active_tasks.length}
  ```

### Task Lifecycle

- [ ] `memguard_runtime_task_lookup({"task_id": "TASK-XXX"})` returns correct `found` / `location` / `status`.
- [ ] `commit_event(TaskCreated)` with a new `id` succeeds.
- [ ] `commit_event(TaskUpdated)` with `new_status` succeeds (not `status`).
- [ ] Terminal transitions (`Done` → `InProgress`, `Done` → `Todo`) are rejected by MCP or blocked by agent.
- [ ] Duplicate task ID creation is prevented or detected.

### ADR Lifecycle

- [ ] `commit_event(AdrCommitted)` with all required fields succeeds.
- [ ] Querying memory returns active, superseded, and rejected ADRs.
- [ ] Superseded ADR has `superseded_by` reference populated.

### Error Recovery

- [ ] `-32602` payload error triggers loading of `diagnostics/payload-errors.md`.
- [ ] Agent retries exactly once after loading the diagnostic.
- [ ] If retry fails, agent stops and informs the user (no infinite loop).
- [ ] Stale cache (`cache_status: "RebuildRequired"`) is detected and reported.
- [ ] Cache corruption (`cache_status: "Corrupted"`) triggers read-only degraded mode.

### Diagnostics & References

- [ ] All files listed in `index.json` exist on disk.
- [ ] No stale references to `references/deployment.md` remain.
- [ ] `diagnostics/payload-errors.md`, `diagnostics/runtime-drift.md`, `diagnostics/recovery.md` are loadable.
- [ ] `docs/deployment.md` is accessible for human operators.

### Long Session Resilience

- [ ] Shadow summary survives at least one context compression cycle.
- [ ] After compression, no duplicate tasks or ADRs are created.
- [ ] If shadow summary is lost, `bootstrap()` is rerun before any state change.

---

## Automated Regression (Future)

When MCP supports programmatic test execution, this suite should be automated:

1. Load each fixture from `tests/fixtures/`.
2. Verify expected MCP response.
3. Verify expected agent action.
4. Assert no state corruption in `memory/*.md`.

---

## Related

- `tests/TS-001-long-session.md`
- `tests/TS-002-payload-recovery.md`
- `tests/TS-003-bootstrap-rerun.md`
- `tests/TS-004-runtime-drift.md`
- `tests/TS-005-blocked-transition.md`
