---
layout: default
title: 接口与 Context 请求
parent: Outlook Copilot Insight Service
grand_parent: 工作经历
nav_order: 1
permalink: /docs/career/copilot-insight-service/context-api/
---

# 接口与 Context 请求

## 接口边界

Insight Service 提供需要检索、领域解析和 Context 处理的能力，服务内部不调用大模型。模型根据用户问题生成结构化 Tool Call，**Copilot Runtime** 负责执行调用，并附带用户访问令牌和 Outlook 客户端提供的界面 Context。

```text
用户问题与 Outlook 界面 Context
→ 模型生成 Insight Tool Call
→ Copilot Runtime 附带访问令牌和客户端 Context
→ Insight Service 校验查询范围和权限
→ Exchange、日历和组织目录
→ 受权限控制的 Context 返回模型
```

已知 Message ID 或 Event ID 的精确对象读取不经过 Insight Service，Runtime 直接使用 Outlook 读取工具调用 Microsoft Graph API。人员解析也不作为 Insight Service 接口暴露；模型使用已有的人员与目录 Context 取得稳定 Person ID，再将它作为 Query 或 Calendar 的过滤参数。

## Runtime 中的工具定义

Runtime 注册三个高层工具，与 Insight Service 的三类 Context 能力对应。

### 搜索 Outlook Context

```json
{
  "name": "search_outlook_context",
  "description": "Search the user's Outlook mail for information needed to answer the current request.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string" },
      "startTime": { "type": "string", "format": "date-time" },
      "endTime": { "type": "string", "format": "date-time" },
      "participantIds": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "required": ["query"]
  }
}
```

Runtime Handler 将它映射到 `POST /v1/insights/query`。

### 读取 Conversation Context

```json
{
  "name": "get_outlook_conversation",
  "description": "Read and prepare an Outlook email conversation for answering the current request.",
  "parameters": {
    "type": "object",
    "properties": {
      "conversationId": { "type": "string" },
      "query": { "type": "string" }
    },
    "required": ["conversationId"]
  }
}
```

Runtime Handler 将它映射到 `POST /v1/insights/conversations`。`query` 用来帮助长线程选择与当前问题最相关的消息，不用于重新搜索邮箱。

### 搜索 Calendar Context

```json
{
  "name": "search_outlook_calendar",
  "description": "Search the user's Outlook calendar and return events relevant to the current request.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string" },
      "startTime": { "type": "string", "format": "date-time" },
      "endTime": { "type": "string", "format": "date-time" },
      "participantIds": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "required": ["query"]
  }
}
```

Runtime Handler 将它映射到 `POST /v1/insights/calendar`。

```text
模型 Tool Call
→ Runtime 校验工具名与参数 Schema
→ Handler 附加访问令牌、客户端 Context 和 Trace
→ 调用 Insight Service HTTP API
→ 将响应转换成 Tool Result
→ 模型继续生成答案
```

身份、权限、候选上限和 Token 预算不出现在模型可填的 Schema 中。Runtime 也不把 HTTP 状态码原样交给模型，而是转换成参数有歧义、权限不足、数据源暂时不可用或未找到结果等错误语义。

## 主要接口

### 邮件查询

`POST /v1/insights/query` 根据 Query、绝对时间范围和稳定 Person ID 搜索邮件。完整链路是：

```text
用户问题
→ 模型生成 search_outlook_context Tool Call
→ Runtime 校验参数并附加用户 Token、界面 Context 和 Trace
→ Insight Service 验证身份与请求边界
→ 校验绝对时间和 Person ID
→ 生成邮件搜索条件
→ Outlook 搜索召回候选
→ 对象级权限过滤与结果规范化
→ 去重、打分和 Top-K 排序
→ 返回 Message ID、Conversation ID、Snippet 和 Citation
→ Runtime 转换为 Tool Result
→ 模型决定回答、读取单封邮件或展开 Conversation
```

请求进入 Insight Service 后依次完成以下处理：

1. **认证与参数校验**：从访问令牌取得 Tenant ID、User ID 和权限范围，校验 Query、时间范围和客户端 Anchor；模型不能在请求体中覆盖身份和服务端上限；
2. **时间与人员校验**：校验 `startTime`、`endTime` 的格式、顺序和最大跨度，并检查 `participantIds` 是否是当前租户中有效的稳定 Person ID；
3. **搜索条件生成**：组合 Query 关键词、绝对时间、邮件参与者、Folder 和当前邮件 Anchor，限制搜索范围；
4. **候选召回**：使用用户身份调用 Outlook 邮件搜索，一次取得最多 **50 条**候选及搜索分数、Message ID、Conversation ID 和 Snippet；
5. **权限与规范化**：再次校验候选对象是否可读，统一主题、发件人、收件时间、对象 ID 和 Citation；未授权对象不会进入结果，也不会暴露命中数量；
6. **去重与排序**：删除重复 Message，同一 Conversation 出现多个候选时保留最相关的邮件；综合搜索分数、时间衰减、人员匹配和 Anchor 关系重新打分；
7. **结果截取**：选取前 **12 条**候选，返回轻量结果和 `partial`、`warnings`，不在 Query 响应中加载完整邮件正文。

Query 返回的是用于定位的候选。Snippet 已足以回答简单事实时，模型可以直接生成答案；需要完整单封邮件时，模型调用 Outlook 读取工具直接访问 Graph API；需要理解上下文和线程演变时，再调用 `get_outlook_conversation`。这条检索链路全部由确定性程序完成，Insight Service 内部不调用大模型。分页、Top-K 和候选存储见 [Outlook 邮件 RAG]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/)。

### Conversation Context

`POST /v1/insights/conversations` 根据 Conversation ID 读取邮件线程并按时间排序。服务解析邮件 HTML/MIME 结构、引用分隔符和头部标记，删除回复中重复携带的历史正文，并按规则清理签名与免责声明。

短 Conversation 返回全部处理后的邮件；长 Conversation 根据 Query 命中、参与人和时间选择消息，再按 Token 上限截断过长正文。发送者、时间和 Citation 始终保留。它返回的是程序整理后的线程证据，不是模型生成的邮件摘要，也不是 Graph API 的原始响应。

### Calendar Context

`POST /v1/insights/calendar` 根据 Query、绝对时间范围和稳定 Person ID 查询事件。服务校验时间跨度和人员 ID，再返回与问题相关的事件。

Calendar 结果保留事件主题、起止时间、时区、参与人、地点、会议链接、Event ID 和 Citation，并删除模型不需要的内部字段。已知 Event ID 的详情读取直接走 Outlook/Graph 工具。

## Tool Call 与内部查询

模型直接生成绝对时间和稳定 Person ID：

```json
{
  "query": "总结 Alice 上周发给我的预算更新",
  "startTime": "2024-05-06T00:00:00Z",
  "endTime": "2024-05-13T00:00:00Z",
  "participantIds": ["person-4821"]
}
```

Copilot Runtime 附加用户访问令牌、当前 Outlook 界面 Context、Request ID 和 Trace Context。Insight Service 校验时间与人员 ID，再补充服务端执行参数：

```json
{
  "maxCandidates": 50,
  "maxItems": 12,
  "maxContextTokens": 6000
}
```

模型不能填写 Tenant ID、User ID、候选数量或服务端 Token 上限。Person ID 必须来自当前客户端 Context 或此前可信的人员查询结果，不能由模型凭空编造。Insight Service 从访问令牌的 Claims 中取得用户与租户身份，并在查询和结果读取时执行对象级授权。

## 返回结构

三个接口共享响应外壳：

```json
{
  "requestId": "req-8f21",
  "data": {},
  "partial": false,
  "warnings": []
}
```

`data` 保留各自的领域结构：

| 接口 | 返回内容 |
| --- | --- |
| `/insights/query` | 邮件候选，包括 Message ID、Conversation ID、标题、Snippet、发送者、时间、分数和 Citation |
| `/insights/conversations` | Conversation 与按时间排序、去重和裁剪后的 Message 列表 |
| `/insights/calendar` | Event 列表，包括 Event ID、主题、起止时间、时区、参与人、地点、会议链接和 Citation |

Query 的单个候选是：

```json
{
  "messageId": "msg-1742",
  "conversationId": "conv-308",
  "subject": "FY25 budget update",
  "sender": "alice@contoso.com",
  "sentAt": "2024-05-09T08:31:00Z",
  "snippet": "The approved budget has been updated to...",
  "score": 0.87,
  "citation": {
    "type": "outlook-message",
    "id": "msg-1742"
  }
}
```

Conversation 返回整理后的消息列表，模型根据这些证据回答和生成 Citation。`partial` 表示下游超时或预算不足，`warnings` 说明缺失范围；每个领域结果保留裁剪标记，使模型知道内容是否完整。

## 权限与查询边界

用户身份来自 Copilot Runtime 转发的 Microsoft Entra ID 用户访问令牌，不来自模型参数。Insight Service 验证 Token Claims，并通过 On-Behalf-Of 流程取得 Outlook 下游令牌，使搜索继续以用户身份执行对象级授权。候选返回后，服务再校验租户、邮箱和对象边界。完整权限链路见 [Outlook 邮件 RAG]({{ site.baseurl }}/docs/career/copilot-insight-service/mail-rag/)。

请求还受到时间范围、候选数量和 Token 预算限制。模型不能通过省略时间条件或扩大返回数量绕过服务端上限；Person ID 只用于缩小查询范围，不会扩大用户对邮箱或日历对象的访问权限。
