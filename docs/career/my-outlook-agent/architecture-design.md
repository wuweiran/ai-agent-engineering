---
layout: default
title: Architecture Design
grand_parent: 工作经历
parent: My Outlook Agent 部署
nav_order: 1
permalink: /docs/career/my-outlook-agent/architecture-design/
---

# Architecture Design

这份文档从 Agent 的基本构成解释 My Outlook，适合面试时先讲整体。任务持久化与恢复、Context 裁剪和 AKS 部署分别由后续文档展开。

## 一句话说明

My Outlook 是自托管 Agent：**POS Service 维护 Agent Loop 和任务状态，模型根据本轮 Context 决定回答、澄清或调用工具；Runtime 把模型与工具动作持久化为 Step，交给独立 Worker 执行，再用 Result 推动下一轮。**

```text
用户目标或后台触发
→ POS Service 创建 / 恢复 Task
→ Context Builder 构造本轮 Model Input
→ LLM Worker 调用模型
→ Final / Clarify / Tool Call
→ 对应 Worker 执行并保存 Result
→ POS 根据最新状态继续 Agent Loop
```

一条典型后台链路是：POS 为 Briefing 创建任务，Deep Scan Worker 从邮件、日历和组织信号中提取 Finding 并写入 Annotation Store；Synthesis Worker 根据这些 Finding 生成带 Citation 的 Briefing Artifact，写入 SDS。用户打开 Briefing 后继续追问时，POS 恢复对应 Task，Context Builder 选择当前目标、相关 Finding、Artifact 和最近消息，模型再决定直接回答还是调用 Graph/API Worker 核实原始邮件或日历。

## Agent 的基本组成

| Agent 问题 | My Outlook 的实现 |
| --- | --- |
| 目标是什么 | Task 保存用户目标、完成条件、Deadline 和当前状态 |
| 谁决定下一步 | POS Service 维护循环；模型在开放判断处返回 Final、Clarify 或 Tool Call |
| 怎样执行动作 | POS 把 Model、Deep Scan、Synthesis、Graph/API 动作创建为 Step，由对应 Worker 执行 |
| 模型每轮看到什么 | Context Builder 从消息、Task State、Finding、Artifact 和最新 Worker Result 中构造 Model Input |
| 过程保存在哪里 | Conversation Store 保存消息，Task Store 保存执行状态，Annotation Store 与 SDS 保存 Finding 和 Artifact |
| 怎样暂停和恢复 | Step Result 落库后继续；等待用户或依赖时释放 Worker；重启后从持久化 Step 恢复 |
| 怎样结束 | 模型建议 Final 或 Clarify，Runtime 再根据状态、完成条件、Deadline 和错误判断完成、等待或失败 |

## Agent Loop 与 Planning

My Outlook 没有在现有材料中定义一个独立的 Planner Service，也不是先生成一份完整 Plan 再机械执行。它采用**状态驱动的逐步决策**：POS 读取当前 Task State 和上一步 Result，构造下一次模型调用；模型可以返回最终回答、请求澄清或选择工具，POS 再创建对应 Step。

同时，并非所有步骤都交给模型决定。保存 Synthesis Artifact、状态推进等固定业务流程由 POS 直接创建确定性 Step。只有“是否需要调用某项能力、调用什么参数”这类开放判断才暴露为模型工具。

```text
开放判断：模型根据 Context 选择工具或结束
确定流程：POS 按业务状态直接创建 Step
真实执行：Worker 调用受控 Handler
状态推进：POS 根据已落库 Result 决定下一轮
```

因此面试中可以说它有 Agent Planning，但应准确描述为**基于 Task State 的逐步规划与工具选择**，不能声称项目实现了独立 Planner 或复杂计划搜索。

## 工具怎样接入

工具不是让模型直接访问 Service Bus、数据库或 Microsoft Graph。POS 维护版本化 Tool Catalog 和参数契约；模型产生 Tool Call 后，Runtime 校验工具名、版本与参数，创建持久化 Step，再由内置对应 Handler 的 Worker 执行。

```text
Model Tool Call
→ Runtime 校验 Tool + Version + Arguments
→ 创建持久化 Step
→ Queue 通知 Worker
→ Worker 调用 Deep Scan / Synthesis / Graph / Model Handler
→ 结构化 Result 写回 Task Store
```

Worker Result 只返回状态、Artifact / Evidence Reference、短摘要、Warning 和错误类型。大型内容保存在 SDS 或权威业务系统中，避免把完整响应反复塞回模型。

## Context 怎样管理

Context 不是 Conversation 的同义词，也不是把所有持久数据拼起来。每次模型调用前，Context Builder 按当前 Step 和 Token Budget 重建临时 Model Input：

1. 固定保留当前目标、确认约束、完成条件和当前 Step；
2. 加入当前步骤必需的证据和最新 Worker Result；
3. 保留最近消息原文；
4. 按相关性、新鲜度和权限选择 Finding 与 Artifact；
5. 去除重复或过期结果，必要时压缩较早 Conversation。

完整邮件和日历仍保存在 Microsoft 365 权威系统，Context 中通常只放 Reference、摘要和 Citation，需要核实时再由 Graph/API Worker 读取。详细预算、选择、替换与压缩规则见 [Context Builder 与 Token Budget]({{ site.baseurl }}/docs/career/my-outlook-agent/context-builder/)。

## 状态、历史和记忆怎样区分

My Outlook 没有把所有持久数据统一抽象成 Memory，而是按用途分开保存：

| 类型 | 项目中的载体 | 作用 |
| --- | --- | --- |
| 消息历史 | Conversation Store | 保存用户与 Agent 说过什么 |
| 当前任务状态 | Task / Run / Step / Attempt | 保存目标、确认事实、进度、重试与恢复位置 |
| 产品 Finding | Annotation Store | 保存从邮件、日历和组织信号提取的重点、承诺与待办 |
| 生成产物 | SDS Artifact Store | 保存 Briefing、Catch-up、Draft、Recommendation、版本和 Citation |
| 本轮 Context | ContextBuildResult | 从上述来源选择出的临时模型工作集 |

项目中没有独立的通用长期记忆模块，例如自动从每次对话提炼用户偏好，再通过向量检索跨任务召回。Finding 和 Artifact 会持久化，也可以在后续任务中按权限、目标和版本被选择，但它们首先是有明确业务 Schema、来源和生命周期的产品数据，不应笼统包装成“做了长期记忆”。

My Outlook 具备跨轮消息历史、可恢复任务状态和可跨任务使用的个性化 Finding / Artifact；Context Builder 负责按需取用。这与从对话中自动提炼、检索和失效用户长期记忆是两套机制。

## 任务怎样结束或等待

模型可以建议结束，但 Runtime 控制生命周期：

- **Final**：保存最终回答并完成 Run；
- **Clarify**：保存问题，Task 进入 `waiting_user`，新消息到达后继续；
- **Continue**：创建下一 Model 或 Tool Step；
- **waiting_dependency**：依赖返回 `Retry-After` 时保存 `not_before`，释放 Worker 后再调度；
- **failed / cancelled**：达到 Deadline、重试预算、永久错误或取消条件后终止。

这使任务可以脱离原始 HTTP 请求持续运行，也避免模型无限循环或在外部动作尚未确认时自行宣布完成。

## 可靠性为什么属于 Agent 设计

Agent Loop 会跨模型、队列、Worker 和外部 API，仅有 Prompt 与 Tool Schema 不足以保证生产正确性。My Outlook 将每个动作持久化为 Step，并用 Task Store、Lease、Attempt、幂等键和固定版本处理重复投递、Worker 重启、迟到结果和滚动发布。

这部分是我的主要参与边界：维护 Service—Queue—Worker 运行链路，沿 Trace 定位任务卡住、重复执行和 Context 异常，并修复恢复与版本问题。详细故障窗口见 [Agent Runtime 与任务恢复]({{ site.baseurl }}/docs/career/my-outlook-agent/runtime-task/)。

## 面试讲述顺序

```text
先讲产品目标：主动理解工作信号并完成查询、生成和执行任务
→ 再讲 Agent Loop：Context → Model → Tool → Result → 下一轮
→ 再区分：Context、消息历史、任务状态、Finding / Artifact
→ 再讲生产化：异步 Step、恢复、幂等和版本
→ 最后用一个真实 Bug 展开根因、修复与结果
```

需要守住两个边界：Deep Scan 与 Synthesis 的业务算法由对应团队维护；我的重点是自托管 Agent 的部署、Runtime 运行链路、Context 问题和生产修复，不声称是整个 Agent 的 Owner。