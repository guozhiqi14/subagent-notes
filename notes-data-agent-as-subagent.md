# Notes: 数据分析 SQL Agent 是否适合做成 Subagent？

## 场景描述

一个数据分析 Agent 的完整工作流：

```
用户提出数据问题
  → Agent 理解问题，拆解为数据问题
  → 找到对应的库表、指标、口径
  → Text-to-SQL
  → 执行 SQL，拿到 Raw Data
  → 基于结果做分析，回答用户
```

核心问题：**把"找表 + 写 SQL + 执行"这一整块单独拆成一个 Subagent，是否合适？**

---

## 结论：适合，但边界怎么画至关重要

---

## 一、好处

### 1. 巨大的 Context 保护价值

Schema 探索是 token 黑洞。量化估算：

```
不用 Subagent（全在主线程）:
  浏览数据目录/元数据表：          ~2,000 tokens
  Load 5 张候选表的 Schema：       ~5,000 tokens（每张 ~1,000）
  读取字段说明、指标口径文档：       ~3,000 tokens
  中间推理和 SQL 草稿：            ~1,500 tokens
  SQL 执行结果（Raw Data）：        ~2,000 tokens
  ─────────────────────────────────
  总计：                           ~13,500 tokens

用 Subagent:
  最终 SQL + 结果摘要 + 元信息：     ~1,500 tokens
```

**节省约 90% 的 context。** 一次会话中用户可能连续问 5-10 个数据问题，不用 subagent 的话 context window 会迅速耗尽。

### 2. 符合核心决策规则："中间过程重要吗？"

| 中间步骤 | 主 Agent 需要看到吗？ |
|---------|---------------------|
| 遍历数据目录，找到 10 张候选表 | 不需要 |
| Load 每张表的 Schema | 不需要 |
| 排除不相关的表，选定 3 张 | 不需要（但需要知道选了哪 3 张） |
| 生成 SQL 的草稿和迭代 | 不需要（但需要最终 SQL） |
| 执行 SQL 的过程 | 不需要 |
| 拿到 Raw Data | 需要看到结果 |

**过程不重要，结果重要 → 完全符合 subagent 的适用条件。**

### 3. 有结构性差异，不是"专家声明"

这个 subagent 有真实的结构性价值：
- **独特的工具**：数据库连接、Schema Reader、SQL Executor
- **独特的输出格式**：SQL + 结果 + 口径说明
- **真实的 context 节省**：Schema 探索被完全隔离

对比课程中的反模式："you are a SQL expert" —— 那是专家声明，没有结构性差异。

### 4. 单次委托而非流水线

```
✅ 正确的做法（打包为一个整体）：
主 Agent → [SQL Subagent: 找表 + 写SQL + 执行] → 主 Agent 分析结果

❌ 错误的做法（拆成流水线）：
主 Agent → [Agent A: 找表] → [Agent B: 写SQL] → [Agent C: 执行] → 主 Agent
```

打包成一个 subagent = 单次委托，只有一次交接，信息损失可控。
拆成流水线 = 多次交接，每次都丢信息，链条越长损失越大（传话游戏效应）。

---

## 二、风险点

### 风险 1：信息在交接中丢失

Schema 探索过程中，subagent 可能发现了关键的业务语义，比如：

> "'日活' 这个指标在数仓里有两个口径：table_a 用的是登录口径，table_b 用的是启动口径。我选了 table_a 的登录口径。"

如果 subagent 没有把这个信息带回来，主 Agent 在做分析时就可能给出误导性的结论。

**这是最大的风险 —— 业务语义在 context 隔离中被丢弃。**

### 风险 2：SQL 执行失败时的诊断

类似课程中提到的"测试运行器反模式"：如果 SQL 报错了，subagent 只返回"查询失败"，主 Agent 没有足够的错误信息来诊断问题。

必须确保 SQL 执行失败时，完整的错误信息被返回。

### 风险 3：迭代查询的效率

数据分析天然是迭代的。用户看到第一轮结果后经常说：
- "按地区拆分一下"
- "把时间范围改成最近 7 天"
- "加一个环比"

每次迭代都创建一个全新的 subagent 实例。**新的 subagent 对上一轮的工作完全不知情**——如果第一轮的返回信息不够，每次都要重新探索 Schema，效率很低。

---

## 三、设计上的关键考虑

### 关键设计 1：边界画在哪里

有三种画法，只有 Option B 是合理的：

```
Option A（太薄）— 不值得做 subagent:
  Subagent 只负责 "执行 SQL"
  Input:  一条写好的 SQL
  Output: 查询结果
  → 本质上就是一个 Tool Call，没有 context 节省
    （schema 探索还是在主线程里做的）

Option B（刚好）— 最佳边界 ✅:
  Subagent 负责 "找表 + 理解口径 + 写 SQL + 执行"
  Input:  自然语言的数据问题 + 数据库连接信息
  Output: SQL + 结果 + 用了哪些表 + 口径假设 + 障碍
  → schema 探索的 context 被隔离
    主 Agent 得到的信息足够做分析和回答用户

Option C（太厚）— 走太远了:
  Subagent 负责 "找表 + 写 SQL + 执行 + 分析 + 回答用户"
  Input:  用户原始问题
  Output: 给用户的最终回答
  → 分析环节需要用户对话上下文（之前问过什么、偏好什么维度）
    这些上下文在主 Agent 里，subagent 拿不到
```

### 关键设计 2：输出格式（解决信息丢失问题）

输出格式的设计直接决定了：
1. 当前这一轮主 Agent 能否做好分析
2. 后续轮次主 Agent 能否有效地传递上下文给新的 subagent

```
推荐的输出格式：
1. SQL：最终执行的 SQL 语句
2. 结果：查询结果数据
3. 数据资产：用了哪些表、哪些字段，以及选择的理由
4. 口径说明：涉及的指标口径，如果存在多个口径时选了哪个、为什么
5. 障碍与不确定性：执行过程中遇到的问题，以及任何不确定的假设
```

**输出格式不仅是为了当前轮次，更是为了让主 Agent 在后续轮次中有足够的"记忆素材"可以传递。**

### 关键设计 3：Description 中的 Meta-Instruction

Description 影响的是主 Agent 如何写"任务书"。需要在 description 中加入提示，确保主 Agent 在迭代场景下传递正确的上下文：

```yaml
description: >
  Use this agent to query the data warehouse.
  You must provide the natural language data question.
  If this is a follow-up query, include the previous SQL,
  tables used, and any schema context from prior results.
```

最后一句是给主 Agent 的 meta-instruction —— 提醒它在迭代查询时要传递上一轮的信息。

### 关键设计 4：错误信息必须完整返回

在输出格式中明确要求：SQL 执行失败时，返回完整的错误信息而非摘要，避免"测试运行器反模式"。

---

## 四、关于 Subagent "记忆"的重要认知

### Subagent 没有跨轮次的记忆

每次调用都是一个全新的、空白的 context window。上一轮的 subagent 实例在返回结果后就被销毁了。

```
第一轮：Subagent 实例 #1 完成工作 → 返回结果 → context 被丢弃 💀
第二轮：Subagent 实例 #2（全新的）→ 对 #1 完全不知情
```

### 主 Agent 是唯一的记忆桥梁

```
Subagent #1 的工作成果
      ↓ （以结果摘要的形式）
存入主 Agent 的 context（主 Agent 贯穿整个会话）
      ↓ （主 Agent 推理后，挑选相关信息）
写进给 Subagent #2 的 prompt
      ↓
Subagent #2 以为这是"第一次"接到任务
```

### 迭代查询的实际工作方式

```
第一轮：
  主 Agent → SQL Subagent: "查一下上个月的 DAU 趋势"
  SQL Subagent → 主 Agent: {
    sql: "SELECT date, COUNT(DISTINCT user_id)...",
    results: [...],
    tables_used: "user_activity (登录口径, 按天聚合)",
    schema_summary: "关键字段: user_id, login_date, platform"
  }

第二轮（用户说"按地区拆分"）：
  主 Agent → 新的 SQL Subagent: "在以下基础上修改查询，按地区拆分。
    上一轮的 SQL: SELECT ...
    使用的表: user_activity
    关键字段: user_id, login_date, platform
    注意: 地区字段可能需要 JOIN user_profile 表"
```

第二轮中，主 Agent 能给出这么精确的 prompt，不是因为 description 教它的，而是因为：

1. **主 Agent 的 context 里有第一轮的返回结果**（SQL、表名、字段信息还在）
2. **主 Agent 作为 LLM 有通用推理能力**——它看到用户要"按地区拆分"，推理出新的 subagent 需要上一轮的 SQL 和表信息
3. **Description 中的 meta-instruction 起辅助引导作用**——提醒主 Agent "follow-up 时记得传上一轮的上下文"

```
主 Agent 写 prompt 的能力 = LLM 自身推理（核心驱动力）
                          + Description 的引导（辅助）
                          + Context 中已有的历史信息（素材）
```

### 推论：第一轮的输出质量决定了后续所有轮次的质量

如果第一轮 subagent 只返回 `"DAU 在上个月呈下降趋势"` 而没有返回 SQL 和表信息，那主 Agent 在第二轮就没有任何"记忆素材"可以传递，新的 subagent 不得不从头开始。

**这就是为什么输出格式的设计是整个方案中最关键的一环。**

---

## 总结 Checklist

```
✅ 适合做 Subagent 的理由：
  - Context 保护价值巨大（节省 ~90%）
  - 过程不重要，结果重要
  - 有结构性差异（独特工具 + 输出格式），不是专家声明
  - 单次委托而非流水线

⚠️ 必须解决的设计问题：
  - 边界画在 Option B（找表+写SQL+执行），不要太薄也不要太厚
  - 输出格式必须包含口径说明和表/字段信息（为迭代查询提供记忆素材）
  - Description 需要 meta-instruction 引导主 Agent 在迭代时传递上下文
  - SQL 失败时必须返回完整错误信息
  - Subagent 没有跨轮次记忆，主 Agent 是唯一的记忆桥梁
```
