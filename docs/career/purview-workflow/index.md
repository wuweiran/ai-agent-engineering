---
layout: default
title: Microsoft Purview 工作流
parent: 工作经历
nav_order: 1
has_children: true
permalink: /docs/career/purview-workflow/
---

# Microsoft Purview 工作流

## 项目介绍

Microsoft Purview 是面向企业数据治理的产品，这个项目为数据资产提供审批、通知和自动处理工作流，底层流程调度由 Azure Logic Apps 完成，团队负责上层领域模型、Action 和产品能力。我参与工作流的全栈开发，主要担任 **Expression 领域 Owner**。过去每项客户定制都要开发新的 Action；我从零完成 Expression 的[设计、解析、验证和运行时求值]({{ site.baseurl }}/docs/career/purview-workflow/expression/#expression-core-implementation)，并将它接入 Workflow Definition 和 Custom Action，使已有函数可以直接组合，新领域函数通常约一周交付，而完整 Action 需要四到八周。具体复用方式见[交付周期对比]({{ site.baseurl }}/docs/career/purview-workflow/expression/#expression-reuse-delivery)。

2023 年，我还参与将 LLM 接入现有工作流，落地 Definition 的自然语言概括和生成。模型只生成说明或可编辑草稿，Action、Parameter、依赖关系和 Expression 仍由后端校验，用户确认后才能保存或发布。这个项目的技术主线是**可定制工作流能力**和[确定性业务系统中的 LLM 接入]({{ site.baseurl }}/docs/career/purview-workflow/model-integration/#workflow-llm-integration)，不是 Agent Runtime。

## 系统处在什么位置

团队不负责底层工作流引擎。工作流的实例执行、节点调度和依赖推进由 **Azure Logic Apps** 提供；我们负责其上的 Purview 产品层和集成层，包括 Workflow API、Workflow Definition、Purview Action、Expression，以及对 Logic Apps 运行实例的简单包装。

Purview Catalog 需要启动业务流程时调用我们的接口。Logic Apps 根据 Workflow Definition 调度节点；其中大部分节点配置为 HTTP Action，调用 Purview Action API 并传入 Action 类型和参数。Purview 执行完成后回调 Logic Apps，最后再由固定 Action 调用 Catalog 接口，完成上游业务闭环。

```text
Purview Catalog
→ Purview Workflow API
→ Azure Logic Apps 调度 HTTP Action
→ Purview Action API 执行业务
→ Purview 回调 Logic Apps 报告结果
→ Logic Apps 继续调度
→ 固定完成 Action 回调 Catalog
```

后端采用 **Scala + ZIO**。Workflow Definition 参考 Logic Apps 的 JSON 层级结构，并通过 `runAfter` 串联 Action。完整边界见 [工作流系统设计]({{ site.baseurl }}/docs/career/purview-workflow/system-design/)。

## Expression 核心能力

Expression 是符合自定义语法的单行字符串，作为指定 Workflow Action 的某个 Parameter 值保存在 Workflow Definition 中。哪些 Parameter 支持 Expression 由系统定义。它可以引用工作流的触发输入和前序节点输出，也可以通过 `getUserName`、`getManager` 等后端注册函数取得外部数据。Logic Apps 执行到相应 Action 时，Purview Action 执行层对支持 Expression 的 Parameter 求值，再使用实际参数完成任务。

完整的语言形式、系统位置和核心实现见 [Workflow Expression]({{ site.baseurl }}/docs/career/purview-workflow/expression/)。

## 开始接触大模型

2023 年初，团队开始把 OpenAI 接入 Purview Workflow，主要实现两项能力：根据已有 Workflow Definition 概括工作流的用途和主要步骤，以及根据用户的自然语言需求生成 Workflow Definition 草稿。

模型生成的 Definition 必须通过 Action、Parameter、`runAfter` 和 Expression 验证，并由用户预览确认后才能保存或发布。完整链路见 [工作流概括与生成]({{ site.baseurl }}/docs/career/purview-workflow/model-integration/)。

这个阶段最重要的认识是：调用模型不等于 Agent。输入输出和调用步骤由程序确定，模型负责理解或生成 Definition，后端继续控制验证、权限和发布。

## 这个项目可以深入到哪里

- Workflow Expression 的形式、语法解析、验证和后端求值；
- Expression 怎样缩短定制需求交付，并支撑后来的 Custom Action；
- Logic Apps 的 `runAfter`、节点状态、调度、执行权和实例包装；
- 节点卡住怎样排查，以及超时、重试、幂等、结果未知和恢复怎样处理；
- Purview Action 与 Catalog 回调怎样形成可靠的业务闭环；
- Workflow、单次模型调用和 Agent 的区别；
- 怎样在已经上线的业务流程中接入模型；
- 哪些步骤适合交给模型，哪些规则和状态继续由程序控制。

这个项目既体现了从零负责一项平台能力的经历，也为后续 Agent 工程提供了后端基础。它还能够具体说明：**系统使用了大模型，不代表它就是 Agent。**

## 项目文档

- [Workflow Expression]({{ site.baseurl }}/docs/career/purview-workflow/expression/)：形式定义、系统位置、语法解析、验证、求值和扩展边界；
- [工作流系统设计]({{ site.baseurl }}/docs/career/purview-workflow/system-design/)：Scala + ZIO 服务、Logic Apps、Workflow Definition、Catalog 集成与实例包装；
- [工作流运行与可靠性]({{ site.baseurl }}/docs/career/purview-workflow/reliability/)：端到端节点状态、调度、执行权、超时、重试、幂等、故障恢复和 Catalog 闭环；
- [部署与生产运行]({{ site.baseurl }}/docs/career/purview-workflow/production/)：服务拓扑、计算资源、存储分区、容量、监控和值班事故；
- [工作流概括与生成]({{ site.baseurl }}/docs/career/purview-workflow/model-integration/)：Definition 概括、自然语言生成、结构验证和用户确认。

项目内容增加后，还可以继续拆分：

- 业务与团队：Purview 数据治理场景、工作流用户和团队分工。
