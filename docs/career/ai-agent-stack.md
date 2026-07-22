---
layout: default
title: AI Agent 技术栈与项目映射
parent: 工作经历
nav_order: 6
permalink: /docs/career/ai-agent-stack/
---

# AI Agent 技术栈与项目映射

这张图用于把通用 AI Agent 技术栈映射到实际工作经历。它不表示每个项目都采用同一种架构，也不把依赖的平台能力算成个人实现。

## 通用技术分层

```text
产品与交互
→ Model 与 Prompt
→ Context、RAG 与 Memory
→ Agent Runtime 与 Planning
→ Tool Calling 与业务集成
→ 安全与确定性执行
→ Evaluation、Trace 与发布
→ 应用后端和模型推理基础设施
```

| 技术层 | 解决的问题 | 常见实现方式 |
| --- | --- | --- |
| 产品与交互 | Agent 从哪里获得用户目标和界面状态 | Web/移动端、Side Panel、Chat、Computer Use、API |
| Model 与 Prompt | 模型怎样理解任务并生成文本或结构化请求 | 模型 API、System Prompt、Few-shot、结构化输出 |
| Context、RAG 与 Memory | 本轮需要哪些事实，历史和知识怎样取得 | 搜索型 RAG、向量 RAG、Chunk、Embedding、Rerank、任务状态、长期记忆 |
| Runtime 与 Planning | 怎样运行模型—工具循环并控制状态和终止 | 厂商托管 Agent、Agent SDK、LangChain/LangGraph、Dify、自建 Runtime |
| Tool Calling | 怎样连接外部数据和动作 | Function Calling、工具 Schema、本地工具、MCP、企业 Extension |
| 安全与执行 | 怎样约束权限和真实副作用 | 身份与 Scope、对象级授权、确认、幂等、资源版本、业务状态机 |
| Evaluation 与生产 | 怎样判断质量、定位失败并安全发布 | Golden Set、Assertion、LLM Judge、Trace、质量门禁、灰度和回滚 |
| 后端与推理 | 应用和模型服务怎样部署运行 | Java/Spring、Scala/ZIO、消息与存储、云平台；vLLM 或厂商模型服务 |

这些选项不是一套必须同时采用的清单。例如 LangChain 是代码框架，Dify 是应用平台，MCP 是工具接入协议，vLLM 是模型推理引擎，它们处在不同层次。

## 我的项目覆盖

| 技术层 | 覆盖项目 | 使用的内部或实际技术 | 我的工作 |
| --- | --- | --- | --- |
| 固定 Workflow 与模型调用 | Microsoft Purview 工作流 | Azure Logic Apps、Workflow Definition、Scala + ZIO、Azure OpenAI Chat Completions、GPT-3.5 Turbo | Expression 领域 Owner；定义 EBNF、AST、Validator 和 Evaluator；接入 Workflow 概括与 Definition 生成 |
| Prompt 与结构化输出 | Purview；Outlook Copilot Agent | System Prompt、Few-shot、Action Catalog、JSON Validator；Agent Instructions、Capability Prompt、工具 Description | 组装模型输入、限制可用能力、验证模型输出；按 Outlook 场景拆分 Prompt 并处理工具结果 |
| RAG 与 Context | Outlook Copilot Insight Service；Outlook Copilot Agent | Outlook Search、Microsoft Graph、OBO、搜索型 RAG、分页召回、Top 12、Citation、Side Panel Context | 负责 `/insights/query` 的邮件召回、权限过滤、去重和排序；设计当前邮件、选区、附件与 Folder Context |
| Agent Runtime | Outlook Copilot Agent | Microsoft 365 Copilot Declarative Agent 平台、Agent Definition、Conversation、Planning、Confirmation、Trace | 在平台已有 Runtime 上配置能力和行为；没有自建模型循环、Planning 或确认服务 |
| Tool Calling | Insight Service；Outlook Copilot Agent | Copilot Extension、Tool Schema、Tool Call / Tool Result、附件提取、邮件搜索与写入工具 | 参与搜索 Extension 契约；设计场景工具集合、Description、参数语义和错误后的 Agent 行为 |
| 多轮状态与任务执行 | Outlook Copilot Agent | 平台 Conversation 与 Plan、结构化任务约束、资源版本、Operation ID | 处理用户修改范围、重新计划、部分成功、结果未知和任务终止；没有建设跨任务长期记忆 |
| 安全与权限 | Insight Service；Outlook Copilot Agent | Microsoft Entra ID、OBO、`Mail.Read` / `Mail.ReadWrite`、对象级权限、平台确认 | 实现或接入权限过滤；限制工具可见性，确保写操作走确认和 Extension 校验 |
| Agent Evaluation | Copilot Evaluation 与 Golden Set | SEVAL、Query Set、CIQ、Grounding Data、Assertion、LM Checklist、业务 Metrics | 持续维护评测资产、真实脱敏 Utterance、Baseline/SDF 回归、质量门禁和 Trace 定位 |
| 竞品评测 | Copilot Bake-off | Playwright scraping、Google Workspace ingestion、User/Email/Golden Set Mapping、SEVAL LM Checklist | 搭建 Gmail Gemini 自动执行和跨系统映射，生成 Query 级产品差距并回流常规回归 |
| 生产后端 | Purview；Insight Service；Bake-off | Scala + ZIO、Java 17 + Spring Boot 3 + WebFlux、Substrate、Playwright Worker | 后端服务开发、性能与容量控制、监控、灰度、故障排查和跨团队依赖协作 |

## 项目在 Agent 链路中的位置

```text
用户与产品入口
└─ Outlook Side Panel
   └─ Outlook Copilot Agent 能力开发
      ├─ Instructions、Capability Prompt 与 Side Panel Context
      ├─ Microsoft 365 Copilot 平台负责 Runtime、Planning 和确认
      └─ Extension 工具
         ├─ Insight Service：邮件搜索型 RAG
         ├─ 附件提取
         └─ Outlook 邮件读写

质量链路
└─ Copilot Evaluation：Golden Set、Metric、回归与发布门禁
   └─ Copilot Bake-off：与 Gmail Gemini 做共同能力对比

更早的技术基础
└─ Purview Workflow
   ├─ 固定 Workflow、Expression 和确定性执行
   └─ 模型概括与 Workflow Definition 生成
```

## 通用实现方式与我的对应经验

| 通用实现方式 | 典型定位 | 我的对应经验 |
| --- | --- | --- |
| 直接调用模型 API | 单次生成、抽取、总结和结构化输出 | Purview 通过 Azure OpenAI Chat Completions 实现概括和 Workflow Definition 生成 |
| 固定 Workflow + LLM | 程序决定步骤，模型处理局部非确定任务 | Purview 使用 Logic Apps 调度，模型只负责概括或生成 Definition 草稿 |
| LangChain | 用代码组合 Model、Prompt、Retriever、Tool 和 Agent | 没有在这些项目中使用；同类职责由 Microsoft 365 Copilot 平台、Extension 和业务代码承担 |
| LangGraph | 显式状态图、条件分支、检查点和恢复 | 没有在这些项目中使用；Purview 的确定流程由 Logic Apps 承担，Outlook Agent 的 Planning 和 Conversation 由 Copilot 平台承担 |
| Dify | 可视化构建 Workflow、RAG 和 Agent 应用 | 没有使用；微软项目采用内部平台和产品集成 |
| 自建 Agent Runtime | 自己实现模型循环、工具调度、状态和终止 | 没有自建；Outlook 使用 Microsoft 365 Copilot Declarative Agent 平台 |
| MCP | 标准化连接外部工具和资源 | Outlook 项目使用 Copilot Extension，不使用 MCP；两者解决的都是能力接入，但协议和平台不同 |
| 向量 RAG | Chunk、Embedding、向量索引和 Rerank | 知识上覆盖，但 Insight Service 实际采用 Outlook Search 的搜索型 RAG，不使用 Chunk、Embedding 或向量数据库 |
| vLLM / Ollama | 生产 GPU 推理或本地模型运行 | 没有负责模型推理部署；Purview 使用 Azure OpenAI，Outlook 使用 Copilot 平台模型服务 |
| Multi-Agent / A2A | 多个独立 Agent 分工和通信 | 没有使用；Outlook 三项能力属于同一个通用 Agent |

## 面试问题怎样映射

| 面试问题 | 优先使用的项目 |
| --- | --- |
| LLM、Workflow 和 Agent 有什么区别 | Purview Workflow + Outlook Copilot Agent |
| 怎样做 Prompt 和结构化输出 | Purview 模型集成；Outlook Prompt 与 Context |
| RAG 是怎样实现和优化的 | Insight Service 搜索型 RAG |
| Agent 工具调用链路怎样设计 | Outlook Copilot Agent + Insight Extension |
| 多轮状态、Planning 和错误恢复怎样做 | Outlook 邮箱整理 |
| 怎样处理权限、确认和 Prompt Injection | Insight Service 权限链 + Outlook Agent 安全边界 |
| 如何评估 Agent、判断回归并定位问题 | Copilot Evaluation 与 Golden Set |
| 如何用真实数据形成产品改进闭环 | Evaluation + Bake-off |
| 后端性能、异步、容量和生产事故 | Purview + Insight Service + Bake-off Worker |

## 需要守住的经历边界

- 没有自建 Microsoft 365 Copilot Runtime、模型循环、Planning、确认或流式输出；
- 没有开发既有附件提取和邮件写入 Extension 的 Handler；
- 没有在项目中使用 LangChain、LangGraph、Dify、MCP、vLLM、Ollama 或 Multi-Agent；
- Insight Service 是搜索型 RAG，不是向量 RAG；
- Copilot Evaluation 的 SEVAL Job 平台由统一中台提供，我负责 Outlook 业务评测资产、回归和失败分析；
- Bake-off 的执行与映射系统由 Outlook Team 建设，SEVAL 只负责最后运行 LM Checklist。

面试时可以使用通用技术解释架构选择，但应明确区分“理解过”“可类比”和“项目实际使用过”。
