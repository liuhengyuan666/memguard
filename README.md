# Memguard — Agent Memory & Runtime Contract

> **Version**: 2.0.0  
> **Last Updated**: 2026-05-21

一套面向 AI Agent 的**结构化记忆与行为运行时协议（Behavioral Runtime Contract）**。

通过系统化的 memory 分层、ADR 化的决策沉淀、双模式工作流以及 Memory 压缩机制，在确保 Agent 上下文连续性的同时，将 token 开销控制在可接受范围 — 杜绝决策循环讨论、项目结构幻觉、上下文漂移三大核心痛点。

兼容 [OpenCode](https://opencode.ai) / [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) / Claude Code 等支持 Agent Skills 的平台。

---

## v2.0.0 核心升级

相比 v1.x 的「规范说明书」定位，v2.0.0 进化为 **可执行的 Agent 行为协议（Agent Operating Spec）**，关键变化：

| 维度 | v1.x | v2.0.0 |
|------|------|--------|
| **风格** | 教程化、解释性文本 | 声明式约束 (MUST / MUST NOT / WHEN) |
| **Memory 优先级** | Memory 高于用户输入 | 用户可显式覆盖，冲突时提示确认 |
| **Memory 加载** | 每次执行强制全部加载 | 按需加载：最少加载 context + active decisions |
| **structure.md** | append-only（会腐化） | overwrite（反映当前状态） |
| **history/** | 记录所有过程 | 仅记录里程碑 |
| **Memory 压缩** | ❌ 无 | ✅ 自动压缩过时 decisions / 去重 glossary |
| **架构边界** | 弱约束 | 明确禁止 CI/CD、Git hosting、外部服务等越界行为 |
| **工作流** | 探索 ↔ 执行 双模式 | 双模式 + 显式切换条件 |
| **决策格式** | 简单模板 | 完整 ADR（Status lifecycle + Rejected check） |

---

## 核心能力

| 能力 | 说明 |
|------|------|
| 🧠 **Explore ↔ Execution 双模式** | 需求模糊时发散，需求明确时收敛，防止过早工程化 |
| 📝 **结构化 Memory 系统** | 7 类 memory 文件 + archive/，分层管理长期知识、决策、上下文 |
| 🔒 **幻觉防御** | 4 层校验（Structure / Decision / Technology / Assumption Visibility） |
| ✅ **ADR 决策不可逆** | 关键决策写入 `decisions.md`（append-only），已拒绝方案不得重复提出 |
| 🔄 **Memory 压缩** | 过时决策摘要化、废弃上下文归档、术语去重，防止 token 膨胀 |
| 📐 **选择性加载** | 按任务相关性加载 memory，最少仅需 context + active decisions |
| 🚫 **行为边界** | 禁止 Agent 自主操作 CI/CD、Git hosting、外部服务等 |

---

## 快速开始

### 方式一：项目级安装（推荐，项目专属）

```bash
# 在你的项目根目录执行
git clone https://github.com/liuhengyuan666/memguard.git /tmp/memguard-spec
mkdir -p .opencode/skills
cp -r /tmp/memguard-spec/memguard .opencode/skills/
rm -rf /tmp/memguard-spec
```

然后 Agent 会自动发现该 skill，或通过以下方式手动加载：

```
skill({ name: "memguard" })
```

### 方式二：全局安装（个人所有项目通用）

```bash
# macOS / Linux
mkdir -p ~/.config/opencode/skills
git clone https://github.com/liuhengyuan666/memguard.git /tmp/memguard-spec
cp -r /tmp/memguard-spec/memguard ~/.config/opencode/skills/
rm -rf /tmp/memguard-spec

# Windows (PowerShell)
New-Item -ItemType Directory -Path "$env:USERPROFILE\.config\opencode\skills" -Force
git clone https://github.com/liuhengyuan666/memguard.git "$env:TEMP\memguard-spec"
Copy-Item -Recurse "$env:TEMP\memguard-spec\memguard" "$env:USERPROFILE\.config\opencode\skills\"
Remove-Item -Recurse "$env:TEMP\memguard-spec"
```

### 方式三：远程 URL 加载（最优雅，自动更新）

在你的 `opencode.json` 配置中添加：

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/liuhengyuan666/memguard/main/"
    ]
  }
}
```

OpenCode 会自动下载并缓存该 skill。

> ⚠️ 远程加载要求 URL 指向的目录包含 `index.json` 文件（本仓库已提供）。

---

## 仓库结构

```text
memguard/
├── README.md                          # 本文件
├── index.json                         # 远程 skill 发现索引
├── opencode.json.example              # 权限配置示例
├── memguard/                          # Skill 目录（名称必须匹配 skill.name）
│   └── SKILL.md                       # Skill 核心 — 行为运行时协议
└── templates/                         # Memory 文件模板
    ├── decisions.md.template
    ├── context.md.template
    └── history.md.template
```

---

## 使用流程

### 新项目（Greenfield）

1. 安装 skill（上述任一方式）
2. 启动 Agent 会话
3. Agent 检测到无 `memory/` 目录 → **自动初始化**
4. 进入 **Explore 模式**，输出：问题框架 + 2-5 个候选方案 + 权衡 + 假设 + 验证策略
5. 方案收敛后 → Agent 写入 `decisions.md` → 切换至 **Execution 模式**
6. 编码实现，Memory 按策略选择性更新

### 既有项目（Brownfield）

1. 安装 skill
2. 启动 Agent 会话
3. Agent 检测到无 `memory/` 但存在代码 → **自动扫描**
4. 生成初始 `structure.md` + `tech.md`
5. 引导用户确认/补全 `product.md`
6. 确定 runtime mode，按 Memory 约束开发

---

## Bootstrap 引导词

以下提示词可直接发送给已加载 MemGuard 的 Agent，快速完成项目初始化或状态刷新。

### 场景一：全新项目（Greenfield）

```
Load memguard runtime.

Analyze the current project: detect architecture,
active decisions, and implementation phase.

Initialize memory structure. Infer runtime mode
automatically. If ambiguous, enter Explore mode.

Summarize:
1. current phase
2. active constraints
3. recommended next actions
```

### 场景二：既有项目，无 Memory（Brownfield）

```
Load memguard runtime.

Scan the current repository and bootstrap memory:
- detect architecture and tech stack → tech.md
- map project structure → structure.md
- request user confirmation for product.md
- infer runtime mode automatically

Summarize:
1. current phase
2. active ADRs (if any inferred)
3. architecture summary
4. recommended next actions
```

### 场景三：既有项目，已有 Memory（Refresh）

```
Load memguard runtime.

Analyze existing memory and refresh runtime state:
- validate active decisions
- detect stale context and structure entries
- identify obsolete history
- infer current runtime mode

Summarize:
1. current phase
2. active ADRs
3. architecture state
4. runtime mode
5. memory cleanup suggestions
```

### 维护命令：刷新结构快照

```
Re-scan the current repository structure and
refresh memory/structure.md.

Requirements:
- remove obsolete structure entries
- reflect current active architecture only
- summarize module responsibilities
- avoid historical append logs
- preserve only active topology
```

---

## Memory 目录结构（由 Agent 自动创建）

```text
memory/
├── context.md        # 当前运行时状态（Mode、Goal、Phase、Tasks、Constraints、Risks）  ⭐ overwrite
├── decisions.md      # ADR 格式决策记录（active | superseded | deprecated | rejected）⭐ append-only
├── product.md        # 稳定业务知识（产品定位、用户角色、核心场景）  overwrite
├── tech.md           # 技术架构记忆（技术栈、架构设计、设计原则）  overwrite
├── structure.md      # 当前项目结构（目录结构、模块划分、依赖关系）  overwrite
├── glossary.md       # 术语表（统一业务/技术术语）  append
├── history/          # 里程碑摘要记录（仅里程碑，非执行日志）  append
└── archive/          # 压缩后的历史 memory（过时决策摘要等）  overwrite
```

### 读写策略总表

| 文件 | 本质 | 策略 | 加载策略 |
|------|------|------|----------|
| context.md | 当前状态 | overwrite | 始终加载 |
| decisions.md | 历史事实 | append-only | 始终加载 active 部分 |
| product.md | 稳定业务知识 | overwrite | 业务推理时加载 |
| tech.md | 稳定技术架构 | overwrite | 实现/架构时加载 |
| structure.md | 当前项目结构 | overwrite | 文件/模块操作时加载 |
| glossary.md | 术语标准化 | append | 术语歧义时加载 |
| history/ | 里程碑摘要 | append（仅新文件） | 历史推理时加载 |
| archive/ | 压缩历史 | overwrite | 长期上下文恢复时加载 |

---

## 关键设计：为什么这样组织？

| 问题 | 解决方案 |
|------|---------|
| Agent 忘记之前的决策 | `decisions.md` — ADR 格式，append-only，Status lifecycle |
| Agent 重复提出已拒绝方案 | 提案前检查 rejected entries，必须说明差异 |
| Agent 幻觉项目结构 | `structure.md` — 任何文件操作前必须校验 |
| Agent 偏离当前目标 | `context.md` — 始终加载，锚定当前 Mode/Goal/Phase |
| 多轮对话后上下文丢失 | Memory 是外部持久化状态，不受 token 限制 |
| Memory 无限膨胀导致 context collapse | Memory 压缩机制 — 过时决策摘要化、废弃上下文归档 |
| Agent 越界操作 | 行为边界约束 — 禁止 CI/CD、Git hosting、外部服务等 |
| 探索阶段过早执行 | 双模式工作流 + 显式切换条件（3 条全部满足才切换） |

---

## 与 OpenCode 的集成原理

OpenCode 的 skill 系统支持：

1. **自动发现**：从 `.opencode/skills/`、`~/.config/opencode/skills/` 等路径扫描
2. **按需加载**：Agent 通过 `skill({ name: "..." })` 调用，内容注入上下文
3. **Compaction Resilient**：skill 内容在上下文压缩后仍然保留
4. **远程加载**：通过 `skills.urls` 配置自动拉取并缓存

本仓库的 `SKILL.md`（v2.0.0）包含：

- **Core Runtime Principles** — Agent 行为基础约束（Memory as Context Anchor、Decision Continuity、Explicit Assumptions、Dual-Mode Operation、Operational Boundaries）
- **Runtime Modes** — Explore ↔ Execution 双模式 + 切换条件
- **Memory Structure** — 8 类 memory 文件 + 读写策略
- **Memory Loading Strategy** — 最少加载 + 条件加载 + 优先级
- **Memory Write Policy** — Write Triggers + Forbidden Writes + Write Rules
- **ADR Decision Format** — Status lifecycle + Context/Alternatives/Decision/Rationale/Consequences
- **Hallucination Guardrails** — Structure / Decision / Technology / Assumption 四层校验
- **Memory Compression** — 过时决策摘要化、废弃上下文归档、术语去重
- **Runtime Self-Check** — Session 结束前强制检查清单
- **Design Philosophy** — continuity > statelessness, decisions > conversation history, active context > historical detail

---

## 与 OhMyOpenAgent 的兼容性

Memguard 与 [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) **高度互补**，两者负责完全不同的层面：

| 能力 | Memguard | OhMyOpenAgent |
|------|---------|---------------|
| **记忆系统** | ✅ 8 类 memory 文件 + archive/ | ❌ 无持久化记忆（仅 .sisyphus/drafts/ 草稿） |
| **决策沉淀** | ✅ ADR 格式 `decisions.md`，禁止重复讨论 | ❌ 无决策注册机制 |
| **工作流模式** | ✅ Explore ↔ Execution 双模式 + 显式切换条件 | ❌ 无显式模式切换 |
| **幻觉防御** | ✅ 4 层校验（Structure / Decision / Technology / Assumption） | ⚠️ 间接（Metis + Momus 审查） |
| **Memory 压缩** | ✅ 过时决策摘要化、废弃上下文归档、术语去重 | ❌ 无 |
| **行为边界** | ✅ 限制 CI/CD、Git hosting、外部服务等自主操作 | ❌ 无显式边界约束 |
| **编码规范** | ⚠️ 弱（仅通用约束） | ✅ 强（Hard Blocks + Anti-Patterns + Hooks） |
| **任务完成** | ⚠️ 弱（自检清单） | ✅ 强（Todo Enforcer + Ralph Loop） |
| **代码质量** | ⚠️ 弱 | ✅ 强（ai-slop-remover + review-work + Comment Checker） |

### 为什么不会冲突

OhMyOpenAgent 的 guardrails 是**运行时行为约束**（编码时不能做什么），而 Memguard 是**项目级状态治理**（记住什么、决定什么、当前在哪）。两者正交：

- **Memguard** 回答："这个项目现在是什么阶段？之前决定了什么？当前目标是什么？"
- **OhMyOpenAgent** 回答："这段代码写得对不对？有没有过度抽象？任务完成了吗？"

---

## 自定义与扩展

你可以 fork 本仓库并修改：

- `SKILL.md`：调整行为约束、更新策略、自检清单
- `templates/`：添加适合你团队的 Memory 文件模板
- `index.json`：如果你有多个 skill，可以在此注册

---

## 许可

MIT
