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

主要工作围绕 Declarative Agent 的横向能力展开：Agent Instructions、界面 Context 装配、Extension 选择与参数使用、Tool Result 处理、读写边界和用户确认。不同场景复用同一个 Copilot 平台，但按任务需要加载不同 Context 与已有工具。

## 这个项目可以深入到哪里

- **Declarative Agent 接入**：Agent Definition、平台模型循环、Tool Call 和 Tool Result 怎样衔接；
- **Prompt Engineering**：System Prompt、能力说明和场景约束怎样让模型理解任务边界；
- **Context Engineering**：当前邮件、选区、附件和文件夹怎样按任务进入 Context，并控制 Token 与噪声；
- **工具设计**：名称、描述和 Schema 怎样影响模型选择，读取、提取和写入工具怎样划分；
- **执行控制**：参数校验、超时、错误语义、降级、幂等和部分失败怎样处理；
- **安全与确认**：对象级权限、Prompt Injection、资源版本和写操作确认怎样由程序保证；
- **版本与发布**：Agent Definition、Instructions、Context 和 Extension 引用怎样灰度、观测和回滚；
- **生产运行**：模型和工具的延迟、Token、成本、容量、监控和值班事故。

每个主题都可以用三个业务场景说明差异。例如 Context Engineering 中比较选区、附件和文件夹 Context；工具设计中比较 Insight 查询、附件提取和邮件写入工具；安全与确认中说明只读解释、内容读取和批量写操作为何采用不同边界。

这里最大的深度不是调用了几次模型，而是：**怎样把邮件和附件这类真实业务对象安全地暴露给 Agent，并让模型判断稳定地落到可验证、可确认和可恢复的执行链路。**

## 项目文档

- [Declarative Agent 接入与执行]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/)：Agent Definition、Extension、平台模型循环、Tool Call、Tool Result、用户确认和终止；
- [Prompt 与 Context Engineering]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/)：分层 Prompt、界面 Context、Token 预算、工具描述、版本和失败归因；
- [工具设计与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)：Extension、错误语义、超时、Operation ID、幂等、资源版本和部分失败；
- [权限与用户确认]({{ site.baseurl }}/docs/career/copilot-capabilities/security-confirmation/)：工具 Scope、对象级授权、Risk Policy、计划确认和 Prompt Injection；
- [发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)：版本清单、配置校验、质量门禁、Ring 灰度、Trace、指标和回滚。
