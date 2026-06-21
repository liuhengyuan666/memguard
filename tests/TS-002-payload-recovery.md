# TS-002 — Payload Recovery After -32602

> **Objective**: Verify that an agent correctly recovers from a `memguard_runtime_commit_event` validation error without corrupting state.

## Test Setup

1. Trigger an event with an **incorrect payload** (e.g., `TaskUpdated` with `status` instead of `new_status`).
2. Observe the MCP response: `-32602` (Invalid params).

## Expected Behavior

1. **Agent Does NOT Retry Blindly**
   - The agent must **not** guess the correct schema and retry immediately.
2. **Reference Loading**
   - The agent loads `diagnostics/payload-errors.md` (or the relevant reference file) to identify the correct field names.
3. **Exactly One Retry**
   - After loading the reference, the agent retries **once** with the corrected payload.
4. **No State Corruption**
   - If the second attempt also fails, the agent informs the user and stops retrying. No partial or malformed state is written.

## Pass Criteria

- [ ] First failure triggers reference loading, not blind retry.
- [ ] Second attempt uses the exact schema from the reference.
- [ ] If second attempt fails, agent stops and reports to user.
- [ ] No duplicate or malformed entries in `memory/*.md` after the test.

## Related

- `SKILL.md` Section 5.2: Error Recovery Rule
- `diagnostics/payload-errors.md`: Common payload error signatures and fixes
- `references/task-lifecycle.md`: Correct payload schema

## Fixture

See `tests/fixtures/payload-error/TS-002-invalid-payload.json` for a reproducible invalid payload.
