---
layout: default
title: Outlook Copilot Agent Mode
parent: 工作经历
nav_order: 6
has_children: true
permalink: /docs/career/copilot-agent-mode/
---

# Outlook Copilot Agent Mode

## 项目是什么

参与 Outlook Copilot Agent Mode，为普通问答难以一次完成的复杂指令提供多轮 Agent 执行能力。用户仍然从 Outlook Copilot 对话入口发起任务，Agent 可以搜索邮件、读取会话、形成计划并调用 Outlook 工具。

邮箱整理是代表性场景。用户描述整理目标后，Agent 查找并分类邮件，在 Copilot 中展示动作计划，用户确认后再执行移动、归档或加旗标。

## 系统处在什么位置

上游仍是 Outlook Copilot 客户端和 Orchestrator。Agent Mode 复用此前的 Insight Service、邮件搜索、会话读取和 Outlook 写工具，也继续使用 Golden Set 与 Evaluation 流程作为发布门槛。

```text
Outlook Copilot 复杂指令
→ Orchestrator 进入 Agent Mode
→ 模型根据 Context 选择工具
→ Tool Result 更新计划和任务状态
→ 用户确认高影响动作
→ Outlook 工具执行
→ Copilot 返回任务结果
```

## 主要工作

主要工作包括 Agent Mode 的任务路由、Context、工具契约、Planning、用户确认和执行控制。复杂任务还需要限制搜索范围、工具轮次、Token 和任务时间，并处理部分成功、资源已经变化和重复操作。

## 这个项目可以深入到哪里

- 普通 Copilot 问答与 Agent Mode 复杂任务怎样路由；
- ReAct、Planning 和 Replanning 怎样配合；
- 工具搜索、工具集合控制和工具选择评测；
- 多轮 Context 的保留、压缩和重建；
- Task、Run、Step 和 Tool Call 怎样保存任务事实；
- 完成、等待、失败和结果未知分别怎样处理；
- 轮次、Token、时间、成本和工具预算；
- 重复工具调用与连续无进展怎样检测；
- 用户确认为什么要绑定具体计划版本；
- 幂等、资源版本、批量操作和部分成功；
- Worker 重启后的任务恢复与长任务生命周期；
- SSE 或任务事件怎样向 Copilot 展示进度；
- Trace、线上指标、灰度、停止条件和回滚；
- 外部邮件与附件中的 Prompt Injection；
- 越权、错误写入和数据外传怎样由确定性软件阻止。

这个项目可以成为完整的生产级 Agent Runtime 经历。重点不是搭建一个脱离产品的通用演示平台，而是把 Context、规划、工具、状态、安全、评测和生产运行放进 Outlook Copilot 的真实任务中。

## 后续可以展开的文档

- Agent Mode 入口：任务识别、普通问答与复杂任务的路由；
- Runtime：模型循环、Context、预算、终止和等待；
- Planning 与状态：计划版本、任务步骤、恢复和用户确认；
- Outlook 工具：搜索、读取、移动、归档、加旗标和错误契约；
- 邮箱整理案例：范围限定、分类、批量执行和部分成功；
- 安全与评测：Prompt Injection、权限、Shadow、Golden Set 和线上指标；
- 部署与生产运行：服务拓扑、机器配置、扩缩容、队列、监控和事故处理。
