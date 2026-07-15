---
layout: default
title: Token、Attention 与生成
parent: 大模型基础
nav_order: 1
permalink: /docs/llm/model-inference/
---

# Token、Attention 与生成

用户输入一句话时，模型并不是直接读取人类理解的“词义”。文本先变成 Token 和向量，Transformer 根据上下文计算信息之间的关系，再预测下一个 Token。输出 Token 被追加到 Context，模型继续预测，直到结束。

这条链解释了几个应用工程现象：为什么请求按 Token 计量，为什么长 Context 增加成本，为什么 Embedding 能用于检索，以及为什么模型生成的是概率结果而不是数据库查询。

## Token 是模型处理文本的基本单位

Tokenizer 把文本切成模型词表中的 Token。一个 Token 可能是汉字、词的一部分、标点或代码片段，不等于自然语言中的一个词。

常见词可以用较少 Token 表示，罕见名称和长数字可能被拆得更细。Context Window 和 API 费用通常按 Token 计算，因此字符数相同的两段内容，实际占用可能不同。

BPE 是常见的子词切分方法。它从较小单位开始，反复合并语料中高频出现的相邻组合，在词表规模和表示长度之间取舍。具体模型使用的 Tokenizer 和词表并不完全相同，应用需要使用目标模型提供的计数能力估算输入。

## Embedding 把离散 Token 映射成向量

Token ID 本身只是编号。模型通过 Embedding 表把它映射成高维向量，使后续计算能够表达语义和关系。

面向检索的 Embedding 模型也把句子或文档映射成向量。语义相近的文本通常距离更近，系统可以用余弦相似度等方法寻找候选内容。生成模型内部的 Token Embedding 与检索 API 产生的文档向量用途不同，不能直接混为同一套索引。

## Attention 按当前问题聚合信息

Transformer 会把每个位置映射成 Query、Key 和 Value。Query 表示当前位置要寻找什么，Key 表示其他位置可以怎样被匹配，Value 承载要聚合的内容。Query 与 Key 的相关程度形成权重，模型再按权重汇总 Value。

多头注意力让模型同时学习不同关系，例如语法联系、指代和局部模式。位置编码补充词序，否则相同 Token 的不同排列难以区分。现代大模型大多采用 Decoder-only 架构，并使用因果遮罩保证当前位置只能依据此前内容预测下一个 Token。

[Attention]({{ site.baseurl }}/docs/interview/ai-agent/#attention-mechanism) 能利用长距离信息，却不保证 Context 中每条内容都被同等可靠地使用。重要规则被大量无关内容淹没，或者关键证据处于不易利用的位置，仍可能降低回答质量。

## 自回归生成会逐步扩大历史

第一次生成依据用户输入，第二次还要带上第一个输出 Token，后续步骤不断重复。为了避免每次重新计算所有历史 Token 的 Key 和 Value，推理服务通常使用 KV Cache 保存已有计算。

KV Cache 能降低连续生成的重复计算，却会占用显存；输入越长、并发越高，推理资源越紧张。这属于模型推理后端的优化问题。应用后端更关心 Context 放了什么、请求何时超时，以及输出怎样交付和保存。

## 模型通过训练学会预测与遵循指令

预训练让模型在大规模文本上学习预测下一个 Token，从中形成语言、知识和模式能力。监督微调使用指令和参考回答，让模型更习惯按任务要求作答。偏好对齐再利用人类或规则反馈，使输出更符合有用、安全和风格要求。

SFT、[PPO 和 DPO]({{ site.baseurl }}/docs/interview/ai-agent/#ppo-vs-dpo)、GRPO 是不同训练与对齐方法。SFT 模仿参考回答；PPO 常与奖励模型配合，通过强化学习更新策略；DPO 直接利用优选和劣选回答对调整相对概率；GRPO 比较同一问题的一组回答，适合具有可验证奖励的任务。

[这些训练与对齐方法]({{ site.baseurl }}/docs/interview/ai-agent/#llm-training-alignment)改变模型总体行为，不能替代应用层的业务校验。一个经过充分对齐的模型仍可能使用过期知识、误解当前状态或生成不存在的事实。

## 应用工程需要记住什么

一次回答可以压缩为：

```text
文本被切成 Token
→ Token 变成向量
→ Attention 聚合 Context 中的信息
→ 模型预测下一个 Token
→ 输出追加后继续生成
```

Token 决定容量和计费单位，Embedding 支持语义表示，Attention 决定模型怎样使用上下文，自回归生成带来流式输出，训练决定模型通常会怎样响应。应用系统仍要提供可信信息、验证关键结果，并把模型放在明确的软件边界中。
