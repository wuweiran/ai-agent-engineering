---
layout: default
title: Golden Set 与 Grounding Data
grand_parent: 工作经历
parent: Copilot Evaluation 与 Golden Set
nav_order: 1
permalink: /docs/career/copilot-evaluation/golden-set/
---

# Golden Set 与 Grounding Data

## 数据定义

在这个项目中，Golden Set 通常指一组由 **CIQ 和用户输入 Utterance** 组成的评测输入，每一条称为一条 **Query**：

```text
Query
├─ CIQ：当前邮件等 Context 入口
└─ Utterance：用户向 Agent 提出的任务
```

CIQ 是 Outlook 当前邮件 Context 的实现方式。Current Email ID 随 Utterance 一起进入 Agent SDK，Copilot Runtime 再根据 Email ID 读取当前邮件：

```text
Golden Set Query：CIQ 中的 Current Email ID + Utterance
→ Agent SDK
→ Copilot Runtime 读取当前邮件
→ Agent Result 与 Trace
```

Golden Set 保存“从什么邮件场景发起什么用户请求”，不把标准答案、Assertion 和全部邮件内容混在 Query 里。CIQ 也不经过 Insight Service：前者已经知道当前 Email ID，由 Runtime 直接取得邮件；后者用于开放式邮件搜索，需要根据查询条件召回候选。

Grounding Data 是这些 Query 背后使用的邮件集合。邮件内容、对象 ID 和邮件之间的关系由 Grounding Data 维护；一封邮件可以被多条 Query 复用，一条 Query 也通过 CIQ 关联到对应邮件。

Assertion 是与 Query 配套的评测规则，描述 `lm_checklist` 要检查的事实和禁止项。三者的关系是：

```text
Golden Set Query：CIQ + Utterance
        ↓ 关联
Grounding Data：Agent 实际读取的邮件集合
        ↓ 提供证据
Assertion：根据邮件证据判断 Agent Result
```

单轮评测以 Query 为单位。需要验证追问、澄清或约束修改时，多条 Turn 进一步组织成一个 Scenario，并复用同一 Conversation。Scenario 可以保存固定的后续 Utterance，也可以为动态澄清场景配置 User Simulator；具体执行方式见[多轮对话评测]({{ site.baseurl }}/docs/career/copilot-evaluation/multi-turn/)。

## 最初的维护方式

最初没有统一的 Golden Set。每个功能开发者在开发新能力时，同时增加一批测试 Query 和对应邮件。例如开发附件总结、邮件问答或其他能力时，各自在自己的测试范围内准备 Utterance、CIQ 和邮件。

这种方式在功能较少时交付很快，但随着能力增加暴露出两个问题：

- 邮件集合持续膨胀，同类邮件被不同功能重复准备；
- Query、邮件和 Assertion 分散在功能代码或个人维护范围内，邮件变化后很难判断会影响哪些测试。

继续由每个功能开发者单独加数据，会让评测资产和功能实现绑定，既难复用，也难做跨版本的统一回归。

## 集中建设 Golden Set

后来把分散的 Query 和邮件收敛为统一维护的 Golden Set 与 Grounding Data：

- Golden Set 统一保存 CIQ 与 Utterance；
- Grounding Data 统一保存可复用的邮件集合；
- Query 通过稳定的邮件标识关联 Grounding Data；
- Assertion 与 Query 对齐，单独维护预期事实和禁止项；
- 功能开发者仍可提交新场景，但不再各自拥有一套孤立的测试邮件。

集中维护后，新功能优先复用已有邮件，再补充确实缺少的邮件类型。修改或删除邮件前先检查它关联的 Query 和 Assertion，避免修复一个场景却让其他回归测试失效。

## 引入真实用户 Utterance

统一数据集解决了复用和维护问题，但已有 Grounding Data 与开发者编写的 Utterance 仍不能充分反映真实用户分布。开发者容易围绕功能设计“理想问题”，线上用户则会使用更短、更模糊或更依赖当前邮件上下文的表达。

真实用户输入通过内部脱敏平台处理。平台使用 LLM，在 **eyes-off** 条件下识别并去除姓名、邮箱地址、组织、账号和其他个人隐私敏感信息，数据维护人员不直接查看原始用户输入。只有完成隐私处理的 Utterance 才能进入后续筛选和 Golden Set 维护流程。

```text
真实用户 Utterance
→ 内部 LLM 脱敏平台（eyes-off）
→ 去除个人隐私敏感信息
→ 筛选脱敏后的问题模式
→ 对齐 CIQ、Grounding Data 与 Assertion
→ 加入 Golden Set
```

Grounding Data 的选择优先使用 **热点 Email**，即脱敏后的真实输入中能够关联较多 Query 的邮件。围绕同一封热点 Email，可以覆盖总结、事实查找、行动项、解释和追问等多种 Utterance。这样既能用较少的邮件支持更多 Query，提高 Grounding Data 的复用率，也更接近真实用户集中询问的邮件类型。

热点优先不代表只保留高频场景。Golden Set 仍需要补充低频但高风险的边界 Query；热点 Email 主要解决真实分布和数据复用问题，边界样例负责防止严重能力退化。

这一步改变的不是 SEVAL 的 Job 执行方式，而是 Golden Set 的 Query 分布。数据集从“开发者验证自己功能的测试用例”逐步变成能够覆盖真实邮件任务表达的共享回归资产。内部平台负责 LLM 脱敏能力，我负责使用脱敏结果维护 Query、Grounding Data 和 Assertion，不实现脱敏平台本身。

## 我的职责

我负责 Golden Set 以及对应 Grounding Data 和 Assertion 的维护，具体包括：

- 维护由 CIQ 与 Utterance 组成的 Query；
- 整理 Query 与邮件集合之间的关联；
- 合并重复邮件和重复 Query，优先使用能够承载较多 Query 的热点 Email；
- 使用内部平台在 eyes-off 条件下产生的脱敏结果，将真实用户 Utterance 纳入 Golden Set；
- 根据邮件证据维护每条 Query 对应的 Assertion；
- 在邮件、Query 或 Assertion 变化时保持三者一致。

SEVAL 的 Job 调度和运行管理由平台负责，Metric 的通用执行逻辑也不属于我的实现范围。我的工作重点是保证输入给平台的 Query、邮件证据和评测规则长期可信，否则 Job 即使稳定运行，得到的分数也不能代表真实产品质量。

## 维护原则

一次变更先判断影响的是哪类资产：

- 用户表达或 CIQ 变化，修改 Golden Set Query；
- 邮件内容或对象关系变化，修改 Grounding Data；
- 预期事实或禁止项变化，修改 Assertion；
- Agent、模型或 Prompt 变化，不修改评测资产，只创建新的 Evaluation Job 进行对比。

这条边界保证回归结果可解释。不能为了让新版本通过而直接改 Assertion，也不能在不记录数据变化的情况下替换邮件，否则无法判断分数变化来自 Agent，还是来自评测集本身。
