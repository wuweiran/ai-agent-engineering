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

## 2022.05—2023.03　[Microsoft Purview 工作流]({{ site.baseurl }}/docs/career/purview-workflow/)

加入 Microsoft Purview 团队后，负责数据治理产品中的工作流系统，包括工作流定义、运行状态、异步任务、失败重试、权限校验和服务运维。

2023 年初开始在工作流中接入 OpenAI，使用模型完成审批材料摘要、分类和说明生成。执行路径仍由工作流确定，模型只是其中一个处理节点。

## 2023.04—2024.06　[Outlook Copilot Insight Service]({{ site.baseurl }}/docs/career/copilot-insight-service/)

组织调整后转入 Outlook 团队，参与 Outlook Copilot Insight Service。该服务从邮件、会话、日历、联系人和组织信息中取得材料，为 Copilot 准备当前请求需要的 Context。

主要工作包括数据接入、权限过滤、相关性排序、去重、摘要和 Context 长度控制。工作重点从“怎样调用模型”转向“怎样让模型看到正确、及时且有权限的信息”。

## 2024.07—2025.04　[Outlook Copilot Agent 能力开发]({{ site.baseurl }}/docs/career/copilot-capabilities/)

在已有 Outlook Copilot Agent 中增加划词解释、附件总结和邮箱整理等能力。客户端向 Copilot 提供当前邮件、选区、附件或文件夹等界面 Context，Agent 根据用户目标选择 Insight Service、附件读取和 Outlook 邮件工具完成任务。

主要工作是设计这些能力在 Agent 中的 Context、工具和执行路径。划词解释需要补充有限邮件上下文，附件总结需要读取和处理文档，邮箱整理则需要搜索邮件、生成动作计划，并在移动、归档或加旗标前取得用户确认。

## 2024.11—2025.10　[Copilot Evaluation 与 Golden Set]({{ site.baseurl }}/docs/career/copilot-evaluation/)

与 Copilot 功能开发并行，参与建立 Golden Set 和回归评测流程，用固定邮箱、附件和用户任务比较不同模型、Prompt、Context 与工具版本。

评测同时检查回答质量、事实完整性、引用、工具与动作是否正确，以及延迟和成本。确定性结果由程序检查，开放语义使用模型评分和人工抽查。

## 2025.08—2026.02　[Copilot Bake-off]({{ site.baseurl }}/docs/career/copilot-bakeoff/)

参与建设 Bake-off 评测框架，对比 Outlook Copilot 与 Gmail Gemini 在邮件问答、附件总结、信息查找和邮箱整理等相近任务上的表现。

框架负责准备可比较的测试邮箱和任务，通过两侧适配器执行，并统一保存输出、引用、交互轮次、耗时和失败原因，再使用盲评规则生成对比结果。

## 2026.03—至今　[Outlook Copilot Agent Mode]({{ site.baseurl }}/docs/career/copilot-agent-mode/)

参与 Outlook Copilot Agent Mode，为复杂、多步骤指令提供 Agent 执行能力。用户仍然从 Copilot 对话入口发起任务，Agent 可以多轮搜索邮件、读取会话、形成计划并调用 Outlook 工具。

邮箱整理是其中一个代表性场景。Agent 根据用户目标查找和分类邮件，在 Copilot 中展示整理计划，用户确认后再执行移动、归档或加旗标。该项目继续复用 Insight Service、Outlook 工具和已有 Evaluation 流程。

## 时间线中的工作变化

```text
Purview 后端工作流
→ 在固定流程中接入模型
→ 为 Outlook Copilot 提供 Context
→ 为 Copilot Agent 增加用户任务能力
→ 建立 Golden Set 和产品对比评测
→ 在 Copilot Agent Mode 中支持复杂指令
```
