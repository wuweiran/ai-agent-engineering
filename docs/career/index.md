---
layout: default
title: 工作经历
nav_order: 7
has_children: true
permalink: /docs/career/
---

# AI Agent 工程师工作经历

2022 年 5 月加入微软时，主要从事后端开发。此后的工作从工作流系统逐渐转向 Outlook Copilot，包括 Context 服务、Agent 能力开发和评测体系。

这条时间线只记录项目和主要工作。团队分工、系统设计、接口、评测方法、上线过程和实际问题，会在后续项目文档中分别展开。

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

2025 年 6 月开始参与建立 Golden Set 和回归评测流程，最初与 Copilot Agent 功能开发并行，2025 年 10 月后成为主要工作。评测使用固定邮箱、附件和用户任务比较不同模型、Prompt、Context 与工具版本。

评测同时检查回答质量、事实完整性、引用、工具与动作是否正确，以及延迟和成本。确定性结果由程序检查，开放语义使用模型评分和人工抽查。

## 2026.03—至今　[Copilot Bake-off]({{ site.baseurl }}/docs/career/copilot-bakeoff/)

2026 年 3 月开始参与建设 Bake-off 评测框架，作为 Copilot Evaluation 主线下的专项对比工作，评估 Outlook Copilot 与 Gmail Gemini 在邮件问答、附件总结、信息查找和邮箱整理等相近任务上的表现。

框架负责准备可比较的测试邮箱和任务，通过两侧适配器执行，并统一保存输出、引用、交互轮次、耗时和失败原因，再使用盲评规则生成对比结果。

## 时间线中的工作变化

```text
Purview Workflow Expression 与 Logic Apps 集成
→ 在固定流程中接入模型
→ 为 Outlook Copilot 提供 Context
→ 为 Copilot Agent 增加用户任务能力
→ 建立 Golden Set 和产品对比评测
```
