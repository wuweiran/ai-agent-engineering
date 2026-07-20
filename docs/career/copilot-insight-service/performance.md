---
layout: default
title: Query 性能与缓存
parent: Outlook Copilot Insight Service
grand_parent: 工作经历
nav_order: 3
permalink: /docs/career/copilot-insight-service/performance/
---

# Query 性能与缓存

## 线上性能

`/insights/query` 的生产流量是单区域峰值 **250 RPS**，单次请求最多读取 4 页、评估 200 条邮件候选，并返回 Top 12。优化后的端到端延迟是：

- P50：**180 ms**；
- P95：**620 ms**；
- P99：**1.4 s**；
- 超时率：**0.2%**；
- 成功率：**99.95%**。

接口 Deadline 是 **3 秒**。达到 Deadline 时停止继续翻页，使用已经处理的候选返回 `partial=true` 和 Warning，而不是让整个 Tool Call 失败。

延迟主要来自 Outlook Search。一次 Query 的 P95 分解为：

| 阶段 | P95 |
| --- | ---: |
| Token 验证与 OBO | 25 ms |
| 第一页 Outlook Search | 260 ms |
| 后续分页 | 210 ms |
| 权限校验、去重和打分 | 55 ms |
| Top-K 与响应序列化 | 20 ms |
| 网络与框架开销 | 50 ms |

第一页决定大部分请求的延迟。后续页面依赖上一页游标，不能并行请求；性能优化的重点是减少不必要的翻页，而不是盲目增加并发。

## 查询优化

### 尽早下推过滤条件

绝对时间、Person ID、Folder 和当前邮件 Anchor 尽量传给 Outlook Search，在数据源侧缩小候选。服务端过滤只能删除已经返回的数据，无法节省下游检索时间和网络开销。

Query 缺少时间范围时，服务默认只搜索最近 **90 天**。模型明确请求更早内容时可以扩大到 **1 年**，再大的范围要求收紧 Query，避免一次工具调用扫描整个邮箱历史。

### 分页提前停止

第一页返回 50 条候选后立即完成权限过滤、去重和打分。满足以下条件时不继续翻页：

- Top 12 已填满；
- 第 12 名分数高于 **0.72**；
- 当前页最后一名的校准搜索分数低于 **0.35**。

这表示已经保留 12 条高质量候选，后续低分页面很难改变 Top-K。提前停止把平均搜索页数从 **3.1 页降到 1.7 页**，P95 从 **980 ms 降到 620 ms**。

### 有界内存和返回体

请求内只保存去重索引和 Top 12 元数据，不保存完整邮件正文。单个 Query 的内存占用低于 **256 KB**，响应体 P95 为 **18 KB**。这让服务实例可以同时处理更多请求，也减少 Tool Result 进入模型 Context 的 Token。

### 连接与并发控制

Outlook Search 使用共享 HTTP/2 连接池，每个服务实例保持 **100 个并发连接**。单实例允许 **200 个并发 Query**，超过后进入有界队列；队列超过 **500 条**时返回过载错误，避免排队时间侵占 3 秒 Deadline。

下游出现 429 时遵守 `Retry-After`，单次 Query 只重试 **1 次**。搜索是只读操作，但多次重试会放大下游负载并增加 Agent 工具延迟，因此不会在一个请求中持续重试。

## 缓存策略

### 不缓存搜索结果

Query 结果、Snippet 和邮件正文不缓存。原因有三点：

- 邮件和权限持续变化，缓存容易返回已删除、移动或失去权限的对象；
- 缓存键必须包含 Tenant、User、Query、时间、Person、Folder 和 Anchor，命中率低；
- 缓存邮件内容会产生额外的数据驻留、删除同步和合规负担。

相同 Query 重新执行时直接访问 Outlook Search，取得当时最新且符合当前权限的结果。服务也不把候选写入 Cosmos DB。

### 缓存 OBO Token

OBO 换取的 Outlook Search Token 按用户、租户、Scope 和目标资源缓存在内存中，在 Token 到期前 **5 分钟**停止复用。缓存容量是每实例 **5 万条**，使用 LRU 淘汰，命中率为 **96%**。

缓存只保存访问令牌及到期时间，不缓存邮件权限结果。Token 获取失败时不复用过期 Token，也不降级为应用身份。

### 缓存稳定元数据

以下稳定数据使用内存缓存：

- Tool Schema 与接口配置；
- 排序权重和阈值；
- Tokenizer 配置；
- 下游 Endpoint 和功能开关。

这些配置带版本号，发布或配置变更时主动失效。Person ID 和对象权限不放入跨请求长期缓存，避免组织关系或授权变更后继续使用旧结果。

## 降级与过载保护

Outlook Search 延迟升高时，服务按剩余 Deadline 决定是否继续翻页。已经有候选就返回部分结果，没有候选则返回数据源暂时不可用。`partial` 和 `warnings` 会进入 Tool Result，模型可以说明证据范围或收紧 Query 后重试。

限流分为用户、租户和区域三层：

- 单用户每秒 **5 个 Query**；
- 单租户每秒 **50 个 Query**；
- 区域总入口 **250 RPS**。

限流防止单个用户或租户占满下游配额。服务实例根据 RPS、P95 和队列长度扩缩容，但扩容不能突破 Outlook Search 的区域配额。

## 性能验证

每次排序、分页或连接池修改都使用同一组 Query 和邮箱快照做回归，比较：

- Recall@12 和 Top 3 命中率；
- 平均搜索页数和下游请求数；
- P50、P95、P99 和超时率；
- 响应大小和 Tool Result Token；
- `partial` 比例；
- 权限过滤结果。

性能优化不能只降低延迟。提前停止、候选上限和默认时间范围都可能降低召回，因此只有在 Recall@12 没有显著退化时才发布。完整指标与门槛见 [Query 评测与发布]({{ site.baseurl }}/docs/career/copilot-insight-service/evaluation/)。
