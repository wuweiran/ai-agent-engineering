---
layout: default
title: Agent Context 与记忆面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 3
permalink: /docs/interview/ai-agent/context-memory/
---

# Agent Context 与记忆面试题

这些问题区分当前 Context、会话历史、任务状态和跨任务记忆，并说明系统怎样为每次模型调用选择真正需要的信息。

## Agent 的状态、短期记忆和长期记忆有什么区别？
{: #agent-state-and-memory }

任务状态保存当前目标、已确认事实、进度和恢复位置；短期记忆通常指当前 Context 中可直接使用的近期信息；长期记忆保存跨任务仍值得复用的经验或偏好。

Context 是模型当前看到的工作集，不等于完整任务状态。RAG 外部知识也不是用户记忆，它由独立知识库维护。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## Agent 的长期记忆怎样实现？
{: #agent-long-term-memory }

长期记忆要保存在 Context 之外的持久存储中。写入时从任务结果中提取候选经验，经过事实验证、去重、权限与适用范围检查，再附上来源和失效条件；读取时根据当前目标做结构化过滤、关键词或向量检索，排序后只把少量相关内容放进 Context。

存储方式取决于内容。用户偏好、资源 ID 和确定字段适合关系数据库或键值存储；大量非结构化经验可以保存为文档，并使用 Embedding 和向量索引做语义检索。RAG 是取回这些内容的一种方法，不等于长期记忆本身。

长 Context 只是扩大当前调用的工作集，不能代替跨会话持久化。分层摘要或递归压缩可以延长任务，但摘要有损；完整记录、权威状态和可重新读取的资料仍要保存在外部系统。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## 多轮对话的存储和记忆怎样设计？
{: #conversation-storage-memory }

完整会话记录应持久化到数据库或对象存储，当前 Context 只选择近期消息和必要摘要。任务事实、外部资源 ID 与审批不能只存在于对话中；跨会话经验再经过筛选进入长期记忆。

Context 接近上限时可以摘要旧历史，但摘要有损，容易变化的业务事实应从权威系统重新读取。不同用户、租户和频道的会话必须隔离。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## Agent 的记忆怎样更新和遗忘？
{: #memory-update-forgetting }

更新可以采用追加、覆盖单值事实、合并冲突信息或生成摘要。遗忘可以根据有效期、相关性、访问情况和保留政策移除或压缩内容。

重要的是保留来源、适用范围和失效条件。不能把模型猜测自动写成长期事实，也不能让不同用户或租户的记忆相互污染。

## Claude Code 怎样管理当前 Context、会话记录和跨会话记忆？
{: #claude-code-context-memory }

Context Window 是下一次模型调用实际可见的信息工作集；Session Transcript 保存较完整的消息和工具记录，用于恢复与审计；Auto Memory 保存未来会话仍值得复用的项目经验。项目文件和 `CLAUDE.md` 则分别保存代码事实与团队规则。

Context 接近上限时，Claude Code 会清理旧工具结果并压缩历史，但不会因此删除完整会话记录。压缩后若要继续修改代码，仍应重新读取磁盘上的当前文件。

相关内容：[Claude Code Context 组装]({{ site.baseurl }}/docs/ai-agent/claude-code/context-assembly/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## Lost in the Middle 是什么？怎样缓解？
{: #lost-in-the-middle }

模型在长 Context 中可能更容易利用开头和结尾的信息，而忽略中间的关键证据。内容仍在窗口中，不代表会被稳定使用。

可以减少无关材料、通过 Rerank 选出高价值候选，并把重要目标、约束和证据放在清楚位置。位置重排只能缓解问题，不能修复错误检索和过期知识。

## Context 噪声太多时怎样处理？
{: #context-noise-cleaning }

先通过来源、时间、权限和相关性过滤材料，再让模型在回答前标记无关、冲突和过期内容。Self-Refine 一类自反馈方法可以迭代清洗和修订结果。

同一模型可能重复原误判，因此不能让它自行删除关键反面证据。高风险内容仍需确定性规则、可信来源或独立评审。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## 一次 Agent 调用的 Context 怎样组装？
{: #agent-context-assembly }

通常按来源和权威性分层：稳定的系统规则与工具定义在前，当前任务目标和持久状态说明要完成什么，检索材料与实时工具结果提供证据，近期会话和必要摘要补充连续性，最后给出本轮输入与输出要求。具体消息格式取决于 Runtime，但外部文档不能与系统规则混成同一权威层。

组装时先按权限、租户、版本和时效过滤，再做相关性排序、去重和 Token 预算分配。目标、不可违反的约束和关键证据应处于清楚位置；原始日志与完整历史留在外部系统，需要时重新取得。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。
