---
layout: default
title: Outlook Copilot Agent 能力开发
parent: 工作经历
nav_order: 3
has_children: true
permalink: /docs/career/copilot-capabilities/
---

# Outlook Copilot Agent 能力开发

## 项目是什么

Outlook Copilot 是运行在 Side Panel 中的通用 Agent，基于 Microsoft 365 Copilot 的 **Declarative Agent** 平台实现。我在同一个 Agent 中增加划词解释、附件总结和邮箱整理等任务能力，不分别建设 Agent 或模型服务。

客户端把正在阅读的邮件、选中文字、附件或当前文件夹交给 Copilot。Declarative Agent 平台根据入口加载 Agent Instructions、场景 Context 和可用 Extension：划词解释走固定 Prompt，附件总结走固定工具链路，开放问答和复杂任务则由模型根据 Tool Result 动态决定继续检索、读取、澄清或提出动作。

## 系统处在什么位置

上游是 Outlook 客户端和 Microsoft 365 Copilot Declarative Agent 平台。下游是已有的 Insight、附件提取和 Outlook 邮件读写 Extension。

```text
用户在 Outlook Side Panel 请求 Copilot
→ 客户端提供界面 Context
→ Declarative Agent 加载 Instructions、Context 和 Extension
→ 模型理解目标并生成 Tool Call
→ Copilot 平台调用已有 Extension
→ Tool Result 返回模型
→ Outlook 展示回答或待确认动作
```

## 业务场景

划词解释、附件总结和邮箱整理复用同一个 Copilot Agent，只是在执行形态、Context、工具和风险上不同。划词解释使用选区和固定 Prompt；附件总结按固定链路提取内容并生成总结；邮箱整理根据搜索和读取结果形成动作计划，并在移动、归档或加旗标前取得用户确认。通用 Agent 可以同时包含这些固定快速路径和模型驱动的动态工具路径。

## 主要工作

主要工作是在统一 Declarative Agent 中落地三类执行形态，并让开放模型判断受到业务 Context、工具契约和确定性安全边界约束：

- 为显式入口和自然语言请求设计能力路由与工具白名单；
- 设计当前邮件、选区、附件、Folder 和多轮任务状态的 Context 生命周期；
- 让邮箱整理通过搜索、按需读取、Planning、确认和写工具完成多步任务；
- 规定 `version_conflict`、`result_unknown` 和 `partial_success` 后的 Agent 行为；
- 使用 Trace、Golden Set 和 Ring 发布定位并阻止行为回归。

## 这个项目可以深入到哪里

- **为什么是一个 Agent**：三个 Capability 为什么共享 Agent，而不拆成多个 Agent 或固定 Workflow；
- **能力怎样路由**：显式入口、界面 Anchor、用户 Utterance 和工具白名单怎样共同决定执行路径；
- **Context 怎样管理**：哪些信息进入本轮、哪些进入任务状态、哪些必须从 Outlook 重新读取；
- **邮箱整理怎样执行**：怎样从模糊目标进入搜索、Planning、确认、写操作、部分成功和 Replanning；
- **工具错误怎样恢复**：权限、版本冲突、结果未知和部分成功为什么需要不同 Agent 行为；
- **安全边界在哪里**：Instructions、工具白名单、平台确认和 Extension 授权分别解决什么问题；
- **怎样定位工具误选**：如何通过 Trace 找到 Instructions、候选工具和 Description 的第一次偏离并回滚；
- **怎样证明修复有效**：失败 Query、Capability 切片、全量 Golden Set 与 Ring 发布怎样形成闭环。

项目的技术深度不在自建模型循环，而在平台边界内把开放的模型判断约束成一条可观察、可确认、可恢复并可持续回归的邮件任务链。

## 项目文档

- [Declarative Agent 接入与执行]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/)：Agent 代码与发布形态、统一 Agent、能力路由、多轮状态、Planning、终止和 Trace；
- [Prompt 与 Context Engineering]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/)：分层 Prompt、界面 Context、证据选择、工具描述和失败归因；
- [邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)：模糊目标、搜索、按需读取、计划确认、Replanning、部分成功和评测；
- [工具设计与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)：Extension、错误语义、Operation ID、幂等、资源版本和部分失败；
- [权限与用户确认]({{ site.baseurl }}/docs/career/copilot-capabilities/security-confirmation/)：工具 Scope、对象级授权、风险配置、计划确认和 Prompt Injection；
- [发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)：版本清单、质量门禁、Ring 灰度、Trace、指标和回滚；
- [工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)：全局规则污染、工具误选、Trace 定位、配置修复和 Golden Set 闭环。
