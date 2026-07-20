---
layout: default
title: Outlook Copilot Insight Service
parent: 工作经历
nav_order: 2
has_children: true
permalink: /docs/career/copilot-insight-service/
---

# Outlook Copilot Insight Service

## 项目是什么

组织调整后转入 Outlook 团队，参与 Outlook Copilot Insight Service。Copilot 上线初期曾使用老旧的 .NET UserActivityLogService 作为搜索工具，但延迟高且稳定性不足，很快由 Insight Service 替代。这个服务位于 Outlook 业务数据和 Copilot 之间，为模型准备当前请求真正需要的邮件、会话、日历、联系人和组织信息。

它是一套基于 Outlook 原生搜索和业务对象的 **搜索型 RAG**，不依赖文档 Chunk、Embedding 或向量数据库。Insight Service 负责 Retrieval 和 Context Augmentation，Copilot 模型负责最终 Generation。项目重点不再是怎样调用模型，而是怎样让模型看到正确、及时且有权限的信息；它也是后续 Copilot Agent 能力的数据基础。

## 系统处在什么位置

上游是 Outlook Copilot Runtime。Runtime 将 Insight Service 的能力注册为邮件查询、Conversation Context 和 Calendar Context 三个 Extension；每个 Extension 以 Tool Schema 暴露给模型，并由 Handler 映射到 Insight Service API。模型生成 Tool Call 后，Runtime 代表用户执行调用并附带访问令牌与 Outlook 界面 Context。模型直接提供绝对时间和稳定 Person ID，Insight Service 校验这些参数，访问邮件和日历，完成权限过滤、排序、去重和压缩后返回模型。

```text
Outlook Copilot 请求
→ Insight Service 理解所需信息
→ 邮件、日历、联系人和目录服务
→ 权限与时效过滤
→ 排序、引用去重、结构清理和 Token 控制
→ Context 返回 Copilot
```

## 接口与请求

模型通过 Tool Call 使用 Insight Service，直接传入 Query、绝对时间和稳定 Person ID。Copilot Runtime 负责执行调用并附带访问令牌和 Outlook 界面 Context；候选数量和 Token 预算由 Insight Service 控制。

对外接口只保留邮件 Query、Conversation Context 和 Calendar Context。已知 Message ID 或 Event ID 的详情由 Outlook 读取工具直接调用 Microsoft Graph API；人员 ID 由模型从已有人员或目录 Context 中取得，不由 Insight Service 解析姓名。具体请求与返回结构见 [接口与 Context 请求]({{ site.baseurl }}/docs/career/copilot-insight-service/context-api/)。

## 主要工作

团队共同维护 Insight Service 的邮件 Query、Conversation 和 Calendar Context。我的核心职责是 **`/insights/query` 的邮件检索链路**：将模型传入的 Query、绝对时间、稳定 Person ID 和当前邮件 Anchor 转成 Outlook Search 请求，完成分页召回、对象级权限过滤、Message 与 Conversation 去重、多特征打分和 Top-K 排序，最后返回 Message ID、Conversation ID、Snippet 和 Citation。

Conversation 与 Calendar Context 由团队其他成员负责。我参与 `search_outlook_context` Extension 和 HTTP 契约，以及邮件对象级权限接入；Query 返回 Conversation ID 后，模型可以通过团队提供的 Conversation Extension 继续取得线程证据。

这条工作的重点是**在海量且持续变化的邮箱数据中，以有界延迟和内存找到足够回答问题的候选证据，同时保证对象级权限、结果时效和 Citation**。

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

- **Query 与召回**：怎样使用绝对时间、稳定 Person ID 和当前邮件 Anchor 生成 Outlook Search 条件；
- **分页与有界候选池**：怎样用游标分页处理大量命中，并限制单次请求的下游、内存和延迟成本；
- **权限过滤**：怎样使用用户身份检索并再次校验候选对象，避免未授权标题、Snippet 和命中数量泄露；
- **去重与排序**：怎样按 Message 和 Conversation 去重，并组合搜索分数、时间、邮件参与者匹配与 Anchor 关系选择 Top-K；
- **返回与协作**：为什么 Query 只返回 Message ID、Conversation ID、Snippet 和 Citation，以及模型何时继续读取邮件或 Conversation；
- **RAG 评测**：怎样评估 Recall@K、排序质量、权限泄露、Citation、P95/P99、下游请求量和 Token 成本。

Insight Service 同时属于大模型应用后端和 Agent 工具后端：它为以 Extension 形式注册的 Insight 工具提供后端实现，把实时、受权限控制的 Outlook 数据转换成 Tool Result，Runtime 再将结果放入模型 Context。

## 项目文档

- [接口与 Context 请求]({{ site.baseurl }}/docs/career/copilot-insight-service/context-api/)：邮件 Query、Conversation、Calendar Extension 及各自的请求和返回结构；
- [Outlook 邮件 RAG]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/)：分页召回、有界 Top-K、候选存储和 Conversation 二阶段检索；
- [Query 性能与缓存]({{ site.baseurl }}/docs/career/copilot-insight-service/performance/)：延迟拆分、提前停止、连接池、限流、OBO Token Cache 和缓存边界；
- [Query 评测与发布]({{ site.baseurl }}/docs/career/copilot-insight-service/evaluation/)：Recall@12、MRR、NDCG、权限泄露、Citation、P95/P99 和发布门槛；
- [部署与生产运行]({{ site.baseurl }}/docs/career/copilot-insight-service/production/)：Java 服务、Substrate、实例规格、扩缩容、监控和 Outlook Search 限流事故。

