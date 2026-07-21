---
layout: default
title: Prompt 与 Context Engineering
parent: Outlook Copilot Agent 能力开发
grand_parent: 工作经历
nav_order: 2
permalink: /docs/career/copilot-capabilities/prompt-context/
---

# Prompt 与 Context Engineering

## Prompt 配置

项目没有维护一份覆盖所有场景的超长 Prompt。Agent Definition 中保留通用回答和安全规则，划词解释、附件总结和邮箱整理分别维护自己的 Capability Prompt。场景 Prompt 只写当前任务的目标、可用信息、工具使用条件和完成要求。

这种拆分主要解决实际维护问题：修改邮箱整理的搜索规则时，不应改变划词解释；附件提取失败的处理，也不应该进入普通问答。三个 Capability Prompt 随 Agent Definition 一起版本化发布。

## 三类 Context

### 划词解释

客户端提供选中文字、前后各一个段落、邮件主题、发送者、时间和 Message ID。选区足以回答时不读取整封邮件，也不加载邮箱搜索和写工具。

### 附件总结

客户端只提供 Message ID 和 Attachment ID，附件提取 Extension 返回文本、页码或章节位置、提取状态和 Citation。内容不完整时保留 `partial` 状态，回答只总结已经取得的部分。原始二进制和 OCR 中间数据不进入模型 Context。

### 邮箱整理

初始 Context 只有当前 Folder、用户目标和同一任务中已经确认的条件。Agent 先调用 Insight Query 取得 Top 12 候选，需要进一步判断时再读取邮件详情，不会把整个文件夹预先放进 Context。

邮件正文、附件和搜索 Snippet 都按外部数据处理，不能覆盖 Agent Instructions，也不能改变工具权限和确认要求。

## 工具描述

工具名称和 Description 直接影响模型选择。项目只写清楚三个问题：什么时候调用、返回什么、不能替代什么。

- `search_outlook_context` 用来发现未知邮件，不代替已知 Message ID 的读取；
- `read_outlook_message` 读取指定邮件，不执行写操作；
- 移动、归档和加旗标工具只接收已经确认的目标对象。

Tenant、User、Token 和服务端候选上限不让模型填写，由平台和 Extension 从当前用户请求中取得。

## 一次 Token 优化

平台 Trace 会记录每轮输入和输出 Token。一次版本检查发现，划词解释仍携带邮箱搜索和写工具 Schema；邮箱整理读取邮件详情后，旧搜索候选也继续留在后续轮次。

优化只做了两件事：划词解释移除无关工具，只保留场景 Prompt 和选区 Context；邮箱整理取得邮件详情后，删除已经被替代的候选列表和旧 Tool Result。

在同一批 Golden Set Query 上：

| 场景 | 优化前 | 优化后 | 变化 |
| --- | ---: | ---: | ---: |
| 划词解释平均输入 | 12,400 Token | 7,600 Token | -39% |
| 多轮邮箱整理累计输入 | 46,000 Token | 34,000 Token | -26% |

输出 Token 基本不变，因为回答要求没有缩短。`lm_checklist` 和 Citation 没有下降，工具误选减少后，这个版本才进入 Ring。

## 长对话处理

较早对话不会无限保留原文。项目保留当前用户目标、已确认条件、资源 ID、计划和执行结果，已经被新结果替代的 Tool Result 从下一轮删除。邮件和附件仍保存在 Outlook，需要核实时重新读取。

超过当前模型预算时，先删除旧 Tool Result 和重复引用，再缩短较早对话；单条邮件过长时才截断，并把结果标记为 `truncated` 或 `partial`。项目没有设定一个跨模型版本永久不变的 Token 上限。

## 评测与定位

Prompt、Context 和工具 Description 的修改都通过相同 Golden Set Query 比较。当前评测基线中，`lm_checklist` 为 **87.6%**、Citation 为 **96.2%**、`tool_call` 为 **93.8%**。如果结果下降，先看 Trace 中模型实际收到的 Context 和第一次 Tool Call，再判断是 Prompt、Context 还是依赖 Extension 的问题。
