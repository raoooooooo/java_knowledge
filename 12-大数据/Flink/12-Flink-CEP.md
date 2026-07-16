# 第12章 Flink CEP

> 本文基于《尚硅谷大数据技术之Flink（Java）》整理，面向 Java 后端面试。

---

## 12.1 核心概念

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

## 12.2 常见面试题

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

## 资料勘误与重点提醒

1. ⚠️ **MATCH_RECOGNIZE**：资料末尾提到但未展开。这是 SQL 标准的模式识别子句，让 SQL 也能做 CEP，是面试加分点。
2. ⚠️ **CEP 状态膨胀风险**：`oneOrMore` 等循环模式若不配 `within` 或 `until` 会导致状态无限增长 OOM，面试常被追问。务必配合时间限制或终止条件使用。
