---
layout: default
title: 结果分析与业务闭环
grand_parent: 工作经历
parent: Copilot Bake-off
nav_order: 2
permalink: /docs/career/copilot-bakeoff/business-loop/
---

# 结果分析与业务闭环

## 结果有什么业务意义

Bake-off 的目的不是宣布 Outlook Copilot 或 Gmail Gemini 哪个产品整体更好。测试集已经按标签裁剪，只覆盖双方都有的功能，因此结果回答的是一个更具体的问题：**在相同邮件 Grounding、相同用户任务和相同 Assertion 下，两个产品对共同能力的完成情况有什么差异。**

结果主要支持三类业务判断：

1. **发现相对短板**：Gemini 通过而 Outlook 失败的 Query，说明 Outlook 在共同能力中存在明确差距；
2. **确认相对优势**：Outlook 通过而 Gemini 失败的 Query，说明现有 Context、工具或回答方式可能形成产品优势，需要避免后续版本破坏；
3. **判断行业基线**：双方都失败的 Query 说明任务本身可能仍然困难，也可能是 Assertion、Grounding 或产品形态都不适合，需要进一步分析，不能直接归因于 Outlook。

Bake-off 不评价 Gmail Gemini 没有的 Outlook 独有功能，也不能代替 Outlook 的线上指标、完整 Golden Set 和发布门禁。它提供的是共同能力上的外部参照，不是单一业务 KPI。

## 怎样阅读配对结果

结果按同一 Query 配对：

| 结果 | 业务含义 | 下一步 |
| --- | --- | --- |
| Outlook pass / Gemini fail | Outlook 相对优势 | 检查优势来自 Context、工具还是回答策略，并加入保护性回归 |
| Outlook fail / Gemini pass | Outlook 相对短板 | 进入优先分析队列，判断是否值得形成产品改进 |
| Both pass | 共同能力已达到要求 | 维持覆盖，关注延迟、稳定性和后续版本变化 |
| Both fail | 双方都未满足 Assertion | 检查任务难度、数据和 Assertion，再决定是否建设能力 |

不能只看一个总胜率。报告会按功能标签、Query 类型和失败 Assertion 切片，例如邮件总结、事实查找、行动项和多轮追问。一个产品可能总分接近，但在高频场景上存在集中差距。

Scraping、Mapping 或 ingestion 失败不进入上述四类产品结果，而是单独报告为 Bake-off 系统健康度。否则 Playwright 页面故障会被错误解释为 Gemini 产品失败。

## 从差异到产品问题

对 `Outlook fail / Gemini pass` 的 Query，先人工复核 Grounding Data、两侧 Response 和 LM Checklist 结果，排除三类伪差异：

- 两边实际看到的邮件、用户关系或线程不等价；
- scraping 提取了不完整 Response；
- Assertion 或 LM Checklist 误判。

确认差异真实后，再判断 Outlook 第一次偏离发生在哪一层：

```text
当前邮件或搜索 Context
→ Agent Instructions 与任务理解
→ 工具选择和参数
→ Tool Result
→ 最终生成和 Citation
```

差异不会统一归为“模型不如 Gemini”。例如：

- 缺少关键邮件证据，属于 Context 或检索问题；
- 证据完整但答案漏掉行动项，属于 Instructions 或生成问题；
- 调用了错误工具，属于工具 Description 或选择问题；
- 回答正确但引用错误，属于 Citation 处理问题；
- Outlook 产品根本没有同等入口，则不应进入 Bake-off 子集。

每个确认问题需要记录功能标签、影响 Query、失败 Assertion、根因层、Owner 和建议动作，才能进入产品排期，而不是只留在对比报告里。

## 优先级怎样判断

不是所有竞品差距都立即修复。优先级结合：

- 该功能标签在真实脱敏 Utterance 中的出现频率；
- 失败是否影响任务完成，而只是表达风格差异；
- 是否涉及 Grounding、Citation 或安全等高风险问题；
- 是否能通过 Prompt、Context 或工具配置低成本修复；
- 是否符合 Outlook Copilot 的产品方向；
- 改动是否可能损害其他功能。

高频且影响任务完成的共同能力差距优先级最高。纯文风差异、低频场景或不符合产品方向的行为，可以记录但不进入近期开发计划。

## 怎样形成闭环

确认要修复的问题会回到正常的 Copilot Evaluation 和发布流程：

```text
Bake-off 配对差异
→ 排除数据、scraping 和评分问题
→ 定位 Outlook 第一次偏离
→ 确认业务价值与 Owner
→ 将 Query 加入或标记到常规 Outlook Golden Set
→ 修改 Prompt、Context、工具或产品能力
→ 功能切片回归
→ 全量 Golden Set 回归
→ 质量门禁与 Ring 发布
→ 下次 Bake-off 复测
```

Bake-off Query 本来来自 Outlook Golden Set 的标签子集，但确认差距后仍要维护对应 Assertion 和保护性标签，使这个问题进入 Outlook 的长期回归。不能只在下一次竞品对比中观察，因为 Bake-off 不是每次发布都运行。

修复通过常规 Copilot Evaluation 后才进入发布。下一次 full run 再用相同 Query 配对，检查结果是否从：

```text
Outlook fail / Gemini pass
```

变为：

```text
Both pass
```

同时检查原有的 `Outlook pass / Gemini fail` 优势 Query 是否保持，避免追赶一个差距时损害其他能力。

## 报告怎样服务决策

阶段性报告包括：

- 本次共同能力范围和裁剪后 Query 数量；
- 四类配对结果及其按功能标签的分布；
- 主要 `Outlook fail / Gemini pass` 聚类；
- Outlook 相对优势 Query；
- scraping、ingestion 和 Mapping 失败率；
- 相对上一次 Bake-off 新增、修复和持续存在的差距；
- 已确认问题的 Owner、状态和验证结果。

这样报告既能支持产品团队确定改进方向，也能让工程团队追踪问题是否真正进入 Golden Set、完成修复并通过回归。最终闭环的标准不是报告发布，而是高价值差距有明确决策：修复、延期或基于产品边界不处理，并且理由可以追溯。
