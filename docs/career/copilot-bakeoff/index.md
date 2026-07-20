---
layout: default
title: Copilot Bake-off
parent: 工作经历
nav_order: 5
has_children: true
permalink: /docs/career/copilot-bakeoff/
---

# Copilot Bake-off

## 项目是什么

2026 年 3 月开始参与 Copilot Bake-off，作为 Copilot Evaluation 主线下的专项对比工作。项目只比较 Outlook Copilot 和 Gmail Gemini **双方都具备的功能**：先通过 Bake-off 标签从 Outlook Golden Set 中筛选共同能力 Query，再复用这些 Query 对应的 Grounding Data 和 Assertion。Outlook Team 的系统完成两侧执行和 Response 配对，最后才把评分输入提交到 SEVAL，通过 LM Checklist 判断哪一侧对相同任务要求的满足程度更高。

这里比较的不是裸模型 API，而是两个邮件产品的完整结果。Outlook 侧沿用已有 Evaluation 执行链路；Gmail Gemini 没有供项目直接调用的对等评测接口，因此核心实现是用 **Playwright 操作 Gmail Gemini 页面并采集结果**，内部称为 **scraping**。

## 系统怎样工作

```text
Outlook Golden Set
→ 按 Bake-off / 双方支持标签筛选
→ Bake-off Golden Set 子集
   ├─ Query / Utterance
   ├─ 对应 Grounding Data
   └─ Assertion
        ↓
跨系统数据映射
├─ User Mapping
├─ Email Mapping
└─ Golden Set Mapping
        ↓
Google Workspace Ingestion
        ↓
Outlook Team Bake-off Runner
├─ Outlook Copilot → Response
├─ Playwright Scraping → Gmail Gemini → Response
└─ 有效 Response 配对与导出
        ↓
SEVAL Job：仅运行 LM Checklist
        ↓
Outlook Team 生成对比结果
```

Bake-off 不使用完整 Outlook Golden Set。Query 在进入数据同步和执行前已经按标签裁剪，只保留 Gmail Gemini 有对应功能的共同能力集合；Gemini 不支持的功能不会运行，也不记为 Gemini 失败。辅助系统只把这个子集关联的 Grounding Data 导入 Google Workspace，并维护用户、邮件和 Golden Set 之间的映射。两侧 Response 使用同一组 Assertion 和 LM Checklist 评分，避免为某个产品单独改变成功标准。

## 运行方式

Bake-off 不是 SEVAL 提供或编排的能力。Outlook Team 自己搭建批处理系统，负责 Golden Set 裁剪、数据导入、两侧 Runner、Playwright Worker、Query 调度、重试、状态和 Response 配对；两侧结果准备完成后，才创建 SEVAL Job 运行 LM Checklist。

正式 full run 由项目成员在 Bake-off 系统中手动创建，选择 Golden Set、两侧版本、Mapping 和 scraping 配置；启动后执行过程自动完成。另有少量固定 Query 定时运行 scraping smoke，提前发现 Gmail 页面或登录状态变化，但不生成正式产品结论。Outlook Copilot 自身的发布门禁仍由 Copilot Evaluation 回归负责，不依赖 Bake-off。

## 核心实现

Gmail 一侧通过 Playwright scraping 模拟测试用户打开映射后的邮件、调用 Gemini 并提取 Response。Outlook Grounding Data 导入 Google Workspace 后，辅助系统维护 User、Email 和 Golden Set Mapping，保证同一 Query 在两边使用语义对应的用户、邮件和 Utterance。

Outlook Team 的 Bake-off 系统先过滤执行失败并配对两侧有效 Response，再把 Response、Grounding Data 和同一组 Assertion 提交到 SEVAL 运行 `lm_checklist`。SEVAL 返回评分结果后，由 Bake-off 系统生成 Query 级产品对比；数据导入、映射或 scraping 失败单独统计。具体执行和失败分类见[Scraping 与跨系统执行]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/)。

## 结果与业务闭环

Bake-off 不给两个产品做笼统总排名，而是按共同能力 Query 识别 `Outlook fail / Gemini pass` 的相对短板、`Outlook pass / Gemini fail` 的相对优势，以及双方都失败的困难任务。确认是真实产品差异后，再按 Context、工具、生成或 Citation 定位 Outlook 的第一次偏离。

高价值差距会回到常规 Outlook Golden Set，经过功能切片、全量回归、质量门禁和 Ring 发布；下一次 Bake-off 再验证差距是否收敛。Bake-off 提供外部参照，但不替代日常发布评测。完整流程见[结果分析与业务闭环]({{ site.baseurl }}/docs/career/copilot-bakeoff/business-loop/)。

## 主要工作

主要工作包括：

- 使用 Playwright 自动操作 Gmail Gemini 并采集 Response；
- 处理页面加载、流式生成、超时和 scraping 失败分类；
- 维护 Bake-off 标签，只选择双方都有对应功能的 Query；
- 将裁剪后 Query 对应的 Outlook Grounding Data 导入 Google Workspace；
- 维护 User、Email 和 Golden Set Mapping；
- 在 Outlook Team 的系统中调度两侧执行、过滤失败并配对有效 Response；
- 将 Response、Grounding Data 和 Assertion 提交到 SEVAL 运行 LM Checklist；
- 取得评分后生成 Query 级产品对比；
- 聚类 Outlook 相对短板和优势，将高价值差距转入常规 Golden Set 与发布回归。

## 这个项目可以深入到哪里

- Bake-off 为什么采用手动 full run 与自动 scraping smoke 两种触发方式；
- Playwright 怎样稳定操作 Gmail Gemini 并判断生成结束；
- 为什么 scraping 失败不能直接计为 Gemini 质量失败；
- 怎样通过标签从完整 Golden Set 中裁剪双方共同支持的功能；
- 裁剪后的 Outlook Grounding Data 怎样导入 Google Workspace；
- User、Email 和 Golden Set 怎样跨系统映射；
- 怎样保证同一 Query 在两侧使用语义等价的邮件和用户关系；
- Outlook Team 系统怎样准备两侧评分输入，以及 SEVAL 为什么只负责最后的 LM Checklist；
- 如何区分产品质量差异、数据映射问题和 scraping 基础设施问题；
- 怎样把 Outlook 相对短板转成有 Owner 的产品问题，并进入常规 Golden Set、修复和发布闭环。

这个项目最核心的工程问题不是设计一个抽象的竞品评测框架，而是：**在无法直接调用 Gmail Gemini 评测接口的情况下，通过浏览器自动化和跨系统数据映射，得到可以与 Outlook Copilot 公平配对的 Response。**

## 项目文档

- [Scraping 与跨系统执行]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/)：Outlook Team Bake-off 系统、Playwright、Google Workspace ingestion、跨系统 Mapping、失败分类和最终 SEVAL 评分；
- [结果分析与业务闭环]({{ site.baseurl }}/docs/career/copilot-bakeoff/business-loop/)：配对结果的业务含义、差距归因、优先级、Owner、常规回归和后续复测。
