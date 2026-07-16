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

## 怎样量化上线后的 Agent？除了准确率还看什么？
{: #online-agent-metrics }

指标可以分四层：

- **用户结果**：任务完成率、成功交付率、人工接管率、重试率和放弃率；
- **严重失败**：越权、错误写入、重复副作用和虚假完成，必须单独统计，不能被平均分掩盖；
- **执行过程**：工具选择率、参数正确率、故障恢复率和平均步骤数；
- **效率体验**：首个有效反馈时间、P95/P99 完整耗时、每个成功任务的 Token 与成本。

用户满意度和投诉可以补充观察，但不能替代可核验的业务结果。指标还要按任务类型和风险分层，并能回到具体 Trace。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)。

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
