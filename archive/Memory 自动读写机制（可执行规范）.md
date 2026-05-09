# Memory 自动读写机制（可执行规范）

# 1. 核心设计思路（很重要）

把 memory 从“文件”升级为：

> **Agent 的外部持久化状态（State Store）**

然后定义三件事：

1. **什么时候读（Read Trigger）**
2. **什么时候写（Write Trigger）**
3. **写什么（Write Policy）**

------

# 2. Memory 读取机制（Read Pipeline）

## 2.1 默认读取策略（强制）

每次 Agent 执行前，必须加载：

```
memory/context.md        ← 当前状态（必须）
memory/decisions.md      ← 决策约束（必须）
memory/structure.md      ← 代码结构（执行阶段）
memory/product.md        ← 业务相关（按需）
memory/tech.md           ← 技术相关（按需）
```

------

## 2.2 分模式读取优化（关键）

### 🧠 探索模式

重点加载：

- product.md
- context.md
- （可选）decisions.md

👉 目的：理解问题空间

------

### 🔧 执行模式

重点加载：

- structure.md
- tech.md
- decisions.md
- context.md

👉 目的：避免做错 & 遵守已有约束

------

## 2.3 实现建议（opencode 可落地）

你可以在 Agent prompt 里强制插入：

```
[MEMORY CONTEXT]

You must read and follow:

- memory/context.md (current state)
- memory/decisions.md (do not violate)
- memory/structure.md (if coding)
```

或者更工程一点：

👉 做一个 pre-hook：

```
cat memory/context.md memory/decisions.md
```

------

# 3. Memory 写入机制（Write Pipeline）

## 3.1 写入触发条件（核心）

只有在以下情况才允许写 memory：

| 触发类型          | 写入目标     |
| ----------------- | ------------ |
| 做出技术/产品决策 | decisions.md |
| 阶段切换          | context.md   |
| 重要结构变化      | structure.md |
| 阶段性总结        | history/     |

------

## 3.2 写入流程（强约束）

```
1. 判断是否需要写 memory
2. 生成结构化内容
3. 标记写入类型
4. 输出写入 patch（而不是直接改）
```

------

## 3.3 写入格式（统一协议）

让 Agent 输出：

```
{
  "memory_write": [
    {
      "target": "decisions.md",
      "action": "append",
      "content": "## [2026-04-24] 选择 FastAPI ..."
    }
  ]
}
```

👉 关键点：

- ❌ 不直接改文件
- ✅ 输出“意图”，由外部执行

------

## 3.4 为什么必须这样？

否则你会遇到：

- AI 覆盖文件
- 写乱结构
- 不可控 diff

------

# 4. Memory 写入策略（Write Policy）

## 4.1 decisions.md（最重要）

只有在以下情况写入：

- 技术选型确定
- 架构变更
- 关键 trade-off 已决定

------

## 4.2 context.md（高频更新）

每轮可以更新，但必须：

- 不覆盖历史结构
- 只更新“当前状态”

------

## 4.3 history/（日志型）

用于：

- 阶段结束总结
- 重要变更记录

------

## 4.4 禁止写入

- 不完整信息
- 未验证假设
- 探索阶段的随意想法

------

# 5. Agent 内部决策逻辑（核心智能）

你可以强制 Agent 遵循这个判断：

```
IF 信息是“长期稳定” → product / tech
IF 信息是“已决策” → decisions
IF 信息是“当前状态” → context
IF 信息是“过程记录” → history
ELSE → 不写
```

------

# 6. Memory Hook 机制（工程关键）

## 6.1 Pre-Run Hook（执行前）

```
# 加载 memory
cat memory/context.md
cat memory/decisions.md
```

------

## 6.2 Post-Run Hook（执行后）

1. 检查 Agent 输出是否包含：

```
"memory_write": [...]
```

1. 自动写入文件

------

## 6.3 可选：Diff 模式（更高级）

```
git diff memory/
```

👉 用于审计 AI 行为