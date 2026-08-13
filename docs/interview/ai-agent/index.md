---
layout: default
title: AI Agent 常见面试题
parent: 面试题库
nav_order: 2
has_children: true
permalink: /docs/interview/ai-agent/
---

# AI Agent 常见面试题

AI Agent 岗位既考查模型与 RAG 等应用基础，也考查 Runtime、工具、状态、规划、评测和生产可靠性。题库按知识职责拆成七个部分，答案以能够直接口述为目标，详细机制通过链接回到知识正文。

## AI Agent 面试关键词图谱

```text
用户提出目标
→ 判断使用模型调用、固定 Workflow 还是 Agent
→ Prompt 与 Context 提供规则和当前信息
→ RAG、Memory 与工具连接外部知识和业务系统
→ Runtime 驱动判断、行动与 Observation 循环
→ Planning 或多 Agent 处理复杂任务
→ 确定性软件守住权限、状态和副作用
→ 评测、Trace 与生产运行形成质量闭环
```

- **任务边界与模型能力**
  - **[大模型与应用基础]({{ site.baseurl }}/docs/interview/ai-agent/llm/)**：模型训练、System Prompt、Prompt Engineering、结构化输出与缓存
  - **[Agent 基础与工具]({{ site.baseurl }}/docs/interview/ai-agent/runtime/)**：模型调用、Chain、Workflow、Agent、ReAct、意图识别与执行循环
  - 这一层回答：**任务为什么需要 Agent，模型应该参与哪些开放判断。**

- **信息、状态与知识**
  - **[Agent Context 与记忆]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/)**：Dialog State、会话历史、任务状态、短期和长期记忆、Context 裁剪与组装
  - **[RAG 与知识检索]({{ site.baseurl }}/docs/interview/ai-agent/rag/)**：Query 改写、Chunk、混合检索、Rerank、GraphRAG、召回和排序评测
  - 这一层回答：**模型本轮应该看到什么，信息从哪里取得，怎样控制噪声、时效和 Token。**

- **动作、工具与协议**
  - **[Agent 基础与工具]({{ site.baseurl }}/docs/interview/ai-agent/runtime/#agent-tools-protocols)**：Function Calling、Tool Schema、MCP、Skill、工具检索和失败降级
  - 工具把模型决策连接到外部数据与动作；Runtime 和业务服务继续负责参数、权限、确认、幂等和真实结果。
  - 这一层回答：**模型怎样提出动作，软件怎样安全地执行动作。**

- **任务规划与协作**
  - **[Planning 与多 Agent]({{ site.baseurl }}/docs/interview/ai-agent/planning-collaboration/)**：任务拆分、依赖、Replanning、协作模式和通信
  - Planning 解决一个 Agent 内怎样安排步骤；Multi-Agent 只在并行、专业化或 Context 隔离收益大于协调成本时使用。
  - 这一层回答：**复杂任务怎样分解、调整和协作完成。**

- **系统设计与生产交付**
  - **[Agent 系统设计与生产运行]({{ site.baseurl }}/docs/interview/ai-agent/system-production/)**：Harness、从零落地、短长任务部署、生产级架构和 SSE
  - 短任务可以在 API 内运行；需要等待、恢复和接管的长任务使用持久 Task、Queue 和 Worker。
  - 这一层回答：**Agent 怎样部署、保存状态、恢复执行并交付用户结果。**

- **质量、安全与持续改进**
  - **[Agent 质量与安全]({{ site.baseurl }}/docs/interview/ai-agent/quality-production/)**：Prompt Injection、幻觉、任务结果、过程指标、安全门禁和优化证据
  - 质量先看业务任务是否完成和严重风险，再看工具路径、延迟、Token 与成本；Trace 用于找到第一次偏离。
  - 这一层回答：**怎样证明 Agent 正确、安全，并且新版本真的更好。**

## 贯穿 Agent 工程的核心关系

- **Prompt 与 Context**：Prompt 说明怎样做，Context 提供本轮做判断所需的信息；二者都不能代替权限与业务校验。
- **Conversation、State 与 Memory**：Conversation 保存交互，State 保存当前任务如何继续，Memory 保存跨任务仍值得复用的信息。
- **RAG 与工具**：RAG 取得外部证据，工具既可以查询也可以执行动作；返回内容进入 Context，但不自动成为权威业务状态。
- **Planning 与 Workflow**：Planning 允许模型根据现场情况调整计划，Workflow 用确定性节点守住审批、状态转换和高风险流程。
- **Runtime 与业务系统**：Runtime 管理模型和工具循环，领域系统掌握订单、邮件、支付等真实业务状态。
- **评测与可观测性**：评测定义什么叫正确，Metrics 发现整体变化，Trace 解释单次任务为什么偏离。
- **完成率与效率**：先保证任务完成和安全，再优化轮次、延迟、Token 与成本；快速失败不能算性能提升。
