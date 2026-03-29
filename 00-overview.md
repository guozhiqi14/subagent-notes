# Introduction to Subagents -- 课程翻译与注释

> **来源**: Anthropic Academy / Skilljar 课程
> **原始链接**: https://anthropic.skilljar.com/introduction-to-subagents
> **翻译 & 注释**: Claude (基于课程原文 + 官方文档补充)

---

## 课程简介

Subagent（子代理）是在更长的 Claude Code 会话中提升效率的最实用方式之一。它们允许你将任务委派给隔离的助手，这些助手独立完成工作，然后只返回你需要的信息——保持你的主 context window 干净、对话保持专注。

**在这门课程中，你将学到：**

- **Subagent 的工作原理** —— 当 Claude Code 启动一个独立的 context window 时会发生什么，输入如何流入，以及摘要如何返回
- **创建自定义 subagent** —— 使用 `/agents` 命令构建针对你工作流的 subagent，从 code reviewer 到文档生成器
- **设计高效的 subagent** —— 让 subagent 可靠运行的模式，包括结构化输出格式、障碍报告和限制工具访问
- **何时使用它们（以及何时不用）** —— 关于 subagent 最有价值的场景和需要避免的常见反模式的实用指导

**学完课程后**，你将知道如何将复杂工作拆解为专注的小块、构建能按时完成并清晰汇报的 subagent，以及在何时做出"是否值得委派"的正确判断。

---

## 目录

| 章节 | 标题 | 文件 | 视频时长 |
|------|------|------|----------|
| 1 | [What are subagents? -- 什么是 Subagent？](./01-what-are-subagents.md) | `01-what-are-subagents.md` | 2:49 |
| 2 | [Creating a subagent -- 创建 Subagent](./02-creating-a-subagent.md) | `02-creating-a-subagent.md` | 3:45 |
| 3 | [Designing effective subagents -- 设计高效的 Subagent](./03-designing-effective-subagents.md) | `03-designing-effective-subagents.md` | 3:42 |
| 4 | [Using subagents effectively -- 高效使用 Subagent](./04-using-subagents-effectively.md) | `04-using-subagents-effectively.md` | 4:44 |

---

## 阅读说明

- **原文翻译**：正常文字段落为课程原文的忠实翻译
- **我的注释**：以 `> **[Claude 注释]**` 开头的引用块是我自己的理解和补充说明
- **技术术语**：subagent、context window、token、system prompt 等特定技术术语保留英文
- **代码示例**：代码块保留英文原文，注释部分会翻译

---

## 补充资源

- [官方文档: Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [YouTube 播放列表: Claude Code subagents](https://www.youtube.com/playlist?list=PLmWCw1CzcFilWIFAY4hapAgFtGB7UlvVQ)
