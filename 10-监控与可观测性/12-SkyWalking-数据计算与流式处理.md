# 12 - SkyWalking 数据计算与流式处理

## 核心概念

### 1. 数据计算架构全景

SkyWalking 的数据计算分为两个阶段：**Agent 端预处理（组装 Segment）** 和 **OAP 端聚合（含三级降采样 L1/L2/L3）**。

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据计算流水线                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Agent 端预处理（不叫 L1，只是 Segment 组装）              │  │
│  │  Span → Segment → 过滤 → 序列化 → 上报                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  OAP 端聚合（L1/L2/L3 三级降采样都在这里）                    │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ ① TraceAnalyzer：解析 Segment，提取 Source 数据     │  │  │
│  │  │     ├── Entry Span → Service/Endpoint 指标          │  │  │
│  │  │     ├── Exit Span  → Relation 指标 + DB/MQ 指标     │  │  │
│  │  │     └── Local Span → 附加分析                        │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                         │                                 │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ ② OAL 引擎：实时聚合（每来一条数据更新指标）           │  │  │
│  │  │     └── 产出 L1 指标（分钟级，1 分钟一个值）          │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                         │                                 │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ ③ 定时降采样任务（每小时/每天触发）                    │  │  │
│  │  │     ├── 每小时：60 个 L1 -> 1 个 L2（小时级）         │  │  │
│  │  │     └── 每天：24 个 L2 -> 1 个 L3（天级）             │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                         │                                 │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ ④ TopN 计算：滑动窗口 + 最小堆                       │  │  │
│  │  │ ⑤ 慢查询检测：阈值判定 + 采样                         │  │  │
│  │  │ ⑥ 热力图：慢端点分布统计                              │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│                     Storage（持久化存储）                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

> ⚠️ **概念澄清**：L1/L2/L3 **全在 OAP 端**，不是"Agent 端做 L1、OAP 端做 L2"。
> - **Agent 端**：只做 Span -> Segment 组装和预过滤，不做时间窗口聚合，不叫 L1
> - **OAP 端**：TraceAnalyzer + OAL 引擎实时聚合产出 **L1（分钟级）**，再由定时任务降采样到 **L2（小时级）** 和 **L3（天级）**
>
> 详见第 5.1 节 L1/L2/L3 的详细说明。

### 2. OAL 引擎（Observability Analysis Language）

#### 2.1 OAL 是什么？

**OAL（Observability Analysis Language）** 是 SkyWalking 自研的 **DSL（领域特定语言）**，用于定义指标聚合规则。它将 Trace/Metrics 的原始数据（Source）聚合成可查询的指标（Metrics）。

**设计理念**：让运维人员（非程序员）也能定义监控指标，无需编写 Java 代码。

#### 2.2 OAL 语法详解

```
语法结构：
<metric_name> = from(<source>).<sourceAttribute>.<filter?>.function(<args>)

示例：
endpoint_cpm = from(Endpoint.latency).sum(1)
service_resp_time = from(Service.latency).avg()
service_sla = from(Service.*).filter(status == true).percent()
```

**OAL 语法组成**：

| 组成部分 | 含义 | 示例 |
|---------|------|------|
| `metric_name` | 指标名称 | `endpoint_cpm` |
| `from(Source)` | 数据来源（Source 类型） | `from(Endpoint.latency)` |
| `sourceAttribute` | 聚合维度 | `latency`, `status`, `*` |
| `filter` | 过滤条件 | `.filter(status == true)` |
| `function` | 聚合函数 | `.sum(1)`, `.avg()`, `.percent()` |

**完整语法规则（来自 OALParser.g4 源码）**：

```antlr
// 聚合语句
aggregationStatement
    : variable (SPACE)? EQUAL (SPACE)? metricStatement SEMI
    ;

// 指标语句
metricStatement
    : FROM LR_BRACKET source sourceAttributeStmt+ RR_BRACKET
      filterStatement* DOT aggregateFunction
    ;

// 过滤语句
filterStatement
    : DOT FILTER LR_BRACKET filterExpression RR_BRACKET
    ;

// 过滤表达式
filterExpression
    : expression  // 支持：==, !=, >, >=, <, <=, like, in, contain, notContain
    ;
```

#### 2.3 OAL 聚合函数

| 函数 | 含义 | 示例 | 备注 |
|------|------|------|------|
| `sum(N)` | 累加 N | `sum(1)` 统计次数 | 用于计数场景 |
| `avg()` | 平均值 | `avg()` 平均延迟 | 只对 aggregation 维度有效 |
| `max()` | 最大值 | `max()` 最大延迟 | — |
| `min()` | 最小值 | `min()` 最小延迟 | — |
| `percent(condition)` | 百分比 | `percent(status == true)` | 计算满足条件的比例 |
| `p99()`, `p95()`, `p90()`, `p75()`, `p50()` | 百分位 | `p99()` 99% 请求的延迟 | 需要排序计算 |
| `longAvg()` | Long 类型平均值 | `longAvg()` | 用于 Long 类型字段 |
| `thermodynamic()` | 热力图 | `thermodynamic()` | 生成延迟分布热力图 |

#### 2.4 OAL 脚本示例（core.oal）

```java
// 文件：oap-server/server-starter/src/main/resources/oal/core.oal

// ===== Service 级别指标 =====
// 服务每分钟调用量（Cpm）
service_cpm = from(Service.*).sum(1);

// 服务平均响应时间
service_resp_time = from(Service.latency).avg();

// 服务成功率（SLA）
service_sla = from(Service.*).filter(status == true).percent();

// 服务吞吐量（每分钟字节数）
service_throughput = from(Service.*).sum(throughput);

// 服务 Apdex
service_apdex = from(Service.latency).apdex(500);  // 500ms 阈值

// ===== Endpoint 级别指标 =====
// 端点每分钟调用量
endpoint_cpm = from(Endpoint.*).sum(1);

// 端点平均响应时间
endpoint_avg = from(Endpoint.latency).avg();

// 端点成功率
endpoint_sla = from(Endpoint.*).filter(status == true).percent();

// 端点 P99 延迟
endpoint_p99 = from(Endpoint.latency).p99();

// 端点 P95 延迟
endpoint_p95 = from(Endpoint.latency).p95();

// ===== Relation 级别指标（服务间调用） =====
// 服务间调用量（Client 视角）
service_relation_client_cpm = from(ServiceRelation.*)
    .filter(detectPoint == DetectPoint.CLIENT).sum(1);

// 服务间调用量（Server 视角）
service_relation_server_cpm = from(ServiceRelation.*)
    .filter(detectPoint == DetectPoint.SERVER).sum(1);

// 服务间调用延迟（Client 视角，包含网络延迟）
service_relation_client_resp_time = from(ServiceRelation.latency)
    .filter(detectPoint == DetectPoint.CLIENT).avg();

// 服务间调用延迟（Server 视角，不含网络延迟）
service_relation_server_resp_time = from(ServiceRelation.latency)
    .filter(detectPoint == DetectPoint.SERVER).avg();

// ===== ServiceInstance 级别指标 =====
// 实例调用量
service_instance_cpm = from(ServiceInstance.*).sum(1);

// ===== JVM 指标 =====
// GC 次数
instance_jvm_young_gc_count = from(ServiceInstanceJVMGC.phrase)
    .filter(phrase == GCPhase.NEW).sum(count);

instance_jvm_old_gc_count = from(ServiceInstanceJVMGC.phrase)
    .filter(phrase == GCPhase.OLD).sum(count);

// JVM 堆内存使用
instance_jvm_heap_used = from(ServiceInstanceJVMMemory.used)
    .filter(isHeap == true).avg();

// ===== 数据库指标 =====
// 数据库访问次数
database_access_cpm = from(DatabaseAccess.*).sum(1);

// 慢 SQL 次数
database_slow_access_cpm = from(DatabaseSlowStatement.*).sum(1);

// 数据库访问延迟
database_access_resp_time = from(DatabaseAccess.latency).avg();

// ===== 缓存指标 =====
// 缓存访问次数
cache_access_cpm = from(CacheAccess.*).sum(1);

// 缓存访问延迟
cache_access_resp_time = from(CacheAccess.latency).avg();
```

#### 2.5 OAL 编译与执行流程

```
OAL 脚本 (.oal 文件)
  │
  ├── 1. 编译阶段
  │     ├── OALParser（ANTLR4 解析器）解析语法
  │     ├── 生成 Java 源代码（OALClassGenerator）
  │     └── 编译生成 .class 文件
  │
  ├── 2. 加载阶段
  │     ├── 加载编译后的 .class 文件
  │     ├── 反射创建指标聚合实例
  │     └── 注册到 MetricsStreamProcessor
  │
  └── 3. 执行阶段
        ├── 接收 Source 数据（来自 TraceAnalyzer）
        ├── 调用聚合实例的 combine() 方法
        ├── 分钟级聚合：每 1 分钟计算一次
        ├── 小时级聚合：每 1 小时计算一次
        └── 天级聚合：每 24 小时计算一次
```

### 3. Agent 端预处理（Segment 组装）

> ⚠️ **澄清**：Agent 端做的是 **Segment 组装和预过滤**，**不叫 L1 聚合**。L1/L2/L3 是 OAP 端的三级降采样层级（见第 5 节），Agent 端只是把 Span 打包成 Segment 上报，不做时间窗口聚合。

Agent 端负责将 Span 数据组装成 Segment，并进行初步过滤：

```
Agent 端预处理流程：

1. 请求到达 → 创建 Entry Span
2. 调用下游 → 创建 Exit Span
3. 本地操作 → 创建 Local Span
4. 请求结束 → 所有 Span 完成
5. 组装 Segment → 包含所有 Span 和一个 segmentId
6. 序列化 Segment → Protobuf 编码
7. 放入 TraceBuffer → 等待批量上报
8. 批量上报 → gRPC 流式发送到 OAP
```

**Agent 端的过滤逻辑**：
- 忽略端点（ignore_suffix）：`.jpg`, `.js`, `.css` 等静态资源
- 采样控制（sample rate）：每 3 秒最多 N 个 Trace
- Span 数量限制（span_limit_per_segment）：默认 300 个 Span

### 4. OAP 端实时聚合（对应 L1 分钟级）

> ⚠️ **澄清**：OAP 的 TraceAnalyzer 做的是**实时聚合**，每来一条 Segment 就更新指标，按 **1 分钟**滚动窗口产出 L1 指标。这就是降采样的第一级 L1（见第 5 节）。

#### 4.1 TraceAnalyzer 解析流程

```java
// 源码：TraceAnalyzer.java - Segment 解析核心逻辑
public void doAnalysis(SegmentObject segmentObject) {
    // 1. 创建 Span 监听器
    createSpanListeners();

    // 2. 通知 Segment 级别的监听器
    notifySegmentListener(segmentObject);

    // 3. 遍历每个 Span，根据类型分发到不同的监听器
    segmentObject.getSpansList().forEach(spanObject -> {
        if (spanObject.getSpanId() == 0) {
            notifyFirstListener(spanObject, segmentObject);  // 第一个 Span
        }

        if (SpanType.Exit.equals(spanObject.getSpanType())) {
            notifyExitListener(spanObject, segmentObject);   // Exit Span
        } else if (SpanType.Entry.equals(spanObject.getSpanType())) {
            notifyEntryListener(spanObject, segmentObject);  // Entry Span
        } else if (SpanType.Local.equals(spanObject.getSpanType())) {
            notifyLocalListener(spanObject, segmentObject);  // Local Span
        }
    });

    // 4. 所有监听器完成分析后，触发 build（生成 Source）
    notifyListenerToBuild();
}
```

#### 4.2 监听器类型与职责

| 监听器 | 触发条件 | 生成的 Source | 聚合的指标 |
|--------|---------|--------------|-----------|
| **EntryAnalysisListener** | Entry Span | Service, Endpoint | service_cpm, endpoint_cpm |
| **ExitAnalysisListener** | Exit Span | ServiceRelation, DatabaseAccess, CacheAccess | relation_cpm, db_access_cpm |
| **LocalAnalysisListener** | Local Span | 附加分析 | 本地方法耗时 |
| **FirstAnalysisListener** | 每个 Segment 的第一个 Span | EndpointMeta | 端点元数据 |
| **SegmentListener** | 每个 Segment | Segment | 整体 Segment 信息 |
| **RPCAnalysisListener** | RPC 类型 Span | ServiceRelation | 服务间调用指标 |
| **DatabaseSlowStatementBuilder** | DB 慢查询 | DatabaseSlowStatement | 慢 SQL 统计 |
| **EndpointDependencyBuilder** | 端点间调用 | EndpointRelation | 端点依赖关系 |

#### 4.3 虚拟数据库/缓存/消息队列

对于没有 Agent 探针的数据库/缓存/消息队列，SkyWalking 通过 **Exit Span 分析**自动推断出虚拟服务：

```java
// 源码：VirtualDatabaseProcessor.java
// 从 Exit Span 的 db.type 和 peer 字段推断虚拟数据库服务

if (span.getSpanLayer() == SpanLayer.Database) {
    // 创建虚拟数据库服务
    String dbName = span.getPeer();  // 如 "192.168.1.1:3306"
    String dbType = span.getTag("db.type");  // 如 "MySQL"
    // 生成虚拟服务名：MySQL:192.168.1.1:3306
    createVirtualDatabaseService(dbType + ":" + dbName);
}
```

### 5. Downsampling（降采样）

SkyWalking 使用三级聚合架构，数据按时间窗口逐级降采样：

```
┌──────────────────────────────────────────────────────────────────┐
│                    三级降采样架构                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  原始数据（Segment）                                               │
│    │                                                              │
│    ▼                                                              │
│  L1 聚合（分钟级）                                                 │
│  ├── 窗口大小：1 分钟                                             │
│  ├── 聚合方式：实时聚合（每来一条数据就更新）                         │
│  ├── 存储周期：7 天（默认）                                        │
│  └── 示例：service_cpm = 每分钟请求数                              │
│    │                                                              │
│    ▼                                                              │
│  L2 聚合（小时级）                                                 │
│  ├── 窗口大小：1 小时                                             │
│  ├── 聚合方式：从 L1 数据二次聚合                                    │
│  ├── 存储周期：30 天（默认）                                       │
│  └── 示例：service_cpm_hour = 每小时平均请求数                      │
│    │                                                              │
│    ▼                                                              │
│  L3 聚合（天级）                                                   │
│  ├── 窗口大小：1 天                                               │
│  ├── 聚合方式：从 L2 数据二次聚合                                    │
│  ├── 存储周期：365 天（默认）                                      │
│  └── 示例：service_cpm_day = 每天平均请求数                         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

#### 5.1 L1/L2/L3 到底是什么意思？ ⭐⭐⭐

很多人只记住 L1/L2/L3 这三个名字，但不清楚具体含义。这里彻底讲明白：

**核心概念：L1/L2/L3 是 OAP 端按时间粒度逐级降采样的三个层级**

```
L = Level（层级），数字越大，时间粒度越粗，保留越久

  原始数据（Segment，每条请求一份）
      │  OAP 实时聚合（每来一条更新指标）
      ▼
  ┌──────────────────────────────────────────┐
  │  L1：分钟级聚合（最细粒度）                │
  │  ├── 时间窗口：1 分钟                     │
  │  ├── 产出：每分钟一个指标值               │
  │  ├── 存储周期：7 天（默认）               │
  │  └── 用途：看"最近"的实时监控             │
  └──────────────────────────────────────────┘
      │  每小时定时任务，把 60 个 L1 值聚合成 1 个
      ▼
  ┌──────────────────────────────────────────┐
  │  L2：小时级聚合（中等粒度）                │
  │  ├── 时间窗口：1 小时                     │
  │  ├── 产出：每小时一个指标值               │
  │  ├── 存储周期：30 天（默认）              │
  │  └── 用途：看"最近一个月"的趋势          │
  └──────────────────────────────────────────┘
      │  每天定时任务，把 24 个 L2 值聚合成 1 个
      ▼
  ┌──────────────────────────────────────────┐
  │  L3：天级聚合（最粗粒度）                  │
  │  ├── 时间窗口：1 天                       │
  │  ├── 产出：每天一个指标值                 │
  │  ├── 存储周期：365 天（默认）             │
  │  └── 用途：看"一年"的长期趋势             │
  └──────────────────────────────────────────┘
```

**用一个具体例子理解：监控 service_cpm（服务每分钟调用量）**

```
假设某服务一天的调用量如下：

  00:00-01:00  每分钟约 100 次（L1 有 60 个值）
  01:00-02:00  每分钟约 50 次（L1 有 60 个值）
  ...
  23:00-24:00  每分钟约 200 次（L1 有 60 个值）

  → 一天产生 1440 个 L1 数据点（每分钟一个）

  L2 聚合后：
  → 一天变成 24 个 L2 数据点（每小时一个，值=该小时 60 个 L1 的平均）

  L3 聚合后：
  → 一天变成 1 个 L3 数据点（值=该天 24 个 L2 的平均）
```

**为什么需要三级降采样？存储和查询的权衡**

```
查询"最近 1 小时"的数据：
  → 用 L1（分钟级），60 个数据点，曲线精细，但只能查 7 天内的

查询"最近 30 天"的趋势：
  → 用 L2（小时级），720 个数据点，曲线适中，能查 30 天

查询"最近 1 年"的长期趋势：
  → 用 L3（天级），365 个数据点，曲线粗，但能查 1 年

如果只用 L1（分钟级）存所有数据：
  → 1 年 = 365 × 1440 = 52 万个数据点（存储爆炸！）
  → 查询慢、存储贵

三级降采样后：
  → 1 年 = 365 个 L3 数据点（存储极小，查询飞快）
```

**OAP 查询时自动选层级**：UI 选择时间范围后，OAP 自动判断用哪一级
- 选最近 1 小时 -> 用 L1
- 选最近 30 天 -> 用 L2
- 选最近 1 年 -> 用 L3

---

#### 5.2 降采样策略表

| 函数 | 降采样方式 | 含义 |
|------|-----------|------|
| `avg()` | 加权平均 | 保留 count 和 sum，降采样时计算 weighted avg |
| `sum()` | 累加 | 直接相加 |
| `max()` | 取最大值 | 取所有子窗口的最大值 |
| `min()` | 取最小值 | 取所有子窗口的最小值 |
| `p99()` | 重新计算 | 不能简单降采样，需要保留原始分布数据 |
| `percent()` | 重新计算 | 保留分子和分母，降采样时重新计算 |

### 6. TopN 计算

TopN 用于统计"最慢的 N 个端点"或"最频繁调用的 N 个服务"。

```
算法：滑动窗口 + 最小堆

1. 维护一个滑动窗口（如 5 分钟）
2. 使用最小堆（PriorityQueue）保存 TopN 元素
3. 新数据到达：
   if (堆大小 < N) {
       直接入堆
   } else if (新值 > 堆顶) {
       移除堆顶，新值入堆
   }
4. 窗口滑动时，过期的数据从堆中移除
5. 每分钟输出当前 TopN 结果
```

### 7. 慢查询检测

#### 7.1 慢 SQL 检测

```java
// 源码：DatabaseSlowStatementBuilder.java
// 当 Exit Span 的 db.statement 存在，且延迟超过阈值时触发

if (span.getLatency() > slowThreshold) {
    DatabaseSlowStatement slowStatement = new DatabaseSlowStatement();
    slowStatement.setStatement(span.getTag("db.statement"));
    slowStatement.setLatency(span.getLatency());
    slowStatement.setTraceId(traceId);
    // 写入慢 SQL 记录
}
```

**慢阈值配置**：
```yaml
# config/application.yml
agent-analyzer:
  default:
    # 慢 SQL 阈值（毫秒）
    slowDBAccessThreshold: ${SW_SLOW_DB_THRESHOLD:default:200,mongodb:100}
    # 慢 HTTP 请求阈值
    slowHttpRequestThreshold: ${SW_SLOW_HTTP_THRESHOLD:default:1000}
```

### 8. 热力图

热力图用于展示**慢端点的分布**，帮助快速定位性能瓶颈：

```
热力图数据模型：
┌──────────────────────────────────────────────┐
│  Endpoint 热力图（延迟分布）                    │
│                                               │
│  端点 A: ████████░░░░░░░░ (P99=50ms)          │
│  端点 B: ████████████████░░ (P99=100ms)        │
│  端点 C: ████████████████████████ (P99=500ms) │
│                                               │
│  █ = 0-50ms 的请求                             │
│  ░ = 50ms+ 的请求                              │
└──────────────────────────────────────────────┘
```

---

## 常见面试题

### Q1: OAL 引擎是什么？解决了什么问题？

**OAL（Observability Analysis Language）** 是 SkyWalking 自研的 DSL，用于定义指标聚合规则。

**解决的问题**：
1. **配置化**：运维人员不需要写 Java 代码，通过 `.oal` 脚本文件定义指标
2. **解耦**：指标定义与存储引擎解耦，切换存储引擎不需要修改指标定义
3. **可扩展**：新增指标只需添加 OAL 脚本，无需修改核心代码
4. **编译执行**：OAL 脚本编译为 Java 类，运行时性能接近原生 Java 代码

### Q2: 三级降采样（L1/L2/L3）是什么？为什么需要？

**L1/L2/L3 是 OAP 端按时间粒度逐级降采样的三个层级**（L = Level）：

| 层级 | 时间窗口 | 存储周期 | 用途 |
|------|---------|---------|------|
| **L1** | 分钟级 | 7 天 | 看最近的实时监控 |
| **L2** | 小时级 | 30 天 | 看最近一个月的趋势 |
| **L3** | 天级 | 365 天 | 看一年的长期趋势 |

**聚合链路**：原始数据 → L1（每分钟聚合）→ L2（每小时从 L1 二次聚合）→ L3（每天从 L2 二次聚合）

**为什么需要**：
1. **存储成本**：只用 L1 存一年的数据 = 52 万个数据点（存储爆炸）；三级降采样后，一年只用 365 个 L3 数据点
2. **查询性能**：查询时 OAP 自动选层级（看 1 小时用 L1、看 1 年用 L3），数据量小查询快
3. **数据生命周期**：细粒度数据短期保留（7 天），粗粒度数据长期保留（365 天）

### Q3: TopN 计算使用什么算法？

**最小堆（PriorityQueue）** 算法：

1. 维护一个大小为 N 的最小堆
2. 新数据到达时，如果堆未满，直接入堆
3. 如果堆已满，比较新值与堆顶（最小值），如果新值更大，则替换堆顶
4. 堆中始终保存着最大的 N 个值
5. 时间复杂度：O(log N) 每次插入

### Q4: 虚拟数据库/缓存/MQ 服务是如何被发现的？

SkyWalking 通过分析 Exit Span 的元数据来推断虚拟服务：

1. **数据库**：Exit Span 的 `spanLayer=Database` + `peer` 地址 + `db.type` → 创建虚拟数据库服务
2. **缓存**：Exit Span 的 `spanLayer=Cache` + `peer` 地址 + `cache.type` → 创建虚拟缓存服务
3. **消息队列**：Exit Span 的 `spanLayer=MQ` + `mq.topic` + `peer` 地址 → 创建虚拟 MQ 服务

这些虚拟服务会出现在拓扑图中，帮助运维人员了解完整的服务依赖关系，即使数据库/缓存没有安装 Agent。

---

## 延伸阅读

- OAL 语法定义：`oap-server/oal-grammar/src/main/antlr4/org/apache/skywalking/oal/rt/grammar/OALParser.g4`
- 核心 OAL 脚本：`oap-server/server-starter/src/main/resources/oal/core.oal`
- TraceAnalyzer 源码：`oap-server/analyzer/agent-analyzer/.../trace/parser/TraceAnalyzer.java`