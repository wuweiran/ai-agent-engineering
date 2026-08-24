---
layout: default
title: Copilot Evaluation 与 Golden Set
parent: 工作经历
nav_order: 4
has_children: true
permalink: /docs/career/copilot-evaluation/
---

# Copilot Evaluation 与 Golden Set

## 核心思路

Feature 开发需要持续判断改动是否有效、是否引入回归

→ 各 Feature 自建 Query 和测试邮件，数据重复分散，邮件变化后无法判断影响范围

→ 通过**共享 Golden Set**分开并集中维护 Query、Grounding Data 和 Assertion，用热点 Email 承载多条 Query

→ 开发者编写的理想 Utterance 不能代表真实用户输入

→ 通过**真实分布与边界覆盖**引入 eyes-off 脱敏 Utterance，同时保留低频高风险样例

→ 通过**Baseline / Candidate 配对与 Trace 归因**验证 Feature 效果并定位第一次偏离

→ 同一套评测资产支持 Feature 迭代、全量回归和发布门禁

**同类项目通常关注：** 任务质量（LM Checklist 任务通过率）、证据引用（Citation 正确率）、工具调用（Tool Call 正确率）、安全（严重安全违规率）、性能（端到端延迟 P95）、评测成本。

## 项目介绍

这个项目为 Outlook Copilot 建立可重复的 Agent 评测，支持 Feature 效果验证、回归定位和发布判断。微软内部 SEVAL 负责 Evaluation Job 的创建、调度和结果管理；Outlook 团队定义邮件业务的评测数据、成功标准和上线门禁。我负责持续维护 [Golden Set Query、Grounding Data 和 Assertion]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/#evaluation-data-definition)，并将 eyes-off 脱敏后的[真实用户 Utterance]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/#real-user-utterance)纳入回归，使评测既覆盖真实邮件 Context，也能长期复用邮件证据。

Feature 开发时，我先运行 Baseline，再用同一套资产评估 Candidate，按功能汇总 [LM Checklist、Citation、Tool Call 和 Safety]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/#evaluation-metrics)，判断改动是否有效。多轮任务使用固定脚本或 [User Simulator]({{ site.baseurl }}/docs/career/copilot-evaluation/multi-turn/#user-simulator)补充后续输入；出现失败时，从 Query 级结果进入 [Trace 定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/#trace-diagnosis)，找到 Context、模型、Tool Call、Tool Result 或评分中的第一次偏离。Feature 达标后再运行全量 Golden Set，并通过[绝对与相对质量门禁]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/#quality-gates)判断是否可以发布。

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
→ 按功能汇总任务通过率，配对比较 Baseline / Candidate，并沿 Trace 定位新增失败
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

## 项目文档

- [Golden Set 与 Grounding Data]({{ site.baseurl }}/docs/career/copilot-evaluation/golden-set/)：Query、邮件证据、真实 Utterance 和资产维护；
- [指标与结果分析]({{ site.baseurl }}/docs/career/copilot-evaluation/metrics/)：LM Checklist、核心 Metric 和结果判断；
- [多轮对话评测]({{ site.baseurl }}/docs/career/copilot-evaluation/multi-turn/)：固定脚本、User Simulator 和跨轮评分；
- [回归判断与问题定位]({{ site.baseurl }}/docs/career/copilot-evaluation/regression/)：版本配对、质量门禁和 Trace 定位。
