---
layout: default
title: Copilot Evaluation 与 Golden Set
parent: 工作经历
nav_order: 4
has_children: true
permalink: /docs/career/copilot-evaluation/
---

# Copilot Evaluation 与 Golden Set

## 项目是什么

2025 年 6 月开始参与建立 Golden Set 和回归评测流程，最初与 Copilot Agent 功能开发并行，2025 年 10 月后成为主要工作。模型、Prompt、Context、内容提取和工具版本都会影响结果，开发人员手工试几个邮箱无法判断一次修改是否真正改善。

微软内部有统一的 AI 评测中台 **SEVAL**，不同 AI 业务的评测都在这个平台上运行。SEVAL 负责 Evaluation Job 的创建、调度、运行状态和结果管理；我们不重复建设评测任务平台，而是负责 Outlook 邮件业务的 Grounding Data、Golden Set、执行配置、定制指标和失败分析。

## 系统边界

```text
SEVAL Evaluation Job
├─ 待测 Agent、模型与配置版本
├─ 邮件业务 Grounding Data
├─ Golden Set
└─ 通用与业务定制 Metrics
        ↓
Agent SDK → Copilot Runtime → Agent Result / Trace
        ↓
LM Checklist + 确定性检查 + 邮件业务指标
        ↓
结果保存、基线比较与回归分析
```

SEVAL 解决的是所有业务共用的 Job 管理问题，项目团队解决的是“邮件 Agent 的成功应该怎样定义”。两者的职责不能混在一起：平台能稳定地把任务跑起来，但不知道某封邮件中的哪些事实必须出现、哪些结论不能生成，也不知道某个邮件场景应该采用什么 Context 链路和质量指标。

## Golden Set 与 Grounding Data

在这个项目中，Golden Set 指由 **CIQ 和用户输入 Utterance** 组成的评测输入集合，每一条称为一条 **Query**。Grounding Data 是这些 Query 背后用到的邮件集合，Assertion 则定义 `lm_checklist` 应该依据邮件证据检查什么。

```text
Golden Set Query：CIQ + Utterance
→ Grounding Data：关联的邮件集合
→ Assertion：必须满足的事实和禁止项
```

最初由每个功能开发者在开发新能力时各自添加测试 Query 和邮件。随着邮件越来越多，数据难以复用和维护，团队才集中建设统一的 Golden Set 与 Grounding Data。之后又发现开发者构造的输入不能反映真实用户分布，于是通过内部 LLM 脱敏平台，在 eyes-off 条件下去除个人隐私敏感信息，再将真实 Utterance 纳入 Golden Set。Grounding Data 优先使用能够关联较多 Query 的热点 Email，以较少邮件覆盖更多真实问题。

我负责 Golden Set 以及对应 Grounding Data 和 Assertion 的维护，保证 Query、邮件证据和评分规则保持一致。具体演进与维护方式见[Golden Set 与 Grounding Data]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/)。

## Metric

SEVAL 提供通用 Metric，邮件业务再接入定制 Metric。主回归关注 `lm_checklist`、`citation`、`tool_call`、`safety` 和 `latency_p95`；其中 `lm_checklist` 配合 Query 对应的 Assertion 自动判断任务结果，CIQ Context 问题通过 Email ID 和 Runtime Trace 归因，不虚构单独的 Context Metric。

各指标的当前值、计算方式以及 LM Checklist 与 Assertion 的执行方式见[指标与结果分析]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/)。

## 主要工作

我主要负责 Golden Set 及其对应的 Grounding Data 和 Assertion：

- 维护由 CIQ 与 Utterance 组成的 Query；
- 整理并复用 Query 背后的邮件集合；
- 合并功能开发阶段分散维护的测试数据；
- 使用内部平台生成的 eyes-off 脱敏结果，将真实用户 Utterance 纳入 Golden Set；
- 优先复用能够关联较多 Query 的热点 Email；
- 根据邮件证据编写和更新 Assertion；
- 保证 Query、Grounding Data 与 Assertion 的变更一致性。

SEVAL 的 Job 管理由平台承担，通用 Metric 和各功能实现也不属于我的主要职责。

## 这个项目可以深入到哪里

- **评测任务怎样运行**：SEVAL Job 如何组合被测版本、Golden Set Query、Grounding Data、Assertion 和 Metrics；
- **Golden Set 怎样演进**：功能开发者分散添加的 Query 和邮件，为什么需要收敛成统一维护的数据集；
- **真实用户分布怎样进入回归集**：内部平台怎样通过 LLM 以 eyes-off 方式脱敏真实 Utterance，以及为什么优先选择能关联较多 Query 的热点 Email；
- **三类资产怎样保持一致**：Query、Grounding Data 与 Assertion 发生变化时，怎样控制影响范围和避免评测集失真；
- **多轮输入怎样评测**：怎样复用同一个 Conversation，通过固定脚本和 User Simulator 生成后续 Utterance，并检查跨轮约束与最终任务结果；
- **Metric 怎样设计**：邮件业务的确定性 Metric 与 LM Checklist 怎样分工，Assertion 怎样表达必须覆盖的事实和禁止生成的结论；
- **回归怎样判断和定位**：怎样通过功能切片、全量 Golden Set、Query 级配对和质量门禁确认无回归，再沿 Trace 找到候选版本第一次偏离。

## 项目文档

- [Golden Set 与 Grounding Data]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/)：Query 和 CIQ 定义、功能分散维护、集中建设、真实 Utterance 脱敏和三类资产的一致性；
- [指标与结果分析]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/)：LM Checklist、核心 Metric 的当前值、计算方式和发布判断；
- [多轮对话评测]({{ site.baseurl }}/docs/career/copilot-evaluation/multi-turn/)：同一 Conversation、固定后续 Utterance、User Simulator、停止条件和跨轮评分；
- [回归判断与问题定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/)：SEVAL Job、功能切片、全量回归、质量门禁、Query 配对、重复运行和 Trace 定位。
