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

普通模型调用接收输入并返回一次输出，后续步骤通常由应用预先安排。Agent 围绕目标多轮运行，模型会根据工具和环境反馈参与选择下一步，Runtime 负责执行、状态、权限和终止。

工具、记忆和规划是常见能力，却不是判断 Agent 的固定清单。关键是模型的选择能否根据现场结果改变执行路径。

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)。

## 模型和 Agent 有什么区别？
{: #model-vs-agent }

模型是根据 Context 生成文本或结构化请求的推理组件。Agent 是包含模型、Runtime、工具、状态和执行边界的完整系统，能够读取环境并影响外部世界。

模型能力决定哪些判断可能完成，Agent 工程决定这些判断能否安全、持续地成为真实结果。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## Agent 的基本架构是什么？它和 LLM Chain 或 Workflow 有什么区别？
{: #agent-vs-workflow }

Agent 通常由模型、Runtime 或 Harness、Context 与状态、工具和执行循环组成。规划和长期记忆可以按任务需要加入，不是判断 Agent 的固定清单。模型根据目标和工具反馈选择下一步，Runtime 负责执行、权限、状态与终止。

LLM Chain 或 Workflow 的关键步骤和分支由程序预先定义，即使其中多个节点调用模型，执行路径仍由代码控制。Agent 的路径会根据运行中的观察动态变化。两者也可以组合：确定性主流程负责权限、审批和业务状态，Agent 只处理难以穷举的判断。

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)、[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## ChatBot、RPA、Workflow 和 Agent 有什么区别？
{: #chatbot-rpa-workflow-agent }

ChatBot 描述通过对话与用户交互；RPA 主要模拟人在界面上的固定操作；Workflow 按预设节点和条件推进；Agent 让模型根据现场反馈参与决定路径。

它们不是互斥层级。聊天界面可以承载 Workflow 或 Agent，Agent 也可以调用 RPA 操作缺少 API 的旧系统。

相关内容：[LLM 应用、固定工作流与 Agent]({{ site.baseurl }}/docs/ai-agent/llm-workflow-agent/)。

## 什么是 ReAct？
{: #react-loop }

ReAct 是“判断下一步、执行动作、观察结果、继续判断”的循环。它让模型依据环境反馈调整路径，不要求向用户展示模型的内部推理。

工程上，一轮通常保存 User 任务、Assistant 返回的文本与 Tool Call、Runtime 生成的 Tool Result，再发起下一次模型调用。角色和消息格式由模型 API 决定：有的 API 把工具结果放在独立 `tool` 角色；有的使用 User 消息中的 `tool_result` 内容块，并通过调用 ID 与原 Tool Call 关联。不能脱离供应商协议断言工具结果必须使用某个固定角色。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)。

## ReAct、Act-only 和 Plan-then-Execute 有什么区别？
{: #agent-execution-patterns }

Act-only 让模型直接行动并根据结果继续，延迟较低但更容易偏离目标。ReAct 强调判断、行动和观察交替。Plan-then-Execute 先形成计划再执行，适合步骤相对稳定的复杂任务，但环境变化后需要重新规划。

生产系统常组合这些方式，不必把它们当作互斥框架。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)、[Agent Planning]({{ site.baseurl }}/docs/ai-agent/agent-design/planning/)。

## Agent 出现死循环怎么办？异常处理机制怎样设计？
{: #agent-loop-exception-handling }

Runtime 不能只靠模型自行停止。它要定义完成、等待和失败状态，限制轮次、时间、Token、费用和工具调用次数，并检测相同工具与近似参数反复调用、任务事实和待办长期没有变化。达到阈值后可以要求模型换路，仍无进展则保存已确认事实和剩余问题，停止或转人工。

异常先按能否重试及副作用分类：参数、权限和业务冲突通常不重试；限流和临时连接失败采用有限退避；写操作超时属于结果未知，要先用幂等键或业务 ID 查询真实状态。每一步保存状态、尝试次数和错误语义，Worker 才能从稳定检查点恢复，而不是重放整个循环。

相关内容：[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)。

## 工具与协议

## 什么是 Function Calling？模型会自己执行函数吗？
{: #function-calling }

Function Calling 或 Tool Calling 是模型与应用交换结构化工具请求的协议。应用提供工具名称、描述和参数 Schema；模型返回调用请求；Runtime 执行真实函数，再把结果送回模型。

模型不会自己访问数据库或运行函数。参数校验、权限、审批、幂等和真实执行都发生在模型之外。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## Function Calling、MCP、A2A 和 Skill 有什么区别？
{: #function-calling-mcp-a2a-skill }

它们解决不同层次的问题：

- Function Calling 是模型输出结构化工具请求、Runtime 返回执行结果的交互机制；
- MCP 是应用接入外部工具、资源和提示模板的标准协议；
- A2A 用于独立 Agent 之间发现能力并交换任务、消息、状态和产物；
- Skill 是按需加载的任务方法和领域知识，告诉 Agent 某类工作怎样完成，本身不执行外部动作。

一次任务可以同时使用四者：Agent 加载报表 Skill，通过 A2A 接收另一个 Agent 委派的任务，经 MCP 发现企业数据工具，再由模型使用 Function Calling 请求具体工具。MCP 和 A2A 规定系统间怎样连接，Function Calling 规定模型怎样提出调用，Skill 提供做事方法。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)、[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)。

## 本地工具、MCP 和 Skill 有什么区别？
{: #local-tool-mcp-skill }

本地工具由应用直接注册和执行，与当前 Runtime 紧密集成。MCP 用标准协议向 Host 暴露工具、资源和提示模板，便于不同客户端接入。Skill 描述完成某类任务的方法、步骤和约束，可以使用本地工具或 MCP 工具，但本身不等于执行协议。

MCP 解决能力怎样接入，Skill 解决 Agent 应该怎样使用能力。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## MCP、A2A 和 ACP 分别解决什么问题？
{: #mcp-a2a-acp }

MCP 主要连接模型应用与工具和数据。A2A 关注不同 Agent 的发现、任务、消息和产物交换。ACP 连接代码编辑器等客户端与编码 Agent。

它们处在不同边界，不应当作彼此替代的同类协议。具体实现和版本应以各自规范为准。参考：[MCP](https://modelcontextprotocol.io/docs/learn/server-concepts)、[A2A](https://a2a-protocol.org/latest/specification/)、[ACP](https://agentclientprotocol.com/)。

相关内容：[多 Agent 协作]({{ site.baseurl }}/docs/ai-agent/agent-design/multi-agent/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## 工具有上百个时，怎样让模型快速、准确地选择？
{: #large-tool-library-selection }

不要把上百份完整 Schema 全部塞给模型。先按 Agent 角色、当前任务和权限缩小工具域，再用关键词或语义 Tool Search 召回少量候选，按需加载完整 Schema，由模型完成最终选择。工具名、使用时机、边界和参数描述还要清楚；高度相似的工具应合并或增加显式路由。

工程上同时关注选择准确率和成本：评测该调用时是否调用、工具是否选对、参数是否正确，以及候选召回率、延迟和 Context 占用。敏感工具即使进入候选集，也必须在执行时重新鉴权和审批。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## 工具调用失败后怎样设计降级？
{: #tool-failure-fallback }

先区分失败类型：参数和权限错误不应重试；临时限流和连接失败可以有限退避；有副作用的调用超时后结果可能未知，应先查询真实状态。

支付等写操作应在调用前保存业务请求 ID，并让下游支持幂等和结果查询。超时后先把状态记为结果未知，再用原请求 ID 查询；只有明确未执行时，才用同一幂等键重试，不能让模型自行再次支付。

非关键步骤可以跳过或使用明确标注的低精度结果，关键动作无法确认时应等待或转人工。备用工具、缓存、超时、熔断和重试都要服从业务正确性，不能为了给出答案而伪造成功。

相关内容：[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)、[Agent 任务可靠性]({{ site.baseurl }}/docs/ai-agent/agent-production/reliability/)。延伸问题：[工具调用超时后，模型能否自行重试？]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#tool-timeout-retry)
