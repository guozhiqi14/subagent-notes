# 第二章：Creating a subagent -- 创建 Subagent

> **课程章节**: 2/4 | **视频时长**: 3:45
> **视频链接**: https://www.youtube.com/watch?v=arD6qEWa2Xc

---

Claude Code 自带内置 subagent，但你也可以创建自己的。自定义 subagent 专注于特定任务——比如审查代码、编写测试或检查文档。它们被定义为带有 YAML frontmatter 的 Markdown 文件，告诉 Claude 何时使用这个 subagent 以及 subagent 应该如何行为。

> **[Claude 注释]** 这种"Markdown + YAML frontmatter"的设计模式非常优雅。如果你熟悉 Jekyll、Hugo 等静态网站生成器，你会发现这是同样的思路：元数据放在文件头部的 YAML 块中，正文内容用 Markdown 书写。Anthropic 选择这种格式而不是 JSON 或其他配置格式，是因为：
> 1. **人类可读性好**——Markdown 天然适合写长文本（system prompt）
> 2. **版本控制友好**——可以直接 commit 到 Git 仓库，方便团队协作
> 3. **编辑门槛低**——任何文本编辑器都能编辑，不需要特殊工具

---

## 创建 Subagent

创建 subagent 最简单的方式是使用 `/agents` slash command。这会打开管理 subagent 的主界面。从那里选择 **Create new agent**。

你首先会被要求选择 subagent 的**作用范围（scope）**：

- **Project-level（项目级）**——仅在当前项目中可用
- **User-level（用户级）**——在你机器上的所有项目中共享

> **[Claude 注释]** 怎么选择 scope？一个简单的判断规则：
>
> - 如果这个 subagent 是**针对特定项目**的（比如"这个项目的 API 规范审查器"），选 **Project-level**，文件存在 `.claude/agents/` 下，可以 commit 给团队共享
> - 如果这个 subagent 是**你个人的通用工具**（比如"我的代码风格审查器"），选 **User-level**，文件存在 `~/.claude/agents/` 下，跟着你走
>
> 类比：Project-level 就像项目里的 `.eslintrc`（项目团队共享的规则），User-level 就像你个人的 `~/.gitconfig`（你自己的全局配置）。

接下来，你可以选择如何创建它。你可以手动编写配置，但**推荐的方式是让 Claude 帮你生成**。只需描述你希望 subagent 做什么，Claude 会根据你的输入生成名称、描述和 system prompt。

---

## 自定义工具

在创建过程中，你有机会自定义 subagent 可以访问哪些工具。工具类别包括：

- Read-only tools（只读工具）
- Edit tools（编辑工具）
- Execution tools（执行工具）
- MCP tools（MCP 工具）
- Other tools（其他工具）

想想你的 subagent 实际需要什么。一个 code reviewer 可能不需要 edit tools——它应该读取和分析代码，而不是修改代码。不过，你可能想保留 execution tools，这样它可以更容易地识别待处理的变更。

> **[Claude 注释]** **最小权限原则（Principle of Least Privilege）** 在这里同样适用。给 subagent 的工具越少，它就越专注，也越安全。这和 Unix 权限、Docker 容器权限、AWS IAM 策略是同一个理念。
>
> 一个真实的例子：如果你的 code review subagent 同时有 `Read` 和 `Edit` 权限，它可能在审查过程中"顺手"修改了代码——这不是你想要的行为。限制它只有 `Read` 权限，就从结构上防止了这种意外。

---

## 选择模型和颜色

配置完工具后，你需要选择哪个 Claude 模型来驱动 subagent。选项包括：

- **Haiku**——最适合快速、轻量的任务
- **Sonnet**——速度和深度之间的良好平衡
- **Opus**——最适合复杂分析
- **Inherit（继承）**——使用主会话正在运行的任何模型

最后，你选择一个**颜色**。这会在 UI 中显示出来，让你可以快速分辨哪个 subagent 正在活动。这是一个小细节，但当你有多个 subagent 运行时很有帮助。

> **[Claude 注释]** 模型选择是一个**成本-质量-速度**的三角权衡：
>
> | 模型 | 速度 | 能力 | 成本 | 适合场景 |
> |------|------|------|------|---------|
> | Haiku | 最快 | 基础 | 最低 | 文件搜索、简单分析 |
> | Sonnet | 中等 | 强 | 中等 | 代码审查、一般性任务 |
> | Opus | 最慢 | 最强 | 最高 | 复杂架构分析、深度推理 |
> | Inherit | 取决于主会话 | 取决于主会话 | 取决于主会话 | 不确定时的默认选择 |
>
> **实际建议**：大多数 subagent 用 **Sonnet** 就够了。只有在 subagent 需要做复杂推理（如架构设计、安全审计）时才考虑 Opus。对于只是搜索和浏览的 subagent，Haiku 是最经济的选择。

---

## 配置文件

创建完成后，subagent 配置文件会保存到你的项目中（通常在 `.claude/agents/your-agent-name.md`）。下面是一个典型的 subagent 配置文件：

```yaml
---
name: code-quality-reviewer
description: Use this agent when you need to review recently written
  or modified code for quality, security, and best practice compliance.
tools: Bash, Glob, Grep, Read, WebFetch, WebSearch
model: sonnet
color: purple
---

You are an expert code reviewer specializing in quality assurance,
security best practices, and adherence to project standards. Your role
is to thoroughly examine recently written or modified code and identify
issues that could impact reliability, security, maintainability, or
performance.
```

让我们逐个字段说明：

- **`name`**——subagent 的唯一标识符。这是你引用它的方式，可以直接向 Claude 请求使用，或者在消息中输入 `@agent code-quality-reviewer`。
- **`description`**——控制 Claude 何时决定使用这个 subagent。必须是单行（如果需要换行，使用转义换行符 `\n`）。你可以在这里包含示例对话，帮助 Claude 理解何时适合委派。
- **`tools`**——列出 subagent 可以访问的工具。这与你在生成过程中选择的内容一致，但你可以随时在这里编辑列表。
- **`model`**——指定使用哪个 Claude 模型：`sonnet`、`opus`、`haiku` 或 `inherit`。
- **`color`**——UI 中用于标识 subagent 的颜色。

> **[Claude 注释]** 几个容易被忽略的关键细节：
>
> 1. **`description` 是最重要的字段之一**——它不仅决定"何时触发"，还影响"主 agent 写给 subagent 的 input prompt 内容"。下一章会详细讲。
> 2. **`tools` 列表是白名单模式**——只列出的工具才能用。如果你想用"排除法"（给所有工具，但排除某几个），可以用 `disallowedTools` 字段代替。
> 3. 还有一些课程没提到的高级字段（来自官方文档）：`permissionMode`（权限模式）、`maxTurns`（最大轮次）、`hooks`（生命周期钩子）、`memory`（持久化记忆）、`skills`（预加载技能）等。这些在你需要更精细控制时非常有用。

---

## System Prompt

Markdown 文件的正文（YAML frontmatter 下面的所有内容）就是 **system prompt**。这是你给 subagent 的指令：它应该专注于什么，应该如何分析事物，以及应该如何向主 agent 汇报结果。

一个写得好的 system prompt 是区分有用 subagent 和抓不住重点的 subagent 的关键。对 subagent 应该寻找什么以及如何组织输出要具体明确。

> **[Claude 注释]** System prompt 的质量直接决定了 subagent 的表现。几个写好 system prompt 的原则：
>
> 1. **明确角色**："You are a code reviewer" 比 "Help with code" 好得多
> 2. **明确行为**：告诉它"When invoked, do X, then Y, then Z"——给一个清晰的步骤流程
> 3. **明确输出格式**：定义它应该如何组织返回结果（第三章会详细讲）
> 4. **明确限制**：告诉它什么**不**应该做，这和告诉它什么应该做同样重要
>
> 一个简单但有效的模板：
> ```
> You are a [角色]. When invoked:
> 1. [第一步]
> 2. [第二步]
> 3. [第三步]
>
> Focus on: [重点关注的内容]
> Do not: [不应该做的事情]
> Output format: [输出格式定义]
> ```

---

## 让 Claude 自动使用你的 Subagent

如果你想让 Claude 在无需你明确要求的情况下将任务委派给 subagent，在 description 字段中加入 **"proactively"** 这个词。例如：

```yaml
description: Proactively suggest running this agent after major code changes...
```

你也可以在 description 中添加**示例对话**，帮助 Claude 理解 subagent 应该在哪些具体场景中被使用。示例越具体，Claude 在知道何时委派方面就越准确。

> **[Claude 注释]** "proactively" 这个关键词相当于一个**触发器信号**。没有它，Claude 会等你主动请求才使用 subagent。有了它，Claude 会在它认为合适的时候主动触发。
>
> **类比**：就像告诉你的助理"每次我提交代码后，主动帮我做一下 code review"vs"我叫你做 code review 的时候才做"。前者是 proactive（主动的），后者是 reactive（被动的）。
>
> 不过要注意：过度使用 proactive subagent 可能会导致不必要的 token 消耗。只对真正高频且有价值的任务设置 proactive。

---

## 测试你的 Subagent

创建完 subagent 后，通过做一些代码修改并请求 Claude 审查它们来测试它。

如果 subagent 在你期望它被使用时没有被使用，回去检查 description。添加更具体的示例和触发场景有助于 Claude 理解何时将工作委派给你的 subagent。

> **[Claude 注释]** 调试 subagent 的常见问题及解决方案：
>
> | 问题 | 可能原因 | 解决方案 |
> |------|---------|---------|
> | Subagent 不被触发 | Description 太模糊 | 添加具体的触发场景描述 |
> | Subagent 被触发但效果差 | System prompt 不够具体 | 细化 system prompt 的步骤和输出格式 |
> | Subagent 运行太久 | 没有定义输出格式 | 添加结构化输出模板（第三章会讲） |
> | Subagent 做了不该做的事 | 工具权限过多 | 收紧 tools 列表 |
>
> **记住**：创建 subagent 后需要**重启会话**或使用 `/agents` 重新加载才能生效。手动创建的文件不会自动被识别。
