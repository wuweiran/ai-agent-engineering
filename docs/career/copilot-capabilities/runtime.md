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

Outlook Copilot 使用 Microsoft 365 Copilot Declarative Agent。平台负责模型调用、Conversation、Tool Call 循环、Planning、用户确认和流式输出；已有 Extension 负责附件提取、邮件搜索、邮件读写以及底层权限和幂等。

我的代码不包含一套独立运行的 Agent Runtime，主要是两部分：

- Outlook Copilot 仓库中的 Agent Definition、Instructions、Capability 配置和 Extension 声明；
- Outlook 客户端或接入层中的 Side Panel Context 代码，用来提供当前 Message、选区、Attachment、Folder 和入口类型。

这些配置构建成带版本的 Agent 发布包，由 Microsoft 365 Copilot 平台加载。各 Extension 的 Handler 仍运行在对应团队的后端服务中。

```text
Outlook Side Panel
→ Agent Definition、Instructions 和 Context
→ Microsoft 365 Copilot 平台
→ 已有 Extension
→ Tool Result
→ 最终回答或下一步操作
```

## 代码以什么形式存在

Agent 仓库中通常包含 Microsoft 365 应用 Manifest、Declarative Agent 声明、Instructions 文件和 Extension 声明。内部文件名会随仓库模板变化，但产物只有三个作用：注册 Agent、描述模型行为、声明可以调用哪些已有工具。

```text
Agent 源码与配置
→ 构建和校验
→ 版本化 Agent 包
→ Microsoft 365 Copilot 平台
```

因此，“Agent 代码运行在哪里”的准确回答是：行为配置由平台托管执行，Side Panel Context 代码运行在 Outlook 接入层，工具实现运行在各 Extension 后端；项目没有单独的 Agent 服务或容器。

## 三项能力怎样接入

划词解释、附件总结和邮箱整理共用同一个 Agent，因为它们都从 Outlook Side Panel 进入，并共享用户身份和当前邮件 Context。实现上没有让所有请求都走同一条路径：

| 场景 | 输入 | 可用工具 | 执行方式 |
| --- | --- | --- | --- |
| 划词解释 | 选区、当前邮件元数据 | 无写工具 | 固定 Prompt 直接生成 |
| 附件总结 | Message ID、Attachment ID | 附件提取 | 取得内容后生成总结 |
| 邮箱整理 | Folder、用户目标 | 搜索、读取、移动、归档、加旗标 | 根据 Tool Result 分多轮执行 |

显式的选区和附件入口由客户端直接选择场景。普通 Side Panel 输入由模型结合用户表达和当前 Outlook 对象判断是直接回答、搜索邮件还是进入邮箱整理。划词解释不会加载邮箱搜索和写工具，避免简单请求误调用无关 Extension。

## 完整工具调用链路

工具首先以 Extension 的形式注册到 Microsoft 365 Copilot 平台，向模型暴露工具名称、Description 和参数 Schema。项目不实现这些工具的 Handler，但需要把使用边界写清楚：`search_outlook_context` 用于发现未知邮件，`read_outlook_message` 用于读取已知 Message ID，移动、归档和加旗标工具只处理已经确认的目标对象。

一次调用分成四步：

### 工具选择

客户端只提供当前任务需要的对象标识和少量界面内容，不预加载整个邮箱或全部附件。平台根据 Capability 加载当前场景的 Instructions、Context 和工具 Schema：划词解释不加载工具，附件总结只加载附件提取，邮箱整理才加载搜索、读取和写工具。

模型结合用户输入、当前 Message、Attachment 或 Folder，以及工具 Description 选择是否调用工具。这里的“模型选择”是模型从当前候选工具中选择下一步，不是项目团队选择底层模型；Model Deployment 由 Microsoft 365 Copilot 平台负责。

### 参数提取

模型按照工具 Schema 从用户输入和已确认任务状态中生成结构化参数。以“找出最近两周需要我跟进的邮件”为例，搜索参数包含查询条件、起止时间和已知参与人；当前用户、Tenant、访问令牌和服务端候选上限不让模型填写，由平台和 Extension 根据请求补充。

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

Conversation 和计划由平台保存。项目只规定哪些业务信息可以继续使用：用户已经确认的时间、人员、Folder 和排除项可以在同一任务中保留；搜索候选、资源版本和权限结果需要按当前步骤重新取得。用户修改动作或范围后，旧计划失效并重新确认。

邮件和附件仍以 Outlook 中的数据为准，项目没有跨任务长期记忆，也不会把旧 Tool Result 当成最新业务状态。

## 问题定位

平台 Trace 可以看到入口、当前 Context、模型轮次、Tool Call、Tool Result、计划确认和最终回答。排查时从 Run ID 开始找第一次偏离：如果模型首轮就选错工具，检查场景 Prompt 和工具集合；如果工具选择正确但执行失败，交给对应 Extension 团队继续定位。

项目日志不保存完整邮件、附件、选区或用户 Token。一次工具误选的处理见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。
