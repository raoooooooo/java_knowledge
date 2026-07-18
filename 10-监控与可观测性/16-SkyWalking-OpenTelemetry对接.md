# 16 - SkyWalking 与 OpenTelemetry 对接

## 核心概念

### 1. OpenTelemetry 是什么？

**OpenTelemetry（OTel）** 是 CNCF 孵化的**可观测性统一标准**，旨在提供一套与厂商无关的 API、SDK 和工具，用于生成、收集和导出遥测数据（Traces、Metrics、Logs）。

```
OpenTelemetry 的核心组件：

┌──────────────────────────────────────────────────────────────────┐
│  OpenTelemetry 架构                                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  API（接口定义）                                           │   │
│  │  ├── Trace API：创建 Span，管理上下文传播                   │   │
│  │  ├── Metrics API：创建 Counter/Gauge/Histogram             │   │
│  │  └── Logs API：日志桥接                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SDK（具体实现）                                           │   │
│  │  ├── TracerProvider：Span 处理器 + 导出器                  │   │
│  │  ├── MeterProvider：Metric Reader + 导出器                 │   │
│  │  └── LoggerProvider：Log Record 处理器 + 导出器            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Collector（收集器）                                        │   │
│  │  ├── Receiver：接收 OTLP/Zipkin/Jaeger 数据                │   │
│  │  ├── Processor：批处理/过滤/采样/转换                       │   │
│  │  └── Exporter：导出到 SkyWalking/Jaeger/Prometheus/...     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 2. SkyWalking 与 OTel 的关系

SkyWalking v9+ **原生支持** OpenTelemetry 协议，主要体现在：

```
┌──────────────────────────────────────────────────────────────────┐
│  SkyWalking 与 OTel 的集成点                                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ① OTLP Receiver（OAP 内置）                                     │
│     └── OAP 可以直接接收 OTel SDK 发送的 OTLP 数据               │
│                                                                   │
│  ② OTel Java Agent 兼容                                          │
│     └── 使用 OTel Java Agent 的应用，数据可以发送到 SkyWalking    │
│                                                                   │
│  ③ SkyWalking Agent → OTel Collector                             │
│     └── SkyWalking Agent 的数据可以转发到 OTel Collector          │
│                                                                   │
│  ④ Prometheus Fetcher                                            │
│     └── OAP 可以通过 Prometheus 拉取 OTel SDK 的 Metrics          │
│                                                                   │
│  ⑤ W3C TraceContext 兼容                                         │
│     └── SkyWalking 支持 W3C TraceContext 传播协议                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 3. 三种混合架构方案

#### 3.1 方案 A：SkyWalking Agent + OAP + OTel Backend

```
┌──────────────────┐
│ SkyWalking Agent │──→ SkyWalking OAP (Trace/Logs)
│  (Java 应用)      │
└──────────────────┘
         │
         │ OTLP Exporter
         ▼
┌──────────────────┐
│ OTel Collector   │──→ OTel Backend (Jaeger/Prometheus)
└──────────────────┘
```

**适用场景**：已有 SkyWalking 部署，想逐步引入 OTel 生态。

#### 3.2 方案 B：OTel SDK + OAP

```
┌──────────────────┐
│  OTel Java Agent │──→ OTLP ──→ SkyWalking OAP
│  (Java 应用)      │              (OTLP Receiver)
└──────────────────┘
```

**适用场景**：已经使用 OTel SDK 的应用，想利用 SkyWalking 的分析和 UI 能力。

#### 3.3 方案 C：全链路 OTel

```
┌──────────────────┐     OTLP      ┌──────────────┐
│  OTel Agent/SDK  │──────────────→│ OTel Collector│
└──────────────────┘               └───────┬──────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
            ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
            │  SkyWalking  │     │   Jaeger     │     │  Prometheus  │
            │  (OTLP Rcvr) │     │  (OTLP Rcvr) │     │  (OTLP Rcvr) │
            └──────────────┘     └──────────────┘     └──────────────┘
```

**适用场景**：标准化可观测性平台，所有数据走 OTLP 协议。

### 4. OTLP Receiver 配置

```yaml
# config/application.yml
receiver-otel:
  selector: ${SW_OTEL_RECEIVER:default}
  default:
    # OTLP gRPC 接收端口
    gRPCHost: ${SW_OTEL_RECEIVER_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_OTEL_RECEIVER_GRPC_PORT:4317}
    # OTLP HTTP 接收端口
    restHost: ${SW_OTEL_RECEIVER_REST_HOST:0.0.0.0}
    restPort: ${SW_OTEL_RECEIVER_REST_PORT:4318}
    # 是否启用
    enabledHandlers: ${SW_OTEL_RECEIVER_ENABLED_HANDLERS:"oc,otlp"}
    # 是否启用 OTel 指标
    enabledOtelMetricsRules: ${SW_OTEL_RECEIVER_ENABLED_OTEL_METRICS_RULES:"default"}
```

### 5. sw8 协议 vs W3C TraceContext 协议

| 对比维度 | sw8（SkyWalking 原生） | W3C TraceContext（OTel 标准） |
|---------|----------------------|------------------------------|
| Header Key | `sw8` | `traceparent` + `tracestate` |
| 编码格式 | Base64 编码 | 明文（Hex） |
| 传播内容 | TraceId + 父服务信息 | TraceId + SpanId + TraceFlags |
| 标准化 | SkyWalking 自研 | W3C 国际标准 |
| 生态兼容 | 仅 SkyWalking 生态 | 所有遵循 W3C 标准的工具 |
| OTel 支持 | 需转换 | 原生支持 |

**SkyWalking 的兼容策略**：同时支持两种协议，自动检测和转换。

#### 5.1 OTel TraceId → SkyWalking 的完整转换流程

当 OTel SDK 上报 Trace 数据到 SkyWalking OAP 时，转换发生在**两个层面**：**跨服务传播时的 Header 转换**，和 **OAP 接收时的内部格式转换**。

**层面 1：跨服务传播时（Header 转换）**

```
┌──────────────────────────────────────────────────────────────────┐
│  场景：SkyWalking 服务 A 调用 OTel 服务 B（sw8 → W3C）             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Service A（SW Agent）发起 HTTP 调用：                              │
│    Header: sw8 = "1-4bf92f3577b34da6a3ce929d0e0e4736.1.1690000000000-3..."
│                                                                   │
│  ↓ 网关/代理层做协议转换（或者 OTel Agent 自动识别 sw8）            │
│                                                                   │
│  Service B（OTel Agent）接收：                                      │
│    Header: traceparent = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
│                                                                   │
│  转换公式：                                                        │
│    W3C traceId = sw8 traceId 中第一个 "-" 之前的部分（去掉 .threadId.timestamp）│
│    W3C parentId = sw8 segmentId（UUID，取前 16 字节）              │
│    W3C trace-flags = sw8 sample 标志（0 或 1）                     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

反向转换（W3C → sw8）同理：

```
traceparent = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
              ↓
sw8 traceId = "4bf92f3577b34da6a3ce929d0e0e4736.0." + 当前时间戳
              (保留 OTel 原始 32 位 hex)  (补 0 占位)   (补当前时间戳)
```

> ⚠️ **关键结论**：**OTel 的 32 位 hex TraceId 会原样保留**，SkyWalking 不会生成新的 TraceId。只是在需要 sw8 格式时，会在后面补上 `.0.时间戳`（让格式看起来像 SkyWalking 原生的），但存储和查询时用的还是 OTel 原始的 32 位 hex。

**层面 2：OAP 接收时（OTLP Receiver 内部转换）**

OTLP Receiver 收到 OTel 的 `ExportTraceServiceRequest` 后，会做如下转换：

```java
// 伪代码：OTel Span → SkyWalking SegmentObject
public SegmentObject convert(InstrumentationLibrarySpans otelSpans) {

    // 1. TraceId 直接复用（OTel 原生 32 位 hex）
    String swTraceId = otelSpan.getTraceId();  // 不做任何转换！

    // 2. SpanId 也直接复用（OTel SpanId 是 16 位 hex）
    int swSpanId = hash(otelSpan.getSpanId());  // 哈希成整数（SW 要求 int）

    // 3. parentSpanId 转换：
    //    OTel parentSpanId（16 位 hex）→ 哈希成整数
    int swParentSpanId = hash(otelSpan.getParentSpanId());

    // 4. 其他字段映射：
    //    otelSpan.getName() → operationName
    //    otelSpan.getKind() → SpanType（Entry/Exit/Local）
    //    otelSpan.getAttributes() → tags
    //    otelSpan.getEvents() → logs
    //    otelSpan.getStatus() → isError

    return SegmentObject.newBuilder()
        .setTraceId(swTraceId)       // OTel 原生 TraceId！
        .setTraceSegmentId(...)      // 生成新的 SegmentId（UUID）
        .setSpans(spans)
        .build();
}
```

**为什么不重新生成 TraceId？**

```
如果 OAP 重新生成 TraceId，会发生什么？

  Service A（SW Agent）: traceId = abc.1.1690000000000
        ↓ 调用
  Service B（OTel Agent）: traceId = 4bf92f3577b34da6a3ce929d0e0e4736
        ↓ OAP 重新生成
  OAP 存储的 traceId = xyz.2.1690000000001

结果：两个 Segment 的 traceId 不一样 → 无法组装成一条完整的 Trace → 断链！
```

**核心原则**：**TraceId 必须是跨服务传播的，不是 OAP 生成的。** 传播协议（sw8 或 W3C）负责把 TraceId 从上游带到下游，OAP 只负责存储，不负责生成。

#### 5.2 混合部署的三种协议模式

| 模式 | 所有服务用什么协议？ | OAP 怎么处理？ | 适用场景 |
|------|-------------------|---------------|---------|
| **纯 sw8 模式** | 所有服务用 SkyWalking Agent（sw8） | 原生处理，不需要转换 | 传统 SW 部署 |
| **纯 W3C 模式** | 所有服务用 OTel Agent（W3C TraceContext） | OTLP Receiver 原生接收 | 新建系统，全 OTel 栈 |
| **混合模式** | 一部分 sw8，一部分 W3C | 网关层做协议转换，OAP 兼容 | 迁移过渡阶段 |

**混合模式的最佳实践**：

```
最佳方案：在网关层统一转换（推荐用 OTel Collector）

  入口流量（W3C 协议）
       │
       ▼
  OTel Collector / API Gateway
       │
       ├──→ sw8 协议 → SkyWalking Agent 服务
       └──→ W3C 协议 → OTel Agent 服务

这样整条链路的 TraceId 完全一致，不会出现断链。
```

### 6. 迁移策略

```
阶段 1（当前）：纯 SkyWalking 生态
  ├── Java Agent 使用 sw8 协议
  └── OAP 接收 sw8 协议数据

阶段 2（过渡）：SkyWalking + OTel 混合
  ├── 部分服务使用 SkyWalking Agent（sw8）
  ├── 部分服务使用 OTel Agent（OTLP）
  ├── OAP 同时接收 sw8 和 OTLP 数据
  └── 两种协议在 OAP 中统一处理

阶段 3（目标）：全链路 OTel
  ├── 所有服务使用 OTel Agent/SDK
  ├── OTel Collector 统一接收和处理
  └── SkyWalking 作为 OTel 的后端分析平台
```

### 7. SkyWalking 与 Prometheus 集成

#### 7.1 Prometheus Fetcher（拉取模式）

```yaml
# config/fetcher-prometheus-config/prometheus-fetcher.yml
fetcher-prometheus:
  selector: default
  default:
    target: http://prometheus:9090
    # 拉取间隔
    interval: 60
    # 拉取规则
    rules:
      - name: "http_requests"
        query: "rate(http_requests_total[5m])"
```

#### 7.2 Meter → Prometheus Exporter（推送模式）

SkyWalking 的 Meter 指标可以通过 Prometheus Exporter 暴露：

```yaml
# 配置后，SkyWalking 的指标可以暴露为 Prometheus 格式
# 访问 http://oap:1234/metrics 获取指标
exporter:
  prometheus:
    host: 0.0.0.0
    port: 1234
```

---

## 常见面试题

### Q1: SkyWalking 和 OpenTelemetry 是什么关系？

- **SkyWalking**：完整的 APM 平台（Agent + 后端 + UI + 存储）
- **OpenTelemetry**：可观测性标准（API + SDK + Collector + 协议）

**关系**：SkyWalking 是 OTel 的**后端实现之一**。SkyWalking 可以接收 OTel SDK 的数据，也可以用 OTel Collector 替代 SkyWalking Agent 的部分功能。

**趋势**：SkyWalking 正在深度融合 OTel 生态，v9+ 原生支持 OTLP 协议。

### Q2: sw8 协议和 W3C TraceContext 协议有什么区别？如何选择？

| 场景 | 推荐协议 |
|------|---------|
| 纯 SkyWalking 生态 | sw8（性能更好，信息更丰富） |
| 混合 SkyWalking + OTel | W3C TraceContext（兼容性更好） |
| 全链路 OTel | W3C TraceContext（标准协议） |
| 多厂商混合 | W3C TraceContext（唯一选择） |

### Q3: 如何从 SkyWalking 迁移到 OpenTelemetry？

1. **不急于全面替换**：SkyWalking Agent 和 OTel Agent 可以共存
2. **先迁移新服务**：新服务使用 OTel Agent，老服务保持不变
3. **OAP 开启 OTLP Receiver**：同时接收 sw8 和 OTLP 数据
4. **逐步迁移老服务**：按优先级逐个迁移
5. **最终统一**：全量迁移后，可考虑关闭 sw8 协议

### Q4: SkyWalking 和 Prometheus 如何配合使用？

- **SkyWalking**：负责 Trace + 应用级指标（服务/端点/JVM）
- **Prometheus**：负责基础设施指标（主机/容器/网络）
- **Grafana**：统一展示 SkyWalking 和 Prometheus 的数据

**集成方式**：
1. SkyWalking 通过 Prometheus Fetcher 拉取 Prometheus 指标到 OAP
2. SkyWalking 的指标通过 Prometheus Exporter 暴露给 Prometheus
3. Grafana 配置 SkyWalking DataSource 和 Prometheus DataSource，统一展示

---

## 延伸阅读

- OpenTelemetry 官方文档：[https://opentelemetry.io/docs/](https://opentelemetry.io/docs/)
- SkyWalking OTel Receiver 配置：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/otel-receiver/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/otel-receiver/)