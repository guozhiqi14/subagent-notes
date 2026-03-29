# Notes: 主 Agent 与子 Agent 的通信原理

## 核心模型：信封通信（两封信，两个独立世界）

主 Agent 和子 Agent 之间的通信本质上就是**两封信**，运行在**完全隔离的上下文窗口**中。

```
┌───────────────────────────────────────────────────┐
│  主 Agent 的上下文窗口                              │
│                                                     │
│  [用户对话历史] [System Prompt] [工具结果...]        │
│                                                     │
│  ┌───────────────────────────────┐                  │
│  │  Agent Tool Call              │  ← 信封 1（出）   │
│  │  prompt: "去做 XXX..."        │                  │
│  └──────────────┬────────────────┘                  │
│                 │                                    │
│                 ▼                                    │
│  ┌───────────────────────────────┐                  │
│  │  Tool Result                  │  ← 信封 2（回）   │
│  │  "我做完了，结果是..."         │                  │
│  └───────────────────────────────┘                  │
│                                                     │
└───────────────────────────────────────────────────┘

          ┌───────────────────────────────────┐
          │  子 Agent 的上下文窗口（独立的）    │
          │                                     │
          │  [自己的 System Prompt]             │
          │  [收到的 prompt 作为任务]            │
          │  [自己调用工具的过程和结果...]       │
          │  [最终输出 → 返回给主 Agent]        │
          │                                     │
          └───────────────────────────────────┘
```

### 两个窗口之间共享的只有三样东西

1. **同一个文件系统（磁盘）** — 子 Agent 可以读写主 Agent 能访问的同一批文件
2. **同一套工具**（Read, Grep, Bash 等）
3. **那两封"信"** — prompt（主→子）和 result（子→主）

### 不共享的

- 对话历史：子 Agent 看不到用户跟主 Agent 之前说了什么
- System Prompt：各自有各自的，互不可见
- 中间过程：子 Agent 调了多少次工具、思考了什么，主 Agent 一概不知
- 子 Agent 之间也完全隔离，互相不知道对方的存在

---

## 类比：Tech Lead 和同事

你是一个 Tech Lead（主 Agent），子 Agent 是你的同事。

你不可能把你脑子里所有的上下文都复制到同事脑子里。你只能：

- 写一封清晰的任务描述（prompt）
- 同事自己去看代码、查资料（工具调用）
- 同事做完后给你一个汇报（返回结果）

同事有自己的"工作习惯"（System Prompt），你不需要知道，也控制不了。

---

## 通信的单向性：没有"交互"，只有"委托"

子 Agent **不能反问主 Agent**。它不能说"我需要更多信息"然后等主 Agent 回复。

它只能：

1. 收到任务（prompt）
2. 自己想办法完成（调用工具去读文件、搜索、执行命令等）
3. 返回一个最终结果

如果任务描述不清楚，子 Agent 只能基于自己的判断去做，或者在返回结果里说"我不确定 X，以下是我基于假设 Y 做的结果"。

**这就是为什么 prompt 的质量极其重要 —— 它是子 Agent 唯一的输入上下文。**

---

## 主 Agent 怎么知道该委派谁？—— Description 就是"简历"

主 Agent 对子 Agent 的能力认知 **100% 来自 Agent 工具定义中的 description + tools 列表**。

主 Agent 在自己的 System Prompt 中看到的内容大概长这样：

```
Available agent types and the tools they have access to:

- general-purpose: General-purpose agent for researching
  complex questions... (Tools: *)

- Explore: Fast agent specialized for exploring codebases.
  Use this when you need to quickly find files by patterns...
  (Tools: All tools except Agent, Edit, Write, NotebookEdit)

- Plan: Software architect agent for designing implementation
  plans... (Tools: All tools except Agent, Edit, Write...)

- code-reviewer: Use this agent when a major project step
  has been completed...
```

### 这意味着

| 信息来源 | 主 Agent 从中获得什么 |
|---------|---------------------|
| description 文字 | 什么时候该委派给这个子 Agent |
| Tools 列表 | 子 Agent 能做什么、不能做什么（如 Explore 没有 Edit → 不能改代码） |
| 使用指南（When NOT to use） | 什么时候不应该委派，应该自己做 |

**如果 description 没提到某个能力，主 Agent 就不会因为那个能力去委派它。**

### Description 写得好 vs 写得差

```
❌ "A helpful agent"
→ 主 Agent 完全不知道什么时候该用

✅ "Use this agent to run the full test suite and report
    failures with root cause analysis. Specialized for
    this project's pytest + Docker setup. (Tools: Bash, Read)"
→ 主 Agent 能精准地在需要跑测试时委派，并知道能力边界
```

---

## 主 Agent 该给子 Agent 多少信息？

### 检验标准

> 如果你把这段 prompt 发给一个刚入职的聪明工程师，TA 能不能独立完成任务？
> - 不能 → 信息不够
> - prompt 里大量信息跟任务无关 → 信息太多

### 该给 vs 不该给

| 该给的 | 不该给的 |
|--------|----------|
| 明确的任务目标 | 完整的对话历史 |
| 相关的文件路径/名称（如果已知） | 不相关的背景信息 |
| 约束条件（只读/可修改、范围限制） | 主 Agent 的推理过程 |
| 期望的输出格式 | 其他子 Agent 的任务内容 |

---

## 完整 Case：用户说"帮我把项目里所有的 console.log 替换成 logger 调用"

### Step 1：主 Agent 收到用户请求

主 Agent 的上下文里有用户的话、自己的 System Prompt、之前的对话历史。

### Step 2：主 Agent 决定并行委派两个子 Agent

```
Agent Call 1 (Explore 类型):
prompt: "在当前项目中搜索所有包含 console.log 的文件，
        列出每个文件的路径、出现次数、以及每处的行号和上下文。
        不需要修改任何文件。"

Agent Call 2 (Explore 类型):
prompt: "在当前项目中找到 logger 的定义和使用方式。
        我需要知道：
        1. logger 从哪里 import
        2. 它有哪些方法（info, warn, error 等）
        3. 典型的调用写法是什么样的
        给我 2-3 个真实的使用示例。"
```

### Step 3：两个子 Agent 独立且并行工作

- **子 Agent 1**：用 Grep 搜索 `console.log`，用 Read 读取上下文，整理结果
- **子 Agent 2**：用 Grep 搜索 `logger`/`import.*logger`，找到定义文件，整理使用模式
- 它们互相不知道对方的存在，各自在自己的上下文窗口中工作

### Step 4：结果返回主 Agent

```
Result 1: "找到 15 个文件包含 console.log，共 47 处：
  - src/api/auth.ts: 行 23, 45, 67 (3处)
  - src/utils/helpers.ts: 行 12 (1处)
  - ..."

Result 2: "项目使用 @company/logger 包。
  import: import { logger } from '@company/logger'
  方法: logger.info(), logger.warn(), logger.error()
  示例: logger.info('User logged in', { userId: user.id })
  ..."
```

**主 Agent 看到的就是这两段纯文本。** 中间过程完全不可见。

### Step 5：主 Agent 综合信息，执行修改

现在主 Agent 知道了"哪些文件需要改"和"怎么改"，于是自己用 Edit 工具去修改，或者再委派子 Agent 去改特定文件。

---

## 总结

1. **通信模型是"委托"而非"对话"** — 写清楚任务书，等结果报告，中间过程不可见也不可干预
2. **Prompt 质量决定一切** — 它是子 Agent 唯一的输入上下文
3. **Description 是子 Agent 的简历** — 主 Agent 100% 依赖它来做委派决策
4. **上下文窗口完全隔离** — 共享文件系统和工具，但不共享对话历史和 System Prompt
5. **子 Agent 不能反问** — 信息不足时只能自己判断或在结果中声明不确定性
