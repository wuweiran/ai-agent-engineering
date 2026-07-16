---
layout: default
title: Agent 基础与工具面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 2
permalink: /docs/interview/ai-agent/runtime/
---

# Agent 基础与工具面试题

这些问题讨论 Agent 的定义、执行循环、工具协议和协作协议。判断 Agent 的关键不是是否使用工具或多次调用模型，而是模型能否根据环境反馈改变后续路径。

## Agent 与执行循环

## 什么是 AI Agent？它与普通模型调用有什么区别？
{: #agent-vs-llm-call }

**普通模型调用完成一次输入输出；Agent 围绕目标多轮判断和行动。**

- 普通调用的后续步骤通常由应用预先安排；
- Agent 会根据工具和环境反馈选择下一步；
- Runtime 负责执行、状态、权限、预算和终止。

工具、记忆和规划是常见能力，但不是判断 Agent 的固定清单。关键在于：**模型的选择是否会根据现场结果改变执行路径。**

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)。

## 模型和 Agent 有什么区别？
{: #model-vs-agent }

- **模型**：根据 Context 生成文本或结构化请求的推理组件；
- **Agent**：由模型、Runtime、工具、状态和执行边界组成的完整系统，能够读取环境并影响外部世界。

模型能力决定哪些判断可能完成，Agent 工程决定这些判断能否**安全、持续地成为真实结果**。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## Agent 的基本架构是什么？它和 LLM Chain 或 Workflow 有什么区别？
{: #agent-vs-workflow }

Agent 的基本架构包括：

1. **模型**：理解目标并提出下一步；
2. **Context 与状态**：提供当前信息并保存任务进度；
3. **工具**：连接外部数据与动作；
4. **Runtime 或 Harness**：执行工具，控制权限、预算和终止；
5. **执行循环**：根据 Observation 持续调整路径。

与 Chain 或 Workflow 的核心区别是**谁决定路径**：Workflow 的步骤和分支由程序预定义；Agent 的路径会根据运行中的观察动态变化。生产系统常把二者组合：确定性流程管理权限、审批和业务状态，Agent 处理难以穷举的判断。

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)、[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## ChatBot、RPA、Workflow 和 Agent 有什么区别？
{: #chatbot-rpa-workflow-agent }

- **ChatBot**：描述对话交互形式；
- **RPA**：按既定规则模拟人在界面上的操作；
- **Workflow**：按预设节点和条件推进任务；
- **Agent**：让模型根据现场反馈参与决定路径。

它们不是互斥层级。聊天界面可以承载 Workflow 或 Agent，Agent 也可以调用 RPA 操作缺少 API 的旧系统。

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)。

## 什么是 ReAct？
{: #react-loop }

**ReAct 是“判断下一步 → 执行动作 → 观察结果 → 继续判断”的循环。**它让模型根据环境反馈调整路径，不要求向用户展示内部推理。

工程上，一轮通常包括：

1. 模型返回文本和 Tool Call；
2. Runtime 校验并执行工具；
3. Tool Result 通过调用 ID 与原请求关联；
4. 模型读取结果并决定下一步。

工具结果使用独立 `tool` 角色，还是 User 消息中的 `tool_result` 内容块，取决于模型 API，不能脱离供应商协议断言只有一种格式。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## ReAct、Act-only 和 Plan-then-Execute 有什么区别？
{: #agent-execution-patterns }

- **Act-only**：模型直接行动并根据结果继续，延迟较低，但更容易偏离目标；
- **ReAct**：判断、行动和观察交替，适合路径随反馈变化的任务；
- **Plan-then-Execute**：先形成计划再执行，适合步骤较多、依赖较清楚的任务，但环境变化后需要 Replanning。

它们不是互斥框架，生产系统常按任务复杂度组合使用。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)、[Agent Planning]({{ site.baseurl }}/docs/ai-agent/agent-design/planning/)。

## Agent 出现死循环怎么办？异常处理机制怎样设计？
{: #agent-loop-exception-handling }

可以从**循环控制、异常分类、状态恢复**三层设计：

1. **循环控制**：定义完成、等待和失败状态；限制轮次、时间、Token、费用和工具调用次数；检测重复调用和连续无进展；
2. **异常分类**：参数、权限和业务冲突通常不重试；限流和临时连接失败有限退避；写操作超时按“结果未知”处理；
3. **状态恢复**：保存步骤、尝试次数、错误语义和稳定检查点。达到阈值后要求模型换路，仍无进展则保存事实与剩余问题，停止或转人工。

最大轮数只是最后防线，**完成条件和进展检测**才能更早阻止无意义循环。

相关内容：[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)。

## 工具与协议

## 什么是 Function Calling？模型会自己执行函数吗？
{: #function-calling }

**Function Calling 或 Tool Calling 是模型与应用交换结构化工具请求的协议，模型不会自己执行函数。**

流程是：应用提供工具名、描述和参数 Schema → 模型返回 Tool Call → Runtime 校验并执行真实函数 → Tool Result 返回模型继续判断。

参数校验、权限、审批、幂等和真实执行都发生在模型之外。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## Function Calling、MCP、A2A 和 Skill 有什么区别？
{: #function-calling-mcp-a2a-skill }

它们处在不同层次：

- **Function Calling**：模型怎样提出结构化工具调用；
- **MCP**：应用怎样标准化接入外部工具、资源和提示模板；
- **A2A**：独立 Agent 怎样发现能力并交换任务、消息、状态和产物；
- **Skill**：Agent 按需加载的任务方法和领域知识，本身不执行外部动作。

可以记成：**Function Calling 管模型调用，MCP 管能力接入，A2A 管 Agent 协作，Skill 管做事方法。**一次任务可以同时使用四者。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)、[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## 本地工具、MCP 和 Skill 有什么区别？
{: #local-tool-mcp-skill }

- **本地工具**：由当前应用直接注册和执行，与 Runtime 紧密集成；
- **MCP**：用标准协议暴露工具、资源和提示模板，便于多个客户端接入；
- **Skill**：描述完成某类任务的方法、步骤和约束，可以使用本地工具或 MCP 工具。

一句话概括：**MCP 解决能力怎样接入，Skill 解决 Agent 应该怎样使用能力。**

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## MCP、A2A 和 ACP 分别解决什么问题？
{: #mcp-a2a-acp }

- **MCP**：连接模型应用与工具和数据；
- **A2A**：连接不同 Agent，交换任务、消息、状态和产物；
- **ACP**：连接代码编辑器等客户端与编码 Agent。

三者位于不同边界，不能互相替代。具体实现和版本应以各自规范为准。参考：[MCP](https://modelcontextprotocol.io/docs/learn/server-concepts)、[A2A](https://a2a-protocol.org/latest/specification/)、[ACP](https://agentclientprotocol.com/)。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## 工具有上百个时，怎样让模型快速、准确地选择？
{: #large-tool-library-selection }

采用“**缩小工具域 → 检索候选 → 加载 Schema → 模型选择**”的分层过程：

1. 按 Agent 角色、当前任务和用户权限缩小工具域；
2. 用关键词或语义 Tool Search 召回少量候选；
3. 只加载候选工具的完整 Schema；
4. 由模型完成最终选择，并在执行时重新鉴权。

工具名和描述要明确“何时调用、不能替代什么”；高度相似的工具应合并或增加显式路由。评测同时看候选召回率、工具与参数正确率、延迟和 Context 占用。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## 工具调用失败后怎样设计降级？
{: #tool-failure-fallback }

先按失败语义分类：

- **不可重试**：参数错误、权限不足、永久业务冲突；
- **可以有限重试**：限流、临时连接失败，采用退避和重试预算；
- **结果未知**：支付、退款等有副作用的调用超时，先用业务请求 ID 查询真实状态；
- **无法完成**：非关键步骤可跳过并标明降级，关键动作则等待或转人工。

写操作调用前要保存业务请求 ID，并让下游支持幂等和结果查询。只有确认前一次未执行，才能使用同一幂等键重试；**不能让模型自行再次支付或退款。**

相关内容：[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)、[Agent 任务可靠性]({{ site.baseurl }}/docs/ai-agent/agent-production/reliability/)。延伸问题：[工具调用超时后，模型能否自行重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#tool-timeout-retry)
