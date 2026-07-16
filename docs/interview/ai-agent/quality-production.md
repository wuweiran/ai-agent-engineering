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

Agent Harness 是包围模型的执行与控制层，也常称为 Runtime。它负责组装 Context、暴露和执行工具、实施权限、保存状态、限制预算、处理等待和恢复，并记录 Trace。

Harness 让模型的概率判断进入确定的软件边界。这个术语没有唯一实现清单，重点是模型之外的控制责任。

相关内容：[Agent 任务与运行循环]({{ site.baseurl }}/docs/ai-agent/agent-runtime/)、[Agent 实现方式]({{ site.baseurl }}/docs/ai-agent/agent-building/implementation-choice/)。

## Prompt Injection 怎样防御？
{: #prompt-injection-defense }

Prompt Injection 的根因是不可信网页、文档、用户输入或工具结果也会进入模型 Context，并试图伪装成指令。输入过滤可以拦截已知模式，却无法可靠识别所有自然语言攻击，因此不能作为唯一边界。

系统应把外部内容标记为数据，按来源和权限先过滤；只暴露当前任务所需工具，并在执行层重新鉴权、校验参数和业务状态；敏感工具使用白名单、最小权限、沙箱或人工确认；输出和数据外传也要检查。即使模型受注入影响，确定性软件仍不能允许它越权。

相关内容：[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

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

## 怎样量化上线后的 Agent？除了准确率还看什么？
{: #online-agent-metrics }

线上首先看用户目标是否真正完成，例如任务完成率、成功交付率、人工接管率、用户重试和放弃率，而不只看一次回答是否“准确”。高风险场景还要单独统计越权、错误写入、重复副作用和虚假完成等严重失败，不能被平均分掩盖。

过程与运行指标用于解释结果：工具选择和参数正确率、失败恢复率、平均步骤数、P95/P99 完整耗时、首个有效反馈时间、每个成功任务的 Token 与成本。用户满意度和投诉可以补充观察，但会受界面和期望影响，不能替代可核验的业务结果。指标应按任务类型和风险分层，并结合 Trace 与抽样审计。

相关内容：[Agent 评测]({{ site.baseurl }}/docs/ai-agent/agent-quality/evaluation/)。

## 怎样证明一次 Agent 优化真的有效？
{: #agent-optimization-evidence }

先建立评测基线，再修改最接近根因的层次，例如 Context、Skill、工具或业务服务。重跑失败样本、相邻任务和完整回归集，必要时关闭新机制做消融实验。

离线通过后逐步放量，并观察用户完成率和严重失败。只调整 Prompt、只看几个演示或只比较平均总分都不足以证明改善。

相关内容：[Agent 质量改进]({{ site.baseurl }}/docs/ai-agent/agent-quality/improvement/)、[Agent 发布与升级]({{ site.baseurl }}/docs/ai-agent/agent-production/release-upgrade/)。

## 怎样使用缓存优化大模型应用？
{: #llm-application-cache }

可以缓存完全相同请求的最终结果，也可以在业务允许时复用语义相近问题的答案；供应商 Prompt Cache 则复用稳定提示词前缀的计算。

缓存键必须包含模型、Prompt、工具定义和关键 Context 版本，并定义失效与权限边界。动态业务状态、高风险回答和用户隔离内容不能只按问题文本复用。

相关内容：[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## 怎样使用 SSE 推送 Agent 进度？
{: #agent-sse-events }

服务端返回 `text/event-stream`，持续发送以空行分隔的事件。应用可以定义文本增量、工具开始、工具结束、等待确认和任务完成等事件，并携带任务 ID 与序号。

SSE 是服务端到客户端的单向通道；客户端提交命令仍可使用普通 HTTP。断线不等于取消，重连应从持久任务状态或事件位置恢复。

相关内容：[流式生成与任务事件]({{ site.baseurl }}/docs/backend/llm-backend/streaming-generation/)。
