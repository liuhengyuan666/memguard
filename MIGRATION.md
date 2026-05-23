# MemGuard v2 → v3 迁移指南

> 如果你已经在项目中使用 MemGuard v2 (纯 Skill 模式)，按此指南无缝迁移到 v3 (MCP + Skill 双层架构)。

---

## 迁移概览

| 维度 | v2 (纯 Skill) | v3 (MCP + Skill) |
|------|--------------|-----------------|
| 安装方式 | 复制 SKILL.md | npm 安装 exe + 配置 Skill |
| memory 文件 | 8 类文件 | 3 类核心文件 |
| 写入方式 | Agent 直接写 Markdown | Agent 通过 MCP 工具写入 |
| 并发安全 | 无保护 | Rust RwLock |
| 旧格式保护 | 无 | Parse guard 自动保护 |

---

## 迁移步骤

### Step 1: 安装 v3 运行时

```bash
npm install -g @henry_lhy/memguard-mcp
```

或从源码编译：
```bash
cd memguard-mcp && cargo build --release
```

### Step 2: 配置双层架构

在项目 `opencode.json` 中添加：

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

### Step 3: 首次 Bootstrap

启动 Agent 会话后，Agent 会自动调用 `memguard_runtime_bootstrap`。

**此时发生的事**：
- `memory/context.md`（旧版中文 H2 格式）→ **自动解析**（v3 支持双语解析）
- `memory/decisions.md`（旧版自由文本格式）→ **受保护不覆盖**，但暂不解析为结构化 ADR
- `memory/traps.md`（不存在）→ 自动创建空文件
- `.memguard/` 缓存目录 → 自动创建

### Step 4: 迁移 context.md

如果你使用的是旧版中文格式（如 `## 当前阶段` / `## 关键任务` / `## 当前约束`），v3 会自动识别。首次 `PhaseChanged` 事件提交后，文件会被 flush 为新版英文格式。

### Step 5: 迁移 decisions.md（手动）

旧版 decisions.md 使用自由文本列表格式（`## [YYYY-MM-DD] 标题 - 背景：... - 决策：... - 状态：...`），与新版 ADR 格式不兼容。

**推荐方式**：让 Agent 读取旧文件并通过 MCP 工具逐个重建：

```
请读取 memory/decisions.md，将其中的决策逐条通过
memguard_runtime_commit_event 重建为 ADR 格式。
每条决策的 id 格式为 ADR-XXX，status 根据原文中的"状态"字段设置。
```

重建完成后，旧文件可以保留或删除（v3 会保护它不被覆盖）。

### Step 6: 清理 v2 遗留文件

v3 只管理 3 个核心文件：
- `memory/context.md`
- `memory/decisions.md`
- `memory/traps.md`

以下 v2 遗留文件**不再被 memguard 管理**，你可以：
- **保留**：作为项目文档继续使用（v3 不会修改它们）
- **删除**：如果内容已迁移到新版格式

| 文件 | v2 角色 | v3 状态 |
|------|---------|---------|
| `memory/product.md` | 业务知识 | 不被管理，可保留或删除 |
| `memory/tech.md` | 技术架构 | 不被管理，可保留或删除 |
| `memory/structure.md` | 项目结构 | 不被管理，可保留或删除 |
| `memory/glossary.md` | 术语表 | 不被管理，可保留或删除 |
| `memory/history/` | 里程碑 | 不被管理，可保留或删除 |
| `memory/archive/` | 压缩历史 | 不被管理，可保留或删除 |

### Step 7: 建议添加 .gitignore

```gitignore
# MemGuard v3 runtime cache (derived from memory/*.md)
.memguard/
```

> `memory/*.md` 文件**应该**随 Git 提交（它们是 Source of Truth）。  
> `.memguard/*.json` 文件**不应该**提交（它们是派生缓存，可随时从 memory/ 重建）。

---

## 回退到 v2

如果迁移遇到问题，可以随时回退：

1. 移除 `opencode.json` 中的 `mcp.memguard` 配置
2. 恢复 `.opencode/skills/memguard/SKILL.md` 为 v2 版本
3. v3 不会修改你的旧版 memory 文件（parse guard 保护）

---

## 常见问题

**Q: 迁移后旧版 decisions.md 还能用吗？**  
A: 文件本身受保护不会被覆盖，但不会被加载到 v3 的运行时内存中。建议手动迁移到 ADR 格式。

**Q: 迁移后 memory 文件会变多吗？**  
A: 不会。v3 只管理 3 个核心文件，v2 的 8 个文件你可以选择性保留或删除。

**Q: 可以同时运行 v2 和 v3 吗？**  
A: 不建议。两个版本对 memory 文件的写入策略不同，同时运行可能导致冲突。选择一个版本即可。
