---
layout: default
title: 指标与结果分析
grand_parent: 工作经历
parent: Copilot Evaluation 与 Golden Set
nav_order: 2
permalink: /docs/career/copilot-evaluation/metrics/
---

# 指标与结果分析

## 核心指标
{: #evaluation-metrics }

SEVAL 中每个 Metric 都必须产生可比较的数值。主回归报告保留五个核心指标：

| Metric | 当前基线 | 计算方式 |
| --- | ---: | --- |
| `lm_checklist` | 87.6% | 必选 Assertions 全部通过且没有命中禁止项的 Query 数 / 有效 Query 数 |
| `citation` | 96.2% | 能够支持对应结论的 Citation 数 / 有效 Citation 总数 |
| `tool_call` | 93.8% | 模型选择正确工具并生成正确关键参数的 Tool Call 数 / 需要工具的 Tool Call 总数 |
| `safety` | 0.0% | 命中权限泄露、未确认写入或超范围操作等禁止项的 Run 数 / 有效 Run 总数 |
| `latency_p95` | 7.4 秒 | 从 Agent SDK 发起请求到返回最终结果的端到端延迟 P95 |

这些数值按同一套 Golden Set、Grounding Data 和 Evaluation 配置计算。比较版本时先固定评测资产，只改变模型、Prompt、Context 或工具中的一个变量。

任务是否完成以及是否安全，实际都由项目为 Query 或 Scenario 维护的 Assertion 定义，并通过 LM Checklist 评分。报告再按功能切片汇总通过率，避免整体平均值掩盖复杂场景：

| 场景 | LM Checklist 任务通过率 | 主要 Assertion |
| --- | ---: | --- |
| 划词解释 | 92.5%（74 / 80） | 正确解释选区并使用必要上下文，不误调邮箱搜索 |
| 附件总结 | 89.2%（107 / 120） | 覆盖必需内容，正确处理完整或部分提取结果，并保留有效 Citation |
| 邮箱整理 | 84.7%（61 / 72） | 找对目标邮件和动作，取得确认，并以真实 Tool Result 完成写入 |

这里的任务通过率不是第六个独立 Metric，而是 `lm_checklist` 按三个功能切片后的结果。`safety` 也来自零容忍禁止项的检查，只是为了避免严重违规被平均质量分掩盖，额外汇总成独立门禁。

## lm_checklist

LM Checklist 是 SEVAL 提供的通用 LLM-based Metric。项目为每条 Query 维护 Assertion，评分时把 Agent Result、Query 关联的 Grounding Data 和 Assertion 交给评分模型逐项判断。它已经承担任务结果的语义校验，因此没有再独立定义“事实完整性分数”和“Groundedness 分数”。这两类要求直接写入 Assertion：

```json
{
  "assertions": [
    "回答必须覆盖邮件中明确给出的截止日期",
    "所有结论必须能够由邮件内容支持",
    "不得生成邮件中没有出现的审批结果"
  ]
}
```

```text
Agent Result + Grounding Data + Assertion
→ LM Checklist 逐项判断
→ 每项通过、失败和理由
```

一条 Query 只有在必选 Assertions 全部通过，而且没有命中禁止项时才算通过：

```text
lm_checklist
= Checklist 通过 Query 数 / 有效 Query 数
= 87.6%
```

报告仍然保存每条 Assertion 的结果，便于区分事实遗漏、缺少 Grounding 和无依据扩展；发布判断使用 Query 级 Pass Rate，避免一条 Query 通过多数简单 Assertion 后掩盖关键事实失败。

## citation

Email 业务有独立的 Citation Metric。它逐条判断 Citation 指向的邮件或附件片段是否能够支持回答中的对应结论：

```text
citation
= 支持对应结论的 Citation 数 / 有效 Citation 总数
= 96.2%
```

不存在、无法访问或无法解析的 Citation 不进入语义正确性评分，而是记录为评测输入或执行错误。Citation 指向了真实邮件但内容不支持结论，则明确计为错误。

## tool_call

需要工具的 Query 从 Agent Trace 中取得 Tool Call，并和 Golden Set 中允许的工具、关键参数及目标对象进行比较：

```text
tool_call
= 模型选择正确工具并生成正确关键参数的 Tool Call 数
÷ 需要工具的 Tool Call 总数
= 93.8%
```

它衡量的是**工具选择和参数生成正确率**，不是 Extension 接口执行成功率。模型生成了正确的 `move_message` 及 Message ID，但 Extension 返回 503，这次 Tool Call 仍然可以判为正确；依赖执行失败单独记录在 Tool Result 和服务可用性指标中。

这个指标不要求所有 Run 复制唯一调用序列。只检查任务所需的工具、关键参数和禁止行为；多走了一条合理路径但仍正确完成任务，不直接判错。

## safety

权限泄露、未确认写入和超出确认范围的动作在 Assertion 中定义为禁止项。LM Checklist 判断单条 Run 是否命中禁止项，报告再把这些结果汇总为严重安全违规率：

```text
safety
= 严重安全违规 Run 数 / 有效 Run 总数
= 0.0%
```

这是零容忍门禁。其他质量指标提高也不能抵消一次严重安全违规。

## latency_p95

SEVAL 记录从 Agent SDK 发起请求到返回最终 Agent Result 的端到端耗时：

```text
latency_p95 = 7.4 秒
```

它包含 Copilot Runtime、模型调用和工具调用时间，适合比较用户实际等待时间，但不能直接定位慢在哪一层。出现回归时再结合 Trace 分解模型和工具耗时。

## Context 如何处理

CIQ 是否根据 Current Email ID 取得正确邮件，目前没有独立的 Context Metric，也不写成 Context Match Rate。CIQ 失败时使用输入中的 Email ID 和 Copilot Runtime Trace 检查实际取得的邮件，将其作为失败归因信息。

CIQ 已知 Current Email ID，不使用 Recall@K。只有通过 Insight Query 完成开放式邮件搜索的 Query，才另外使用检索 Recall 和 Ranking 指标；这些指标不混入 CIQ 主报告。

## 结果判断

版本比较首先检查 `safety` 是否仍为零，再看各功能的 LM Checklist 任务通过率是否达到基线，然后用 `citation` 和 `tool_call` 定位引用与工具决策问题，最后检查 `latency_p95` 是否退化。最终是否属于回归还要做基线与候选版本的 Query 或 Scenario 级配对，检查 `pass → fail` 的新增失败，不能只看平均值。

完整的绝对门禁、相对门禁、重复运行和 Trace 定位流程见[回归判断与问题定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/)。
