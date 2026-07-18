# 06 - SkyWalking 指标体系与分类

## 核心概念

### 1. 指标全景图

SkyWalking 的指标体系分为五个层级，从宏观到微观：

```
┌──────────────────────────────────────────────────────────────────┐
│                    SkyWalking 指标体系全景                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Level 1: Service 级别指标（服务整体健康度）                    │ │
│  │  Apdex | Cpm(每分钟调用量) | SLA | RT(平均响应时间) |          │ │
│  │  Throughput(吞吐量) | ErrorRate(错误率)                        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Level 2: ServiceInstance 级别指标（单个实例健康度）            │ │
│  │  JVM内存(堆/非堆/直接内存) | JVM GC(次数/时间/阶段) |         │ │
│  │  JVM线程(活跃/守护/峰值) | JVM CPU | JVM 类加载               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Level 3: Endpoint 级别指标（接口级别）                        │ │
│  │  QPS | 延迟(P50/P75/P90/P95/P99) | 错误率 | 状态码分布       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Level 4: Relation 级别指标（服务间调用关系）                   │ │
│  │  Client → Server 调用量/延迟/错误率（双向视角）                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Level 5: Meter 自定义指标（基础设施/业务指标）                │ │
│  │  Counter(计数器) | Gauge(瞬时值) | Histogram(分布) |          │ │
│  │  DistributionSummary(摘要)                                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.1 指标层级与 Layer/Component 的关系

上一章讲了 Layer（分层）和 Component（组件），这里解释它们**如何决定指标内容**：

**核心关系：Layer 决定"量什么"，五级层次决定"怎么量"。**

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer 与指标层级的关系                                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  同一个 Service 层级，不同 Layer 量的指标完全不同：                  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  GENERAL 层（通用服务）                                      │  │
│  │  ├── Service 级：Apdex、Cpm、SLA、RT、Throughput            │  │
│  │  ├── Endpoint 级：QPS、P99、HTTP 状态码分布                  │  │
│  │  └── Relation 级：Client/Server 双向调用指标                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  DB 层（数据库）                                             │  │
│  │  ├── Service 级：连接数、慢查询数、QPS、响应时间              │  │
│  │  ├── Endpoint 级：SQL 语句级别的延迟、慢 SQL 检测             │  │
│  │  └── Relation 级：哪个服务调了哪个数据库                       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  CACHE 层（缓存）                                            │  │
│  │  ├── Service 级：命中率、QPS、响应时间、内存使用             │  │
│  │  ├── Endpoint 级：每个 Key 的访问次数、延迟                   │  │
│  │  └── Relation 级：哪个服务调了哪个缓存                         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  MQ 层（消息队列）                                           │  │
│  │  ├── Service 级：生产速率、消费速率、消息积压、发送延迟       │  │
│  │  ├── Endpoint 级：每个 Topic 的生产/消费量                    │  │
│  │  └── Relation 级：哪个服务生产/消费了哪个 Topic               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Component 的作用**：在同一个 Layer 内，不同 Component 的指标**计算方式相同**，但**标签值不同**，用来区分具体技术栈。

```
示例：DB 层下的两个服务

┌──────────────────────────────────────────────────────────────────┐
│  Service: "MySQL:192.168.1.1:3306"                                │
│  ├── Layer: DB                                                    │
│  ├── Component: MySQL (ID=2)                                      │
│  ├── service_cpm: 5000                                            │
│  ├── database_access_latency: 3ms                                 │
│  └── database_slow_access: 12/min                                 │
│                                                                   │
│  Service: "PostgreSQL:192.168.1.2:5432"                           │
│  ├── Layer: DB                                                    │
│  ├── Component: PostgreSQL (ID=22)                                │
│  ├── service_cpm: 3000       ← 同一套指标计算公式                   │
│  ├── database_access_latency: 5ms                                 │
│  └── database_slow_access: 3/min                                  │
│                                                                   │
│  两个数据库的指标类型完全一样（都是 DB 层的指标），                   │
│  但 Component 不同，在 UI 中显示为不同的实例。                       │
└──────────────────────────────────────────────────────────────────┘
```

**一句话总结**：

> **Layer 决定指标模板**（DB 有慢查询指标，MQ 有积压指标，GENERAL 有 Apdex），**Component 决定指标标签**（MySQL vs PostgreSQL 用不同的 Component ID 标记），**五级层次决定指标粒度**（Service 级看整体，Endpoint 级看接口，Relation 级看调用关系）。

### 2. Service 级别指标详解

#### 2.1 Apdex（应用性能指数）

**Apdex（Application Performance Index）** 是衡量用户满意度的标准化指标，取值范围 0~1：

```
Apdex = (满意数 + 0.5 × 可容忍数) / 总采样数

阈值定义（T = Apdex 阈值，通常设为服务 SLA 目标延迟）：
- 满意（Satisfied）：RT ≤ T
- 可容忍（Tolerating）：T < RT ≤ 4T
- 失望（Frustrated）：RT > 4T
```

**Apdex 评分标准**：

| Apdex 值 | 评级 | 含义 |
|----------|------|------|
| 0.94 ~ 1.00 | 优秀 | 用户满意度很高 |
| 0.85 ~ 0.93 | 良好 | 大多数用户满意 |
| 0.70 ~ 0.84 | 一般 | 部分用户感觉延迟 |
| 0.50 ~ 0.69 | 较差 | 需要优化 |
| < 0.50 | 不可接受 | 严重影响用户体验 |

**SkyWalking 中的 Apdex 配置**：在 `application.yml` 中可通过 `service-apdex-threshold` 配置，默认 500ms。

#### 2.2 Cpm（Call Per Minute，每分钟调用量）

```
Cpm = 每分钟该服务处理的请求总数
```

- 用于衡量服务负载
- 与 QPS（Queries Per Second）的关系：Cpm = QPS × 60

#### 2.3 SLA（Service Level Agreement）

在 SkyWalking 中，SLA 指标通常指**请求成功率**：

```
SLA = 成功请求数 / 总请求数 × 100%
```

SkyWalking 将 SLA 以百分比形式展示，100% 表示所有请求都成功。

#### 2.4 RT（Response Time，平均响应时间）

```
RT = 所有请求的响应时间总和 / 请求总数
```

**注意**：平均响应时间容易被长尾请求拉高，需要结合百分位延迟（P95/P99）一起看。

#### 2.5 Throughput（吞吐量）

```
Throughput = 每分钟处理的请求总字节数
```

#### 2.6 ErrorRate（错误率）

```
ErrorRate = 错误请求数 / 总请求数 × 100%
```

### 3. ServiceInstance 级别指标（JVM 指标）

SkyWalking 通过 Java Agent 自动采集 JVM 指标，无需额外配置。

#### 3.1 JVM 内存指标

| 指标 | 含义 | 数据来源 | 告警建议 |
|------|------|---------|---------|
| **堆内存使用量** | 当前堆内存使用大小 | `MemoryMXBean.getHeapMemoryUsage()` | > 80% 告警 |
| **堆内存最大值** | 堆内存最大可用大小 | `-Xmx` 配置 | — |
| **非堆内存使用量** | 元空间/CodeCache 等 | `MemoryMXBean.getNonHeapMemoryUsage()` | > 80% 告警 |
| **直接内存使用量** | 堆外内存使用 | `BufferPoolMXBean` | 关注 OOM 风险 |
| **各内存池使用量** | Eden/Survivor/Old/Metaspace | `MemoryPoolMXBean` | Old 区持续增长 → 内存泄漏 |

**源码对应**：
```java
// oap-server/server-core/.../source/ServiceInstanceJVMMemory.java
@ScopeDeclaration(id = DefaultScopeDefine.SERVICE_INSTANCE_JVM_MEMORY, name = "ServiceInstanceJVMMemory")
public class ServiceInstanceJVMMemory extends Source {
    private boolean isHeap;        // 是否堆内存
    private long init;             // 初始值
    private long max;              // 最大值
    private long used;             // 已使用
    private long committed;        // 已提交
}
```

#### 3.2 JVM GC 指标

| 指标 | 含义 | 数据来源 |
|------|------|---------|
| **Young GC 次数** | 年轻代 GC 次数 | `GarbageCollectorMXBean.getCollectionCount()` |
| **Young GC 时间** | 年轻代 GC 总耗时 | `GarbageCollectorMXBean.getCollectionTime()` |
| **Old GC 次数** | 老年代/Full GC 次数 | 同上 |
| **Old GC 时间** | 老年代/Full GC 总耗时 | 同上 |
| **GC 阶段** | New/Old | `GCPhase` 枚举 |

**源码对应**：
```java
// oap-server/server-core/.../source/ServiceInstanceJVMGC.java
@ScopeDeclaration(id = DefaultScopeDefine.SERVICE_INSTANCE_JVM_GC, name = "ServiceInstanceJVMGC")
public class ServiceInstanceJVMGC extends Source {
    private GCPhase phase;         // 枚举：NEW, OLD
    private long time;             // GC 耗时
    private long count;            // GC 次数
}
```

#### 3.3 JVM 线程指标

| 指标 | 含义 |
|------|------|
| **活跃线程数** | 当前活跃（非守护）线程数 |
| **守护线程数** | 守护线程数 |
| **峰值线程数** | 历史最高线程数 |
| **总启动线程数** | 累计创建的线程数 |

#### 3.4 JVM CPU 指标

| 指标 | 含义 | 数据来源 |
|------|------|---------|
| **进程 CPU 使用率** | JVM 进程的 CPU 使用率 | `OperatingSystemMXBean.getProcessCpuLoad()` |
| **系统 CPU 使用率** | 整机 CPU 使用率 | `OperatingSystemMXBean.getSystemCpuLoad()` |

#### 3.5 JVM 类加载指标

| 指标 | 含义 |
|------|------|
| **已加载类数量** | 当前已加载的类总数 |
| **已卸载类数量** | 累计卸载的类总数 |

### 4. Endpoint 级别指标

| 指标 | 含义 | 计算方式 |
|------|------|---------|
| **QPS** | 每秒请求数 | 请求总数 / 统计周期秒数 |
| **RT（平均响应时间）** | 平均响应时间 | 总耗时 / 请求数 |
| **P50/P75/P90/P95/P99** | 百分位延迟 | 排序后取对应百分位值 |
| **ErrorRate** | 错误率 | 错误请求数 / 总请求数 |
| **状态码分布** | HTTP 状态码分布 | 按 status_code 分组统计 |
| **SlowCount** | 慢请求数 | RT > 慢阈值 的请求数 |

#### 百分位延迟（Percentile）详解

**为什么需要百分位延迟？**

平均响应时间（RT）会掩盖长尾请求。例如：
- 100 个请求，99 个 10ms，1 个 10s → RT ≈ 109ms（看起来还好）
- 但 P99 = 10s，说明 **99% 的请求在 10s 以内，但最慢的请求达到了 10s**——平均值 109ms 完全掩盖了这个长尾问题

**P50/P95/P99 的含义**：

| 百分位 | 含义 | 面试问法 |
|--------|------|---------|
| P50 | 50% 请求的延迟 ≤ 此值（中位数） | "一半用户的响应时间" |
| P95 | 95% 请求的延迟 ≤ 此值 | "95% 用户的体验" |
| P99 | 99% 请求的延迟 ≤ 此值 | "长尾请求的严重程度" |

```
示例：100 个请求的响应时间（ms）：
[5, 5, 5, 8, 8, 8, 10, 10, 10, 10, 10, 15, 15, 20, 20, 50, 100, 500, 1000, 5000]

P50（第50个）= 10ms    → 一半用户响应在 10ms 以内
P95（第95个）= 1000ms  → 5% 的用户响应超过 1000ms
P99（第99个）= 5000ms  → 1% 的用户响应达到 5000ms
P99.9（第99.9个）= 5000ms
```

### 5. Relation 级别指标（服务间调用指标）

这是 SkyWalking 最独特的指标维度。对于 A → B 的一次调用，SkyWalking 同时记录：

```
                    ┌───────────┐
                    │  Service A │
                    └─────┬─────┘
                          │ 调用
                          ▼
                    ┌───────────┐
                    │  Service B │
                    └───────────┘

Client 视角（ServiceRelation.ClientSide）：
  - A 调用 B 的 QPS、延迟、成功率
  - 从 A 的 Exit Span 中统计

Server 视角（ServiceRelation.ServerSide）：
  - B 被 A 调用的 QPS、延迟、成功率
  - 从 B 的 Entry Span 中统计
```

**为什么需要双向视角？**

1. **网络延迟**：Client 端延迟 - Server 端延迟 ≈ 网络延迟（传输延迟 + 序列化开销）
2. **错误定位**：Client 端报错而 Server 端正常 → 网络问题
3. **调用量分析**：Client 端调用量 ≠ Server 端接收量时 → 可能存在网络丢包或负载均衡问题

**源码对应**：
```java
// oap-server/server-core/.../source/ServiceRelation.java
@ScopeDeclaration(id = DefaultScopeDefine.SERVICE_RELATION, name = "ServiceRelation")
public class ServiceRelation extends Source {
    private String entityId;
    private int sourceServiceId;     // 调用方（Client）服务 ID
    private String sourceServiceName;
    private int destServiceId;       // 被调用方（Server）服务 ID
    private String destServiceName;
    private DetectPoint detectPoint; // CLIENT 或 SERVER 视角
    private String componentId;       // 使用的组件
}
```

### 6. Meter 自定义指标

#### 6.0 什么是 Meter？为什么需要它？

Meter（计量器）是 SkyWalking v8 引入的**通用指标接入框架**，它解决的核心问题是：**SkyWalking Agent 采集的 Trace 数据，只能算出来与请求相关的指标——但你还有大量"不与任何请求绑定"的指标需要监控。**

**Trace 指标的局限性**（只能算请求相关的）：

```
┌──────────────────────────────────────────────────────────────────┐
│  Trace 指标（OAL 产出）只能覆盖：                                    │
│                                                                   │
│  ✅ 每个请求的响应时间（RT）→ 算出 p50/p99                         │
│  ✅ 每分钟请求数（Cpm）→ 算出 QPS                                │
│  ✅ 每个请求的成功/失败 → 算出 SLA / 错误率                        │
│  ✅ 每个请求调用了哪个下游 → 算出 Relation 指标                     │
│                                                                   │
│  ❌ 无法覆盖：当前数据库连接数（不与任何请求绑定）                    │
│  ❌ 无法覆盖：消息队列积压量（不与任何请求绑定）                      │
│  ❌ 无法覆盖：订单总数（业务指标，不与 HTTP 请求绑定）                │
│  ❌ 无法覆盖：主机 CPU 使用率（基础设施指标，在应用之外）            │
└──────────────────────────────────────────────────────────────────┘
```

**Meter 的本质**：为 SkyWalking 打开一扇"后门"，让任何来源的、任何维度的指标，都能接入 SkyWalking 的分析和可视化体系。它是**"非 Trace 来源指标"的统一入口**。

```
┌──────────────────────────────────────────────────────────────────┐
│  Meter = 所有"不来自 Trace"的指标的总称                             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  第一类：基础设施指标（OS/中间件）                              │  │
│  │  • 主机 CPU、内存、磁盘、网络                                  │  │
│  │  • MySQL 连接数、慢查询数、QPS                                 │  │
│  │  • Kafka Topic 的消息生产速率、消费速率、Lag 积压量             │  │
│  │  • Redis 命中率、内存使用、连接数                               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  第二类：应用内部指标（JVM / 自定义）                           │  │
│  │  • JVM 堆内存、非堆内存、GC 次数、GC 耗时                      │  │
│  │  • 线程池活跃线程数、队列长度、拒绝策略触发次数                 │  │
│  │  • 自定义业务指标：每分钟订单数、每分钟支付成功数、库存告警阈值   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  第三类：外部系统指标（第三方监控）                           │  │
│  │  • Prometheus Exporter 暴露的指标                             │  │
│  │  • OpenTelemetry SDK 采集的 Metrics                           │  │
│  │  • Micrometer 采集的 Spring Boot Actuator 指标                │  │
│  │  • Telegraf / Zabbix 采集的系统监控数据                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**一句话总结**：Trace 指标是"**从请求调用链里提炼出来的指标**"（请求驱动），Meter 指标是"**所有不是从调用链里来的指标**"（定时采集驱动）。两者在 SkyWalking UI 中统一展示，但来源和分析引擎完全不同（Trace → OAL，Meter → MAL）。

#### 6.1 Meter 指标类型

| 类型 | 说明 | 适用场景 | 数据结构 |
|------|------|---------|---------|
| **Counter** | 单调递增计数器 | 请求总数、错误总数、消息生产数 | `long value` |
| **Gauge** | 瞬时值，可增可减 | 当前连接数、队列长度、内存使用 | `double value` |
| **Histogram** | 分布统计，自动计算 P50/P95/P99 | 请求延迟分布、数据包大小分布 | `double[] buckets` + `long[] counts` |
| **DistributionSummary** | 统计学摘要 | 类似 Histogram | `count` + `total` + `max` |

#### 6.2 Meter 数据来源

| 来源 | 接入方式 | 说明 |
|------|---------|------|
| **Prometheus** | Prometheus Fetcher 拉取 | OAP 定时拉取 Prometheus Exporter 指标 |
| **OpenTelemetry** | OTLP Receiver | 接收 OTel SDK 的 Meter 数据 |
| **Java Agent（AgentMeter）** | Agent 内置 Meter | 如 JVM 指标、线程池指标 |
| **Micrometer** | Micrometer Bridge | Spring Boot Actuator 指标桥接 |
| **Telegraf** | Telegraf Receiver | 接收 Telegraf 采集的系统指标 |
| **Zabbix** | Zabbix Receiver | 接收 Zabbix Agent 数据 |

#### 6.3 MAL 处理 Meter 指标

Meter 数据进入 OAP 后，通过 **MAL（Meter Analysis Language）** 进行聚合分析：

```yaml
# config/meter-analyzer-config/spring-sleuth.yaml
metricsRules:
  - name: http_server_requests_seconds_sum
    exp: http_server_requests_seconds_sum.tagEqual("uri", "/api/**").sum(['service','uri'])
```

#### 6.4 ⚠️ 关键区分：OTel 的 Traces 和 Metrics 在 SkyWalking 中走的是两条路

很多初学者会混淆：用 OpenTelemetry 采集指标走 Meter，那 OTel 的调用链（Traces）去哪了？**答案是：OTel 发送的是三种信号，SkyWalking 分别走三条不同的处理管道。**

```
┌──────────────────────────────────────────────────────────────────┐
│  OpenTelemetry SDK 发送三种信号到 SkyWalking OAP                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────┐                                          │
│  │  OTel Java Agent    │                                          │
│  │  (或 OTel SDK)       │                                          │
│  └────────┬────────────┘                                          │
│           │ OTLP 协议（gRPC :4317 或 HTTP :4318）                   │
│           ▼                                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  SkyWalking OAP — OTLP Receiver                            │  │
│  │                                                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │  │
│  │  │  OTel Traces    │  │  OTel Metrics   │  │  OTel Logs  │ │  │
│  │  │  (Spans)        │  │  (Counter/Gauge/│  │  (LogRecords│ │  │
│  │  │                 │  │   Histogram)     │  │   )         │ │  │
│  │  └───────┬─────────┘  └───────┬─────────┘  └──────┬─────┘ │  │
│  │          │                    │                    │       │  │
│  └──────────┼────────────────────┼────────────────────┼───────┘  │
│             │                    │                    │           │
│             ▼                    ▼                    ▼           │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐  │
│  │  Trace 管道       │ │  Meter 管道       │ │  Log 管道         │  │
│  │  (TraceAnalyzer) │ │  (MAL 引擎)       │ │  (LAL 引擎)       │  │
│  │                  │ │                  │ │                  │  │
│  │  OTel Span →     │ │  OTel Counter →  │ │  OTel LogRecord  │  │
│  │  SkyWalking      │ │  SkyWalking      │ │  → SkyWalking    │  │
│  │  Segment/Span    │ │  Meter 指标      │ │  Log 记录        │  │
│  │                  │ │                  │ │                  │  │
│  │  → OAL 聚合      │ │  → MAL 聚合      │ │  → LAL 分析      │  │
│  │  → Service/      │ │  → Meter 指标    │ │  → 日志指标      │  │
│  │    Endpoint/     │ │    存储          │ │    存储          │  │
│  │    Relation 指标 │ │                  │ │                  │  │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**核心结论**：

| OTel 信号 | 进入 SkyWalking 后变成 | 用什么语言分析 | 产生什么指标 |
|-----------|----------------------|---------------|-------------|
| **Traces（Spans）** | Segment / Span（Trace 模型） | OAL | service_cpm、endpoint_p99、service_apdex 等 |
| **Metrics（Counter/Gauge/Histogram）** | Meter 指标 | MAL | 自定义业务指标（如订单数、库存量） |
| **Logs（LogRecords）** | Log 记录 | LAL | 日志错误率、日志模式匹配 |

> **一句话**：OTel 的调用链（Traces）**不走 Meter 管道**，它走的是 Trace 管道——和 SkyWalking Agent 上报的 Segment 走的是**同一条路**。OAP 的 OTLP Receiver 会把 OTel Span 转换成 SkyWalking 的 Segment/Span 模型，然后用 OAL 脚本聚合出 Service/Endpoint/Relation 指标。Meter 只负责处理 OTel 的 Metrics 信号（以及 Prometheus、Micrometer 等其他来源的指标）。

### 7. OAL 与 Prometheus 指标对照

SkyWalking 的 OAL 定义了很多指标，但这些指标也可以通过 Prometheus 格式暴露：

| OAL 指标 | 含义 | Prometheus 等价指标 |
|----------|------|-------------------|
| `service_cpm` | 服务每分钟调用量 | `rate(http_requests_total[1m])` |
| `service_resp_time` | 服务平均响应时间 | `rate(http_request_duration_seconds_sum[1m]) / rate(http_request_duration_seconds_count[1m])` |
| `service_sla` | 服务成功率 | `(sum(rate(http_requests_total{status!~"5.."}[1m])) / sum(rate(http_requests_total[1m]))) * 100` |
| `endpoint_cpm` | 端点每分钟调用量 | `rate(http_requests_total[1m])` by endpoint |
| `service_apdex` | 服务 Apdex | 需自定义计算 |

---

## 常见面试题

### Q1: Apdex 指标是什么？如何计算？

Apdex（Application Performance Index）是衡量用户满意度的标准化指标，范围 0~1。

**计算公式**：
```
Apdex = (满意数 + 0.5 × 可容忍数) / 总采样数
```

**阈值规则**（T = 满意阈值，通常 500ms）：
- 满意：RT ≤ T
- 可容忍：T < RT ≤ 4T
- 失望：RT > 4T

**优势**：将复杂的延迟分布转化为一个 0~1 的单一指标，便于设定 SLA 和告警。1.0 表示所有用户都满意，0.5 表示一半用户不满意。

### Q2: P99 和平均响应时间（RT）有什么区别？为什么 P99 更重要？

| 对比维度 | 平均响应时间（RT） | P99 |
|---------|-------------------|-----|
| 计算方式 | 总耗时 / 总请求数 | 排序后第 99 百分位 |
| 受极端值影响 | 容易被长尾拉高 | 不容易被极端值影响 |
| 反映的问题 | 整体趋势 | 长尾问题的严重程度 |
| 告警设计 | 波动大，不适合设阈值 | 适合设 SLA 告警阈值 |

**为什么 P99 更重要？**
- 平均值掩盖了长尾问题：99% 的用户响应用时在 10ms 以内，但 P99 可能达到 10s——平均值 109ms 看起来完全正常，实际上最慢的请求已经严重超时
- P99 直接标出了"最慢 1% 请求的耗时下限"，这与 SLA 设计一致（通常 SLA 承诺 99% 的请求在 Xms 内完成，P99 ≤ Xms 才算达标）

### Q3: SkyWalking 的 ServiceRelation 指标有什么特殊之处？

SkyWalking 的 Relation 指标最特殊的地方是**双向视角**：

- **Client 视角**：从调用方（Exit Span）统计，包含网络延迟
- **Server 视角**：从被调用方（Entry Span）统计，不含网络延迟

通过对比两个视角的差异，可以定位网络问题：
- Client 延迟 > Server 延迟 → 网络延迟较大
- Client 错误率 > Server 错误率 → 网络不可靠
- Client 调用量 ≠ Server 接收量 → 可能存在负载均衡问题

### Q4: Meter 指标和 Trace 指标有什么区别？

| 对比维度 | Trace 指标 | Meter 指标 |
|---------|-----------|-----------|
| 数据来源 | Agent 从请求链路中提取 | 外部系统推/拉 |
| 数据粒度 | 与请求相关（service/endpoint） | 任意维度 |
| 指标类型 | 预定义（OAL 脚本） | 自定义（MAL 脚本） |
| 典型场景 | HTTP 请求延迟、QPS、错误率 | JVM 指标、业务指标、系统指标 |
| 接入方式 | Agent 自动 | 需配置 MAL 规则 |

### Q5: 如果用 OpenTelemetry 采集指标走 Meter，那 OTel 的调用链怎么处理？

这是一个常见的误解。**OpenTelemetry 发送三种信号（Traces / Metrics / Logs），SkyWalking OAP 的 OTLP Receiver 会分别走三条管道：**

| OTel 信号 | SkyWalking 管道 | 分析引擎 | 产物 |
|-----------|----------------|---------|------|
| **Traces（Spans）** | Trace 管道 | OAL | service_cpm、endpoint_p99、拓扑图等 |
| **Metrics（Counter/Gauge/Histogram）** | Meter 管道 | MAL | 自定义业务指标 |
| **Logs（LogRecords）** | Log 管道 | LAL | 日志模式、错误率 |

**关键点**：OTel 的调用链（Traces）**不走 Meter 管道**，而是走 Trace 管道——OTLP Receiver 会把 OTel Span 转换成 SkyWalking 的 Segment/Span 模型，然后和 SkyWalking Agent 上报的数据**走同一套 OAL 聚合逻辑**。所以：

- 你用 OTel Agent 采集 → 调用链照样展示在 SkyWalking 拓扑图和 Trace 详情页中
- 你用 OTel Agent 采集 → Service/Endpoint/Relation 指标照样自动生成（OAL 分析）
- Meter 只是额外处理 OTel 的 Metrics 信号，不影响 Trace 的处理

**一句话**：OTel 的 Traces 和 Metrics 在 SkyWalking 中是**两条完全独立的管道**，互不干扰，也互不替代。

---

## 延伸阅读

- SkyWalking OAL 指标定义：`oap-server/server-starter/src/main/resources/oal/core.oal`
- Prometheus Fetcher 配置：`config/fetcher-promethus-config/`