---
layout: default
title: Outlook Copilot Agent 能力开发
parent: 工作经历
nav_order: 3
has_children: true
permalink: /docs/career/copilot-capabilities/
---

# Outlook Copilot Agent 能力开发

## 项目介绍

Outlook 用户阅读邮件时，经常需要解释局部内容、总结附件，或者继续搜索和整理相关邮件。这个项目的目标不是再做一个通用聊天入口，而是让用户留在当前邮件场景中完成“理解内容—取得证据—执行动作”的任务。我基于 Microsoft 365 Copilot 的 [Declarative Agent 框架]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/#declarative-agent-configuration)，从零开发了划词解释、附件总结和邮箱整理三项能力。

Microsoft 365 Copilot 提供模型循环、Conversation、Planning 和确认；三项能力包含 Agent Definition、场景 Instructions、Sydney Context Config 和 Extension 接入。核心设计包括：[Prompt/Context Engineering]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/#prompt-context-design)决定每轮输入哪些界面信息、任务状态和 Tool Result，[动态 Context 裁剪]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/#dynamic-context-trimming)控制 Token Budget，[Tool Calling]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/#tool-calling-pipeline)和[多轮状态规则]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/#multi-turn-state)决定任务怎样继续；写操作还要经过[用户确认]({{ site.baseurl }}/docs/career/copilot-capabilities/security-confirmation/#write-confirmation)，并处理版本冲突、部分成功和结果未知。

```text
Microsoft 365 Copilot 网页或 Outlook 入口
→ Sydney Payload 与当前场景 Context
→ BizChat 后端接口
→ Microsoft 365 Copilot Agent Runtime
→ 已有 Extension
→ 回答或邮件操作结果
```

## 三项能力

| 能力 | 实现方式 | 项目工作 |
| --- | --- | --- |
| 划词解释 | 选区加固定 Prompt 直接生成 | 配置选区 Context、回答范围和 Citation |
| 附件总结 | 调用附件提取 Extension 后生成 | 处理提取结果、不完整内容和来源引用 |
| 邮箱整理 | 搜索、按需读取、计划确认后调用写工具 | 处理模糊目标、多轮修改和执行失败 |

## 业务闭环与结果

业务指标重点看**邮箱整理线上任务完成率**。系统用 BizChat Conversation ID、Plan 版本和 Operation ID 串联任务开始、计划展示、用户确认、写请求与逐项 Tool Result；只有当前确认计划中的全部邮件都取得 `succeeded` 终态，才记录 `task_completed`。模型说“完成”、HTTP 返回 200 或 Tool Call 已发出都不能单独算成功。`partial_success` 要继续关联剩余项，`result_unknown` 要查询原 Operation ID；未形成计划、确认后执行未收敛或用户直接离开都记为未完成，主动取消和产品未支持的请求不进入分母。详细事件口径见[线上观测]({{ site.baseurl }}/docs/career/copilot-capabilities/production/#online-observability)。

初版线上任务完成率为 **62.4%**。漏斗分析发现两类主要损失：一类是目标模糊，Agent 反复追问或候选范围不清，任务没有到达确认；另一类发生在用户确认后，邮件状态变化、批量部分成功或调用结果未知导致任务没有完整收敛。

针对第一类问题，我把流程拆成“收敛范围”和“确认执行”两段：只追问会改变结果的条件，先搜索候选并按需读取详情，再生成包含 Message ID、动作和目标 Folder 的计划。针对第二类问题，执行阶段根据 Tool Result 分别处理[版本冲突]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/#version-conflict)、[部分成功]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/#partial-success)和[结果未知]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/#result-unknown)，避免使用过期计划、重复提交整个批次或把未知结果当成成功。完整流程见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)。

调整后，到达并确认有效计划的比例从 **69.3% 提升到 81.5%**，确认后全部操作完成的比例从 **90.0% 提升到 96.4%**，最终线上任务完成率从 **62.4% 提升到 78.6%**。离线邮箱整理 Scenario 通过率同时从 **68.1% 提升到 84.7%**，严重安全违规保持为 **0**。

项目做了明确取舍：第一版只开放移动、归档和加旗标，不开放删除、发送和修改收件人；不建设跨任务长期记忆，动态邮件状态需要重新读取；写操作增加一次用户确认，牺牲少量交互效率，换取错误范围可见和操作可控。Context 裁剪和 Token 优化继续作为成本指标，但不用于代替业务完成率。

## 项目文档

- [Declarative Agent 接入与执行]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/)：代码形态、平台边界和三项能力的接入方式；
- [Prompt 与 Context Engineering]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/)：三类 Context、工具描述和 Token 优化；
- [邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)：项目的核心业务与技术难点；
- [工具错误与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)：版本冲突、部分成功和结果未知；
- [权限与用户确认]({{ site.baseurl }}/docs/career/copilot-capabilities/security-confirmation/)：权限边界、写操作确认和 Prompt Injection；
- [发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)：评测、灰度、观测和回滚；
- [工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)：一次场景工具配置问题的排查过程。
