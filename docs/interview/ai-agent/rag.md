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

RAG 分成三部分：

1. **离线索引**：文档解析与清洗 → Chunk → Embedding → 建立向量、关键词和元数据索引；
2. **在线检索**：Query 改写 → 召回 → 过滤与融合 → Rerank → Context 选择 → 生成回答；
3. **分层评测**：检索是否找到正确证据，生成是否忠于证据。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Naive RAG、Advanced RAG 和 GraphRAG 有什么区别？
{: #naive-advanced-graph-rag }

- **Naive RAG**：Chunk 向量索引 + Query 向量检索 Top-K + 生成，是最小文本检索链路；
- **Advanced RAG**：加入 Query 改写、混合检索、过滤、父子索引、Rerank 和 Context 选择，重点是提高文本召回与排序质量；
- **GraphRAG**：显式建立实体和关系图，通过子图、关系路径和社区摘要取得多跳或全局证据。

三者不是必须依次升级的版本。**Advanced RAG 优化文本检索链路，GraphRAG 改变知识表示和检索路径。**生产系统也可以组合使用。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## GraphRAG 适合什么问题？什么时候不值得使用？
{: #graph-rag-use-cases }

**适合：**实体关系密集、多跳关联、跨文档证据组合和资料库全局主题概括，例如投资关系查询和多份报告的风险归纳。

**不适合：**简单 FAQ、局部事实查询和小型知识库。这些场景通常先用混合检索与 Rerank，因为图谱抽取、关系维护和社区摘要都有额外成本与错误。

判断标准是：只有评测证明普通文本检索持续败在**关系推理或全局视角**上，GraphRAG 才值得引入。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## RAG 为什么需要 Query 改写？
{: #rag-query-rewriting }

Query 改写主要解决三类问题：

- **语义鸿沟**：用户口语与文档术语不同；
- **上下文依赖**：多轮问题存在指代和省略；
- **复杂目标**：一个问题包含多个可独立检索的子问题。

它把原问题转换成一个或多个独立、明确、适合检索的 Query。简单查询直接检索更快；复杂、多轮或初次召回不足时才值得改写。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query 改写有哪些常见策略？
{: #rag-query-rewriting-strategies }

- **上下文补全**：完成指代消解，把多轮追问改成独立 Query；
- **查询扩展**：补充同义词、实体、错误码和领域术语，或生成多个改写；
- **任务分解**：把多跳问题拆成多个子查询；
- **HyDE**：生成假设性相关文档，用它的 Embedding 检索真实材料；
- **Step-back**：先提出更抽象的原则性问题，再辅助回答原问题。

HyDE 的假设文档是检索表示，不是事实答案；Step-back 与细粒度任务分解解决的也不是同一个问题。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query Drift 怎样发现和控制？
{: #rag-query-drift }

**Query Drift 是改写后的 Query 偏离用户原意。**常见表现是丢失实体、时间、租户或业务约束。

控制方法包括：

1. 保留原始 Query，与改写 Query 一起召回；
2. 检查关键实体和约束是否仍然存在；
3. 比较语义相似度与两路检索结果；
4. 改写不满足约束时拒绝使用。

固定相似度阈值没有通用答案，最终要用原问题的召回和回答指标验证。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Query 改写怎样控制延迟和成本？
{: #rag-query-rewriting-cost }

- **按需启用**：先做意图或复杂度路由，只改写多轮、组合或召回不足的查询；
- **降低单次成本**：使用较小模型完成分类和改写；
- **减少等待**：将独立的改写 Query 并行检索；
- **限制扩散**：控制改写数量和候选集规模，避免后续召回与 Rerank 成本成倍增长。

优化目标是**端到端回答质量与延迟**，不是生成尽可能多的 Query。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Chunk 大小怎样选择？
{: #rag-chunk-size }

核心取舍是：

- **太小**：匹配更精确，但可能缺少回答所需上下文；
- **太大**：上下文更完整，但语义混杂、匹配精度下降，还会占用更多 Token。

应根据文档结构、问题类型、证据 Recall、回答忠实度和 Context 成本评测，不套用固定数字。需要“小块匹配、大块回答”时可以使用父子索引。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## 父子索引什么时候值得使用？
{: #rag-parent-child-index }

**适用条件：定位需要细粒度匹配，回答又需要完整段落、表格说明或上下章节。**

- 子块参与检索，提高定位精度；
- 命中后返回所属父块，补充回答上下文；
- 多个子块指向同一父块时要去重。

代价是额外的索引、映射和 Context；父块过大仍会引入噪声。是否采用要比较证据 Recall、回答忠实度、Token 和延迟。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## 为什么 RAG 常结合关键词检索和 Rerank？
{: #hybrid-search-rerank }

它们分别解决**语义召回、精确词召回、细粒度排序**：

- **向量检索**：擅长语义近似；
- **BM25 等关键词检索**：擅长错误码、编号和专有名词；
- **融合**：通过归一化加权或 RRF 合并两路候选；
- **Rerank**：使用 Cross-Encoder、专用 Reranker 或模型，对少量候选做更精细的相关性判断。

融合比例没有通用值，应在带证据标注的查询集上调整；Rerank 提高精度，但会增加延迟和成本。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## Rerank 后保留多少内容？怎样处理 Context 预算？
{: #rag-rerank-context-budget }

**没有通用的块数或 Top-K。**常见过程是：

1. 第一阶段召回较多候选；
2. Rerank 后按分数排序；
3. 按 Token 预算逐块加入，同时考虑来源多样性和证据覆盖；
4. 去除重复和高度重叠内容，必要时返回父块或压缩片段。

过少会降低证据 Recall，过多会增加噪声、Lost in the Middle、延迟和费用。参数要通过检索指标、回答忠实度、任务成功率、Token 和 P95/P99 联合确定。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## RAG 系统怎样评测？评测集包含什么？
{: #rag-evaluation-dataset }

评测分三层：

- **检索层**：Recall@K、Precision@K、MRR、NDCG，以及权限、版本和时间过滤；
- **生成层**：答案相关性、证据忠实度、引用正确性和无答案处理；
- **端到端**：任务完成、延迟和成本。

评测样本至少包含：原始问题、应召回证据、允许答案或判断规则、无答案标记、资料版本和权限范围。样本应覆盖术语差异、多跳、过期或冲突文档和权限隔离，并保留独立测试集。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。

## RAG 优化应从哪里开始？
{: #rag-optimization }

**先找第一次失败的层次，再改对应机制：**

- 证据没有进入候选：改 Query、Chunk、Embedding、过滤或召回；
- 正确证据已召回但排名靠后：改融合与 Rerank；
- 证据已进入 Context 仍答错：改 Context 结构、Prompt、模型或结果校验。

每次尽量只修改一层，在同一评测集上比较质量、严重退化、延迟、Token 和费用，必要时做消融。平均相似度提高不等于回答变好。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[Agent 质量改进]({{ site.baseurl }}/docs/ai-agent/agent-quality/improvement/)。

## RAG 的端到端性能怎样优化？
{: #rag-performance }

先拆分延迟：**Query 处理、各路召回、融合、Rerank、Context 组装、模型首 Token、完整生成**，再优化占比最大的阶段。

常见手段包括：

- 并行执行相互独立的检索；
- 使用元数据过滤缩小范围；
- 限制候选集和 Rerank 数量；
- 批量或缓存 Embedding；
- 按 Query 复杂度决定是否改写和 Rerank；
- 缓存带租户、权限、知识版本和检索配置的结果。

并行会增加瞬时并发，缓存会引入陈旧和污染风险，因此要同时观察回答质量、P95/P99、资源饱和度与成本。

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)、[长 Context、压缩与缓存]({{ site.baseurl }}/docs/llm/context-cost-cache/)。

## 为什么代码搜索不一定优先使用 RAG？
{: #code-search-rag-vs-grep }

- **Grep 或符号索引**：适合函数名、变量名、错误码和调用关系等精确查询；
- **语义检索**：适合自然语言描述、跨文件概念和相似实现。

工程系统可以组合两者。**技术更复杂不代表在所有查询上更准确。**

相关内容：[RAG 与知识检索]({{ site.baseurl }}/docs/llm/rag/)。
