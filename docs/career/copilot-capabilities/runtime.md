---
layout: default
title: Declarative Agent 接入与执行
parent: Outlook Copilot Agent 能力开发
grand_parent: 工作经历
nav_order: 1
permalink: /docs/career/copilot-capabilities/runtime/
---

# Declarative Agent 接入与执行

## 平台与项目边界

Outlook Copilot 基于 Microsoft 365 Copilot Declarative Agent 平台实现。平台提供：

- 模型与会话；
- Tool Call 与 Tool Result 循环；
- Extension 调度和认证；
- Planning、计划状态和用户确认；
- 流式返回、预算和基础 Trace。

我没有开发 Agent Runtime，也没有实现附件提取和邮件读写 Extension。我的工作是：

- 编写 Agent Instructions 和场景 Prompt；
- 将 Side Panel 的选区、邮件、附件和文件夹 Context 交给 Agent；
- 在 Agent Definition 中引用已有 Extension；
- 设计工具 Description、参数使用方式和 Tool Result 处理规则；
- 配置哪些场景可以使用哪些工具；
- 处理工具错误、部分成功和幂等语义对 Agent 行为的影响；
- 维护发布配置和线上行为指标。

```text
Outlook Side Panel
→ Declarative Agent Definition
→ Agent Instructions、Context 和 Extension Schema
→ Microsoft 365 Copilot 平台调用模型
→ 模型生成 Tool Call
→ 平台调用已有 Extension
→ Tool Result 返回模型
→ 回答、继续调用、澄清或等待确认
```

## Agent Definition

划词解释、附件总结和邮箱整理接入同一个 Declarative Agent。Agent Definition 声明：

- Agent 名称、Description 和 Instructions；
- Side Panel 入口；
- 可用 Extension；
- Conversation Starter；
- Extension 的工具名称、Description、参数 Schema 和认证；
- Agent Definition Version。

场景配置决定入口附加什么 Instructions、Context 和工具：

```json
{
  "scenarioId": "mailbox_organization",
  "instructionVersion": "mailbox-organization-v3",
  "contextVersion": "folder-context-v12",
  "allowedExtensions": [
    "search_outlook_context",
    "read_outlook_message",
    "move_messages",
    "archive_messages",
    "flag_messages"
  ]
}
```

划词解释只使用选区和固定 Instructions；附件总结加载内容提取工具；邮箱整理加载 Insight 查询和邮件写工具。平台负责模型轮次与工具调度，现有 Extension 负责权限、业务规则、幂等和实际执行。

## 请求进入 Agent

Outlook 客户端提供：

- 用户消息；
- 当前 Message ID、Conversation ID、Attachment ID 或 Folder ID；
- 选区和入口类型；
- Locale、Time Zone 和会话 ID。

项目的 Side Panel 接入层根据入口构造场景 Context，并提交给 Declarative Agent：

```text
Agent Instructions
+ 场景 Instructions
+ 当前用户消息
+ 平台会话历史
+ Outlook 界面 Context
+ 当前可用工具 Schema
```

客户端只传对象 ID 和必要选区，不预加载整个邮箱、全部附件或完整 Conversation。模型需要更多信息时，通过工具按需获取。

## Tool Call 与 Tool Result

工具是通用概念，在 Microsoft 365 Copilot 中以 Extension 形式注册。平台根据 Extension Definition 把 Tool Schema 提供给模型。

```json
{
  "name": "search_outlook_context",
  "arguments": {
    "query": "find budget updates",
    "startTime": "2025-09-01T00:00:00Z",
    "endTime": "2025-10-01T00:00:00Z",
    "participantIds": ["person-4821"]
  }
}
```

平台负责：

1. 校验 Tool Schema；
2. 附加用户身份和 Extension 认证；
3. 调用 Extension Endpoint；
4. 将 Tool Call 和 Tool Result 放回模型会话；
5. 继续下一轮模型调用。

Extension 返回稳定 Tool Result：

```json
{
  "status": "success",
  "data": {
    "results": []
  },
  "partial": false,
  "warnings": [],
  "citations": []
}
```

我关注 Tool Result 是否足以让模型正确判断下一步，因此在 Instructions 中明确：

- `partial` 时不能把证据说成完整结果；
- `permission_denied` 不能尝试绕过权限；
- `version_conflict` 要重新读取或重新计划；
- `result_unknown` 不能换参数重复执行；
- `partial_success` 要区分成功、失败和未确认项。

工具自身的重试、幂等和资源版本由已有 Extension 实现。我需要理解并正确使用它们的错误契约，不能声称实现这些后端机制。

## 多轮对话

平台保存会话历史和当前计划。项目通过 Instructions 和 Side Panel Context 规定信息怎样继承：

- 当前邮件、附件和文件夹 Anchor 随 Side Panel 变化更新；
- 用户明确确认的时间、人员和范围可以在同一任务中继承；
- 旧搜索结果、资源版本和权限状态不能长期复用；
- 用户修改范围或动作后，平台中的旧计划与确认失效。

意图模糊时，Agent 先使用界面 Context 和会话历史消歧。低风险读取可以使用有界默认值；写操作缺少对象、范围或动作时必须最小澄清。

```text
识别当前任务
→ 继承仍有效的约束
→ 检查缺失信息
→ 回答、调用工具或澄清
→ Tool Result 返回平台会话
→ 继续或结束
```

完整消息由平台管理。项目只控制本轮输入哪些界面 Context、工具结果和场景规则，不自行实现会话数据库或模型循环。

## 邮箱整理的 Planning

邮箱整理使用 Copilot 平台已有的 Planning 和确认能力：

```text
用户目标
→ 模型调用 Query 工具探索邮件
→ 根据 Tool Result 改写 Query 或读取详情
→ 平台形成结构化动作计划
→ Side Panel 展示计划
→ 用户确认
→ 平台调用邮件写 Extension
→ Tool Result 返回模型汇总
```

我负责的部分是：

- Instructions 中说明怎样从模糊目标提取时间、Folder 和排除项；
- 配置探索阶段可以调用哪些只读工具；
- 定义计划必须包含 Message ID、动作、目标 Folder 和理由；
- 确认界面需要展示范围、数量和代表性邮件；
- 根据 `version_conflict`、`partial_success` 和 `result_unknown` 规定 Agent 的后续行为；
- 根据平台 Trace 和线上结果分析 Planning、工具选择和确认范围。

平台负责 Plan Version、确认状态和写工具调度；邮件写 Extension 负责权限、幂等和资源版本检查。

## 三个场景

| 场景 | Declarative Agent 配置 | 执行形态 |
| --- | --- | --- |
| 划词解释 | 固定 Instructions、选区 Context、不加载写工具 | 一次模型调用并流式返回 |
| 附件总结 | Attachment ID、内容提取工具和 Citation 规则 | 固定提取链路后生成总结 |
| 邮箱整理 | Folder Context、Insight 与邮件读写工具、确认规则 | 模型根据 Tool Result 动态探索，平台负责计划与确认 |

同一个 Declarative Agent 可以包含固定快速路径和动态工具路径。平台提供通用 Agent 能力，项目通过 Instructions、Context 和 Extension 配置塑造 Outlook 场景行为。

## Trace

Microsoft 365 Copilot 平台提供 Run、模型调用、Planning、确认和 Tool Call 的 Trace。项目使用这些字段定位问题：

```text
Copilot Run
→ Agent Definition / Instructions Version
→ Model Turn
→ Tool Call ID / Extension Version
→ Tool Result
→ Platform Plan / Confirmation
→ Final Answer
```

我关注模型看到了什么 Context、为什么选择某个工具、Tool Result 是否被正确使用、确认范围是否符合用户目标。完整邮件、附件、选区和用户 Token 不进入项目日志。发布和指标见 [发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)。
