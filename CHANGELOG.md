# Changelog

All notable changes to the MemGuard Skill Spec will be documented in this file.

## [4.4.0] — 2026-06-21

### Validation & Diagnostics Release

This release focuses on **testability** and **diagnosability** — adding validation assets and diagnostic rules to complement the existing behavioral contract.

#### Added

- **`tests/`** — Validation test assets
  - `TS-000-regression-suite.md` — Release acceptance checklist
  - `TS-001-long-session.md` — Long session compression survival
  - `TS-002-payload-recovery.md` — `-32602` error recovery
  - `TS-003-bootstrap-rerun.md` — Bootstrap rerun after context loss
  - `TS-004-runtime-drift.md` — Duplicate task ID & stale cache
  - `TS-005-blocked-transition.md` — Terminal task transition guard
  - `tests/fixtures/` — Reproducible JSON test data for all scenarios above

- **`docs/`** — Human-facing documentation
  - `docs/README.md` — Docs navigation
  - `docs/deployment.md` — Installation & registration guide (migrated from `references/`)

- **`memguard/diagnostics/`** — Runtime diagnostic rules
  - `diagnostics/payload-errors.md` — Common `-32602` errors and field fixes
  - `diagnostics/runtime-drift.md` — Duplicate IDs, stale cache, bootstrap/lookup mismatch
  - `diagnostics/recovery.md` — Degraded read-only mode, cache rebuild, manual repair

#### Improved

- `SKILL.md` Section 5.2 (Error Recovery) — Now routes by error type to specific `diagnostics/*.md` files instead of generic "load the corresponding reference."
- `SKILL.md` Section 10 (Reference Routing) — Added `diagnostics/*.md` and `docs/deployment.md` to the routing table.
- `SKILL.md` Quick Reference Index — Updated to include all `diagnostics/` files.
- `index.json` — Added `diagnostics/*.md` to the skill file list; removed `references/deployment.md`.
- `README.md` — Directory structure diagram updated to include `tests/`, `docs/`, and `diagnostics/`.

#### Removed

- `memguard/references/deployment.md` — Migrated to `docs/deployment.md`.

#### Fixed

- Stale path in `SKILL.md` §13 Integration (`references/deployment.md` → `docs/deployment.md`).
- `TS-005` fixture relocated from `fixtures/duplicate-task-id/` to `fixtures/blocked-transition/` for semantic accuracy.

---

## [4.3.0] — 2026-06-20

### Post-Execution Reliability Release

- Added Bootstrap Shadow Summary (§7.2) to survive context compression.
- Added Payload Cheat Sheet (§5.1) to prevent schema guesswork.
- Improved Exception Handling (§9) with degraded read-only fallback.
- Added Reference Routing Rules (§10) with decision tree for mandatory pre-action loading.
- Cache-aware bootstrap with `cache_status` health signals.

---

## [4.2.0] — 2026-06-19

### Task Lifecycle & Archive Governance

- Introduced 6-status task lifecycle (`Todo` → `InProgress` → `Blocked` → `Done` / `Superseded` / `Cancelled`).
- Terminal auto-archive with `superseded_by` causal chains.
- Archive format specification with global deduplication rules.

---

## [4.1.0] — 2026-06-18

### Task Lifecycle Foundation

- Added `TaskCreated`, `TaskUpdated`, `PhaseChanged` event types.
- Task integrity rules with `task_lookup` pre-check.

---

## [4.0.0] — 2026-06-17

### v4 Dual-Layer Architecture

- Decoupled Skill (SOP) from MCP (Runtime).
- Introduced `memguard_runtime_bootstrap`, `query_memory`, `commit_event`.
- Replaced v2's 8-file memory model with 3-core file model (`context.md`, `decisions.md`, `traps.md`).
- Added Rust-powered MCP runtime with `RwLock` concurrency protection.

---

## [3.0.0] — 2026-06-15

### v3 MCP + Skill Transition

- Introduced MCP layer for atomic file I/O.
- Parse guard for legacy memory format protection.
- Backward compatibility with v2 memory files.

---

[4.4.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v4.4.0
[4.3.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v4.3.0
[4.2.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v4.2.0
[4.1.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v4.1.0
[4.0.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v4.0.0
[3.0.0]: https://github.com/liuhengyuan666/memguard/releases/tag/v3.0.0
