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

组织调整后转入 Outlook 团队，参与 Outlook Copilot Insight Service。这个服务位于 Outlook 业务数据和 Copilot 之间，为模型准备当前请求真正需要的邮件、会话、日历、联系人和组织信息。

项目重点不再是怎样调用模型，而是怎样让模型看到正确、及时且有权限的信息。它属于大模型应用中的 Context Engineering，也是后续 Copilot Agent 能力的数据基础。

## 系统处在什么位置

上游是 Outlook Copilot Orchestrator。它根据用户请求向 Insight Service 查询所需信息。下游包括邮件与会话搜索、日历、联系人、组织目录和权限相关服务。Insight Service 负责取得候选、过滤、排序、去重和压缩，再把受控结果返回 Copilot。

```text
Outlook Copilot 请求
→ Insight Service 理解所需信息
→ 邮件、日历、联系人和目录服务
→ 权限与时效过滤
→ 排序、去重、摘要和长度控制
→ Context 返回 Copilot
```

## 主要工作

主要工作包括数据源接入、权限过滤、相关性排序、会话去重、摘要和 Context Token 预算。服务既不能把所有可访问信息都交给模型，也不能因为过度裁剪而漏掉关键邮件或会议。

实际问题通常落在信息选择上：长邮件线程占用过多 Context，旧日历事件干扰当前请求，搜索结果虽然语义相关却已经过期，或者同一内容从不同来源重复出现。改进需要同时观察回答质量、延迟和成本，而不是只提高某一个检索分数。

## 项目能够展开的 Agent 知识

这个项目适合深入说明：

- Context 与完整业务数据的区别；
- Query 理解、搜索、排序和去重；
- 权限、来源、版本与时效；
- 长邮件线程的压缩和 Token 预算；
- Lost in the Middle 与 Context 噪声；
- Context 质量、延迟和成本的共同评测。

如果实际链路使用关键词、语义检索或 Rerank，可以在详细文章中如实展开；不为了覆盖 RAG 术语虚构文档切片或向量数据库。

## 后续可以展开的文档

- 业务与数据源：Outlook Copilot 怎样使用邮件、日历和目录信息；
- 检索与 Context：Query、召回、排序、去重和 Context 组装；
- 权限与合规：对象级授权、租户、敏感数据和审计；
- 性能与缓存：延迟拆分、并行查询、Token 和缓存策略；
- 评测与故障：遗漏关键邮件、旧信息和 Context 回归；
- 部署与生产运行：服务实例、机器配置、扩缩容、依赖、监控和值班。
