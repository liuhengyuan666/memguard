# TS-005 — Blocked Transition Handling

> **Objective**: Verify that MemGuard correctly prevents and handles illegal task status transitions.

## Test Setup

1. Create a task `TASK-020` with status `Todo`.
2. Transition it to `InProgress`.
3. Transition it to `Done`.
4. Attempt to transition `TASK-020` from `Done` back to `InProgress` or `Todo`.

## Expected Behavior

1. **Terminal State Guard**
   - `commit_event(TaskUpdated)` with `new_status: "InProgress"` on a `Done` task returns an error or is rejected by the MCP.
2. **Agent Response**
   - The agent loads `references/task-lifecycle.md` to confirm the terminal state rule.
   - The agent informs the user that the task is terminal and cannot be reopened.
3. **Correct Reopening Path**
   - If the user genuinely needs to reopen the work, the agent creates a **new** task (e.g., `TASK-021`) with `Todo` status, referencing `TASK-020` in the description.
   - The agent does **not** modify the terminal task.

## Pass Criteria

- [ ] Terminal transition is rejected by MCP or blocked by agent.
- [ ] Agent loads `task-lifecycle.md` to confirm the rule.
- [ ] Agent creates a new task instead of modifying the terminal one.
- [ ] Original `TASK-020` remains in `Done` status in `memory/context.md`.

## Related

- `SKILL.md` Section 6: Task Integrity (terminal tasks cannot be modified)
- `references/task-lifecycle.md`: 6-status lifecycle and terminal rules

## Fixture

See `tests/fixtures/blocked-transition/TS-005-terminal-transition.json` for a simulated illegal transition payload.
