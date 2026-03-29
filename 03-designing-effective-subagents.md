# 第三章：Designing effective subagents -- 设计高效的 Subagent

> **课程章节**: 3/4 | **视频时长**: 3:42
> **视频链接**: https://www.youtube.com/watch?v=WPxWKT_OaU4

---

现在你知道如何创建 subagent 了，让我们来看看哪些模式能让它们真正高效。一个配置不当的 subagent 会游离不定、运行太久，或者产生主 agent 无法使用的输出。修复方法归结为四件事：**写好 description、定义输出格式、报告障碍、限制工具访问**。

> **[Claude 注释]** 这四个要素构成了一个完整的 subagent "质量框架"。如果说第二章教你"如何造车"，那这一章教你"如何造一辆好开的车"。每个要素解决一个具体问题：
>
> | 要素 | 解决的问题 |
> |------|-----------|
> | 好的 description | subagent 被错误触发，或收到模糊的指令 |
> | 输出格式 | subagent 不知道何时停止，返回的信息无法使用 |
> | 障碍报告 | 主线程重复踩 subagent 已经踩过的坑 |
> | 限制工具 | subagent 做了不该做的事情 |

---

## Subagent 配置数据是如何被使用的

当你发送消息给主 context window agent 时，**每个可用 subagent 的名称和描述都会被包含在 system prompt 中**。这就是主 agent 决定启动哪个 subagent 以及何时启动的方式。如果你想更好地控制 subagent 何时被自动触发，名称和描述就是你应该调整的内容。

Description 还扮演第二个角色。当主 agent 启动一个 subagent 时，它会**编写一个 input prompt 来启动任务**。它使用 description 作为编写该 prompt 的指导。所以 description 不仅控制 subagent *何时*运行——它还影响 *subagent 被告知要做什么*。

> **[Claude 注释]** 这是一个非常重要但容易被忽视的机制。让我用一个具体的数据流来说明：
>
> ```
> 你的消息: "帮我看看最近的代码改动有没有问题"
>         ↓
> 主 Agent 的 system prompt 中包含:
>   - code-reviewer: "Use this agent when..."
>   - test-runner: "Use this agent when..."
>         ↓
> 主 Agent 决定: "应该用 code-reviewer"
>         ↓
> 主 Agent 根据 description 编写 input prompt:
>   "Review the recent code changes in files X, Y, Z..."
>         ↓
> code-reviewer subagent 收到这个 input prompt，开始工作
> ```
>
> **关键洞察**：description 影响的是主 Agent 写给 subagent 的"任务书"的质量。一个好的 description 等于一个好的"任务书模板"。

---

## 编写能影响 Input Prompt 的 Description

以一个 code review subagent 为例。如果 description 很笼统，主 agent 可能会写出这样的 input prompt："use get diff to find the current changes."——这太模糊了。Subagent 必须自己搞清楚哪些文件重要。

如果你把 description 更新为包含类似"**You must tell the agent precisely which files you want it to review**"的内容，主 agent 现在就会写出一个更具体的 input prompt，列出实际要审查的文件。

这种技术同样适用于不同类型的 subagent。例如，在 web search subagent 的 description 中添加"**return sources that can be cited**"会导致主 agent 在委派任务时包含该指令。

> **[Claude 注释]** 这是一种非常巧妙的 **间接控制** 技术。你不是直接控制 subagent 的行为（那是 system prompt 的工作），而是通过 description 来**控制主 agent 如何"下达命令"**给 subagent。
>
> 打个比方：你不是直接告诉厨师怎么做菜（system prompt），而是告诉领班"你给厨师下单时，一定要注明客人的过敏信息"（description）。领班（主 Agent）就会每次下单时自动附上这个信息。
>
> **实用模式**：在 description 中加入 "You must tell the agent..." 或 "Always include ... when delegating" 这类 meta-instruction（元指令），可以显著提升 subagent 收到的 input prompt 质量。

---

## 定义输出格式

**你能对 subagent 做的最重要的改进就是在 system prompt 中定义输出格式。** 这做了两件事：

1. **它创建了自然的停止点**——subagent 知道当它填满格式的每个部分时就完成了
2. **它防止 subagent 运行太久**——没有定义输出格式的 subagent 很难判断何时已经做了足够的研究，往往会运行得比必要时间长得多

> **[Claude 注释]** 这一点怎么强调都不为过。没有输出格式的 subagent 就像一个没有 deadline 和 deliverable 定义的任务——它不知道什么时候算"做完了"。
>
> **现实类比**：想象你让实习生"调研一下竞品"。如果你不告诉他输出格式（"给我一个表格，列出每个竞品的价格、功能、市场份额"），他可能花三天写一份 50 页的报告，也可能花 5 分钟给你一句"他们都挺好的"。定义输出格式 = 定义"完成标准"。

下面是一个 code review subagent 的结构化输出格式示例：

```
Provide your review in a structured format:
1. Summary: Brief overview of what you reviewed and overall assessment
2. Critical Issues: Any security vulnerabilities, data integrity risks,
   or logic errors that must be fixed immediately
3. Major Issues: Quality problems, architecture misalignment, or
   significant performance concerns
4. Minor Issues: Style inconsistencies, documentation gaps, or minor
   optimizations
5. Recommendations: Suggestions for improvement, refactoring
   opportunities, or best practices to apply
6. Approval Status: Clear statement of whether the code is ready to
   merge/deploy or requires changes
```

这个格式给了 subagent 一个清晰的 checklist 来逐项完成。一旦每个部分都填完，subagent 就知道它可以停止了。

> **[Claude 注释]** 注意这个格式的巧妙之处：
>
> - **从严重到轻微排列**（Critical → Major → Minor → Recommendations），确保最重要的问题不会被淹没
> - **最后有一个明确的"Approval Status"**，这是一个二元决策（merge or not），让主 agent 能直接采取行动
> - **每个 section 都有清晰的定义**（"security vulnerabilities, data integrity risks..."），而不仅仅是一个模糊的标题
>
> 你可以把这个模式套用到任何类型的 subagent。比如一个"文档审查 subagent"的输出格式可能是：
> 1. Coverage（覆盖率）：哪些 API 有文档，哪些缺失
> 2. Accuracy（准确性）：文档是否与实际代码一致
> 3. Clarity（清晰度）：哪些描述让人困惑
> 4. Action Items（待办事项）：按优先级排列的修改建议

---

## 报告障碍

当 subagent 在工作中发现了变通方案（workaround）——比如解决了一个依赖问题或发现某个命令需要特定的 flag——这些细节需要出现在它返回的摘要中。如果没有，主线程就必须自己重新发现同样的解决方案，这浪费了时间和 token。

你想要浮现的信息类型包括：

- 环境设置问题或环境怪癖
- 在任务过程中发现的变通方案
- 需要特殊 flag 或配置的命令
- 导致问题的依赖项或 import

获取这些信息的方法是**在输出格式中明确要求**。在输出模板中添加一个"Obstacles Encountered"部分可以可靠地浮现这些信息。

```
7. Obstacles Encountered: Report any obstacles encountered during the
   review process. This can be: setup issues, workarounds discovered or
   environment quirks. Report commands that needed a special flag or
   configuration. Report dependencies or imports that caused problems.
```

> **[Claude 注释]** 这是一个在软件工程中被广泛低估的实践。类比 **postmortem（事后复盘）** 文化：
>
> 在 Google、Meta 等公司，每次出 incident 后都会写一份 postmortem，不仅记录"发生了什么、怎么修的"，还记录"在排查过程中踩了什么坑"。这些"坑"就是 obstacles。
>
> **为什么这对 subagent 特别重要？** 因为 subagent 的 context 会被丢弃！如果 subagent 在研究过程中发现"跑测试之前需要先 `export NODE_ENV=test`"，但它没报告这个发现，主线程之后尝试跑测试时就会再次遇到同样的问题，然后又要花时间去排查。
>
> **黄金法则**：如果 subagent 发现了任何"反直觉"的东西——任何你不会从文档中轻易得知的东西——它都应该报告。

---

## 限制工具访问

不是每个 subagent 都需要访问所有工具。想想 subagent 实际需要做什么，然后只给它完成工作所需的工具。这做了两件事：防止意外的副作用，以及当你有多个 subagent 时让每个 subagent 的角色更清晰。

以下是常见 subagent 类型的工具访问思路：

- **Research / 只读 subagent**——只需要 `Glob`、`Grep` 和 `Read`。不可能意外修改文件。
- **Code reviewer**——需要 `Bash` 访问来运行 `git diff` 看看什么改变了，但仍然不需要 `Edit` 或 `Write`。
- **Styling / 代码修改 agent**——这就是你给 `Edit` 和 `Write` 访问的地方，因为 subagent 的工作就是实际修改你的代码。

> **[Claude 注释]** 这和 **Unix 文件权限** 的思路一模一样：
>
> | 角色 | 类比 Unix 权限 | 工具权限 |
> |------|-------------|---------|
> | Research agent | `r--`（只读） | Glob, Grep, Read |
> | Code reviewer | `r-x`（只读+执行） | Glob, Grep, Read, Bash |
> | Code modifier | `rwx`（完全权限） | Read, Edit, Write, Bash |
>
> **进阶技巧**（来自官方文档）：如果你需要更精细的控制——比如"允许 Bash 但只能跑 SELECT 查询"——可以用 `hooks` 中的 `PreToolUse` 钩子来做命令级别的验证。第二章的官方文档部分有详细的例子。

---

## 总结

高效的 subagent 共享四个特征：

1. **具体的 description**——description 控制 subagent 何时被启动以及收到什么指令。写 description 时要同时考虑这两个目标。
2. **结构化输出**——在 system prompt 中定义输出格式，让 subagent 知道何时完成，并返回主线程可以使用的信息。
3. **障碍报告**——在输出格式中包含一个部分用于记录变通方案、怪癖和问题，这样主线程就不必重新发现它们。
4. **限制工具访问**——只给 subagent 它实际需要的工具。只读用于研究，Bash 用于审查者，Edit/Write 只给需要修改代码的 agent。

这些模式每个单独来看都很简单，但组合在一起，它们能把一个"模糊地试图帮忙"的 subagent 变成一个**专注的、可预测的工作者**——按时完成并清晰汇报。

> **[Claude 注释]** 让我用一个完整的例子把这四个要素整合起来。假设你要创建一个 **API 安全审计 subagent**：
>
> ```yaml
> ---
> name: api-security-auditor
> description: Use this agent to audit API endpoints for security
>   vulnerabilities. You must tell the agent which specific endpoints
>   or route files to audit. Always request OWASP Top 10 coverage.
> tools: Glob, Grep, Read, Bash
> model: sonnet
> ---
>
> You are a security auditor specializing in API security.
>
> When invoked:
> 1. Read the specified route/endpoint files
> 2. Trace the request handling chain
> 3. Check for OWASP Top 10 vulnerabilities
>
> Provide your audit in this format:
> 1. Scope: Which endpoints were audited
> 2. Critical Vulnerabilities: SQL injection, auth bypass, etc.
> 3. High Risk: Missing rate limiting, improper error handling
> 4. Medium Risk: Missing input validation, loose CORS
> 5. Low Risk: Missing security headers, verbose errors
> 6. Obstacles Encountered: Any issues during the audit
> 7. Remediation Priority: Ordered list of fixes by urgency
> ```
>
> 注意这个例子如何体现了所有四个要素：
> - **Description** 指导主 Agent 提供具体的文件路径
> - **输出格式** 有 7 个明确的 section
> - **障碍报告** 有专门的 section
> - **工具** 只包含只读 + Bash（审计不需要修改代码）
