---
layout: default
title: 工作流概括与生成
parent: Microsoft Purview 工作流
grand_parent: 工作经历
nav_order: 5
permalink: /docs/career/purview-workflow/model-integration/
---

# 工作流概括与生成

## 接入背景

2023 年初，我们开始把大模型接入 Purview Workflow。最先实现的不是在工作流中增加一个通用模型 Action，而是围绕 Workflow Definition 提供两项产品能力：

1. 根据已有 Workflow Definition，概括这个工作流的用途和主要流程；
2. 根据用户用自然语言描述的需求，生成一份可以继续编辑和发布的 Workflow Definition。

这两个功能都使用模型理解或生成工作流，但执行路径仍由程序固定。后端准备输入、调用模型、验证结果并返回前端，模型不会在运行时自主选择工具或推进节点。

## 模型与 API

模型通过 **Azure OpenAI Service** 接入，使用部署在微软 Azure 订阅中的 **GPT-3.5 Turbo（`gpt-35-turbo`）**。Scala 后端没有让前端直接访问模型，而是通过 HTTP Client 调用 Azure OpenAI 的 **Chat Completions REST API**：

```text
POST https://{resource}.openai.azure.com/
     openai/deployments/{deployment}/chat/completions
     ?api-version=2023-05-15
```

请求使用部署名而不是直接传模型名。服务通过 Microsoft Entra ID 的托管身份取得访问令牌，模型 Endpoint、Deployment 和 API Version 由区域配置管理，不在 Workflow Definition 或前端保存凭据。

一次请求包含：

- `system` 消息：定义概括或生成任务、支持边界和输出要求；
- `user` 消息：清理后的 Workflow Definition，或者用户需求与 Action Catalog；
- `temperature`：设置为 **0.1**，减少结构和措辞的随机变化；
- `max_tokens`：概括请求为 **800**，Definition 生成请求为 **3000**。

概括接口读取模型返回的文本；生成接口要求只返回 Workflow Definition JSON。后端将完整响应严格解析为 JSON，再执行 Definition Validator。API 调用设置 **30 秒** Deadline，只对限流和临时网络错误退避重试 **2 次**；模型输出不合法时直接返回验证错误，不再次调用模型修复。

当时使用的 Chat Completions API 没有 JSON Mode 或 Schema 结构化输出，因此无法从模型侧保证每次响应都是 JSON。我们通过四层措施控制结果：

1. System Prompt 明确要求只返回一个 JSON 对象，禁止 Markdown 代码块、解释和前后缀文本；
2. Few-shot 的 `assistant` 消息只包含合法的 Workflow Definition JSON；
3. `temperature` 设置为 **0.1**，减少输出格式波动；
4. 后端对完整响应执行严格 JSON 解析，不从自然语言中猜测或截取局部 JSON。

解析失败的响应不会进入 Workflow Definition，也不会自动再次调用模型。前端显示生成失败，用户可以重新生成。解析成功后还要经过 Definition Validator，因此系统真正保证的是：**所有被接受和展示的生成结果都是合法 JSON，并且符合 Workflow Definition 约束；模型的原始输出本身并没有绝对保证。**

```text
前端请求
→ Scala Workflow Service
→ Prompt 与 Context 组装
→ Azure OpenAI Chat Completions API
→ 文本或 Definition JSON
→ 后端解析与验证
→ 前端展示
```

模型调用的输入输出、Token、延迟、模型 Deployment、Prompt Version 和验证错误都会记录到 Trace；敏感 Parameter 在进入 Prompt 和日志前过滤。

## 模型成本

选型时对比了 Azure OpenAI 中的 GPT-3.5 Turbo 和 GPT-4。GPT-3.5 Turbo 的输入和输出价格分别是 **$0.0015 / 1K Token** 和 **$0.002 / 1K Token**；GPT-4 8K 分别是 **$0.03 / 1K Token** 和 **$0.06 / 1K Token**。

单次请求成本按下面的公式计算：

```text
输入 Token ÷ 1000 × 输入单价
+ 输出 Token ÷ 1000 × 输出单价
```

Workflow 概括平均使用 2500 个输入 Token 和 400 个输出 Token，Workflow 生成平均使用 4500 个输入 Token 和 1800 个输出 Token。计算后，GPT-3.5 Turbo 的单次成本分别是 **$0.00455** 和 **$0.01035**，GPT-4 分别是 **$0.099** 和 **$0.243**。

上线后的日调用量为 **2 万次概括和 5000 次生成**，月成本按下面的公式计算：

```text
（每日概括量 × 概括单价 + 每日生成量 × 生成单价）× 30
```

GPT-3.5 Turbo 的月费用是 **$4,282.5**；相同流量全部使用 GPT-4，月费用是 **$95,850**。GPT-4 的整体成本是 GPT-3.5 Turbo 的 **22 倍**。

成本不是唯一标准。我们用一组真实 Workflow Definition 和自然语言需求比较概括完整性、Action 选择、Parameter 正确率、`runAfter` 正确率和最终 Validator 通过率。GPT-4 在复杂生成上更稳定，但 GPT-3.5 Turbo 配合 Action Catalog、Few-shot 和后端 Validator 后已经达到上线要求。最终全量使用 GPT-3.5 Turbo，并把 GPT-4 保留为离线对比模型，没有在生产请求中按失败自动升级，避免成本和输出行为不可预测。

模型费用通过调用次数、输入 Token、输出 Token、网络重试率和 Deployment 维度监控。Prompt 优化优先删除无关 Definition 字段、只提供当前可用的 Action Schema，并限制概括和生成的最大输出，不能为了降 Token 而删除 `runAfter`、Parameter 或 Expression 等决定正确性的内容。

## Prompt 设计与 Few-shot

概括和生成使用两套独立 Prompt。它们共用 Workflow 术语和数据过滤规则，但任务目标、输出格式和 Few-shot 不混在一起。

### 概括 Prompt

概括 Prompt 由三部分组成：

1. **任务和边界**：说明输入是 Purview Workflow Definition，只能依据定义解释，不推测不存在的审批人、通知对象或执行结果；
2. **Definition 内容**：后端删除内部 ID、连接信息和无关元数据，保留触发方式、Action、关键 Parameter、`runAfter` 和 Expression；
3. **输出结构**：要求依次说明工作流用途、触发条件、主要步骤、条件分支和最终结果，控制在 300～500 字。

Prompt 不直接让模型阅读完整生产 JSON。后端先把 Definition 转换成稳定结构，避免字段顺序、内部元数据和长 Parameter 干扰概括结果。

### 生成 Prompt

生成 Prompt 按固定顺序组装：

```text
System 任务与禁止事项
→ Workflow Definition JSON 规则
→ 当前 Action Catalog 与 Parameter Schema
→ Expression 和 runAfter 规则
→ Few-shot
→ 用户需求
→ 只返回 JSON 的输出要求
```

System 消息明确要求只能使用 Action Catalog 中的类型，不能创造 Action 或 Parameter，不能输出 Markdown 和解释文字。Action Catalog 由后端根据当前版本和用户权限动态生成，因此 Prompt 中暴露的能力与后端 Validator 保持一致。

### Few-shot 设计

Few-shot 以完整的 `user` 和 `assistant` 消息对放在规则之后、当前用户需求之前。`user` 消息是一条脱敏后的工作流需求，`assistant` 消息是能够通过 Validator 的 Workflow Definition JSON：

```text
system：任务、Action Catalog 和 Definition 规则
user：样例需求
assistant：样例 Workflow Definition JSON
user：当前用户需求
```

这样模型看到的不只是格式说明，还能学习用户语言怎样映射到 Action、Parameter 和 `runAfter`。Prompt 只保留少量具有代表性的样例，样例与 Prompt 一起版本化；Action 或字段变化通过更新 Action Catalog 处理，不为每个失败不断追加 Few-shot。

### 迭代与评测

Prompt、Few-shot 和 Action Catalog 分别版本化，每次调用在 Trace 中记录 Prompt Version。修改前后使用同一组概括和生成任务比较：

- 概括是否覆盖触发条件、主要 Action、分支和最终结果；
- 生成结果的 JSON 解析成功率；
- Action 与 Parameter 正确率；
- `runAfter` 和 Expression 正确率；
- Validator 通过率与用户重新生成率；
- Token、延迟和单次成本。

出现失败时先判断是模型没有理解任务、Prompt 缺少边界、Action Catalog 信息不足，还是 Validator 规则本身不清楚。只有能够代表一类结构问题的失败才进入 Few-shot；单个长尾问题优先通过规则、Schema 或后端验证解决。

## 概括已有工作流

复杂 Workflow Definition 是分层 JSON，Action 之间还通过 `runAfter` 表达依赖。用户直接阅读定义，很难快速判断工作流何时触发、包含哪些步骤，以及不同条件会进入什么分支。

概括功能把 Definition 转换成适合模型理解的结构，保留：

- Workflow 的触发方式；
- Action 类型和显示名称；
- `runAfter` 依赖；
- 影响流程含义的 Parameter；
- Expression 表达的条件和动态数据来源。

模型生成一段面向用户的说明，概括工作流解决什么问题、主要步骤和关键分支。Prompt 要求只依据传入的 Definition，不补充不存在的 Action、审批人或业务结果。

输入中不包含运行实例历史，也不会把密钥、Token 或内部连接信息交给模型。后端在组装模型输入时只保留解释流程所需的 Definition 字段，并对敏感 Parameter 做过滤。

## 根据需求生成工作流

用户可以用自然语言描述目标，例如要求在数据访问申请提交后完成审批、通知和最终状态更新。后端除了用户需求，还会向模型提供当前支持的 Action Catalog：

- Action 类型和用途；
- 必填与可选 Parameter；
- 哪些 Parameter 支持 Expression；
- `runAfter` 的结构规则；
- 少量有效 Workflow Definition 示例。

模型输出结构化的 Workflow Definition，而不是一段操作说明：

```text
用户需求
→ Action Catalog 与 Definition 规则
→ 模型生成 Workflow Definition JSON
→ 后端验证
→ 前端预览和编辑
→ 用户确认后保存或发布
```

模型只使用系统已经支持的 Action，不能通过生成新的类型扩展后端能力。需要动态值时，它可以在允许的 Parameter 中生成 Expression，但 Expression 仍要通过现有 Parser 和 Validator。

生成结果经过与普通 Workflow 相同的后端校验。合法结果进入前端预览和编辑；不合法时直接显示错误，不再次调用模型修复。模型没有发布权限，用户确认后才能保存或发布 Definition。

Expression 项目为这项能力提供了基础。模型生成动态 Parameter 时继续使用已经上线的 Expression 语法和验证链路。

## 功能边界

工作流概括可能遗漏细节，生成结果也可能语法正确但不符合用户真实意图，因此两项能力都保留人工确认：

- 概括用于帮助理解，不替代 Workflow Definition 本身；
- 生成用于创建可编辑草稿，不直接发布；
- 权限、Action Schema、Expression 和依赖关系始终由后端验证；
- 工作流运行仍由 Logic Apps 按确定性的 Definition 执行。

这段经历让我第一次把大模型用于已有企业产品。核心工作不是完成一次模型 API 调用，而是让不稳定的模型输出进入现有的 Definition、验证、权限和发布体系。它仍然属于固定的大模型应用流程，并不是 Agent。
