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

在已有 Outlook Copilot Agent 中增加划词解释、附件总结和邮箱整理等任务能力。它们不是三个彼此独立的模型服务，而是同一个 Agent 在不同界面 Context 下采用的执行路径。

客户端会把正在阅读的邮件、选中的文字、点击的附件或当前文件夹交给 Copilot。Agent 根据用户目标和这些 Context，选择 Insight Service、附件读取或 Outlook 邮件工具完成任务。

## 系统处在什么位置

上游是 Outlook 客户端和 Copilot Orchestrator。下游复用 Insight Service、附件内容提取、邮件搜索以及 Exchange Online 的邮件读写接口。

```text
用户在 Outlook 中请求 Copilot
→ 客户端提供当前界面 Context
→ Copilot Agent 理解目标并选择工具
→ Insight Service、附件工具或 Outlook 邮件工具
→ Tool Result 返回 Agent
→ Outlook 展示回答或待确认动作
```

## 三类能力

**划词解释**主要解决界面 Context。Agent 不仅需要选中文字，还需要有限前后文、邮件主题和语言，才能解释缩写与指代。

**附件总结**需要 Agent 确认目标附件，调用内容提取工具，再处理短文档、长文档、OCR、格式不支持和引用等情况。

**邮箱整理**开始涉及业务动作。Agent 搜索并分类邮件，生成移动、归档或加旗标计划；写操作在用户看到受影响范围并确认后才执行。

## 主要工作

主要工作是设计能力说明、Context 装配、工具 Schema、Tool Result 和确认边界。不同能力需要不同的信息与工具，不能只靠一个不断增长的 Prompt 处理全部场景。

## 这个项目可以深入到哪里

- 当前邮件、选区、附件和文件夹等界面 Context 怎样进入 Agent；
- Function Calling、Tool Result 与模型的后续判断；
- 工具名称、描述和 Schema 怎样影响工具选择；
- 工具参数校验与服务端业务校验的区别；
- 读工具与写工具怎样划分；
- 邮件、附件和文件夹的对象级授权；
- 写操作怎样展示参数并取得用户确认；
- 工具超时、错误语义、降级和部分失败；
- 附件和邮件正文作为不可信内容带来的 Prompt Injection；
- 为什么划词解释、附件总结和邮箱整理需要不同的 Context 与执行路径。

这里最大的深度不是调用了几次模型，而是：**怎样把邮件和附件这类真实业务对象安全地暴露给 Agent。**

## 后续可以展开的文档

- Agent 能力接入：Copilot Orchestrator、能力注册、Context 和工具循环；
- 划词解释：选区 Context、流式结果、语言和受保护内容；
- 附件总结：内容提取、分段、引用、OCR 和错误契约；
- 邮箱整理：搜索、分类、动作计划、确认和批量执行；
- 工具安全：权限、资源版本、幂等和 Prompt Injection；
- 部署与生产运行：模型网关、服务资源、容量、延迟、灰度和值班。
