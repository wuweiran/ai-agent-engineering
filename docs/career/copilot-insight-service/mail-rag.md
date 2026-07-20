---
layout: default
title: Outlook 邮件 RAG
parent: Outlook Copilot Insight Service
grand_parent: 工作经历
nav_order: 2
permalink: /docs/career/copilot-insight-service/mail-rag/
---

# Outlook 邮件 RAG

## Query 检索链路

`/insights/query` 接收自然语言 Query、绝对时间、稳定 Person ID 和当前邮件 Anchor。Insight Service 校验身份与参数后，将这些条件交给 Outlook Search，召回邮件候选，再执行对象级权限过滤、Conversation 去重和排序，最终向模型返回轻量候选。

```text
Query 与过滤条件
→ Outlook Search 分页召回
→ 对象级权限过滤
→ Message 与 Conversation 去重
→ 多特征打分
→ Top-K
→ Message ID、Conversation ID、Snippet 和 Citation
```

Query 阶段不读取所有邮件全文。候选只包含排序需要的元数据和 Snippet；模型确定需要某封邮件或某个 Conversation 后，才继续读取详细证据。

## 用户身份与权限过滤

用户在 Outlook 中登录后，Copilot Runtime 代表当前用户调用 Insight Service，并转发 Microsoft Entra ID 用户访问令牌。模型不能在 Tool Call 中填写 Tenant ID、User ID 或权限。Insight Service 验证令牌的签发方、Audience、有效期和 Scope，再从 Claims 中取得 Tenant ID 和 User Object ID。

Insight Service 调用 Outlook Search 时使用 **On-Behalf-Of（OBO，代表用户）**流程，将上游用户令牌换成面向 Outlook 搜索服务的下游访问令牌。下游请求继续带着用户身份执行，而不是使用拥有全租户邮箱权限的应用身份。

```text
Outlook 用户登录
→ Copilot Runtime 获得用户 Token
→ Runtime 调用 Insight Service
→ Insight Service 验证 Token Claims
→ OBO 换取 Outlook Search Token
→ Outlook Search 按用户权限召回邮件
```

权限过滤分成两层：

1. **数据源权限裁剪**：Outlook Search 根据用户身份只搜索用户有权访问的邮箱、共享邮箱和文件夹，未授权对象不会进入候选集；
2. **候选对象校验**：Insight Service 检查候选的 Tenant、Mailbox 和对象标识是否与当前身份及请求范围一致，再生成 Snippet 和 Citation。

第二层校验不能替代数据源授权，作用是防止错误路由、跨租户 ID、过期客户端 Anchor 或下游异常结果进入模型 Context。被过滤的对象不会以标题、Snippet、Citation 或命中数量暴露。

稳定 Person ID 只用于匹配邮件的 From、To 和 Cc，不授予读取权限。即使模型传入其他租户或用户的 Person ID，最终仍只能搜索当前访问令牌有权读取的邮件。

## 打分与排序

对象级权限过滤和硬性时间条件先执行，不满足条件的邮件不会参与打分。剩余候选使用确定性公式重排，Insight Service 不调用 LLM 或额外的 Reranker：

```text
FinalScore =
    0.65 × SearchRelevance
  + 0.15 × Recency
  + 0.15 × ParticipantMatch
  + 0.05 × AnchorRelation
```

四个特征都归一化到 0～1：

- **SearchRelevance**：Outlook Search 的原始相关性分数按离线标注集校准到 0～1；
- **Recency**：以请求结束时间为基准做时间衰减，越接近用户指定时间范围的末端分数越高；
- **ParticipantMatch**：查询中的 Person ID 命中发件人得 1，命中 To 或 Cc 得 0.7，没有命中得 0；
- **AnchorRelation**：与当前邮件属于同一 Conversation 得 1，位于当前 Folder 得 0.5，没有界面 Anchor 关系得 0。

搜索相关性占主要权重，其他特征只用于调整顺序，不能把语义不相关的邮件推到前面。权重通过带相关邮件标注的 Query 集调整，并按 Query 类型检查：人员查询不能只依赖时间，当前邮件追问也不能让 Anchor 完全压过 Query 相关性。

分数相同时，先选择发送时间较新的邮件，再使用 Message ID 做稳定排序，保证同一批候选重复执行时结果顺序一致。

## 候选过多与分页

时间范围较长或 Query 较宽时，Outlook Search 可能命中数千封邮件。Insight Service 不把所有结果拉回内存，而是使用搜索服务提供的游标分页：

- 每页读取 **50 条**候选；
- 最多读取 **4 页**，单次请求最多评估 **200 条**候选；
- 每页返回后立即完成权限过滤、去重和打分；
- 内存中只保留分数最高的 **12 条**候选；
- Top-K 已经稳定，或者已经处理 200 条时停止翻页。

使用游标而不是 Offset 分页，是因为搜索结果可能在查询期间变化，Offset 越大，重复和漏读越明显。游标由 Outlook Search 返回，Insight Service 只在当前请求中继续使用，不向模型暴露。

如果 200 条候选中仍没有足够证据，接口通过 `partial` 和 `warnings` 说明搜索范围受限。模型可以缩小时间、补充人员或改写 Query 后发起新请求，而不是无限翻页。

## 有界 Top-K

最终 **K 取 12**。我们在带相关邮件标注的 Query 集上比较 K=5、8、12 和 20：K 从 5 增加到 12 时 Recall@K 明显提高，能够覆盖同一问题涉及的多封邮件；从 12 增加到 20 后召回提升已经很小，但响应体、模型选择噪声和后续工具调用都会增加。因此线上固定返回 12 条，模型不能自行扩大 K。

服务不需要保存全部 200 条候选。处理每一页时维护：

- Message ID 去重集合；
- Conversation ID 到当前最佳候选的映射；
- 容量为 12、按 FinalScore 排序的最小堆。

候选完成权限检查和打分后，先按 Message ID 去重。同一 Conversation 已经存在时，只在新候选分数更高时替换映射中的邮件。随后更新最小堆：

- 堆未满时直接加入；
- 分数低于堆顶时丢弃；
- 分数高于堆顶时移除当前最低分候选并加入新候选；
- 分数相同时按发送时间和 Message ID 执行稳定 Tie-break。

所有页面处理完成后，将堆中 12 条候选按 FinalScore 从高到低排序并生成响应。时间复杂度是 **O(N log K)**，其中 N 最多为 200、K 为 12；内存只保存去重索引和 12 条候选元数据。

这样内存占用由搜索总命中数变成固定的候选池大小。即使邮箱中命中数很大，单次 Query 的服务内存和返回 Token 仍然有上限。

## 候选怎样存储

搜索候选不写入 Cosmos DB，也不长期缓存邮件标题和 Snippet。它们只存在于当前请求的内存中：

```text
请求开始
→ 当前搜索游标
→ 去重集合
→ Top-K 候选元数据
→ 生成响应
→ 请求结束后释放
```

单个候选只保存 Message ID、Conversation ID、Subject、Sender、Sent Time、Snippet、搜索分数和 Citation，不保存邮件全文。这样既减少内存，也避免建立一份难以保持权限和时效一致的邮件副本。

持久化的只有运行指标和 Trace 元数据，包括 Request ID、候选数量、过滤数量、读取页数、耗时和错误类型。日志不记录邮件正文、完整 Query 或 Snippet。

Query 是只读操作，请求中途失败后不需要恢复原来的分页现场。Runtime 使用同一组 Query 参数重新调用即可；重新搜索得到的是当时邮箱中的最新结果。

## 分页为什么不直接交给模型

模型需要的是最相关证据，而不是操作搜索页码。如果把游标交给模型，它需要判断何时翻页、翻多少页，还会增加工具轮次、延迟和 Token。

Insight Service 因此在一次调用中完成有界分页和 Top-K。只有搜索范围过宽或结果不足时，才通过 `partial` 和 `warnings` 返回原因，让模型缩小时间范围、补充 Person ID 或改写 Query 后发起新请求。延迟拆分、提前停止、连接池和缓存策略见 [Query 性能与缓存]({{ site.baseurl }}/docs/career/copilot-insight-service/performance/)。
