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

- **任务状态**：保存当前目标、已确认事实、进度和恢复位置，回答“任务怎样继续”；
- **短期记忆**：当前 Context 中可直接使用的近期信息；
- **长期记忆**：跨任务仍值得复用的经验、偏好或知识。

Context 只是模型当前看到的工作集，不等于完整任务状态；RAG 外部知识也不等于用户记忆，它由独立知识库维护。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## Agent 的长期记忆怎样实现？
{: #agent-long-term-memory }

长期记忆需要**写入、存储、取回**三条能力：

1. **写入**：从任务结果提取候选经验，验证事实、去重，检查权限和适用范围，并记录来源与失效条件；
2. **存储**：偏好、资源 ID 等确定字段适合关系数据库或键值存储；非结构化经验可以保存为文档并建立关键词或向量索引；
3. **取回**：根据当前目标先过滤，再检索和排序，只把少量相关记忆放入 Context。

RAG 是取回手段，不等于长期记忆本身；长 Context 也不能代替跨会话持久化。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## 多轮对话的存储和记忆怎样设计？
{: #conversation-storage-memory }

可以分四层保存：

- **完整会话**：持久化到数据库或对象存储，用于恢复和审计；
- **当前 Context**：只放近期消息和必要摘要；
- **任务状态**：独立保存业务事实、资源 ID、审批和进度；
- **长期记忆**：从会话结果中筛选跨任务仍有价值的经验。

摘要是有损的，容易变化的业务事实应从权威系统重新读取；不同用户、租户和频道必须隔离。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## Agent 的记忆怎样更新和遗忘？
{: #memory-update-forgetting }

- **更新**：追加新经验、覆盖单值事实、合并冲突信息，或重新生成摘要；
- **遗忘**：根据有效期、相关性、访问情况和保留政策删除或压缩内容；
- **安全边界**：保留来源、适用范围和失效条件，不把模型猜测自动写成事实，也不让不同租户的记忆互相污染。

相关内容：[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## Claude Code 怎样管理当前 Context、会话记录和跨会话记忆？
{: #claude-code-context-memory }

- **Context Window**：下一次模型调用实际可见的信息工作集；
- **Session Transcript**：较完整的消息和工具记录，用于恢复与审计；
- **Auto Memory**：未来会话仍值得复用的项目经验；
- **项目文件与 `CLAUDE.md`**：分别保存代码事实和团队规则。

Context 接近上限时，Claude Code 会清理旧工具结果并压缩历史，但不会因此删除完整 Transcript。压缩后继续改代码时，仍应重新读取磁盘上的当前文件。

相关内容：[Claude Code Context 组装]({{ site.baseurl }}/docs/ai-agent/claude-code/context-assembly/)、[Claude Code 长会话]({{ site.baseurl }}/docs/ai-agent/claude-code/session-compaction/)。

## 长对话中怎样保留关键信息？除了向量检索还有什么方法？
{: #long-context-key-information }

向量检索只解决“找候选”，长对话还需要：

1. **结构化状态**：保存目标、约束、已确认事实、待办和资源 ID；
2. **分层保留**：近期消息保留原文，较早过程使用滚动或分层摘要；
3. **外部存储**：大型日志、文档和工具结果留在外部系统，按需重读；
4. **Context 清理**：删除过期结果，把重要目标和约束固定在清楚位置；
5. **Context 隔离**：复杂子任务使用独立 Context，只返回结论、证据和未知项。

金额、审批和工具执行结果等权威事实不能只存在于摘要，因为摘要是有损的。

相关内容：[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)、[Agent 状态与记忆]({{ site.baseurl }}/docs/ai-agent/agent-design/state-memory/)。

## Lost in the Middle 是什么？怎样缓解？
{: #lost-in-the-middle }

**Lost in the Middle 指模型在长 Context 中没有稳定利用位于中间的关键信息。**内容仍在窗口里，不代表一定会被使用。

缓解手段包括：减少无关材料、Rerank 高价值候选，把目标、约束和关键证据放在清楚位置。位置重排只能缓解注意力问题，不能修复错误检索和过期知识。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## Context 噪声太多时怎样处理？
{: #context-noise-cleaning }

按三层清理：

1. **确定性过滤**：按来源、时间、权限、版本和相关性先排除明显无效内容；
2. **模型筛选**：让模型标记无关、冲突和过期材料，必要时通过 Self-Refine 迭代清洗；
3. **独立验证**：高风险内容依靠可信来源、规则或独立评审，不能让同一个模型随意删除反面证据。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## Agent Context Engineering 有哪些主要技术？管理策略怎样设计？
{: #agent-context-engineering-strategy }

可以按 Context 生命周期记忆：

- **获取**：RAG、工具查询、记忆检索；
- **选择**：权限与时效过滤、Rerank、去重和相关性判断；
- **组织**：按权威性组装系统规则、目标、状态、证据和近期历史；
- **压缩**：滑动窗口、摘要、Context Editing、工具结果裁剪；
- **隔离**：让子 Agent 使用独立 Context，只返回结论和证据；
- **复用**：稳定前缀使用 Prompt Cache，减少重复计算。

管理策略是：**先定义当前判断需要什么和 Token 预算，再为每类信息指定保留方式。**目标、约束和待办进入结构化状态；近期对话保留原文；早期过程摘要；动态事实重新查询；完整资料留在外部系统。最后用长任务、冲突资料、过期信息和噪声样本评测质量、延迟与成本。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## 一次 Agent 调用的 Context 怎样组装？
{: #agent-context-assembly }

通常按来源和权威性分层：

1. **稳定规则**：System Prompt、权限说明和工具定义；
2. **当前任务**：目标、约束和持久任务状态；
3. **相关长期记忆**：按当前目标检索出的用户偏好、项目经验或已验证历史结论；
4. **当前证据**：知识检索材料、实时业务数据和 Tool Result；
5. **连续性信息**：近期会话与必要摘要；
6. **本轮输入**：用户当前请求和输出要求。

组装前先按租户、权限、适用范围、版本和时效过滤，再做排序、去重和 Token 预算分配。长期记忆只有被检索并放入本轮 Context 才会影响模型，不能把其他用户或租户的记忆直接带入。外部文档不能与系统规则混成同一权威层；原始日志和完整历史保留在外部系统，需要时重新读取。

相关内容：[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。
