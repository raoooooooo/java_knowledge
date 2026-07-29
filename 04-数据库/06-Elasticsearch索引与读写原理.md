# ES 索引与读写原理

> 本章是 ES 的「灵魂」：倒排索引为什么快、分词器怎么工作、文档怎么写进去、查询怎么读出来、相关性怎么打分。理解这些，调优和面试题就都有了根基。

---

## 一、倒排索引（Inverted Index）★

> 这是 ES（和所有全文搜索引擎）的根基，必考。

### 1.1 为什么需要倒排索引

- **正排索引**：文档 -> 词（「文档 1 包含哪些词」）。查「包含某词的文档」要扫所有文档，慢。
- **倒排索引**：词 -> 文档列表（「这个词出现在哪些文档」）。查「包含某词的文档」直接查词，拿到文档列表，**O(1) 级别**。

```mermaid
graph LR
    subgraph 正排索引
        doc1["doc1"] --> hello1["hello"]
        doc1 --> world1["world"]
        doc2["doc2"] --> hello2["hello"]
        doc2 --> es1["es"]
    end

    subgraph 倒排索引
        hello3["hello"] --> doc_a["doc1, doc2"]
        world3["world"] --> doc_b["doc1"]
        es3["es"] --> doc_c["doc2"]
    end
```

### 1.2 倒排索引的组成

一个字段的倒排索引由三部分构成：

| 部分 | 作用 | 存哪 |
|------|------|------|
| **Term Dictionary（词典）** | 所有 term（词项）的列表 | 磁盘 |
| **Term Index（词典索引）** | 加速词典查找（FST） | 内存 |
| **Posting List（倒排表）** | 每个 term 对应的文档列表（含 docId、词频、位置、偏移） | 磁盘 |

查一个词的流程：**Term Index（FST）-> 定位到 Term Dictionary 中的 term -> 拿到对应的 Posting List（文档列表）**。

### 1.3 FST（Finite State Transducer，有限状态转换器）

- Term Dictionary 可能极大（几百万 term），逐个二分查找也慢。
- **FST** 是一种**前缀压缩的有限状态机**，把公共前缀共享、公共后缀共享，极度压缩，能放进**内存**。
- 查 term 时先在内存 FST 上走状态机定位，再少量磁盘 IO 读 Term Dictionary，极快。
- 类比：FST 像「字典的目录」，但比普通目录压缩得多。

### 1.4 Posting List 的压缩

- Posting List 是一串 docId，可能很长。ES 用压缩算法节省空间和加速：
  - **Frame of Reference（FOR）**：把递增 docId 变成差值（delta），再用变长位编码，小数字用少 bit。
  - **Roaring Bitmaps**：对大规模 docId 集合做高效位图压缩，做 AND/OR 交集极快（多条件查询时合并多个 Posting List 的关键）。

### 1.5 Doc Values（列式正排）★ 高频考点

- 倒排索引擅长「按词找文档」，但**不擅长排序和聚合**（排序要按文档取字段值，倒排是反着的）。
- 所以 ES 为（默认非 text）字段额外存一份**正排的列式存储**叫 **Doc Values**：`docId -> 字段值`，列存压缩。
- 排序、聚合、脚本访问字段值都走 Doc Values，不走倒排。
- **代价**：一份字段多存一份数据，是 ES 写入放大的来源之一（见 07 章写放大回顾，及 SkyWalking 存储引擎章节）。
- 可按字段关闭（`doc_values: false`）省空间，前提是该字段不需要排序/聚合/脚本访问。

> 💡 **一句话**：倒排管「搜索」，Doc Values 管「排序聚合」。二者并存是 ES 既快搜索又能聚合的代价。

### 1.6 还有哪些数据结构（呼应写放大）

一个 ES 文档落盘时同时存了多种结构（这是 ES 写放大的根源，详见 SkyWalking 存储引擎章节 + 本系列 07）：

| 数据结构 | 作用 |
|---------|------|
| `_source` | 原始 JSON 全文，GET 返回原文 |
| **倒排索引** | 每个 `index:true` 字段各一份，搜索用，放大主源 |
| `doc_values` | 列式正排，排序聚合 |
| BKD-Tree | 数值/地理/IP 范围查询 |
| `translog` | 预写日志 WAL，崩溃恢复 |

---

## 二、分词器（Analyzer）

### 2.1 分词三段式

文本字段写入和搜索时要先**分词**。Analyzer 由三部分组成：

```mermaid
graph TD
    A["原始文本"]
    --> B["char filter<br/>字符过滤（如去 HTML 标签）"]
    --> C["tokenizer<br/>分词（切成一个个 token，如按空格/中文切词）"]
    --> D["token filter<br/>token 过滤（转小写、去停用词、同义词、词干化）"]
    --> E["最终 token 流（建倒排 / 用于搜索）"]
```

- **character filter**：预处理字符，如 `html_strip` 去 HTML 标签。
- **tokenizer**：切词，如 `standard`（按 Unicode 文本边界切）、`whitespace`（按空格）。
- **token filter**：处理 token，如 `lowercase`（转小写）、`stop`（去停用词 the/a/an）、`synonym`（同义词）、`stemmer`（词干化 running->run）。

### 2.2 内置 vs 中文分词

- **standard analyzer**：ES 默认，对中文会**按字切**（"我爱北京" -> 我/爱/北/京），中文搜索效果差。
- **ik 分词器**：中文最常用插件，支持 `ik_smart`（粗粒度）和 `ik_max_word`（细粒度，尽可能多切词）。

```mermaid
graph LR
    subgraph ik_smart["ik_smart（粗粒度）"]
        A1["我爱北京天安门"] --> B1["我, 爱, 北京, 天安门"]
    end

    subgraph ik_max["ik_max_word（细粒度）"]
        A2["我爱北京天安门"] --> B2["我, 爱, 北京, 天安门, 安门"]
    end
```

### 2.3 ⚠️ 写入分词 vs 搜索分词（高频考点）

- 写入时用 `analyzer` 分词建倒排。
- 搜索时**默认用同一个 analyzer** 对查询文本分词，再去匹配倒排。
- 可单独指定 `search_analyzer`，让搜索时用不同分词（如写入用 `ik_max_word` 尽量多切保证召回，搜索用 `ik_smart` 粗切避免噪音）。
- **经典坑**：写入分词和搜索分词不一致会导致搜不到。比如写入按字切、搜索按词切，token 不匹配。

---

## 三、Mapping 与字段类型

### 3.1 动态映射 vs 显式映射

- **动态映射**：写入文档时 ES 自动推断字段类型。方便但有风险（如把 `"123"` 推成 text、把 ID 数字推成长整型导致精度问题、版本号 `"1.2"` 推成 float）。
- **显式映射**：建索引时手动定义每个字段类型，生产**强烈建议显式定义**，关掉动态推断（`"dynamic": false` 或 `strict`）。

### 3.2 text vs keyword ★ 必考

| 维度 | text | keyword |
|------|------|---------|
| 是否分词 | **分词**，建倒排 | **不分词**，整个值作为一个 term |
| 用途 | 全文搜索（match） | 精确匹配（term）、聚合、排序 |
| 能聚合/排序吗 | 不能（除非开启 fielddata，不推荐） | 能 |
| 例子 | 文章正文、商品标题 | 订单号、状态、标签、IP |

- 一个字段常**同时**设为 `text` + 子字段 `keyword`：既支持全文搜索又支持精确聚合排序：

```json
"title": {
  "type": "text",
  "analyzer": "ik_max_word",
  "fields": {
    "keyword": { "type": "keyword" }
  }
}
```
搜索用 `title`，排序/聚合用 `title.keyword`。

### 3.3 其他常见类型

- 数值：`long/integer/short/byte/double/float`。选够用即可，省空间。
- `date`：支持多种格式字符串解析为时间戳。
- `boolean`、`ip`、`geo_point`（地理坐标）、`geo_shape`。
- `object`：嵌套 JSON（默认会被 flatten，丢失数组独立性）。
- `nested`：**保持数组中每个对象的独立性**，能精确查询数组里某个对象同时满足多条件，代价高。

### 3.4 ⚠️ Mapping 一旦建好不能改字段类型

- 已有字段的类型**不能修改**（如 text 改 keyword），要改只能 reindex 到新索引。
- 这就是为什么生产要提前规划 mapping、用索引别名（alias）便于 reindex 无感切换。

---

## 四、写入流程（一篇文档怎么进 ES）

> 这是高频面试题，务必记住 refresh / flush / translog 三者的关系。

### 4.1 写入全流程

```mermaid
graph TD
    A["1. 写请求到 coordinating 节点<br/>路由到对应主分片所在节点"]
    --> B["2. 主分片节点"]
    --> C["a. 写入 indexing buffer（内存 buffer）"]
    --> D["b. 同时写 translog（WAL，磁盘）"]
    --> E["3. 副本分片：主分片写成功后<br/>并行转发到副本分片写入"]
    --> F["4. 全部成功才返回成功给客户端"]
```

### 4.2 三个关键动作：refresh / flush / translog

| 动作 | 触发 | 做了什么 | 结果 |
|------|------|---------|------|
| **refresh** | 默认每 **1s** | buffer 生成一个新 **segment**（内存中，可直接搜索） | 文档**近实时**可搜（NRT） |
| **flush** | translog 满（默认 512MB）或定时 | segment **fsync 落盘** + 清空 translog | 数据持久化，崩溃不丢 |
| **translog** | 每次写都写 | 预写日志 WAL | 崩溃后用 translog 恢复未落盘的数据 |

> ⚠️ **核心理解**：
> - **refresh** 让数据「可搜」（生成内存 segment），但**还没真正落盘**，断电会丢。
> - **flush** 才让数据「真正持久化」（fsync + 清 translog）。
> - **translog** 是 refresh 和 flush 之间的安全网：即使 segment 还在内存没 flush，translog 在磁盘，崩溃后能恢复。
> - 这就是 ES 的「**近实时（NRT）**」：写进去默认要等 1s refresh 才能搜到。

### 4.3 Segment 不可变 + Merge

- **Segment 是不可变的**：一旦生成不再修改。这带来好处（并发读无锁、易缓存）也带来代价。
- **删除是「标记删除」**：删文档只是在 `.del` 文件里标记，数据还在。
- **更新 = 标记旧文档删除 + 写新文档**：旧版本留到 merge。
- **Merge**：后台把多个小 segment 合并成大 segment，合并时才**物理删除**被标记删除的文档。
- 这正是 ES **写放大**的隐蔽来源（merge 重写）：详见 07 章和 SkyWalking 存储引擎章节。

> 💡 **面试答「ES 删除/更新为什么慢」**：删除是标记删除不立即释放空间，要等 segment merge；更新是先标记删再写新；大量删除/更新会让磁盘占用虚高，直到 merge 才回收。

---

## 五、读取流程（query then fetch）★

### 5.1 两阶段查询

ES 的一次搜索分两阶段：

```mermaid
graph TD
    subgraph Query阶段["Query 阶段"]
        direction TB
        Q1["coordinating 节点收到请求"]
        Q2["路由到每个相关分片（primary 或 replica 之一）"]
        Q3["每个分片本地查询，返回 [docId, score] 的 TopN"]
        Q4["coordinating 节点合并所有分片结果，重排序取最终 TopN"]
        Q1 --> Q2 --> Q3 --> Q4
    end

    subgraph Fetch阶段["Fetch 阶段"]
        direction TB
        F1["coordinating 节点按 docId 去对应分片"]
        F2["取出 _source（完整文档）"]
        F3["返回给客户端"]
        F1 --> F2 --> F3
    end
```

### 5.2 为什么要两阶段

- 如果一阶段就返回完整文档，每个分片要传回大量 _source 数据到协调节点，网络和内存浪费。
- 两阶段：先轻量传 docId+score 选出 TopN，再只 Fetch 这 N 条的完整文档，省网络。

### 5.3 分片选择（读负载均衡）

- 查询会 fan-out 到**每个相关分片**（主或副本之一）。
- 默认**轮询/随机**在 primary 和 replica 间选，实现读负载均衡。
- 副本越多，读吞吐越高（但要付写入和存储代价）。

### 5.4 ⚠️ fan-out 的代价

- 查询要打到**所有**含该索引数据的分片（默认每个分片都要查）。
- 分片越多，单次查询 fan-out 越多，协调开销越大，慢分片拖累整体（木桶效应）。
- 这解释了为什么「分片不是越多越好」。

---

## 六、相关性打分

### 6.1 TF-IDF -> BM25

- 搜索结果按**相关性分数**排序。打分模型演进：
  - **TF-IDF**：词频（TF，词在文档出现越多越相关）× 逆文档频率（IDF，词在所有文档越常见越不重要）。
  - **BM25**（默认，6.0+）：TF-IDF 的改进版，引入**饱和度**和**文档长度归一**：
    - 词频有上限（出现 100 次不比 10 次强太多），避免长文档刷分。
    - 短文档更相关时不会因为短就吃亏太多。

### 6.2 为什么要用 BM25 而非 TF-IDF

- TF-IDF 的问题：长文档天然词频高，得分虚高（不公平）；词频不饱和，堆关键词能刷分。
- BM25 让长文档不再靠长度占便宜，词频饱和防刷分，结果更符合直觉。这也是 ES 默认改用 BM25 的原因。

### 6.3 filter 不打分（重要优化）

- **query 上下文**：算相关性分，影响排序。
- **filter 上下文**：只判断「匹配/不匹配」，**不算分**，且**结果可缓存**。
- 能用 filter 就用 filter（如范围、精确匹配、时间过滤），更快且省缓存。

> 详见「07-查询与调优」。

---

## 七、常见面试题

1. **倒排索引是什么？由哪几部分组成？**
   倒排索引是「词 -> 文档列表」的结构，让按词搜文档 O(1)。由 Term Dictionary（词典，磁盘）、Term Index（FST，内存，加速定位）、Posting List（倒排表，docId+词频+位置，压缩存储）组成。

2. **ES 查词为什么快？FST 起什么作用？**
   先在内存的 FST 上走状态机定位 term，再少量磁盘 IO 读 Term Dictionary，拿到 Posting List。FST 是前缀压缩的有限状态机，极度压缩可放内存，避免了在大词典上二分查找的多次磁盘 IO。

3. **Doc Values 是什么？为什么需要它？**
   列式正排存储（docId -> 字段值）。倒排擅长「按词找文档」但不擅长排序聚合（要按文档取值），Doc Values 解决排序/聚合/脚本访问。代价是字段多存一份数据，是写放大来源之一，不需要排序聚合时可关闭。

4. **ES 写入一个文档的流程？refresh / flush / translog 分别是什么？**
   写 indexing buffer + 写 translog，主分片成功后转发副本。refresh（默认1s）把 buffer 生成内存 segment 使文档可搜（NRT）；flush 把 segment fsync 落盘并清 translog 保证持久化；translog 是 WAL，崩溃后恢复未 flush 的数据。refresh 让可搜，flush 才持久化。

5. **ES 为什么是近实时（NRT）？**
   写入后默认要等 1s refresh 生成 segment 才能被搜到，这 1s 间隔就是近实时的来源。可调大 refresh_interval 换写入性能（代价是可搜延迟变大）。

6. **ES 删除和更新为什么「慢」/不立即释放空间？**
   segment 不可变，删除是标记删除、更新是标记删+写新，数据要等后台 segment merge 才物理删除。大量删除/更新会让磁盘占用虚高，直到 merge 才回收。

7. **ES 一次搜索的 query then fetch 两阶段？**
   Query 阶段：协调节点把请求路由到各分片，各分片返回 docId+score 的 TopN，协调节点合并排序取最终 TopN。Fetch 阶段：按 docId 去分片取完整 _source 返回。两阶段省网络，避免每个分片传回完整文档。

8. **text 和 keyword 区别？**
   text 分词建倒排用于全文搜索（match），不能直接聚合排序；keyword 不分词整值存用于精确匹配（term）和聚合排序。常组合使用：text 主字段 + keyword 子字段。

9. **BM25 比 TF-IDF 好在哪？**
   BM25 引入词频饱和（避免堆关键词刷分）和文档长度归一（避免长文档靠长度占便宜），打分更符合直觉，是 ES 6.0+ 默认。

10. **写入分词和搜索分词不一致会怎样？**
    会搜不到。写入用 analyzer 分词建倒排，搜索也要把查询文本分词成相同 token 才能匹配。可单独配 search_analyzer，常见做法是写入用细粒度（ik_max_word）保召回、搜索用粗粒度（ik_smart）减噪音。

---

> 本章讲透读写原理。下一章「07-查询与调优」讲 Query DSL、聚合、深分页、调优和集群运维。
