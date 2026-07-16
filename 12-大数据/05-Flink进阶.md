# Flink 进阶（第9-12章）

> 本文基于《尚硅谷大数据技术之Flink（Java）》整理，面向 Java 后端面试。力求通俗易懂，对原资料中表述不准或过时之处已就地用 ⚠️ 标注修正。
>
> 本章覆盖 Flink 面试最核心的两大重灾区--**状态编程**与**容错机制**，以及上层声明式 API（Table API/SQL）和复杂事件处理（CEP）。

---

## 目录

1. [状态编程](#9-状态编程)⭐
2. [容错机制](#10-容错机制)⭐⭐
3. [Table API 和 SQL](#11-table-api-和-sql)
4. [Flink CEP](#12-flink-cep)
5. [补充：性能与调优要点](#13-补充性能与调优要点) ⭐

---

## 9. 状态编程 ⭐

### 9.1 核心概念

**为什么需要状态**

Flink 的本质是「有状态的流式计算」。流式数据连续到来，很多计算不仅依赖当前数据，还需要历史数据：求和要保存累加值、窗口要缓存窗口内数据、CEP 要保存前序事件。这些「任务在本地维护、用来计算输出结果的数据」就是状态。

如果用外部数据库存状态，频繁读写会成为性能瓶颈；所以 Flink 把状态直接放在内存（或 RocksDB）中，由 Flink Runtime 统一托管：负责存储访问、故障恢复、并行度变更时的重组分配。

> 通俗类比：状态就像 RPG 游戏的存档，算子每处理一条数据相当于推进一段剧情，状态就是当前进度；宕机后从存档继续即可。

**有状态 vs 无状态算子**

- 无状态算子：map / filter / flatMap，只看当前事件直接转换输出。
- 有状态算子：聚合（sum/min/max）、窗口、CEP、ProcessFunction、富函数中自定义的状态。

**状态的分类**

**(1) 托管状态（Managed State） vs 原始状态（Raw State）**

| 维度 | 托管状态 | 原始状态 |
|---|---|---|
| 管理 | Flink Runtime 统一托管 | 用户自己实现 |
| 序列化/恢复 | 自动 | 自己实现，Flink 只当 byte[] |
| 推荐度 | 推荐（绝大多数场景够用） | 不推荐，仅托管状态无法满足时用 |

实际开发几乎都用托管状态。

**(2) Keyed State vs Operator State（重点⚠️）**

| 维度 | Keyed State（按键分区状态） | Operator State（算子状态） |
|---|---|---|
| 作用范围 | 按 key 隔离，每个 key 一份状态 | 当前算子并行实例一份，与 key 无关 |
| 前提 | 必须先 keyBy，定义在 KeyedStream 上 | 任意算子均可（需实现 CheckpointedFunction） |
| 获取方式 | 在富函数 RuntimeContext 中获取 | 在 OperatorStateStore 中获取 |
| 并行度调整重组 | 按「键组 key group」按 key 哈希重新分配 | 按列表轮询 / 联合广播重分配 |
| 典型场景 | 聚合、窗口、按 key 的自定义逻辑 | Kafka Source 的 offset、Sink 缓冲、广播规则 |
| 支持类型 | ValueState / ListState / MapState / ReducingState / AggregatingState | ListState / UnionListState / BroadcastState |

> 通俗类比：Keyed State 像「按用户分账户记账」，每个用户的余额互不干扰；Operator State 像「整个收银台共用的钱箱」，凡是被分到这个收银台的数据都共用同一个。

⚠️ 资料原文有 `ReducintState` 笔误，正确为 `ReducingState`。

**Keyed State 支持的五种类型**

| 类型 | 说明 | 典型用途 | 关键方法 |
|---|---|---|---|
| ValueState<T> | 单值状态 | 计数器、最近一次行为 | value() / update(T) |
| ListState<T> | 列表状态 | 缓存窗口数据、双流 Join | add(T) / get() / update(List) |
| MapState<UK,UV> | 映射状态 | 按子维度统计（如 url->pv） | put / get / entries / remove |
| ReducingState<T> | 归约状态 | 输入与状态同类型的累加 | add(T) 自动用 ReduceFunction 归约 |
| AggregatingState<IN,OUT> | 聚合状态 | 求平均、自定义累加器，输入/输出类型可不同 | add(IN) 自动用 AggregateFunction 聚合 |

ReducingState 和 AggregatingState 的区别（常考）：ReducingState 输入与状态同类型（如求和 sum）；AggregatingState 引入累加器 ACC，输入 IN 与输出 OUT 可不同类型，更灵活（如求平均 = sum/count）。

**状态 TTL（生存时间）**

为防止状态无限增长耗尽存储，可给状态配置 TTL。状态创建时设置 `失效时间 = 当前时间 + TTL`，按配置策略更新失效时间，清除时判断是否过期。

- `setUpdateType`：`OnCreateAndWrite`（默认，仅写更新）/ `OnReadAndWrite`（读也更新，延长生存）。
- `setStateVisibility`：`NeverReturnExpired`（默认，过期即视为不存在）/ `ReturnExpiredIfNotCleanedUp`（还没真正清理就仍可读）。

⚠️ 重要限制：**TTL 当前只支持处理时间（Processing Time）**，不支持事件时间。另外 ListState / MapState 等集合状态是 per-entry 失效，即列表中每个元素有自己的失效时间，而不是整列表一起清理。

**Operator State 的三种类型**

- **ListState（列表状态）**：并行度变更时采用「平均分割重组 even-split redistribution」，所有元素收集起来后轮询（round-robin）平均分发给新并行子任务。
- **UnionListState（联合列表状态）**：并行度变更时把完整列表广播给所有新子任务（「联合重组 union redistribution」），由各子任务自行挑选。⚠️ 元素过多时不宜使用，开销大。
- **BroadcastState（广播状态）**：每个并行子任务持有相同的状态副本，并行度扩展直接复制、缩小直接丢弃。底层是 MapState 结构，只能用于广播连接流（BroadcastConnectedStream），典型用于「动态规则 / 动态配置」场景。

**为什么要用 List 而不是 Value？** 因为算子状态没有 key group 这样按 key 重组的天然单元，所以定义为列表，让每个元素成为可重分配的最小粒度。

**状态后端（State Backend）**

状态后端负责两件事：① 本地状态存储管理；② 将检查点写入远程持久化存储。

| 后端 | 本地状态位置 | 性能 | 容量 | 增量检查点 | 适用场景 |
|---|---|---|---|---|---|
| HashMapStateBackend | TaskManager JVM 堆内存 | 最快 | 受限于内存 | ❌ | 状态不大、追求低延迟 |
| EmbeddedRocksDBStateBackend | TaskManager 本地磁盘（RocksDB） | 慢一个数量级（需序列化/反序列化） | 仅受磁盘限制 | ✅（唯一支持） | 海量状态、长窗口、大 KV 状态 |

⚠️ **资料原文表述不准确**：「HashMapStateBackend 适用于具有大状态、长窗口、大键值状态的作业」--这与 HashMap 受限于内存的事实矛盾。**HashMapStateBackend 适合中小状态、追求性能的场景；大状态应选 RocksDB**。资料该段表述前后冲突，已修正。

默认状态后端由 `flink-conf.yaml` 中 `state.backend` 指定（hashmap / rocksdb），默认是 hashmap；可在代码中 `env.setStateBackend(...)` 为单个作业覆盖。

### 9.2 常见面试题

**Q1：Keyed State 和 Operator State 的区别？**
- 范围：Keyed State 按 key 隔离，每个 key 一份；Operator State 是算子并行实例共享一份，与 key 无关。
- 前提：Keyed State 必须先 keyBy；Operator State 无需 keyBy，但要实现 CheckpointedFunction。
- 重组方式：Keyed State 按 key group 按 key 哈希重新分配；Operator State 用 even-split（ListState）/ union（UnionListState）/ 复制（BroadcastState）。
- 场景：Keyed State 用于聚合、窗口、按 key 的自定义逻辑；Operator State 多用于 Source（Kafka offset）、Sink 缓冲、广播规则。

**Q2：Flink 有哪些 Keyed State 类型？分别在什么场景用？**
ValueState（计数器/最近一次值）、ListState（缓存窗口内元素、双流 Join）、MapState（按子维度统计，如 url->pv）、ReducingState（输入输出同类型的累加，如求和）、AggregatingState（用累加器 ACC，输入输出可不同类型，如求平均）。获取方式都是通过 StateDescriptor 注册，在 RichFunction 的 open() 中通过 RuntimeContext 获取。

**Q3：ReducingState 和 AggregatingState 有何区别？**
ReducingState 用 ReduceFunction，输入数据类型和状态类型必须相同；AggregatingState 用 AggregateFunction，引入中间累加器 ACC，输入 IN、累加器 ACC、输出 OUT 三者类型可以不同，灵活性更高。

**Q4：状态 TTL 是什么？有哪些注意点？**
为防止状态无限增长，给状态配置生存时间，过期后清除。配置项包括 updateType（何时刷新失效时间）和 stateVisibility（过期后是否可读）。⚠️ 注意：**TTL 只支持 Processing Time，不支持 Event Time**；ListState/MapState 是 per-entry 失效。

**Q5：HashMapStateBackend 和 EmbeddedRocksDBStateBackend 如何选择？**
HashMap 在 JVM 堆内存，读写最快，但受限于内存容量，状态大易 OOM，不支持增量检查点；RocksDB 数据落本地磁盘，可承载海量状态，唯一支持增量检查点，但读写需序列化/反序列化，平均慢一个数量级。**小状态追求性能选 HashMap，大状态选 RocksDB**。

**Q6：广播状态的使用场景和特点？**
动态配置 / 动态规则场景（如规则引擎实时下发规则）。特点：① 所有并行子任务持有相同状态副本；② 底层是 MapState，必须基于广播流 BroadcastStream 创建，只能用于 BroadcastConnectedStream；③ BroadcastProcessFunction 中 processBroadcastElement() 可写状态，processElement() 只读状态；④ 并行度变更时直接复制或丢弃。

---

## 10. 容错机制 ⭐⭐

### 10.1 核心概念

**检查点（Checkpoint）保存什么、何时保存**

**保存内容**：所有算子任务状态在某一时刻的快照（一份拷贝），加上 Source 算子读取数据的偏移量（作为 Operator State）。本质是「存档」。

**保存时机**：周期性触发（如每 1 秒），关键是要让**所有任务都恰好处理完同一个输入数据后**对状态做快照。这样既避免保存半截状态，又使检查点具备「事务」性质：要么所有任务都保存好，要么全没保存。

> 通俗类比：Checkpoint Barrier 就像「在数据流中插入一道栅栏，让数据流过栅栏时给所有算子同时拍照快照」。栅栏之前的数据引发的状态变更进入当前检查点；栅栏之后的数据进入下一个检查点。

⚠️ 重要修正：资料称「检查点存储到 JobManager 堆内存或文件系统」--JobManager 内存存储只用于测试或小状态，**生产环境必须用 HDFS/S3 等分布式文件系统**，且要做高可用。

**从检查点恢复状态的流程**

1. **重启应用**：所有任务状态清空。
2. **读取检查点，重置状态**：从最近成功的检查点读出每个算子的状态快照填充回去，状态回退到检查点时刻。
3. **重放数据**：Source 任务向数据源重新提交检查点保存的偏移量，重新读取检查点之后的数据（数据源必须支持重放，如 Kafka）。
4. **继续处理**：从重放的数据接着处理，追上故障前的进度，就像没发生过故障。

**检查点算法（Chandy-Lamport 分布式快照算法）**

Flink 采用 Chandy-Lamport 算法的变体--「异步分界线快照 Asynchronous Barrier Snapshotting」。

**核心两原则**：
- 上游向多个并行下游传递 barrier 时：**广播**给所有下游。
- 多个上游向同一下游传递 barrier 时：下游执行 **barrier 对齐**--必须等所有上游分区的 barrier 都到齐才能做状态快照。

**算法流程**（以并行度=2 的 WordCount 为例）：
1. JobManager 的 Checkpoint Coordinator 周期性发出带检查点 ID 的指令，Source 任务保存偏移量、注入 barrier 到数据流。
2. barrier 像普通数据一样向下游传递；每个算子收到 barrier 后对当前状态做快照存到状态后端，再把 barrier 继续往下传。
3. 下游算子若有多条上游输入（如 keyBy 后多分区），需等所有上游 barrier 都到齐才快照--这就是 barrier 对齐。
4. 所有任务快照完成 -> JobManager 确认该检查点成功保存。
5. 在 barrier 对齐等待期间，先到 barrier 的分区若又来新数据（属于下一检查点），需缓存不能处理；后到 barrier 的分区若来数据（属于当前检查点），正常处理。

**Barrier 对齐 vs 非对齐检查点（Unaligned Checkpoint）⚠️**

**Barrier 对齐（Aligned Checkpoint，默认）**：
- 必须等所有上游 barrier 都到齐才能做快照。
- 问题：背压严重时，下游缓存大量 in-flight data，检查点可能很久才完成，甚至超时失败。
- 保证 at-least-once（对齐了就不丢数据）和 exactly-once（重复数据处理被状态回滚抵消）。

**非对齐检查点（Unaligned Checkpoint，Flink 1.11+）**：
- 收到任意一个上游的 barrier 就立即开始快照，把未处理的 in-flight data 也一起存入检查点。
- 大幅减少背压场景下检查点保存时间。
- 限制：要求 CheckpointingMode = EXACTLY_ONCE 且并发检查点数 = 1。
- 代价：检查点体积更大（多了 in-flight data）。

> 通俗类比：Barrier 对齐像「等所有考生答完再一起拍毕业照」，慢的考生拖累大家；非对齐像「每个考生答完就拍单人照，把没写完的卷子一起存档」。

**检查点配置**

- `enableCheckpointing(interval)`：启用检查点，指定周期。
- `CheckpointingMode`：EXACTLY_ONCE（默认） / AT_LEAST_ONCE。
- `checkpointTimeout`：超时时间，超时丢弃。
- `minPauseBetweenCheckpoints`：上一个检查点完成后，至少等多久才能开始下一个，给数据处理留间隙（设置后 maxConcurrentCheckpoints 强制为 1）。
- `maxConcurrentCheckpoints`：并发检查点最大数。
- `enableExternalizedCheckpoints`：开启外部持久化，作业取消时清理策略 DELETE_ON_CANCELLATION / RETAIN_ON_CANCELLATION。
- `enableUnalignedCheckpoints`：开启非对齐检查点。
- `setCheckpointStorage`：检查点存储位置，生产用 HDFS/S3。

**保存点（Savepoint）vs 检查点（Checkpoint）⚠️ 重点**

| 维度 | Checkpoint | Savepoint |
|---|---|---|
| 触发方式 | Flink 自动周期触发 | 用户手动触发 |
| 用途 | 故障恢复（容错） | 有计划的手动备份、版本升级、应用更新、调整并行度、暂停应用 |
| 格式 | 紧凑、可能依赖特定状态后端 | 标准格式，跨版本/后端兼容 |
| 自动清理 | 默认保留最近几个 | 不会自动清理，需手工管理 |
| 算子 ID | 自动生成 | 建议用 .uid() 手动指定，保证兼容 |

**Savepoint 适用场景**：版本归档、Flink 版本升级、应用更新（前提是状态拓扑和数据类型不变）、调整并行度、暂停应用释放资源。

⚠️ 重要提醒：**为算子手动指定 .uid() 非常关键**。Flink 默认会基于算子结构自动生成 ID，一旦程序改了（增删算子），自动 ID 就会变，导致从 Savepoint 恢复时找不到对应状态。所以生产代码每个有状态的算子必须显式 `.uid("xxx")`。

**状态一致性级别**

| 级别 | 含义 | 实现 |
|---|---|---|
| AT-MOST-ONCE（最多一次） | 故障后直接重启，丢数据也不重放，最多处理一次 | 无任何保证，只追求快时可用 |
| AT-LEAST-ONCE（至少一次） | 数据不丢，可能重复处理 | 数据源可重放（如 Kafka 重置 offset）+ 检查点 |
| EXACTLY-ONCE（精确一次） | 数据不丢且只处理一次 | 检查点 + Chandy-Lamport 快照算法保证状态回滚一致 |

⚠️ **EXACTLY-ONCE 的准确含义**：不是说某条数据只被 Flink 处理一次（重放后必然处理多次），而是「数据对状态和输出结果的影响只体现一次」。检查点保证重放数据引起的状态改变不会进入已保存的检查点，状态回滚后最终只计入一次。

**端到端 EXACTLY-ONCE**

端到端一致性 = 数据源 + 流处理器（Flink 内部）+ 外部存储 三者一致性取最弱一环。Flink 内部靠检查点可做到 exactly-once，所以要端到端 exactly-once 还需两端配合：

**(1) 输入端**：数据源必须可重放，即支持重置读取偏移量。Kafka 是最佳选择：FlinkKafkaConsumer 把 offset 作为算子状态存入检查点，故障恢复时重置 offset 重放。socket 文本流不支持重放，只能 at-most-once。

**(2) 输出端**：两种保证方式：

**① 幂等写入（Idempotent Write）**
重复写入不影响最终结果，如 Redis set、MySQL UPDATE。优点：实现简单，对存储系统要求低；缺点：故障恢复瞬间会出现短暂不一致（结果「跳回」再「重播」），但最终一致。

**② 事务写入（Transactional Write）**--真正严格的 exactly-once

思想：把写入操作绑定到一个与检查点绑定的事务中。两种实现：

- **预写日志（WAL）**：先把结果数据作为日志状态保存，检查点保存时一并持久化，收到检查点完成通知后一次性批量写入外部系统。优点：不要求外部系统支持事务；缺点：批处理性能差，且确认信息丢失会导致重复写入。Flink 提供 GenericWriteAheadSink 模板类。
- **两阶段提交（2PC）**：① Sink 任务收到 barrier 时开启事务；② 之后所有数据通过该事务预写入（数据已写入但不可见）；③ 收到 JobManager 检查点完成通知后正式提交事务，数据才对外可见。故障则事务回滚，预提交数据被撤销。

**2PC 对外部系统的要求**：
- 必须支持事务；
- 检查点间隔内能开启事务并接收写入；
- 提交前事务须保持「等待提交」状态（防止超时被关闭）；
- Sink 任务能恢复失败的事务；
- 提交事务必须幂等。

⚠️ 资料原文 ACID 中"C"写成 Correspondence，这是错误的。**ACID 中的 C 是 Consistency（一致性）**，非 Correspondence。

**Flink + Kafka 的 EXACTLY-ONCE 实现（高频考点）**

完整链路：Kafka(Source) -> Flink 内部 -> Kafka(Sink)。

**具体流程**（结合检查点）：
1. JobManager 触发检查点 -> Source 任务注入 barrier，并把当前 Kafka offset 作为算子状态存入检查点。
2. barrier 沿算子链传递，各算子收到后做状态快照。
3. Sink 任务（FlinkKafkaProducer 实现 TwoPhaseCommitSinkFunction）收到 barrier 时**开启 Kafka 事务**，之后数据通过该事务预写入 Kafka，标记为「未确认 uncommitted」。
4. 所有算子快照完成 -> JobManager 确认检查点成功 -> 通知 Sink -> Sink **正式提交事务**，数据转为「已确认 committed」。
5. 任何中间失败 -> 从上次检查点恢复 -> 未提交事务回滚 -> 预提交数据被撤销。

**必需配置**：
1. 启用检查点。
2. FlinkKafkaProducer 构造函数传 `Semantic.EXACTLY_ONCE`。⚠️ 注意：**新版本 Flink（1.15+）已用 KafkaSink 取代 FlinkKafkaProducer**，配置方式改为 `KafkaSink.builder().setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)`，并需指定事务前缀 `setTransactionalIdPrefix`。
3. **下游 Kafka 消费者隔离级别设为 read_committed**（默认是 read_uncommitted，会读到未提交数据，使事务保证失效）。
4. **事务超时配置**：Flink 的 `transaction.timeout.ms`（默认 1 小时）必须 ≤ Kafka 的 `transaction.max.timeout.ms`（默认 15 分钟）。⚠️ 否则会出现 Kafka 已超时丢弃预提交数据、Flink 还在等的情况，故障恢复后这部分数据真丢。

### 10.2 常见面试题

**Q1：Checkpoint 保存什么？何时保存？**
保存所有算子任务状态的快照 + Source 算子的数据偏移量。周期性触发，关键是所有任务都处理完同一个输入数据后做快照，保证快照的事务性。

**Q2：Flink 检查点算法原理（Chandy-Lamport）？**
采用异步分界线快照算法。JobManager 触发后，Source 注入带 ID 的 barrier 到数据流；barrier 像普通数据向下游传递；每个算子收到 barrier 后对状态做快照；上游向多下游广播 barrier，多上游向同一下游时下游做 barrier 对齐（等所有上游 barrier 到齐才快照）。整个快照过程不暂停数据处理，是异步的。

**Q3：什么是 Barrier 对齐？有什么问题？非对齐检查点如何解决？**
Barrier 对齐指多上游输入的算子，要等所有上游分区的 barrier 都到齐才能做状态快照；等待期间先到 barrier 的分区若又来新数据（属于下一检查点）需缓存。问题：背压严重时缓存大量 in-flight data，检查点可能超时失败。非对齐检查点（Flink 1.11+）直接保存 in-flight data 到检查点，收到任一 barrier 就开始快照，极大降低背压下的检查点延迟，但检查点体积更大，且要求 exactly-once 模式和并发检查点数为 1。

**Q4：Checkpoint 和 Savepoint 的区别？（高频⚠️）**
- 触发：Checkpoint 自动周期触发，Savepoint 手动触发。
- 用途：Checkpoint 用于故障恢复，Savepoint 用于版本升级、应用更新、调整并行度、暂停应用。
- 格式：Savepoint 是标准格式可跨版本/后端兼容，Checkpoint 格式可能依赖特定状态后端。
- Savepoint 建议手动给每个有状态算子 .uid()，保证恢复时 ID 匹配。

**Q5：什么是端到端 exactly-once？如何保证？**
端到端 = 数据源 + Flink 内部 + 外部存储 三者一致性取最弱一环。Flink 内部靠检查点保证；输入端要求数据源可重放（如 Kafka 重置 offset）；输出端要么幂等写入（最终一致），要么事务写入（严格 exactly-once）。事务写入中 2PC 最严格：Sink 收到 barrier 开启事务、预提交；检查点完成后正式提交；失败则回滚。

**Q6：Flink + Kafka 如何实现 exactly-once？**
- 内部：Flink 检查点机制保证状态一致性。
- 输入端：FlinkKafkaConsumer 把 offset 作为算子状态存入检查点，故障恢复时重置 offset 重放。
- 输出端：FlinkKafkaProducer 实现 TwoPhaseCommitSinkFunction，收到 barrier 开启 Kafka 事务预提交，检查点完成后正式提交。
- 配置：必须启用检查点；Kafka Producer 传 EXACTLY_ONCE 语义；下游消费者隔离级别设 read_committed；Flink 事务超时 ≤ Kafka 事务最大超时。

**Q7：三种状态一致性级别 at-most-once / at-least-once / exactly-once 的区别？**
at-most-once：故障直接重启，丢数据也不重放，无保证。at-least-once：数据不丢可能重复，靠可重放数据源+检查点实现。exactly-once：数据不丢且只处理一次（指对状态/输出的影响只一次），靠检查点+Chandy-Lamport 快照实现，最难。

---

## 11. Table API 和 SQL

### 11.1 核心概念

**定位与层级**

Table API 和 SQL 是 Flink 顶层"声明式"应用 API，位于 DataStream API 之上：

- **Table API**：内嵌在 Java/Scala 中的 DSL，用 `$("字段")` 链式调用。
- **SQL**：基于 Apache Calcite 实现，几乎兼容标准 SQL 语法。

两者底层统一，查询同一张表结果一致，可混合使用。Flink 是批流统一的处理框架，Table API/SQL 对批和流都适用。

> ⚠️ 资料基于 Flink 1.13，称 Table API/SQL"不算稳定、接口还在调整"。当前 Flink 版本（1.17+）SQL 已相当成熟，是 Flink 主推的应用入口；Old Planner 已移除，Blink Planner 成为唯一 planner，无需再通过 `useBlinkPlanner()` 指定。

**基本程序架构**

核心是 **TableEnvironment**（表环境），它负责：① 注册 Catalog 和表；② 执行 SQL 查询；③ 注册 UDF；④ DataStream 与 Table 的互转。

程序架构 = 创建表环境 -> CREATE TABLE 注册输入/输出表（WITH 指定 connector）-> sqlQuery 查询 -> executeInsert 输出。可完全脱离 DataStream API。

**创建表：连接器表 vs 虚拟表**

- **连接器表（Connector Tables）**：通过 `CREATE TABLE ... WITH ('connector' = ...)` 连外部系统（Kafka、文件、JDBC、ES、HBase、Hive 等）。早期 TableSource/TableSink 已被弃用，统一用 connector。
- **虚拟表（Virtual Tables）**：通过 `createTemporaryView("name", table)` 把中间 Table 注册到环境中，类似视图。不实际存储数据，引用时把查询语句嵌入。

**表和流的转换 ⚠️ 重点**

**Table -> DataStream**（两种方式）：
- `toDataStream(table)`：仅追加流（Insert-Only）场景。如果表上有聚合（产生 Update）会抛 `TableException`。
- `toChangelogStream(table)`：更新日志流（Changelog Stream），用 `+I`/`-U`/`+U` 等 RowKind 编码增删改。

> ⚠️ 资料原文示例代码 `tableEnv.toDataStream(urlCountTable).print();` 与正文说明矛盾--urlCountTable 是分组聚合结果（更新查询），应当调用 `toChangelogStream()`，此处为资料笔误。

**DataStream -> Table**：
- `fromDataStream(stream)`：基础转换。
- `createTemporaryView("name", stream)`：直接注册为虚拟表。
- `fromChangelogStream(stream)`：将已带 RowKind 的 Row 流转为表。

**动态表与持续查询 ⚠️ 核心原理**

关系型表面向"有界、固定数据集"做批处理；而流处理是"无界、持续到来"的数据。Flink 引入两个核心概念打通二者：

**动态表（Dynamic Table）**：随时间不断变化的表。流中每条数据到来 = 对表的一次 Insert 操作。借鉴了关系库中"物化视图"的思想。

**持续查询（Continuous Query）**：对动态表的查询永不停止，每来一条数据就更新结果，结果本身也是一个动态表。

**三步闭环**（重要）：
```
流(Stream) -> ① 转换为 -> 动态表(Dynamic Table)
            -> ② 持续查询(Continuous Query) -> 新的动态表
            -> ③ 转换回 -> 流(Stream)
```

**两类持续查询**：
- **更新查询（Update Query）**：结果表中有 UPDATE 操作。例如 `SELECT user, COUNT(url) FROM EventTable GROUP BY user`，count 会随新数据累加。结果转流必须用 `toChangelogStream()`。
- **追加查询（Append Query）**：结果表只有 INSERT。例如简单过滤 `WHERE user='Alice'`，或者**窗口聚合**（窗口关闭时一次性输出，不会更新之前结果）。结果转流可用 `toDataStream()`。

> 关键规律：**不是所有聚合都是更新查询**。窗口聚合虽然用了聚合函数，但因为每个窗口只输出一次，结果不会更新，是 Append 查询。判断标准是"结果表中的行是否会被 UPDATE"。

**查询限制**：状态大小（分组聚合 key 持续增多导致状态无限增长，可配 `table.exec.state.ttl` TTL）、更新计算复杂度（如 `RANK()` 每来一条数据要全表重排序，代价巨大）。

**动态表转流的三种编码 ⚠️ 重点**

| 编码方式 | 消息类型 | 适用条件 |
|---|---|---|
| **仅追加流（Append-only）** | 只有 Insert | 表仅有 Insert 操作 |
| **撤回流（Retract）** | add（+）/retract（-） | Insert->add；Delete->retract；Update->先 retract 旧值再 add 新值（两条消息） |
| **Upsert 流** | upsert / delete | 表必须有唯一键（key）。Insert 和 Update 都编码为 upsert（单条消息），Delete 为 delete |

关键点：
- 转 DataStream 时只支持 Append 和 Retract（因为 DataStream 没有 key 概念）。`toChangelogStream()` 得到的就是 Retract 流。
- 写外部系统时三种编码都支持，由外部系统特性决定。例如 Kafka（普通）只支持 append；Upsert Kafka 要求声明 PRIMARY KEY；JDBC/ES/HBase 有主键时用 Upsert 模式。

**时间属性**

时间属性是表 schema 的一部分，类型为 TIMESTAMP，定义后可在窗口等操作中引用。

**事件时间**（最常用）：
- DDL 中用 `WATERMARK FOR ts AS ts - INTERVAL '5' SECOND` 定义。
- 数据流转表时用 `$("ts").rowtime()` 指定。
- 类型可为 `TIMESTAMP` 或 `TIMESTAMP_LTZ`（带本地时区，适合长整型毫秒数）。

**处理时间**（系统时间）：
- DDL 中 `ts AS PROCTIME()`（计算列形式）。
- 数据流转表时 `$("ts").proctime()`，必须放在字段列表最后。

**窗口**

- **老版本（1.12 前）：分组窗口（Group Window）**，`TUMBLE()`/`HOP()`/`SESSION()` 直接放在 GROUP BY 中。已弃用。
- **新版本（1.13+）：窗口表值函数（Windowing TVF）**，把窗口放在 FROM 子句，返回带 `window_start`/`window_end`/`window_time` 三列的扩展表。

主要 TVF：
- `TUMBLE(TABLE t, DESCRIPTOR(ts), INTERVAL '1' HOUR)`：滚动窗口。
- `HOP(TABLE t, DESCRIPTOR(ts), INTERVAL '5' MIN, INTERVAL '1' HOUR)`：滑动窗口。注意第三参数是 slide（步长），第四是 size。
- `CUMULATE(TABLE t, DESCRIPTOR(ts), INTERVAL '1' HOUR, INTERVAL '1' DAY)`：累积窗口，统计周期内多次输出且累加。适合"按天统计 PV，每小时输出一次当日累计值"。
- `SESSION`：会话窗口。

> ⚠️ 资料称"会话窗口目前尚未完全支持"。新版 Flink（1.16+）已支持 Session Window TVF。

**聚合查询**

- **分组聚合**：`GROUP BY user`，更新查询，转流用 toChangelogStream。对应 DataStream 的 keyBy + 聚合。
- **窗口聚合**：配合 TVF，`GROUP BY user, window_start, window_end`，追加查询。
- **开窗（Over）聚合**：`OVER (PARTITION BY ... ORDER BY ts RANGE BETWEEN ...)`，每行都算一次。Flink 中 ORDER BY 只能是时间属性升序，上界只能 CURRENT ROW。
- **Top N**：通过 `ROW_NUMBER() OVER (...) AS row_num` + 外层 `WHERE row_num <= N` 实现。普通 Top N 是更新查询；窗口 Top N 是追加查询。

**联结查询**

- **常规联结（Regular Join）**：标准 INNER/LEFT/RIGHT/FULL JOIN，仅支持等值条件。两侧任何变更都更新结果，是 Update Query。
- **间隔联结（Interval Join）**：加时间间隔约束，只支持 append-only 表。
- **时间联结（Temporal Join）**：针对版本表的联结，按数据发生时间找当时的版本。

**函数**

**系统函数**：标量函数（一行输入一值输出，如 UPPER）、聚合函数（多行输入一值，如 COUNT/ROW_NUMBER）。

**UDF（自定义函数）** 四类：

| 类型 | 输入->输出 | 抽象类 | 核心方法 |
|---|---|---|---|
| 标量函数 | 一标量->一标量 | ScalarFunction | eval() |
| 表函数 | 一标量->多行 | TableFunction | eval() + collect() |
| 聚合函数 | 多行->一标量 | AggregateFunction | createAccumulator/accumulate/getValue |
| 表聚合函数 | 多行->多行 | TableAggregateFunction | createAccumulator/accumulate/emitValue |

**连接器（Connector）**：Kafka（append-only）/ Upsert Kafka（要求 PRIMARY KEY）/ 文件系统 / JDBC（有主键 Upsert、无主键 Append）/ ES（仅 Sink）/ HBase（始终 Upsert）/ Hive（HiveCatalog 持久化元数据）。

### 11.2 常见面试题

**Q1：Flink Table API/SQL 中"动态表"和"持续查询"是什么？为什么需要它们？**
流是无界数据，关系型表是有界数据集，二者天生"八字不合"。动态表是会随时间变化的表--流中每条数据到来视作对表的 Insert。持续查询是针对动态表永不停止的查询，每来一条数据更新结果，结果也是动态表。三者关系是：流 -> 动态表 -> 持续查询 -> 新动态表 -> 流。这套机制让用户可以用熟悉的 SQL 写流处理。

**Q2：什么是更新查询和追加查询？怎么判断？**
判断标准是结果表中的数据是否会被 UPDATE。
- 更新查询：如 `GROUP BY user` + `COUNT`，新数据让已有行的 count 累加 -> 有 UPDATE。
- 追加查询：如简单 WHERE 过滤；或窗口聚合（窗口关闭时一次性输出，不更新历史行）-> 只有 INSERT。
更新查询的结果必须用 `toChangelogStream()` 转 DataStream；追加查询可直接 `toDataStream()`。**注意：聚合 ≠ 更新查询**，窗口聚合就是追加查询。

**Q3：动态表转流有哪几种编码方式？有什么区别？**
三种：Append-only 流（仅 Insert）；Retract 流（Insert->add，Delete->retract，Update->两条消息：先 retract 旧值再 add 新值）；Upsert 流（必须有唯一 key，Insert/Update 都编码为单条 upsert 消息，Delete 编码为 delete）。转 DataStream 只支持前两种；写外部系统时三种都支持，Upsert Kafka/JDBC/ES 等需要声明主键。

**Q4：Table API 和 DataStream API 如何选择？**
- **优先 SQL/Table API**：业务以统计聚合、多表 join 为主，对延迟要求一般的场景。开发效率高，SQL 易维护、易迁移，有优化器自动优化。Flink 当前主推方向。
- **用 DataStream API**：需要精细控制状态、定时器、时间语义、复杂状态机、CoProcessFunction 等底层能力；或 CEP；或对延迟极敏感、需要侧输出流、自定义 Watermark 策略等。
- **混合使用**：通过 `fromDataStream` / `toDataStream` / `toChangelogStream` 在两者间切换。

**Q5：窗口 TVF 相比老版分组窗口有什么改进？**
更符合 SQL 标准（窗口作为表出现在 FROM 后）；性能优化更好；功能更强（支持窗口 Top-N、窗口 Join、窗口 deduplication 等）；返回额外 `window_time` 字段支持窗口级联。老版分组窗口已弃用，新版本（1.16+）已支持会话窗口 TVF。

---

## 12. Flink CEP

### 12.1 核心概念

**CEP 是什么**

CEP = Complex Event Processing（复杂事件处理）。Flink CEP 是 Flink 提供的专门库（`flink-cep` 依赖），用于在无界事件流中检测"特定事件组合"。

**典型场景**：连续登录失败 3 次报警、下单后 15 分钟未支付提示、刷单检测、异常交易监控等。这类需求涉及事件顺序和时间约束，用 SQL 或普通 DataStream 状态编程实现代码复杂度高、可读性差。

**三步流程**：① 定义匹配规则（Pattern）；② 将 Pattern 应用到事件流得到 PatternStream；③ 对匹配的复杂事件处理并输出。

**模式（Pattern）**

模式定义两部分：每个简单事件的特征 + 事件之间的组合关系（近邻关系）。可扩展时间限制、是否可重复出现等。

**个体模式**

每个简单事件构成一个"个体模式"，以连接词（begin/next/followedBy）开始，带名称。

**量词（Quantifier）**：默认单例（匹配一个事件）；加量词变循环模式：
- `times(n)` / `times(from, to)`：固定次数或范围。
- `oneOrMore()`：1 次或多次（a+）。
- `timesOrMore(n)`：n 次或多次。
- `greedy()`：贪心匹配，尽量多匹配。
- `optional()`：可选。

**条件（Condition）**：
- `subtype(SubEvent.class)`：限定子类型。
- SimpleCondition：仅基于当前事件判断（filter 操作）。
- IterativeCondition：可访问上下文 `ctx.getEventsForPattern("name")` 拿已匹配事件，做跨事件判断（最通用）。
- 组合条件：`.where().where()` = AND，`.where().or()` = OR。
- `.until()`：循环模式的终止条件，防止状态无限增长，必须与 oneOrMore/optional 结合。

**组合模式（模式序列）**

以 `Pattern.begin("start")` 开头，用连接词拼接。

**近邻条件（三种）**：

| 连接词 | 关系 | 说明 |
|---|---|---|
| `next()` | 严格近邻 | 中间不能有任何其他事件 |
| `followedBy()` | 宽松近邻 | 顺序对即可，中间可有其他事件 |
| `followedByAny()` | 非确定性宽松近邻 | 已匹配事件可被重复使用，匹配结果最多 |

**否定连接词**：`notNext()`（不能严格紧邻某事件）、`notFollowedBy()`（不会出现某事件，不能用于模式结尾，因为永远无法证伪未来）。

**时间限制**：`within(Time.seconds(10))`，整个模式序列首末事件最大间隔；只能有一个，多次以最小为准。

**循环模式中的近邻**：默认宽松近邻。`.consecutive()`：强制严格近邻；`.allowCombinations()`：非确定性宽松近邻。

> 实战经验：检测"连续 3 次登录失败"若用 `times(3)` 默认宽松近邻（中间允许成功事件），不符合需求。正确写法：`.times(3).consecutive()` 或用三个单例模式 + next 串联。

**模式组**

复杂场景可把一个"模式序列"作为参数传入 begin/next/followedBy，得到 GroupPattern。GroupPattern 是 Pattern 子类，同样可加 times/oneOrMore/optional 等量词。

**匹配后跳过策略（AfterMatchSkipStrategy）**

控制循环模式匹配结果精简，作为 `begin("name", strategy)` 的第二参数：

| 策略 | 效果（以 a+a+b 输入 a1 a2 a3 b 为例）|
|---|---|
| NO_SKIP（默认） | 输出全部 6 种匹配 |
| SKIP_TO_NEXT | 跳过当前起始事件的其他匹配，输出 3 种 |
| SKIP_PAST_LAST_EVENT | 跳到上一个匹配的最后一个事件之后，最精简（1 种） |
| SKIP_TO_FIRST["a"] | 跳至指定模式的第一个匹配事件（3 种） |
| SKIP_TO_LAST["a"] | 跳至指定模式的最后一个匹配事件（2 种） |

**模式的检测处理**

**应用到流**：`CEP.pattern(inputStream, pattern)` -> `PatternStream`。建议先 keyBy 再 pattern（按 key 独立检测）。

**两种处理方式**：
- **select / flatSelect**（简单）：`PatternSelectFunction.select(Map<String, List<Event>>)` 返回单值；`PatternFlatSelectFunction.flatSelect` 多了 Collector，可多次输出。Map 的 key 是个体模式名，value 是事件列表（循环模式有多元素）。
- **process**（推荐）：`PatternProcessFunction.processMatch(Map, Context, Collector)`，多一个 Context，可获取时间戳/处理时间、可写侧输出流。

**处理超时事件**

`within()` 限定的模式，超时"部分匹配"不应直接丢弃，需输出提示。

- **方式一（推荐）**：`PatternProcessFunction` 同时实现 `TimedOutPartialMatchHandler` 接口，重写 `processTimedOutMatch(Map, Context)`，通过 `ctx.output(outputTag, data)` 写侧输出流。
- **方式二（兼容老版）**：`patternStream.select(OutputTag, PatternTimeoutFunction, PatternSelectFunction)`，超时结果进侧输出流，正常结果进主流。

**处理迟到数据**

CEP 沿用 Watermark 处理乱序：事件先进 buffer 按时间戳排序，Watermark 到来时才取出小于水位线的事件做匹配。超过 Watermark 延迟的迟到数据默认被丢弃。

可通过 `patternStream.sideOutputLateData(outputTag)` 将迟到数据写入侧输出流另行处理，与窗口机制一致。

**CEP 底层原理：状态机（NFA）**

CEP 模式匹配底层是一个**非确定有限状态自动机（NFA）**，与正则表达式引擎原理一致。每个事件到来会根据"当前状态 + 事件特性"转移到新状态。

> Flink SQL 也提供了 `MATCH_RECOGNIZE` 子句（SQL:2016 标准）在 SQL 中做模式识别，与 CEP 互补。

### 12.2 常见面试题

**Q1：什么是 Flink CEP？典型应用场景有哪些？**
CEP 是复杂事件处理库，用于在无界事件流中检测满足特定规则的"事件组合"（复杂事件），并对检测到的事件做处理输出。典型场景：风控（连续登录失败、刷单、欺诈交易）、订单超时未支付监控、用户画像、运维多指标告警。流程是：定义 Pattern -> 应用到流得到 PatternStream -> select/process 处理。

**Q2：CEP 中 next、followedBy、followedByAny 有什么区别？**
- `next`：严格近邻，两事件之间不能有任何其他事件。
- `followedBy`：宽松近邻，只保证先后顺序，中间可有其他事件。
- `followedByAny`：非确定性宽松近邻，已匹配事件可被后续匹配重复使用，匹配结果最多。
严格近邻匹配最少，非确定性宽松近邻匹配最多。另有 `notNext`/`notFollowedBy` 表示否定约束，但 `notFollowedBy` 不能作为模式结尾（无法证伪未来事件）。

**Q3：如何处理 CEP 中的超时事件和迟到数据？**
- **超时事件**：`within()` 设置模式时间限制后，未在时间内完成匹配的"部分匹配"通过 `TimedOutPartialMatchHandler`（推荐）或 `PatternTimeoutFunction`（老版）捕获，输出到侧输出流。
- **迟到数据**：CEP 用 Watermark 处理乱序，事件先进 buffer 按时间戳排序，Watermark 到来才匹配。超过 Watermark 延迟的迟到数据默认丢弃，可通过 `patternStream.sideOutputLateData(outputTag)` 写侧输出流另行处理。

**Q4：CEP 底层是怎么工作的？为什么不能直接用 DataStream API 替代？**
CEP 底层是**非确定有限状态自动机（NFA）**，与正则表达式引擎原理一致。每来一个事件，根据当前状态 + 事件特性做状态转移。理论上用 DataStream API + RichFlatMapFunction + ValueState + 手写状态机也能实现，但代码非常繁琐、易错、可读性差、难扩展。CEP 把 NFA 封装好，提供 Pattern API 让用户以声明式方式定义规则，大幅降低复杂度。

---

## 13. 补充：性能与调优要点

> 本节内容原 PDF 未专门成章，但属 Flink 面试高频考点（反压、内存模型、数据倾斜），特此补充。

### 13.1 核心概念

**反压机制（Backpressure）⭐**

什么是反压：流处理中，当下游算子处理速度跟不上上游发送速度时，会反向"压迫"上游减速，这种机制叫反压。类似水管堵塞--下游堵了，水（数据）向上游倒灌积压。

为什么需要：如果没有反压，上游不断发数据、下游来不及处理，会在下游缓冲区堆积最终 OOM。反压让上游自动降速，是流系统的"流量控制安全阀"。

**Flink 反压的演进**：
- **Flink 1.5 之前**：基于 TCP 滑动窗口的反压。当 socket 缓冲区满，TCP 自然反压。问题：反压粒度粗，会影响整个 TaskManager 的网络栈，一个作业的反压可能波及同 TM 上的其他作业。
- **Flink 1.5+：Credit-Based 流控机制**。Flink 在应用层实现了精细化反压，不再完全依赖 TCP。

**Credit-Based 流控原理**：
1. 下游 TaskManager 为每个输入通道分配一定数量的 **credit（信用/缓冲凭证）**，告诉上游"我能接收多少数据"。
2. 上游按下游给的 credit 数量发送对应数量的数据 + **backlog（积压量，表示上游还有多少未发数据）**。
3. 下游根据上游的 backlog 决定下次分配多少 credit：上游积压多 -> 下游多分配 credit（如果自己还有余力）；下游自己忙不过来 -> 减少 credit 分配。
4. credit 耗尽后上游停止向该下游发数据，实现精准反压。

> 通俗类比：上游是供货商、下游是零售商。下游每次告诉上游"我能再进 N 货"（credit=N），上游就发 N 货并告知"我仓库还堆着 M 货"（backlog=M）。下游看自己货架空、上游积压多就多要货；下游卖不动就少要货，上游自然就发得少了。

**反压的定位与排查**：
- Flink Web UI 的 **BackPressure** 面板显示每个算子反压状态：OK（正常）/ LOW / HIGH（高反压）。基于采样判断算子是否在等待网络缓冲。
- 常见反压原因：① 下游算子处理慢（复杂计算、外部 IO 慢）；② 数据倾斜导致某子任务过载；③ GC 停顿；④ 下游 sink（Kafka/DB）慢。

⚠️ **反压与 Checkpoint 的关系**：反压严重时 Barrier 对齐变慢、Checkpoint 超时失败（这正是非对齐检查点要解决的问题）。两者关联紧密，常一起考察。

**Flink 内存模型 ⭐**

为什么 Flink 自己管理内存：JVM 管理 GB 级以上堆内存时 GC 停顿严重（STW），对流处理的低延迟是致命的。Flink 把大量数据（如状态、排序缓冲）放在 JVM 堆外的**托管内存（Managed Memory）**中，自己以 segment 为单位管理，避开 GC，实现稳定低延迟。

**TaskManager 进程内存组成**（Flink 1.10+ 新内存模型）：

| 内存区域 | 说明 | 用途 |
|---|---|---|
| **JVM 堆内存** | TaskManager 的 JVM 堆 | 用户代码对象、HashMapStateBackend 的状态 |
| **托管内存** | Flink 管理的堆外内存 | RocksDB StateBackend、排序、批处理中间结果 |
| **网络内存** | 堆外 | 网络数据收发缓冲（shuffle、反压 credit 缓冲） |
| **框架堆内存** | Flink 框架自身堆内存，与用户代码隔离 | 框架运行 |
| **任务堆外内存** | 用户代码的堆外内存 | 用户直接 ByteBuffer 等 |
| **JVM Metaspace** | 类元数据 | 加载类 |
| **JVM 开销** | JVM 自身开销 | 线程栈、直接内存等 |

> 通俗类比：Flink 把内存分成几个独立"账户"--框架用框架的、网络用网络的、状态用托管的、用户代码用堆的。互不挪用、各管各的，避免一个作业把内存吃光拖垮其他作业，也便于精确调优。

关键点：
- **托管内存是堆外**，不受 GC 影响，RocksDB StateBackend 就用它存数据。
- 流处理主要关注：堆内存（HashMap 状态、用户对象）、托管内存（RocksDB 状态）、网络内存（数据传输）。

**数据倾斜与调优要点**

数据倾斜表现：某个 subtask 处理的数据量远超其他，成为瓶颈，拖慢整个作业且易 OOM。Web UI 上表现为某算子各 subtask 处理记录数严重不均。

**常见原因与解决**：

| 原因 | 解决方案 |
|---|---|
| keyBy 后 key 分布不均（如某大 V 用户行为占比极高） | **两阶段聚合**：先加随机前缀打散 key 聚合，再去前缀二次聚合 |
| keyBy 的 key 选择不当 | 换更均匀的 key，或预聚合 |
| 某分区数据天然多（Kafka 分区不均） | `rebalance()` 重新轮询均衡 |
| 窗口内某 key 数据量巨大 | 开窗 + 增量聚合，避免全缓存 |
| 状态膨胀（RocksDB 大 key） | 配状态 TTL，清理过期状态 |

**两阶段聚合思路**（高频）：
```
阶段1：key = 原始key + 随机数(1~N) -> keyBy + 聚合  (打散，局部聚合)
阶段2：key = 原始key           -> keyBy + 聚合  (汇总，全局聚合)
```
把大 key 的负载分散到 N 个子任务局部聚合，再汇总，缓解单点热点。类似 MapReduce 的 combiner + reduce。

**其他调优要点**：
- **并行度**：与 Kafka 分区数匹配（避免某分区空闲），与 slot 总数匹配。
- **状态后端选择**：大状态用 RocksDB + 增量 Checkpoint。
- **Checkpoint 间隔**：不宜过短（影响吞吐），不宜过长（故障重算多），通常秒到分钟级。
- **异步 I/O**：外部查询用 AsyncFunction 并发请求，避免阻塞主流程。
- **算子链**：默认开启减少序列化与线程切换开销。

### 13.2 常见面试题

**Q1：Flink 的反压机制是什么？Credit-Based 流控怎么工作？**
反压是下游处理不过来时让上游减速的机制。Flink 1.5+ 采用 Credit-Based 流控：下游给上游分配 credit（可接收数据量）并告知缓冲余量，上游按 credit 发数据并回报 backlog（积压量），下游根据 backlog 和自身能力动态调整 credit。credit 耗尽上游停止发送。相比老的 TCP 反压，粒度更细、不波及同 TM 的其他作业。

**Q2：反压和 Checkpoint 有什么关系？**
反压严重时 Barrier 对齐变慢（下游缓存大量 in-flight data），导致 Checkpoint 超时失败。这是非对齐检查点（Unaligned Checkpoint）要解决的痛点：直接保存 in-flight data，不再依赖 barrier 对齐，缓解反压下 checkpoint 失败。

**Q3：如何排查 Flink 反压？**
Web UI BackPressure 面板看各算子反压状态（OK/LOW/HIGH）。定位到高反压算子后排查原因：下游算子处理慢（复杂计算、外部 IO）、数据倾斜、GC、下游 sink 慢。对症解决：优化算子逻辑、解决倾斜、调大并行度、异步 IO。

**Q4：Flink 为什么自己管理内存而不全用 JVM 堆？**
JVM 管理 GB 级堆内存时 GC 停顿严重，对流处理低延迟致命。Flink 把状态、排序缓冲等放在堆外托管内存，以 segment 为单位自管理，避开 GC，保证稳定低延迟。

**Q5：Flink TaskManager 内存由哪几部分组成？**
JVM 堆（用户对象、HashMap 状态）、托管内存（堆外，RocksDB 状态、排序、批中间结果）、网络内存（堆外，数据传输与反压缓冲）、框架堆/堆外、任务堆外、JVM Metaspace 和开销。各区域独立配置，互不挪用。

**Q6：Flink 数据倾斜怎么处理？两阶段聚合的原理？**
先 Web UI 定位（看各 subtask 处理量是否不均）。若是 keyBy 后 key 分布不均，用**两阶段聚合**：阶段1 给 key 加随机前缀打散做局部聚合，阶段2 去前缀做全局聚合，相当于先 combiner 后 reducer，把单点热点负载分散开。若是 Kafka 分区不均用 rebalance 重新均衡；状态膨胀配 TTL；外部 IO 用异步。

---

## 资料勘误与重点提醒（第9-12章）

1. ⚠️ **HashMapStateBackend 适用场景描述前后矛盾**：资料既说它「适用于具有大状态、长窗口、大键值状态的作业」，又说它「状态大小受集群可用内存限制」。正确：HashMapStateBackend 适合中小状态、追求性能；大状态必须用 RocksDBStateBackend。
2. ⚠️ **`ReducintState` 笔误**：原文拼写错误，正确为 `ReducingState`。
3. ⚠️ **ACID 的 C 不是 Correspondence**：原文事务四特性写为「原子性(Atomicity)、一致性(Correspondence)、隔离性(Isolation)和持久性(Durability)」。**正确为 Consistency（一致性）**，Correspondence 是错误的英文翻译。
4. ⚠️ **EXACTLY-ONCE 的精确含义**：资料明确强调「不是数据只被处理一次，而是数据对状态和输出的影响只体现一次」。这是面试中常被误解的点，必须分清。
5. ⚠️ **TTL 只支持 Processing Time**：资料已提及，但是高频考点，需重点记忆；ListState/MapState 是 per-entry 失效。
6. ⚠️ **JobManager 堆内存存储检查点仅用于测试**：生产环境必须用 HDFS/S3 等分布式文件系统，且要高可用。
7. ⚠️ **新版本 Flink API 演进**（资料未涵盖，面试可能问）：
   - Flink 1.13+ 废弃了 MemoryStateBackend / FsStateBackend，统一为 HashMapStateBackend + CheckpointStorage 模型。
   - Flink 1.15+ 用 KafkaSink 替代 FlinkKafkaProducer，配置改为 DeliveryGuarantee.EXACTLY_ONCE。
8. ⚠️ **算子 .uid() 的重要性**：从 Savepoint 恢复时，Flink 通过算子 ID 匹配状态。不指定 .uid() 时 Flink 基于算子结构自动生成 hash，一旦程序改动 ID 就变，导致状态无法匹配恢复，作业直接失败。生产代码必须显式 .uid()。
9. ⚠️ **Chandy-Lamport 算法是面试核心**：要能讲清楚 barrier 注入、广播、对齐、异步快照四个关键点，并能说出对齐导致的背压问题与非对齐检查点的解决方案。这是区分候选人水平的关键题。
10. ⚠️ **资料笔误（Table API）**：示例代码中应使用 `toChangelogStream(urlCountTable).print()` 却误写为 `toDataStream(urlCountTable).print()`。`urlCountTable` 是分组聚合的更新查询结果，用 `toDataStream` 会抛 `TableException`。
11. ⚠️ **Table API/SQL 版本信息滞后**：资料基于 Flink 1.13，多处描述已过时："Table API 和 SQL 依然不算稳定" -> 1.17+ 已成熟；"blink planner" 需手动指定 -> 1.15 后 Old Planner 已移除；"会话窗口 TVF 尚未完全支持" -> 1.16+ 已支持 Session Window TVF。面试中应说明自己使用的是较新版本（如 1.17/1.18）。
12. ⚠️ **MATCH_RECOGNIZE**：资料末尾提到但未展开。这是 SQL 标准的模式识别子句，让 SQL 也能做 CEP，是面试加分点。
13. ⚠️ **CEP 状态膨胀风险**：`oneOrMore` 等循环模式若不配 `within` 或 `until` 会导致状态无限增长 OOM，面试常被追问。
14. ⚠️ **流到表的转换桥梁**：`Row` 类型 + `RowKind` 是 Flink 中连接"流"和"表"的核心数据结构。`RowKind` 取值 `+I`（INSERT）、`-U`（UPDATE_BEFORE）、`+U`（UPDATE_AFTER）、`-D`（DELETE），是动态表编码的物理载体。

---

## 附录：Flink 面试核心知识地图（速记）

```
Flink 核心 = 流处理 + 有状态 + 容错

【架构】 JobManager(JobMaster/ResourceManager/Dispatcher/WebMonitorEndpoint) + TaskManager(slot)
         四层图: StreamGraph(客户端) -> JobGraph(客户端, 合并算子链) -> ExecutionGraph(JM, 并行化) -> PhysicalGraph(TM)

【API】   SQL > Table API > DataStream API > ProcessFunction
         程序: env -> source -> transform -> sink -> execute()

【时间】  事件时间(默认,1.12+) > 处理时间 > 摄入时间(已弱化)
【水位线】= maxTimestamp - delay - 1ms; 传递取所有上游最小(木桶原理); 周期是处理时间
【窗口】  滚动(无重叠)/滑动(可重叠)/会话(动态,仅时间); 增量聚合 vs 全窗口(最佳:结合)
【迟到】  三道防线: 水位线延迟(全局) -> allowedLateness(窗口级,先近似再修正) -> 侧输出流(兜底)

【状态】  Keyed State(按key,5种类型) vs Operator State(算子级,List/UnionList/Broadcast)
         状态后端: HashMap(堆,快,小状态) vs RocksDB(磁盘,大状态,唯一支持增量CP)
【容错】  Checkpoint = Chandy-Lamport + Barrier 对齐; Savepoint = 手动备份(需.uid())
         一致性: at-most-once / at-least-once / exactly-once(对状态影响只一次)
         端到端EO: 输入可重放(Kafka offset) + Flink CP + 输出幂等或2PC事务
         Flink+Kafka EO: offset存CP + Sink两阶段提交 + 下游read_committed + 事务超时配置

【多流】  Union(同类型,多条) vs Connect(不同类型,两条,CoProcessFunction)
         双流Join: Window Join(笛卡尔积) / Interval Join(动态区间,仅事件时间) / CoGroup(可外连接)
【处理函数】 KeyedProcessFunction 可注册定时器(key+时间戳去重,容错); 侧输出流分流
【Table/SQL】动态表 + 持续查询(流->表->查询->流); 更新查询(toChangelogStream) vs 追加查询(toDataStream)
            Append/Retract/Upsert 三种编码; 窗口用 TVF(新)替代分组窗口(旧,已弃用)
【CEP】   复杂事件处理,底层NFA状态机; next严格/followedBy宽松/followedByAny非确定; 超时+迟到走侧输出流
【调优】  反压=Credit-Based流控(1.5+,下游给credit+backlog,不波及同TM); 严重反压致CP对齐慢->非对齐CP
         内存=自管理堆外托管内存避GC; 堆/托管/网络/框架独立账户
         倾斜=两阶段聚合(随机前缀打散+去前缀全局聚合); 空闲源withIdleness; 状态TTL
```
