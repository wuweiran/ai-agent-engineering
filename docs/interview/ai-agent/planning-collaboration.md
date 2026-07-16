---
layout: default
title: Planning 与多 Agent 面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 5
permalink: /docs/interview/ai-agent/planning-collaboration/
---

# Planning 与多 Agent 面试题

这些问题讨论复杂目标怎样拆解、工具调用怎样表达依赖，以及多个 Agent 何时值得协作。

## Agent 怎样进行任务规划？
{: #agent-planning }

Planning 把目标拆成子任务、依赖、执行者和完成信号。简单任务可以每轮只决定下一步，复杂任务可以先形成粗计划，再根据工具结果 Replanning。

模型适合生成和调整开放任务的候选计划，程序规则负责不可违反的阶段、依赖、权限和审批。多工具调用时，先按数据依赖形成 DAG：无依赖的只读调用可以并行，后一步需要前一步结果时必须串行，可能写入同一资源的动作则要避免并发冲突。Runtime 还要检查预算和完成条件。

相关内容：[Agent Planning]({{ site.baseurl }}/docs/ai-agent/agent-design/planning/)。

## 多 Agent 有哪些常见协作模式？
{: #multi-agent-patterns }

常见模式包括顺序接力、并行执行和 Manager-Worker 分层。顺序简单但延迟累积；并行需要处理重复、冲突和结果合并；分层职责清晰，但 Manager 可能成为瓶颈。

只有任务能够清楚拆分，并行、专业化或 Context 隔离收益足够时，才值得使用多个 Agent。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## Multi-Agent 系统怎样划分三层职责？
{: #multi-agent-three-layer-architecture }

“三层架构”不是统一标准，但生产系统可以按职责理解为三层：协调层负责拆分目标、调度依赖、合并结果和判断终止；Agent 执行层包含具有独立 Context、工具和能力边界的专业 Agent；共享 Runtime 与资源层负责工具执行、权限、任务状态、消息、Artifact 和 Trace。

这种划分把开放判断留给 Agent，把任务事实和执行边界留给确定性系统。三层不是三个模型，也不要求每层独立部署；小系统可以由一个服务同时承担，只要职责边界清楚。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## Agent 之间怎样传递信息？
{: #multi-agent-communication }

可以同步直接调用、异步发送消息或事件，也可以通过共享任务状态和 Artifact 交换结果。短而强依赖的子任务适合同步调用；长任务更适合异步消息；大文件和完整证据通常保存为 Artifact，只在消息中传引用。

通信契约至少要包含任务与父任务 ID、发送者和接收者、目标与范围、输入引用、状态、结果或错误、版本和幂等标识。消费者还要处理超时、重复、迟到和部分失败。共享完整 Context 不是默认答案，它会增加信息污染和并发冲突，通常只传递子任务真正需要的目标、证据和未知项。

同一 Runtime 内可以使用内部调用或消息队列；跨产品 Agent 可以使用 A2A 交换任务、消息、状态和产物。协议改变连接方式，不会替代任务状态、权限和失败恢复。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)、[Claude Code 子 Agent]({{ site.baseurl }}/docs/ai-agent/claude-code/subagents/)。
