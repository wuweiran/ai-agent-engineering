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

## 请求执行

客户端只提供当前任务需要的对象标识和少量界面内容，不预加载整个邮箱或全部附件。模型需要更多信息时生成 Tool Call，平台附加用户身份并调用 Extension，Tool Result 再返回模型。

邮箱整理第一轮只使用搜索和读取。Insight Query 最多返回 Top 12 候选，模型只对需要判断行动项、回复状态或归属的邮件读取详情。形成目标范围后，平台展示计划并确认，再调用移动、归档或加旗标工具。

完整的邮箱整理流程见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)。

## 多轮状态

Conversation 和计划由平台保存。项目只规定哪些业务信息可以继续使用：用户已经确认的时间、人员、Folder 和排除项可以在同一任务中保留；搜索候选、资源版本和权限结果需要按当前步骤重新取得。用户修改动作或范围后，旧计划失效并重新确认。

邮件和附件仍以 Outlook 中的数据为准，项目没有跨任务长期记忆，也不会把旧 Tool Result 当成最新业务状态。

## 问题定位

平台 Trace 可以看到入口、当前 Context、模型轮次、Tool Call、Tool Result、计划确认和最终回答。排查时从 Run ID 开始找第一次偏离：如果模型首轮就选错工具，检查场景 Prompt 和工具集合；如果工具选择正确但执行失败，交给对应 Extension 团队继续定位。

项目日志不保存完整邮件、附件、选区或用户 Token。一次工具误选的处理见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。
