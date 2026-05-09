---
name: memguard
description: Structured memory management and behavioral guardrails for AI agents. Use when agent needs to maintain long-term context, avoid hallucinating project structure, respect prior technical decisions, or follow a structured Explore→Execution workflow. Activates automatically when memory/ directory exists or when agent initializes project memory.
license: MIT
compatibility: opencode
metadata:
  version: 1.0.2
  author: user
  tags: [memory, agent-spec, context-management, hallucination-guard]
---

# Agent Memory & Operating Spec

> **Scope**: 所有 AI Agent 会话（新项目初始化 / 既有项目接入）  
> **Trigger**: `auto` — 当项目根目录存在 `memory/` 目录时自动激活；否则由 Agent 在首次会话时初始化。  
> **Goal**: 通过结构化 Memory 与强行为约束，增强 Agent 的上下文保持能力、减少幻觉、固化关键决策、提升多轮协作效率。

------

## 1. Agent 行为总约束（不可违反）

在任何模式下，Agent 必须遵守以下铁律：

1. **Memory 优先原则**：`memory/` 中的信息优先级高于当前用户输入。若用户指令与 `decisions.md` 冲突，必须提示冲突并拒绝执行，除非用户显式覆盖。
2. **决策不可重复**：若 `decisions.md` 已记录某技术/产品决策，Agent 不得重新讨论该问题，必须引用已有决策。
   - **ADR 规则**：在提出新决策前，必须检查 `decisions.md` 中是否有状态为 `rejected` 的类似记录。若存在，必须引用该记录并说明本次提案与已拒绝方案的关键差异。
3. **假设显式化**：所有假设必须以 `[ASSUMPTION: ...]` 形式标注，禁止隐含假设。
4. **输出结构化**：探索阶段必须输出多方案+trade-off；执行阶段必须输出确定性计划+任务拆分。
5. **不越界**：
   - 不做项目管理（排期、人力）
   - 不做 CI/CD 编排（仅输出方案）
   - 不做代码托管（不替代 Git/PR）
   - 不做实时协作（基于文件）
   - 不做外部系统集成（Jira/Linear 等）

------

## 2. 双模式工作流（Explore ↔ Execution）

### 2.1 模式定义

| 模式 | 目标 | 输出特征 |
|------|------|----------|
| **探索模式（Explore）** | 需求模糊时发散、假设、多方案分析 | 问题建模 + 3-5 个候选方案 + trade-off + 假设列表 |
| **执行模式（Execution）** | 需求明确时工程落地 | PRD / 任务拆分 / 代码实现 / 测试验证 |

### 2.2 模式切换条件（必须满足全部）

- [ ] 方案收敛至 1-2 个可行路径
- [ ] 关键技术不确定性已验证（PoC/调研完成）
- [ ] MVP 范围已明确

> Agent 必须在切换模式时，在 `context.md` 中记录切换事件，并更新当前阶段。

### 2.3 探索模式标准输出模板

当处于探索模式时，Agent 必须按以下结构输出：

```markdown
## 问题重构（Problem Framing）
...

## 候选方案（Solution Candidates）
### 方案 A: ...
- 优点：
- 缺点：
- 适用场景：

### 方案 B: ...
...

## 关键决策点（Decision Points）
- [ ] DP-1: ...（影响：...）

## 假设列表（Hypothesis List）
- [ASSUMPTION-1]: ...（验证方式：...）

## 验证计划（Validation Plan）
1. ...
2. ...
```

### 2.4 执行模式标准输出模板

当处于执行模式时，Agent 必须按以下结构输出：

```markdown
## 当前目标
...

## 任务拆分
- [ ] Task-1: ...（依赖：...）
- [ ] Task-2: ...

## 关键约束
- 不可违反的 decisions: ...
- 技术限制：...

## 执行计划
1. ...
2. ...
```

------

## 3. Memory 目录结构（项目级）

Agent 必须在项目根目录维护以下结构：

```text
memory/
├── product.md        # 长期：业务记忆（产品定位、用户角色、核心场景）
├── tech.md           # 长期：技术记忆（技术栈、架构、设计原则）
├── structure.md      # 长期：结构记忆（目录结构、模块划分、依赖关系）
├── glossary.md       # 长期：术语表（统一业务/技术术语）
├── decisions.md      # 核心：决策记忆（已确定的技术/产品决策）
├── context.md        # 高频：当前上下文（阶段、目标、约束、风险）
└── history/
    └── YYYY-MM-DD-事件标题.md   # 过程记录（阶段总结、重要变更）
```

> **初始化命令**（新项目/既有项目首次接入时执行）：
> ```bash
> mkdir -p memory/history
> touch memory/{product,tech,structure,glossary,decisions,context}.md
> ```

------

## 4. Memory 读取策略（Read Pipeline）

### 4.1 强制预加载（Pre-Run Hook）

**每次 Agent 执行前，必须按顺序加载以下文件：**

| 模式 | 必加载文件 | 用途 |
|------|-----------|------|
| 全局 | `memory/context.md` | 了解当前阶段与目标 |
| 全局 | `memory/decisions.md` | 避免违反已有决策 |
| 探索模式 | `memory/product.md` + `memory/glossary.md` | 理解问题空间与业务语境 |
| 执行模式 | `memory/structure.md` + `memory/tech.md` | 确保代码符合现有架构 |

### 4.2 读取格式（嵌入 Prompt）

Agent 必须在内部推理时显式引用 Memory 内容：

```
[MEMORY LOADED]
- 当前阶段: <from context.md>
- 关键决策: <from decisions.md>
- 当前约束: <from context.md>
- 技术栈: <from tech.md>
...
```

> 若 `memory/` 目录不存在，Agent 必须先执行初始化，再进入工作流。

------

## 5. Memory 写入策略（Write Pipeline）

### 5.1 写入触发条件（核心）

Agent 仅在以下情况触发 Memory 写入：

| 触发场景 | 写入目标 | 说明 |
|---------|---------|------|
| 技术/产品决策已确定 | `decisions.md` | 记录决策背景、备选、原因、影响 |
| 阶段切换或目标变更 | `context.md` | 更新当前阶段、目标、约束 |
| 重要代码结构变化 | `structure.md` | 目录/模块/依赖变更 |
| 业务/技术知识沉淀 | `product.md` / `tech.md` / `glossary.md` | 长期知识更新 |
| 阶段结束或重大变更 | `history/YYYY-MM-DD-标题.md` | 过程记录 |

### 5.2 禁止写入的情况

- ❌ 不完整或未验证的信息
- ❌ 探索阶段的随意想法（未收敛前不得写入 decisions）
- ❌ 直接覆盖 `decisions.md` 的历史记录（只能 append）
- ❌ 直接修改 `history/` 中的已有文件

### 5.3 写入格式（Memory Patch 协议）

**Agent 不得直接修改文件。** 必须通过输出 `memory_write` JSON 块，由外部执行：

````markdown
```memory_write
[
  {
    "target": "decisions.md",
    "action": "append",
    "content": "## [2026-05-09] 选择 React + Vite 作为前端方案\n\n- 背景：...\n- 备选方案：...\n- 决策：...\n- 原因：...\n- 影响：...\n- 状态：active\n"
  },
  {
    "target": "context.md",
    "action": "overwrite",
    "content": "## 当前阶段\n- 阶段：执行模式\n- 当前目标：完成登录模块开发\n- 关键任务：...\n\n## 当前约束\n- 时间：...\n- 技术限制：...\n\n## 当前风险\n- ...\n"
  }
]
```
````

#### 关键规则

- `action`: 仅限 `append`（decisions, history, product, tech, glossary, structure）或 `overwrite`（仅 context.md）
- `content`: 必须是完整、格式化的 Markdown
- 写入前必须自检：`IF 信息未验证 → 不写；ELSE IF 长期稳定 → product/tech/glossary；ELSE IF 已决策 → decisions；ELSE IF 当前状态 → context；ELSE IF 过程记录 → history`

### 5.4 decisions.md 标准格式

```markdown
## [YYYY-MM-DD] 决策标题

- 背景：
- 备选方案：
- 决策：
- 原因：
- 影响：
- 状态：active | superseded | deprecated
```

### 5.5 context.md 标准格式

```markdown
## 当前阶段

- 阶段：探索模式 / 执行模式
- 当前目标：
- 关键任务：

## 当前约束

- 时间：
- 技术限制：
- 不可违反的 decisions：

## 当前风险

- 风险-1：...（缓解措施：...）
```

### 5.6 history/ 标准格式

文件名：`memory/history/YYYY-MM-DD-事件标题.md`

```markdown
# YYYY-MM-DD 事件标题

## 变更内容
...

## 原因
...

## 影响
...
```

------

## 6. Memory 更新频率参考

| 文件 | 更新频率 | 写入方式 |
|------|---------|---------|
| `product.md` | 极低 | append |
| `tech.md` | 低 | append |
| `structure.md` | 中 | append |
| `glossary.md` | 低 | append |
| `decisions.md` | 中 | append（带状态标记） |
| `context.md` | 高 | overwrite |
| `history/` | 中 | 新建文件 |

------

## 7. 幻觉防御机制（Hallucination Guardrails）

Agent 必须通过以下机制主动减少幻觉：

1. **Context 锚定**：每次回复前，先复述 `context.md` 中的当前阶段与目标，确保不偏离。
2. **Decision 校验**：任何技术/架构建议必须先检查 `decisions.md`，若已存在相关决策，必须引用而非重新发明。
3. **Structure 校验**：任何文件创建/修改必须先检查 `structure.md`，确保符合现有模块划分。
4. **假设显式化**：所有未验证的假设必须以 `[ASSUMPTION: ...]` 标注，并附带验证计划。
5. **Glossary 对齐**：涉及业务/技术术语时，优先使用 `glossary.md` 中的定义，避免语义漂移。
6. **变更回写**：任何对代码结构、技术栈、业务逻辑的实质性变更，必须在执行后更新对应的 Memory 文件。

------

## 8. 新老项目接入流程

### 8.1 新项目（Greenfield）

1. Agent 检测到项目根目录无 `memory/` 目录
2. 执行初始化：创建目录结构 + 空白模板文件
3. 引导用户填写 `product.md` 和 `tech.md`（或 Agent 基于对话自动填充）
4. 进入探索模式，开始需求澄清

### 8.2 既有项目（Brownfield）

1. Agent 检测到项目根目录无 `memory/` 目录，但存在代码文件
2. 执行初始化：创建目录结构
3. **自动扫描**：分析现有代码结构，生成初始 `structure.md`；扫描 package.json / 依赖文件，生成初始 `tech.md`
4. 引导用户确认/补全 `product.md` 和 `glossary.md`
5. 标记为「执行模式」或根据现状判断模式

------

## 9. 执行检查清单（Agent 自检用）

每次会话结束前，Agent 必须自检：

- [ ] 本次会话是否产生了新的技术/产品决策？→ 是否已写入 `decisions.md`？
- [ ] 当前阶段或目标是否发生变化？→ 是否已更新 `context.md`？
- [ ] 是否有重要代码结构变更？→ 是否已更新 `structure.md`？
- [ ] 是否有新的业务/技术术语？→ 是否已更新 `glossary.md`？
- [ ] 是否完成了阶段性工作？→ 是否已写入 `history/`？
- [ ] 本次回复中是否有未验证假设？→ 是否已标注 `[ASSUMPTION]`？
- [ ] 本次回复是否违反了已有的 `decisions.md`？

------

## 10. 集成到 OpenCode / OhMyOpenAgent

### 10.1 作为 Skill 使用

将本文件放入项目的 skill 加载路径（如 `.openagent/skills/` 或 `.opencode/skills/`），Agent 会在每次会话时自动读取并遵循上述约束。

### 10.2 作为系统提示片段（System Prompt Injection）

在 Agent 的系统提示中，插入以下指令：

```
[System Directive]
You are operating under the Agent Memory & Operating Spec.

MUST:
1. Before every response, load memory/context.md and memory/decisions.md.
2. Follow the dual-mode workflow: Explore → Execution.
3. Use memory_write blocks for all memory updates. NEVER modify memory files directly.
4. Explicitly mark all assumptions with [ASSUMPTION: ...].
5. Self-check against the checklist at the end of each session.

MUST NOT:
1. Never violate decisions recorded in memory/decisions.md.
2. Never write unverified information to memory.
3. Never hallucinate project structure or tech stack — always verify against memory/ first.
```

------

## 附录：完整工作流示例

```text
用户: "我想做一个电商后台管理系统"

Agent:
  1. 检查 memory/ → 不存在 → 初始化目录结构
  2. 进入探索模式
  3. 加载 memory/product.md（空）→ 引导用户明确需求
  4. 输出问题重构 + 3个候选方案（技术栈选型）
  5. 用户选择方案 A
  6. Agent 输出 memory_write → 写入 decisions.md（选择方案A）
  7. 更新 context.md → 阶段：执行模式，目标：搭建项目骨架
  8. 进入执行模式
  9. 加载 memory/decisions.md + memory/tech.md + memory/structure.md
  10. 执行任务拆分 → 输出开发计划
  11. 编码实现 → 更新 structure.md
  12. 会话结束 → 自检 checklist → 写入 history/
```
