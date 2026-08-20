---
layout: default
title: My Outlook Agent 部署
parent: 工作经历
nav_order: 5
has_children: true
permalink: /docs/career/my-outlook-agent/
---

# My Outlook Agent 部署

## 项目介绍

My Outlook 是 Outlook 中面向工作的**个人 Agent**。它通过后台扫描邮件、日历和组织关系等 Microsoft 365 信号，识别用户近期的重点、承诺和待办，把 Briefing、Catch-up、Draft 和 Recommendation 主动呈现在 Outlook 中；用户也可以继续与 Agent 对话，让它查询信息、生成内容或执行工作任务。

这个项目与此前在 Microsoft 365 Copilot Declarative Agent 上开发场景不同：My Outlook 当时拥有**独立部署的 Agent 后端**。Outlook Client 负责交互，MyOutlook（POS）Service 承载**在线 Agent Loop 和任务编排**，AKS Worker 执行 Deep Scan、Synthesis、模型调用和 Graph/API 访问，SDS 与 Annotation Store 保存个性化产物、标注和中间状态。

我参与了 **Agent 部署与运行链路**，重点是把在线对话和后台工作拆成生命周期不同的执行单元：Service 负责低延迟接入、Agent 决策和状态推进，Worker 负责可排队、可重试、资源消耗较高的任务。我的工作包括 **Service—Worker 任务契约、队列调度、Context 组装与裁剪、任务恢复、AKS 扩缩容和生产监控**；Deep Scan 和 Synthesis 的业务算法由对应团队维护。

```text
Outlook Client
      ↓
MyOutlook (POS) Service on AKS
├─ Session / Task API
├─ Agent Loop
├─ Context 与 Tool Orchestrator
└─ Task Scheduler
      ↓
Service Bus / Task Store
      ↓
AKS Worker Pools
├─ Deep Scan Worker
├─ Synthesis Worker
├─ LLM Worker
└─ Graph / API Worker
      ↓
SDS / Annotation Store / Microsoft 365 APIs
```

## 三条技术主线

### Agent Runtime 与任务恢复

**POS Service 是在线控制面，Worker 是异步执行面。**长时间或需要等待的步骤持久化为 Task，再通过队列交给 Worker。Task、Run、Step 和 Attempt 分开保存，Worker 使用 Lease 取得推进权，副作用使用业务幂等键保证重复投递不会产生重复结果。完整设计见 [Agent Runtime 与任务恢复]({{ site.baseurl }}/docs/career/my-outlook-agent/runtime-task/)。

这条主线主要回答：为什么长任务不能绑在 HTTP 请求中、数据库与队列如何保持一致、Worker 崩溃后如何恢复、等待 `Retry-After` 时为什么不占住 Worker，以及任务如何跨版本发布继续运行。

### 动态 Context 装配与裁剪
{: #dynamic-context-assembly }

My Outlook 自己运行 Agent Loop，**Context Builder 是每次模型调用前的 Runtime 模块**。它按当前 Step 和 Token Budget，从 Conversation、Task State、Deep Scan Finding、Synthesis Artifact 和 Worker Result 中重建本轮 Model Input；**当前目标和确定状态固定保留，候选按阶段选择或替换，较早历史超预算时压缩**。完整设计见 [Context Builder 与 Token Budget]({{ site.baseurl }}/docs/career/my-outlook-agent/context-builder/)。

这条主线主要回答：Context 与任务状态为什么必须分开、长对话如何裁剪、Artifact 为什么只以 Reference 进入模型、Token Budget 怎样分配，以及如何证明节省 Token 没有降低任务完成率。

### AKS 部署与生产运行

POS Service、Deep Scan、Synthesis、LLM 和 Graph/API Worker **分成独立 AKS Deployment，并按各自瓶颈独立扩缩容**。部署使用 Helm 生成 Deployment、Service、HPA、KEDA ScaledObject、ConfigMap 和 PodDisruptionBudget 等 Manifest；系统通过灰度、版本兼容和端到端 Trace 保障发布，并处理过一次**“Worker 扩容反而加剧 Graph 429 和队列积压”**的事故。完整设计和代表性 YAML 见 [AKS 部署与生产运行]({{ site.baseurl }}/docs/career/my-outlook-agent/aks-production/)。

这条主线主要回答：为什么 Service 和 Worker 要隔离、为什么不能只看 Queue Depth 扩容、如何做背压和优先级调度，以及怎样区分 AKS 资源不足和下游配额瓶颈。

## 面试深挖路径

这段项目不是靠组件名体现深度，而是能从一条主线继续回答故障窗口和设计取舍：

| 面试方向 | 可以使用的项目事实 | 对应问题 |
| --- | --- | --- |
| Agent 部署 | 短决策运行在 POS Service，长任务进入 Task Store、Queue 和 Worker | [短任务和长任务的 Agent Runtime 怎样部署？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#short-long-agent-runtime-deployment) |
| 生产架构 | Runtime、模型调用、Worker、Task State、Artifact Store 和业务 API 分层 | [生产级 Agent 架构怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/system-production/#production-agent-architecture) |
| 状态恢复 | Task/Run/Step、稳定 Checkpoint、Lease、Attempt 和结果未知 | [Worker 重启后怎样恢复 Agent 任务？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#agent-worker-recovery) |
| Context Engineering | 从 Task State、Finding 和 Artifact 动态重建 Context，历史按预算压缩 | [一次 Agent 调用的 Context 怎样组装？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-context-assembly) |
| 队列与幂等 | Task Store、补发 Dispatcher、At Least Once、Lease 与业务幂等 | [消息队列在 AI Agent 系统中解决什么问题？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#message-queue-agent-tasks) |
| 容量与扩缩容 | Queue Age、Worker 并发、模型和 Graph 配额共同限制吞吐 | [消息积压时能否直接增加消费者？]({{ site.baseurl }}/docs/interview/backend/message-queue/#scale-consumers-for-backlog) |
| 发布与可观测性 | Task 固定版本，Trace 串联 Conversation、Task、Step 和依赖调用 | [模型、Prompt 和工具变化怎样安全发布？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#llm-release-versioning) |

## 经历边界

这段经历最有价值的是**参与自托管 Agent 的部署与 Runtime 工程**，不是声称拥有整个 My Outlook。可以深入讲 Service—Worker 任务契约、Context Builder、恢复、AKS 扩缩容和生产事故；Deep Scan 与 Synthesis 的业务算法、Microsoft 365 数据平台和基础模型 Endpoint 由对应团队维护。

它补充了 Declarative Agent 项目主要使用平台 Runtime 的经验：前者说明怎样在托管框架上开发业务场景，My Outlook 则说明自己掌握 Agent Loop 时怎样部署、持久化、恢复和治理生产系统。
