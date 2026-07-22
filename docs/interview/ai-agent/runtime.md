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

## LangChain 和 LangGraph 分别解决什么问题？
{: #langchain-langgraph }

- **LangChain**：提供模型、Prompt、Retriever、工具和 Agent 等组件及集成，适合快速组装 LLM 应用、RAG 和工具链路；
- **LangGraph**：用节点、边和共享状态显式描述有状态执行流程，适合条件分支、循环、检查点、恢复和人工介入。

两者可以组合，LangGraph 可以复用 LangChain 的模型和工具组件。简单调用或线性流程不一定需要 LangGraph；路径会动态变化、需要保存执行状态并从中断处恢复时，图编排更有价值。两者都不会自动解决业务权限、权威状态和生产部署。

相关内容：[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## 什么是意图识别？它在 Agent 系统中处于什么环节？
{: #agent-intent-recognition }

**意图识别是判断用户想完成什么目标，以及系统接下来应进入哪类任务。**同一句“帮我处理一下”可能表示总结、查询或执行写操作，意图不等于原始句子中的关键词。

它通常发生在输入和界面 Context 规范化之后、任务路由和 Planning 之前：

```text
用户输入与当前 Context
→ 意图识别和必要的实体提取
→ 选择任务、工具集合或固定流程
→ Planning 与执行
```

常见实现包括：

- 明确按钮、命令和关键词使用确定性规则；
- 高频且类别稳定的场景使用分类模型或 Embedding 相似度；
- 开放表达使用大模型按 Schema 返回意图、实体、置信信息和缺失字段；
- 生产系统常采用规则优先、模型补充的混合路由，并保留 `unknown` 和多意图结果。

低置信度不应机械等同于追问。只读、可撤销操作可以说明假设后继续；存在多个合理解释或涉及高风险写入时，才提出会改变后续路径的最小澄清。意图确定后仍要由权限和业务规则校验真实动作。

相关内容：[模型、程序与人的决策边界]({{ site.baseurl }}/docs/ai-agent/agent-design/decision-boundary/)。

## 用户意图模糊或信息不足时，Agent 怎样处理？
{: #agent-intent-ambiguity }

**存在多个合理解释，或者缺少执行所需的信息时，应先澄清，不能猜测后执行高风险动作。**

1. **判断缺口**：根据当前任务、工具 Schema 和业务规则，识别缺失字段、冲突信息或多个候选意图；
2. **最小追问**：只询问会改变后续路径的关键信息，并保留已经确认的事实；
3. **按风险处理**：低风险回答可以说明假设后继续；支付、退款和数据修改必须得到明确确认，并由服务端重新校验。

连续澄清仍无法推进时，应停止当前动作或转人工。高频且能够稳定枚举的意图适合由程序路由，模型主要处理开放表达和长尾判断。

相关内容：[模型、程序与人的决策边界]({{ site.baseurl }}/docs/ai-agent/agent-design/decision-boundary/)、[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)。

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

## 如何提升 Agent 的工具调用正确率？
{: #agent-tool-calling-design }

工具调用正确率要拆成**候选工具召回、工具选择、参数生成和结果处理**分别优化：

1. **减少候选**：只暴露当前任务和权限范围内的工具；工具很多时先检索候选，再加载完整 Schema；
2. **写清契约**：名称和 Description 说明何时使用、返回什么、不能替代什么，重叠工具优先合并或显式路由；
3. **约束参数**：使用明确类型、枚举、必填字段和业务对象 ID，减少自由文本；Runtime 仍要执行 Schema 与业务校验；
4. **稳定结果**：Tool Result 区分成功、参数错误、权限不足、临时失败和结果未知，让模型能够正确选择下一步；
5. **用 Trace 和评测迭代**：分别统计候选召回率、工具选择率、参数正确率和任务完成率，从第一次错误决策定位 Prompt、Schema 还是工具实现。

高风险工具还要在模型之外实施鉴权、确认和幂等。提升正确率不能靠不断向 Prompt 追加规则，也不能把参数符合 Schema 等同于业务执行正确。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

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

- **MCP**：让模型应用通过统一的客户端—服务器协议接入工具、资源和提示模板，减少每个 Runtime 分别适配外部能力的成本；
- **A2A**：让具有独立身份和生命周期的 Agent 发现能力，并交换任务、消息、状态和产物，适合跨产品或跨团队协作；
- **ACP**：连接代码编辑器等客户端与编码 Agent。

可以记成：**MCP 连接 Agent 与能力，A2A 连接 Agent 与 Agent，ACP 连接交互客户端与编码 Agent。**协议只统一通信契约，不会自动解决鉴权、任务状态、幂等和故障恢复。具体实现和版本应以各自规范为准。参考：[MCP](https://modelcontextprotocol.io/docs/learn/server-concepts)、[A2A](https://a2a-protocol.org/latest/specification/)、[ACP](https://agentclientprotocol.com/)。

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
