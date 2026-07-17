# 11 - SkyWalking 数据采集与传输

## 核心概念

### 1. 数据采集架构全景

```
┌──────────────────────────────────────────────────────────────────┐
│                    数据采集与传输架构                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐                                             │
│  │   Java Agent     │──┐                                         │
│  └─────────────────┘  │                                         │
│                        │  gRPC (sw8协议)                          │
│  ┌─────────────────┐  │  ┌──────────────────────────────────┐   │
│  │   Python Agent   │──┼→│  DataCarrier（内存队列）           │   │
│  └─────────────────┘  │  │  ┌──────────────────────────────┐│   │
│                        │  │  │  Disruptor RingBuffer        ││   │
│  ┌─────────────────┐  │  │  │  ├── TraceBuffer              ││   │
│  │   Go Agent       │──┘  │  │  ├── LogBuffer               ││   │
│  └─────────────────┘      │  │  ├── MeterBuffer             ││   │
│                            │  │  └── EventBuffer             ││   │
│  ┌─────────────────┐      │  └──────────────┬───────────────┘│   │
│  │  OpenTelemetry   │──┐  │                 │ 批量消费         │   │
│  │  OTLP 数据        │  │  │                 ▼                │   │
│  └─────────────────┘  │  │  ┌──────────────────────────────┐│   │
│                        │  │  │  Analyzer 分析引擎            ││   │
│  ┌─────────────────┐  │  │  │  ├── TraceAnalyzer            ││   │
│  │  Kafka 消息       │──┼─→│  │  ├── LogAnalyzer             ││   │
│  └─────────────────┘  │  │  │  ├── MeterProcessor           ││   │
│                        │  │  │  └── EventAnalyzer            ││   │
│  ┌─────────────────┐  │  │  └──────────────┬───────────────┘│   │
│  │  Prometheus      │──┘  │                 │ 指标聚合         │   │
│  │  Fetcher 拉取     │     │                 ▼                │   │
│  └─────────────────┘      │  ┌──────────────────────────────┐│   │
│                            │  │  Storage 存储层              ││   │
│                            │  └──────────────────────────────┘│   │
│                            └──────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 2. 上报协议对比

| 协议 | 传输方式 | 适用场景 | 默认端口 | 序列化 | 背压 |
|------|---------|---------|---------|--------|------|
| **gRPC** | 长连接 + 流式 | 默认，Agent 直连 OAP | 11800 | Protobuf | 内置流控 |
| **Kafka** | 消息队列 | 高吞吐、解耦 Agent 和 OAP | 9092 | Protobuf | Kafka 消费组 |
| **HTTP** | REST API | 轻量级、防火墙友好 | 12800 | JSON | 无 |

### 3. gRPC 上报（默认方式）

#### 3.1 通信模型

```
Agent 端                                    OAP 端
  │                                           │
  │ ① ServiceRegister  (服务注册)              │
  │──────────────────────────────────────────→│
  │                    注册成功，返回 ServiceId  │
  │←──────────────────────────────────────────│
  │                                           │
  │ ② ServiceInstanceRegister (实例注册)       │
  │──────────────────────────────────────────→│
  │                 注册成功，返回 InstanceId   │
  │←──────────────────────────────────────────│
  │                                           │
  │ ③ KeepAlive (心跳，每 30 秒)               │
  │──────────────────────────────────────────→│
  │                                           │
  │ ④ TraceSegmentReport (上报 Segment)        │
  │──────────────────────────────────────────→│
  │   (流式上报，持续发送)                       │
  │                                           │
  │ ⑤ JVMMetricReport (上报 JVM 指标)          │
  │──────────────────────────────────────────→│
  │   (每 30 秒上报一次)                        │
  │                                           │
  │ ⑥ LogReport (上报日志)                     │
  │──────────────────────────────────────────→│
  │                                           │
```

#### 3.2 Agent 注册流程

1. **服务注册**：Agent 启动时，向 OAP 注册 Service（服务名 → ServiceId）
2. **实例注册**：注册 ServiceInstance（IP:Port → InstanceId）
3. **心跳保活**：每 30 秒发送心跳，OAP 超时 90 秒未收到心跳则标记实例下线
4. **数据上报**：注册成功后，开始持续上报 Trace/Metrics/Logs

### 4. Kafka 桥接模式

#### 4.1 为什么需要 Kafka？

- **解耦**：Agent 和 OAP 通过 Kafka 解耦，OAP 离线不影响 Agent 上报
- **削峰填谷**：Kafka 缓冲突发流量，保护 OAP 不被冲垮
- **多消费者**：多个 OAP 实例可以消费同一个 Topic，实现水平扩展

#### 4.2 Kafka 桥接架构

```
┌──────────┐    ┌──────────┐          ┌─────────────┐
│  Agent 1 │───→│          │─────────→│  OAP 1 (消费)│
└──────────┘    │          │          └─────────────┘
                │  Kafka   │
┌──────────┐    │  Topic:  │          ┌─────────────┐
│  Agent 2 │───→│  sw-trace│─────────→│  OAP 2 (消费)│
└──────────┘    │          │          └─────────────┘
                └─────────────┘
```

#### 4.3 Kafka Fetcher 配置

```yaml
# config/application.yml
kafka-fetcher-plugin:
  selector: ${SW_KAFKA_FETCHER:default}
  default:
    bootstrapServers: ${SW_KAFKA_FETCHER_SERVERS:localhost:9092}
    partitions: ${SW_KAFKA_FETCHER_PARTITIONS:3}
    # 按 Topic 消费不同数据类型
    topics:
      trace: ${SW_KAFKA_FETCHER_TOPIC_TRACE:sw-trace}
      metrics: ${SW_KAFKA_FETCHER_TOPIC_METRICS:sw-metrics}
      logs: ${SW_KAFKA_FETCHER_TOPIC_LOGS:sw-logs}
```

### 5. DataCarrier（内存队列）

#### 5.1 设计原理

DataCarrier 是 SkyWalking OAP 中的**高性能内存队列**，用于解耦数据接收和数据分析：

```
                         ┌─────────────────────────────────┐
                         │         DataCarrier              │
gRPC Receiver ──────────→│  ┌───────────────────────────┐  │──→ Analyzer
(接收线程)               │  │  Disruptor RingBuffer       │  │    (分析线程)
                         │  │  ├── Channel 0 (Trace)      │  │
                         │  │  ├── Channel 1 (Metrics)    │  │
                         │  │  ├── Channel 2 (Logs)       │  │
                         │  │  └── Channel 3 (Events)     │  │
                         │  └───────────────────────────┘  │
                         └─────────────────────────────────┘
```

**为什么用 RingBuffer？**
- 无锁设计（CAS 操作），极高的并发性能
- 预分配内存，避免 GC 压力
- 批量消费，减少线程切换开销

#### 5.2 背压机制

```
当 DataCarrier 满时：
  1. 接收线程尝试写入 → 失败
  2. 根据策略处理：
     ┌── BLOCKING（阻塞）：等待直到有空间
     ├── IF_POSSIBLE（尽力）：写入失败则丢弃
     └── OVERRIDE（覆盖）：覆盖最旧的数据
  3. 默认策略：IF_POSSIBLE（不阻塞 OAP 接收线程）
```

### 6. 批量处理与 Buffer

#### 6.1 TraceBuffer

Agent 端的 Trace Segment 不是每个 Span 立即上报，而是等到整个 Segment 完成后再上报：

```
Agent 端：
  Span 创建 → Span 完成 → 放入 TracingContext
     ↓
  Segment 所有 Span 完成 → 序列化 Segment → 放入 TraceBuffer
     ↓
  TraceBuffer 攒够一批 → 批量上报到 OAP（gRPC 流式发送）
```

#### 6.2 LogBuffer

```
Agent 端：
  Log 事件 → 放入 LogBuffer
     ↓
  LogBuffer 攒够一批 → 批量上报到 OAP
```

---

## 常见面试题

### Q1: SkyWalking 的数据上报有哪些方式？各自优缺点？

| 方式 | 优点 | 缺点 |
|------|------|------|
| **gRPC 直连** | 低延迟、配置简单、默认方式 | OAP 故障时数据丢失 |
| **Kafka 桥接** | 解耦、削峰填谷、OAP 离线不影响 | 引入 Kafka 依赖、增加运维成本 |
| **HTTP** | 防火墙友好、无需 gRPC 依赖 | 性能较差、无流控 |

**推荐**：小规模用 gRPC 直连，大规模用 Kafka 桥接。

### Q2: DataCarrier 的 RingBuffer 设计为什么能提高性能？

1. **无锁设计**：通过 CAS 操作实现生产者和消费者的并发安全，避免了锁竞争
2. **预分配内存**：RingBuffer 在初始化时分配固定大小的数组，运行时不会产生 GC 压力
3. **缓存友好**：数组在内存中连续存储，CPU 缓存命中率高
4. **批量消费**：消费者可以批量拉取多个元素，减少上下文切换

### Q3: Agent 注册流程是怎样的？什么时候认为实例下线？

1. Agent 启动 → 注册 Service（服务名 → ServiceId）
2. 注册 ServiceInstance（IP:Port → InstanceId）
3. 每 30 秒发送心跳（KeepAlive）
4. OAP 端设置心跳超时时间（默认 90 秒）
5. 超过 90 秒未收到心跳 → 标记实例为下线 → 从拓扑图中移除

### Q4: Kafka 桥接模式下，如何保证数据不丢失？

1. **Kafka 持久化**：数据写入 Kafka 后持久化到磁盘
2. **消费确认**：OAP 消费成功后提交 offset
3. **重试机制**：消费失败时重试，超出重试次数后进入死信队列
4. **多副本**：Kafka Topic 配置多副本，Broker 故障不影响数据

---

## 延伸阅读

- SkyWalking OAP 接收器源码：`oap-server/server-receiver-plugin/`
- Kafka Fetcher 源码：`oap-server/server-fetcher-plugin/kafka-fetcher-plugin/`
- DataCarrier 源码：`oap-server/server-library/library-datacarrier/`