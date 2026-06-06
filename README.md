# MemGuard v4 — Agent Memory & Runtime SOP

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/spec-4.2.0-green.svg)](https://github.com/liuhengyuan666/memguard)
[![MCP Support](https://img.shields.io/badge/MCP-Supported-orange.svg)](https://modelcontextprotocol.io)

> **The Git-Native Memory Engine & Operating Spec for AI Agents.**
> Stop your agents from hallucinating architecture, repeating resolved decisions, or corrupting project state.

MemGuard v4 adopts a **decoupled dual-layer architecture**:

- **The Brain (Skill Layer):** Standard Operating Procedures (SOP) defining *when* to invoke tools and *what* constraints to follow.
- **The Hand (MCP Layer):** A high-performance Rust runtime ([memguard-mcp](https://github.com/liuhengyuan666/memguard-mcp)) providing zero-error tool execution, atomic concurrency control, and lightning-fast search.

Compatible with [OpenCode](https://opencode.ai) / [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) / Claude Code and any platform supporting MCP + Skills.

---

## ⚡ Why MemGuard? (The Pain Points We Kill)

| The Problem in Vanilla Agents | The MemGuard Way |
| :--- | :--- |
| **Context Drift & Hallucination** | **4-Layer Verification**: Forces verification against `structure`, `decisions`, `tech`, and `assumptions` before coding. |
| **Decision Blindness** | **Immutable ADRs**: Prevents agents from entering loop discussions or re-proposing previously `rejected` paths. |
| **State Overwrites (Concurrency)** | **Rust `RwLock` & Concurrency Guard**: Safely manages state when multiple agents/sub-agents write simultaneously. |
| **Old Format Corruption** | **Parse Guard**: Detects legacy `memory/*.md` files and prevents empty-state overwrites, preserving content until explicit migration. |
| **Task Chaos** | **Task Lifecycle (v4.1)**: 6 statuses (Todo→InProgress→Blocked→Done/Superseded/Cancelled) with terminal auto-archive and `superseded_by` causal chains. |
| **Memory Rot** | **Archive Governance**: Stale ADRs and completed tasks auto-migrate to dated archive sections with global deduplication. |

---

## 🏗️ Dual-Layer Architecture Overview

```text
       ┌────────────────────────────────────────────────────────┐
       │                 AI Agent Session Loop                  │
       └───────────────────────────┬────────────────────────────┘
                                   │ Mentally Bound By
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │         SKILL.md (Agent SOP / Behavioral Contract)      │
       │   - Tracks Explore ↔ Execution Mode Transitions         │
       │   - Enforces Guardrails & Out-of-Bounds Restrictions   │
       └───────────────────────────┬────────────────────────────┘
                                   │ Automatically Invokes
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │          memguard-mcp (Rust-Powered Runtime)           │
       │   - Atomic File I/O with 500ms Debounce                │
       │   - In-Memory State Cache (.memguard/*.json)           │
       │   - Thread-Safe RwLock Concurrency Protection          │
       └────────────────────────────────────────────────────────┘
```

---

## 🚀 Quick Start

### Method 1: Remote URL Auto-Load (Recommended)

Add this to your project's `opencode.json`:

```json
{
  "mcp": {
    "memguard": {
      "type": "local",
      "command": ["npx", "-y", "@henry_lhy/memguard-mcp"],
      "enabled": true
    }
  },
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/main/"
    ]
  }
}
```

OpenCode will automatically download and cache the Skill. The MCP server starts with OpenCode.

### Method 2: Project-Level Install

```bash
# 1. Install the MCP runtime
npm install -g @henry_lhy/memguard-mcp

# 2. Install the Skill in your project
mkdir -p .opencode/skills/memguard
curl -o .opencode/skills/memguard/SKILL.md \
  https://raw.githubusercontent.com/liuhengyuan666/memguard/main/memguard/SKILL.md

# 3. See opencode.json.example for full configuration options
```

### Method 3: Global Skill + Per-Project MCP

```bash
# Install Skill globally (all projects)
mkdir -p ~/.config/opencode/skills/memguard
curl -o ~/.config/opencode/skills/memguard/SKILL.md \
  https://raw.githubusercontent.com/liuhengyuan666/memguard/main/memguard/SKILL.md

# Each project only needs the MCP entry in opencode.json:
# { "mcp": { "memguard": { "type": "local", "command": ["npx", "-y", "@henry_lhy/memguard-mcp"] } } }
```

---

## ❓ Troubleshooting

### `Skill "memguard" not found` in another project

This happens when the MCP server is installed but the Skill is not. The
`memguard-mcp` package provides only the runtime tools — it does **not**
include the Agent SOP.

**Fix**: Ensure the Skill is installed in the target project's `opencode.json`:

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/main/"
    ]
  }
}
```

You can also install the Skill globally (`~/.config/opencode/skills/memguard/`)
or per-project (`.opencode/skills/memguard/`). See [Quick Start](#-quick-start)
for all installation methods.

### Agent produces `MCP error -32602` when calling `runtime_commit_event`

This means the Agent is calling MCP tools without following the SOP. This
happens when:

1. The Skill is not installed → install it as above
2. The Skill is installed but the Agent skipped `runtime_bootstrap` → restart the session

With the Skill installed, the Agent automatically follows correct payload schemas:
- `TaskUpdated`: requires `task_id` + `new_status` (not `status`)
- `AdrCommitted`: requires all 6 fields including `context` and `decision`

### Quick Verification

After configuration, verify both layers are active:

1. OpenCode should list `memguard` in its available tools
2. Agent should call `memguard_runtime_bootstrap` at session start
3. If neither happens, check `opencode.json` syntax and restart OpenCode

---

## 📂 Memory Layout (Source of Truth)

MemGuard maintains a human-readable, Git-tracked `memory/` directory alongside a high-performance machine-readable cache:

```text
[Your Project Root]
├── memory/                        # 💡 Source of Truth (Human Readable, Git Committed)
│   ├── context.md                 # Active phase, goals, current tasks, and constraints
│   ├── decisions.md               # Active ADRs (Accepted / Proposed)
│   ├── traps.md                   # Error signatures, context, and solutions
│   ├── tasks_archive.md           # Historical completed tasks (auto-generated)
│   └── decisions_archive.md       # Historical stale ADRs (auto-generated)
│
└── .memguard/                     # ⚡ Runtime Cache (Machine Readable, Add to .gitignore)
    ├── runtime_state.json         # Serialized state graph for concurrent validation
    ├── search_index.json          # Inverted keyword index for instant retrieval
    └── backups/                   # Manual cleanup snapshots (YYYYMMDD-HHMMSS/)
        ├── context.md
        ├── decisions.md
        ├── runtime_state.json
        ├── search_index.json
        └── manifest.json
```

**The `traps.md` Evolution:** The dedicated trap file maps error traces directly to past architectural contexts, completely eliminating the loop where an agent repeatedly rediscovers the same bug across multiple sessions.

---

## 🛠️ Typical Agent Workflow

1. **Bootstrap**: Session starts → Agent runs `runtime_bootstrap` → Syncs Git Markdown state into Rust memory.
2. **Explore Mode**: Requirements are ambiguous → Agent produces 2-5 technical options with trade-offs.
3. **Commit & Freeze**: Decision made → Agent fires `runtime_commit_event { AdrCommitted }` → State transitions to Execution mode, freezing the architecture choice in `decisions.md`.
4. **Guardrailed Execution**: Coding phase → Rust background engine validates operations, prevents old-format file corruption.

---

## 📂 Repository Structure

```text
memguard/                              # Skill Spec Repository (this repo)
├── README.md                          # This file
├── index.json                         # OpenCode skill discovery index
├── opencode.json.example              # MCP + Skill dual-layer config example
├── MIGRATION.md                       # v3 → v4 migration guide
├── memguard/                          # Skill directory
│   ├── SKILL.md                       # ⭐ Agent SOP Router — core behavioral contract (~1000 tokens)
│   └── references/                    # 📚 Detailed rules loaded on demand
│       ├── adr-lifecycle.md           # ADR state machine & transitions
│       ├── task-lifecycle.md          # Task 6-status lifecycle & archive rules
│       ├── trap-rules.md              # Trap recording guidelines
│       ├── archive-format.md          # Archive structure & cleanup
│       └── task-lookup.md             # Task lookup usage guide
└── templates/                         # Memory file templates (v2 legacy, v4 independent)

memguard-mcp/                          # MCP Implementation Repository (separate)
├── README.md                          # MCP runtime documentation
├── src/                               # Rust source code
├── architecture.md                    # Architecture reference
├── blueprint.md                       # Design blueprint
├── Cargo.toml                         # Rust dependencies
└── build.ps1                          # Build script
```

### Migration from v2

See [MIGRATION.md](MIGRATION.md) for the step-by-step v3 → v4 migration guide, including backward compatibility notes and manual cleanup instructions.

---

## 🎯 Design Philosophy

> continuity > statelessness · decisions > conversation history · active context > historical detail · structure > improvisation · controlled autonomy > unrestricted generation

---

## ⚖️ License

The source code is licensed under MIT.

However, the project name, logo, and branding are not permitted to be used for commercial distribution without explicit permission.
