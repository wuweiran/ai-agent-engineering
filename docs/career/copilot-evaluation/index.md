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

2025 年 6 月开始参与 Outlook Copilot 评测，最初与 Agent 功能开发并行，2025 年 10 月后成为主要工作。微软内部的 **SEVAL** 负责 Evaluation Job 的创建、调度和结果管理；Outlook 团队负责准备邮件业务的评测数据、评分规则和发布判断。

我负责持续维护 Query Set、Grounding Data 和 Assertion，使用这些资产比较生产版本与候选版本，并通过 Trace 定位失败。

## Outlook 评测有什么不同

其他 AI 业务也会有 Query、Grounding Data 和 Assertion。Outlook 的差异在于，一次请求是否正确高度依赖**用户当时打开的邮件及其邮箱权限**，不是只看 Utterance 和参考答案。

```text
Golden Set Query = CIQ 中的 Current Email ID + Utterance
Grounding Data = 受控评测邮箱中的邮件和附件
Assertion = 针对该邮件证据的必须覆盖事实和禁止项
```

运行时，SEVAL 将 Email ID 和 Utterance 发送给 Agent SDK，Copilot Runtime 再从评测邮箱读取邮件。这使评测覆盖的是 Outlook 的真实 Context 链路，也带来三个业务特有问题：

- 同一封邮件要支持总结、事实查找、行动项和多轮追问，邮件与 Query 需要复用和关联管理；
- 真实用户 Utterance 包含隐私，只能使用内部平台在 eyes-off 条件下完成脱敏后的结果；
- 失败可能发生在 CIQ、Runtime Context、模型、Extension 或 Metric，不能统一归因成回答错误。

## 评测框架

框架只有四步：

```text
Query Set + Grounding Data + Assertion
→ SEVAL 运行固定版本的评测资产和待测 Agent
→ lm_checklist、citation、tool_call、safety、latency_p95
→ Baseline / Candidate 配对比较，并沿 Trace 定位新增失败
```

Query、邮件或 Assertion 变化时发布新的资产版本；比较 Agent、模型、Prompt、Context 或工具版本时固定评测资产。功能切片先验证当前修改，全量 Golden Set 再检查跨功能回归。

## 除了得到评测分数，还解决了什么

### 测试邮件复用

早期每个功能开发者各自准备 Query 和邮件，同类邮件重复、变更影响也难以追踪。集中维护后，一封热点 Email 可以承载多条 Query，新增能力优先复用已有 Grounding Data，修改邮件时也能找到受影响的 Query 和 Assertion。

### 真实用户分布

开发者构造的通常是表达完整的理想问题，真实用户输入更短、更模糊，也更依赖当前邮件。将 eyes-off 脱敏后的真实 Utterance 纳入 Query Set，使回归集能够反映产品中的常见表达，而不要求维护人员查看原始用户输入。

### 发布门禁

生产版本和 SDF 版本使用同一套资产按 Query 配对，重点检查 `pass → fail`。门禁同时考虑绝对基线、相对变化和严重安全失败，避免整体平均分掩盖某项能力回归或低频高风险问题。具体指标和阈值见[指标与结果分析]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/)与[回归判断与问题定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/)。

### 跨团队问题归属

Trace 将 Email ID、Runtime Context、模型决策、Tool Call、Tool Result 和评分结果串起来。团队可以判断问题应由评测数据、CIQ、Agent、Extension 还是 Metric 的 Owner 处理，减少只凭最终回答反复猜测根因。

所以这个项目没有替代功能团队开发 Copilot 能力；它提供了各功能共同使用的邮件测试资产、真实输入分布、发布标准和失败归因方式。

## 主要工作

我负责 Query Set、Grounding Data 和 Assertion 的持续维护，引入 eyes-off 脱敏后的真实 Utterance，维护 Metric、Baseline/Candidate 回归、质量门禁和 Trace 失败定位。统一 Golden Set 的初次整合由团队完成，SEVAL 平台和各项 Copilot 功能实现不属于我的交付。

## 项目文档

- [Golden Set 与 Grounding Data]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/)：Query、邮件证据、真实 Utterance 和资产维护；
- [指标与结果分析]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/)：LM Checklist、核心 Metric 和结果判断；
- [多轮对话评测]({{ site.baseurl }}/docs/career/copilot-evaluation/multi-turn/)：固定脚本、User Simulator 和跨轮评分；
- [回归判断与问题定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/)：版本配对、质量门禁和 Trace 定位。
