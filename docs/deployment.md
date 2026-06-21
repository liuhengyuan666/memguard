# MemGuard Deployment Guide

How to install and register the MemGuard skill and MCP server.

## Skill Installation

### Option 1: Remote URL (Recommended)

Add the skill URL to your `opencode.json`:

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/main/"
    ]
  }
}
```

This can be done at the project level (`./opencode.json`) or globally
(`~/.config/opencode/opencode.jsonc`).

### Option 2: Local Copy

Copy `memguard/SKILL.md` and `memguard/references/` into one of:

- `./.opencode/skills/memguard/` (project-level)
- `~/.config/opencode/skills/memguard/` (global)

## MCP Server Registration

The MCP server must be registered separately from the skill. Add this to your
`opencode.json`:

```json
{
  "mcp": {
    "memguard": {
      "type": "local",
      "command": ["npx", "-y", "@henry_lhy/memguard-mcp"],
      "enabled": true
    }
  }
}
```

See `opencode.json.example` in the repository for a complete dual-layer
configuration template.

## Verification

After configuration, restart OpenCode and verify:

1. `memguard_runtime_bootstrap()` appears in available tools.
2. The skill content appears in the Agent context when memory-related topics
   are discussed.
3. `~/.cache/opencode/skills/memguard/` contains `SKILL.md` and the
   `references/` directory.
