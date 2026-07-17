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