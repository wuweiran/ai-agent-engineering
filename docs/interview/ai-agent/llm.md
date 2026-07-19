---
layout: default
title: 大模型与应用基础面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 1
permalink: /docs/interview/ai-agent/llm/
---

# 大模型与应用基础面试题

这些问题来自 AI Agent 岗位面试，讨论模型结构、面向工具使用的模型训练，以及普通大模型应用与 Agent 都会遇到的 Context 管理。它们是 Agent 工程师需要理解的上游基础，但不等于 Agent Runtime 本身，也不要求读者具备训练基础模型的完整经验。

## 模型结构

## 多模态大模型怎样连接视觉编码器与语言模型？
{: #multimodal-vision-language-connection }

**主链路是视觉编码、模态对齐、语言生成：**

1. **视觉编码**：把图片切成 Patch，由 ViT 等视觉编码器生成视觉特征；
2. **模态对齐**：通过线性投影、MLP 或 Resampler，把特征转换成语言模型能够接收的视觉 Token；
3. **语言生成**：视觉 Token 与文本 Token 一起进入语言模型，通过注意力融合信息并自回归生成答案。

具体模型可能冻结视觉编码器、联合训练全部模块，或通过 Cross-Attention 接入视觉特征，但这条主链基本不变。

相关内容：[Token、Attention 与生成]({{ site.baseurl }}/docs/llm/model-inference/)。

## 面向 Agent 的模型训练中，CPT、SFT 和 RL 分别做什么？
{: #agentic-model-training }

三者解决的是**适应数据、模仿行为、优化结果**：

- **CPT（Continued Pre-Training，继续预训练）**：用领域文本、代码或工具语料继续训练下一个 Token 预测，使模型适应目标数据分布；
- **SFT（Supervised Fine-Tuning，监督微调）**：用高质量任务轨迹示范规划、Tool Call、Observation 利用和任务完成；
- **RL（Reinforcement Learning，强化学习）**：根据环境反馈、验证器或奖励，优化任务成功、工具选择和失败恢复。

“Agentic CPT”不是严格统一的训练范式，三阶段也不是所有模型都必须采用的固定流水线。SFT 时，System、User 和 Observation 仍作为输入，但通常从训练标签中 Mask 掉；意思是**不计算这些 Token 的预测损失**，不是把它们从 Context 删除。

相关内容：[Token、Attention 与生成]({{ site.baseurl }}/docs/llm/model-inference/)。

## Prompt 与结构化交互

## System Prompt 是什么？能否代替权限规则？
{: #system-prompt }

**System Prompt 定义模型在当前应用中的身份、稳定行为边界和输出要求，是 Context 的一部分。**

它只能影响模型行为，不能强制执行权限和业务规则。越权访问、金额限制、参数校验和审批必须由 Runtime 与业务服务实施。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## Prompt Engineering 和 Context Engineering 有什么区别？
{: #prompt-vs-context-engineering }

- **Prompt Engineering**：优化指令、Few-shot 示例和输出要求，让模型更稳定地使用已有能力；
- **Context Engineering**：管理模型本轮看到的信息，包括系统规则、工具、检索材料、任务状态、记忆、排序和压缩。

**判断方法：**模型没理解任务，优先检查 Prompt；缺少事实、资料过期或噪声过多，优先检查 Context。两者共同影响结果，但都不能代替权限和业务校验。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## Prompt、RAG 和微调应该怎样选择？
{: #prompt-rag-fine-tuning-selection }

按问题根因选择：

- **Prompt**：模型没有正确理解任务、边界或输出要求；
- **RAG**：缺少外部知识，或者知识需要及时更新、保留来源和权限；
- **微调**：希望模型在大量同类任务中稳定形成特定行为，而且 Prompt、Context、工具和确定性规则仍不能可靠解决。

三者可以组合使用。RAG 不会自动修复指令不清，微调也不适合注入实时事实；权限和业务规则始终由程序执行。通常先尝试成本更低、更新更容易的 Prompt 和 RAG，再根据代表性评测决定是否微调。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[Token、Attention 与生成]({{ site.baseurl }}/docs/llm/model-inference/)。

## 怎样设计和优化 Prompt？
{: #prompt-optimization }

可以按三步回答：

1. **说清任务**：给出背景、目标、可用信息、限制和输出要求；
2. **补充示例**：边界难以描述时加入少量代表性 Few-shot，不靠无限追加规则；
3. **用评测迭代**：针对真实失败修改，在固定任务上比较质量、严重错误、Token、延迟和工具行为，必要时做消融实验。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## DSPy 适合解决什么问题？
{: #dspy-prompt-optimization }

**DSPy 适合解决 Prompt 对措辞敏感、人工迭代缺乏稳定评测的问题。**它把指令和 Few-shot 示例视为可优化参数，在训练样本和评分函数上搜索组合。

边界有三点：质量取决于样本与指标；必须保留独立测试集；还要审查成本、延迟和严重失败。它不会脱离数据自动产生“最佳 Prompt”。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## CoT 和 Self-Consistency 分别解决什么问题？
{: #cot-self-consistency }

- **CoT**：通过分步推理或中间结构，降低复杂问题跳步的概率；
- **Self-Consistency**：采样多条独立推理路径，再对最终答案投票或做一致性选择，减少单一路径的偶然错误。

代价是更多 Token、延迟和费用，而且 Self-Consistency 只适合答案能够比较的任务。关键业务结论仍要由外部证据验证，应用也不应依赖模型不可见的内部推理。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## JSON Mode、结构化输出和工具调用有什么区别？
{: #structured-output-vs-tool-calling }

- **JSON Mode**：通常只保证输出是合法 JSON；
- **结构化输出**：进一步按 Schema 限制字段、类型、枚举和必填项；
- **工具调用**：用结构化参数表达模型希望应用执行的动作。

共同边界是：**只保证结果形状，不保证语义、权限和业务状态正确。**

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## Context 管理

## Context 超出限制时怎样处理？滑动窗口和摘要有什么区别？
{: #context-overflow-window-summary }

先把完整会话、任务状态和大文件保存在 Context 之外，再组合几种手段：

- 清理过期工具结果，按需重新读取原文；
- 结构化保存目标、约束、事实和待办；
- 保留近期原文，对较早历史做摘要或压缩。

两种方法的区别是：

- **滑动窗口**保留最近消息，原文准确、实现简单，但可能丢失早期目标和证据；
- **摘要**保留更远的事实、决定和待办，但属于有损转换，可能遗漏或引入错误。

生产系统通常组合**近期原文、结构化任务状态、历史摘要和外部可重读资料**，而不是只选一种。

相关内容：[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。
