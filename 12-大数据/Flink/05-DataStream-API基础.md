# 第5章 DataStream API 基础

> 本文基于《尚硅谷大数据技术之Flink（Java）》整理，面向 Java 后端面试。

---

## 5.1 核心概念

**执行环境**

创建执行环境三种方式：

| 方式 | 说明 | 场景 |
|---|---|---|
| `getExecutionEnvironment()` | **最常用**，智能判断本地/集群 | 生产推荐 |
| `createLocalEnvironment()` | 本地环境，可传并行度 | 本地调试 |
| `createRemoteEnvironment(host, port, jar)` | 指定 JM 主机端口和 jar | 远程提交 |

**执行模式**（Flink 1.12+ 流批一体）：

| 模式 | 说明 | 场景 |
|---|---|---|
| **STREAMING**（默认） | 每来一条触发计算输出 | 无界流 |
| **BATCH** | 全部处理完一次性输出 | 有界数据，更高效 |
| **AUTOMATIC** | 按数据源是否有界自动选 | 一般用不到 |

**触发执行**：`env.execute()`。⚠️ Flink 是**懒执行（lazy execution）**--`execute()` 前只是构建 JobGraph 未提交，调用后才真正提交 JobManager 运行。

**Source 源算子**

| 数据源 | API | 场景 |
|---|---|---|
| 集合 | `fromCollection` / `fromElements` | 测试（有界） |
| 文件 | `readTextFile(path)` | 批处理 |
| Socket | `socketTextStream` | 测试 |
| Kafka | `addSource(new FlinkKafkaConsumer(...))` | **生产首选** |
| 自定义 | 实现 `SourceFunction` | Flink 无预实现的系统 |

⚠️ `SourceFunction` 并行度只能为 1，要并行用 `ParallelSourceFunction`。
⚠️ **资料用老 API**：`SourceFunction`/`FlinkKafkaConsumer` 已 `@Deprecated`，Flink 1.15+ 推荐新 Source 接口（FLIP-27，`KafkaSource.builder()`）。

**支持的数据类型与 TypeInformation**

Flink 用 **TypeInformation** 统一描述类型，为每种生成专属序列化器。支持：基本类型、数组、Tuple(0~25)、Row、**POJO**、Scala 样例类、泛型（由 Kryo 序列化，效率低）。

**POJO 要求（面试重点）**：① 类 public 且独立（无非静态内部类）；② 有 public 无参构造；③ 字段 public 且非 final，或有符合 JavaBean 规范的 getter/setter。

⚠️ **Lambda + 泛型擦除陷阱**：Lambda 中返回 `Tuple2<String,Long>` 会被擦除，需 `.returns(Types.TUPLE(Types.STRING, Types.LONG))` 显式声明，否则按 Object 用低效 Kryo。

**Transformation 转换算子（重点）**

基本算子：`map`(1->1)、`filter`(1->0或1)、`flatMap`(1->0/1/N，最灵活)。

**keyBy（哈希分区原理）**：按 key 将流逻辑分区，相同 key 发往同一下游子任务，是聚合前必做操作。

分区原理：①对 key 求哈希；② `hashCode & Integer.MAX_VALUE` 保证非负；③对下游并行度取模。

⚠️ **资料勘误**：Flink 内部用 **MurmurHash3** 对 key 求哈希（非直接调用 `hashCode()`），但 POJO 仍需正确实现 `hashCode()/equals()`。

**keyBy 后得到 KeyedStream**（重点）：`keyBy` 返回 `KeyedStream<T,K>`（不是 DataStream），**不是转换算子**，只是逻辑分区；只有 KeyedStream 才能调用 sum/max/reduce 等聚合算子；让算子**状态按 key 隔离**。

**基本聚合算子（sum/max/maxBy/min/minBy）**

⚠️ **max vs maxBy（面试必考）**：

| 算子 | 行为 | 非聚合字段 |
|---|---|---|
| `max("字段")` | 只更新指定字段为当前最大值 | 跟随**当前累加器记录**的字段 |
| `maxBy("字段")` | 返回**包含字段最大值的整条记录** | 整条记录替换 |

⚠️ **资料勘误**：资料说「max() 其他字段保留最初第一个数据的值」**不准确**。准确：`max()` 非聚合字段保留当前累加器记录的字段，会随最大值刷新而更新。**记忆口诀**：max/min 只更新聚合字段，附带字段跟随当前最大值记录；maxBy/minBy 返回整条最大值对应的完整记录。

**reduce**：接收 KeyedStream，实现 `reduce(v1,v2)` 两条合并一条，底层维护累加器状态，比 sum/max 更通用。

**富函数 RichFunction（重点：生命周期）**

富函数比普通函数多的能力：①**生命周期方法 open()/close()**；② **getRuntimeContext()**（并行度、子任务索引、状态）。

⚠️ **生命周期调用规则（面试必考）**：

| 方法 | 调用次数 | 时机 |
|---|---|---|
| `open()` | **每个并行子任务一次** | 工作方法前 |
| `map()/filter()` 等工作方法 | **每条数据一次** | 数据到来时 |
| `close()` | **每个并行子任务一次** | 算子结束时 |

**经典场景**：在 map 里连数据库是反模式（每条建连接）。正确做法：open() 建连接 -> map() 复用 -> close() 关连接。

**物理分区（Physical Partitioning）**

⚠️ keyBy 是逻辑分区，物理分区返回的仍是 DataStream，只控制数据传输方式。

| 算子 | 行为 |
|---|---|
| `shuffle()` | 随机均匀分发到下游所有分区 |
| `rebalance()` | 轮询（Round-Robin）发到下游**所有**分区，跨 TM 网络传输 |
| `rescale()` | 轮询发到下游**部分**分区（仅本 TM 内），不跨节点 |
| `broadcast()` | 数据复制发到下游所有分区 |
| `global()` | 所有数据发到下游第一个子任务（慎用，易瓶颈） |
| `forward()` | 一一对应直通（上下游并行度同） |
| `partitionCustom()` | 自定义分区 |

⚠️ **rebalance vs rescale（面试必考）**：

| 维度 | rebalance | rescale |
|---|---|---|
| 覆盖范围 | 下游**所有**子任务 | 下游**部分**（本 TM 内） |
| 网络传输 | 跨 TM，有开销 | 仅本机 slot，无网络 |
| 场景 | 数据倾斜需全局重平衡 | 下游是上游整数倍、TM 多 slot |

通俗比喻：rebalance 是"每个发牌人给所有人发牌"；rescale 是"分小团体，只给本团体发牌"。

**Sink 输出算子**

⚠️ **重要原则**：不要在 map/filter 里随意读写外部系统（破坏 checkpoint 一致性）。用专用 Sink 算子。

| Sink | 关键类 | 一致性 |
|---|---|---|
| 文件 | `StreamingFileSink` | exactly-once |
| Kafka | `FlinkKafkaProducer`（TwoPhaseCommitSinkFunction） | exactly-once（两阶段提交） |
| Redis | `RedisSink` | at-least-once |
| ES | `ElasticsearchSink` | at-least-once |
| MySQL | `JdbcSink.sink(...)` | 支持批次写入 |

要点：Kafka 是 Flink 最佳搭档（双端连接、端到端 exactly-once）；自定义 Sink 用富函数版本（open 建连接、invoke 写、close 关）。⚠️ 资料用老 Sink API，Flink 1.15+ 推荐 `KafkaSink`/`FileSink` 等。

## 5.2 常见面试题

**Q1：keyBy 的分区原理？为什么说是逻辑分区？**
对 key 计算 MurmurHash3 哈希再对下游并行度取模，将相同 key 发往同一分区。称"逻辑分区"因：① 返回 KeyedStream 而非 DataStream；② 只保证相同 key 同分区，无法控制具体去哪、是否均匀；③ 物理分区只控制传输方式不改变数据。

**Q2：max 和 maxBy 的区别？**
`max("字段")` 只更新指定字段为最大值，非聚合字段跟随当前累加器记录；`maxBy("字段")` 直接返回包含最大值的整条记录。单字段场景结果相同，多字段 maxBy 更直观。

**Q3：rebalance 和 rescale 的区别？**
两者都是 Round-Robin 轮询。rebalance 发往下游**所有**子任务，跨 TM 有网络开销；rescale 只发往**本 TM 内的部分**子任务，不跨节点。下游是上游整数倍且 TM 多 slot 时 rescale 更高效；数据倾斜需全局重平衡用 rebalance。

**Q4：富函数的生命周期方法？调用时机？**
富函数有 `open()`/`close()`。open() 每个并行子任务只调一次，在工作方法前；close() 算子结束时每个子任务一次；工作方法每条数据一次。典型：open 建连接，map 复用，close 关连接。还提供 getRuntimeContext()。

**Q5：Flink 程序为什么要调用 execute()？**
Flink 是懒执行，main 里代码只构建数据流图（DAG），未提交。必须显式 `env.execute()` 才把 JobGraph 提交到 JobManager 触发计算。

**Q6：POJO 需要满足什么条件？**
① 类 public 且独立；② 有 public 无参构造；③ 字段 public 且非 final，或有 JavaBean 规范的 getter/setter。POJO 可在 keyBy 中用字段名做 key，性能好。

---

## 资料勘误与重点提醒

1. ⚠️ **keyBy 不是算子**：是数据分区操作，返回 KeyedStream，不能设并行度，会断开算子链。
2. ⚠️ **max/min 非聚合字段行为**：资料说"保留最初第一个数据的值"不准确。准确为保留当前累加器记录的字段，会随最大值刷新而更新。
3. ⚠️ **keyBy 哈希算法**：资料说"通过 key 的 hashCode() 取模"。实际 Flink 用 MurmurHash3 求哈希再取模，但 POJO 仍需正确实现 hashCode()/equals()。
4. ⚠️ **新老 Source/Sink API**：资料用 SourceFunction/FlinkKafkaConsumer/FlinkKafkaProducer 等已 @Deprecated API。Flink 1.15+ 推荐新接口（KafkaSource/KafkaSink 等）。
