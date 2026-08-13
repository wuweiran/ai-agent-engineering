---
layout: default
title: Agent 系统设计与生产运行面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 6
permalink: /docs/interview/ai-agent/system-production/
---

# Agent 系统设计与生产运行面试题

这些问题讨论 Agent 的 Runtime、系统设计、部署形态和任务交付。

## 什么是 Agent Harness？
{: #agent-harness }

**Agent Harness 是包围模型的执行与控制层，也常称为 Runtime。**主要职责包括：

- 组装和压缩 Context；
- 暴露、校验并执行工具；
- 实施权限、审批和预算限制；
- 保存任务状态，处理等待、恢复和终止；
- 记录 Trace、指标和版本。

它让模型的概率判断进入确定的软件边界。这个术语没有唯一实现清单，重点是**模型之外的控制责任**。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## 怎样从零设计并落地一个 Agent 系统？
{: #agent-zero-to-production-methodology }

**不要从选择框架开始，而要沿着“业务目标 → Agent 边界 → 技术方案 → 评测 → 上线”推进。**一套完整方法可以分成七步：

1. **还原业务任务**：明确目标用户、触发入口、现有人工流程、输入和最终交付物，列出正常路径、例外情况和转人工条件；成功标准要落到可验证的业务终态，而不是“模型回答看起来不错”；
2. **[判断是否需要 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)**：单次总结、分类和抽取优先使用一次模型调用；步骤和分支能够预先枚举时使用 Workflow；只有下一步需要根据工具或环境反馈动态变化时，才引入 Agent；
3. **[确定自动化边界]({{ site.baseurl }}/docs/ai-agent/agent-design/decision-boundary/)**：划分模型、程序和人的职责。模型处理意图理解、开放判断和路径选择；程序执行权限、参数、预算、幂等和状态校验；高风险动作由用户确认或人工审批。第一版可以只给建议或生成草稿，再逐步开放真实执行；
4. **[完成技术选型]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)**：根据产品入口、任务时长、工具环境、状态恢复、安全隔离和团队维护能力，在托管 Agent 平台、Agent SDK、LangChain/LangGraph 等代码框架与自建 Runtime 之间选择。MCP 解决能力接入，不等于 Agent 框架；没有明确的职责、权限或 Context 隔离需求时，不默认拆成 Multi-Agent；
5. **[设计核心链路]({{ site.baseurl }}/docs/ai-agent/agent-building/business-integration/)**：定义 Prompt 和完成条件，确定每轮需要的 Context、Token Budget 和裁剪策略；为工具设计名称、Description、参数 Schema 和 Tool Result；区分 Conversation、Agent 任务、业务状态与长期记忆，并提前处理超时、重试、部分成功、结果未知和恢复；
6. **[评测先于扩量]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)**：从真实任务建立 Golden Set，分别检查任务完成、工具和参数、安全边界、故障恢复、延迟与每个成功任务的 Token 成本。失败时通过 Trace 找到第一次偏离，而不是只看最终回答或平均分；
7. **[最小闭环上线]({{ site.baseurl }}/docs/ai-agent/agent-production/release-upgrade/)**：先交付一个高价值场景和最少工具，完成鉴权、确认、可观测性和回滚；再经过离线回归、灰度发布和线上指标观察，把生产失败脱敏后补入评测集，持续扩大任务范围和自动化程度。

需求阶段至少应产出四项内容：**任务和成功标准、模型/程序/人工边界、工具与权威状态清单、评测样本与发布门槛**。技术选型只有能够解释这些约束，才是方案，而不是框架名称列表。

## 短任务和长任务的 Agent Runtime 怎样部署？
{: #short-long-agent-runtime-deployment }

判断标准不是固定耗时，而是任务是否需要**跨越当前请求和服务实例继续存在**：

- **短任务**：在 Agent API 内运行 LangChain 或 LangGraph 循环，通过 HTTP 或 SSE 返回结果；Conversation 可以存入外部 Checkpointer；
- **长任务**：API 创建持久 Task 并返回 `202 + task_id`，队列调度 Worker 执行。等待确认或外部事件时释放 Worker，条件满足后重新入队恢复。

需要重启恢复、排队接管或独立取消的任务应采用 **Task DB + Queue + Worker**。有副作用的 Tool Call 还要保存幂等键和结果状态，不能把 SSE 连接或队列消息当成任务状态。

相关内容：[Agent 应用的部署边界]({{ site.baseurl }}/docs/ai-agent/agent-building/#short-vs-long-agent-deployment)。

## 生产级 Agent 架构怎样设计？
{: #production-agent-architecture }

**生产级 Agent 的重点不是固定分成六个微服务，而是把入口控制、模型决策、确定性执行、权威状态和质量闭环分开。**一种常见分层是：

1. **接入与网关**：完成认证、租户隔离、限流、配额、请求 ID、会话路由和流式连接；
2. **Runtime 与编排**：组装 Context，驱动模型与 Tool Call 循环，管理 Planning、预算、确认、取消、超时和终止；Planner 与 Executor 可以是这一层的模块，不必各自成为服务；
3. **模型网关**：统一封装模型供应商、版本、路由、降级、Token 统计、Prompt Cache 和并发配额；
4. **工具与业务集成**：通过明确的名称、Description 和参数 Schema 暴露业务能力，执行时重新鉴权、校验业务状态，并处理幂等和结果未知；
5. **状态、Context 与 Memory**：分别保存 Conversation、Task/Run/Step、Tool Call、短期工作集和长期记忆。Context 可以压缩重建，业务状态必须由数据库或领域系统负责；
6. **结果处理与安全**：校验结构化结果、Citation 和工具状态，过滤敏感信息，区分成功、部分成功与结果未知，再生成用户响应。它通常属于 Runtime 或领域服务，不一定需要独立的 Aggregator；
7. **异步执行与生产闭环**：长任务使用持久队列和 Worker；Metrics、Log、Trace 串联整个链路，并通过离线评测、灰度发布、告警和回滚形成闭环。

一次请求可以概括为：

```text
Gateway 接收请求
→ Runtime 读取状态并组装 Context
→ Model Gateway 调用模型
→ Runtime 校验并执行 Tool Call
→ 领域服务提交真实业务结果
→ Runtime 保存状态并校验最终输出
→ SSE/HTTP 返回结果
```

设计时先确认四个问题：**权威业务状态在哪里、哪些动作有副作用、超时后怎样判断真实结果、系统如何恢复**。Memory、Aggregator 和 Planner 都是职责名称，不应为了架构图完整就强行拆成独立服务。

相关内容：[Agent Harness](#agent-harness)、[Agent 状态与记忆]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-state-and-memory)、[工具调用后端链路]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#tool-call-backend-flow)、[Agent 评测]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-evaluation)。

## 怎样使用 SSE 推送 Agent 进度？
{: #agent-sse-events }

SSE 适合把服务端的 Agent 进度单向推给客户端：

1. 服务端返回 `text/event-stream`；
2. 事件区分文本增量、工具开始与结束、等待确认、任务完成和失败；
3. 每个事件携带稳定任务 ID 和序号；
4. 客户端命令仍使用普通 HTTP；
5. 断线不等于取消，重连从持久任务状态或最后事件位置恢复。

**SSE 是观察通道，不是任务本身的权威状态。**

相关内容：[流式生成与任务事件]({{ site.baseurl }}/docs/backend/llm-backend/streaming-generation/)。
