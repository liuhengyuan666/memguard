# MemGuard Diagnostic — Recovery Procedures

> **When to load**: Cache is corrupted, MCP is unavailable, or the agent needs to enter a degraded mode.

## Scenario 1: Cache Corruption

**Symptom**: `bootstrap()` returns `cache_status: "Corrupted"`, or `memory/*.md` content is inconsistent with `.memguard/*.json`.

**Root cause**: Disk write failure, power loss during commit, or manual tampering with `.memguard/` files.

**Recovery**:
1. **Stop committing new state.** Do not call `commit_event()` until the cache is healthy.
2. **Enter degraded read-only mode.** Continue using session context and any memory already loaded.
3. **Inform the user** that MemGuard needs repair.
4. **Rebuild from source of truth**:
   - Delete `.memguard/runtime_state.json` and `.memguard/search_index.json`.
   - The MCP will rebuild the cache from `memory/*.md` on the next `bootstrap()` call.
5. **Verify** by calling `bootstrap()` and checking `cache_status: "Healthy"`.

## Scenario 2: MCP Unavailable

**Symptom**: `memguard_runtime_commit_event()` returns a timeout or connection error.

**Recovery**:
1. **Inform the user** that memory synchronization is temporarily unavailable.
2. **Continue the task** using session context and any memory already loaded.
3. **Treat memory state as potentially stale.** Do not assume the absence of an ADR or task means it does not exist.
4. **Retry later** before the next significant decision or at session end.

## Scenario 3: Manual Memory Repair

**Symptom**: `memory/*.md` has structural errors (e.g., missing headers, malformed tables) that prevent MCP parsing.

**Recovery**:
1. Make a backup of `memory/*.md` to `.memguard/backups/`.
2. Fix the structural error manually.
3. Delete `.memguard/*.json` to force a cache rebuild.
4. Rerun `bootstrap()` to verify.

## Checklist Before Declaring Recovery Complete

- [ ] `bootstrap()` returns `cache_status: "Healthy"`.
- [ ] `query_memory()` returns consistent results.
- [ ] No duplicate task IDs or ADR IDs.
- [ ] All active tasks have valid, non-terminal statuses.

## Related

- `SKILL.md` Section 9: Exception Handling
- `tests/TS-003-bootstrap-rerun.md`: Regression test for context loss recovery
- `tests/fixtures/cache-corruption/TS-003-stale-state.json`: Reproducible stale cache state
