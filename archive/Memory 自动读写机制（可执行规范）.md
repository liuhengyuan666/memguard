# Memory 自动读写机制（可执行规范）

> **版本**: v2.0.0（2026-05-21 更新，对齐 SKILL.md v2.0.0）

---

# 1. 核心设计思路（很重要）

把 memory 从"文件"升级为：

> **Agent 的外部持久化状态（State Store）**

然后定义三件事：

1. **什么时候读（Read Trigger）**
2. **什么时候写（Write Trigger）**
3. **写什么（Write Policy）**

v2.0.0 核心升级：从「全量加载」进化为「按需加载 + 主动压缩」。

---

# 2. Memory 读取机制（Read Pipeline）

## 2.1 最少必需加载

每次 Agent 执行前，**最少必须**加载：

```
memory/context.md        ← 当前状态（始终加载）
memory/decisions.md      ← active 部分（始终加载）
```

---

## 2.2 按需条件加载

其余文件仅在相关时加载：

| 文件 | 加载条件 |
|------|----------|
| product.md | 业务/领域推理 |
| tech.md | 实现/架构操作 |
| structure.md | 文件/模块操作 |
| glossary.md | 术语歧义 |
| history/ | 历史推理 |
| archive/ | 长期上下文恢复 |

---

## 2.3 加载优先级

```
active context → active decisions → current structure → compressed summaries → historical detail
```

> 关键原则：active state > historical detail

---

## 2.4 分模式读取优化

### 🧠 Explore 模式

加载：
- context.md
- product.md（业务相关）
- active decisions

👉 目的：理解问题空间，避免重复已拒绝方案

---

### 🔧 Execution 模式

加载：
- context.md
- structure.md
- tech.md
- active decisions

👉 目的：避免结构幻觉 & 遵守已有约束

---

# 3. Memory 写入机制（Write Pipeline）

## 3.1 写入触发条件

| 触发类型 | 写入目标 | 说明 |
|----------|----------|------|
| 确认的决策 | decisions.md | append-only，含 Status lifecycle |
| 阶段/目标切换 | context.md | overwrite |
| 重大架构变更 | tech.md | overwrite |
| 重大结构调整 | structure.md | overwrite |
| 术语稳定化 | glossary.md | append |
| 里程碑完成 | history/ | 新文件 |

---

## 3.2 禁止写入（Forbidden Writes）

- 推测性想法
- 临时思考
- 未解决的探索
- 重复信息
- 临时调试细节
- 冗长执行日志

---

## 3.3 写入策略总表

| 文件 | 策略 | 原因 |
|------|------|------|
| decisions.md | append-only | 历史事实不可覆盖 |
| context.md | overwrite | 当前状态，无需历史版本 |
| structure.md | overwrite | 反映当前真实结构 |
| tech.md | overwrite | 反映当前技术栈 |
| product.md | overwrite | 稳定业务知识 |
| glossary.md | append | 术语持续积累 |
| history/ | 仅新里程碑文件 | 不做逐条执行日志 |
| archive/ | overwrite | 压缩后的历史摘要 |

---

# 4. Memory 压缩机制（v2.0.0 新增）

防止 memory 无限膨胀导致 context collapse：

当 memory 过度增长时：
- 被替代的 decisions → 摘要化
- 废弃的 context → 归档至 archive/
- 历史记录 → 压缩
- 重复术语 → 合并去重
- 保留 active 操作知识

压缩原则：

```
active state > historical detail
```

---

# 5. Agent 内部决策逻辑

```
IF 信息是「稳定业务知识」 → product.md
IF 信息是「稳定技术架构」 → tech.md
IF 信息是「已确认决策」   → decisions.md（append-only）
IF 信息是「当前状态」     → context.md（overwrite）
IF 信息是「里程碑完成」   → history/（新文件）
IF 信息是「废弃/过期」   → archive/（摘要化）
ELSE                     → 不写
```

---

# 6. 行为边界（Operational Boundaries）

Agent 不得自主：
- 执行项目管理
- 管理 CI/CD 流水线
- 管理 Git hosting 工作流
- 执行外部服务集成
- 编造不存在的项目基础设施
- 无充分理由重写架构

可提供计划/建议，但不得自主执行。

---

# 7. Memory 与用户输入的关系

Memory 提供默认操作上下文。
**用户显式指令可覆盖 memory。**

当存在冲突时：
1. 指出冲突
2. 如有歧义请求确认
3. 确认后将覆盖写入 memory

> v2.0.0 修复：v1.x 中 "memory 优先级高于用户输入" 会导致 Agent 拒绝用户修正历史。
