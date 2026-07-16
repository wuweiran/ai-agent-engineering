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

常见架构先把图片切成 Patch，由 ViT 等视觉编码器生成视觉特征，再通过线性投影、MLP 或 Resampler 把特征维度和数量转换成语言模型能够接收的视觉 Token。视觉 Token 与文本 Token 一起进入语言模型，模型通过注意力结合两种信息并自回归生成答案。

不同模型并非都采用同一结构。有的冻结视觉编码器，只训练连接层和语言模型；有的联合训练；有的不用前置视觉 Token，而在语言模型若干层加入 Cross-Attention。回答时应先说清“视觉编码—模态对齐—语言生成”这条主链，再说明具体实现存在差异。

相关内容：[Token、Attention 与生成]({{ site.baseurl }}/docs/llm/model-inference/)。

## 面向 Agent 的模型训练中，CPT、SFT 和 RL 分别做什么？
{: #agentic-model-training }

“Agentic CPT”不是严格统一的训练范式。CPT 通常指继续预训练，用领域文本、代码或工具相关语料继续训练下一个 Token 预测，使模型适应目标数据分布。SFT 使用高质量任务轨迹，示范怎样规划、生成 Tool Call、利用 Observation 并完成任务。RL 再根据环境反馈、验证器或奖励优化端到端任务成功、工具选择和失败恢复。

三阶段不是所有模型都必须采用的固定流水线。数据目标、环境可执行性和奖励是否可靠，会决定是否需要每一阶段。

SFT 中通常只对模型应该生成的 Assistant 内容计算损失。System、User 和工具返回的 Observation 仍作为输入让模型读取，但常从训练标签中 Mask 掉，因为它们由用户或环境提供，不是模型下一步应该生成的内容。Mask 是“不计算这些 Token 的预测损失”，不是把 Observation 从 Context 删除。

相关内容：[Token、Attention 与生成]({{ site.baseurl }}/docs/llm/model-inference/)。

## Prompt 与结构化交互

## System Prompt 是什么？能否代替权限规则？
{: #system-prompt }

System Prompt 通常承载模型在当前应用中的身份、稳定行为边界和输出要求，是每轮 Context 的一部分。

它能影响模型行为，却不能强制执行权限和业务规则。越权访问、金额限制和审批必须由 Runtime 与业务服务检查。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## Prompt Engineering 和 Context Engineering 有什么区别？
{: #prompt-vs-context-engineering }

Prompt Engineering 优化指令、Few-shot 示例和输出要求，帮助模型更稳定地使用已有能力。Context Engineering 管理模型本轮看到的信息流，包括系统规则、工具、检索材料、任务状态、记忆、排序和压缩。

模型没理解任务时优先检查 Prompt；缺事实、资料过期或噪声过多时优先检查 Context。两者共同决定结果，但都不能代替权限和业务校验。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[Agent Context]({{ site.baseurl }}/docs/ai-agent/agent-design/context/)。

## 怎样设计和优化 Prompt？
{: #prompt-optimization }

先明确背景、目标、可用信息、限制和输出要求；边界难以理解时加入少量代表性示例。Prompt 不应靠不断追加规则增长，而要针对真实失败修改。

优化时使用固定任务比较版本，观察质量、严重错误、Token、延迟和工具行为。修改多项机制时可做消融实验，确认改善究竟来自哪里。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## DSPy 适合解决什么问题？
{: #dspy-prompt-optimization }

DSPy 把大模型程序中的指令和 Few-shot 示例作为可优化参数，在训练样本和评分函数上搜索更好的组合。它适合 Prompt 对措辞敏感、人工迭代缺乏稳定评测的场景。

优化质量取决于样本和指标。必须保留独立测试集，并审查成本、延迟和严重失败；DSPy 不是脱离数据自动生成最佳 Prompt。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## CoT 和 Self-Consistency 分别解决什么问题？
{: #cot-self-consistency }

CoT 通过分步推理或中间结构帮助模型处理复杂问题。Self-Consistency 采样多条独立推理路径，再对最终答案进行投票或一致性选择，以减少单一路径的偶然错误。

它们会增加 Token、延迟和费用，也不适合无法比较答案的开放任务。应用不应依赖或保存模型不可见的内部推理；关键结论仍需外部证据验证。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)。

## JSON Mode、结构化输出和工具调用有什么区别？
{: #structured-output-vs-tool-calling }

JSON Mode 通常只保证输出是合法 JSON。结构化输出进一步按 Schema 限制字段、类型和枚举。工具调用使用结构化参数表达模型希望应用执行的动作。

三者都只解决结果形状，不保证语义、权限和业务状态正确。

相关内容：[Prompt 与结构化输出]({{ site.baseurl }}/docs/llm/prompt-structured-output/)、[Agent 工具]({{ site.baseurl }}/docs/ai-agent/agent-design/tools/)。

## Context 管理

## Context 超出限制时怎样处理？滑动窗口和摘要有什么区别？
{: #context-overflow-window-summary }

先把完整会话、任务状态和大文件保存在 Context 之外，再为每次调用选择工作集。接近限制时，可以清理过期工具结果、按需重新读取原文、保留关键目标与状态，并对较早历史做摘要或压缩。

滑动窗口只保留最近若干消息，实现简单且保留原文，但可能丢失最初目标、约束和早期证据。摘要把旧历史转换成较短的事实、决定和待办，能保留更远语义，却是有损转换，可能漏掉细节或引入错误。生产系统通常组合近期原文、结构化任务状态、历史摘要和外部可重读资料，而不是只使用其中一种。

相关内容：[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。
