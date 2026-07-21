---
layout: default
title: Prompt 与 Context Engineering
parent: Outlook Copilot Agent 能力开发
grand_parent: 工作经历
nav_order: 2
permalink: /docs/career/copilot-capabilities/prompt-context/
---

# Prompt 与 Context Engineering

## Prompt 的分层

Outlook Copilot 的 Prompt 不是一份不断追加规则的长文本。Declarative Agent 平台结合 Agent Definition 和项目场景配置组装三层内容：

1. **Agent 基础规则**：身份、回答原则、数据边界、工具使用方式和安全要求，所有任务共用；
2. **Capability Prompt**：说明当前任务的目标、可用 Context、允许的工具、完成条件和输出要求；
3. **本轮任务信息**：用户消息、Side Panel 入口、当前 Outlook 对象和必要的会话历史。

```text
稳定的 Agent 基础规则
+ Capability Prompt
+ 当前用户任务
+ 当前允许的工具 Schema
+ 本轮 Context
```

基础规则和 Capability Prompt 分别版本化。修改某个任务时不改全局 Prompt，避免划词解释的规则影响附件总结，或者邮箱整理的写操作要求干扰普通问答。

## Capability Prompt

三个场景使用不同的任务约束，但共享同一个 Declarative Agent 和平台执行循环。

| 场景 | Prompt 重点 | 完成条件 |
| --- | --- | --- |
| 划词解释 | 解释选区在当前邮件中的含义，保留原语言和必要上下文，不扩展成整封邮件总结 | 给出简洁解释；信息不足时说明依赖的上下文 |
| 附件总结 | 只依据提取出的附件内容，覆盖主题、结论和行动项，保留 Citation | 总结已选附件；格式不支持或提取不完整时明确说明 |
| 邮箱整理 | 将模糊目标转成可检查的筛选条件和动作计划，写操作前必须展示范围并确认 | 计划得到确认并执行，或者明确未完成和失败项 |

固定入口可以直接选择 Capability。划词解释入口携带选区并加载对应 Prompt；普通 Side Panel 请求则由模型结合用户表达和界面 Context 选择直接回答或调用工具。

Prompt 只描述模型应做的判断，权限、对象版本、幂等和用户确认由 Copilot 平台与已有 Extension 执行。即使 Prompt 写明“不要移动未确认的邮件”，写工具仍然必须遵守平台确认和自身权限检查。

## 工具描述也是 Prompt

模型能否选择正确工具，不只取决于 System Prompt，还取决于工具名称、Description 和 Schema。在 Microsoft 365 Copilot 平台中，这些工具以 Extension 形式注册。

工具 Description 明确三件事：

- **何时使用**：当前证据不足，需要搜索邮件或读取附件；
- **返回什么**：轻量候选、完整邮件、附件文本或动作结果；
- **不能替代什么**：Query 工具不返回完整邮件，读取工具不执行写操作。

相近能力不使用只有名词差异的描述。`search_outlook_context` 负责开放检索，`read_outlook_message` 读取已知邮件，`move_messages` 执行已确认的移动计划。Schema 只保留模型能够判断的业务参数，用户身份、租户、权限和服务端上限不暴露给模型填写。

## Context 的组成

项目提供 Context Builder，将场景 Context 和结构化任务状态交给 Declarative Agent 平台，平台再组装模型本轮需要的信息：

```text
System 与 Capability 规则
→ 当前用户消息
→ 最近会话历史与已确认任务状态
→ Outlook 界面 Context
→ 当前允许的工具 Schema
→ 必要的 Tool Result
→ 输出 Token 预留
```

Context 来源分成不同可信级别：

- **受信任规则**：System Prompt、Capability 配置、确认策略；
- **受信任元数据**：登录身份、Message ID、Attachment ID、Folder ID 和资源版本；
- **用户输入**：当前请求和用户确认；
- **不可信内容**：邮件正文、附件文本、搜索 Snippet 和外部链接。

邮件和附件即使由用户有权读取，也不能被当成系统指令。Context Builder 用明确边界标记这些内容，Instructions 要求模型将其视为数据；平台工具白名单与已有 Extension 不允许正文中的文字改变工具权限或绕过确认。

## 界面 Context

Side Panel 的价值在于客户端能够提供用户当前正在操作的 Outlook 对象，但不同任务只取必要部分。

### 划词解释

Context 包含：

- 选中文字；
- 选区附近的有界上下文，按句子和段落边界截取；
- 邮件主题、发送者、时间和语言；
- 当前 Message ID，用于 Citation 和必要时读取详情。

不默认放入整个 Conversation。选区已经能够独立解释时，前后文进一步缩小，降低噪声和 Token。

### 附件总结

客户端只传 Attachment ID 和当前 Message ID。附件正文由内容提取工具读取，Tool Result 返回按页或章节组织的文本、位置、提取状态和 Citation。短附件一次进入 Context；长附件按结构分段，只选择当前任务需要的片段，并保留来源位置。跨段总结先按来源整理主题、结论和行动项，再合并去重；最终 Citation 仍指向原始页或章节，不能只引用中间摘要。原始二进制和全部 OCR 中间结果不进入模型 Context。

### 邮箱整理

初始 Context 包含当前 Folder ID、用户目标和已有会话状态，不预加载文件夹中的邮件。模型先调用 Insight Query 取得候选，再按需读取详情。动作计划以结构化状态保存邮件 ID、版本和拟执行动作，不依赖模型从长对话中重新回忆。

## Token 预算

模型的总 Context Window 由 Microsoft 365 Copilot 平台和模型部署决定。项目不控制平台窗口大小，但会在 Context Builder 中为规则、任务状态、界面 Context、工具 Schema 和证据设置相对优先级，防止单一来源占满输入。

证据占主要预算，因为回答必须建立在邮件或附件内容上；Agent 基础规则、当前目标和已确认约束优先保留；较早闲聊、重复引用和被新结果替代的 Tool Result 优先删除。具体阈值随模型部署和 Capability 版本调整，不把某个固定 Token 数量当成跨版本不变的产品约束。

超过预算时按以下顺序处理：

1. 删除已被新结果替代的 Tool Result；
2. 对同一 Conversation、附件段落和 Citation 去重；
3. 保留近期原始消息，将较早会话压缩成结构化任务状态；
4. 按相关性和来源多样性选择证据；
5. 截断单条过长内容，并设置 `truncated`；
6. 仍不足时返回 `partial`，不把证据缺失包装成完整答案。

## Context 生命周期

完整数据不会只保存在模型 Context 中。平台会话、项目结构化状态和 Outlook 业务数据分成三层：

- **会话窗口**：保留近期且与当前任务相关的用户与模型原文，以及当前仍在使用的 Tool Call 和 Tool Result；
- **结构化任务状态**：保存目标、用户约束、已确认事实、资源 ID、资源版本、动作计划、已完成步骤和待办；
- **外部业务数据**：邮件、Conversation、附件和文件夹继续保存在 Outlook 或附件存储中，需要时通过工具重新读取。

```text
完整会话与业务对象
→ 存在 Context 之外

本轮模型调用
→ 最近原文
+ 结构化任务状态
+ 当前步骤需要的外部证据
```

Tool Result 有生命周期。搜索候选被邮件详情替代后，旧候选列表从下一轮 Context 删除；附件内容完成总结后，只保留 Citation 和当前任务仍需要的片段；写操作完成后保留 Operation ID、资源 ID 和结果，不继续携带完整请求与响应。

滑动窗口只负责限制近期消息数量，不能保存早期目标。结构化任务状态保留目标、约束和执行事实，但不替代原始邮件证据。需要核对事实或执行写操作时，Agent 调用已有读取工具取得最新数据，而不是依赖有损摘要或旧 Tool Result。

较早会话压缩时只提取显式事实和决定，不生成新的业务结论。压缩结果带 Summary Version 和来源消息范围；用户后续修改约束时，结构化状态覆盖旧值，并记录新的确认版本。

## Context 排序

Context 顺序固定，避免重要信息被长证据淹没：

```text
System 与 Capability 规则
→ 当前用户目标和完成条件
→ 已确认任务状态
→ 工具 Schema
→ 按来源分组的邮件或附件证据
→ 最近会话历史
```

同一来源的证据保持时间顺序，Citation 紧跟对应内容。最重要的邮件不会全部堆在 Context 开头或末尾，而是按主题分组并在组内排序，减少长 Context 中间信息难以被利用的问题。

## 三个具体优化决策

### 防止规则跨场景污染

Agent 基础规则只保存跨场景约束，任务目标和工具触发条件下沉到 Capability Prompt。这样改善邮箱检索的规则不会改变划词解释和附件总结。一次全局搜索规则导致简单场景误调用工具的问题及修复见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。

### 提高证据密度

Context Builder 利用 Side Panel Anchor 缩小范围：划词解释只取选区附近内容，附件总结按 Attachment ID 提取，邮箱整理通过 Insight Query 找候选而不预加载文件夹。搜索候选被邮件详情替代后从后续 Context 移除；重复正文、签名、免责声明和重复附件片段先去重；写操作前重新读取资源版本。

### 保持来源边界

附件和邮件证据按来源分组，Citation 紧跟对应内容；Tool Result 保留 `partial` 和 `truncated`，防止模型把不完整证据表述成完整结论。邮件正文、附件文本和搜索 Snippet 都标记为不可信数据，不能覆盖 Agent Instructions 或改变工具权限。

优化不能只看 Token 减少。删除过多证据可能降低任务完成率，压缩历史可能丢失用户约束，减少工具 Schema 可能导致模型找不到正确工具。因此每次只改变 Prompt、Context Builder 或工具定义中的一层，并通过相同 Query 对比失败类型，而不是只比较平均 Token。

## 版本、评测与失败归因

Prompt、Context Builder 和工具 Schema 分别版本化，并与 Agent Definition、Extension Version 和 Model Deployment 一起写入平台 Trace。出现问题时从第一次偏离归因：模型没理解任务检查 Capability Prompt；缺少证据或噪声过多检查 Context Builder；工具选错检查候选工具、名称和 Description；参数正确但执行失败检查 Tool Result 与依赖 Extension。

2025 年 6 月至 10 月，Agent 开发与 Evaluation 工作重叠。真实失败模式会进入 Golden Set：固定 Query 验证工具和答案，多轮 Scenario 验证任务切换、指代和约束更新。修复后先跑场景切片，再跑全量 Golden Set，避免局部 Prompt 或 Context 改动影响其他 Capability。完整评测体系见[Copilot Evaluation 与 Golden Set]({{ site.baseurl }}/docs/career/copilot-evaluation/)。
