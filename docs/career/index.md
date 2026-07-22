---
layout: default
title: 工作经历
nav_order: 7
has_children: true
permalink: /docs/career/
---

# AI Agent 工程师工作经历

2022 年 5 月加入微软时，主要从事后端开发。此后的工作从工作流系统逐渐转向 Outlook Copilot，包括 Context 服务、Agent 能力开发和评测体系。

这条时间线只记录项目和主要工作。团队分工、系统设计、接口、评测方法、上线过程和实际问题，会在后续项目文档中分别展开。需要按通用 Agent 技术栈理解这些项目时，可查看 [AI Agent 技术栈与项目映射]({{ site.baseurl }}/docs/career/ai-agent-stack/)。

## 2022.05—2023.12　[Microsoft Purview 工作流]({{ site.baseurl }}/docs/career/purview-workflow/)

加入 Microsoft Purview 团队后，主要担任 Workflow Expression 领域的 Owner，从零定义表达式语法，并实现前后端解析、验证和后端求值。Expression 让用户能够自行组合动态逻辑，后来也成为 Custom Action 的核心运行时能力。Purview 服务采用 Scala + ZIO，底层执行和节点调度由 Azure Logic Apps 提供。

2023 年初开始在 Purview Workflow 中接入 OpenAI，使用模型概括已有 Workflow Definition，并根据用户的自然语言需求生成 Definition 草稿。模型结果必须经过 Action、Parameter、`runAfter` 和 Expression 验证，再由用户确认保存。

## 2024.01—2024.07　[Outlook Copilot Insight Service]({{ site.baseurl }}/docs/career/copilot-insight-service/)

组织调整后转入 Outlook 团队，参与 Outlook Copilot Insight Service。该服务从邮件、会话、日历、联系人和组织信息中取得材料，为 Copilot 准备当前请求需要的 Context。

主要负责 Insight Service 的邮件 Query 检索链路，包括 Outlook Search 分页召回、对象级权限过滤、Message 与 Conversation 去重、多特征打分、Top-K 排序和候选存储；同时参与邮件检索工具契约与权限接入。工作重点从“怎样调用模型”转向“怎样在有界延迟和成本内找到足够且可信的邮件证据”。

## 2024.08—2025.10　[Outlook Copilot Agent 能力开发]({{ site.baseurl }}/docs/career/copilot-capabilities/)

在已有 Outlook Copilot Agent 中增加划词解释、附件总结和邮箱整理等能力。客户端向 Copilot 提供当前邮件、选区、附件或文件夹等界面 Context，Agent 根据用户目标选择 Insight Service、附件读取和 Outlook 邮件工具完成任务。

主要工作是设计这些能力在 Agent 中的 Context、工具和执行路径。划词解释需要补充有限邮件上下文，附件总结需要读取和处理文档，邮箱整理则需要搜索邮件、生成动作计划，并在移动、归档或加旗标前取得用户确认。

## 2025.06—至今　[Copilot Evaluation 与 Golden Set]({{ site.baseurl }}/docs/career/copilot-evaluation/)

2025 年 6 月开始参与 Outlook Copilot 评测工作，最初与 Copilot Agent 功能开发并行，2025 年 10 月后成为主要工作。主要负责评测体系，以及 Golden Set Query、邮件 Grounding Data 和 Assertion 的持续维护；Evaluation Job 运行在统一评测中台 SEVAL。

这套数据最初由各功能开发者分散添加，后来集中维护以复用不断增长的邮件集合；再通过内部 LLM 平台以 eyes-off 方式脱敏真实用户 Utterance，并优先复用能关联较多 Query 的热点 Email，改善开发者构造样例无法反映真实输入分布的问题。Agent 结果通过 LM Checklist 和邮件业务 Metric 评分。

## 2026.03—至今　[Copilot Bake-off]({{ site.baseurl }}/docs/career/copilot-bakeoff/)

2026 年 3 月开始参与 Copilot Bake-off，作为 Copilot Evaluation 主线下的专项对比工作。核心实现是使用 Playwright 操作 Gmail Gemini 页面并采集 Response，内部称为 scraping；同时把 Outlook Grounding Data 导入 Google Workspace，维护 User、Email 和 Golden Set Mapping。

Bake-off 先按标签从 Outlook Golden Set 中裁剪出双方都有对应功能的 Query；Gmail Gemini 不支持的功能不执行，也不记为失败。Golden Set 裁剪、数据 ingestion、mapping、两侧执行、scraping 和 Response 配对全部由 Outlook Team 搭建的系统完成。两侧有效 Response 准备好后才提交到 SEVAL 运行 LM Checklist，评分返回后再由 Bake-off 系统生成 Query 级对比。

## 时间线中的工作变化

```text
Purview Workflow Expression 与 Logic Apps 集成
→ 在固定流程中接入模型
→ 为 Outlook Copilot 提供 Context
→ 为 Copilot Agent 增加用户任务能力
→ 建立 Golden Set 和产品对比评测
```
