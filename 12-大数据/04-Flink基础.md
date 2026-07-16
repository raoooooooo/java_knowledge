# Flink 基础（第1-8章）

> 本文基于《尚硅谷大数据技术之Flink（Java）》整理，面向 Java 后端面试。力求通俗易懂，对原资料中表述不准或过时之处已就地用 ⚠️ 标注修正（详见各章末「资料勘误」）。
>
> 一句话理解 Flink：**Flink 是一个分布式"流处理"引擎，世界观是"万物皆流"--实时数据是无界流，离线数据是有界流（批是流的特例）；它把计算所需的中间状态放在本地内存里（有状态计算），来一条数据就处理一条，做到毫秒级低延迟 + 任意规模扩展 + 结果精确一次。**

---

## 目录

1. [初识 Flink](#1-初识-flink)
2. [快速上手](#2-快速上手)
3. [部署](#3-部署)
4. [运行时架构](#4-运行时架构)⭐
5. [DataStream API 基础](#5-datastream-api-基础)
6. [时间与窗口](#6-时间与窗口)⭐⭐
7. [处理函数](#7-处理函数)
8. [多流转换](#8-多流转换)

---

## 1. 初识 Flink

### 1.1 核心概念

**Flink 是什么**

Apache Flink 是分布式流处理引擎，官网一句话定位：**数据流上的有状态计算（Stateful Computations over Data Streams）**。用于对无界和有界数据流进行有状态计算，能跑在所有常见集群环境，以内存速度和任意规模执行。两大能力：**速度快**（毫秒级延迟）、**可扩展**（任意规模）。

> 通俗类比：传统数据库像"仓库管理员"--每次来料都先翻远程账本再处理；Flink 像"流水线工人"--把中间状态直接放在手边内存（本地状态），来一条处理一条，不用反复查远程库。

**无界流 vs 有界流**

| | 无界数据流 | 有界数据流 |
|---|---|---|
| 特点 | 有头无尾、持续不断 | 有明确起止 |
| 处理 | 来一条处理一条，需保证顺序 | 可等齐再处理 |
| 对应 | 流处理（实时） | 批处理（离线） |

Flink 世界观：**万物皆流**，批是流的特例。

**数据处理架构演变（4 代）**

1. **传统事务处理**：应用直接读写远程数据库，数据量大时表重构和 SQL 调优痛苦。
2. **有状态流处理（Storm）**：状态存本地内存、检查点持久化。速度快但 ⚠️ 无法保证 exactly-once、吞吐低。
3. **Lambda 架构**：批层 + 速度层双管齐下，流算近似、批修正。**致命缺点**：要维护两套语义等价的代码（批/流 API 完全不同），运维成本翻倍。
4. **新一代流处理器（Flink）**：单套系统搞定 Lambda 两套的活，解决乱序 + exactly-once + 高吞吐低延迟。

**Flink 核心特性**

高吞吐低延迟（每秒百万事件、毫秒级）、事件时间处理乱序、exactly-once 一致性、丰富 connector、高可用、可热更新迁移作业。

**Flink vs Spark（高频考点）**

| 维度 | Spark | Flink |
|---|---|---|
| 世界观 | 万物皆批，流 = 微批次（micro-batch） | 万物皆流，批 = 有界流 |
| 数据模型 | RDD / DStream（小批集合） | DataFlow 事件流（Google DataFlow 模型） |
| 执行 | DAG 划 Stage，stage 间 shuffle | 标准流式，事件直接发往下游 |
| 流延迟 | 秒级（微批） | 毫秒级 |

⚠️ **常见误区修正**：
- 「Spark 不能做流处理」--不准确。Spark 2.x 的 **Structured Streaming** 已支持低延迟和 exactly-once；Spark 2.3 的 **Continuous Processing** 模式在 at-least-once 下可达 1ms 延迟。正确表述：「传统 Spark Streaming(DStream) 是微批、秒级延迟，Flink 在原生流式 + 状态管理 + 事件时间上更优」。
- 「Flink 不能做批」--错误。Flink 1.12 起流批一体，DataSet API 软弃用，用 DataStream API + `execution.runtime-mode=BATCH` 即可批处理。

**分层 API（从下到上）**

```
Process Function（最底层，有状态流，嵌在 DataStream API 中）
   ↑
DataStream API（核心，流批统一）   DataSet API（已软弃用⚠️）
   ↑
Table API（声明式，关系模型，有优化器）
   ↑
SQL（最高层抽象）
```

⚠️ 资料仍以 DataSet API 教批处理，但 **Flink 1.12+ 起 DataSet 处于 soft deprecated**，推荐 DataStream API 一套代码处理流和批。

### 1.2 常见面试题

**Q1：Flink 和 Spark 最本质的区别？**
世界观相反--Spark 以批为本，流是微批次；Flink 以流为本，批是有界流。底层 Spark 数据模型是 RDD/DStream（小批集合），Flink 是 DataFlow 事件流。Spark 流处理秒级延迟，Flink 毫秒级。

**Q2：什么是 Lambda 架构？它有什么问题？Flink 怎么解决？**
Lambda = 批层 + 速度层，流算近似、批修正。问题是要维护两套语义等价的代码、API 不同、运维复杂。Flink 单套系统同时保证低延迟和准确性（事件时间 + exactly-once），替代两套系统。

**Q3：Flink 的"有状态计算"是什么意思？为什么重要？**
状态是处理中需保存的中间数据（累加器、窗口数据等）。有状态计算把这些状态存本地内存而非远程库，每来一条读取/更新本地状态。重要性：避免每条查远程库的 IO 开销，大幅提升吞吐和延迟；通过 checkpoint 持久化保证容错。

**Q4：无界流和有界流的区别？**
无界流有头无尾、持续不断，必须连续处理且保证顺序；有界流有明确起止，可等齐再处理、不严格保序。无界对应流处理，有界对应批处理。

---

## 2. 快速上手

### 2.1 核心概念

**基本开发流程（四步走 + 触发）**：`env -> source -> transform -> sink`，最后 `execute()` 触发。

```
1. 创建执行环境
2. 读取数据源（source：readTextFile / socketTextStream / Kafka）
3. 转换操作（transformation：flatMap / map / keyBy / sum）
4. 输出结果（sink：print / writeAsText）
5. 触发执行（流处理必须 env.execute()）
```

> 通俗类比：做菜--备厨房(env)、去菜场拿食材(source)、切炒烹炸(transform)、装盘上桌(sink)。

**批处理 vs 流处理代码差异**

| 维度 | 批处理（DataSet） | 流处理（DataStream） |
|---|---|---|
| 执行环境 | `ExecutionEnvironment` | `StreamExecutionEnvironment` |
| 分组 | `groupBy(0)` | `keyBy(t -> t.f0)` |
| 需 execute | 不需要（print 自动触发） | **必须 `env.execute()`** |
| 输出 | 每个单词只输出最终一次 | 每来一条数据输出一次累计值 |

⚠️ 流处理输出前缀的数字（如 `2> (hello,1)`）是**并行子任务（线程）编号**，不是数据顺序。多线程并行所以输出顺序可能与输入不同。

⚠️ **新版推荐**：Flink 1.12+ 流批一体后，批处理不再需单独写 DataSet 代码，用 DataStream API + 提交参数 `-Dexecution.runtime-mode=BATCH` 即可。

### 2.2 常见面试题

**Q1：Flink 程序的基本开发流程？**
env -> source -> transform -> sink -> execute。

**Q2：为什么流处理需要 `env.execute()`，批处理不需要？**
批处理是数据集计算，print 等操作自动触发整个 DAG 执行。流处理是事件驱动、持续运行，需显式 `execute()` 启动作业并阻塞等待数据，否则 main 一结束程序就退出。

**Q3：流处理输出前的数字代表什么？为什么顺序和输入不一致？**
是并行子任务（线程）编号。Flink 是分布式引擎，本地用多线程模拟集群，不同线程并行处理所以输出顺序与输入不同。

---

## 3. 部署

### 3.1 核心概念

**三种部署模式（核心考点）**：区分维度是①集群生命周期；②main 在哪执行。

| 模式 | 集群生命周期 | main 在哪执行 | 资源隔离 | 适用场景 |
|---|---|---|---|---|
| **会话 Session** | 集群独立于作业（先启集群后提交） | 客户端 | 共享 | 规模小、短时大量作业 |
| **单作业 Per-Job** | 每作业一个集群，作业完集群关 | 客户端 | 独享 | 生产环境首选 |
| **应用 Application** | 每应用一个集群（可含多作业） | **JobManager** | 独享 | 客户端压力大时 |

通俗类比：
- **会话** = 公共食堂，多家共用厨房，先开门等客来。缺点：满了进不来，一家坏影响所有。
- **单作业** = 私厨包场，每家单独开厨房，做完关掉。隔离好但开销大。
- **应用** = 把主厨请进厨房做菜（main 在 JM 执行），客户端只下单，不再占带宽下依赖。

⚠️ **单作业模式**：Flink 本身无法直接以 Per-Job 启动，**必须借助 YARN/K8s**。Standalone 不支持 Per-Job。

⚠️ **应用模式**：与单作业像，但区别是「一个应用（即使含多个作业）只创建一个集群，且 main 在 JM 执行」，解决了客户端下载依赖的带宽压力。

⚠️ **演进加分项**：Flink 1.15+ 起 **Per-Job 模式已被官方弃用**，推荐 Application 模式替代。

**部署方式（资源提供者）**

1. **Standalone**：Flink 自带，不依赖外部资源管理器。最简单但无自动扩展。支持会话/应用模式，**不支持单作业**。
2. **YARN**：国内最主流，由 YARN RM 分配容器，按需动态分配 TM。支持三种部署模式。
3. **K8s**：容器化部署，原理类似 YARN，近年兴起。

**Standalone HA vs YARN HA（易混）**

- **Standalone HA**：同时启动多个 JobManager，一主多备，靠 ZooKeeper 选举。
- **YARN HA**：**只启动一个 JobManager**，挂了由 YARN 重新拉起，靠 YARN 重试次数实现。⚠️ `flink-conf.yaml` 的 `yarn.application-attempts` 要**小于** `yarn-site.xml` 的上限。

### 3.2 常见面试题

**Q1：Session、Per-Job、Application 三种模式的区别？**
Session 先启集群后提交、多作业共享资源；Per-Job 每作业一个集群、资源独享、作业完集群关；Application 每应用一个集群、main 在 JM 执行、避免客户端带宽压力。Per-Job 和 Application 都是提交后才创建集群。

**Q2：为什么有了 Per-Job 还要 Application 模式？**
Per-Job 和 Session 的 main 都在客户端执行，客户端要下载依赖、把二进制发给 JM，多作业用同一客户端压力很大。Application 模式直接在 JM 上执行 main，节省带宽。

**Q3：Standalone 支持哪些部署模式？**
只支持会话模式和应用模式，**不支持单作业模式**（需外部资源管理器动态启停集群）。

**Q4：YARN 的高可用和 Standalone 的高可用有什么区别？**
Standalone HA 是多 JM 同时启动、主备切换（靠 ZK 选举）；YARN HA 是单 JM，挂了由 YARN 重新拉起（靠 YARN 重试次数）。

---

## 4. 运行时架构 ⭐

### 4.1 核心概念

**整体架构**：两大核心组件 + 客户端。

- **JobManager（Master）**：管理调度，一个集群一个（HA 下有主备）
- **TaskManager（Worker）**：执行任务，一个或多个
- **Client**：不算处理系统一部分，只负责提交作业（调用 main、生成数据流图、发给 JM）

**JobManager 的四个组件（高频考点）**

| 组件 | 职责 | 数量 |
|---|---|---|
| **JobMaster** | 处理单个作业（与 Job 一一对应）；接收 JobGraph -> 转 ExecutionGraph -> 向 RM 申请资源 -> 分发任务给 TM；运行中协调 checkpoint | 每作业一个 |
| **ResourceManager** | 管理 task slots 分配/回收；资源不够时向 YARN/K8s 申请新 TM；停掉空闲 TM | 集群一个 |
| **Dispatcher** | 提供 REST 接口接收作业提交；为每个新作业启动 JobMaster | 可选 |
| **WebMonitorEndpoint** | 提供 Web UI 和 REST 监控接口 | - |

⚠️ **资料勘误**：书中 JobManager 只讲 3 个组件，Web UI 实际由 WebMonitorEndpoint 提供，面试答到 4 个更完整。
⚠️ **区分两个 ResourceManager**：Flink 内置的管 task slots；YARN 的管容器。名字一样但层级不同。

**数据流图（Dataflow Graph）**

程序 = 一连串「算子」构成的管道，数据如水流过。三部分：Source（源）-> Transformation（转换）-> Sink（输出）。

⚠️ **易错点**：不是每个方法都是算子。`keyBy` **不是算子**，是数据分区操作（返回 KeyedStream 而非 SingleOutputStreamOperator），所以**不能对 keyBy 设并行度**，且会断开算子链。

**并行度（Parallelism）**

算子可拆成多个并行「子任务」，子任务数 = 并行度。程序并行度 = 所有算子并行度的最大值。

**并行度设置优先级（从高到低）**：
1. 代码中算子单独 `setParallelism()` -- 最高
2. 代码中 `env.setParallelism()` 全局
3. 提交时 `-p` 参数
4. 配置文件 `parallelism.default` -- 最低

⚠️ 实践：代码中只针对算子设并行度，**不要硬编码全局并行度**，否则无法动态扩容。⚠️ `socketTextStream` 是非并行 Source，无论怎么设并行度都是 1。

**算子链 Operator Chain（核心考点）**

**算子间数据传输两种方式**：
- **一对一（forwarding）**：分区和顺序不变，如 source->map。类似 Spark 窄依赖。
- **重分区（Redistributing）**：分区改变，如 keyBy、rebalance。类似 Spark 宽依赖/shuffle。

**算子链合并条件（面试必背）**：

⚠️ 资料只说「并行度相同的一对一算子可链接」不够完整，**完整 4 条件**：
1. **并行度相同**
2. **数据传输是一对一（forwarding）**，非重分区（keyBy/rebalance/rescale 会断链）
3. **同一 slot 共享组**（默认所有算子在 "default" 组）
4. 算子 chaining 策略允许（未被 `disableChaining` 禁用）

**举例**：source(P2)->flatMap(P2)->keyBy->sum(P2)->sink(P2)
- source->flatMap：一对一 + 并行度同 -> **合并**
- flatMap->keyBy：keyBy 是重分区 -> **断链**
- sum->sink：一对一 + 并行度同 -> **合并**

⚠️ **为什么合并**：减少线程切换和缓冲区数据交换，降低延迟、提升吞吐。控制：`disableChaining()`（当前算子不参与链）、`startNewChain()`（从当前算子起新链）。

**四层调度图**

| 层级 | 生成位置 | 内容 |
|---|---|---|
| **StreamGraph（逻辑流图）** | 客户端 | 代码直接映射的 DAG，节点=算子 |
| **JobGraph（作业图）** | 客户端 | StreamGraph 优化版，**合并算子链**为任务节点 |
| **ExecutionGraph（执行图）** | JobMaster | JobGraph 并行化版，**按并行度拆子任务**，调度核心数据结构 |
| **PhysicalGraph（物理图）** | TaskManager | 部署后的物理执行情况 |

⚠️ **口诀**：客户端生成 StreamGraph + JobGraph（算子链优化）；JobMaster 生成 ExecutionGraph（并行化）；TaskManager 形成 PhysicalGraph（物理执行）。

**任务槽 Task Slots 与槽共享 Slot Sharing（核心考点）**

**任务槽**：
- TM 是 JVM 进程，可多线程并行执行多个子任务
- slot = TM 计算资源的固定大小子集，**每个 slot 独占一份内存**
- slot 数量 = TM 能并行执行任务数上限
- ⚠️ **slot 只隔离内存，不隔离 CPU**（高频考点）

**槽共享**：
- 默认开启：**同一作业不同算子节点的子任务可共享同一 slot**
- 但**同一算子节点的并行子任务不能共享**（必须分到不同 slot 才能数据并行）
- 结果：一个 slot 可放整个作业的完整 pipeline（source->...->sink 各一个子任务）

**为什么允许槽共享**：不同算子资源占用差异大（source/map 轻量、window 重），不共享时轻量算子 slot 空闲、重算子 slot 忙死，资源浪费；共享后重活平均分到所有 slot，负载均衡。

⚠️ **关键结论**：开启槽共享后，**作业所需 slot 数 = 所有算子并行度的最大值**（不是子任务总数）。

**slot vs 并行度**：slot 是静态概念（TM 并发能力，`taskmanager.numberOfTaskSlots`）；parallelism 是动态概念（实际使用的并发，`parallelism.default`）。

### 4.2 常见面试题

**Q1：JobManager 包含哪些组件？各自职责？**
① JobMaster--处理单个作业，接收 JobGraph 转 ExecutionGraph、申请资源、分发任务、协调 checkpoint；② ResourceManager--管理 task slots 分配回收、资源不足申请新 TM；③ Dispatcher--提供 REST 接口接收作业、启动 JobMaster；④ WebMonitorEndpoint--提供 Web UI 和 REST 监控。

**Q2：算子链什么条件下合并？什么情况下断开？**
合并条件：① 并行度相同；② 一对一（forwarding）非重分区；③ 同一 slot 共享组；④ 未被 disableChaining。断开：遇到 keyBy/rebalance 等重分区、并行度变化、主动 disableChaining 或 startNewChain。

**Q3：JobGraph 和 ExecutionGraph 的区别？分别在哪生成？**
JobGraph 是 StreamGraph 优化（合并算子链）后的版本，在**客户端**生成；ExecutionGraph 是 JobGraph 并行化版本（按并行度拆子任务、明确数据传输），在 **JobMaster** 生成，是调度核心数据结构。

**Q4：task slot 和并行度有什么区别和关系？**
slot 是静态概念（TM 并发能力）；并行度是动态概念（实际使用的并发）。并行度 ≤ slot 总数才能正常执行。开启槽共享后，作业所需 slot 数 = 所有算子并行度的最大值。

**Q5：为什么 Flink 要做槽共享？**
不同算子资源占用差异大，不共享时轻量算子 slot 空闲、重算子 slot 忙死。共享后重活平均分配、负载均衡，一个 slot 保存完整 pipeline，单 TM 故障不影响其他。注意同一算子的并行子任务不能共享 slot。

---

## 5. DataStream API 基础

### 5.1 核心概念

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

### 5.2 常见面试题

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

## 6. 时间与窗口 ⭐⭐

> 本章是 Flink 全书最重要、面试最高频的章节。时间语义、水位线、窗口、迟到数据处理几乎必问。

### 6.1 核心概念

**时间语义（三种时间）**

| 时间语义 | 定义 | 特点 | 场景 |
|---|---|---|---|
| **处理时间 Processing Time** | 机器系统时间 | 最简单、延迟最低；各节点时钟不统一、结果不确定 | 实时性极高、准确性不高 |
| **事件时间 Event Time** | 数据在源头发生的时间（自带时间戳） | 结果正确、可重放；需水位线推进有延迟 | 绝大多数业务（PV/UV、统计、监控） |
| **摄入时间 Ingestion Time** | 数据进入 Flink 的时间 | 折中，相当于 Source 处理时间当事件时间 | 历史概念，已弱化 |

> 通俗类比（星球大战）：电影上映时间 = 处理时间；电影故事背景时间 = 事件时间。数据处理中我们更关心数据本身发生的时间，所以事件时间更常用。

**为什么事件时间更重要**：23:59:59 产生的数据可能 00:00:01 才被处理，按处理时间会错分到第二天，按事件时间永远归属当天窗口。事件时间支持重放。⚠️ Flink 1.12 起**默认时间语义改为事件时间**。

**乱序问题**：网络延迟导致到达顺序与产生顺序不一致（7 秒数据晚于 9 秒到达）。处理时间无法应对，必须事件时间 + 水位线。

**水位线 Watermark（重中之重）**

**(1) 为什么需要水位线**

事件时间下，用数据时间戳定义"逻辑时钟"：来一条数据把它的时间戳当当前时间。但有问题：①上游 reduce 后数据变少，时钟推进不精细；②数据只发给一个下游子任务，其他子任务时钟无法推进。解决：把"当前时间进展"作为特殊数据（水位线）广播给下游。

**(2) 水位线是什么（通俗解释）**

> **水位线 = "我预估在这个时间点之前的数据都到齐了，之后不会再有更早的数据来了"**

通俗类比：团队团建等人到齐。说好 9 点发车，有人堵车，于是等到 9:10 再走。"把表调慢 10 分钟"就是水位线的延迟。

特性：
- 插入数据流中的一个标记，是一条特殊数据，内容是一个时间戳
- 基于数据的时间戳生成
- 时间戳**单调递增**（时光不能倒流）
- `Watermark(t)` 表示：事件时间已到 t，**时间戳 ≤ t 的数据全部到齐**
- 可设延迟保证正确处理乱序

**(3) 如何生成水位线**

总体原则：完美水位线可望不可即，实际是"低延迟 vs 正确性"的权衡--等得越久越不漏数据，但实时性越差。

生成方式：`assignTimestampsAndWatermarks(WatermarkStrategy)`，包含：
- **TimestampAssigner**：从数据某字段提取时间戳
- **WatermarkGenerator**：`onEvent()`（每条数据调）、`onPeriodicEmit()`（周期性调，默认每 200ms 发一次）

⚠️ **易错点（生成周期）**：水位线的"周期"是**处理时间**（系统时间），不是事件时间。用事件时间定义周期会死循环。

**两种内置生成器**：

| 生成器 | 适用 | 水位线时间戳 |
|---|---|---|
| `forMonotonousTimestamps()` | 有序流 | 最大时间戳 - 1ms |
| `forBoundedOutOfOrderness(Duration)` | 乱序流 | 当前最大时间戳 - 延迟时间 |

⚠️ **关键细节（减 1 毫秒）**：乱序流真正的时间戳是 `maxTimestamp - outOfOrdernessMillis - 1`。减 1 是因为水位线语义是"≤ t 的数据都到齐"，不减 1 会矛盾。

**乱序程度与延迟权衡**：延迟设大 -> 不漏数据但实时性降低；延迟设小 -> 实时性强但可能漏数据。实战网络乱序一般毫秒~百毫秒级，延迟常设秒级。

**(4) 水位线传递机制（⚠️ 面试高频易错点）**

一个下游任务收到多个上游分区的数据，各上游时钟不同步，下游听谁的？

> **核心规则：当前任务的时钟 = 所有上游分区水位线中的最小值（木桶原理）。**

为什么取最小：水位线语义是"之前数据都到齐"。若上游两分区发来 5 秒和 7 秒，取 7 秒表示"7 秒前都到齐"是假的--第一分区才到 5 秒，5~7 秒数据还会来；取 5 秒才安全。

⚠️ **易错点（取最小，不是取最大也不是取平均）**：很多人第一反应是"取最大"或"取平均"，错。必须取所有上游分区水位线的最小值。

**多分区影响**：只要一个上游分区慢（甚至空闲不发数据），下游时钟就被拖住。这是"空闲源（idle source）"问题--Kafka 某分区没数据，整个下游水位线卡住。解决：`withIdleness` 标记空闲源超时后不参与最小值计算。

⚠️ **水位线会被阻塞**：水位线是数据流中的一条记录，排前面的数据没处理完不能"弯道超车"。算子若有耗时计算会阻塞水位线向下传递，导致下游窗口/定时器迟迟不触发。

**(5) 水位线总结**

- 事件时间世界里，水位线是唯一的逻辑时钟，窗口闭合、定时器触发都靠它
- 默认公式：`水位线 = 观察到的最大事件时间 - 最大延迟时间 - 1ms`
- 数据流开始时插入 `-Long.MAX_VALUE`，结束时插入 `+Long.MAX_VALUE`，保证所有窗口闭合、定时器触发

**窗口 Window（重点）**

**(1) 窗口概念与作用**

无界流无穷无尽，不能等所有数据到齐再处理。窗口把无界流切成有界的"数据块"，每块分别聚合、结果只输出一次。

⚠️ **关键认知（窗口不是"框"是"桶"）**：不要把窗口想象成固定位置的"框"（数据只能进一个窗口）。应理解为"存储桶（bucket）"--每个数据按自身时间戳分配到对应桶，到窗口结束时间才计算。迟到数据能进正确窗口。窗口是**数据到达时动态创建**的。窗口区间左闭右开 `[start, end)`，`maxTimestamp = end - 1`。

**(2) 窗口分类**

按驱动类型：
- **时间窗口 Time Window**：以时间点定义，"定点发车"
- **计数窗口 Count Window**：按元素个数截取，"人满发车"，底层用全局窗口实现

按分配规则（⚠️ 三种重点区分）：

| 窗口类型 | 特点 | 参数 | 数据重叠 | 场景 |
|---|---|---|---|---|
| **滚动 Tumbling** | 固定大小、首尾相接、无重叠 | size | 每条只属 1 个窗口 | 每小时/每天 PV |
| **滑动 Sliding** | 固定大小、可重叠 | size + slide | slide<size 时属多个 | 股票 24h 涨跌幅、最近 N 分钟每分钟更新 |
| **会话 Session** | 基于会话分组、长度不定 | gap | 不重叠 | 用户会话行为统计 |

要点：
- 滚动：size 唯一参数，无缝衔接，每条属一个窗口
- 滑动：size 和 slide。slide<size -> 重叠（属 size/slide 个窗口）；slide=size 退化为滚动；slide>size -> 有间隔丢数据，一般不用
- 会话：相邻数据间隔 < gap 同窗口，> gap 开新窗关旧窗；长度不定，底层每来新数据判断合并。⚠️ **会话窗口只能基于时间**，没有"会话计数窗口"

**(3) 窗口 API**

- **按键分区 Keyed**：先 `keyBy()` 再 `.window()`，窗口在多个子任务上基于每个 key 独立计算（推荐）
- **非按键分区**：直接 `.windowAll()`，并行度恒为 1，很少用

窗口算子核心两部分：**窗口分配器**（`.window()`，指定类型）+ **窗口函数**（`.reduce/.aggregate/.process`，定义计算），缺一不可。

**(4) 窗口函数（⚠️ 增量 vs 全窗口）**

| 类型 | 代表 | 工作方式 | 优点 | 缺点 |
|---|---|---|---|---|
| **增量聚合** | `ReduceFunction`/`AggregateFunction` | 每来一条立即与状态聚合，窗口结束直接输出状态 | 高效实时 | 拿不到窗口上下文信息 |
| **全窗口** | `ProcessWindowFunction` | 收集窗口所有数据缓存，窗口结束遍历计算 | 可拿窗口信息、水位线、状态 | 低效（全缓存） |

**Reduce vs Aggregate**：Reduce 输入、状态、输出类型必须一致；Aggregate 输入 IN、累加器 ACC、输出 OUT 三种类型可不同，更灵活（4 方法：`createAccumulator`/`add`/`getResult`/`merge`）。

**选择建议**：
- 只做简单聚合 -> 增量聚合
- 需窗口起止时间、水位线上下文 -> 全窗口
- **最佳实践：两者结合**--`.aggregate(aggFunc, processWindowFunc)`，增量负责计算、全窗口负责包装信息。此时全窗口函数的 Iterable 里通常只有一个元素（增量结果），既高效又灵活

⚠️ 全窗口单独用性能差，不要无脑用 ProcessWindowFunction。

**(5) 窗口生命周期**

1. **创建**：数据驱动，第一个属于某窗口的数据到达才创建，不预先创建
2. **触发计算**：由触发器（Trigger）控制。事件时间窗口默认水位线到 end 触发；计数窗口默认元素数达 size 触发。设了 allowedLateness 时迟到数据也会再次触发
3. **销毁**：默认触发计算后清除状态销毁。⚠️ 只有**时间窗口**有销毁机制，**计数窗口**基于全局窗口**不会被销毁**（触发后清空数据但窗口保留）；设了 allowedLateness 时，真正销毁时间是 `窗口结束时间 + 允许延迟`

其他 API：触发器 `.trigger()`、移除器 `.evictor()`、允许延迟 `.allowedLateness()`、侧输出流 `.sideOutputLateData()`。Trigger 四种返回：CONTINUE / FIRE / PURGE / FIRE_AND_PURGE。**触发计算和窗口销毁可分开**--allowedLateness 利用这点：先 FIRE 输出近似，延迟期内继续 FIRE 更新，最后 PURGE。

**迟到数据处理（三道防线，重点）**

> 迟到数据 = 时间戳小于当前水位线的数据（在水位线之后才到）。只在事件时间语义下讨论。

| 防线 | 机制 | 范围 | 效果 |
|---|---|---|---|
| **第一道：水位线延迟** | `forBoundedOutOfOrderness(delay)` | 全局 | 把表调慢，挡掉大部分乱序 |
| **第二道：allowedLateness** | `.allowedLateness(Time)` | 单个窗口算子 | 窗口触发后不立即销毁，延迟期内迟到数据仍可进窗口、再次触发更新结果（先近似再修正） |
| **第三道：侧输出流** | `.sideOutputLateData(outputTag)` | 单个窗口算子 | 窗口真正关闭后，迟到数据不丢，送侧输出流，需手动合并 |

通俗类比（班车）：
- 第一道（水位线延迟）：把表调慢几分钟，多等一会儿，大部分人赶上
- 第二道（allowedLateness）：准点发车后慢慢开，迟到的人追上来停车开门让他上，有时间限制
- 第三道（侧输出流）：车上高速开走了，留路线和联系方式，迟到的人辗转到目的地后会合

⚠️ **关键区分**：水位线延迟是**全局**的，影响所有定时器和窗口；allowedLateness 是**窗口算子级别**的，只影响该窗口何时真正关闭。叠加效果：水位线到窗口 end -> 触发计算输出近似值 -> allowedLateness 期内迟到数据更新结果 -> 超时窗口销毁 -> 侧输出流兜底。

### 6.2 常见面试题

**Q1：事件时间和处理时间有什么区别？实际用哪个？**
处理时间是机器系统时间，延迟最低但各节点时钟不统一、结果不确定不可重放。事件时间是数据在源头发生的时间（自带时间戳），结果正确、可重放，需水位线推进有少量延迟。实际业务几乎都用事件时间；只有实时性极高、准确性不高的场景才用处理时间。Flink 1.12 后默认事件时间。

**Q2：水位线是什么？怎么生成？**
水位线是插入数据流的时间戳标记，表示"该时间戳之前的数据都已到齐"。生成：`assignTimestampsAndWatermarks(WatermarkStrategy)`，含 TimestampAssigner（提取时间戳）和 WatermarkGenerator（生成水位线）。内置两种：有序流 `forMonotonousTimestamps`（=最大时间戳-1ms）、乱序流 `forBoundedOutOfOrderness(delay)`（=最大时间戳-delay-1ms）。⚠️ 两者都减 1ms（`Watermark(t)` 表示 ≤t 的数据到齐，不能等于已观察的最大时间戳，否则会把该时间戳的数据排除）。默认每 200ms（处理时间）周期性生成。

**Q3：水位线在任务间怎么传递？为什么取最小？**
当前任务为每个上游分区维护"分区水位线"，自身时钟=所有分区水位线最小值（木桶原理）。只有最小值增大时才推进时钟并向下游广播。取最小是因为水位线语义是"之前数据都到齐"--取较大值会承诺了"7 秒前都到齐"但某上游才到 5 秒，承诺就破了。易错点：是取最小不是取最大。

**Q4：滚动、滑动、会话窗口有什么区别？分别用于什么场景？**
- 滚动：固定大小、无缝衔接、每条只属一个窗口，参数只有 size。用于每时段聚合（每小时 PV）
- 滑动：固定大小、可重叠，参数 size+slide，slide<size 时数据属多个窗口。用于"最近 N 分钟每分钟更新"（股票涨跌幅、报警）
- 会话：基于会话超时分组、长度不定、不重叠，参数 gap。用于用户会话行为统计。会话窗口只能基于时间

**Q5：增量聚合函数和全窗口函数有什么区别？怎么选？**
增量聚合（Reduce/Aggregate）每来一条与状态聚合，窗口结束直接输出，高效但拿不到窗口上下文。全窗口（ProcessWindowFunction）缓存所有数据、窗口结束遍历计算，可拿上下文但性能差。选择：简单聚合用增量；需上下文用全窗口；**最佳实践是结合** `.aggregate(aggFunc, processWindowFunc)`。Reduce 要求输入输出类型一致，Aggregate 允许三种类型不同更灵活。

**Q6：Flink 怎么处理迟到数据？三道防线是什么？**
- 第一道：水位线延迟（forBoundedOutOfOrderness），全局把表调慢，挡大部分乱序
- 第二道：allowedLateness，窗口算子级别，触发后不立即销毁，延迟期内迟到数据继续进窗口再次触发更新（先近似再修正）
- 第三道：sideOutputLateData，窗口真正关闭后迟到数据送侧输出流，不丢，需业务侧手动合并。三道层层兜底

**Q7：水位线的延迟和窗口的 allowedLateness 有什么区别？**
水位线延迟是全局的，影响整个应用所有定时器和窗口的触发时机；allowedLateness 是窗口算子级别的，只影响该窗口何时真正销毁。水位线延迟应对"一般性乱序"，allowedLateness 应对"水位线之后还有零星迟到"。

---

## 7. 处理函数

### 7.1 核心概念

**什么是处理函数（ProcessFunction）**

前面学的 map/filter/window 是 DataStream API 的具体算子，而 **ProcessFunction 是更底层的"统一处理"接口**--直面流的三要素：**事件、状态、时间**。

> 通俗理解：基本算子像"专用工具"（map 只能转换、filter 只能过滤），处理函数像"瑞士军刀"，什么都能做，是实现各种自定义逻辑的兜底"大招"。

它继承自 `AbstractRichFunction`，因此：拥有富函数特性（getRuntimeContext 访问状态）；可通过 `TimerService` 访问时间戳、水位线并注册定时器；可将数据输出到**侧输出流**。

**两个核心方法**：
- `processElement(value, ctx, out)`：每来一条数据调一次（必须实现）
- `onTimer(timestamp, ctx, out)`：注册的定时器触发时调用（可选）

**处理函数的 8 种分类（重点记前 4 种）**

| 处理函数 | 基于的流 | 特点 |
|---|---|---|
| **ProcessFunction** | DataStream | 最基本，不支持注册定时器 |
| **KeyedProcessFunction** | KeyedStream | ⭐支持定时器，最常用 |
| **ProcessWindowFunction** | WindowedStream | 全窗口函数 |
| **CoProcessFunction** | ConnectedStreams | 连接两条流后处理 |

⚠️ **关键点**：**只有 KeyedStream 才能注册定时器**。普通 DataStream 的 ProcessFunction 不能注册/删除定时器。

**KeyedProcessFunction 与定时器 Timer/TimerService（⭐重点）**

**TimerService 六个方法**：

| 操作 | 处理时间 | 事件时间 |
|---|---|---|
| 获取当前时间 | `currentProcessingTime()` | `currentWatermark()` |
| 注册定时器 | `registerProcessingTimeTimer(long)` | `registerEventTimeTimer(long)` |
| 删除定时器 | `deleteProcessingTimeTimer(long)` | `deleteEventTimeTimer(long)` |

**定时器核心原理**：
1. **去重机制**：以 **key + 时间戳** 为标准去重，同一个 key 同一时间戳的定时器**只触发一次**
2. **同步调用**：`onTimer()` 与 `processElement()` 同步执行，不会状态并发修改
3. **容错性**：定时器和状态一起存 checkpoint，故障恢复时重建。处理时间定时器恢复时已过期会立即触发
4. **事件时间定时器触发条件**：水位线推进到设定时间戳时触发（不是数据时间戳到达就触发）
5. **性能优化**：可降低时间戳精度减少定时器数量

**定时器典型应用场景**：去重（key+时间戳天然去重）、TopN（注册 `windowEnd+1ms` 定时器等数据到齐再排序）、超时检测（实时对账）、延迟输出。

**ProcessWindowFunction 与 KeyedProcessFunction 的关键区别**：方法是 `process(KEY key, Context ctx, Iterable<IN> elements, Collector<OUT> out)`；**没有 TimerService，不能注册定时器**（需定时用 Trigger）；有 `windowState()` 和 `globalState()` 两种状态。

**侧输出流（Side Output）**：从一条主流产生多条"支流"，且**各支流数据类型可不同**。`ctx.output(OutputTag, value)` 输出，`stream.getSideOutput(outputTag)` 获取。应用：分流、迟到数据兜底。

**TopN 案例（⭐经典思路）**：实时统计最近 10 秒热门 URL Top2，每 5 秒更新。

推荐方案（KeyedProcessFunction + 增量聚合 + ListState）：
```
步骤1：按 url keyBy -> 开滑动窗口 -> AggregateFunction 增量计数 + ProcessWindowFunction 包装窗口信息 -> 输出 UrlViewCount
步骤2：按 windowEnd keyBy -> KeyedProcessFunction：
        - 每来一个结果存入 ListState
        - 注册 windowEnd+1ms 的事件时间定时器
        - 定时器触发时（水位线到达，数据到齐），从状态取出排序输出 TopN
```
巧妙点：利用定时器去重（同一 windowEnd 只触发一次）+ 水位线语义（延迟 1ms 保证数据完整）。

### 7.2 常见面试题

**Q1：ProcessFunction 和普通算子（如 map）有什么区别？**
普通算子功能单一，无法访问时间戳、水位线，无法注册定时器。ProcessFunction 是底层 API，可访问事件、时间、状态三要素，还能输出侧输出流，能实现任意自定义逻辑。map/filter/flatMap 都能用 ProcessFunction 实现。

**Q2：定时器的触发机制是什么？为什么只能在 KeyedStream 上用？**
事件时间定时器在**水位线推进到设定时间戳**时触发；处理时间定时器在**系统时间到达设定时间**时触发。只能在 KeyedStream 上用是因为定时器以 key 为维度管理（底层是 key 的状态），需按 key 分区才能独立触发。定时器有去重：同一 key+时间戳只触发一次，与 processElement 同步执行保证线程安全。

**Q3：定时器如何实现容错？**
定时器作为状态的一种存到 checkpoint。故障恢复时重建；处理时间定时器恢复时已过期会立即触发；事件时间定时器等水位线再次推进到触发时间时触发。

**Q4：ProcessWindowFunction 能注册定时器吗？不能的话怎么办？**
不能。它没有 TimerService。因为窗口本身有触发计算的时间点，一般不需要额外定时。若需要，使用**窗口触发器 Trigger**，其 TriggerContext 提供类似 TimerService 的能力。

**Q5：侧输出流有什么用？和 filter 多次分流有什么区别？**
侧输出流可从一条主流分叉多条流，类型可不同，无需复制流。filter 多次分流会复制多份数据、类型必须一致、效率低。侧输出流是 Flink 1.13 后推荐的分流方式（旧 split 方法已废弃）。

---

## 8. 多流转换

### 8.1 核心概念

**分流（Split）**

将一条流拆成多条独立流。三种方式演进：

| 方式 | 特点 | 状态 |
|---|---|---|
| 多次 filter | 简单但复制多份流 | 可用不推荐 |
| split() | 盖戳拣选不复制流，类型必须一致 | **已废弃** |
| **侧输出流** | 不复制流，类型可不同 | ⭐推荐 |

**合流：Union vs Connect（⭐高频面试题）**

| 维度 | Union（联合） | Connect（连接） |
|---|---|---|
| **数据类型** | **必须相同** | **可以不同** |
| **流数量** | 可合并**多条** | 只能连接**两条** |
| **输出类型** | DataStream（不变） | ConnectedStreams（需再处理） |
| **处理方式** | 直接合并 | 需 CoMapFunction/CoProcessFunction 分别处理 |
| **水位线** | 取所有流水位线**最小值** | 取最小值 |
| **比喻** | 多车道汇合（同种车） | "一国两制"（不同车种各自处理） |

**水位线合并原理**：合流算子为每条流维护"分区水位线"，算子水位线 = 所有分区水位线**最小值**。时效性由最慢的流决定。

**Connect 的后续处理**：
- `.map(CoMapFunction)` -> 实现 `map1()`、`map2()`
- `.process(CoProcessFunction)` -> 实现 `processElement1()`、`processElement2()`，可选 `onTimer()`。典型场景：**实时对账**--两条支付流相互等待，5 秒内未匹配报警。

**基于时间的双流联结（Join）**

**(1) 窗口联结 Window Join**

```
stream1.join(stream2)
    .where(<KeySelector>)       // 第一条流的 key
    .equalTo(<KeySelector>)     // 第二条流的 key
    .window(<WindowAssigner>)   // 滚动/滑动/会话窗口
    .apply(<JoinFunction>)      // 只能用 apply()
```

处理：两条流数据按 key 分组进对应窗口存储，窗口结束时做**笛卡尔积**（交叉连接），每对匹配调用 `JoinFunction.join(first, second)`。类似 SQL 的 **inner join**，窗口结束才触发。

**(2) 间隔联结 Interval Join（⭐重点）**

**适用场景**：两条流数据时间戳相差不大但不在固定窗口内，如**实时对账**、**订单与浏览行为关联**。窗口 join 会把"卡在窗口边缘两侧"的匹配漏掉，interval join 解决此问题。

**原理**：对一条流 A 的每个元素 a，以 a 的时间戳为中心开辟区间 `[a.ts + lowerBound, a.ts + upperBound]`，另一条流 B 中时间戳落在此区间的元素即可匹配。

```
stream1.keyBy(key1)
    .intervalJoin(stream2.keyBy(key2))  // 两条流 key 类型必须一致
    .between(Time.seconds(-5), Time.seconds(10))  // 下界、上界，可正可负
    .process(new ProcessJoinFunction<IN1, IN2, OUT>() {...})
```

关键特性：
- **只支持事件时间语义**
- 上下界可正可负，但 lowerBound ≤ upperBound
- 也是 **inner join**
- B 流数据可被多个区间匹配（不像窗口 join 一个数据只在一个窗口）
- 底层基于水位线清理过期数据

**与 Window Join 对比**：

| 维度 | Window Join | Interval Join |
|---|---|---|
| 时间范围 | 固定窗口 | 基于每条数据时间戳动态区间 |
| 匹配方式 | 窗口内笛卡尔积 | 单条数据时间戳区间内匹配 |
| 数据复用 | 一条数据只在一个窗口 | 一条数据可被多次匹配 |
| 时间语义 | 处理时间/事件时间 | **仅事件时间** |

**(3) 窗口同组联结 Window CoGroup**

调用形式与 window join 几乎一样，把 `.join()` 换成 `.coGroup()`。

**与 Window Join 的核心区别**：

| 维度 | Window Join | Window CoGroup |
|---|---|---|
| 函数参数 | `join(first, second)` 单对数据 | `coGroup(Iterable<IN1>, Iterable<IN2>, Collector)` 数据集合 |
| 调用次数 | 每对匹配一次 | **每个窗口每个 key 只调用一次** |
| 连接类型 | 仅 inner join | 可实现 **inner/left/right/full outer join** |

外连接实现原理：coGroup 传入两个 Iterable 集合，即使某流无数据，对应集合为空（非 null）仍会调用，判断集合是否为空即可实现外连接。⚠️ **Window Join 底层就是通过 CoGroup 实现的**，CoGroup 更通用。

### 8.2 常见面试题

**Q1：Union 和 Connect 的区别？（高频）**
Union 合并**多条流**，要求**数据类型相同**，合并后还是 DataStream；Connect 只能连接**两条流**，但**数据类型可以不同**，得到 ConnectedStreams 需再处理。Connect 更灵活，底层提供 ProcessFunction 接口可用状态和定时器。两者合流后水位线都取各流最小值。

**Q2：Interval Join 的原理？适用什么场景？**
对一条流每个元素以时间戳为中心开辟 `[ts+lowerBound, ts+upperBound]` 区间，另一条流时间戳落在此区间的数据匹配。只支持事件时间，基于 KeyedStream 调用。适用：两条流数据时间戳相近但不在固定窗口内（实时对账、订单与浏览行为关联）。与 window join 不同，匹配时间段动态，一条数据可被多次匹配。

**Q3：Window Join 和 Window CoGroup 有什么区别？**
Window Join 对窗口内两条流数据做笛卡尔积，每对匹配调用一次 JoinFunction，只实现 inner join；CoGroup 把窗口内每个 key 的数据作为集合传入，只调用一次 CoGroupFunction，可自定义配对逻辑，能实现 inner/left/right/full outer join。Window Join 底层就是用 CoGroup 实现的，CoGroup 更通用。

**Q4：合流后水位线如何推进？**
合流算子为每条流（每个分区）维护独立水位线，算子水位线 = 所有分区水位线**最小值**。因为水位线含义是"之前数据已到齐"，必须所有流都到齐才能推进。时效性由最慢的流决定。

**Q5：如何用 CoProcessFunction 实现实时对账（双流匹配+超时报警）？**
两条流按订单 ID connect+keyBy，用 CoProcessFunction 处理。`processElement1` 处理 A 流时检查 B 流状态是否已有数据，有则匹配输出，无则存入状态并注册 5 秒后的事件时间定时器；`processElement2` 同理。`onTimer` 触发时检查状态，若还在说明对方未到，输出报警并清空状态。

---

## 资料勘误与重点提醒（第1-8章）

1. ⚠️ **DataSet API 已软弃用**：资料基于 Flink 1.13 仍用 DataSet 教批处理。Flink 1.12+ 推荐用 DataStream API + `execution.runtime-mode=BATCH` 实现流批一体。
2. ⚠️ **Spark vs Flink 对比不要绝对化**：Spark 2.3+ 的 Continuous Processing 模式在 at-least-once 下可达 1ms 延迟。应表述为"传统 Spark Streaming(DStream) 微批延迟秒级，Flink 原生流式 + 状态管理 + 事件时间更优"。
3. ⚠️ **WebMonitorEndpoint**：书中 JobManager 只讲 3 个组件，Web UI 实际由 WebMonitorEndpoint 提供，答到 4 个更完整。
4. ⚠️ **算子链合并条件不完整**：资料只说"并行度相同的一对一算子"，遗漏「同一 slot 共享组」条件。完整 4 条件见 4.1。
5. ⚠️ **Per-Job 模式演进**：资料推荐 Per-Job 为生产首选，但 Flink 1.15+ 起 Per-Job 已被官方弃用，推荐 Application 模式。
6. ⚠️ **slot 只隔离内存不隔离 CPU**：高频考点，资料提到但易被忽略。
7. ⚠️ **keyBy 不是算子**：是数据分区操作，返回 KeyedStream，不能设并行度，会断开算子链。
8. ⚠️ **max/min 非聚合字段行为**：资料说"保留最初第一个数据的值"不准确。准确为保留当前累加器记录的字段，会随最大值刷新而更新。
9. ⚠️ **keyBy 哈希算法**：资料说"通过 key 的 hashCode() 取模"。实际 Flink 用 MurmurHash3 求哈希再取模，但 POJO 仍需正确实现 hashCode()/equals()。
10. ⚠️ **新老 Source/Sink API**：资料用 SourceFunction/FlinkKafkaConsumer/FlinkKafkaProducer 等已 @Deprecated API。Flink 1.15+ 推荐新接口（KafkaSource/KafkaSink 等）。
11. ⚠️ **水位线"取所有上游最小"是高频易错点**：面试者常误答"取最大"或"取平均"。务必强调取最小--水位线语义是"之前数据都到齐"，只有最小值才成立。
12. ⚠️ **水位线减 1 毫秒**：`水位线 = maxTimestamp - outOfOrdernessMillis - 1`。减 1 是因 `Watermark(t)` 表示"≤t 的数据都到齐"，不减 1 会矛盾。
13. ⚠️ **水位线生成周期是处理时间不是事件时间**：用事件时间定义周期会死循环。
14. ⚠️ **计数窗口不会被销毁**：基于全局窗口实现，触发后清空数据但窗口保留。答生命周期要区分时间窗口（会销毁）和计数窗口（不销毁）。
15. ⚠️ **空闲源（idle source）问题**：资料未明确提及，是多分区取最小值的直接副作用--某分区无数据时水位线卡住整个下游。解决用 `withIdleness`，面试可主动提。
16. ⚠️ **窗口是"桶"不是"框"**：理解事件时间窗口的关键认知，数据按时间戳分配到对应桶，迟到数据能进正确窗口。
