---
layout: default
title: RAG 与知识检索面试题
parent: AI Agent 常见面试题
grand_parent: 面试题库
nav_order: 4
permalink: /docs/interview/ai-agent/rag/
---

# RAG 与知识检索面试题

这些问题覆盖 RAG 的离线索引、在线检索、Query 改写、混合召回、Rerank、Context 选择、评测和性能。RAG 是 AI Agent 岗位常见的应用基础，但并不要求模型自主调用工具，也不等于 Agent。

## 一个标准 RAG 系统包含哪些步骤？
{: #rag-pipeline }

离线阶段包括文档解析、Chunk、Embedding 和索引；在线阶段包括 Query 改写、问题向量化、召回、过滤或混合检索、Rerank、组装 Context 和生成回答。

评测要分开检查检索是否找到正确证据，以及回答是否受到证据支持。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Naive RAG、Advanced RAG 和 GraphRAG 有什么区别？
{: #naive-advanced-graph-rag }

Naive RAG 是最小文本检索链路：离线建立 Chunk 向量索引，在线对 Query 做向量 Top-K 检索，再把候选交给模型生成。文档切片不发生在用户提问之后。

Advanced RAG 在文本检索前后增加 Query 改写、混合检索、过滤、Rerank 和 Context 选择，重点是提高召回与排序质量。GraphRAG 则显式建立实体和关系图，通过子图与路径取得多跳证据，也可以用社区摘要支持全局概括。

它们不是所有系统都必须依次升级的三个版本。Advanced RAG 优化文本检索链路，GraphRAG 改变知识表示和检索路径；生产系统可以组合关键词、向量和图检索。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## GraphRAG 适合什么问题？什么时候不值得使用？
{: #graph-rag-use-cases }

GraphRAG 适合实体关系密集、多跳关联、跨文档证据组合和资料库全局主题概括。例如查询公司之间的投资关系，或综合大量报告识别主要风险主题。

简单 FAQ、局部事实查询和小型知识库通常先用混合检索与 Rerank。图谱抽取存在错误，知识更新还要维护实体、关系与摘要；只有评测证明普通文本检索持续败在关系或全局视角上时，这些成本才值得承担。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## RAG 为什么需要 Query 改写？
{: #rag-query-rewriting }

用户问题可能存在口语与文档术语的语义鸿沟、多轮指代和复杂子目标。Query 改写把它转换成一个或多个独立、明确、适合检索的查询，从而提高召回率。

改写不是必选步骤。简单查询直接检索更快；复杂、多轮或初次召回不足时，改写才更有价值。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query 改写有哪些常见策略？
{: #rag-query-rewriting-strategies }

上下文补全把“它为什么失败”改成带实体和条件的独立查询；查询扩展补充同义词、领域术语或多个改写；任务分解把多跳问题拆成子查询。

HyDE 生成假设性相关文档并用其 Embedding 检索；Step-back 先提出更抽象的原则性问题。Step-back 与细粒度任务分解解决的不是同一问题。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query Drift 怎样发现和控制？
{: #rag-query-drift }

改写可能丢失原问题中的实体、时间、租户或约束，使检索偏离用户意图。可以同时保留原始 Query 与改写 Query，混合召回并比较结果；改写缺少关键约束时拒绝使用。

还可以比较语义相似度和实体覆盖，但固定阈值不是通用答案。最终应通过原问题对应的召回与回答指标评估。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query 改写怎样控制延迟和成本？
{: #rag-query-rewriting-cost }

先做意图或复杂度路由，只对多轮、组合问题和召回不足的查询改写。可以使用较小模型生成改写，并将多个独立查询并行检索。

多查询会增加向量检索、关键词检索和 Rerank 成本。优化目标是端到端回答质量，而不是生成尽可能多的 Query。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Chunk 大小怎样选择？
{: #rag-chunk-size }

Chunk 太小会缺少上下文，太大会降低匹配精度并占用更多 Context。应根据文档结构、问题类型和检索指标选择，而不是套用固定数字。

父子索引可以用小块匹配，再返回所属大块，兼顾检索精度和上下文完整性，但会增加索引与映射复杂度。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## 父子索引什么时候值得使用？
{: #rag-parent-child-index }

当定位问题需要细粒度匹配，而回答又必须读取完整段落、表格说明或上下章节时，父子索引更有价值。子块用于召回，命中后回到父块；多个子块指向同一父块时还应去重，避免 Context 重复。

它会增加索引、映射和返回内容，父块过大仍会带来噪声。是否采用以及父子块大小，应比较证据 Recall、回答忠实度、Context Token 和延迟，不能只看向量相似度。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## 为什么 RAG 常结合关键词检索和 Rerank？
{: #hybrid-search-rerank }

向量检索擅长语义近似，BM25 等关键词检索更容易找到错误码、编号和专有名词。两路先分别召回候选，再使用加权分数、归一化分数或 RRF 等排名融合；融合比例没有通用值，应在带相关证据标注的查询集上调整。

融合后的候选再交给 Cross-Encoder、专用 Reranker 或模型做更精细排序。这样可以把快速召回和精细判断分开，但会增加延迟、成本和调参工作。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Rerank 后保留多少内容？怎样处理 Context 预算？
{: #rag-rerank-context-budget }

没有通用的块数或 Top-K。系统先取得较大的候选集并 Rerank，再按分数、来源多样性和 Token 预算逐个选择，去除重复或高度重叠内容；必要时返回父块、压缩片段，或为不同来源保留配额。

参数要用端到端评测确定：过少会降低证据 Recall，过多会增加噪声、Lost in the Middle、延迟和费用。应同时比较检索指标、回答忠实度、任务成功率、Token 与 P95/P99，而不是因为某次实验效果好就永久固定为几个块。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## RAG 系统怎样评测？评测集包含什么？
{: #rag-evaluation-dataset }

检索和生成要分开评测。检索关注 Recall@K、Precision@K、MRR 或 NDCG，以及权限、版本和时间过滤是否正确；生成关注答案是否解决问题、结论是否由证据支持、引用是否对应原文。端到端还要观察无答案问题能否拒答、延迟和成本。

评测样本至少保存原始问题、必要上下文、应召回证据、允许答案或判断规则、无答案标记、资料版本和权限范围。样本应来自真实查询与故障，并覆盖术语差异、多跳、过期或冲突文档和权限隔离；人工校验后保留独立测试集。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## RAG 优化应从哪里开始？
{: #rag-optimization }

先根据 Trace 和分层指标定位第一次失败：证据未进入候选就改 Query、Chunk、Embedding、过滤或召回；正确证据已召回但排序靠后就改融合与 Rerank；证据进入 Context 后仍答错，再改 Context 结构、Prompt、模型或结果校验。

每次尽量只修改一层，在同一评测集上比较总体质量、严重退化、延迟、Token 和费用，必要时做消融。只提高平均相似度不代表回答变好，相似度也不能跨模型或索引直接比较。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[Agent 质量改进]({{ site.baseurl }}/docs/ai-agent/agent-quality/improvement/)。

## RAG 的端到端性能怎样优化？
{: #rag-performance }

先把延迟拆成 Query 处理、各路召回、融合、Rerank、Context 组装、模型首 Token 和完整生成，再优化占比最大的阶段。常见手段包括并行执行独立检索、限制候选集、批量或缓存 Embedding、使用元数据缩小搜索范围、缓存有版本和权限边界的结果，以及按查询复杂度决定是否改写和 Rerank。

缓存键必须包含租户、权限、知识版本、查询和检索配置。并行会提高下游瞬时并发，缓存会引入陈旧与污染风险，因此优化后仍要同时观察回答质量、P95/P99、资源饱和度和成本。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## 为什么代码搜索不一定优先使用 RAG？
{: #code-search-rag-vs-grep }

函数名、变量名和错误码需要精确匹配时，Grep 或符号索引通常更直接。自然语言描述、跨文件概念和相似实现更适合语义检索。

工程系统可以组合两者，技术更复杂不代表在所有问题上更准确。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。
