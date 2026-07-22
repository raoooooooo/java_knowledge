# 05 - RAG 基础与离线建库

> RAG（Retrieval-Augmented Generation，检索增强生成）是企业落地 AI 最高频的场景，也是面试必考。13 章已讲 RAG 原理，本章从 Java 工程师视角讲**怎么建库、怎么选型、怎么落地**，以及面试怎么答。

---

## 一、RAG 全流程回顾（面试可背版）

```
【离线建库】                      【在线检索】        【生成】
文档 -> 加载 -> 切分 -> Embedding -> 存向量库   |  Query -> Embedding -> 检索 -> Rerank -> 拼 Prompt -> LLM 生成
```

- **离线建库**（本章）：文档加载 -> 文本切分（Chunking）-> 向量化（Embedding）-> 存入向量库。
- **在线检索**（06 章）：Query 向量化 -> 检索 Top-K -> Rerank 重排。
- **生成**：把片段拼进 Prompt -> LLM 基于上下文回答。

> 💡 **面试金句**：RAG 的瓶颈往往不在生成而在检索。建库质量（切分+Embedding）决定上限，检索优化决定下限。

---

## 二、文档加载与切分 ★ 核心考点

> 切分质量直接决定检索质量。切得不好，检索再准也召回不到完整语义。这是 RAG 最容易被忽视又最重要的环节。

### 2.1 为什么切分难

- 切太长：一个 chunk 包含多个主题，检索到了但相关性被稀释，还费 token。
- 切太短：语义被切断，一个完整意思被劈成两半，召回不全。
- 理想：按语义边界切，每个 chunk 主题集中、语义完整。

### 2.2 切分策略对比

| 策略 | 原理 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| **固定长度** | 每 N 个字符切一段 | 简单 | 易切断语义 | 不推荐单独用 |
| **固定+重叠** | 固定长度但相邻块重叠（如 overlap=200） | 缓解切断 | 仍有重复 | 通用基线 |
| **按语义边界** | 按段落/标题/Markdown 标题切 | 语义完整 | 长度不均 | 结构化文档 |
| **递归切分** | 先按大结构切，超长再递归细分 | 兼顾结构与长度 | 实现稍复杂 | 通用推荐（LangChain 默认） |
| **语义切分** | 按 Embedding 相似度变化点切 | 切在语义跳变处 | 慢（要算 Embedding） | 高质量场景 |
| **父子/小块大块** | 小块检索、返回所属大块 | 检索准+上下文全 | 结构复杂 | 生产推荐 |

### 2.3 重叠（Overlap）的作用

- 相邻 chunk 保留一段重叠文本（如 200 字符），防止关键信息正好被切断。
- 经验：overlap 设 chunk size 的 10%~20%。

### 2.4 父子块（Small-to-Big）★ 生产推荐

- 痛点：小块检索准但上下文不全，大块上下文全但检索不准。
- 方案：**用小块做检索匹配，命中后返回它所属的父块（大块）给 LLM**。
- 兼顾检索精度和上下文完整性，是生产 RAG 的主流做法。

### 2.5 Java 实战：Spring AI 文档切分

```java
// 递归切分（推荐通用）
DocumentReader reader = new TikaDocumentReader("classpath:policy.pdf");
List<Document> docs = reader.get();

TextSplitter splitter = TokenTextSplitter.builder()
    .chunkSize(500)        // 每块约500 token
    .minChunkSizeChars(350)
    .chunkOverlap(100)     // 重叠100
    .build();
List<Document> chunks = splitter.apply(docs);
```

```java
// Spring AI Alibaba 的语义切分（按 Markdown 结构）
DocumentTransformer transformer = new MarkdownDocumentTransformer();
List<Document> structured = transformer.apply(docs);
```

### 2.6 LangChain4j 切分

```java
DocumentSplitter splitter = DocumentSplitters.recursive(500, 100);  // 500字符，重叠100
List<TextSegment> segments = splitter.split(document);
```

---

## 三、Embedding 模型选型

### 3.1 Embedding 是什么

- 把文本映射成高维向量（如 768/1024/1536 维），让「语义相近 -> 向量相近」。
- 检索时算 Query 向量和文档向量的相似度（余弦/点积），取最相近的。

### 3.2 主流 Embedding 模型

| 模型 | 出身 | 特点 | 适用 |
|------|------|------|------|
| **BGE 系列** | 智源 | 中文强、开源可私有化、MTEB 榜单常客 | 国内首选 |
| **通义 Embedding** | 阿里 | text-embedding-v3，中文好、百炼调 | 国内、Spring AI Alibaba |
| **Cohere Embed** | Cohere | 多语言强 | 海外 |
| **OpenAI text-embedding-3** | OpenAI | 通用、生态广 | 海外 |
| **Jina Embed** | Jina | 长文本支持好 | 长文档 |

> ⚠️ **Embedding 模型和 LLM 是两个独立模型**，可分别选型。常见误解是「用 Qwen 的 LLM 就得用 Qwen 的 Embedding」--不必。

### 3.3 Java 调用 Embedding

```java
// Spring AI（以通义为例）
spring.ai.openai.chat.options.model=qwen-plus
spring.ai.openai.embedding.options.model=text-embedding-v3

EmbeddingModel embeddingModel;  // 注入
float[] vector = embeddingModel.embed("要向量化的文本");
```

```java
// LangChain4j
EmbeddingModel model = new OpenAiEmbeddingModel.builder()...build();
Response<Embedding> resp = model.embed(TextSegment.from("文本"));
float[] vector = resp.content().vector();
```

### 3.4 选型要点

- **语言**：中文选 BGE/通义，英文选 OpenAI/Cohere。
- **维度**：维度高表达强但存储/检索成本高。1024 维是常见平衡点。
- **私有化**：要数据不出域，选开源 BGE 自部署。
- **一致性**：建库和检索必须用**同一个** Embedding 模型（向量空间一致），换模型要全量重建。

---

## 四、向量数据库选型 ★ 高频面试题

### 4.1 主流向量库对比

| 向量库 | 类型 | 特点 | 适用场景 |
|--------|------|------|---------|
| **Milvus** | 专用 | 开源、云原生、支持 HNSW/IVF、十亿级、可分布式 | 大规模生产、私有化 |
| **Pgvector** | 插件 | PostgreSQL 扩展 | 已有 PG、不想引入新组件、中小规模 |
| **Elasticsearch** | 搜索引擎 | 已有倒排+向量(dense_vector)、混合检索天然 | 已用 ES、要全文+向量混合 |
| **Qdrant** | 专用 | Rust、过滤强、轻量 | 需丰富元数据过滤 |
| **Redis** | 内存 | RediSearch 向量、低延迟 | 缓存+向量、语义缓存 |
| **Chroma** | 轻量 | 嵌入式、开发友好 | 原型/小项目 |

### 4.2 选型决策树

```
数据量 < 百万 + 已有 PG          -> Pgvector（不引入新组件）
数据量大 + 需私有化 + 专业向量     -> Milvus
已用 ES + 要全文+向量混合检索      -> Elasticsearch
需要强元数据过滤                   -> Qdrant
要低延迟 + 语义缓存                -> Redis
原型验证                          -> Chroma
```

### 4.3 核心索引算法（决定检索速度精度，呼应 13 章）

- **HNSW**（Hierarchical Navigable Small World，分层可导航小世界图）：基于图，查询快、精度高、吃内存，主流。
- **IVF**（Inverted File，倒排文件）：聚簇+可选 PQ 量化压缩内存，适合超大规模。
- **Flat**：暴力精确，只适合小数据。

> 💡 **面试金句**：要速度精度选 HNSW（吃内存）；超大规模要省内存选 IVF+PQ（牺牲少量精度）。

### 4.4 Spring AI VectorStore 抽象（一套代码切库）

```java
// 接入 Milvus
@Bean
VectorStore vectorStore(MilvusConnectionDetails details) {
    return MilvusVectorStore.builder(details).build();
}

// 或 Pgvector
@Bean
VectorStore vectorStore(JdbcTemplate jdbc) {
    return PgVectorStore.builder(jdbc).dimensions(1024).build();
}

// 统一 API：写入
vectorStore.add(List.of(new Document("文本内容")));
// 检索
List<Document> results = vectorStore.similaritySearch("查询");
```

- Spring AI 的 `VectorStore` 接口屏蔽底层库差异，换库只改配置。
- LangChain4j 的 `EmbeddingStore` 同理。

---

## 五、完整建库实战（Spring AI + Milvus）

```java
@Service
public class KnowledgeBaseService {
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;

    // 离线建库：文档 -> 切分 -> 向量化 -> 入库
    public void buildFromPdf(Resource pdf) {
        // 1. 加载
        List<Document> docs = new TikaDocumentReader(pdf).get();
        // 2. 切分
        TextSplitter splitter = TokenTextSplitter.builder()
            .chunkSize(500).chunkOverlap(100).build();
        List<Document> chunks = splitter.apply(docs);
        // 3. 加元数据（来源、页码、时间，便于过滤和溯源）
        chunks.forEach(d -> d.getMetadata().put("source", pdf.getFilename()));
        // 4. 入库（VectorStore 自动调 Embedding 向量化 + 存储）
        vectorStore.add(chunks);
    }
}
```

- 切分后加**元数据**（来源/页码/时间/分类）很重要，用于检索过滤和回答溯源。
- Spring AI 的 `vectorStore.add()` 自动完成「调 Embedding + 存向量」。

---

## 六、生产踩坑

1. **切分把表格/代码切断**：固定长度切分会破坏代码块和表格。按结构切（Markdown/代码块感知），或对代码用专门切分器。
2. **Embedding 模型换了没重建库**：向量空间不一致，检索全乱。换模型必须全量重建。
3. **没加元数据**：检索回来不知道是哪个文档哪一页，无法溯源，也无法过滤。建库时务必加来源/时间/分类。
4. **chunk size 设太大**：一个 chunk 几千 token，检索到了但相关性稀释，还费 LLM token。500~1000 token 是经验值。
5. **PDF 解析丢格式**：Tika 解析复杂 PDF 可能丢表格/图片。重要文档要用专业解析（如表格识别）。
6. **增量更新难**：文档改了怎么同步？常见用文档 hash/version 判断是否需要重新切分入库，删旧加新。
7. **向量库没建索引**：百万级数据用 Flat 暴力检索慢。数据量上来要建 HNSW 索引。

---

## 七、常见面试题

1. **RAG 的完整流程是什么？**
   离线建库（加载->切分->Embedding->存向量库）+ 在线检索（Query Embedding->检索 Top-K->Rerank）+ 生成（拼 Prompt->LLM）。瓶颈往往在检索而非生成。

2. **文档怎么切分？有哪些策略？**
   固定长度（+重叠缓解切断）、按语义边界（段落/标题）、递归切分（兼顾结构与长度，推荐）、语义切分（按相似度变化点）、父子块（小块检索大块返回，生产推荐）。经验：chunk 500~1000 token，overlap 10%~20%。

3. **向量数据库怎么选？**
   看规模和场景：中小+已有 PG 用 Pgvector；大规模+私有化用 Milvus；已用 ES+要混合检索用 ES；强元数据过滤用 Qdrant；低延迟+缓存用 Redis；原型用 Chroma。Spring AI 的 VectorStore 抽象可一套代码切库。

4. **HNSW 和 IVF 怎么选？**
   HNSW 基于图、速度快精度高但吃内存，主流。IVF 聚簇+PQ 量化省内存适合超大规模，牺牲少量精度。要速度精度选 HNSW，要省内存选 IVF+PQ。

5. **Embedding 模型和 LLM 是一个模型吗？**
   不是，两个独立模型可分别选型。中文 Embedding 选 BGE/通义，LLM 选 Qwen/DeepSeek。注意建库和检索必须用同一个 Embedding（向量空间一致），换模型要全量重建。

6. **切分把语义切断了怎么办？**
   用按语义边界切分（段落/标题/Markdown）、加重叠（overlap）、或父子块策略（小块检索准，返回所属大块保证上下文完整）。代码和表格要用结构感知切分。

7. **知识库怎么增量更新？**
   按文档 hash/version 判断是否变更，变更则删旧 chunk 加新 chunk。或用时间分区，过期数据整体失效。配合 ILM 思路管理向量库生命周期。

8. **你项目里 RAG 建库怎么做的？**
   （参考）文档加载用 Tika，递归切分 500 token+100 重叠，加来源/页码/时间元数据，BGE/通义 Embedding，存 Milvus（HNSW 索引）。Spring AI VectorStore 统一抽象。踩过切分切断表格的坑，后改用结构感知切分。

---

## 八、资料勘误与重点提醒

1. **「chunk 越大信息越多越好」是误区**：chunk 太大相关性稀释、费 token、检索精度下降。经验 500~1000 token 最佳，质量 > 数量。
2. **「固定长度切分够用」不严谨**：固定长度会切断语义、破坏代码表格。生产要用结构感知/递归/父子块，固定长度只做基线。
3. **「向量库只看检索速度」不全面**：选型还要看元数据过滤能力、是否支持混合检索、运维成本、是否可私有化。资料常只比速度维度。
4. **「换 Embedding 不影响已有数据」是严重错误**：向量空间不一致，检索全乱。换 Embedding 必须全量重建向量库。资料偶尔漏提。
5. **「建库只管切分入库」漏了元数据**：元数据（来源/页码/时间/分类）是检索过滤和回答溯源的基础，没元数据的 RAG 无法生产。务必建库时加。
6. **Pgvector 不是「小规模才用」**：资料常说 Pgvector 只适合小数据。实际上配合 HNSW 索引，Pgvector 能撑千万级，对已有 PG 栈的团队是性价比之选。

---

> 下一章「06-RAG 检索与召回优化」：建好库后怎么检索得准、Rerank、查询改写、上下文压缩。
