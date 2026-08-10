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

Outlook Copilot 通过 Sydney API 调用 BizChat 后端接口平台。Sydney Payload 中带用户输入和 Context Config，Sydney 接入层按 Config 填入界面 Context；Microsoft 365 Copilot Agent Runtime 负责模型调用、Conversation、Tool Call 循环、Planning、用户确认和流式输出。

我的代码不包含一套独立运行的 Agent Runtime，主要是两部分：

- Outlook Copilot 仓库中的 Declarative Agent Manifest、Instructions 和 API Plugin/Extension 声明；
- Sydney 请求 Payload 中的 Context Config，声明当前请求需要 Message、选区、Attachment、Folder 或入口类型。

Sydney 接入层同事负责根据 Config 采集 Context 并放入 Payload，我不实现客户端 Context 代码。Agent Definition 由 Microsoft 365 Copilot Agent Runtime 加载，Context Config 随 Sydney 请求发送；各 Extension 的 Handler 仍运行在对应团队的后端服务中。

```text
Microsoft 365 Copilot 网页或 Outlook 入口
→ Sydney Payload：用户输入 + Context Config
→ Sydney 接入层填入界面 Context
→ BizChat 后端接口平台
→ Microsoft 365 Copilot Agent Runtime
→ 已有 Extension
→ Tool Result 与最终回答
```

## Declarative Agent 怎样配置
{: #declarative-agent-configuration }

Microsoft 365 应用包引用 Declarative Agent Manifest。Manifest 定义整个 Agent，而不是分别定义三项业务场景：

```json
{
  "version": "v1.8",
  "name": "Outlook Agent",
  "description": "Help users understand and organize Outlook content.",
  "instructions": "...",
  "capabilities": [
    { "name": "Email" }
  ],
  "actions": [
    { "id": "outlookTools", "file": "outlook-plugin.json" }
  ]
}
```

- `instructions` 定义 Agent 的行为和边界；
- `capabilities` 声明 Email、People 等 Microsoft 365 内置能力；
- `actions` 引用 API Plugin，接入 Outlook Extension 工具。

API Plugin Manifest 再描述模型能够调用的 Function，以及怎样映射到后端：

```json
{
  "schema_version": "v2.4",
  "functions": [
    {
      "name": "search_outlook_context",
      "description": "Search Outlook when the requested email is not in the current context.",
      "parameters": {
        "type": "object",
        "properties": {
          "query": { "type": "string" }
        },
        "required": ["query"]
      }
    }
  ],
  "runtimes": [
    {
      "type": "OpenApi",
      "run_for_functions": ["search_outlook_context"],
      "spec": { "url": "https://.../openapi.yaml" }
    }
  ]
}
```

Copilot 的加载顺序是：

```text
Microsoft 365 应用包
→ Declarative Agent Manifest
→ Instructions、内置 Capabilities 和 Actions
→ Action 引用 API Plugin Manifest
→ Functions 提供工具描述和参数
→ Runtime 与 OpenAPI 定位后端接口
```

运行时，Microsoft 365 Copilot 再把用户输入、Sydney 注入的 Context 和本轮候选 Functions 一起交给模型。Sydney Context Config 不属于 Declarative Agent Manifest，它随每次请求提供当前界面 Context。

因此，“Agent 代码运行在哪里”的准确回答是：Agent Manifest 由 Microsoft 365 Copilot Agent Runtime 加载，Sydney Context Config 随请求发送，Context 采集代码由 Sydney 接入层维护，工具实现运行在各 Extension 后端；项目没有单独的 Agent 服务或容器。

## 三项能力怎样接入

划词解释、附件总结和邮箱整理共用同一个 Declarative Agent，它们是三个业务场景，不是 Declarative Agent Manifest 中三个独立的 `capabilities`。Sydney Context Config 只决定当前邮件、选区、附件或 Folder 等信息是否进入本轮 Context，不负责选择工具。

Agent 的 `actions` 注册全局可用工具。Microsoft 365 Copilot Runtime 将用户输入、Sydney 注入的 Context、Declarative Agent `instructions` 和工具的 Function Description 一起交给模型，由模型决定直接回答还是调用工具。当前项目没有单独维护请求级工具过滤配置。

| 场景 | Context Config | 同一个 `instructions` 字段中的条件规则 | 执行方式 |
| --- | --- | --- | --- |
| 划词解释 | 选区、当前邮件元数据、入口类型 | 有选区时直接解释，不搜索邮箱 | 直接生成 |
| 附件总结 | Message ID、Attachment ID、入口类型 | 有附件入口时先调用附件提取，再根据提取状态总结 | 取得内容后生成总结 |
| 邮箱整理 | Folder、用户目标、入口类型 | 用户要求整理邮箱时搜索和读取，写操作前必须确认 | 模型根据 Tool Result 分多轮选择 |

因此，即使请求一直带有邮件 Context，也不代表模型一定会调用邮件工具。

## 完整工具调用链路
{: #tool-calling-pipeline }

工具首先以 Extension 的形式注册到 BizChat，向模型暴露工具名称、Description 和参数 Schema。项目不实现这些工具的 Handler，但需要把使用边界写清楚：`search_outlook_context` 用于发现未知邮件，`read_outlook_message` 用于读取已知 Message ID，移动、归档和加旗标工具只处理已经确认的目标对象。

从 Trace 看，Microsoft 365 Copilot Agent Runtime 的一轮工具调用可以拆成四个阶段。这不是 Declarative Agent 中定义的固定 Workflow：项目负责配置 Actions、Function Description、参数 Schema 和 Tool Result 后的行为规则，Runtime 负责模型选择、调用调度和循环执行。

### 工具选择

Declarative Agent 的 `actions` 决定 Agent 注册哪些 API Plugin；Plugin Manifest 中的 `functions` 定义工具名称、Description 和参数 Schema。Sydney Context Config 决定 Payload 携带哪些界面 Context 和入口类型，Declarative Agent 的同一个 `instructions` 字段再按这些信息规定工具使用条件。

模型结合用户输入、当前 Message、Attachment、Folder、入口类型、`instructions` 和 Function Description，决定直接回答还是生成 Tool Call；Model Deployment 由 BizChat 负责。

### 参数提取

模型按照工具 Schema 从用户输入和已确认任务状态中生成结构化参数。以“找出最近两周需要我跟进的邮件”为例，搜索参数包含查询条件、起止时间和已知参与人；当前用户、Tenant、用户 Token 和服务端候选上限不让模型填写，由 BizChat 和 Extension 根据请求补充。

平台先做 Schema 校验，Extension 再检查参数范围、用户权限和业务对象。缺少时间可以在只读搜索中使用场景默认值；动作或目标 Folder 不明确时必须向用户澄清，不能生成写 Tool Call。

### 工具执行

参数校验通过后，平台附加当前用户身份并调用已有 Extension。Insight Query 返回最多 Top 12 轻量候选，包括 Message ID、Snippet、时间和 Citation；模型只对需要判断回复状态、行动项或归属的候选继续调用邮件读取工具。

形成目标邮件和动作范围后，平台展示计划并取得确认，再调用移动、归档或加旗标工具。写 Extension 在执行时继续校验权限和资源版本，并按邮件返回结果。

### 结果处理

Tool Result 回到同一个 Conversation，模型根据结果决定回答、继续读取、更新计划或停止：

- 搜索成功：根据候选决定是否读取详情，不重复搜索相同条件；
- `version_conflict`：重新读取变化的邮件，范围变化时更新计划并重新确认；
- `partial_success`：保留成功项，只处理失败和未知项；
- `result_unknown`：沿用原 Operation ID 查询状态，不创建新的写请求；
- `permission_denied`：停止处理该对象，不扩大搜索绕过权限。

最终回答会区分已经完成、失败和状态未知的邮件，并保留必要 Citation。平台 Trace 记录本轮 Context、候选工具、Tool Call、参数摘要、Tool Result 和最终回答，用于判断问题出在工具选择、参数生成、Extension 执行还是结果理解。

完整的邮箱整理流程见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)，错误处理见[工具错误与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)。

## 多轮状态
{: #multi-turn-state }

Conversation 和计划由平台保存。项目只规定哪些业务信息可以继续使用：用户已经确认的时间、人员、Folder 和排除项可以在同一任务中保留；搜索候选、资源版本和权限结果需要按当前步骤重新取得。用户修改动作或范围后，旧计划失效并重新确认。

邮件和附件仍以 Outlook 中的数据为准，项目没有跨任务长期记忆，也不会把旧 Tool Result 当成最新业务状态。

## 问题定位

平台 Trace 可以看到入口、当前 Context、模型轮次、Tool Call、Tool Result、计划确认和最终回答。排查时从 BizChat Conversation ID 开始找第一次偏离：如果模型首轮就选错工具，检查场景 Prompt 和工具集合；如果工具选择正确但执行失败，交给对应 Extension 团队继续定位。

项目日志不保存完整邮件、附件、选区或用户 Token。一次工具误选的处理见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。
