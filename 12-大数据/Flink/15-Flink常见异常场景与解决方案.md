# 第15章 Flink 常见异常场景与解决方案

> 本章是 Flink 面试「场景题」的集中训练营。不同于第14章偏 UI 工具与排查方法论，本章以**故障场景**为主线，每个场景按「现象 → 根因分类 → 排查路径 → 解决方案 → 面试金句」的框架展开，覆盖面试中最常被问到的生产环境问题。建议结合第10章容错机制、第13章性能调优一起复习。

---

## 一、反压（Backpressure）⭐⭐⭐

### 1.1 什么是反压

反压是流处理系统的**流量控制机制**：当下游算子处理速度跟不上上游发送速度时，下游会反向"压迫"上游减速，防止数据无限堆积最终 OOM。

Flink 1.5+ 采用 **Credit-Based 流控**，在应用层实现精细化反压，不再完全依赖 TCP 滑动窗口。反压从**下游往上游传播**——一个慢算子会把它所有上游都憋住，最终表现为整条链路全红。

> 💡 面试金句：反压是**结果**，不是**原因**。反压面板全红 ≠ 所有算子都慢，真正的瓶颈只有一个或几个。

### 1.2 现象

- Flink UI Backpressure 面板显示多个算子为 HIGH
- 下游 Sink 或中间某算子的 Busy Time 持续 100%
- Source 端消费速度下降、Kafka Lag 增加
- Checkpoint 对齐时间变长，甚至超时失败
- 吞吐量（Records/Sec）持续低于预期

### 1.3 根因分类（5 大类）

| 类别 | 典型场景 | UI 特征 |
|---|---|---|
| **算子计算慢** | 复杂正则、JSON 反复解析、大量字符串操作、深拷贝 | 瓶颈算子 Busy Time 100%，无数据倾斜，CPU 高 |
| **数据倾斜** | 热点 key 导致某 subtask 过载 | 瓶颈算子某 subtask 处理量远超其他 |
| **外部 IO 瓶颈** | Sink 写入慢（DB/Kafka）、同步维表查询、慢 HTTP 调用 | 瓶颈算子 idle 时间也高（在等 IO），Busy Time 未必 100% |
| **GC 停顿** | 大堆 + Full GC 频繁，处理线程暂停 | TM 的 GC Time/Count 异常高，反压时有时无 |
| **资源不足** | CPU 打满、网络带宽瓶颈、磁盘 IO 打满（RocksDB） | 整台机器指标异常，所有 TM 都受影响 |

### 1.4 排查步骤

1. **确认瓶颈算子**：从最下游 Sink 开始往上游找 Backpressure 面板，找到反压从 OK → HIGH 的交界点，交界点的下游算子就是瓶颈
2. **判断是否数据倾斜**：点开瓶颈算子的 Subtasks 列表，看各 subtask 的 Records Received/Sent 和 Busy Time 是否均匀
3. **判断是否 IO 瓶颈**：看瓶颈算子的 Busy Time + Idle Time，如果 Busy 不高但反压严重，大概率在等外部 IO
4. **判断是否 GC 问题**：去对应 TM 看 GC 指标（次数和时间），看 GC 日志
5. **看系统资源**：CPU、内存、网络、磁盘 IO 是否打满
6. **代码审查**：检查瓶颈算子的实现，是否有复杂计算、阻塞调用、大对象创建

### 1.5 解决方案

**按根因对症下药：**

| 根因 | 解决方案 |
|---|---|
| 算子计算慢 | 优化算法（如正则改字符串匹配、JSON 解析复用对象池、减少序列化）、开启算子链减少数据传递、增加并行度 |
| 数据倾斜 | 参见「二、数据倾斜」章节 |
| 外部 IO 慢（Sink） | 批量写入、异步写入、增加 Sink 并行度、优化目标端（DB 索引/批量提交） |
| 外部 IO 慢（维表查询） | 用 AsyncFunction 异步查询 + 本地缓存、改为广播维表、热点 key 本地缓存 |
| GC 频繁 | 增大堆内存、减少对象创建（复用对象/对象池）、切换状态后端到 RocksDB（减少堆内存压力）、调优 GC 参数 |
| CPU 不足 | 增加并行度、扩容 TM、优化热点算子代码 |
| 网络瓶颈 | 减少数据 shuffle、用 forward 模式、调大网络缓冲、检查机架部署 |

### 1.6 反压与 Checkpoint 的关系

反压是 Checkpoint 超时失败的**第一大原因**：
- 反压 → in-flight 数据堆积 → Barrier 对齐变慢 → Alignment Time 变长 → 超过超时阈值 → CP 失败
- 反压严重时建议开启 **Unaligned Checkpoint（非对齐检查点）**，将 in-flight 数据也存入 CP，跳过对齐等待
- 注意：非对齐 CP 会增加 CP 数据量（多存了 in-flight data），且要求并发 CP 数 = 1

---

## 二、数据倾斜（Data Skew）⭐⭐⭐

### 2.1 什么是数据倾斜

数据倾斜是指：在并行处理中，**某一个或少数几个子任务（subtask）处理的数据量远大于其他子任务**，导致这些子任务成为瓶颈，拖慢整个作业的吞吐量，甚至 OOM。

数据倾斜是 Flink 面试最高频的场景题之一，几乎必问。

### 2.2 现象

- UI 上某算子的 Subtasks 列表中，个别 subtask 的 Records Received/Sent 远高于其他（数量级差距）
- 倾斜 subtask 的 Busy Time 持续 100%，其他 subtask 大部分时间空闲
- 倾斜 subtask 所在 TM 的内存、CPU 偏高
- 反压从倾斜 subtask 处开始向上传播
- 严重时倾斜 subtask OOM，作业失败

### 2.3 常见倾斜场景（6 类）

#### 场景 1：keyBy 后 key 分布不均（最常见）

**原因**：业务数据天然存在热点，比如「大 V 用户」的行为数据、「热门商品」的点击数据、「null/空值 key」等。

**特征**：keyBy 后的算子某 subtask 处理量是其他的 N 倍。

#### 场景 2：Kafka 分区数据不均

**原因**：Kafka topic 分区数据量不均匀（生产者分区策略问题，或业务数据集中在某几个分区）。

**特征**：Source 各 subtask 消费速率差异大，且 Source 端就已经出现反压。

#### 场景 3：窗口大 key（热点 key 窗口数据量巨大）

**原因**：热点 key 在窗口内积累了海量数据，窗口触发时计算量巨大。

**特征**：窗口算子的倾斜 subtask 状态大小远高于其他，窗口触发时 CPU 飙升。

#### 场景 4：双流 Join 热点 key

**原因**：两个流按 key join，某个 key 在两个流中都有海量数据，笛卡尔积爆炸。

**特征**：Join 算子某 subtask 处理量和状态都远超其他，吞吐低。

#### 场景 5：维表 Join 热点 key

**原因**：主流中有大量数据需要 join 同一个维表 key（如「默认省份」「系统用户」），导致维表查询热点。

**特征**：AsyncIO 或 RichFlatMap 查询维表的算子出现热点，且该 key 查询量极高。

#### 场景 6：分组聚合数据倾斜

**原因**：group by 某个维度后，某个维度值的数据量特别大。

**特征**：聚合算子某 subtask 处理量和状态远大于其他。

### 2.4 解决方案大全

#### 方案 1：两阶段聚合（加随机前缀打散）⭐ 最高频

**原理**：先给 key 加随机前缀（如 `key_0` ~ `key_N`）打散到多个子任务做局部聚合，再去掉前缀做全局聚合。

```
原始 key → [key_0, key_1, ..., key_N] → 局部聚合 → 去掉前缀 → 全局聚合
```

**适用场景**：分组聚合、窗口聚合等可拆分的聚合操作（sum、count、min、max）。

**优点**：效果显著，代码改动小。
**局限**：只适用于可拆分的聚合（求平均要自己处理 sum + count），不适合非聚合类操作。

#### 方案 2：热点 key 拆分 + 单独处理

**原理**：识别出已知的热点 key（如 `null`、空字符串、特定大 V），用 side output 或分支逻辑把热点 key 单独走一条并行链路处理，非热点 key 正常处理。

**适用场景**：热点 key 已知且数量少（比如就几个大 V、或 null 值）。

**优点**：针对性强，效果最好。
**局限**：需要提前知道热点 key，动态热点不适用。

#### 方案 3：rebalance / rescale 重分区

**原理**：通过 `dataStream.rebalance()` 轮询分配到下游所有并行实例，强制均匀分布。

**适用场景**：Source 端分区不均（如 Kafka 分区不均）、上游数据分布不均但下游不需要按 key 聚合时。

**优点**：简单直接。
**局限**：会产生额外的网络 shuffle，且只解决「均匀分发」问题，不解决 keyBy 后的倾斜。

#### 方案 4：增量聚合 + 预聚合

**原理**：窗口从全量聚合（ProcessWindowFunction 存所有数据）改为增量聚合（AggregateFunction / ReduceFunction 只存累加值），大幅减少状态大小和计算量。

**适用场景**：窗口大 key 导致的倾斜。

**优点**：减少状态、提升性能。
**局限**：只能拿到聚合结果，拿不到原始明细数据。

#### 方案 5：广播维表 + 本地缓存

**原理**：维表 Join 热点问题，将维表（小表）广播到所有 TaskManager，在本地内存中直接查询，不走网络 IO。配合本地缓存（Guava Cache / Caffeine）进一步加速。

**适用场景**：维表 Join 热点，且维表不大（能放入内存）。

**优点**：彻底消除维表查询的网络开销和热点。
**局限**：维表不能太大，需要处理维表更新问题。

#### 方案 6：加盐打散 + 多并行度 Join

**原理**：双流 Join 热点 key，将其中一条流的热点 key 加盐（加随机后缀），另一条流的对应 key 也复制 N 份（每份加一个后缀），这样 join 就能分散到 N 个并行实例。

**适用场景**：双流 Join 热点 key。

**优点**：有效打散 Join 热点。
**局限**：代码复杂，另一条流数据量会膨胀 N 倍。

### 2.5 方案选型建议

| 场景 | 首选方案 | 次选方案 |
|---|---|---|
| keyBy 聚合倾斜 | 两阶段聚合 | 热点 key 拆分 |
| Kafka 分区不均 | rebalance | 调整 Kafka 分区策略 |
| 窗口大 key | 增量聚合 + 两阶段 | 状态 TTL |
| 双流 Join 热点 | 加盐打散 Join | 热点 key 单独处理 |
| 维表 Join 热点 | 广播维表 + 本地缓存 | 热点 key 本地缓存 |

---

## 三、作业频繁重启 ⭐⭐⭐

### 3.1 现象

- Flink UI 上作业状态在 RUNNING / RESTARTING / FAILING 之间反复切换
- Overview 中 Running Jobs 数不稳定
- 作业频繁从 Checkpoint 恢复，数据处理有断点
- JM 日志中反复出现 "Job switched to state RESTARTING"

### 3.2 重启策略概述

Flink 的重启策略决定了作业失败后是否重启、怎么重启：

| 策略 | 说明 | 配置 |
|---|---|---|
| **固定延迟重启**（Fixed Delay） | 最多重启 N 次，每次间隔固定时间 | `restart-strategy: fixed-delay` |
| **失败率重启**（Failure Rate） | 一段时间内失败超过 N 次就不再重启 | `restart-strategy: failure-rate` |
| **无重启**（No Restart） | 失败直接挂掉，不重启 | `restart-strategy: none` |
| **回退策略** | 未配置时使用：启用 Checkpoint → 固定延迟（Integer.MAX_VALUE 次）；未启用 → 不重启 | 默认 |

> 💡 面试金句：如果配置了 Checkpoint 但没配重启策略，Flink 默认用**固定延迟重启，重启次数是 Integer.MAX_VALUE**——也就是无限重启。很多人以为配置了 CP 就只会重启几次，其实默认是无限次。

### 3.3 根因分类（6 大类）

#### 1. OOM 导致 TM 挂掉 → JM 重启作业

**最常见原因。** TM 进程因为内存溢出挂掉，JM 检测到 TM 失联后，认为作业失败，触发重启策略。

**日志特征**：
- TM 日志：`java.lang.OutOfMemoryError: Java heap space` / `Direct buffer memory` / `Metaspace`
- JM 日志：`TaskManager <id> lost`、`Job xxx failed, restarting`

#### 2. Checkpoint 连续失败超过阈值

当 `tolerableCheckpointFailureNumber` 配置了允许失败次数（默认 0，即 CP 失败就认为作业失败），连续 CP 失败达到阈值会触发作业失败重启。

> ⚠️ 注意：Flink 1.13+ 默认 `tolerableCheckpointFailureNumber = 0`，意味着 **CP 失败一次作业就挂**。很多生产事故源于这个配置被忽略。

**特征**：Checkpoint 面板连续多个 Failed，然后作业重启。

#### 3. 用户代码抛出未捕获异常

用户自定义函数（MapFunction、ProcessFunction 等）中抛出的异常没有被 try-catch，会导致 Task 失败，进而触发作业重启。

**常见类型**：
- NPE（空指针异常）
- 类型转换异常
- JSON 解析失败
- 数组越界
- 业务异常（非法数据）

**特征**：Exceptions 面板有明确的业务异常栈，指向用户代码的某一行。

#### 4. 超时导致的失败

**心跳超时**：TM 长时间没有向 JM 发送心跳，JM 认为 TM 失联。
- 原因：Full GC 停顿、网络分区、TM 进程卡死

**Checkpoint 超时**：CP 超过 `checkpointTimeout` 时间未完成。
- 原因：反压、大状态、存储 IO 慢

#### 5. 依赖服务不可用（外部抖动）

Flink 作业依赖的外部服务短暂不可用，导致算子失败：
- Kafka 集群抖动（Source / Sink 报错）
- HDFS 不可用（Checkpoint 存储失败）
- 数据库挂了（Sink 写入失败）
- ZooKeeper 会话超时（HA 模式）

**特征**：异常信息明确指向外部服务，且通常是多个作业同时受影响。

#### 6. 资源问题

- **Slot 不足**：重启时申请不到足够的 Slot，作业一直等待或失败
- **容器被驱逐**（K8s / YARN）：资源不足时容器被调度器杀掉
- **磁盘满**：RocksDB 或 Checkpoint 存储磁盘写满，TM 挂掉

### 3.4 排查路径

```
① 看作业重启时间点
   ↓
② 看 Exceptions 面板：有没有明确的异常栈？
   ├─ 有 → 是代码问题还是外部依赖问题？
   └─ 无 → 往下走
   ↓
③ 看 TaskManagers：有没有 TM 丢失？
   ├─ 有 → 看丢失 TM 的日志 → OOM？GC？磁盘？容器驱逐？
   └─ 无 → 往下走
   ↓
④ 看 Checkpoints 面板：CP 失败次数是否达到阈值？
   ├─ 是 → 查 CP 失败原因（反压？状态大？存储慢？）
   └─ 否 → 往下走
   ↓
⑤ 看资源指标：CPU、内存、磁盘、网络是否异常？
   ↓
⑥ 看外部依赖：Kafka、HDFS、DB 是否正常？
```

### 3.5 解决方案

| 根因 | 解决方案 |
|---|---|
| OOM | 参见「五、内存溢出 OOM」章节 |
| CP 连续失败 | 调大 `tolerableCheckpointFailureNumber`、解决 CP 失败根因、开启非对齐 CP |
| 用户代码异常 | 修复 Bug、加 try-catch 异常处理、异常数据侧输出而不是抛出 |
| 心跳超时 | 调大心跳超时、解决 GC 问题、排查网络 |
| 外部依赖抖动 | 加重试机制（Flink Kafka Connector 自带重试）、降级处理、限流 |
| 资源问题 | 扩容、调大并行度/TM 数、磁盘清理、资源隔离 |

### 3.6 面试金句：如何判断是代码问题还是环境问题？

**三步走**：
1. **看异常栈**：有明确的用户代码行号 → 代码问题；异常指向外部系统（Kafka/HDFS/DB）→ 环境/依赖问题
2. **看是否复现**：从 Savepoint 重启后同样位置同样时间又挂 → 大概率代码/数据问题；随机时间挂 → 环境问题可能性大
3. **看影响范围**：只有这一个作业有问题 → 代码/数据问题；多个作业同时出问题 → 集群/环境问题

---

## 四、Checkpoint 失败 / 超时 ⭐⭐

### 4.1 现象

- Checkpoints 面板 History 列表出现大量 Failed
- Alignment Time 或 Sync/Async Duration 接近/超过超时阈值
- 作业频繁重启（如果 `tolerableCheckpointFailureNumber` 配得低）
- 状态大小持续增长

### 4.2 CP 失败的 6 大原因

#### 1. 反压导致对齐超时（最常见）

**原理**：反压时下游积压大量 in-flight 数据，Barrier 在数据队列中排队，导致所有上游通道的 Barrier 到齐时间变长，对齐超时。

**UI 特征**：
- Backpressure 面板显示高反压
- Checkpoint 的 **Alignment Time** 很长，占端到端时间的主要部分
- State Size 不一定大

**解决方案**：
- 先解决反压问题（参见「一、反压」）
- 开启 **Unaligned Checkpoint**（非对齐检查点），跳过对齐等待
- 调大 `checkpointTimeout`（治标不治本，不推荐）

#### 2. 状态过大 / 状态膨胀

**原理**：状态数据量太大，快照写入耗时久，超过超时阈值。

**UI 特征**：
- **State Size** 很大（GB 级甚至 TB 级）
- Sync Duration 和 Async Duration 都长
- 各 subtask 状态大小可能不均（倾斜）

**解决方案**：
- 配置**状态 TTL**，清理过期状态
- 用增量 Checkpoint（RocksDB 状态后端支持）
- 切换到 RocksDB 状态后端（堆外存储，增量 CP）
- 增大并行度，分散状态
- 排查是否有状态泄漏（参见「八、状态持续膨胀」）

#### 3. RocksDB 性能瓶颈

**原理**：RocksDB 的快照（snapshot）写入慢，或 compaction 导致 IO 抖动。

**UI 特征**：
- **Sync Duration** 较长（同步阶段做快照）
- TM 所在机器磁盘 IO 高
- RocksDB 相关指标（如 `rocksdb_bytes_read` / `rocksdb_bytes_written`）异常

**解决方案**：
- 调优 RocksDB 配置（增大 write buffer、调整 compaction 策略）
- 使用 SSD 存储 RocksDB 本地数据
- 开启 RocksDB 增量 Checkpoint
- 调大 RocksDB 内存配额（增大 state.backend.rocksdb.memory.managed）
- 开启 RocksDB 并发写入

#### 4. 存储 IO 瓶颈（HDFS / S3 慢）

**原理**：Checkpoint 数据写入外部存储（HDFS/S3/OSS）时，存储端读写性能不足。

**UI 特征**：
- **Async Duration** 很长（异步阶段写外部存储）
- Sync Duration 正常
- 网络 IO 高（往 HDFS 传数据）

**解决方案**：
- 优化 HDFS/S3（增加 DataNode、用更快的磁盘、调整副本数）
- 调大异步写入线程数
- 检查网络带宽是否足够
- 考虑使用更快的存储介质

#### 5. TM 失联 / GC 停顿

**原理**：某个 TM 因为 GC 停顿或失联，CP Coordinator 一直等它的 ACK，直到超时。

**UI 特征**：
- 某个 TM 的指标异常或直接失联
- CP 失败原因显示 "Some tasks did not acknowledge checkpoint"
- TM 日志有 GC 日志或 OOM

**解决方案**：
- 解决 GC 问题（调大内存、优化代码、用 RocksDB 减少堆压力）
- 解决 TM 丢失问题（参见「六、TaskManager 丢失」）

#### 6. Barrier 对齐问题（多输入并行度不一致）

**原理**：多个上游的并行度不同，或某个上游通道数据特别慢，导致 Barrier 到达不同步。

**UI 特征**：
- 某算子有多个输入，其中一个输入的 Barrier 到得特别晚
- 各上游通道的 barrier 到达时间差大

**解决方案**：
- 开启非对齐 Checkpoint
- 调整上游并行度使其一致
- 排查慢输入通道的原因

### 4.3 特别专题：状态膨胀 vs 状态泄漏

这是面试常考的辨析题：

| 维度 | 状态膨胀（正常累积） | 状态泄漏（异常只增不减） |
|---|---|---|
| 原因 | 窗口触发前数据正常积累、大状态 key 正常存在 | 状态没配 TTL、ListState 无限 append、广播状态不清理 |
| 趋势 | 周期性涨跌（窗口触发后回落） | 持续增长，只增不减 |
| CP 大小变化 | 有升有降，符合业务周期 | 单调递增，无回落 |
| 是否需要修复 | 正常现象，不需要 | 需要修复，否则最终 OOM |

**判断方法**：看 Checkpoint Size 的趋势图——如果窗口触发后大小明显回落，就是正常累积；如果一直涨上去不下来，就是泄漏。

---

## 五、内存溢出 OOM ⭐⭐

### 5.1 现象

- TaskManager 进程突然挂掉，UI 上 TM 丢失
- 作业异常失败并重启
- TM 日志中出现 `OutOfMemoryError`

### 5.2 四种 OOM 类型详解

#### 1. 堆内 OOM（Java heap space）

**日志特征**：`java.lang.OutOfMemoryError: Java heap space`

**常见原因**：
- HashMapStateBackend 存储大状态
- 用户代码创建大对象（大集合、大数组）
- 窗口全量聚合（ProcessWindowFunction 缓存所有数据）
- 对象泄漏（静态集合、ThreadLocal 不清理）
- 数据倾斜导致某 subtask 状态过大

**排查**：
- UI 上看 Heap Used 趋势（持续飙升）
- 加 `-XX:+HeapDumpOnOutOfMemoryError` 拿到堆转储文件分析
- 看 GC 日志（Full GC 频繁但回收效果差）

**解决**：
- 切换到 RocksDB 状态后端（堆外存储）
- 窗口从全量聚合改为增量聚合
- 优化代码，减少大对象创建
- 调大 TaskManager 堆内存
- 解决数据倾斜

#### 2. 堆外 OOM（Direct buffer memory）

**日志特征**：`java.lang.OutOfMemoryError: Direct buffer memory`

**常见原因**：
- 网络内存配置不足（并行度高 + 数据量大）
- 用户代码中直接使用 `ByteBuffer.allocateDirect()` 分配堆外内存
- Netty 缓冲泄漏

**排查**：
- UI 上看 Network Memory 使用情况
- 检查 `taskmanager.memory.network.min/max/fraction` 配置
- 检查用户代码是否有直接内存分配

**解决**：
- 调大网络内存比例（`taskmanager.memory.network.fraction`）
- 检查并修复用户代码的直接内存泄漏
- 减少并行度或调大 TM 总内存

#### 3. Metaspace OOM

**日志特征**：`java.lang.OutOfMemoryError: Metaspace`

**常见原因**：
- 动态类加载过多（如 CGLIB 动态代理、大量 Groovy/JS 脚本）
- 多作业部署在同一 TM，classloader 过多
- Metaspace 配置太小

**排查**：
- UI 上看 Non-Heap Used 中的 Metaspace 部分
- 加 `-XX:+TraceClassLoading` 看类加载情况

**解决**：
- 调大 `-XX:MaxMetaspaceSize`
- 减少动态类的使用
- 每个作业使用独立的 classloader（Flink 默认已做，检查是否有关闭）

#### 4. Native OOM（RocksDB / 系统级）

**日志特征**：
- TM 日志报 `std::bad_alloc` 或 `Cannot allocate memory`
- 系统日志中出现 OOM Killer 杀掉 Java 进程
- 进程直接消失，JVM 没有 OOM 栈

**常见原因**：
- RocksDB 状态太大，占用的 native 内存超过托管内存配额
- 托管内存配置不足
- 容器内存限制小于 JVM 实际使用量（堆 + 堆外 + 托管 + Metaspace + 线程栈 > 容器限制）

**排查**：
- 看系统日志（`dmesg` 或 K8s event）是否有 OOM Killer
- 看 RocksDB 内存使用指标
- 计算总内存是否超过容器/物理机限制

**解决**：
- 调大 `taskmanager.memory.managed.size`（RocksDB 用的是托管内存）
- 调大容器内存限制
- 减少 RocksDB 状态（TTL、清理）
- 确保 JVM 各内存区域之和不超过容器限制

### 5.3 内存配置调优指南

Flink 1.10+ 新内存模型，关键配置项：

| 配置 | 说明 | 建议值 |
|---|---|---|
| `taskmanager.memory.process.size` | TM 进程总内存（容器内存） | 按需，通常 4~16 GB |
| `taskmanager.memory.managed.fraction` | 托管内存占比 | 默认 0.4，RocksDB 状态大可调高 |
| `taskmanager.memory.network.fraction` | 网络内存占比 | 默认 0.1，高并发网络大可调高 |
| `taskmanager.memory.task.heap.size` | 任务堆内存 | 用户代码用，按需调整 |
| `taskmanager.memory.managed.size` | 托管内存固定大小 | 大状态场景建议显式配置 |

---

## 六、TaskManager 丢失 ⭐

### 6.1 现象

- Flink UI Overview 中 Available TaskManagers 数量减少
- 运行中的作业失败或重启
- JM 日志中出现 "TaskManager lost" / "TaskManager disconnected"

### 6.2 六大原因与排查

| 原因 | 排查方式 | 日志关键字 |
|---|---|---|
| **心跳超时（网络问题）** | JM 日志搜 "lost task manager"，看网络是否抖动，ping 测试 | `heartbeat timeout` |
| **Full GC 导致心跳中断** | TM 日志看 GC 日志，GC pause > 心跳超时阈值（默认 50s） | `Full GC`、`GC pause` |
| **OOM 进程退出** | TM 日志搜 OOM，看系统日志是否被 OOM Killer 杀掉 | `OutOfMemoryError`、`Killed process` |
| **磁盘满** | TM 日志搜磁盘错误，`df -h` 看磁盘使用率 | `No space left on device` |
| **容器被回收（K8s/YARN）** | 看 K8s event 或 YARN 日志，资源不足被驱逐 | `Evicted`、`Container killed` |
| **网络分区** | 多 TM 同时失联，检查网络设备/安全组/防火墙 | 多个 TM 同时 lost |

### 6.3 解决方案

| 原因 | 解决方案 |
|---|---|
| 心跳超时 | 调大 `heartbeat.timeout`（默认 50000ms）、排查网络问题、检查安全组 |
| Full GC | 调大堆内存、优化代码减少对象创建、切换 RocksDB 降低堆压力 |
| OOM | 参见「五、内存溢出 OOM」 |
| 磁盘满 | 清理 RocksDB 数据目录、配置磁盘监控、调大磁盘容量 |
| 容器驱逐 | 调大资源请求和限制、配置 PodDisruptionBudget、优化资源使用 |
| 网络分区 | 排查网络设备、检查跨机房部署、配置可靠网络 |

---

## 七、水位线不推进 / 窗口不触发 ⭐⭐

### 7.1 现象

- 窗口长时间不输出结果
- UI 上 `currentOutputWatermark` 长期为 -9223372036854775808（Long.MIN_VALUE，表示没有水位线）或不再增长
- 窗口算子的状态持续累积但不触发
- Sink 端没有数据输出

### 7.2 四大原因

#### 1. 空闲 Source（最常见也最容易忽略）

**原因**：如果 Source 的并行度 > 1，且其中某个分区没有数据（空闲），该分区不会产生 watermark。而下游算子的 watermark 取所有上游的**最小值**，一个分区没有 watermark 会导致整个下游的 watermark 卡住不推进。

**典型场景**：
- Kafka topic 有 10 个分区，但只有 5 个分区有数据
- 业务低谷期某些分区暂时没有数据
- 新增了分区但还没写入数据

**解决方案**：使用 `WatermarkStrategy.forBoundedOutOfOrderness(...).withIdleness(Duration.ofMinutes(1))`，标记空闲分区，让下游忽略它。

> 💡 面试金句：Flink 1.11 引入了 **idle source（空闲源标记）** 机制。当一个 Source 分区在指定时间内没有数据时，会被标记为 idle，下游算子计算 watermark 时会跳过它。这是解决"多分区数据不均导致 watermark 卡住"的标准方案。

#### 2. 水位线延迟设置过大

**原因**：`forBoundedOutOfOrderness(Duration.ofHours(1))` 设了 1 小时延迟，意味着 watermark 比实际事件时间慢 1 小时，窗口要等 watermark 超过窗口结束时间 1 小时后才会触发。

**解决方案**：根据实际数据乱序程度合理设置 out-of-orderness，不要设得太大。

#### 3. 多源流 watermark 被慢流拖住

**原因**：双流 join 或 union 时，watermark 取所有输入流的最小值。如果其中一个流数据很慢（时间戳滞后），会拖住整个 watermark。

**解决方案**：
- 如果慢流确实慢，接受这个现实（数据一致性优先）
- 如果是空闲流，用 withIdleness 标记
- 如果两条流时间戳差很大，考虑是否应该用窗口 join 而不是 interval join

#### 4. 数据时间戳乱序严重

**原因**：数据乱序非常严重，迟到数据很多，watermark 推进缓慢。

**解决方案**：
- 调大 out-of-orderness（但会增加延迟）
- 用侧输出流处理迟到数据（`sideOutputLateData`）
- 设置 `allowedLateness` 让迟到数据也能触发窗口更新

### 7.3 排查步骤

1. 看 UI 上 Source 算子的 `currentInputWatermark` / `currentOutputWatermark`，确认是哪个环节卡住
2. 如果 Source 端 watermark 就不推进，检查是否有空闲分区（开 idle 标记试试）
3. 如果 Source 正常但下游不推进，检查多流 join/union 的场景
4. 确认 watermark 策略的延迟设置是否合理

---

## 八、状态持续膨胀 / 状态泄漏 ⭐⭐

### 8.1 现象

- Checkpoint 的 State Size 持续增长，只增不减
- CP 耗时越来越长，最终超时失败
- TM 内存使用量持续上升
- 严重时 OOM

### 8.2 五大原因

#### 1. 状态未配置 TTL（最常见）

**原因**：Keyed State 没有配置 State TTL，状态永远不会过期，key 越来越多，状态只增不减。

**典型场景**：
- 用 ValueState 存用户信息，用户量不断增长但从不清理
- 用 MapState 存维度数据，key 越来越多

**解决方案**：配置 State TTL，设置合理的过期时间。

```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.hours(24))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
```

#### 2. TTL 配置不合理（太长）

**原因**：配了 TTL 但过期时间太长（如 30 天），状态积累量超过预期。

**解决方案**：根据业务需求重新评估 TTL 时长，不要盲目设置得过长。

#### 3. 窗口未触发 / 会话窗口永不关闭

**原因**：
- 窗口一直没到触发条件（如 watermark 不推进，参见第七章）
- 会话窗口：如果数据一直来（会话间隙内），会话永远不会结束，状态永不清理

**解决方案**：
- 排查 watermark 推进问题
- 会话窗口设置最大会话时长（`withMaxSessionTime`，Flink 1.15+ 支持），防止无限会话
- 确认窗口数据是否正常

#### 4. ListState / MapState 无限 append

**原因**：代码中对 ListState 或 MapState 只加不减，元素越来越多。

**典型场景**：
- 用 ListState 缓存历史数据，但从不清理
- 用 MapState 存统计维度，key 无限增长

**解决方案**：
- 配合 TTL 使用（注意 TTL 对 ListState 是按整个 list 过期，不是按元素）
- 代码中主动清理（定期 trim、按时间淘汰）
- 考虑是否应该用窗口而不是常驻状态

#### 5. 广播状态不清理

**原因**：Broadcast State 是 Operator State，没有 TTL 机制，需要手动清理。如果不断往广播状态里写数据但不清理，会持续膨胀。

**解决方案**：在 `BroadcastProcessFunction` 中根据业务逻辑定期清理过期的广播状态。

### 8.3 排查方法

1. **看 CP 大小趋势**：Checkpoints 面板 State Size 是否只增不减
2. **看各子任务状态分布**：Subtasks 列表各 subtask 状态大小，判断是整体膨胀还是倾斜
3. **检查代码中的状态使用**：所有 StateDescriptor 是否都配了 TTL
4. **分析状态内容**：如果用 RocksDB，可以用 RocksDB 工具查看 sst 文件中的 key 分布，找出哪些 key 最多

---

## 九、Exactly-Once 语义失效 / 数据不一致 ⭐⭐

### 9.1 现象

- 明明配置了 `CheckpointingMode.EXACTLY_ONCE`，但结果数据出现重复或丢失
- 故障恢复后数据对不上
- 业务上发现统计值偏大（重复）或偏小（丢失）

### 9.2 五大原因

#### 1. Source 端不支持重放

**原理**：Flink 的 exactly-once 依赖 Source 端能**重放数据**（从保存的 offset 重新消费）。如果 Source 不支持重放，故障恢复后就会丢数据或重复消费。

**不支持重放的 Source**：
- Socket 流
- 自定义 Source 未实现 CheckpointedFunction
- 某些消息队列不支持按 offset 消费

**支持重放的 Source**：
- Kafka（按 offset 消费）✅
- 文件系统（按 offset / 位置）✅
- Pulsar 等支持 seek 的 MQ ✅

#### 2. Sink 端非事务 / 非幂等

**原理**：Flink 的 exactly-once 要求 Sink 端要么支持**事务提交**（2PC），要么是**幂等写入**。否则故障恢复时可能重复写入。

| Sink 类型 | Exactly-Once 支持 | 原理 |
|---|---|---|
| Kafka Sink | ✅ | TwoPhaseCommitSinkFunction（2PC） |
| JDBC Sink | ⚠️ 需幂等表设计 | 依赖数据库主键去重 |
| Redis Sink | ⚠️ 部分操作幂等 | set 幂等，incr 非幂等 |
| Elasticsearch Sink | ⚠️ 需指定 doc id | 用主键 id 做幂等写入 |
| 文件系统 Sink | ✅ | 写入临时文件，CP 成功后移动 |

> 💡 面试金句：Flink 的 "端到端 exactly-once" 不是 Flink 单方面能保证的，需要 Source 支持重放 + Flink 内部 exactly-once + Sink 支持事务/幂等，三者缺一不可。Flink 自己只能保证内部状态的 exactly-once，端到端需要上下游配合。

#### 3. Checkpoint 失败时数据不一致

**原理**：CP 失败意味着这次快照不完整，不能用来恢复。故障后从上一个成功的 CP 恢复，两个 CP 之间的数据会被重新处理。如果 Sink 端不是 exactly-once，就会出现重复。

**注意**：这不是 Flink 的 bug，是 CP 机制的正常表现——CP 失败不影响运行，但故障恢复只能回到最近成功的 CP。

#### 4. 用户代码有副作用

**原理**：用户代码中直接操作外部系统（写 DB、发消息、调 API），且这些操作不参与 Checkpoint 的两阶段提交。故障恢复后这些操作会被重复执行，导致副作用重复。

**典型场景**：
- 在 MapFunction 中直接写 Redis
- 在 ProcessFunction 中发 HTTP 请求
- 用 Sink 但用的是非事务 Sink 且没有幂等保证

**解决方案**：
- 用 Flink 提供的带 exactly-once 保证的 Connector
- 自定义 Sink 时实现 `TwoPhaseCommitSinkFunction`
- 或者确保外部写入是幂等的（即使重复执行也不影响结果）

#### 5. 两阶段提交（2PC）失败

**原理**：Flink 的 2PC Sink（如 Kafka）依赖以下条件：
- 事务超时时间 > Checkpoint 间隔 + 恢复时间
- 外部系统支持事务
- JobManager 高可用

如果事务超时设得太短，CP 还没完成事务就过期了，恢复后会丢数据或重复。

**典型配置**：Kafka 的 `transaction.timeout.ms` 必须大于 Flink 的 checkpoint 间隔和恢复时间，否则事务可能在 CP 完成前过期。

---

## 十、Kafka Source 消费延迟 / Lag 堆积 ⭐

### 10.1 现象

- Kafka consumer group 的 lag 持续增加
- Flink Source 算子的消费速度低于生产速度
- 数据处理延迟越来越大

### 10.2 原因分类

| 原因 | 特征 | 排查方式 |
|---|---|---|
| **消费能力不足** | 整体 lag 增加，各分区均匀，Source 端 Busy Time 高 | 看 Source 各 subtask 消费速率是否均衡 |
| **反压拖累** | Source 反压 HIGH，下游有瓶颈算子 | Backpressure 面板看反压来源 |
| **分区倾斜** | 个别分区 lag 远大于其他，Source 某 subtask 处理量高 | Kafka Manager / Flink UI Subtasks 看各分区消费 |
| **GC 停顿** | lag 间歇性突增，TM GC 指标异常 | TM 日志看 GC 情况 |
| **Kafka 集群问题** | 所有消费该集群的作业都 lag 增加 | 看 Kafka 集群监控、磁盘/网络 |

### 10.3 解决方案

| 原因 | 解决方案 |
|---|---|
| 消费能力不足 | 增加 Source 并行度（最多等于 Kafka 分区数）、优化反序列化逻辑 |
| 反压拖累 | 解决下游反压问题（参见第一章） |
| 分区倾斜 | 调整 Kafka 分区策略（换更均匀的 partitioner）、下游 rebalance 打散 |
| GC 停顿 | 参见第五章内存优化 |
| Kafka 集群问题 | 扩容 Kafka、优化 Kafka 配置、检查磁盘/网络 |

---

## 十一、迟到数据过多 ⭐

### 11.1 现象

- UI 上 `numLateRecordsDropped` 指标值很高
- 窗口结果不准确（漏掉了迟到数据）
- 业务数据统计值偏小

### 11.2 原因

**根本原因**：数据的乱序程度超过了 watermark 的容忍度（out-of-orderness 设置得太小）。

**常见场景**：
- 客户端数据先缓存后上报（批量上报导致时间戳乱序）
- 网络延迟导致数据到达顺序与事件时间顺序不一致
- 多数据源合并，各源延迟差异大
- out-of-orderness 设置得过小（为了低延迟牺牲了正确性）

### 11.3 解决方案

#### 方案 1：调大 watermark 延迟

最简单直接，增大 `forBoundedOutOfOrderness` 的延迟时间。

- 优点：简单，数据正确性高
- 缺点：窗口延迟增加，结果输出更晚

#### 方案 2：allowedLateness + 侧输出流

设置 `allowedLateness` 允许迟到数据在窗口触发后仍能触发窗口更新，同时把迟到数据侧输出做兜底处理。

```java
stream
    .keyBy(...)
    .window(TumblingEventTimeWindows.of(Time.minutes(10)))
    .allowedLateness(Time.minutes(30))  // 允许迟到30分钟
    .sideOutputLateData(lateTag)        // 超出30分钟的迟到数据侧输出
    .process(...)
```

- 优点：兼顾延迟和正确性，迟到数据也能更新结果
- 缺点：窗口状态会多保留 allowedLateness 时间，状态更大

#### 方案 3：侧输出流兜底 + 二次处理

迟到特别多的数据走侧输出流，单独处理（比如写入另一个 topic 或做补偿计算）。

- 优点：不影响主流性能，迟到数据有兜底
- 缺点：需要额外的处理链路

### 11.4 选型建议

| 场景 | 推荐方案 |
|---|---|
| 延迟要求不高，数据准确性优先 | 调大 watermark 延迟 |
| 延迟要求高，但偶尔有迟到数据 | allowedLateness + 侧输出 |
| 迟到数据比例很低，但不能丢 | 侧输出流 + 补偿机制 |
| 完全不能接受数据丢失 | 调大延迟 + allowedLateness + 侧输出三重保障 |

---

## 十二、常见面试题（场景题专题）

### Q1：生产环境 Flink 作业出现反压，你怎么排查和解决？

**答题框架：先定位瓶颈 → 再分析根因 → 最后给方案。**

第一步，定位瓶颈算子。打开 Flink UI 的 Backpressure 面板，从最下游的 Sink 开始往上游找，找到反压从 OK 变 HIGH 的交界点，交界点的下游算子就是瓶颈。

第二步，分析根因。点开瓶颈算子的 Subtasks 列表：
- 如果各 subtask 处理量差异很大 → 数据倾斜
- 如果 Busy Time 都很高 → 算子计算慢或资源不足
- 如果 Busy Time 不高但反压严重 → 外部 IO 瓶颈（Sink 写入慢或维表查询慢）
- 再看 TM 的 GC 指标 → GC 停顿问题
- 看机器资源指标 → CPU/内存/网络/磁盘瓶颈

第三步，对症下药。
- 数据倾斜 → 两阶段聚合、热点 key 拆分、rebalance
- 算子慢 → 优化代码、增大并行度
- IO 瓶颈 → 异步 IO、批量写入、本地缓存
- GC 问题 → 调大内存、切 RocksDB、优化对象创建
- 资源不足 → 扩容

### Q2：Flink 数据倾斜怎么处理？说一个你实际解决的案例。

先讲怎么识别（UI 上看 subtask 处理量不均），再讲常见场景和方案：
1. **keyBy 聚合倾斜**：用两阶段聚合，加随机前缀打散后局部聚合，再去前缀全局聚合
2. **Kafka 分区不均**：下游 rebalance 重分区，或调整 Kafka 分区策略
3. **窗口大 key 倾斜**：用增量聚合（AggregateFunction）减少状态，配合两阶段
4. **双流 Join 热点**：热点 key 加盐打散，另一条流复制 N 份对应 join
5. **维表 Join 热点**：广播小维表 + 本地缓存

案例可以说：某用户行为统计作业，因为有几个大 V 用户数据量特别大导致 keyBy 后倾斜，用两阶段聚合（给 userId 加 0~9 随机前缀）后，倾斜 subtask 的处理量降到了原来的 1/10，整体吞吐提升了 5 倍。

### Q3：Flink 作业频繁重启，你怎么排查？

先看重启策略和重启频率，然后按以下路径排查：
1. **看异常栈**（Exceptions 面板）：有明确异常 → 是代码问题还是外部依赖问题
2. **看 TM 状态**：有没有 TM 丢失 → 有就查 TM 日志（OOM/GC/磁盘/容器驱逐）
3. **看 Checkpoint**：CP 失败次数是否达阈值 → 查 CP 失败原因
4. **看资源指标**：CPU/内存/磁盘/网络是否异常
5. **看外部依赖**：Kafka/HDFS/DB 是否抖动

判断是代码问题还是环境问题：看异常栈是否指向用户代码、是否稳定复现、是否多个作业同时受影响。

### Q4：Checkpoint 一直失败怎么办？

看 Checkpoints 面板的 History 列表，主要看三个指标：
1. **Alignment Time（对齐时间）**：长 → 反压导致 → 解决反压或开非对齐 CP
2. **State Size（状态大小）**：大 → 状态膨胀 → TTL/增量CP/RocksDB
3. **Sync/Async Duration**：
   - Sync 长 → RocksDB 快照慢 → 调 RocksDB 配置
   - Async 长 → 存储 IO 慢 → 优化 HDFS/S3

再看失败原因列（Failure Cause），一般能直接给出错误信息。

### Q5：窗口不触发是什么原因？怎么排查？

窗口不触发本质上是 watermark 没推进到窗口结束时间。常见原因：
1. **空闲 Source 分区**：某分区没数据，watermark 卡住。解决：用 withIdleness 标记空闲源
2. **watermark 延迟设太大**：窗口结束后还要等很久才触发。解决：合理设置延迟
3. **多流 join 被慢流拖住**：watermark 取最小值，慢流拖慢整体。解决：空闲标记或评估是否应该 join
4. **数据时间戳有问题**：时间戳提取错误，导致 watermark 不对。解决：检查时间戳分配器

排查：从 Source 开始往下游看各算子的 `currentOutputWatermark` 指标，找到 watermark 卡住的位置。

### Q6：Flink 的端到端 exactly-once 是什么意思？怎么实现的？

端到端 exactly-once 指的是数据从 Source 到 Sink 整个链路中，每条数据**只被处理一次，结果不重复不丢失**。

但 Flink 不能单方面保证，需要三部分配合：
1. **Source 支持重放**：能从保存的 offset 重新消费（如 Kafka）
2. **Flink 内部 exactly-once**：通过 Checkpoint + Chandy-Lamport 算法，保证内部状态的一致性
3. **Sink 支持事务或幂等**：
   - 事务型：TwoPhaseCommitSinkFunction（2PC），CP 成功才提交事务
   - 幂等型：写入操作重复执行结果不变（如 set 操作、主键去重）

所以面试时要说清楚：Flink 自己保证的是**内部状态的 exactly-once**，端到端需要上下游配合。

### Q7：状态大小一直涨，怎么判断是正常还是泄漏？

看 Checkpoint Size 的趋势：
- **正常累积**：周期性涨跌，窗口触发后大小回落。比如天级窗口，每天凌晨触发后状态变小
- **状态泄漏**：持续单调递增，只增不减

常见泄漏原因和解决：
- 没配 TTL → 配置 State TTL
- ListState/MapState 只加不减 → 代码中主动清理，或用 TTL
- 广播状态不清理 → 手动管理清理逻辑
- 会话窗口永不关闭 → 设最大会话时长

### Q8：TaskManager 挂了怎么排查？

1. 先看 TM 日志有没有 OOM 异常（堆内/堆外/Metaspace/Native）
2. 看 GC 日志，是不是 Full GC 太久导致心跳超时
3. 看系统日志，是不是被 OOM Killer 杀掉了（容器内存超了）
4. 看磁盘是不是满了（RocksDB 或日志写满）
5. 看是不是容器被驱逐了（K8s/YARN 资源不足）
6. 看网络是不是有问题（心跳超时）

口诀：**先看日志、再看指标、最后查基础设施**。

### Q9：OOM 有哪几种？怎么区分？

四种：
1. **堆内 OOM**：`Java heap space`，常见于 HashMap 状态、大对象、窗口全量聚合
2. **堆外 OOM**：`Direct buffer memory`，网络内存不足或用户代码直接分配堆外内存
3. **Metaspace OOM**：`Metaspace`，类元数据不足，动态类加载过多
4. **Native OOM**：进程直接消失或报 `bad_alloc`，RocksDB 状态太大或容器内存超限

区分方式主要看日志中的错误信息关键字，以及 UI 上的内存指标趋势。

### Q10：Flink 配置了 exactly-once，但结果还是重复，可能是什么原因？

从三个环节排查：
1. **Source 端**：Source 是否支持重放？不支持的话故障恢复可能丢数据或重复
2. **Flink 内部**：Checkpoint 是否正常？如果 CP 频繁失败，恢复后从上一个成功 CP 重放，中间数据会重复处理
3. **Sink 端**：Sink 是否支持 exactly-once？
   - 用的是事务 Sink 吗？事务超时配置是否足够？
   - 用的是幂等 Sink 吗？幂等键是否正确？
4. **用户代码副作用**：代码中是否有直接写外部系统且不参与 CP 的操作？

最常见的是 Sink 端没用事务/幂等，或者用户代码中有副作用操作。

### Q11：Kafka Source 消费 Lag 很大怎么排查？

1. 先看 Flink 作业是否有反压 → 下游慢拖累 Source → 解决反压
2. 看各分区 lag 是否均匀 → 不均匀 = 分区倾斜 → rebalance 或调整分区
3. 看 Source 算子的 Busy Time → 都高 = 消费能力不足 → 增加并行度
4. 看 TM 的 GC 和内存 → GC 频繁导致消费间歇性卡住 → 优化 GC
5. 看 Kafka 集群本身是否有问题 → 所有消费者都 lag → 排查 Kafka 集群

### Q12：迟到数据怎么处理？

三种方案，按场景选择：
1. **调大 watermark 延迟**：最简单，延迟换正确性，适合对延迟不敏感的场景
2. **allowedLateness + 侧输出**：允许迟到数据触发窗口更新，超出的侧输出兜底，兼顾延迟和正确性
3. **侧输出流二次处理**：迟到数据单独处理或补偿，适合迟到数据比例低但不能丢的场景

选型时要权衡**正确性、延迟、状态大小**三者的关系。

---

## 十三、资料勘误与重点提醒

1. ⚠️ **反压是结果不是原因**：反压面板全红不代表所有算子都慢，真正的瓶颈往往只有一个。定位时一定要从下游往上游找，不能从上往下找。很多资料说"反压的算子就是慢的算子"，这是错的——被反压的是上游，慢的是下游。

2. ⚠️ **数据倾斜的解决没有银弹**：不是所有倾斜都能用"加随机前缀"解决。两阶段聚合只适用于可拆分的聚合操作（sum/count/min/max），Join 倾斜要用加盐打散，窗口倾斜要用增量聚合，要区分场景。

3. ⚠️ **两阶段聚合不能直接用于求平均**：求平均值不能简单地"局部平均 → 全局平均"（加权不对）。正确做法是局部存 sum + count，全局再 sum(sum)/sum(count)。很多资料一笔带过，面试时容易被追问。

4. ⚠️ **Flink 默认重启次数是无限次**：只要启用了 Checkpoint 且没配置重启策略，Flink 默认使用固定延迟重启，次数是 Integer.MAX_VALUE（可以理解为无限重启）。不是"最多重启几次"，这一点很多人误解。

5. ⚠️ **Checkpoint 失败一次不会导致作业失败（默认配置下）**：Flink 1.13+ 默认 `tolerableCheckpointFailureNumber = 0` 表示 CP 失败不导致作业失败（0 是允许失败的次数，但作业失败判定条件是"连续失败次数 > tolerable"还是"失败即失败"需要注意版本差异）。⚠️ 实际要看具体版本：Flink 1.11 引入该参数，含义是"可容忍的连续 CP 失败次数，超过则作业失败"。生产环境建议显式配置一个合理值（如 3 或 5），避免一次 CP 失败就挂掉，也避免无限失败。

6. ⚠️ **状态 TTL 对 ListState 是整个 list 过期，不是按元素过期**：很多人以为 ListState 里每个元素有自己的 TTL，其实整个 ListState 共用一个 TTL（最后一次写入时间算起）。如果要按元素过期，需要自己管理或用 MapState 存时间戳。

7. ⚠️ **端到端 exactly-once 不是 Flink 一个人的事**：很多资料说"Flink 支持 exactly-once"就完了，但实际端到端 exactly-once 需要 Source 可重放 + Flink 内部 CP + Sink 事务/幂等三者配合。面试时只说 Flink 支持是不够的，要讲清楚上下游条件。

8. ⚠️ **watermark 不推进 ≈ 有空闲分区**：多分区 Source 只要有一个分区没数据且没配 idle，下游 watermark 就卡住。这是生产环境非常常见的坑，很多人排查很久才发现是某个空分区拖的。记住 `withIdleness()` 这个配置。

9. ⚠️ **Unaligned Checkpoint 不是万能的**：非对齐 CP 能解决反压下的 CP 超时问题，但代价是 CP 体积变大（多存了 in-flight data），且只支持 EXACTLY_ONCE 模式和并发 CP 数 = 1 的情况。不能一开了之，要评估存储压力。

10. ⚠️ **RocksDB 的状态在堆外，不代表完全不占堆**：RocksDB 本身用 native 内存（托管内存），但 Flink 的状态 API 读写时还是会在堆上创建对象（序列化/反序列化），只是状态数据本身不常驻堆。如果读写频繁，堆内存压力仍然存在。
