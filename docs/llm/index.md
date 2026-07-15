---
layout: default
title: 大模型基础
nav_order: 4
has_children: true
permalink: /docs/llm/
---

# 大模型基础

大模型把文本切成 Token，在向量空间中表示信息，再根据当前 Context 逐步生成输出。Agent、RAG 和大模型应用后端都建立在这些能力及其限制之上。

这一部分沿应用工程真正会遇到的问题展开：模型怎样处理输入并生成输出，应用怎样通过提示词和结构化协议约束结果，RAG 怎样取得外部知识，以及模型为什么会产生没有依据的内容。

## 模型怎样形成一次输出

[大模型怎样处理文本并生成回答](model-inference/)从 Token、Embedding、Attention 和训练过程建立最小认识。理解这些机制不是为了推导完整数学公式，而是为了判断 Context Window、Token 成本、语义检索和模型能力的边界。

## 应用怎样控制模型输入和输出

[提示词与结构化输出怎样约束模型](prompt-structured-output/)说明 System Prompt、消息角色、Few-shot、结构化输出和回归评测分别解决什么问题。提示词影响模型行为，却不能代替权限、业务规则和事实校验。

## Context 怎样控制成本和质量

[长 Context 怎样影响质量、延迟和成本](context-cost-cache/)说明 Context 预算、信息压缩、Lost in the Middle，以及结果缓存、语义缓存和 Prompt Cache 的边界。

## 外部知识怎样进入模型

[RAG 怎样让模型使用外部知识](rag/)说明文档切块、Embedding、检索、混合搜索、重排和生成怎样组成完整链路。RAG 不要求模型自主调用工具，也不等于 Agent；它是给当前回答选择参考材料的方法。

## 为什么回答看起来合理却可能是错的

[大模型为什么会产生幻觉](hallucination/)区分事实错误、推理错误和错误引用，说明证据、检索、结构约束、评测与外部验证怎样降低风险。

大模型基础解释模型和信息链本身。需要模型根据工具反馈持续改变路径时，进入[AI Agent 工程](../ai-agent/)；需要把生成过程接入 API、状态和异步任务时，进入[大模型应用怎样改变后端](../backend/llm-backend/)。
