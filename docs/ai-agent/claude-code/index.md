---
layout: default
title: 从 Claude Code 深入 Agent 系统
parent: AI Agent 工程
nav_order: 8
has_children: true
permalink: /docs/ai-agent/claude-code/
---

# 从 Claude Code 深入 Agent 系统

用户对 Claude Code 说：“修复登录测试失败，并确认没有破坏其他登录方式。”随后发生的事情很像一位工程师在工作：读取项目规则，搜索代码，形成假设，修改文件，运行测试，再根据结果调整。

模型本身只能接收消息并生成下一段内容。让它能够持续完成任务的是外层的 Agent Runtime，也常被称为 Agent Harness。Claude Code 负责组装当前上下文、暴露工具、执行权限检查、保存会话，并在上下文过长时压缩历史；Claude 模型在这些条件下选择下一步。

这一章借 Claude Code 观察四个问题：

1. 每次模型调用之前，Context 到底怎样拼出来；
2. 模型返回工具请求后，Runtime 怎样执行并把结果送回模型；
3. 对话变长后，信息怎样压缩、恢复和跨会话保存；
4. 子 Agent 怎样用独立 Context 完成任务，再把必要结果带回主会话。

这些细节帮助理解前文中的上下文、工具、状态和执行控制。它们不构成业务 Agent 的标准架构。代码 Agent 依赖仓库、文件和终端，退款或故障调查 Agent 会连接完全不同的业务对象；可以迁移的是设计判断。

## 一次任务由谁完成

Claude Code 的核心结构可以拆成三层：

| 层次 | 在一次代码任务中的作用 |
| --- | --- |
| Claude 模型 | 根据当前 Context 输出文本或 `tool_use` 请求 |
| Claude Code Runtime | 组装请求、管理消息历史、检查权限、执行工具并判断是否继续 |
| 本地或远程环境 | 保存仓库和文件，真正运行搜索、编辑、测试与外部服务调用 |

模型不会自己打开文件，也不会直接启动测试。它生成结构化工具请求，Runtime 执行后再把结果作为新消息交给模型。下一次调用看到的 Context 已经发生变化，模型才有依据改变路径。

因此，一次任务并非一段很长的模型输出，而是多次模型调用与环境动作组成的闭环。

## 四篇文章怎样衔接

[Claude Code 怎样组装 Context](context-assembly/)从一次会话启动讲起，说明 System Prompt、环境信息、`CLAUDE.md`、Auto Memory、Skills、MCP 工具和会话历史分别在何时进入模型。

[模型的工具请求怎样变成真实动作](tool-loop/)展开一轮工具调用：`tool_use` 怎样被权限规则和 Hook 检查，工具如何执行，`tool_result` 为什么会成为下一轮输入。

[长会话怎样压缩与恢复](session-compaction/)区分 Context Window、完整 Transcript、Checkpoint 和长期 Memory，说明 `/compact` 之后哪些信息会重新注入，哪些必须再次读取。

[子 Agent 怎样隔离 Context](subagents/)解释主 Agent 为什么不应亲自读取所有文件，以及独立 Context、任务委派、结果摘要与恢复机制怎样配合。
