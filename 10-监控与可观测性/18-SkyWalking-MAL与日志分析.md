# 18 - SkyWalking MAL 与日志分析

## 核心概念

### 1. MAL（Meter Analysis Language）

#### 1.1 MAL 是什么？

**MAL（Meter Analysis Language）** 是 SkyWalking 用于分析 **Meter（计量器）指标** 的 DSL。与 OAL（分析 Trace 数据）不同，MAL 专注于分析从外部系统接入的 Meter 数据。

```
OAL vs MAL 对比：

┌──────────────────────────────────────────────────────────────┐
│  OAL（Observability Analysis Language）                       │
│  ├── 数据来源：Trace Segment（Agent 上报）                    │
│  ├── 分析对象：Service/Endpoint/Relation 等 Source            │
│  ├── 输出：标准化的 APM 指标                                   │
│  └── 示例：service_cpm = from(Service.*).sum(1)              │
│                                                              │
│  MAL（Meter Analysis Language）                               │
│  ├── 数据来源：Meter 数据（Prometheus/OTel/Telegraf/Zabbix）  │
│  ├── 分析对象：自定义指标（Counter/Gauge/Histogram）           │
│  ├── 输出：标准化的 SkyWalking Meter 指标                      │
│  └── 示例：过滤和聚合 Prometheus 指标                          │
└──────────────────────────────────────────────────────────────┘
```

#### 1.2 MAL 配置示例

```yaml
# config/meter-analyzer-config/spring-sleuth.yaml
metricsRules:
  # 示例 1：分析 HTTP 请求指标
  - name: http_server_requests
    # 指标表达式（Prometheus 指标名）
    exp: http_server_requests_seconds_sum
    # 聚合维度（标签）
    group: ['service', 'uri', 'method']
    # 过滤条件
    filter: "tagEqual('status', '200')"
    # 聚合函数
    aggregation: sum

  # 示例 2：分析 JVM 指标
  - name: jvm_memory_used
    exp: jvm_memory_used_bytes
    group: ['service', 'area']  # area: heap/nonheap
    aggregation: avg

  # 示例 3：分析业务指标
  - name: order_count
    exp: order_created_total
    group: ['service', 'type']
    aggregation: sum
```

#### 1.3 MAL 处理流程

```
Meter 数据进入 OAP
  │
  ├── 1. MeterReceiver 接收原始 Meter 数据
  │     ├── Counter（累加值）
  │     ├── Gauge（瞬时值）
  │     └── Histogram（分布值）
  │
  ├── 2. MeterProcessor 根据 MAL 规则处理
  │     ├── 分组（group）：按标签维度分组
  │     ├── 过滤（filter）：过滤不符合条件的数据
  │     └── 聚合（aggregation）：sum/avg/max/min/p99
  │
  └── 3. 输出标准化 Meter 指标
        └── 写入存储（BanyanDB/ES/MySQL）
```

### 2. LAL（Log Analysis Language）

#### 2.1 LAL 是什么？

**LAL（Log Analysis Language）** 是 SkyWalking 用于**分析日志**的 DSL。它可以从原始日志中提取结构化信息，并与 Trace 关联。

```
LAL 的作用：
原始日志（非结构化）→ LAL 解析 → 结构化日志 + 指标提取

示例：
原始日志: "2024-07-17 10:00:00 [ERROR] UserService: User not found: id=123"
LAL 解析后:
  ├── timestamp: 2024-07-17 10:00:00
  ├── level: ERROR
  ├── service: UserService
  ├── message: User not found: id=123
  └── 提取指标: error_count += 1
```

#### 2.2 LAL 脚本示例

```groovy
// config/lal/lal.yaml
rules:
  - name: "nginx-access-log"
    dsl: |
      // 定义过滤条件
      filter {
        // 提取日志内容
        text {
          // 正则匹配
          regexp "(?<remoteIp>\\S+) - (?<user>\\S+) \\[(?<timestamp>[^\\]]+)\\] \"(?<method>\\S+) (?<path>\\S+) [^\"]*\" (?<status>\\d+) (?<bodyBytes>\\d+)"
        }
        // 解析器链
        parser {
          // 解析 JSON 日志
          json {
          }
        }
        // 提取器
        extractor {
          // 提取标签
          tag 'level': parsed.level
          tag 'service': parsed.service
          // 提取指标
          metrics {
            // 错误计数
            counter 'error_count': 1
            // 响应时间
            histogram 'response_time': parsed.responseTime
          }
        }
        // 采样策略
        sampler {
          // 每 10 条采样 1 条
          rate 10
        }
      }
```

#### 2.3 LAL 与 Trace 关联

LAL 支持将日志与 Trace 关联：

```groovy
// 从日志中提取 TraceId，与 Trace 关联
filter {
  extractor {
    // 如果日志中包含 TraceId
    if (parsed.traceId != null) {
      // 设置 Trace 上下文
      traceContext {
        traceId: parsed.traceId
        spanId: parsed.spanId
      }
    }
  }
}
```

### 3. 日志桥接（Logback/Log4j2 → SkyWalking）

#### 3.1 TraceId 注入 MDC

SkyWalking Java Agent 自动将 TraceId 注入到 SLF4J 的 **MDC（Mapped Diagnostic Context）**：

```java
// 无需任何代码改动，Agent 自动注入
// Logback 配置中使用 %tid 即可输出 TraceId
```

```xml
<!-- logback.xml -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <encoder>
        <!-- %tid 是 SkyWalking 自动注入的 TraceId -->
        <pattern>[%tid] %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

#### 3.2 gRPC Log Reporter

SkyWalking Agent 支持通过 gRPC 将日志直接上报到 OAP：

```properties
# agent.config
plugin.toolkit.log.grpc.reporter.server_host=${SW_GRPC_LOG_SERVER_HOST:127.0.0.1}
plugin.toolkit.log.grpc.reporter.server_port=${SW_GRPC_LOG_SERVER_PORT:11800}
plugin.toolkit.log.grpc.reporter.max_message_size=${SW_GRPC_LOG_MAX_MESSAGE_SIZE:10485760}
plugin.toolkit.log.grpc.reporter.upstream_timeout=${SW_GRPC_LOG_GRPC_UPSTREAM_TIMEOUT:30}
```

### 4. MQE（Metrics Query Engine）

#### 4.1 MQE 是什么？

**MQE（Metrics Query Engine）** 是 SkyWalking v9+ 引入的**指标查询引擎**，提供类 SQL 的查询语法。

```
MQE 查询示例：

// 查询 order-service 最近 5 分钟的平均响应时间
service_resp_time{service='order-service'}.avg(5m)

// 查询 Top 10 端点（按 P99 延迟）
endpoint_p99{service='order-service'}.top(10, desc)

// 查询两个服务的指标对比
service_cpm{service='order-service'}.compare(service_cpm{service='user-service'})
```

#### 4.2 MQE 语法

```
基本语法：<metric_name>{<label_selector>}.<function>(<args>)

支持的函数：
├── avg(duration)     → 平均值
├── sum(duration)     → 求和
├── max(duration)     → 最大值
├── min(duration)     → 最小值
├── p99(duration)     → 99 百分位
├── top(n, order)     → Top N
├── bottom(n, order)  → Bottom N
├── rate(duration)    → 速率
├── increase(duration)→ 增量
└── compare(metric)   → 对比
```

---

## 常见面试题

### Q1: OAL、MAL、LAL 三者有什么区别？

| 语言 | 全称 | 数据来源 | 作用 |
|------|------|---------|------|
| **OAL** | Observability Analysis Language | Trace Segment | 分析 Trace 数据，生成 APM 指标 |
| **MAL** | Meter Analysis Language | Meter（Prometheus/OTel） | 分析外部 Meter 指标 |
| **LAL** | Log Analysis Language | 原始日志 | 解析日志，提取结构化信息和指标 |

**三者关系**：OAL 处理 Trace 来源的指标，MAL 处理外部 Meter 来源的指标，LAL 处理日志来源的指标。三者互补，覆盖可观测性三大支柱（Traces、Metrics、Logs）。

### Q2: 如何将 TraceId 关联到日志中？

1. **Agent 自动注入**：SkyWalking Agent 自动将 TraceId 注入到 SLF4J MDC
2. **日志配置**：在 Logback/Log4j2 配置中使用 `%tid` 占位符输出 TraceId
3. **日志上报**：通过 gRPC Log Reporter 将日志上报到 OAP
4. **UI 关联**：在 SkyWalking UI 中，可以从 Trace 详情直接跳转到关联的日志

### Q3: MQE 和 GraphQL 查询有什么区别？

| 对比维度 | GraphQL 查询 | MQE 查询 |
|---------|-------------|---------|
| 查询方式 | 结构化的 API 查询 | 类 SQL 的表达式查询 |
| 复杂度 | 需要指定完整的查询结构 | 灵活的表达式 |
| 学习成本 | 需要理解 GraphQL Schema | 接近 SQL 语法 |
| 使用场景 | UI 查询、程序化查询 | 命令行查询、临时分析 |
| 聚合能力 | 有限 | 强大的聚合函数 |

---

## 延伸阅读

- MAL 配置文档：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/meter-analyzer/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/meter-analyzer/)
- LAL 配置文档：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/log-analyzer/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/log-analyzer/)
- MQE 语法文档：[https://skywalking.apache.org/docs/main/latest/en/api/metrics-query-expression/](https://skywalking.apache.org/docs/main/latest/en/api/metrics-query-expression/)