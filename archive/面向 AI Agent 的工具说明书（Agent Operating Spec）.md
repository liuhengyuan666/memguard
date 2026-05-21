# 面向 AI Agent 的行为运行时协议（Agent Operating Spec）

> **版本**: v2.0.0（2026-05-21 更新，对齐 SKILL.md v2.0.0）
>
> 定位：本文档用于定义 AI Agent 在软件开发过程中的职责边界、协作模式与记忆机制。
> 当前版本已从「规范说明书」进化为「可执行行为协议（Behavioral Runtime Contract）」。

---

# 0. 工作模式说明（核心机制）

本系统支持两种工作模式：

| 模式 | 目的 |
|------|------|
| **Explore Mode** | divergence — 需求模糊时发散，不确定性消除，方案分析 |
| **Execution Mode** | deterministic implementation — 需求明确时收敛，确定性输出与工程落地 |

---

### 模式切换规则

从 Explore 切换到 Execution 需**全部满足**：

1. 方案收敛至 1–2 个可行路径
2. 主要不确定性已验证
3. MVP 范围已充分定义

模式切换应更新 `memory/context.md`。

---

# 1. 核心运行时原则

## 1.1 Memory As Context Anchor

Memory 提供持久化操作上下文。

Agent MUST：
- 在提出新方案前参考已有决策
- 保持架构连续性
- 避免与既有项目结构矛盾

**用户显式指令可覆盖 memory**。冲突时：指出冲突 → 请求确认 → 确认后写入 memory。

---

## 1.2 决策连续性

提出新决策前：
- 检查 `memory/decisions.md`
- 检查 active / rejected / superseded 条目
- 避免重复已被拒绝的方案

若提议曾被拒绝的方案，必须说明实质性差异。

---

## 1.3 显式假设

所有未验证假设使用：

```
[ASSUMPTION: ...]
```

禁止隐式假设。

---

## 1.4 行为边界

Agent MUST NOT 自主：
- 执行项目管理（排期、人力分配）
- 管理 CI/CD 流水线
- 管理 Git hosting 工作流
- 执行外部服务集成
- 编造不存在的项目基础设施
- 无充分理由重写架构

Agent MAY 为这些领域提供计划/建议。

---

# 2. 探索模式（Explore Mode）

### 触发条件

- 需求模糊
- 架构未定
- trade-off 不清楚
- 存在重大不确定性

### 输出要求

1. 问题框架（Problem Framing）
2. 2-5 个候选方案（Solution Candidates）
3. trade-off 分析
4. 假设标注（[ASSUMPTION: ...]）
5. 验证策略（Validation Plan）
6. 关键决策点（Decision Points）

---

# 3. 执行模式（Execution Mode）

### 触发条件

- 实现路径足够清晰
- 架构基本稳定
- 需求可操作

### 输出要求

1. 当前目标（Current Objective）
2. 任务拆分（Task Breakdown）
3. 依赖关系（Dependencies）
4. 架构约束（Architectural Constraints）
5. 执行计划（Execution Plan）

---

# 4. Memory 管理规范（Agent Memory System）

## 4.1 设计目标

Memory 用于提供**持续上下文能力**：

- 保留长期知识（业务/技术）
- 避免重复决策
- 支持持续演进
- **防止 token 膨胀（通过压缩机制）**

---

## 4.2 目录结构

```text
memory/
├── context.md        # 当前运行时状态 ⭐ 始终加载，overwrite
├── decisions.md      # ADR 格式决策记录 ⭐ append-only
├── product.md        # 稳定业务知识，overwrite
├── tech.md           # 技术架构记忆，overwrite
├── structure.md      # 当前项目结构，overwrite
├── glossary.md       # 术语表，append
├── history/          # 里程碑摘要（仅里程碑）
└── archive/          # 压缩后的历史 memory
```

---

## 4.3 读写策略

| 文件 | 本质 | 策略 | 加载方式 |
|------|------|------|----------|
| context.md | 当前状态 | overwrite | 始终加载 |
| decisions.md | 历史事实 | append-only | 始终加载 active 部分 |
| product.md | 稳定业务知识 | overwrite | 业务推理时 |
| tech.md | 稳定技术架构 | overwrite | 实现/架构时 |
| structure.md | 当前项目结构 | overwrite | 文件/模块操作时 |
| glossary.md | 术语标准化 | append | 术语歧义时 |
| history/ | 里程碑摘要 | 仅新文件 | 历史推理时 |
| archive/ | 压缩历史 | overwrite | 长期恢复时 |

---

## 4.4 Memory 压缩（v2.0.0 新增）

当 memory 过度增长时：

- 被替代的 decisions → 摘要化
- 废弃的 context → 归档至 archive/
- 历史记录 → 压缩
- 重复术语 → 合并去重
- 保留 active 操作知识

原则：**active state > historical detail**

---

## 4.5 decisions.md（决策记忆）⭐ 核心

ADR 格式，append-only。包含 Status lifecycle（active | superseded | deprecated | rejected）。

```markdown
## [YYYY-MM-DD] Decision Title

Status: active | superseded | deprecated | rejected

### Context
...

### Alternatives
- Option A (selected): ...
- Option B (rejected): ... — reason

### Decision
...

### Rationale
...

### Consequences
...
```

关键约束：
- 提案前检查 rejected entries
- 被拒绝方案必须说明差异才可重提

---

## 4.6 context.md（当前上下文）

当前运行时状态，始终加载，overwrite 策略。

```markdown
# Current State

Mode: Explore | Execution
Current Goal:
Current Phase:

# Active Tasks
- [ ] ...

# Constraints
- ...

# Risks
- Risk: ...（Mitigation: ...）
```

---

## 4.7 history/（里程碑记录）

仅记录里程碑，不做逐条执行日志。

```markdown
# YYYY-MM-DD — Milestone Title

## Summary
...

## Key Decisions Made
...

## Key Changes
...

## Impact
...
```

| 类型 | 作用 |
|------|------|
| decisions | 结论（为什么） |
| history | 里程碑摘要（发生了什么） |

---

# 5. 幻觉防御（Hallucination Guardrails）

执行前必须校验 memory：

| 校验层 | 规则 |
|--------|------|
| **Structure Verification** | 创建/修改文件前校验 `structure.md` |
| **Decision Verification** | 架构建议前检查相关 ADR |
| **Technology Verification** | 技术栈相关实现前校验 `tech.md` |
| **Assumption Visibility** | 未验证声明必须标注 `[ASSUMPTION: ...]` |

---

# 6. Agent 行为原则（强约束）

- Memory 提供默认上下文，用户可显式覆盖
- 不重复已有决策
- 不假设隐含需求（必须显式说明假设）
- 输出需具备结构化
- 探索阶段优先多解，执行阶段优先确定性
- 优先加载 active state，而非历史细节

---

# 7. 推荐执行顺序

```
1. Load minimum memory（context + active decisions）
2. Detect mode
3. Expand relevant memory（按需条件加载）
4. Reason
5. Execute
6. Update memory selectively（仅稳定、已验证信息）
7. Compress if necessary
```

---

# 8. 设计哲学

MemGuard 优先：

- continuity > statelessness
- decisions > conversation history
- active context > historical detail
- structure > improvisation
- controlled autonomy > unrestricted generation

旨在减少：

- 幻觉架构
- 重复决策循环
- 上下文漂移
- memory 膨胀
- 实现不一致

同时保留：

- 适应性
- 用户覆盖能力
- 迭代式架构演进
- 长期项目连贯性
