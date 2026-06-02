# MemGuard V3 → V4 Migration Guide

> **Scope**: `memguard-mcp` (Rust MCP runtime) + `memguard` (Skill SOP)  
> **Migration Type**: Manual — no automatic rewrite of production memory  
> **Backward Compatibility**: Full read-compat; write-format upgraded

---

## What Changed

### 1. ADR Status — From String to Enum

**V3**: `ADR.status` was a free-form `String`. The only convention was `"active"` vs `"superseded"`.

**V4**: `AdrStatus` is a strict 5-variant enum:

```rust
pub enum AdrStatus {
    Proposed,    // → "Proposed"
    Accepted,    // → "Accepted" (legacy "active" deserializes here)
    Superseded,  // → "Superseded"
    Rejected,    // → "Rejected"
    Archived,    // → "Archived"
}
```

**Impact on existing data**:
- Old `memory/decisions.md` with `**Status:** active` → **automatically parsed as `Accepted`**
- New writes serialize to `"Accepted"`, not `"active"`
- No manual editing required for existing files

### 2. Task Status — Added `Blocked`

**V3**: `TaskStatus` had `Todo | InProgress | Done`

**V4**: `TaskStatus` has `Todo | InProgress | Blocked | Done`

**Impact**: Existing `context.md` with `- [Todo]` / `- [InProgress]` / `- [Done]` continues to parse correctly. New tasks can use `Blocked`.

### 3. Trap Schema — Extended

**V3**: `Trap` had `error_signature`, `context`, `solution`

**V4**: `Trap` adds `root_cause` and `prevention` (both optional via `#[serde(default)]`)

**Impact**: Existing `traps.md` without `### Root Cause` / `### Prevention` sections parse correctly. New traps can include them.

### 4. Memory Layout — Archive Files

**V3**: Only 3 files in `memory/`:
```
memory/
├── context.md
├── decisions.md
└── traps.md
```

**V4**: 2 additional archive files (auto-generated):
```
memory/
├── context.md
├── decisions.md
├── traps.md
├── tasks_archive.md          # NEW: historical Done tasks
└── decisions_archive.md      # NEW: historical Superseded/Rejected/Archived ADRs
```

**Impact**: `tasks_archive.md` and `decisions_archive.md` are **created automatically** by the MCP server when tasks transition to Done or ADRs become stale. No manual setup required.

### 5. Validation Framework

**V3**: Validation was inline, hard-coded in `state_manager.rs`

**V4**: Extracted into `ValidatorRegistry` with 5 reusable validators:
- `EmptyTaskId`
- `DuplicateTaskId`
- `AdrActiveConflict`
- `AdrRejectedRepeat`
- `AdrInvalidTransition`

**Impact**: Agent-facing behavior is identical; errors are now structured and consistent.

### 6. Search — Inverted Index

**V3**: Brute-force `score_match_v3` over all ADRs + Traps on every query

**V4**: `SearchIndex` with inverted term map for O(1) pre-filtering, then scorer for precision

**Impact**: Faster queries for common terms; behavioral parity guaranteed.

### 7. Cleanup CLI

**V3**: No manual migration tool

**V4**: `memguard cleanup` CLI with `--dry-run`, backup, and interactive confirm

---

## Migration Steps

### Step 0: Commit Current State

```bash
git add memory/
git commit -m "chore: snapshot before memguard v4 migration"
```

### Step 1: Upgrade Runtime

```bash
npm install -g @henry_lhy/memguard-mcp@latest
# or build from source
cd memguard-mcp
git pull origin master
cargo build --release
```

### Step 2: Restart MCP

Restart OpenCode / Claude Desktop so the new MCP server loads.

The first `runtime_bootstrap` will:
- Parse existing `memory/*.md` correctly (backward compat)
- Create `tasks_archive.md` and `decisions_archive.md` if needed
- Rebuild `.memguard/search_index.json` in the new inverted index format

### Step 3: Manual Cleanup (Optional but Recommended)

If your project has historical data quality issues:

```bash
# Preview what cleanup would do
memguard cleanup --dry-run --project-root ./your-project

# Run with backup and interactive confirm
memguard cleanup --project-root ./your-project
```

**What cleanup handles**:
- Done tasks still in `context.md` → moved to `tasks_archive.md`
- Stale ADRs (Superseded/Rejected/Archived) in `decisions.md` → moved to `decisions_archive.md`
- Duplicate ADRs (same title + decision) → lower-priority one marked Superseded

**Safety**:
- Backup auto-created at `.memguard/backups/YYYYMMDD-HHMMSS/`
- `--dry-run` shows report without modifying files
- Requires interactive `y` confirmation

### Step 4: Verify

```bash
# Idempotency check — second run should find 0 issues
memguard cleanup --dry-run --project-root ./your-project
```

Expected output:
```
Found:
  No issues found.
```

### Step 5: Upgrade Skill (if self-hosting)

If you pin the Skill URL to a specific version:

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/v4.0.0/"
    ]
  }
}
```

The Skill SOP now includes:
- ADR Lifecycle rules (when to set each status)
- Task Lifecycle rules (Done → archive, Blocked usage)
- Trap Recording Rules (when to record Trap vs ADR)

---

## Backward Compatibility Guarantee

| Feature | V3 Format | V4 Parse | V4 Write |
|---------|-----------|----------|----------|
| `context.md` task status | `Todo`/`InProgress`/`Done` | ✅ | `Blocked` supported |
| `decisions.md` ADR status | `active` | ✅ (→ `Accepted`) | `Accepted` |
| `traps.md` sections | `Context`/`Solution` | ✅ | `Root Cause`/`Prevention` optional |
| `.memguard/runtime_state.json` | V3 flat | ✅ | V4 structure |
| `.memguard/search_index.json` | V3 flat | ✅ (rebuilds on bootstrap) | V4 inverted index |

**No breaking changes** for read paths. All existing `memory/*.md` files parse correctly.

---

## Rollback

If migration causes issues:

```bash
# Restore from cleanup backup
cp .memguard/backups/20260602-080507/context.md memory/
cp .memguard/backups/20260602-080507/decisions.md memory/
# ... restore other files as needed

# Remove auto-generated archive files if they cause issues
rm memory/tasks_archive.md
rm memory/decisions_archive.md

# Restart MCP — it will rebuild from restored state
```

---

## Evolution Rationale

V3 was a **memory logger** — it recorded what the agent did.

V4 is a **memory lifecycle manager** — it actively manages the state of tasks and decisions, prevents data quality degradation, and provides tools for manual governance.

The transition from V3 to V4 was driven by real-world pain points:
- **Archive pollution**: Done tasks accumulated in `context.md`, making bootstrap output noisy
- **ADR duplication**: Multiple active ADRs with the same decision caused query confusion
- **No status machine**: There was no formal way to express "this ADR was rejected and later resubmitted"
- **Search scaling**: Brute-force search became slow as ADR count grew past 50
- **No migration path**: Once data quality degraded, there was no tool to clean it up

V4 addresses all of these while maintaining full backward compatibility.
