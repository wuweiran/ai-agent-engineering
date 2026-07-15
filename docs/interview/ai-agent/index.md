---
layout: default
title: AI Agent 常见面试题
parent: 面试题库
nav_order: 2
permalink: /docs/interview/ai-agent/
---

# AI Agent 常见面试题

这些问题覆盖 Agent 的定义、执行循环、工具、Context、记忆、RAG、规划、多 Agent、质量与生产运行。答案先给出能够直接口述的核心判断，具体实现与取舍通过链接回到知识正文。

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

## Agent 和 Workflow 有什么区别？
{: #agent-vs-workflow }

Workflow 的关键步骤与分支由程序预先定义；Agent 让模型根据运行中的观察动态选择或调整后续步骤。

两者经常组合：确定性的主流程负责权限、审批和业务状态，Agent 只处理难以穷举的调查或判断。路径能够稳定写清时，Workflow 通常更便宜、可靠且容易审计。

## ChatBot、RPA、Workflow 和 Agent 有什么区别？
{: #chatbot-rpa-workflow-agent }

ChatBot 描述通过对话与用户交互；RPA 主要模拟人在界面上的固定操作；Workflow 按预设节点和条件推进；Agent 让模型根据现场反馈参与决定路径。

它们不是互斥层级。聊天界面可以承载 Workflow 或 Agent，Agent 也可以调用 RPA 操作缺少 API 的旧系统。

## 什么是 ReAct？Agent 怎样避免无限循环？
{: #react-loop }

ReAct 是“判断下一步、执行动作、观察结果、继续判断”的循环。它让模型依据环境反馈调整路径，不要求向用户展示模型的内部推理。

防止无限循环不能只设最大轮数。Runtime 还应限制总时间、Token 和工具预算，检测重复调用与无进展，并在满足完成条件、需要等待或风险过高时停止。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)、[Agent 执行与终止]({{ site.baseurl }}/docs/ai-agent/agent-design/execution-control/)。

## ReAct、Act-only 和 Plan-then-Execute 有什么区别？
{: #agent-execution-patterns }

Act-only 让模型直接行动并根据结果继续，延迟较低但更容易偏离目标。ReAct 强调判断、行动和观察交替。Plan-then-Execute 先形成计划再执行，适合步骤相对稳定的复杂任务，但环境变化后需要重新规划。

生产系统常组合这些方式，不必把它们当作互斥框架。

## 工具、MCP 与提示词

## 什么是 Function Calling？模型会自己执行函数吗？
{: #function-calling }

Function Calling 或 Tool Calling 是模型与应用交换结构化工具请求的协议。应用提供工具名称、描述和参数 Schema；模型返回调用请求；Runtime 执行真实函数，再把结果送回模型。

模型不会自己访问数据库或运行函数。参数校验、权限、审批、幂等和真实执行都发生在模型之外。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## 本地工具、MCP 和 Skill 有什么区别？
{: #local-tool-mcp-skill }

本地工具由应用直接注册和执行，与当前 Runtime 紧密集成。MCP 用标准协议向 Host 暴露工具、资源和提示模板，便于不同客户端接入。Skill 描述完成某类任务的方法、步骤和约束，可以使用本地工具或 MCP 工具，但本身不等于执行协议。

MCP 解决能力怎样接入，Skill 解决 Agent 应该怎样使用能力。

## MCP、A2A 和 ACP 分别解决什么问题？
{: #mcp-a2a-acp }

MCP 主要连接模型应用与工具和数据。A2A 关注不同 Agent 的发现、任务、消息和产物交换。ACP 连接代码编辑器等客户端与编码 Agent。

它们处在不同边界，不应当作彼此替代的同类协议。具体实现和版本应以各自规范为准。参考：[MCP](https://modelcontextprotocol.io/docs/learn/server-concepts)、[A2A](https://a2a-protocol.org/latest/specification/)、[ACP](https://agentclientprotocol.com/)。

## 工具调用失败后怎样设计降级？
{: #tool-failure-fallback }

先区分失败类型：参数和权限错误不应重试；临时限流和连接失败可以有限退避；有副作用的调用超时后结果可能未知，应先查询真实状态。

非关键步骤可以跳过或使用明确标注的低精度结果，关键动作无法确认时应等待或转人工。备用工具、缓存、超时、熔断和重试都要服从业务正确性，不能为了给出答案而伪造成功。

## System Prompt 是什么？能否代替权限规则？
{: #system-prompt }

System Prompt 通常承载模型在当前应用中的身份、稳定行为边界和输出要求，是每轮 Context 的一部分。

它能影响模型行为，却不能强制执行权限和业务规则。越权访问、金额限制和审批必须由 Runtime 与业务服务检查。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## Prompt Engineering 和 Context Engineering 有什么区别？
{: #prompt-vs-context-engineering }

Prompt Engineering 优化指令、Few-shot 示例和输出要求，帮助模型更稳定地使用已有能力。Context Engineering 管理模型本轮看到的信息流，包括系统规则、工具、检索材料、任务状态、记忆、排序和压缩。

模型没理解任务时优先检查 Prompt；缺事实、资料过期或噪声过多时优先检查 Context。两者共同决定结果，但都不能代替权限和业务校验。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## 怎样设计和优化 Prompt？
{: #prompt-optimization }

先明确背景、目标、可用信息、限制和输出要求；边界难以理解时加入少量代表性示例。Prompt 不应靠不断追加规则增长，而要针对真实失败修改。

优化时使用固定任务比较版本，观察质量、严重错误、Token、延迟和工具行为。修改多项机制时可做消融实验，确认改善究竟来自哪里。

## DSPy 适合解决什么问题？
{: #dspy-prompt-optimization }

DSPy 把大模型程序中的指令和 Few-shot 示例作为可优化参数，在训练样本和评分函数上搜索更好的组合。它适合 Prompt 对措辞敏感、人工迭代缺乏稳定评测的场景。

优化质量取决于样本和指标。必须保留独立测试集，并审查成本、延迟和严重失败；DSPy 不是脱离数据自动生成最佳 Prompt。

## CoT 和 Self-Consistency 分别解决什么问题？
{: #cot-self-consistency }

CoT 通过分步推理或中间结构帮助模型处理复杂问题。Self-Consistency 采样多条独立推理路径，再对最终答案进行投票或一致性选择，以减少单一路径的偶然错误。

它们会增加 Token、延迟和费用，也不适合无法比较答案的开放任务。应用不应依赖或保存模型不可见的内部推理；关键结论仍需外部证据验证。

## JSON Mode、结构化输出和工具调用有什么区别？
{: #structured-output-vs-tool-calling }

JSON Mode 通常只保证输出是合法 JSON。结构化输出进一步按 Schema 限制字段、类型和枚举。工具调用使用结构化参数表达模型希望应用执行的动作。

三者都只解决结果形状，不保证语义、权限和业务状态正确。

## 状态、记忆与 RAG

## Agent 的状态、短期记忆和长期记忆有什么区别？
{: #agent-state-and-memory }

任务状态保存当前目标、已确认事实、进度和恢复位置；短期记忆通常指当前 Context 中可直接使用的近期信息；长期记忆保存跨任务仍值得复用的经验或偏好。

Context 是模型当前看到的工作集，不等于完整任务状态。RAG 外部知识也不是用户记忆，它由独立知识库维护。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## 多轮对话的存储和记忆怎样设计？
{: #conversation-storage-memory }

完整会话记录应持久化到数据库或对象存储，当前 Context 只选择近期消息和必要摘要。任务事实、外部资源 ID 与审批不能只存在于对话中；跨会话经验再经过筛选进入长期记忆。

Context 接近上限时可以摘要旧历史，但摘要有损，容易变化的业务事实应从权威系统重新读取。不同用户、租户和频道的会话必须隔离。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## Agent 的记忆怎样更新和遗忘？
{: #memory-update-forgetting }

更新可以采用追加、覆盖单值事实、合并冲突信息或生成摘要。遗忘可以根据有效期、相关性、访问情况和保留政策移除或压缩内容。

重要的是保留来源、适用范围和失效条件。不能把模型猜测自动写成长期事实，也不能让不同用户或租户的记忆相互污染。

## Claude Code 怎样管理当前 Context、会话记录和跨会话记忆？
{: #claude-code-context-memory }

Context Window 是下一次模型调用实际可见的信息工作集；Session Transcript 保存较完整的消息和工具记录，用于恢复与审计；Auto Memory 保存未来会话仍值得复用的项目经验。项目文件和 `CLAUDE.md` 则分别保存代码事实与团队规则。

Context 接近上限时，Claude Code 会清理旧工具结果并压缩历史，但不会因此删除完整会话记录。压缩后若要继续修改代码，仍应重新读取磁盘上的当前文件。

相关内容：[Claude Code Context 组装]({{ site.baseurl }}/docs/ai-agent/claude-code/context-assembly/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## Lost in the Middle 是什么？怎样缓解？
{: #lost-in-the-middle }

模型在长 Context 中可能更容易利用开头和结尾的信息，而忽略中间的关键证据。内容仍在窗口中，不代表会被稳定使用。

可以减少无关材料、通过 Rerank 选出高价值候选，并把重要目标、约束和证据放在清楚位置。位置重排只能缓解问题，不能修复错误检索和过期知识。

## Context 噪声太多时怎样处理？
{: #context-noise-cleaning }

先通过来源、时间、权限和相关性过滤材料，再让模型在回答前标记无关、冲突和过期内容。Self-Refine 一类自反馈方法可以迭代清洗和修订结果。

同一模型可能重复原误判，因此不能让它自行删除关键反面证据。高风险内容仍需确定性规则、可信来源或独立评审。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## 一个标准 RAG 系统包含哪些步骤？
{: #rag-pipeline }

离线阶段包括文档解析、Chunk、Embedding 和索引；在线阶段包括 Query 改写、问题向量化、召回、过滤或混合检索、Rerank、组装 Context 和生成回答。

评测要分开检查检索是否找到正确证据，以及回答是否受到证据支持。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Naive RAG、Advanced RAG 和 GraphRAG 有什么区别？
{: #naive-advanced-graph-rag }

Naive RAG 是最小文本检索链路：离线建立 Chunk 向量索引，在线对 Query 做向量 Top-K 检索，再把候选交给模型生成。文档切片不发生在用户提问之后。

Advanced RAG 在文本检索前后增加 Query 改写、混合检索、过滤、Rerank 和 Context 选择，重点是提高召回与排序质量。GraphRAG 则显式建立实体和关系图，通过子图与路径取得多跳证据，也可以用社区摘要支持全局概括。

它们不是所有系统都必须依次升级的三个版本。Advanced RAG 优化文本检索链路，GraphRAG 改变知识表示和检索路径；生产系统可以组合关键词、向量和图检索。

## GraphRAG 适合什么问题？什么时候不值得使用？
{: #graph-rag-use-cases }

GraphRAG 适合实体关系密集、多跳关联、跨文档证据组合和资料库全局主题概括。例如查询公司之间的投资关系，或综合大量报告识别主要风险主题。

简单 FAQ、局部事实查询和小型知识库通常先用混合检索与 Rerank。图谱抽取存在错误，知识更新还要维护实体、关系与摘要；只有评测证明普通文本检索持续败在关系或全局视角上时，这些成本才值得承担。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## RAG 为什么需要 Query 改写？
{: #rag-query-rewriting }

用户问题可能存在口语与文档术语的语义鸿沟、多轮指代和复杂子目标。Query 改写把它转换成一个或多个独立、明确、适合检索的查询，从而提高召回率。

改写不是必选步骤。简单查询直接检索更快；复杂、多轮或初次召回不足时，改写才更有价值。

## Query 改写有哪些常见策略？
{: #rag-query-rewriting-strategies }

上下文补全把“它为什么失败”改成带实体和条件的独立查询；查询扩展补充同义词、领域术语或多个改写；任务分解把多跳问题拆成子查询。

HyDE 生成假设性相关文档并用其 Embedding 检索；Step-back 先提出更抽象的原则性问题。Step-back 与细粒度任务分解解决的不是同一问题。

## Query Drift 怎样发现和控制？
{: #rag-query-drift }

改写可能丢失原问题中的实体、时间、租户或约束，使检索偏离用户意图。可以同时保留原始 Query 与改写 Query，混合召回并比较结果；改写缺少关键约束时拒绝使用。

还可以比较语义相似度和实体覆盖，但固定阈值不是通用答案。最终应通过原问题对应的召回与回答指标评估。

## Query 改写怎样控制延迟和成本？
{: #rag-query-rewriting-cost }

先做意图或复杂度路由，只对多轮、组合问题和召回不足的查询改写。可以使用较小模型生成改写，并将多个独立查询并行检索。

多查询会增加向量检索、关键词检索和 Rerank 成本。优化目标是端到端回答质量，而不是生成尽可能多的 Query。

## Chunk 大小怎样选择？
{: #rag-chunk-size }

Chunk 太小会缺少上下文，太大会降低匹配精度并占用更多 Context。应根据文档结构、问题类型和检索指标选择，而不是套用固定数字。

父子索引可以用小块匹配，再返回所属大块，兼顾检索精度和上下文完整性，但会增加索引与映射复杂度。

## 为什么 RAG 常结合关键词检索和 Rerank？
{: #hybrid-search-rerank }

向量检索擅长语义近似，关键词检索更容易找到错误码、编号和专有名词。混合检索扩大可靠召回后，Rerank 再对少量候选做更精细的相关性判断。

这样可以把快速召回和精细排序分开，但会增加延迟、成本和调参工作。

## 为什么代码搜索不一定优先使用 RAG？
{: #code-search-rag-vs-grep }

函数名、变量名和错误码需要精确匹配时，Grep 或符号索引通常更直接。自然语言描述、跨文件概念和相似实现更适合语义检索。

工程系统可以组合两者，技术更复杂不代表在所有问题上更准确。

## Planning 与多 Agent

## Agent 怎样进行任务规划？
{: #agent-planning }

Planning 把目标拆成子任务、依赖、执行者和完成信号。简单任务可以每轮只决定下一步，复杂任务可以先形成粗计划，再根据工具结果 Replanning。

模型生成候选计划，Runtime 仍要检查权限、依赖、预算和完成条件。

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

## 质量、幻觉与生产运行

## 什么是 Agent Harness？
{: #agent-harness }

Agent Harness 是包围模型的执行与控制层，也常称为 Runtime。它负责组装 Context、暴露和执行工具、实施权限、保存状态、限制预算、处理等待和恢复，并记录 Trace。

Harness 让模型的概率判断进入确定的软件边界。这个术语没有唯一实现清单，重点是模型之外的控制责任。

## 什么是幻觉？怎样减少？
{: #hallucination }

幻觉是模型生成形式可信但事实、推理或引用没有可靠依据的内容。知识缺口、过期资料、冲突 Context 和多步错误传播都可能导致幻觉。

可以通过可信检索、保留来源、缩小回答范围、结构约束、外部事实校验和评测降低风险。RAG 和模型自检都不能保证消除幻觉。

相关内容：[大模型幻觉]({{ site.baseurl }}/docs/llm/hallucination/)。

## 怎样评估 Agent 的执行效果？
{: #agent-evaluation }

先沿用户交付定义端到端成功，再检查关键过程是否越权、重复写入或提前结束，同时记录延迟、Token、工具调用和故障恢复。

使用固定真实任务比较版本，失败时通过 Trace 找到第一次偏离。能精确判断的结果交给程序，开放语义再使用人工或经过校准的模型评审。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)。

## 怎样证明一次 Agent 优化真的有效？
{: #agent-optimization-evidence }

先建立评测基线，再修改最接近根因的层次，例如 Context、Skill、工具或业务服务。重跑失败样本、相邻任务和完整回归集，必要时关闭新机制做消融实验。

离线通过后逐步放量，并观察用户完成率和严重失败。只调整 Prompt、只看几个演示或只比较平均总分都不足以证明改善。

## 怎样使用缓存优化大模型应用？
{: #llm-application-cache }

可以缓存完全相同请求的最终结果，也可以在业务允许时复用语义相近问题的答案；供应商 Prompt Cache 则复用稳定提示词前缀的计算。

缓存键必须包含模型、Prompt、工具定义和关键 Context 版本，并定义失效与权限边界。动态业务状态、高风险回答和用户隔离内容不能只按问题文本复用。

## 怎样使用 SSE 推送 Agent 进度？
{: #agent-sse-events }

服务端返回 `text/event-stream`，持续发送以空行分隔的事件。应用可以定义文本增量、工具开始、工具结束、等待确认和任务完成等事件，并携带任务 ID 与序号。

SSE 是服务端到客户端的单向通道；客户端提交命令仍可使用普通 HTTP。断线不等于取消，重连应从持久任务状态或事件位置恢复。

相关内容：[流式生成与任务事件]({{ site.baseurl }}/docs/backend/llm-backend/streaming-generation/)。
