# Memguard — Agent Memory & Guardrails

> **Version**: 1.0.2  
> **Last Updated**: 2026-05-09

一套面向 AI Agent 的**结构化记忆与行为约束规范**，用于增强 Agent 的上下文保持能力、减少幻觉、固化关键决策，提升多轮协作效率。

兼容 [OpenCode](https://opencode.ai) / [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) / Claude Code 等支持 Agent Skills 的平台。

---

## 核心能力

| 能力 | 说明 |
|------|------|
| 🧠 **探索 ↔ 执行 双模式** | 需求模糊时发散，需求明确时落地，防止过早工程化 |
| 📝 **结构化 Memory 系统** | 7 类记忆文件，分层管理长期知识、决策、上下文 |
| 🔒 **幻觉防御机制** | 6 层校验（Context 锚定、Decision 校验、Structure 校验等） |
| ✅ **决策不可重复** | 关键决策写入 `decisions.md`，Agent 不再反复讨论已确定事项 |
| 🔄 **Memory Patch 协议** | Agent 输出写入意图而非直接改文件，防止覆盖和不可控 diff |

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
│   └── SKILL.md                       # Skill 核心内容（必须大写）
└── templates/                         # 可选：Memory 文件模板
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
4. 进入**探索模式**，开始需求澄清与方案分析
5. 方案收敛后 → Agent 写入 `decisions.md` → 切换至**执行模式**
6. 编码实现，Memory 持续更新

### 既有项目（Brownfield）

1. 安装 skill
2. 启动 Agent 会话
3. Agent 检测到无 `memory/` 但存在代码 → **自动扫描**
4. 生成初始 `structure.md` + `tech.md`
5. 引导用户确认/补全 `product.md`
6. 进入执行模式，按 Memory 约束开发

---

## Memory 目录结构（由 Agent 自动创建）

```text
memory/
├── product.md        # 业务记忆（产品定位、用户角色、核心场景）
├── tech.md           # 技术记忆（技术栈、架构、设计原则）
├── structure.md      # 结构记忆（目录结构、模块划分、依赖关系）
├── glossary.md       # 术语表（统一业务/技术术语）
├── decisions.md      # 决策记忆（已确定的技术/产品决策）⭐ 核心
├── context.md        # 当前上下文（阶段、目标、约束、风险）⭐ 高频更新
└── history/
    └── YYYY-MM-DD-事件标题.md   # 过程记录
```

---

## 关键设计：为什么这样组织？

| 问题 | 解决方案 |
|------|---------|
| Agent 忘记之前的决策 | `decisions.md` — 决策一旦写入，Agent 必须遵守 |
| Agent 幻觉项目结构 | `structure.md` — 任何文件操作前必须校验 |
| Agent 偏离当前目标 | `context.md` — 每次回复前锚定当前阶段 |
| 多轮对话后上下文丢失 | Memory 是外部持久化状态，不受 token 限制 |
| Agent 直接覆盖文件 | Memory Patch 协议 — 输出意图，外部执行 |
| 探索阶段过早执行 | 双模式工作流 — 明确切换条件 |

---

## 与 OpenCode 的集成原理

OpenCode 的 skill 系统支持：

1. **自动发现**：从 `.opencode/skills/`、`~/.config/opencode/skills/` 等路径扫描
2. **按需加载**：Agent 通过 `skill({ name: "..." })` 调用，内容注入上下文
3. **Compaction Resilient**：skill 内容在上下文压缩后仍然保留
4. **远程加载**：通过 `skills.urls` 配置自动拉取并缓存

本仓库的 `SKILL.md` 包含完整的：
- YAML Frontmatter（name / description / metadata）
- Agent 行为约束（不可违反的铁律）
- Memory 读写管道（触发条件 + Patch 协议）
- 双模式工作流（Explore ↔ Execution）
- 自检清单（每次会话结束强制检查）

---

## 与 OhMyOpenAgent 的兼容性

Memguard 与 [OhMyOpenAgent](https://github.com/code-yeongyu/oh-my-openagent) **高度互补**，两者负责完全不同的层面：

| 能力 | Memguard | OhMyOpenAgent |
|------|---------|---------------|
| **记忆系统** | ✅ 7 类 memory 文件（product/tech/structure/decisions/context/glossary/history） | ❌ 无持久化记忆（仅 .sisyphus/drafts/ 草稿） |
| **决策沉淀** | ✅ `decisions.md` 禁止重复讨论 | ❌ 无决策注册机制 |
| **工作流模式** | ✅ 探索 ↔ 执行 双模式切换 | ❌ 无显式模式切换 |
| **幻觉防御** | ✅ 6 层校验（Context 锚定、Decision 校验、Structure 校验等） | ⚠️ 间接（Metis + Momus 审查） |
| **编码规范** | ⚠️ 弱（仅通用约束） | ✅ 强（Hard Blocks + Anti-Patterns + Hooks） |
| **任务完成** | ⚠️ 弱（自检清单） | ✅ 强（Todo Enforcer + Ralph Loop） |
| **代码质量** | ⚠️ 弱（Memory Patch 协议） | ✅ 强（ai-slop-remover + review-work + Comment Checker） |

### 为什么不会冲突

OhMyOpenAgent 的 guardrails 是**运行时行为约束**（编码时不能做什么），而 Memguard 是**项目级状态治理**（记住什么、决定什么、当前在哪）。两者正交：

- **Memguard** 回答："这个项目现在是什么阶段？之前决定了什么？当前目标是什么？"
- **OhMyOpenAgent** 回答："这段代码写得对不对？有没有过度抽象？任务完成了吗？"

### 关于 andrej-karpathy-skills

如果你同时使用 OhMyOpenAgent，**不建议额外引入 [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)**。原因：

| karpathy 原则 | OhMyOpenAgent 已有对应 | 冗余度 |
|--------------|----------------------|--------|
| Simplicity First（不过度抽象） | Metis 检测 premature abstraction + ai-slop-remover skill | 🔴 高 |
| Surgical Changes（不改无关代码） | Comment Checker + Anti-Patterns | 🟡 中 |
| Goal-Driven Execution（测试先行） | Todo Enforcer + Ralph Loop + review-work skill | 🔴 高 |
| Think Before Coding（多问少猜） | Momus 计划审查 + Metis gap 分析 | 🟡 中 |

> 重复加载会增加上下文消耗，且可能导致 Agent 行为过度保守。

---

## 自定义与扩展

你可以 fork 本仓库并修改：

- `SKILL.md`：调整行为约束、更新策略、自检清单
- `templates/`：添加适合你团队的 Memory 文件模板
- `index.json`：如果你有多个 skill，可以在此注册

---

## 许可

MIT
