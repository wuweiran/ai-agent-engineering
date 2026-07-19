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

**Planning 把目标拆成子任务、依赖、执行者和完成信号。**

- 简单任务可以每轮只决定下一步；
- 复杂任务先形成粗计划，再根据 Tool Result 做 Replanning；
- 模型生成和调整开放任务的候选计划，程序守住阶段、依赖、权限和审批；
- 多工具调用按数据依赖形成 DAG：无依赖的只读操作可并行，有依赖的串行，写同一资源的操作避免并发冲突；
- Runtime 最终检查预算和完成条件。

相关内容：[Agent Planning]({{ site.baseurl }}/docs/ai-agent/agent-design/planning/)。

## 多 Agent 有哪些常见协作模式？
{: #multi-agent-patterns }

- **顺序接力**：上一个 Agent 的结果交给下一个，职责清楚，但延迟累积、错误会传播；
- **并行执行**：同时处理独立子问题，速度快、Context 隔离好，但要处理重复、冲突和结果合并；
- **Manager-Worker**：Manager 动态拆分和汇总，Worker 专业执行，适合任务数量变化的场景，但 Manager 可能成为瓶颈。

只有任务能够清楚拆分，并且**并行、专业化或 Context 隔离的收益大于协调成本**时，才值得使用多个 Agent。协调层可以根据子任务数量和依赖关系选择顺序、并行或 Manager-Worker，但切换条件应明确可测，不能让多个 Agent 临时争抢同一任务。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## Multi-Agent 系统怎样划分三层职责？
{: #multi-agent-three-layer-architecture }

“三层架构”不是统一行业标准，可以按生产职责划分为：

1. **协调层**：拆分目标、调度依赖、合并结果、处理部分失败并判断终止；
2. **Agent 执行层**：由具有独立 Context、专业工具和能力边界的 Agent 完成子任务；
3. **Runtime 与资源层**：负责工具执行、权限、任务状态、消息、Artifact、幂等和 Trace。

核心边界是：**开放判断留给 Agent，任务事实和执行约束留给确定性系统。**三层不等于三个模型，也不要求部署成三个服务。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## Agent 之间怎样传递信息？
{: #multi-agent-communication }

常见方式可以分成四类：

- **直接调用**：调用指定 Agent 并等待或稍后取得结果，适合短任务和强依赖委派；
- **点对点消息**：向指定 Agent 或任务线程发送结构化消息，适合长任务和异步协作；
- **共享状态或 Artifact**：通过数据库、文件或对象存储交换任务状态和大型产物；
- **事件驱动**：发布状态变化，由订阅者决定是否处理，适合松耦合扩展。

真实系统通常混用四种方式。直接调用不代表被调用 Agent 没有自主性；事件驱动则是发布订阅形式的消息传递。所谓“共享内存”通常应实现为受控的状态存储或 Artifact，而不是共享完整模型 Context。

通信契约至少包含：任务和父任务 ID、发送者与接收者、目标与范围、输入引用、状态、结果或错误、版本和幂等标识。协调层为子任务指定唯一负责人或租约，共享状态使用版本检查，真实写入使用幂等键，避免多个 Agent 冲突或重复执行。还要处理超时、迟到和部分失败。跨产品 Agent 可以使用 A2A，但协议不会替代任务状态、权限和故障恢复。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)、[Claude Code 子 Agent]({{ site.baseurl }}/docs/ai-agent/claude-code/subagents/)。
