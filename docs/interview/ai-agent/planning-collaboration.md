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

## Agent 之间怎样传递信息？
{: #multi-agent-communication }

可以直接调用、发送消息、共享状态或发布事件。无论采用哪种方式，都要定义结构化输入输出、任务 ID、权限、完成状态和部分失败语义。

共享 Context 不是默认答案。它会增加信息污染和并发冲突，通常只应传递子任务真正需要的内容。
