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

Outlook Copilot 基于 Microsoft 365 Copilot 的 Declarative Agent 平台实现。本文统一使用“Microsoft 365 Copilot 平台”指代承载 Declarative Agent 的平台运行时，它提供：

- 模型、Conversation 和 Tool Call 循环；
- Extension 调度、认证与 Tool Result 回传；
- Planning、计划状态和用户确认；
- 流式输出、执行预算和基础 Trace。

项目不自建 Agent Runtime，也不实现附件提取、邮件搜索和邮件写入 Extension。我的工作落在 Outlook Agent 的行为层：Agent Instructions、Side Panel Context、场景路由、既有工具的选择与契约、错误后的 Agent 行为，以及能力版本的评测和发布。

```text
Outlook Side Panel
→ Declarative Agent Definition
→ Instructions、Context 与可用 Extension Schema
→ Microsoft 365 Copilot 平台调用模型
→ Tool Call
→ 平台调用已有 Extension
→ Tool Result 返回模型
→ 回答、继续探索、澄清或等待确认
```

## Agent 代码以什么形式存在

Declarative Agent 不是一套由项目团队常驻运行的 Agent 后端。我的代码分成两类，分别保存在不同的产品仓库中：

1. **Agent 声明与行为配置**：保存在 Outlook Copilot Agent 的配置仓库中，包括 Agent Definition、Instructions、Capability 配置、Extension 引用、工具 Description 和发布版本；
2. **Outlook 接入代码**：保存在 Outlook 客户端或接入层代码中，负责把当前 Message、选区、Attachment、Folder 和入口类型转换成 Side Panel Context，再提交给 Microsoft 365 Copilot 平台。

已有的 Insight、附件提取和邮件写入 Extension 各自部署在对应团队的服务仓库中，不属于这份 Agent 代码。Agent 仓库只引用它们的声明、Schema 和版本，不复制 Handler 实现。

对照公开的 Microsoft 365 Declarative Agent 形态，Agent 声明通常会被构建成一个应用包：

```text
Agent 源码仓库
├─ manifest.json             # 外层 Microsoft 365 应用声明
├─ declarativeAgent.json     # Agent 名称、说明、Instructions、Capability 与 Action 引用
├─ instructions.txt          # 较长的 Agent Instructions
├─ actions / plugins         # 已有 Extension 的声明引用
└─ icons / localization      # 展示与本地化资源
```

内部项目的目录名和构建流程不一定与公开模板完全相同，但产物角色一致：外层应用声明负责在 Outlook 和 Microsoft 365 中注册、展示 Agent；Declarative Agent Definition 描述模型应遵守的 Instructions、可用能力和 Action；插件或 Extension 声明把工具 Schema 暴露给平台。

```text
Git 中的 Agent Definition 与 Instructions
→ 构建和配置校验
→ 生成带版本的 Agent 发布包
→ 发布到 Microsoft 365 Copilot 平台
→ Outlook Side Panel 按 Agent ID / Version 加载
```

发布后，这些文件不会启动成一个属于项目团队的独立进程。Microsoft 365 Copilot 平台保存并加载指定版本的 Agent Definition，在用户打开 Outlook Side Panel 时创建 Conversation，组装 Instructions、Side Panel Context 和工具 Schema，然后运行模型与工具循环。

因此，如果面试官问“你的 Agent 代码运行在哪里”，准确回答是：

- **Agent 行为定义**以声明文件和 Instructions 的形式存在于 Outlook Copilot 的源码与发布包中，发布后由 Microsoft 365 Copilot 平台托管和执行；
- **当前邮件等 Context 接入代码**运行在 Outlook 客户端或接入层；
- **工具实现**运行在各个既有 Extension 后端；
- 项目本身没有一套名为 Outlook Agent Runtime 的常驻服务或容器。

## 为什么使用一个 Agent

划词解释、附件总结和邮箱整理共享 Outlook Side Panel、用户身份、当前邮件对象和会话历史。拆成多个 Agent 会增加任务切换、Context 传递和权限配置成本，而三个场景并没有独立身份、独立团队协作或 Context 隔离到必须拆分的程度，因此接入同一个 Declarative Agent。

统一 Agent 不等于所有请求走相同流程：

- 划词解释有明确入口和选区，使用固定 Prompt 快速返回；
- 附件总结有明确 Attachment ID，按固定提取链路取得内容后生成总结；
- 普通问答和邮箱整理允许模型根据 Tool Result 动态检索、读取、澄清和形成动作计划。

这里不使用 Multi-Agent。任务共享同一用户目标和邮件 Context，拆分后没有并行或专业化收益，反而需要处理 Agent 间状态和权限传递。

Microsoft 365 Copilot 平台在这个项目中承担 Agent Harness：维护 Conversation、组织模型与工具循环、执行 Planning 和确认、控制预算并记录 Trace。工具通过平台 Extension 接入，不需要项目再引入 MCP；三个 Capability 之间也不存在 A2A 通信。项目选择的是平台已有集成形态，而不是重新实现一套 Harness 或协议层。

## Agent Definition

Agent Definition 声明 Agent 名称、Description、基础 Instructions、Side Panel 入口、Conversation Starter 和可用 Extension。场景配置再决定本轮加载什么 Capability Prompt、Context 和工具：

```json
{
  "scenarioId": "mailbox_organization",
  "scenarioVersion": "mailbox-organization-v3",
  "instructionVersion": "prompt-v27",
  "contextVersion": "side-panel-context-v12",
  "allowedExtensions": [
    "search_outlook_context",
    "read_outlook_message",
    "move_messages",
    "archive_messages",
    "flag_messages"
  ]
}
```

平台负责执行模型轮次和工具调度；Extension 负责权限、业务规则、资源版本、幂等和实际执行。项目配置只决定当前 Agent 能看到哪些工具以及应怎样使用稳定契约。

## 能力路由

同一个 Agent 中，路由先使用确定性产品信号，再让模型处理开放表达：

```text
显式入口
├─ 选区“解释” → 划词解释
└─ 附件“总结” → 附件总结

普通 Side Panel 输入
→ 当前对象与 Utterance
→ 直接回答 / 邮件检索 / 邮箱整理
```

路由优先级是：

1. **显式入口**：用户从选区或附件按钮进入时，直接选择对应 Capability；
2. **界面 Anchor**：当前 Message、Attachment 或 Folder 限定可用 Context；
3. **用户 Utterance**：模型判断是回答当前内容、跨邮箱查找还是提出写操作；
4. **风险与信息缺口**：读取可以按明确 Anchor 继续，写入缺少对象、范围或动作时必须澄清。

任务切换时，不把上一 Capability 的工具结果继续带入新任务。当前邮件等仍有效 Anchor 可以保留，附件片段、旧搜索候选和未确认计划需要失效或重新获取。

如果用户说“帮我处理一下这些邮件”，动作可能是总结、移动或加旗标，Agent 不会猜一个写操作，而是只追问会改变执行路径的信息。高频且入口明确的能力由客户端确定性路由，模型主要处理普通 Side Panel 中的开放表达和长尾组合任务。

工具集合也参与路由。划词解释不暴露邮箱搜索和写工具；附件总结只暴露内容提取与引用能力；邮箱整理才暴露搜索、读取和写入工具。这样既减少工具选择噪声，也把权限面限制在当前任务所需范围。

## 请求与执行循环

Outlook 客户端提供用户消息、入口类型，以及当前 Message ID、Conversation ID、Attachment ID、Folder ID 或选区。客户端只传对象标识和必要选区，不预加载整个邮箱或全部附件。

```text
Agent 基础 Instructions
+ Capability Prompt
+ 当前用户消息
+ 平台会话中的有效任务状态
+ Outlook 界面 Context
+ 当前场景允许的工具 Schema
```

模型需要更多事实时生成 Tool Call。平台负责 Schema 校验、用户身份、Extension 调度和结果回传；模型根据 Tool Result 决定继续、改写查询、读取详情、澄清或结束。

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

Tool Result 使用稳定状态而不是把内部异常直接交给模型。Instructions 明确 `partial`、`permission_denied`、`version_conflict`、`result_unknown` 和 `partial_success` 的行为，详细契约见[工具错误与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)。

## 多轮状态

完整 Conversation 和计划由平台管理，项目不实现会话数据库。项目控制进入本轮 Context 的信息，并区分三层：

- **会话原文**：近期用户输入和 Agent 回答，用于理解指代与任务切换；
- **任务状态**：当前目标、用户约束、资源 ID、计划和已完成步骤；
- **外部业务数据**：邮件、附件和 Folder 继续保存在 Outlook，需要时通过工具重读。

当前 Message、Attachment 和 Folder Anchor 随 Side Panel 变化更新。用户明确确认的时间、人员、动作和排除项可在同一任务中继承；旧搜索结果、资源版本和权限结果不能长期复用。用户改变动作或范围后，旧计划和确认失效。

这个项目没有跨任务长期记忆。邮件事实以 Outlook 为准，写操作前必须重新读取最新状态，不能依赖模型对旧 Tool Result 的记忆。

## Planning 与 Replanning

邮箱整理使用平台已有的 Planning 和确认能力。项目负责从用户目标提取业务约束、配置探索阶段的只读工具、提供计划所需的 Message ID 与动作信息，并定义错误后的行为；平台负责 Plan Version、确认状态和写工具调度。

```text
模糊目标
→ 最小澄清
→ 搜索候选
→ 按需读取
→ 形成可检查的动作范围
→ 平台确认
→ 写工具执行
→ 部分成功或 Replanning
```

用户修改时间、排除项或目标 Folder 时，旧计划不能继续使用；资源发生变化时，冲突项需要重新读取，动作范围变化后重新确认。完整链路见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)。

## 终止与循环控制

模型不能因为仍可调用工具就无限探索。每个 Capability 都定义完成条件：划词解释在给出当前选区含义后结束；附件总结在覆盖所选附件并保留 Citation 后结束；邮箱整理在用户取消、只读结果返回、已确认动作完成或无法安全继续时结束。

平台提供轮次、时间和 Token 预算，项目在 Instructions 中补充业务终止规则：

- 已有证据足够时不重复搜索或读取；
- 连续工具结果没有新增信息时停止并说明限制；
- 相同参数错误不反复调用；
- `result_unknown` 不创建新的写操作；
- 达到预算时返回已完成步骤、当前证据和未完成部分。

这能区分“工具暂时失败”“任务无法完成”和“Agent 没有停止”，也便于在 Trace 中定位无效循环的第一步。

## 三个场景

| 场景 | 主要输入 | 执行形态 | 风险边界 |
| --- | --- | --- | --- |
| 划词解释 | 选区和当前 Message | 固定 Prompt、直接生成 | 只读，不加载写工具 |
| 附件总结 | Attachment ID 和提取内容 | 固定提取链路后生成 | 只读，内容不完整时返回 partial |
| 邮箱整理 | Folder、用户目标和任务状态 | 动态搜索、Planning 与工具执行 | 写入前由平台按风险策略确认 |

三种形态共享同一个 Agent，但不会共享不必要的工具和过期 Context。

## Trace

Microsoft 365 Copilot 平台提供 Conversation、Model Turn、Planning、Confirmation 和 Tool Call 的基础 Trace：

```text
Copilot Run
→ Agent Definition / Scenario / Context Version
→ Model Turn
→ Tool Call ID / Extension Version
→ Tool Result
→ Platform Plan / Confirmation
→ Final Answer
```

排障关注模型本轮看到的 Context、第一次工具决策、Tool Result 是否被正确使用、计划范围是否符合用户目标，以及最终回答是否忠实于证据。完整邮件、附件、选区和用户 Token 不进入项目日志。

一次工具误选问题如何通过 Trace 定位并回滚，见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)；发布与观测见[发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)。
