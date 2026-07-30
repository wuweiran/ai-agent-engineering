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

项目基于 Microsoft 365 Copilot 的 **Declarative Agent** 扩展形态实现，通过 Sydney API 接入 BizChat。网页和 Outlook 内入口共用同一套 Agent Definition、Context Config 和 Extension；我在这个 Agent 中交付了划词解释、附件总结和邮箱整理三项能力。

Sydney Context Config 决定每次请求携带哪些当前邮件、选区、附件、文件夹和入口类型。Declarative Agent 的同一个 `instructions` 字段根据这些信息区分场景并定义行为规则；`actions` 引用 Outlook API Plugin，Plugin Manifest 定义全局可用工具的 Description、参数和后端 Runtime。Microsoft 365 Copilot Agent Runtime 将这些内容一起交给模型，由模型直接回答或生成 Tool Call。

```text
Microsoft 365 Copilot 网页或 Outlook 入口
→ Sydney Payload 与当前场景 Context
→ BizChat 后端接口
→ Microsoft 365 Copilot Agent Runtime
→ 已有 Extension
→ 回答或邮件操作结果
```

## 主要工作

我负责三项能力的 Agent Definition、场景 Prompt、Sydney Context Config 和 Extension 接入，定义每个场景需要哪些 Context、模型什么时候直接回答或调用工具，以及不同 Tool Result 返回后怎样继续。Context 采集代码由 Sydney 接入层同事维护。功能上线前使用 Trace 排查问题，并通过 Golden Set 回归和 Ring 灰度发布。

Microsoft 365 Copilot Agent Runtime 负责模型循环、Conversation、Planning、确认和流式输出；Extension 团队负责工具后端、权限、资源版本和幂等执行，这些不属于我的实现范围。

## 三项能力

| 能力 | 实现方式 | 项目工作 |
| --- | --- | --- |
| 划词解释 | 选区加固定 Prompt 直接生成 | 配置选区 Context、回答范围和 Citation |
| 附件总结 | 调用附件提取 Extension 后生成 | 处理提取结果、不完整内容和来源引用 |
| 邮箱整理 | 搜索、按需读取、计划确认后调用写工具 | 处理模糊目标、多轮修改和执行失败 |

第一版邮箱整理只支持移动、归档和加旗标，不开放删除、发送和修改收件人等高风险动作。

## 核心难点与解决方法

项目最难的部分是邮箱整理。用户经常只说“整理一下最近需要跟进的邮件”，时间、筛选标准、动作和目标文件夹并不完整；候选邮件又只有搜索之后才能确定，因此不能直接写成一条固定 Workflow。但移动、归档和加旗标会真实修改邮箱，也不能让模型根据一段对话直接执行。

我把它分成两个阶段。前半段只允许搜索和读取，Agent 通过必要的澄清、Insight Query 返回的 Top 12 候选和按需读取收敛目标范围；后半段把 Message ID、动作、目标 Folder 和资源版本交给平台 Planning，用户确认后再由邮件 Extension 执行。

执行错误按工具返回的状态处理：`version_conflict` 重新读取并更新计划，`partial_success` 保留已经成功的邮件，`result_unknown` 沿用原 Operation ID 查询结果，不重新创建写请求。修复后的版本再通过邮箱整理场景回归和 Ring 发布。

完整设计见[邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)。

## 结果

三项能力进入同一个 Outlook Declarative Agent。一次 Context 优化后，划词解释的平均输入从 **12,400 Token 降到 7,600 Token**；多轮邮箱整理 Scenario 的累计输入从 **46,000 Token 降到 34,000 Token**。当前评测基线中，`lm_checklist` 为 **87.6%**、`tool_call` 为 **93.8%**，严重安全违规为 **0**。

## 项目文档

- [Declarative Agent 接入与执行]({{ site.baseurl }}/docs/career/copilot-capabilities/runtime/)：代码形态、平台边界和三项能力的接入方式；
- [Prompt 与 Context Engineering]({{ site.baseurl }}/docs/career/copilot-capabilities/prompt-context/)：三类 Context、工具描述和 Token 优化；
- [邮箱整理端到端设计]({{ site.baseurl }}/docs/career/copilot-capabilities/mailbox-organization/)：项目的核心业务与技术难点；
- [工具错误与执行控制]({{ site.baseurl }}/docs/career/copilot-capabilities/tool-execution/)：版本冲突、部分成功和结果未知；
- [权限与用户确认]({{ site.baseurl }}/docs/career/copilot-capabilities/security-confirmation/)：权限边界、写操作确认和 Prompt Injection；
- [发布与生产运行]({{ site.baseurl }}/docs/career/copilot-capabilities/production/)：评测、灰度、观测和回滚；
- [工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)：一次场景工具配置问题的排查过程。
