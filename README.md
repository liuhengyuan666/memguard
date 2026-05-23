# Memguard v3 — Agent Memory & Runtime SOP

> **Version**: 3.0.0  
> **Last Updated**: 2026-05-23  
> **MCP Implementation**: [memguard-mcp](https://github.com/liuhengyuan666/memguard-mcp) (Rust binary, v0.1.0)

Memguard 是一套面向 AI Agent 的**双层架构**：
- **Skill 层（本仓库）**：Agent SOP（标准作业程序），定义何时调用工具、遵循什么规则
- **MCP 层（memguard-mcp）**：Rust 运行时，提供 3 个 MCP 工具作为原子能力

两者**不是替代关系**，而是**手与脑的协作**——Skill 告诉 Agent "现在该做什么"，MCP 提供零误差的执行力。

兼容 [OpenCode](https://opencode.ai) / [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) / Claude Code 等支持 MCP + Skill 的平台。

---

## v3.0.0 核心升级：MCP + Skill 双层架构

相比 v2.x 的「纯 Skill」模式，v3.0.0 进化为 **双层架构**：

| 维度 | v2.x (纯 Skill) | v3.0.0 (MCP + Skill) |
|------|----------------|---------------------|
| **形态** | Agent 直接读写 Markdown 文件 | Skill 指导流程，MCP 工具执行操作 |
| **并发安全** | ❌ 无（多 Agent 可能冲突） | ✅ Rust RwLock + Generation Counter |
| **文件保护** | ❌ Agent 可能误写旧格式文件 | ✅ Parse guard 自动保护旧格式 |
| **防抖写入** | ❌ 无 | ✅ 500ms 防抖，避免频繁 I/O |
| **多项目隔离** | ❌ 手动目录切换 | ✅ 自动项目根检测 + MCP 握手修正 |
| **记忆持久化** | Agent 自行负责 | MCP 自动管理（防抖 + Markdown 双向转换） |
| **行为约束** | ✅ SKILL.md 全量 | ✅ SKILL.md 升级为 Agent SOP |

### 架构对比

```
v2.x (纯 Skill):                    v3.0.0 (MCP + Skill):
┌─────────────────────┐            ┌──────────────────────┐
│ SKILL.md (494行)     │            │ SKILL.md (SOP)       │ ← 行为契约
│ Agent 自己读写文件   │            │ "MUST call bootstrap"│
│ 无并发/防抖保护      │            └──────┬───────────────┘
└─────────────────────┘                   │ WHEN / INVOKES
                                          ▼
                           ┌──────────────────────┐
                           │ memguard-mcp (Rust)  │ ← 能力层
                           │ bootstrap / query    │
                           │ commit / flush / lock│
                           └──────────────────────┘
```

---

## 核心能力

| 能力 | 说明 |
|------|------|
| 🧠 **Explore ↔ Execution 双模式** | 需求模糊时发散，需求明确时收敛，3 条显式切换条件 |
| 📝 **结构化 Memory 系统** | context.md + decisions.md + traps.md，Git-native Markdown |
| 🔒 **幻觉防御** | 4 层校验（Structure / Decision / Technology / Assumption） |
| ✅ **ADR 决策不可逆** | Status 生命周期（Proposed → Accepted → Superseded），已拒绝方案不得重复提出 |
| 🔄 **Memory 压缩** | 过时决策摘要化、废弃上下文归档 |
| 🚫 **行为边界** | 禁止 Agent 自主操作 CI/CD、Git hosting、外部服务 |
| 🛡️ **旧格式保护** | Parse guard 自动保护旧版 memory 文件不被覆盖 |

---

## 快速开始

### 方式一：项目级安装（推荐）

```bash
# 1. 安装 MCP 运行时
npm install -g @liuhengyuan/memguard-mcp

# 2. 在你的项目目录安装 Skill
mkdir -p .opencode/skills/memguard
curl -o .opencode/skills/memguard/SKILL.md \
  https://raw.githubusercontent.com/liuhengyuan666/memguard/main/memguard/SKILL.md

# 3. 配置 opencode.json（见 opencode.json.example）
```

### 方式二：远程 URL 自动加载

在你的 `opencode.json` 中添加：

```json
{
  "mcp": {
    "memguard": {
      "type": "local",
      "command": ["npx", "-y", "@liuhengyuan/memguard-mcp"],
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

OpenCode 会自动下载并缓存 Skill，MCP server 随 OpenCode 启动。

### 方式三：全局 Skill + 项目 MCP

```bash
# 全局安装 Skill（所有项目通用）
mkdir -p ~/.config/opencode/skills/memguard
curl -o ~/.config/opencode/skills/memguard/SKILL.md \
  https://raw.githubusercontent.com/liuhengyuan666/memguard/main/memguard/SKILL.md

# 每个项目只需在 opencode.json 注册 MCP：
# { "mcp": { "memguard": { "type": "local", "command": ["npx", "-y", "@liuhengyuan/memguard-mcp"] } } }
```

---

## 使用流程

### 新项目（Greenfield）

1. 安装 MCP 运行时 + 配置 Skill
2. 启动 Agent 会话
3. Agent 自动调用 `memguard_runtime_bootstrap` → 检测无 `memory/` 目录 → 自动初始化
4. 进入 **Explore 模式**，输出：问题框架 + 2-5 个候选方案 + 权衡 + 假设 + 验证策略
5. 方案收敛后 → Agent 调用 `memguard_runtime_commit_event { AdrCommitted }` → 切换至 **Execution 模式**
6. 编码实现，Memory 按 SOP 策略选择性更新

### 既有项目（Brownfield，从 v2 迁移）

1. 安装 MCP 运行时 + 配置 Skill
2. 首次调用 `memguard_runtime_bootstrap` → 自动解析旧格式 `context.md`
3. 旧格式文件被 parse guard 保护不会被覆盖
4. 通过 `memguard_runtime_commit_event` 逐步重建 ADR 和 Traps
5. 详见 [MIGRATION.md](MIGRATION.md)

---

## 仓库结构

```text
memguard/                              # Skill 规范仓（本仓库）
├── README.md                          # 本文件
├── index.json                         # OpenCode skill 发现索引
├── opencode.json.example              # MCP + Skill 双层配置示例
├── MIGRATION.md                       # v2→v3 迁移指南
├── memguard/                          # Skill 目录
│   └── SKILL.md                       # ⭐ Agent SOP — 行为运行时协议
└── templates/                         # Memory 文件模板（v2 遗留，v3 不依赖）

memguard-mcp/                          # MCP 实现仓（独立仓库）
├── README.md                          # MCP 运行时说明
├── src/                               # Rust 源码
├── architecture.md                    # 架构参考
├── blueprint.md                       # 设计蓝图
├── Cargo.toml                         # Rust 依赖
└── build.ps1                          # 构建脚本
```

### Memory 目录（Agent 自动管理）

```text
memory/                  # Source of Truth — 人类可读，随 Git 提交
├── context.md           # 当前阶段、活跃任务、约束条件
├── decisions.md         # ADR 格式架构决策（追加式）
└── traps.md             # 踩坑记录：错误签名 + 上下文 + 解决方案

.memguard/               # Runtime Cache — 机器可读，应被 .gitignore
├── runtime_state.json   # 序列化状态快照
└── search_index.json    # 轻量关键词索引
```

---

## 设计哲学

continuity > statelessness · decisions > conversation history · active context > historical detail · structure > improvisation · controlled autonomy > unrestricted generation

## 许可

MIT
