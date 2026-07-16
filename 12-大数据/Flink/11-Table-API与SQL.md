# 第11章 Table API 和 SQL

> 本文基于《尚硅谷大数据技术之Flink（Java）》整理，面向 Java 后端面试。

---

## 11.1 核心概念

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

## 11.2 常见面试题

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

## 资料勘误与重点提醒

1. ⚠️ **资料笔误（Table API）**：示例代码中应使用 `toChangelogStream(urlCountTable).print()` 却误写为 `toDataStream(urlCountTable).print()`。`urlCountTable` 是分组聚合的更新查询结果，用 `toDataStream` 会抛 `TableException`。
2. ⚠️ **Table API/SQL 版本信息滞后**：资料基于 Flink 1.13，多处描述已过时："Table API 和 SQL 依然不算稳定" -> 1.17+ 已成熟；"blink planner" 需手动指定 -> 1.15 后 Old Planner 已移除；"会话窗口 TVF 尚未完全支持" -> 1.16+ 已支持 Session Window TVF。面试中应说明自己使用的是较新版本（如 1.17/1.18）。
3. ⚠️ **流到表的转换桥梁**：`Row` 类型 + `RowKind` 是 Flink 中连接"流"和"表"的核心数据结构。`RowKind` 取值 `+I`（INSERT）、`-U`（UPDATE_BEFORE）、`+U`（UPDATE_AFTER）、`-D`（DELETE），是动态表编码的物理载体。
4. ⚠️ **MATCH_RECOGNIZE**：资料末尾提到但未展开。这是 SQL 标准的模式识别子句，让 SQL 也能做 CEP，是面试加分点（详见第12章）。
