---
layout: default
title: Outlook Copilot Insight Service
parent: 工作经历
nav_order: 2
has_children: true
permalink: /docs/career/copilot-insight-service/
---

# Outlook Copilot Insight Service

## 核心思路

Copilot 回答邮件问题需要 Outlook 中的实时证据

→ 在 Outlook Search 与 Copilot 之间建设**搜索型 RAG 服务**

→ 邮箱持续变化，Search 候选存在重复和权限边界，同时请求受延迟、内存与 Token 预算约束

→ 通过**权限过滤、去重与业务重排**从分页候选中选出 Top 12

→ 使用**有界检索与降级**控制分页、内存和延迟，再将 Snippet 与 Citation 返回 Copilot

→ Recall@12 93.2%，权限泄露 0，Citation Validity 100%，P95 1.1 秒

**同类项目通常关注：** 召回质量（Recall@12）、排序质量（Top 3 Hit Rate / MRR / NDCG@12）、权限安全（权限泄露）、证据可用性（Citation Validity）、性能（P95 / P99）、成本（平均下游请求数）。

## 项目介绍

Outlook Copilot Insight Service 位于 Outlook 业务数据和 Copilot 之间，负责为模型取得当前请求需要的邮件、会话和日历证据。它是一套基于 Outlook 原生搜索的**搜索型 RAG**：服务负责 Retrieval 和 Context Augmentation，Copilot 模型负责最终 Generation，不使用文档 Chunk、Embedding 或向量数据库。

我的核心职责是 `/insights/query` 的[邮件检索链路]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/#query-retrieval-pipeline)，包括分页召回、[对象级权限过滤]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/#permission-filtering)、Message 与 Conversation 去重、[多特征排序]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/#ranking)和 Top 12 选择。核心难点是在持续变化的邮箱数据中，以有界延迟和内存返回足够且有权限的证据；为此使用[有界候选池]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/#bounded-top-k)、分页提前停止和[超时降级与过载保护]({{ site.baseurl }}/docs/career/copilot-insight-service/performance/#degradation-overload-protection)，并保留 Citation 供模型回答。

## 系统处在什么位置

上游是 Outlook Copilot Runtime。Runtime 将 Insight Service 的能力注册为邮件查询、Conversation Context 和 Calendar Context 三个 Extension；每个 Extension 以 Tool Schema 暴露给模型，并由 Handler 映射到 Insight Service API。模型生成 Tool Call 后，Runtime 代表用户执行调用并附带用户 Token与 Outlook 界面 Context。模型直接提供绝对时间和用户 UUID，Insight Service 校验这些参数，访问邮件和日历，完成权限过滤、排序、去重和压缩后返回模型。

```text
Outlook Copilot 请求
→ Insight Service 理解所需信息
→ 邮件、日历、联系人和目录服务
→ 权限与时效过滤
→ 排序、引用去重、结构清理和 Token 控制
→ Context 返回 Copilot
```

## 接口与请求

模型通过 Tool Call 使用 Insight Service，直接传入 Query、绝对时间和用户 UUID。Copilot Runtime 负责执行调用并附带用户 Token 和 Outlook 界面 Context；候选数量和 Token 预算由 Insight Service 控制。

对外接口只保留邮件 Query、Conversation Context 和 Calendar Context。已知 Message ID 或 Event ID 的详情由 Outlook 读取工具直接调用 Microsoft Graph API；用户 UUID 由模型从已有人员或目录 Context 中取得，不由 Insight Service 解析姓名。具体请求与返回结构见 [接口与 Context 请求]({{ site.baseurl }}/docs/career/copilot-insight-service/context-api/)。

## 这个项目可以深入到哪里

邮件 RAG 主链路是：

```text
用户 Query
→ 邮件候选召回
→ 对象级权限过滤
→ 多特征打分与 Top-K 排序
→ Message 到 Conversation 的二阶段检索
→ 引用正文去重与 Context 选择
→ Token 预算和 Citation
→ Copilot 生成回答
```

这条链路可以深入到以下问题：

- **Query 与召回**：怎样使用绝对时间、用户 UUID 和当前邮件 Anchor 生成 Outlook Search 条件；
- **分页与有界候选池**：怎样用游标分页处理大量命中，并限制单次请求的下游、内存和延迟成本；
- **权限过滤**：怎样使用用户身份检索并再次校验候选对象，避免未授权标题、Snippet 和命中数量泄露；
- **去重与排序**：怎样按 Message 和 Conversation 去重，并组合搜索分数、时间、邮件参与者匹配与 Anchor 关系选择 Top-K；
- **返回与协作**：为什么 Query 只返回 Message ID、Conversation ID、Snippet 和 Citation，以及模型何时继续读取邮件或 Conversation；
- **RAG 评测**：怎样评估 Recall@K、排序质量、权限泄露、Citation、P95/P99、下游请求量和 Token 成本。

后端工程可以深入到：

- **高并发 I/O**：WebFlux 和 WebClient 怎样在等待 Outlook Search 时不占住请求线程；
- **有界资源**：怎样限制分页、候选数量、Top-K 内存、连接池和下游并发；
- **超时与降级**：怎样在 3 秒 Deadline 内停止翻页并返回 `partial`，而不是让整个 Query 失败；
- **限流与过载保护**：怎样按用户、租户和区域限流，并在 Outlook Search 返回 429 时降低并发；
- **权限与缓存边界**：怎样完成用户 Token、OBO 和对象级校验，以及为什么只缓存 Token 和稳定配置，不缓存邮件结果；
- **生产运行**：怎样处理区域路由、无状态扩缩容、阶段 Trace 和下游限流事故。

Insight Service 同时属于大模型应用后端和 Agent 工具后端：它为以 Extension 形式注册的 Insight 工具提供后端实现，把实时、受权限控制的 Outlook 数据转换成 Tool Result，Runtime 再将结果放入模型 Context。

## 项目文档

- [接口与 Context 请求]({{ site.baseurl }}/docs/career/copilot-insight-service/context-api/)：邮件 Query、Conversation、Calendar Extension 及各自的请求和返回结构；
- [Outlook 邮件 RAG]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/)：分页召回、有界 Top-K、候选存储和 Conversation 二阶段检索；
- [Query 性能与缓存]({{ site.baseurl }}/docs/career/copilot-insight-service/performance/)：延迟拆分、提前停止、连接池、限流、OBO Token Cache 和缓存边界；
- [Query 评测与发布]({{ site.baseurl }}/docs/career/copilot-insight-service/evaluation/)：Recall@12、MRR、NDCG、权限泄露、Citation、P95/P99 和发布门槛；
- [部署与生产运行]({{ site.baseurl }}/docs/career/copilot-insight-service/production/)：Java 服务、Substrate、实例规格、扩缩容、监控和 Outlook Search 限流事故。

