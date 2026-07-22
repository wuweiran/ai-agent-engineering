---
layout: default
title: AI Agent 常见面试题
parent: 面试题库
nav_order: 2
has_children: true
permalink: /docs/interview/ai-agent/
---

# AI Agent 常见面试题

AI Agent 岗位既考查模型与 RAG 等应用基础，也考查 Runtime、工具、状态、规划、评测和生产可靠性。题库按知识职责拆成六个部分，答案以能够直接口述为目标，详细机制通过链接回到知识正文。

## 题库结构

- [大模型与应用基础]({{ site.baseurl }}/docs/interview/ai-agent/llm/)：多模态结构、面向 Agent 的模型训练、Prompt、结构化输出与 Context 管理；
- [Agent 基础与工具]({{ site.baseurl }}/docs/interview/ai-agent/runtime/)：Agent 定义、ReAct、意图识别、LangChain、LangGraph、Function Calling、MCP、Skill 与工具失败；
- [Agent Context 与记忆]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/)：Dialog State、会话历史、短期与长期记忆、Context 组装与去噪；
- [RAG 与知识检索]({{ site.baseurl }}/docs/interview/ai-agent/rag/)：Query 改写、Chunk、混合检索、Rerank、GraphRAG、评测与性能；
- [Planning 与多 Agent]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/)：任务规划、工具依赖、协作模式与通信；
- [Agent 质量、安全与生产运行]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/)：Harness、Prompt Injection、幻觉、量化指标、质量维度、优化、缓存与进度事件。

## 原有问题索引

以下短索引保留原页面的稳定锚点。已有书签仍会落到对应题目入口，新引用应直接指向子页面。

### Agent 基础与工具

### [什么是 AI Agent？它与普通模型调用有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-vs-llm-call)
{: #agent-vs-llm-call }

### [模型和 Agent 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#model-vs-agent)
{: #model-vs-agent }

### [Agent 的基本架构是什么？它和 LLM Chain 或 Workflow 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-vs-workflow)
{: #agent-vs-workflow }

### [ChatBot、RPA、Workflow 和 Agent 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#chatbot-rpa-workflow-agent)
{: #chatbot-rpa-workflow-agent }

### [什么是 ReAct？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#react-loop)
{: #react-loop }

### [ReAct、Act-only 和 Plan-then-Execute 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-execution-patterns)
{: #agent-execution-patterns }

### [什么是意图识别？它在 Agent 系统中处于什么环节？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-intent-recognition)
{: #agent-intent-recognition }

### [LangChain 和 LangGraph 分别解决什么问题？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#langchain-langgraph)
{: #langchain-langgraph }

### [什么是 Function Calling？模型会自己执行函数吗？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#function-calling)
{: #function-calling }

### [如何提升 Agent 的工具调用正确率？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-tool-calling-design)
{: #agent-tool-calling-design }

### [Function Calling、MCP、A2A 和 Skill 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#function-calling-mcp-a2a-skill)
{: #function-calling-mcp-a2a-skill }

### [本地工具、MCP 和 Skill 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#local-tool-mcp-skill)
{: #local-tool-mcp-skill }

### [MCP、A2A 和 ACP 分别解决什么问题？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#mcp-a2a-acp)
{: #mcp-a2a-acp }

### [工具调用失败后怎样设计降级？]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#tool-failure-fallback)
{: #tool-failure-fallback }

### 大模型与应用基础

### [System Prompt 是什么？能否代替权限规则？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#system-prompt)
{: #system-prompt }

### [Prompt Engineering 和 Context Engineering 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#prompt-vs-context-engineering)
{: #prompt-vs-context-engineering }

### [怎样设计和优化 Prompt？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#prompt-optimization)
{: #prompt-optimization }

### [DSPy 适合解决什么问题？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#dspy-prompt-optimization)
{: #dspy-prompt-optimization }

### [CoT 和 Self-Consistency 分别解决什么问题？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#cot-self-consistency)
{: #cot-self-consistency }

### [JSON Mode、结构化输出和工具调用有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/llm/#structured-output-vs-tool-calling)
{: #structured-output-vs-tool-calling }

### Agent Context 与记忆

### [Agent 的状态、短期记忆和长期记忆有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-state-and-memory)
{: #agent-state-and-memory }

### [Agent 的长期记忆怎样实现？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-long-term-memory)
{: #agent-long-term-memory }

### [多轮复杂对话中的 Dialog State 怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#dialog-state-design)
{: #dialog-state-design }

### [多轮对话的存储和记忆怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#conversation-storage-memory)
{: #conversation-storage-memory }

### [Agent 的记忆怎样更新和遗忘？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#memory-update-forgetting)
{: #memory-update-forgetting }

### [Claude Code 怎样管理当前 Context、会话记录和跨会话记忆？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#claude-code-context-memory)
{: #claude-code-context-memory }

### [Agent 怎样管理长短期记忆，并在多轮对话中保留关键事实？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#long-context-key-information)
{: #long-context-key-information }

### [Lost in the Middle 是什么？怎样缓解？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#lost-in-the-middle)
{: #lost-in-the-middle }

### [Context 噪声太多时怎样处理？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#context-noise-cleaning)
{: #context-noise-cleaning }

### [Agent Context Engineering 有哪些主要技术？管理策略怎样设计？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-context-engineering-strategy)
{: #agent-context-engineering-strategy }

### [一次 Agent 调用的 Context 怎样组装？]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-context-assembly)
{: #agent-context-assembly }

### RAG 与知识检索

### [RAG 的原理是什么？怎样缓解幻觉和知识过期？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-principle)
{: #rag-principle }

### [RAG 的完整流程是什么？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-end-to-end-flow)
{: #rag-end-to-end-flow }

### [企业知识库怎样构建，并接入 RAG？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-pipeline)
{: #rag-pipeline }

### [Naive RAG、Advanced RAG 和 GraphRAG 有什么区别？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#naive-advanced-graph-rag)
{: #naive-advanced-graph-rag }

### [GraphRAG 适合什么问题？什么时候不值得使用？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#graph-rag-use-cases)
{: #graph-rag-use-cases }

### [RAG 为什么需要 Query 改写？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-query-rewriting)
{: #rag-query-rewriting }

### [Query 改写有哪些常见策略？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-query-rewriting-strategies)
{: #rag-query-rewriting-strategies }

### [Query Drift 怎样发现和控制？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-query-drift)
{: #rag-query-drift }

### [Query 改写怎样控制延迟和成本？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-query-rewriting-cost)
{: #rag-query-rewriting-cost }

### [RAG 有哪些分块策略？Chunk 大小怎样选择？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-chunk-size)
{: #rag-chunk-size }

### [父子索引什么时候值得使用？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-parent-child-index)
{: #rag-parent-child-index }

### [文档怎样高效索引和检索？Retriever 应怎样选择？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#hybrid-search-rerank)
{: #hybrid-search-rerank }

### [Chunk 过多或质量参差时，怎样 Rerank 和融合？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-rerank-context-budget)
{: #rag-rerank-context-budget }

### [RAG 系统怎样评测？评测集包含什么？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-evaluation-dataset)
{: #rag-evaluation-dataset }

### [知识检索如何提高回答正确率？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-optimization)
{: #rag-optimization }

### [RAG 的端到端性能怎样优化？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#rag-performance)
{: #rag-performance }

### [为什么代码搜索不一定优先使用 RAG？]({{ site.baseurl }}/docs/interview/ai-agent/rag/#code-search-rag-vs-grep)
{: #code-search-rag-vs-grep }

### Planning 与多 Agent

### [Agent 怎样进行任务规划？]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/#agent-planning)
{: #agent-planning }

### [多 Agent 有哪些常见协作模式？]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/#multi-agent-patterns)
{: #multi-agent-patterns }

### [Multi-Agent 系统怎样划分三层职责？]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/#multi-agent-three-layer-architecture)
{: #multi-agent-three-layer-architecture }

### [Agent 之间怎样传递信息？]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/#multi-agent-communication)
{: #multi-agent-communication }

### Agent 质量、安全与生产运行

### [什么是 Agent Harness？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-harness)
{: #agent-harness }

### [Prompt Injection 怎样防御？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#prompt-injection-defense)
{: #prompt-injection-defense }

### [什么是幻觉？怎样减少？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#hallucination)
{: #hallucination }

### [怎样评估 Agent 的执行效果？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-evaluation)
{: #agent-evaluation }

### [如何评估一个 AI Agent？有哪些可量化指标？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#online-agent-metrics)
{: #online-agent-metrics }

### [相关性、完整性和一致性分别衡量什么？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#relevance-completeness-consistency)
{: #relevance-completeness-consistency }

### [怎样证明一次 Agent 优化真的有效？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-optimization-evidence)
{: #agent-optimization-evidence }

### [怎样使用缓存优化大模型应用？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#llm-application-cache)
{: #llm-application-cache }

### [怎样使用 SSE 推送 Agent 进度？]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/#agent-sse-events)
{: #agent-sse-events }
