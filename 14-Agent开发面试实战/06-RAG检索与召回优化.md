# 06 - RAG 检索与召回优化

> 建好库只是起点，检索准不准决定 RAG 成败。本章讲检索的三种模式、Rerank 重排、查询改写、上下文压缩--这些是「RAG 检索不准怎么优化」的标准答案。面试必考。

---

## 一、检索的三种模式 ★

### 1.1 稠密检索（Dense）

- Query 和文档都过 Embedding，算向量相似度（余弦/点积）。
- **擅长语义匹配**：搜「手机续航差」能召回讲「电池不耐用」的文档（没共同词但语义近）。
- **弱点**：精确词匹配差（人名、型号、代码标识符、专有名词，向量相似度可能很低）。

### 1.2 稀疏检索（Sparse）

- 基于 BM25 等关键词检索，本质是带权重的倒排匹配。
- **擅长精确词匹配**：搜「订单号 ORD2026001」能精准命中。
- **弱点**：语义匹配弱（搜「续航差」命中不了「电池不耐用」）。

### 1.3 混合检索（Hybrid）★ 生产推荐

- Dense + Sparse 融合，取长补短。
- 融合方法：**RRF（Reciprocal Rank Fusion，倒数排名融合）** 最常用--不比分数只比排名，把两路结果按排名倒数加权合并。
- 这是生产 RAG 的推荐做法：既抓语义又抓精确词。

```java
// Spring AI 混合检索（Elasticsearch 向量+全文）
SearchRequest req = SearchRequest.builder()
    .query("手机续航差")
    .topK(20)
    .similarityThreshold(0.5)
    .build();
List<Document> docs = vectorStore.similaritySearch(req);
// ES VectorStore 支持 dense_vector + 全文混合打分
```

> 💡 **面试金句**：混合检索（Dense+Sparse+RRF 融合）是生产推荐，既抓语义又抓精确词。纯向量检索对专有名词/代码标识符弱，纯关键词对语义弱，融合取长补短。

---

## 二、Rerank 重排 ★ 必考

### 2.1 为什么需要 Rerank

- **Embedding 检索是 Bi-Encoder（双塔）**：Query 和 Doc 分别编码再算相似度，快但粗（能预计算，适合从百万文档召回 Top-50）。
- **Rerank 是 Cross-Encoder（交叉编码）**：把 Query 和 Doc 拼一起过模型，更准但慢（不能预计算，两两算）。
- 所以：**检索阶段用 Bi-Encoder 快召回 Top-50，Rerank 阶段用 Cross-Encoder 精排取 Top-5**。兼顾速度和精度。

### 2.2 流程

```
Query -> Embedding -> 向量检索召回 Top-50 (Bi-Encoder，快)
                      ↓
                   Rerank 精排 (Cross-Encoder，准但慢，只算50条)
                      ↓
                   取 Top-5 拼进 Prompt
```

### 2.3 主流 Rerank 模型

| 模型 | 出身 | 特点 |
|------|------|------|
| **bge-reranker** | 智源 | 中文强、开源、可私有化 |
| **Cohere Rerank** | Cohere | 效果好、API 调用 |
| **通义 Rerank** | 阿里 | gte-rerank，百炼调 |
| **Jina Rerank** | Jina | 多语言 |

### 2.4 Java 集成

```java
// Spring AI Alibaba 集成 Rerank（DashScope）
@Bean
public RerankModel rerankModel() {
    return new DashScopeRerankModel(dashScopeApiKey, "gte-rerank");
}

// 检索后重排
List<Document> candidates = vectorStore.similaritySearch(req);  // 召回50
List<Document> reranked = rerankModel.rerank(query, candidates, 5); // 精排取5
```

> 💡 **面试金句**：检索用 Bi-Encoder 快召回（可预计算），Rerank 用 Cross-Encoder 精排（Query+Doc 拼一起，更准但慢）。两阶段：先粗筛后精排，兼顾速度精度。

---

## 三、查询改写与扩展 ★

### 3.1 为什么要改写 Query

- 用户提问往往口语化、模糊、带指代（「那个怎么用」），直接检索召回差。
- 改写让 Query 更适合检索。

### 3.2 常见改写手段

| 手段 | 原理 | 适用 |
|------|------|------|
| **查询扩展** | 用同义词/相关词扩展 Query | 召回不足 |
| **查询分解** | 把多意图问题拆成多个子查询 | 复杂问题 |
| **HyDE** | 让 LLM 先「假设一个答案」，用假设答案的 Embedding 检索 | Query 和文档风格差异大 |
| **查询路由** | 根据问题类型决定查哪个库/哪种检索 | 多源多库 |
| **多查询生成** | LLM 生成多个改写 Query 并行检索再合并 | 提升召回 |

### 3.3 HyDE（Hypothetical Document Embeddings）★ 进阶

- 痛点：用户 Query 短、口语化，和文档（长、正式）风格差异大，向量相似度低。
- HyDE：先让 LLM 根据问题「假设一个答案文档」，用这个假设答案的 Embedding 去检索真实文档。
- 假设答案虽然可能不准，但它的**风格/词汇更接近真实文档**，检索召回更高。

```
用户问："怎么配数据库连接池？"
LLM 假设答案："数据库连接池配置包括最大连接数、最小空闲、超时时间..."
用假设答案 Embedding 检索 -> 命中真实文档（风格接近）
```

### 3.4 查询路由

- 不同问题查不同源：技术文档查 Confluence，代码查 Git，FAQ 查知识库。
- 用 LLM/分类器判断 Query 类型，路由到对应检索源。

---

## 四、元数据过滤与 Top-K 调优

### 4.1 元数据过滤

- 检索前用元数据缩小范围，既提精度又提速。
- 例：只检索「2026 年」「产品A」的文档。

```java
SearchRequest req = SearchRequest.builder()
    .query("退款政策")
    .filterExpression("source == 'productA' && year == 2026")  // 元数据过滤
    .topK(10)
    .build();
```

### 4.2 Top-K 调优

- K 太小：召回不全，漏关键信息。
- K 太大：引入噪声（不相关片段），稀释注意力、费 token。
- 经验：召回阶段 Top-50，Rerank 后取 Top-5~10 给 LLM。
- 配合 `similarityThreshold`（相似度阈值）过滤低质量结果。

---

## 五、上下文压缩（Context Compression）★

### 5.1 为什么需要

- 检索回来几十段文档，全塞进 Prompt 超窗口、费 token、稀释注意力。
- 需要压缩：只保留与 Query 最相关的部分。

### 5.2 三种方法

| 方法 | 原理 | 代表 |
|------|------|------|
| **抽取式** | 只保留相关句子/token，删低相关 | LLMLingua 系列 |
| **生成式（摘要）** | 用 LLM 对检索片段做摘要再注入 | LLM 摘要 |
| **过滤式** | 去重、去低分、按相关度截断 | 简单过滤 |

### 5.3 LLMLingua

- 用小模型给每个 token 打分，删掉低相关的，可压到原长 1/10，几乎不损效果。
- 省 token、提速度、降噪声。

### 5.4 工程取舍

- 摘要式压缩有信息损失风险（LLM 摘要可能漏关键）。
- 抽取式更安全（保留原文）。
- 简单场景用过滤式（按相关度截断）够用。

---

## 六、完整检索优化链路

```
用户 Query
  ↓
[查询改写] HyDE/扩展/分解      -- 提召回
  ↓
[元数据过滤] 缩小范围            -- 提精度提速
  ↓
[混合检索] Dense+Sparse+RRF    -- 召回 Top-50
  ↓
[Rerank] Cross-Encoder 精排    -- 取 Top-5
  ↓
[上下文压缩] LLMLingua/过滤      -- 省 token 降噪声
  ↓
拼 Prompt -> LLM 生成 + 引用溯源
```

> 这条链路是「RAG 检索优化」的标准答题框架，面试按这条答。

---

## 七、生产踩坑

1. **只召回不 Rerank**：直接拿向量 Top-5 给 LLM，精度差。生产必须加 Rerank。
2. **Top-K 设太大**：塞 50 段进 Prompt，噪声大、费钱、超窗口。Rerank 后取 5~10 段。
3. **没做查询改写**：口语化 Query 直接检索，召回差。至少做查询扩展/HyDE。
4. **专有名词召回不到**：纯向量检索对人名/型号/代码弱，必须加稀疏检索做混合。
5. **相似度阈值没设**：低相关结果也返回，干扰回答。设 similarityThreshold 过滤。
6. **Rerank 模型和 Embedding 不一致**：不要求一致（Rerank 是 Cross-Encoder 独立模型），但要注意 Rerank 模型的语言匹配。
7. **压缩丢关键信息**：摘要式压缩可能漏关键，重要场景用抽取式或减小压缩比。

---

## 八、常见面试题

1. **RAG 检索不准怎么优化？**
   多层优化：查询改写（HyDE/扩展/分解）-> 元数据过滤缩小范围 -> 混合检索（Dense+Sparse+RRF）召回 -> Rerank（Cross-Encoder）精排取 Top-K -> 上下文压缩。这是标准框架。

2. **为什么需要 Rerank？和向量检索什么区别？**
   向量检索是 Bi-Encoder，Query/Doc 分别编码再算相似度，快但粗（可预计算，适合召回）。Rerank 是 Cross-Encoder，Query+Doc 拼一起精排，准但慢（两两算）。两阶段：Bi-Encoder 召回 50 -> Cross-Encoder 精排 5，兼顾速度精度。

3. **什么是混合检索？为什么要混合？**
   Dense（向量，语义匹配）+ Sparse（BM25，精确词匹配）+ RRF 融合。纯向量对专有名词/代码弱，纯关键词对语义弱，融合取长补短。生产推荐。

4. **HyDE 是什么？解决什么问题？**
   让 LLM 先假设一个答案，用假设答案的 Embedding 检索真实文档。解决用户 Query 短/口语化与文档长/正式风格差异大导致向量相似度低的问题。假设答案风格接近文档，召回更高。

5. **Top-K 怎么选？**
   召回阶段大（50）保召回率，Rerank 后小（5~10）防噪声。配合相似度阈值过滤低质结果。K 太小漏信息，太大引入噪声费 token。

6. **检索回来的上下文太长怎么办？**
   上下文压缩：抽取式（LLMLingua 删低相关 token，压到 1/10）、生成式（LLM 摘要）、过滤式（按相关度截断）。重要场景用抽取式防信息丢失。

7. **混合检索怎么融合结果？**
   用 RRF（倒数排名融合）：不比分数只比排名，`score = Σ 1/(k+rank)`，把 Dense 和 Sparse 两路结果按排名加权合并。比简单加权分数更稳（分数尺度不同不可比）。

8. **元数据过滤有什么用？**
   检索前按元数据（来源/时间/分类）缩小范围，提精度又提速。比如只查 2026 年产品A 的文档。Spring AI 用 filterExpression 表达。

---

## 九、资料勘误与重点提醒

1. **「向量检索就够准」是误区**：纯向量检索对专有名词/型号/代码标识符弱（向量相似度低）。资料常把向量检索说成万能，实际生产必须混合检索。
2. **「Rerank 和 Embedding 是一个模型」是误解**：Embedding 是 Bi-Encoder（双塔，可预计算），Rerank 是 Cross-Encoder（交叉，两两算），是两类不同模型，不要求一致。
3. **「召回越多越好」是错的**：Top-K 太大引入噪声、稀释注意力、费 token。Rerank 后取 5~10 段最佳。
4. **「RRF 直接加权分数」不严谨**：RRF 融合的是**排名**不是分数（Dense 和 Sparse 分数尺度不同不可直接比）。资料偶尔说成「加权分数」是错的。
5. **「HyDE 一定提升」不严谨**：HyDE 在 Query 和文档风格差异大时有效，若 Query 本身清晰且风格匹配，HyDE 反而引入假设答案的偏差。要按场景用。
6. **查询改写不是只有改写**：资料常只讲「改写」，漏了扩展、分解、路由、多查询生成等。改写是一类手段，要按痛点选。

---

> 下一章「07-RAG 进阶架构」：Self-RAG/CRAG/GraphRAG/Agentic RAG，区分度题，答上来加分。
