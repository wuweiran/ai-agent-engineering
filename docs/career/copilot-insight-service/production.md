---
layout: default
title: 部署与生产运行
parent: Outlook Copilot Insight Service
grand_parent: 工作经历
nav_order: 5
permalink: /docs/career/copilot-insight-service/production/
---

# 部署与生产运行

## 技术栈

Insight Service 使用 **Java 17、Spring Boot 3 和 WebFlux** 开发，部署在微软内部的 **Substrate 基础设施**上。服务使用 WebClient 异步调用 Outlook Search 和 Microsoft Graph，通过 MSAL4J 完成 Microsoft Entra ID Token 验证和 On-Behalf-Of 换取下游令牌。

Query 链路没有自己的邮件数据库。邮件候选来自 Outlook Search，只在当前请求内存中维护游标、去重集合和 Top 12 最小堆。服务使用 OpenTelemetry 记录 Metrics、Log 和 Trace，并接入内部监控和告警平台。

```text
Outlook Copilot Runtime
→ Substrate 区域流量入口
→ Insight Service 实例
   ├─ Query API
   ├─ Conversation Context API
   └─ Calendar Context API
→ Outlook Search、Microsoft Graph 和 Entra ID
```

## 区域与实例

服务按 Outlook 数据驻留要求部署在多个 Azure Geography，用户请求进入邮箱数据所在区域，不跨区域检索邮件。Substrate 负责区域流量路由、服务发现、实例编排、健康检查和滚动发布，我们只配置服务资源、最小实例数、最大实例数和扩缩容策略。

每个区域基线运行 **8 个服务实例**，分布在 3 个故障域；高峰扩到 **32 个实例**。单个实例配置：

- **4 vCPU**；
- **16 GB 内存**；
- JVM Heap 上限 **10 GB**；
- 单实例最大处理 **200 个并发 Query**；
- Outlook Search HTTP/2 连接池上限为 **100 个连接**。

服务使用滚动发布，先让新版本承接 **5% 流量**，观察 Query 质量和 P95/P99 后逐步放量。Substrate 健康检查失败后停止向实例发送新请求；正在执行的 Query 在 3 秒 Deadline 内完成或返回部分结果。

## 扩缩容与容量

单区域生产峰值为 **250 RPS**，接口成功率为 **99.95%**。Substrate 根据 CPU、实例 RPS 和入口队列自动扩缩容：

- CPU 连续 10 分钟超过 **65%**时扩容；
- 单实例 RPS 超过 **35**时扩容；
- 区域入口队列超过 **500 条**时扩容；
- CPU 连续 30 分钟低于 **30%**且队列低于 **100 条**时缩容；
- 最少保留 8 个实例，最多 32 个实例。

扩容不会取消下游保护。单用户限制为 **5 Query/s**，单租户限制为 **50 Query/s**，区域入口限制为 **250 RPS**。超过限制时返回明确的限流错误，Runtime 可以降低调用频率，但不会让请求在服务内无限排队。

Outlook Search 是主要容量依赖。每个 Query 最多读取 4 页、评估 200 条候选；提前停止后平均读取 **1.7 页**，平均下游请求数为 **1.8 次**。区域容量评估同时检查 Insight Service 实例和 Outlook Search 配额，不能只增加服务实例。

## 缓存与数据

服务不缓存 Query 结果、Snippet 和邮件正文，也不将候选写入数据库。请求结束后释放搜索游标、去重集合和 Top-K 元数据，避免邮件删除、移动或权限变化后仍返回旧结果。

每个实例在 JVM 内存中缓存 OBO Token 和稳定配置：

- OBO Token Cache 容量为 **5 万条**，命中率为 **96%**；
- Token 到期前 5 分钟停止复用；
- Tool Schema、排序权重、阈值、Endpoint 和功能开关按版本缓存；
- 不缓存对象权限判断和 Person ID。

因此，服务实例可以随时替换，不需要迁移请求状态或本地数据。

## 线上监控

监控分为业务与 RAG、下游依赖、权限安全和服务资源四层。

### 业务与 RAG

- Query 请求量、成功率、无结果率和 `partial` 比例；
- Raw Recall@200、Recall@12、Top 3 Hit Rate、MRR 和 NDCG@12 的离线回归结果；
- 平均搜索页数、候选召回数、权限过滤数、去重数和最终返回数；
- Conversation 后续读取率、Query 改写率和用户重试率；
- Citation Validity 和 Snippet Consistency；
- Tool Result 的响应大小和 Token 数。

离线质量指标不能实时从每个生产请求计算。线上使用无结果率、后续读取率、Query 改写率和用户重试率作为质量代理，并通过抽样数据回流离线评测集。

### 延迟与下游

- Query P50 **180 ms**、P95 **620 ms**、P99 **1.4 s**；
- 超时率 **0.2%**，`partial` 比例 **0.7%**；
- Token 验证与 OBO、第一页搜索、后续分页、权限过滤、打分和序列化的阶段耗时；
- Outlook Search 的请求量、P95/P99、429、5xx、超时和 Retry-After；
- OBO Token Cache 命中率与 Token 获取失败率；
- 平均搜索页数和下游请求数。

P95 超过 **700 ms**或 P99 超过 **1.5 秒**持续 10 分钟触发告警。Outlook Search 429 超过 **1%**时降低区域并发，避免重试继续放大下游压力。

### 权限与安全

- Token 验证失败、Audience 或 Scope 不匹配；
- OBO 失败和跨租户请求；
- 候选对象二次校验失败数；
- 未授权 Subject、Snippet、Citation 或命中数量泄露；
- 敏感字段进入 Log 或 Trace 的检测结果。

权限泄露容忍值为 **0**。出现任何未授权内容返回时立即停止版本放量并启动安全事件处理，不等待错误率达到阈值。

### 服务资源

- 服务实例数、健康实例比例、实例重启和异常退出；
- CPU 保持在 **35%～55%**，内存保持在 **6～10 GB**；
- JVM Heap、Allocation Rate、Old GC 和 GC Pause；
- Reactor Event Loop 延迟、阻塞调用和 Scheduler 队列；
- HTTP 连接池占用、DNS 和 TLS 建连错误；
- 请求并发、入口队列、拒绝数和限流数。

Trace 使用 Request ID 串联 Runtime Tool Call、Insight Query、OBO、Outlook Search 分页、权限过滤、重排和响应序列化。邮件正文、完整 Query 和 Snippet 不写入 Log 或 Trace。

## Outlook Search 限流事故

一次排序版本发布后，Query P99 从 **1.4 秒**升到 **4.8 秒**，`partial` 比例从 **0.7%**升到 **12%**，Outlook Search 429 达到 **8.2%**。服务实例 CPU 和内存正常，但区域入口队列持续增长，说明瓶颈不在 Insight Service 计算资源。

按版本和阶段拆分 Trace 后发现，第一页搜索延迟没有明显变化，平均搜索页数却从 **1.7 页**升到 **3.6 页**。该版本为了提高 Recall@12，提高了提前停止阈值，更多请求会继续读取第三页和第四页。下游请求量随之翻倍，触发 Outlook Search 区域限流；Insight Service 对 429 的一次重试又进一步放大了请求量。

```text
提前停止条件收紧
→ 平均搜索页数从 1.7 升到 3.6
→ Outlook Search 请求量翻倍
→ 下游开始返回 429
→ 同一 Query 重试
→ 下游压力继续增加
→ P99、partial 和入口队列持续上升
```

应急处理先回滚排序与提前停止配置，随后临时关闭 429 自动重试，并把区域 Query 并发从 **250 RPS**降到 **160 RPS**。已有候选的请求直接返回 `partial=true`，没有候选的请求返回数据源繁忙，避免请求在入口队列中等待到 Deadline。

回滚后 429 在 15 分钟内降到 **0.6%**，P99 恢复到 **1.5 秒**以内。根因不是排序公式本身，而是排序门槛改变了分页行为，却只检查了单请求 Recall@12 和延迟，没有评估区域总下游请求量。

最终修复包括：

1. 恢复第 12 名 **0.72**和页尾搜索分数 **0.35**的提前停止条件；
2. 将平均搜索页数和每 Query 下游请求数加入发布门槛；
3. Outlook Search 429 超过 **1%**时自动降低区域并发；
4. 严格遵守 `Retry-After`，同一 Query 最多重试一次，并将重试计入剩余 Deadline；
5. 容量测试同时模拟多个实例的区域总流量，不再只测试单实例吞吐。

这次事故说明，RAG 排序和性能不能分开优化。一个看似只影响 Recall 的阈值，会改变分页深度和下游放大倍数；发布时必须同时比较 Recall@12、平均页数、下游请求数、P95/P99 和 429。

## 值班处理

告警先判断影响发生在哪一层：

- P95/P99 上升且 Outlook Search 同时变慢，优先限制分页和区域并发；
- 实例 CPU、Reactor Event Loop 延迟或 Scheduler 队列上升，但下游正常，检查阻塞调用、序列化和连接池；
- `partial` 和无结果率上升但延迟正常，检查过滤条件、提前停止和排序版本；
- OBO 失败集中增加，检查 Entra ID、Scope 和 Token Cache；
- Citation 或权限检查失败，立即停止发布并隔离受影响版本。

恢复后使用同一邮箱快照重跑 Recall@12、权限和 Citation 回归，再逐步恢复流量。性能恢复但检索质量未通过时，版本仍不能继续放量。
