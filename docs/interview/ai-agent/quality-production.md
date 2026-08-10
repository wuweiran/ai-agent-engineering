---
layout: default
title: Agent 质量、安全与生产运行面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 6
permalink: /docs/interview/ai-agent/quality-production/
---

# Agent 质量、安全与生产运行面试题

这些问题讨论模型之外的 Harness、安全边界、评测闭环、缓存和任务进度交付。

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

相关内容：[Agent Harness](#agent-harness)、[Agent 状态与记忆]({{ site.baseurl }}/docs/interview/ai-agent/context-memory/#agent-state-and-memory)、[工具调用后端链路]({{ site.baseurl }}/docs/interview/backend/llm-application-backend/#tool-call-backend-flow)、[Agent 评测](#agent-evaluation)。

## Prompt Injection 怎样防御？
{: #prompt-injection-defense }

Prompt Injection 的根因是：**不可信网页、文档、用户输入和 Tool Result 也会进入 Context，并可能伪装成指令。**输入过滤只能拦截已知模式，不能成为唯一边界。

防御要形成纵深体系：

1. **区分权威层级**：把外部内容标记为数据，按来源、权限和租户先过滤；
2. **缩小能力面**：只暴露当前任务需要的工具，使用白名单和最小权限；
3. **执行时校验**：重新鉴权，校验参数、业务状态和数据范围；
4. **控制高风险动作**：使用沙箱、用户确认或人工审批；
5. **检查输出与外传**：防止敏感数据被带到未授权目标。

目标不是保证模型永远不受影响，而是**即使模型被误导，确定性软件仍不能允许越权。**

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## 什么是幻觉？怎样减少？
{: #hallucination }

**幻觉是模型生成形式可信、但事实、推理或引用缺乏可靠依据的内容。**知识缺口、过期资料、冲突 Context 和多步错误传播都会增加风险。

降低方法包括：

- 使用可信检索和实时工具取得事实；
- 保留来源、版本和观察时间；
- 缩小模型承担的判断范围；
- 对金额、权限和业务状态做外部校验；
- 加入无答案、冲突资料和错误引用的评测样本。

RAG 和模型自检只能降低风险，不能保证消除幻觉。

相关内容：[大模型幻觉]({{ site.baseurl }}/docs/llm/hallucination/)。

## 怎样评估 Agent 的执行效果？
{: #agent-evaluation }

按三层评估：

1. **最终结果**：用户目标是否端到端完成；
2. **关键过程**：是否选对工具和参数，是否越权、重复写入、提前结束，故障后能否恢复；
3. **运行效率**：延迟、Token、成本、步骤数和工具调用数。

使用固定的真实任务比较版本，失败时通过 Trace 找到第一次偏离。结构、数量和状态等确定结果交给程序判断；开放语义再由人工或经过校准的模型评审。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)。

## 如何评估一个 AI Agent？有哪些可量化指标？
{: #online-agent-metrics }

指标可以分四层：

- **用户结果**：任务完成率、成功交付率、人工接管率、重试率和放弃率；
- **严重失败**：越权、错误写入、重复副作用和虚假完成，必须单独统计，不能被平均分掩盖；
- **执行过程**：工具选择率、参数正确率、故障恢复率和平均步骤数；
- **效率体验**：首个有效反馈时间、P95/P99 完整耗时、每个成功任务的 Token 与成本。

用户满意度和投诉可以补充观察，但不能替代可核验的业务结果。指标还要按任务类型和风险分层，并能回到具体 Trace。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)。

## 相关性、完整性和一致性分别衡量什么？
{: #relevance-completeness-consistency }

它们是开放语义结果的三个不同质量维度：

- **相关性（Relevance）**：回答是否针对用户当前问题，检索证据是否与 Query 相关；内容真实但没有回答问题，相关性仍然低；
- **完整性（Completeness）**：用户目标中的关键要求、子问题和必要事实是否都被覆盖；回答每句话都正确，也可能遗漏一半任务；
- **一致性（Consistency）**：回答内部是否自相矛盾，多轮回答是否遵守已确认约束，结果是否与工具状态和权威数据一致。

评测时不能只给一个模糊总分。相关性可以基于 Query—回答或 Query—Chunk 评分；完整性适合把要求拆成 Checklist 或 Assertion 后计算覆盖率；一致性要加入冲突资料、用户改口和多轮 Scenario，并用程序核对结构化状态与工具结果。三者都不能替代 Faithfulness：回答可能相关、完整且内部一致，却仍然没有证据支持。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)、[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## 怎样证明一次 Agent 优化真的有效？
{: #agent-optimization-evidence }

需要形成完整证据链：

1. **建立基线**：固定评测集、环境和关键指标；
2. **贴近根因修改**：根据第一次偏离，调整 Context、Skill、工具、模型或业务服务；
3. **分层回归**：重跑失败样本、相邻任务和完整回归集，必要时做消融；
4. **逐步上线**：小流量观察用户完成率、严重失败、延迟和成本；
5. **形成闭环**：把新的生产失败脱敏后加入评测集。

只调整 Prompt、只看几个演示或只比较平均总分，都不足以证明优化有效。

相关内容：[Agent 质量改进]({{ site.baseurl }}/docs/ai-agent/agent-quality/improvement/)、[Agent 发布与升级]({{ site.baseurl }}/docs/ai-agent/agent-production/release-upgrade/)。

## 怎样使用缓存优化大模型应用？
{: #llm-application-cache }

三类缓存解决不同问题：

- **精确结果缓存**：输入完全相同时复用最终结果；
- **语义缓存**：在低风险、允许陈旧的场景复用相似问题的答案；
- **Prompt Cache**：复用稳定提示词前缀的模型计算，但仍会执行本次生成。

缓存键至少包含模型、Prompt、工具定义、权限和关键 Context 版本，并明确 TTL 与失效条件。动态业务状态、高风险回答和用户隔离内容不能只按问题文本复用。

相关内容：[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

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
