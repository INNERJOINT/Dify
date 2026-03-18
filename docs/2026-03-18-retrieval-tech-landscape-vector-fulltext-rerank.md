# 向量库、全文检索与重排技术方向总结

- 日期：2026-03-18
- 主题：向量库 / 全文检索 / 稀疏语义检索 / 混合检索 / 重排
- 术语约定：本文中"稠密向量检索"与 dense retrieval 含义相同；"重排"与 rerank 含义相同；"全文检索"特指基于倒排索引的 BM25 等词项匹配方案。

## 概述

这份文档单独整理“向量库 / 全文检索 / 重排”的分析结论，重点回答三个问题：

1. 当前主流检索系统的发展方向是什么。
2. 向量库、全文检索、稀疏检索和重排在系统里各自扮演什么角色。
3. 面向企业搜索、RAG 和代码搜索时，应该怎样组合这些能力。

核心结论是：

- 行业不是在做“向量库替代全文检索”，而是在走“混合召回 + 多阶段排序 + 成本优化”。
- 在需要精确匹配的检索系统中，全文检索没有退场，反而仍是底座。
- 向量检索更适合补充语义召回能力，而不是单独承担所有检索职责。
- `cross-encoder rerank` 仍然是很多生产系统中的效果上限。

---

## 1. 总体趋势

截至 2026-03-18，主流检索系统更接近下面这条路线：

- 第一阶段：全文检索 / 稠密向量 / 稀疏语义检索并行召回
- 第二阶段：融合排序，例如 `RRF`
- 第三阶段：可选重排，例如 `cross-encoder rerank`

换句话说，现代检索系统越来越少直接采用“单一 ANN top-k”作为最终方案，而是强调不同召回信号的组合。

这种趋势出现的原因很现实：

- 纯全文容易漏掉语义近义表达
- 纯向量容易丢掉精确 token、路径、术语和结构化约束
- 纯重排成本太高，不能直接拿来做主召回

因此，检索系统的主流方向不是“单技术统一天下”，而是“多技术分层协作”。

---

## 2. 全文检索为什么仍然是主力

对代码、SKU、路径、人名、专有名词、接口名、文件名这类强精确匹配场景，`BM25`、短语查询、字段加权、过滤和 facet 仍然非常关键。

全文检索重新变得重要的原因包括：

- 对精确 token 的召回能力仍然最强
- 对过滤、聚合、facet、权限控制支持成熟
- 对低延迟、大规模场景更稳定
- 对调试和可解释性更友好

因此，无论是在企业搜索、RAG，还是代码搜索里，全文检索都没有退场，反而重新成为系统底座的一部分。

如果一个场景具有以下特征，就不适合放弃全文检索：

- 用户会搜精确术语
- 查询里有结构化约束
- 需要强过滤和权限控制
- 结果需要可解释和可调试

---

## 3. Hybrid 为什么成为默认路线

主流搜索系统都在推荐全文和语义检索并行，再做融合，而不是二选一。

常见组合包括：

- `BM25 + dense embedding`
- `BM25 + sparse semantic + dense semantic`
- `BM25 + dense retrieval + rerank`

这类方案的优点是：

- 用全文检索守住精确匹配
- 用语义检索补足自然语言和近义表达
- 用融合排序减少单一路径的偏差

行业里最常见的融合方法是 `RRF`，因为它工程简单、鲁棒性强，不依赖不同分数体系之间做复杂标定。

在工程实践里，Hybrid 往往不是“可选增强项”，而是越来越接近默认配置。

---

## 4. 稠密向量检索的定位

稠密向量检索最擅长的是：

- 自然语言查询
- 近义表达匹配
- 语义相似内容召回
- 文本表述差异较大的场景

它非常适合：

- 通用 RAG
- FAQ 检索
- 用户问题和原文表达差异较大的文档库
- 需要支持自然语言描述的搜索入口

但它的典型短板也很明确：

- 对精确 token 不稳定
- 对路径、编号、SKU、类名、函数名一类 query 常常不如全文
- 可解释性较差
- 排序分数不总是适合直接拿来做最终排序

关于 embedding 模型的选择，当前常见选项包括：

- 通用多语言：`BGE-M3`（同时支持稠密、稀疏、多向量三种表示）
- 高维度通用：`text-embedding-3-large`（OpenAI，3072 维，支持 Matryoshka 降维）
- 多语言长上下文：`Jina Embeddings v3`（8K 上下文，支持 task-specific LoRA 适配）
- 轻量高效：`GTE-base` / `E5-base-v2` 等中等规模模型

选型时需要考虑：模型语言覆盖、维度与量化、上下文长度、推理成本、以及是否需要同时输出稀疏表示（如 BGE-M3）。

因此，更合理的定位通常是：

- 稠密向量检索负责语义召回
- 不独自承担所有检索职责

---

## 5. 稀疏语义检索在上升

除了传统 BM25，还出现了 learned sparse retrieval，例如 `SPLADE`、`BGE-M3 sparse`。  
这类方案兼顾倒排索引效率和一定的语义扩展能力。

可以把这一方向理解为：

- 保留倒排索引体系的可扩展性和过滤能力
- 又不完全受限于原始词项匹配

它特别适合：

- 企业文档搜索
- 专业术语密集场景
- 需要兼顾语义和可解释性的检索系统

在很多场景里，稀疏语义不是替代 BM25，而是对 BM25 的升级和补充。

> 注意：当前主流 learned sparse retrieval 模型基本都是在自然语言语料上训练的。对代码 token、符号名、路径、宏、缩写这类内容，效果还没有形成统一稳定结论。在代码检索里，BM25 仍然是更可靠的稀疏基线。

---

## 6. 多向量和 Late Interaction

单向量无法很好表示长文档和细粒度匹配，于是出现了：

- `ColBERT` / `ColBERT v2`
- `JinaColBERT v2`（Jina AI 的 ColBERT 实现，支持多语言和 8K 上下文）
- `ColPali`（将 late interaction 扩展到视觉文档理解）
- Milvus 的 multivector search（原生支持多向量召回与融合）
- Qdrant 的 named vectors（同一 point 存储多个命名向量，用于不同召回路径，但非 ColBERT 式 token 级 late interaction）

它们主要解决的问题是：

- 单个向量压缩整段文本时会丢失细节
- 长文档、表格、PDF 页面、图文混排文档很难被一个 embedding 准确表示
- 查询和文档之间需要更细粒度的 token 或 patch 级匹配

因此，多向量和 late interaction 更适合：

- 长文档问答
- 多模态文档检索
- 高质量 RAG
- 复杂语义匹配，而不是简单关键词查询

代价也很明确：

- 索引体积更大（ColBERT 每个 token 产生一个向量，存储开销可达单向量的 50-100 倍）
- 检索和排序更贵
- 系统复杂度更高

所以在工程上通常会把它放在高价值场景，而不是对所有数据统一启用。

---

## 7. 重排为什么仍然重要

生产上最常见、最稳的精排方案仍然是 `cross-encoder rerank`。  
它适合对第一阶段召回出来的较小候选集做精排，而不适合作为主召回。

它的作用不是“把检索系统从零变好”，而是：

- 在候选集已经大致正确的情况下进一步提纯
- 帮助自然语言问题获得更符合语义的最终排序
- 在多信号召回之后给出更统一的相关性判断

关于重排的成本参考（以典型 API 服务为例）：

- Cohere Rerank：对 top 25 文档重排，整体延迟约 200-500ms（含网络）
- 自部署 cross-encoder（如 `bge-reranker-v2-m3`）：每个 query-doc pair 推理约 5-20ms（GPU），top 25 重排约 50-200ms（可批量并行）
- 相比之下，BM25 召回 top 100 通常在 5-20ms，稠密向量 ANN 召回在 10-50ms

这意味着重排的延迟通常是主召回的 5-20 倍，这也是它只能作为精排而非主召回的根本原因。

当前 rerank 模型的发展方向主要包括：

- 更长上下文
- 更好的多语言能力
- 更好的半结构化数据处理能力
- 更快的轻量模型与更强的高精度模型并存

在实际系统里，重排通常只对 top N 候选执行，例如 top 20 或 top 50。

这意味着：

- 重排是效果增强器
- 不是检索主引擎

---

## 8. 基础设施层的方向

向量索引和检索底座的主流方向包括：

- 量化
- SSD/DiskANN 类索引
- 更强的 metadata filter
- 增量更新
- 大规模 shard 管理

这说明检索系统的竞争点已经不只是“能不能搜”，而是：

- 能不能便宜地搜
- 能不能边更新边搜
- 能不能在大规模和多租户场景下稳定运行
- 能不能把权限、过滤、上下文隔离和在线更新真正做好

换句话说，基础设施能力正在从“算法能力”扩展到“运营级能力”。

---

## 9. 不同技术在系统中的职责

把“向量库 / 全文检索 / 重排”放在同一个系统里看，会更容易理解它们分别适合做什么：

- 全文检索：负责精确 token、路径、术语、字段过滤和结构化约束
- 稠密向量检索：负责自然语言语义召回、近义表达、模糊描述
- 稀疏语义检索：负责在倒排框架下补充语义扩展能力
- 重排：负责对候选集做更高质量的最终排序
- Graph 检索：负责基于实体关系的结构化推理召回（见第 10.6 节）

更实用的理解方式是：

- 全文检索是“底座”（尤其在需要精确匹配的场景中）
- 向量检索是“补充召回能力”
- 重排是“精排器”

---

## 10. 按场景的选型建议

### 10.1 通用企业知识库 / 通用 RAG

推荐默认组合：

- `BM25 + dense retrieval + RRF + cross-encoder rerank`

这是当前最稳的通用路线。

### 10.2 术语密集、专有名词多的场景

推荐：

- `BM25 + learned sparse + dense retrieval + rerank`

原因是这类场景既需要精确 token，又需要一定语义扩展能力。

### 10.3 长文档、PDF、图文页面、多模态检索

推荐：

- `dense retrieval + multivector / late interaction + rerank`

因为单向量通常不足以表达复杂文档结构。

### 10.4 大规模、低成本、强过滤场景

重点应该放在：

- 量化
- SSD/DiskANN 类索引
- 强 metadata filter
- 增量更新与 shard 管理

### 10.5 代码搜索

推荐优先级应调整为：

- `全文 / 符号 / 路径 / 正则`
- 必要时再补自然语言改写和轻量 rerank
- 不把向量检索作为主召回

代码搜索里最重要的往往不是“语义像不像”，而是：

- token 是否精确
- 路径是否合理
- 符号是否匹配
- 过滤条件是否正确

### 10.6 需要多跳推理或实体关系的场景

推荐引入 Graph 检索作为补充：

- 构建知识图谱（实体 + 关系），在召回阶段通过图遍历获取关联上下文
- 与向量检索 / 全文检索并行召回，融合后送入 rerank 或 LLM
- 典型场景：医疗问诊推理、法律条文关联、复杂企业知识问答

> 注意：GraphRAG 仍然是一个快速演进中的领域，工程成熟度不如传统混合检索方案。适合在数据本身具有强实体-关系结构时引入，不建议作为通用默认方案。

---

## 附录：面向 Dify 和 AOSP 代码检索的落地建议

> 以下结论来自本次讨论的特定语境（Dify + AOSP 代码库），不适用于所有场景。

**关于 AOSP 代码库检索**：

- 不应让向量库主导检索架构
- 主检索应优先使用 Zoekt、Sourcegraph 这类代码搜索引擎
- 如需支持“自然语言问代码”，在代码搜索引擎之上增加 query rewrite 和轻量 rerank 即可
- 向量检索只作为自然语言入口的补充层

**关于 Dify 外部知识库集成**：

- Dify 只需要一个稳定的 `/retrieval` API 契约
- 真正的召回策略（混合检索、融合排序、重排）应封装在自有检索服务中
- 不应依赖 Dify 本身处理复杂召回策略

---

## 参考资料

**混合检索与搜索引擎**

- [Elastic Hybrid Search](https://www.elastic.co/docs/solutions/search/hybrid-semantic-text)
- [Azure AI Search Hybrid Search](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview)
- [OpenSearch Semantic and Hybrid Search](https://docs.opensearch.org/latest/tutorials/vector-search/neural-search-tutorial/)

**向量数据库**

- [Qdrant Vectors / Hybrid Queries](https://qdrant.tech/documentation/concepts/vectors/)
- [Milvus Multi-Vector Search](https://milvus.io/docs/multi-vector-search.md)

**重排**

- [Cohere Rerank](https://docs.cohere.com/v2/docs/rerank)
- [Voyage Reranker](https://docs.voyageai.com/docs/reranker)

**稀疏检索与多向量**

- [ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT](https://arxiv.org/abs/2004.12832)
- [ColPali: Efficient Document Retrieval with Vision Language Models](https://arxiv.org/abs/2407.01449)
- [SPLADE v2: Sparse Lexical and Expansion Model for Information Retrieval](https://arxiv.org/abs/2109.10086)
- [BGE-M3: Multi-Lingual, Multi-Functionality, Multi-Granularity Text Embeddings](https://arxiv.org/abs/2402.03216)

**融合排序**

- [Reciprocal Rank Fusion (RRF)](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)

**向量索引**

- [DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node](https://arxiv.org/abs/1900.02)

**Graph 增强检索**

- [Microsoft GraphRAG](https://github.com/microsoft/graphrag)

