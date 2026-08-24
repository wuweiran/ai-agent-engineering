---
layout: default
title: Copilot Bake-off
parent: 工作经历
nav_order: 6
has_children: true
permalink: /docs/career/copilot-bakeoff/
---

# Copilot Bake-off

## 核心思路

从产品维度对比 Outlook Copilot 与 Gmail Gemini

→ 两边能力范围、用户体系和邮件存储不同，直接比较不公平

→ 通过**共同能力筛选与跨系统映射**准备等价的 Query、用户和邮件

→ Gmail 没有对等评测接口，且 UI 与流式 Response 会变化

→ 通过**双侧真实产品 Scraping**取得 Outlook 与 Gmail Response

→ 通过**有效结果配对与同标准评分**避免把 Mapping 或 scraping 失败算成产品失败

→ 产出四类 Query 级对比，定位 Outlook 相对短板，并将高价值问题纳入常规 Golden Set 回归

**同类项目通常关注：** 产品质量差异（四类配对结果分布）、分场景差异（各功能的 `Outlook fail / Gemini pass`）、有效样本覆盖、数据等价性、执行稳定性（scraping / ingestion / Mapping 失败率）、评测成本。

## 项目介绍

Copilot Bake-off 是 Outlook Copilot Evaluation 下的竞品对比系统，用同一组邮件、用户任务和 Assertion 比较 Outlook Copilot 与 Gmail Gemini 的完整产品结果，而不是比较裸模型。项目只选择双方都支持的功能；由于 Gmail Gemini 没有对等评测接口，核心方案是通过 [Playwright scraping]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/#playwright-scraping)操作 Gmail 页面并采集 Response，再与 Outlook 结果做配对评分。

我从零搭建了这套 Bake-off 系统。系统完成共同能力 Query 筛选、Google Workspace 数据导入、[跨系统 Mapping]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/#cross-system-mapping)和两侧 Response 配对；Outlook 侧使用 SEVAL scraping，Gmail 侧使用 Playwright scraping。Playwright 执行器按页面状态推进，并[判断流式生成是否真正结束]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/#response-completion-detection)；scraping、数据映射和产品失败分别统计，只有两侧都得到有效 Response 的 Query 才进入[配对与 LM Checklist 评分]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/#response-pairing-evaluation)。最终结果用于定位 `Outlook fail / Gemini pass` 的具体产品差距。高价值问题会进入[长期回归闭环]({{ site.baseurl }}/docs/career/copilot-bakeoff/business-loop/#product-improvement-loop)：加入常规 Outlook Golden Set，完成修复、功能切片和全量回归，通过质量门禁后发布，再由下一次 Bake-off 验证差距是否收敛。

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
Outlook Team Bake-off System
├─ SEVAL Scraping → Outlook Copilot → Response
├─ Playwright Scraping → Gmail Gemini → Response
└─ 有效 Response 配对与导出
        ↓
SEVAL Job：运行 LM Checklist
        ↓
Outlook Team 生成对比结果
```

Bake-off 不使用完整 Outlook Golden Set。Query 在进入数据同步和执行前已经按标签裁剪，只保留 Gmail Gemini 有对应功能的共同能力集合；Gemini 不支持的功能不会运行，也不记为 Gemini 失败。辅助系统只把这个子集关联的 Grounding Data 导入 Google Workspace，并维护用户、邮件和 Golden Set 之间的映射。两侧 Response 使用同一组 Assertion 和 LM Checklist 评分，避免为某个产品单独改变成功标准。

## 运行方式

Bake-off 的整体编排由 Outlook Team 搭建的系统负责，包括 Golden Set 裁剪、数据导入、跨系统 Mapping、Query 状态和 Response 配对。Outlook 侧调用 SEVAL scraping，Gmail 侧调度 Playwright Worker；两侧结果准备完成后，再创建 SEVAL Job 运行 LM Checklist。

正式 full run 由项目成员在 Bake-off 系统中手动创建，选择 Golden Set、两侧版本、Mapping 和 scraping 配置；启动后执行过程自动完成。另有少量固定 Query 定时运行 scraping smoke，提前发现 Gmail 页面或登录状态变化，但不生成正式产品结论。Outlook Copilot 自身的发布门禁仍由 Copilot Evaluation 回归负责，不依赖 Bake-off。

## 核心实现

Outlook 一侧通过 SEVAL scraping 运行 Golden Set Query 并取得 Response；Gmail 一侧通过 Playwright scraping 模拟测试用户打开映射后的邮件、调用 Gemini 并提取 Response。Outlook Grounding Data 导入 Google Workspace 后，辅助系统维护 User、Email 和 Golden Set Mapping，保证同一 Query 在两边使用语义对应的用户、邮件和 Utterance。

Outlook Team 的 Bake-off 系统先过滤执行失败并配对两侧有效 Response，再把 Response、Grounding Data 和同一组 Assertion 提交到 SEVAL 运行 `lm_checklist`。SEVAL 返回评分结果后，由 Bake-off 系统生成 Query 级产品对比；数据导入、映射或 scraping 失败单独统计。具体执行和失败分类见[Scraping 与跨系统执行]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/)。

## 结果与业务闭环

Bake-off 不给两个产品做笼统总排名，而是按共同能力 Query 识别 `Outlook fail / Gemini pass` 的相对短板、`Outlook pass / Gemini fail` 的相对优势，以及双方都失败的困难任务。确认是真实产品差异后，再按 Context、工具、生成或 Citation 定位 Outlook 的第一次偏离。

高价值差距会回到常规 Outlook Golden Set，经过功能切片、全量回归、质量门禁和 Ring 发布；下一次 Bake-off 再验证差距是否收敛。Bake-off 提供外部参照，但不替代日常发布评测。完整流程见[结果分析与业务闭环]({{ site.baseurl }}/docs/career/copilot-bakeoff/business-loop/)。

## 这个项目可以深入到哪里

- Bake-off 为什么采用手动 full run 与自动 scraping smoke 两种触发方式；
- Playwright 怎样稳定操作 Gmail Gemini 并判断生成结束；
- 为什么 scraping 失败不能直接计为 Gemini 质量失败；
- 怎样通过标签从完整 Golden Set 中裁剪双方共同支持的功能；
- 裁剪后的 Outlook Grounding Data 怎样导入 Google Workspace；
- User、Email 和 Golden Set 怎样跨系统映射；
- 怎样保证同一 Query 在两侧使用语义等价的邮件和用户关系；
- Outlook 侧怎样通过 SEVAL scraping 取得 Response，Gmail 侧怎样通过 Playwright scraping 取得 Response；
- 如何区分产品质量差异、数据映射问题和 scraping 基础设施问题；
- 怎样把 Outlook 相对短板转成有 Owner 的产品问题，并进入常规 Golden Set、修复和发布闭环。

这个项目最核心的工程问题不是设计一个抽象的竞品评测框架，而是：**在无法直接调用 Gmail Gemini 评测接口的情况下，通过浏览器自动化和跨系统数据映射，得到可以与 Outlook Copilot 公平配对的 Response。**

## 项目文档

- [Scraping 与跨系统执行]({{ site.baseurl }}/docs/career/copilot-bakeoff/execution/)：Outlook Team Bake-off 系统、Playwright、Google Workspace ingestion、跨系统 Mapping、失败分类和最终 SEVAL 评分；
- [结果分析与业务闭环]({{ site.baseurl }}/docs/career/copilot-bakeoff/business-loop/)：配对结果的业务含义、差距归因、优先级、Owner、常规回归和后续复测。
