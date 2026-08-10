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

项目没有维护一份覆盖所有场景的超长 Prompt。Declarative Agent 的 `instructions` 保留通用回答和安全规则，划词解释、附件总结和邮箱整理的场景规则分别写当前任务的目标、可用信息、工具使用条件和完成要求。

这种拆分主要解决实际维护问题：修改邮箱整理的搜索规则时，不应改变划词解释；附件提取失败的处理，也不应该进入普通问答。通用 Instructions 和场景规则随 Agent Definition 一起版本化发布。

## 从头设计一项能力
{: #prompt-context-design }

我不会先写 Prompt，而是按五步设计。以“把最近两周需要我跟进的邮件移到 Follow Up”为例：

1. **定义成功和风险**：成功不是生成一段整理建议，而是找对邮件、展示动作范围、取得确认，并逐项返回真实结果；主要风险是范围理解错误、未确认写入和重复执行；
2. **确定最小 Context**：首轮只需要用户输入、当前 Folder 和入口类型，不把整个邮箱放进 Context；搜索返回 Top 12 Snippet，只有无法判断是否需要跟进时才读取邮件详情；
3. **划分工具边界**：搜索工具发现未知邮件，读取工具取得已知 Message ID 的详情，写工具只处理已确认对象；身份、权限和服务端上限不让模型生成；
4. **组织 Instructions 和状态规则**：规定什么时候澄清、搜索、读取、生成计划、确认和停止；用户修改时间、排除项或目标 Folder 后旧计划失效；
5. **用评测迭代**：Golden Set 覆盖目标模糊、用户改口、版本冲突、部分成功和结果未知，通过 Trace 找到第一次偏离，再修改 Instructions、Context Config 或 Function Description。

发布产物中只有一个 `instructions` 字段，我按下面的顺序组织其中内容：

```text
Agent 目标和回答边界
→ 所有场景共用的安全规则
→ 根据入口类型和 Context 区分场景
→ 划词解释规则
→ 附件总结规则
→ 邮箱整理的澄清、搜索、读取、计划和确认规则
→ Tool Result 的错误处理和任务结束条件
```

邮箱整理部分的核心规则接近下面这种形式：

```text
如果用户要求整理邮箱：
1. 提取时间、筛选条件、排除项、动作和目标 Folder。
2. 动作或目标 Folder 不明确时先澄清，不生成写 Tool Call。
3. 先搜索候选；Snippet 不足以判断时，再按 Message ID 读取详情。
4. 写操作前展示筛选条件、目标对象和动作，并等待平台确认。
5. 用户修改范围后废弃旧计划，重新搜索并确认。
6. 根据 version_conflict、partial_success 和 result_unknown 分别处理，不能把未知结果说成成功。
```

这里 Prompt Engineering 负责**决策规则和行为边界**，Context Engineering 负责**每轮让模型看到哪些事实，以及何时用新结果替换旧结果**；二者要与 Function Description、参数 Schema、平台确认和 Extension 权限一起工作。

## 三类 Context

### 划词解释

Context Config 要求 Sydney 注入选中文字、前后各一个段落、邮件主题、发送者、时间、Message ID 和入口类型。`instructions` 规定选区足以回答时直接解释，不读取整封邮件，也不调用邮箱搜索和写工具。

### 附件总结

Context Config 只要求 Sydney 注入 Message ID 和 Attachment ID，附件提取 Extension 返回文本、页码或章节位置、提取状态和 Citation。内容不完整时保留 `partial` 状态，回答只总结已经取得的部分。原始二进制和 OCR 中间数据不进入模型 Context。

### 邮箱整理

Context Config 只要求 Sydney 注入当前 Folder。初始 Context 再加上用户目标和同一任务中已经确认的条件；Agent 先调用 Insight Query 取得 Top 12 候选，需要进一步判断时再读取邮件详情，不会把整个文件夹预先放进 Context。

邮件正文、附件和搜索 Snippet 都按外部数据处理，不能覆盖 Agent Instructions，也不能改变工具权限和确认要求。

## 设计怎样落到配置和状态边界

项目没有实现一套自建状态机服务，业务设计分布在四处：

| 位置 | 我的设计 | 谁负责执行或保存状态 |
| --- | --- | --- |
| Declarative Agent `instructions` | 在同一个字段中按场景写明什么时候澄清、搜索、读取、确认和停止；用户修改范围后旧计划失效；不同 Tool Result 返回后怎样继续 | Microsoft 365 Copilot Agent Runtime 执行规则，并保存 Conversation、Plan 和确认状态 |
| API Plugin Manifest | Agent 全局可用的 Function、Description、参数 Schema、确认与安全语义，以及 `reasoning`、`responding` 阶段怎样理解 Tool Result | BizChat 将 Function 提供给模型并调度 Extension |
| Sydney Context Config | 每次请求需要注入的 Message、选区、Attachment、Folder 和入口类型 | Sydney 接入层采集并注入 Context |
| Extension 契约 | `version_conflict`、`partial_success`、`result_unknown`、Operation ID 等结果语义 | Extension 保存资源版本、幂等记录和实际邮件操作结果 |

状态边界也相应分开：界面状态由 Sydney 每次注入；用户目标、已确认条件、Plan 和确认结果由 Microsoft 365 Copilot Runtime 保存；邮件操作结果、资源版本和 Operation ID 由 Extension 保存；哪些状态可以沿用、哪些变化必须重新读取或确认，则由我写进 Instructions 和 Function 行为规则。

例如用户把目标 Folder 从 `Follow Up` 改成 `Archive` 时，Instructions 规定旧计划失效，由 Runtime 重新生成 Plan 并确认；写工具返回 `version_conflict` 时，Function 行为规则要求重新读取邮件，真实资源版本仍由 Extension 维护。

## 工具描述

工具名称和 Description 直接影响模型选择。项目只写清楚三个问题：什么时候调用、返回什么、不能替代什么。

- `search_outlook_context` 用来发现未知邮件，不代替已知 Message ID 的读取；
- `read_outlook_message` 读取指定邮件，不执行写操作；
- 移动、归档和加旗标工具只接收已经确认的目标对象。

Tenant、User、Token 和服务端候选上限不让模型填写，由 BizChat 和 Extension 从当前用户请求中取得。

## 长对话处理
{: #dynamic-context-trimming }

较早对话不会无限保留原文。项目保留当前用户目标、已确认条件、资源 ID、计划和执行结果，已经被新结果替代的 Tool Result 从下一轮删除。邮件和附件仍保存在 Outlook，需要核实时重新读取。

超过当前模型预算时，先删除旧 Tool Result 和重复引用，再缩短较早对话；单条邮件过长时才截断，并把结果标记为 `truncated` 或 `partial`。删除已被邮件详情替代的搜索候选和旧 Tool Result 后，每条多轮邮箱整理 Scenario 中所有模型调用的平均总输入从 **46,000 Token 降到 34,000 Token**，下降 **26%**。

## 一次根据评测完成的优化
{: #evaluation-driven-optimization }

一次 Instructions 更新后，划词解释专项集的 **80 条 Query 中有 11 条误调用了 `search_outlook_context`**。最终回答有时仍然可读，但工具选择断言已经失败，而且平均输入 Token 明显增加。

我对比同一 Query 的新旧 Trace，发现第一次差异就在模型首轮：Sydney 已经注入选区和入口类型，新版本仍先搜索邮箱。问题不是 Insight Extension，而是三处边界不清楚：通用 `instructions` 把“证据不足时搜索”写得太宽；选区规则没有明确优先级；搜索 Function Description 没说明不能替代当前选区。

旧版通用规则只有一句：

```text
If the available context is insufficient, search Outlook for more evidence before answering.
```

它没有区分“当前选区解释”和“跨邮箱查找”，所以模型认为任何证据不足都可以搜索。新版没有继续追加一句模糊的“不要搜索”，而是按入口类型明确分支：

```text
When entryType is selection:
- Treat selectedText as the primary subject of the request.
- Use surroundingParagraphs only to resolve local meaning.
- Do not call search_outlook_context unless the user explicitly asks to find or compare other emails.
- If the selection is still ambiguous, explain the ambiguity or ask one clarification question.
```

Context Config 也从“当前邮件可用字段全部注入”改成只保留：

```text
entryType
selectedText
surroundingParagraphs
subject
sender
sentAt
messageId
```

删除完整邮件正文、Conversation 历史和附件列表。与此同时，搜索 Function Description 从普通的“Search the user's Outlook mail”改成：

```text
Search Outlook to discover emails that are not already identified in the current context.
Do not use this function to explain selected text or to reread a known Message ID.
```

修复后，80 条专项 Query 不再误调搜索；每条 Query 中所有模型调用的平均总输入从 **12,400 Token 降到 7,600 Token**，下降 **39%**。随后运行三项功能的全量 Golden Set，`lm_checklist` 和 Citation 没有下降，版本才重新进入 Ring。

## 评测与定位

Prompt、Context 和工具 Description 的修改都通过相同 Golden Set Query 比较。当前评测基线中，`lm_checklist` 为 **87.6%**、Citation 为 **96.2%**、`tool_call` 为 **93.8%**。如果结果下降，先看 Trace 中模型实际收到的 Context 和第一次 Tool Call，再判断是 Prompt、Context 还是依赖 Extension 的问题。完整排查过程见[工具误选与发布回滚]({{ site.baseurl }}/docs/career/copilot-capabilities/incident-tool-routing/)。
