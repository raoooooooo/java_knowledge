# 11 - SkyWalking 数据采集与传输

## 核心概念

### 1. 数据采集架构全景

SkyWalking 的数据上报链路可以归纳为**三条管道 + 四种数据来源**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SkyWalking 数据上报全链路                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────── Agent 端 ──────────────────────────────┐   │
│  │                                                                    │   │
│  │  业务应用 (Java Agent)                                              │   │
│  │  │                                                                 │   │
│  │  ├── ① Trace 数据 (Segment/Span)  ──────────┐                     │   │
│  │  │   • 调用链原始数据，每个请求一个 Segment      │                     │   │
│  │  │   • 请求驱动，有请求才有数据                 │                     │   │
│  │  │                                           │                     │   │
│  │  ├── ② JVM 指标 (Metrics)  ──────────────────┤                     │   │
│  │  │   • 内存、GC、CPU、线程                     │                     │   │
│  │  │   • 定时采集（每 30s），非请求驱动           │                     │   │
│  │  │                                           │    gRPC (:11800)    │   │
│  │  └── ③ 日志 (Log)  ──────────────────────────┤   或 Kafka          │   │
│  │      • logback/log4j bridge 注入 traceId      │                     │   │
│  │      • 与调用链关联                            │                     │   │
│  │                                              │                     │   │
│  └──────────────────────────────────────────────┼─────────────────────┘   │
│                                                  │                        │
│  ┌────────────────── 外部数据源 ──────────────────┤──────────────────┐   │
│  │                                               │                   │   │
│  │  ④ Meter 自定义指标 ───────────────────────────┘                   │   │
│  │  • OTel Metrics ──── OTLP (:4317) ────┐                           │   │
│  │  • Prometheus ────── HTTP Pull ───────┤                           │   │
│  │  • Micrometer ────── 桥接 ────────────┤                           │   │
│  │  • Telegraf/Zabbix ─ Receiver ────────┤                           │   │
│  │                                        │                           │   │
│  └────────────────────────────────────────┼───────────────────────────┘   │
│                                            │                              │
│  ┌─────── OAP 端 ──────────────────────────┼─────────────────────────┐   │
│  │                                         ▼                          │   │
│  │  ┌──────────────────────────────────────────────────────────────┐ │   │
│  │  │  Receiver 接收层                                              │ │   │
│  │  │  ├── TraceReceiver (gRPC/Kafka)  ← 接收 Agent 的 ①②③       │ │   │
│  │  │  └── OTLPReceiver (gRPC :4317)   ← 接收 OTel 三信号          │ │   │
│  │  └──────────────────────────────────────────────────────────────┘ │   │
│  │                              │                                     │   │
│  │  ┌───────────────────────────┼──────────────────────────────────┐ │   │
│  │  │  DataCarrier（内存队列，Disruptor RingBuffer）                 │ │   │
│  │  │  ├── TraceBuffer    ├── MetricsBuffer                         │ │   │
│  │  │  ├── LogBuffer       └── EventBuffer                          │ │   │
│  │  └───────────────────────────┼──────────────────────────────────┘ │   │
│  │                              │ 批量消费                             │   │
│  │         ┌────────────────────┼────────────────────┐               │   │
│  │         ▼                    ▼                     ▼              │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │                    三条分析管道                              │  │   │
│  │  │                                                            │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │   │
│  │  │  │ ① Trace 管道  │  │ ② Metrics    │  │ ③ Log 管道   │     │  │   │
│  │  │  │              │  │   管道        │  │              │     │  │   │
│  │  │  │ TraceAnalyzer│  │              │  │ LogAnalyzer  │     │  │   │
│  │  │  │     ↓        │  │ ② JVM 指标   │  │     ↓        │     │  │   │
│  │  │  │ OAL 引擎     │  │   直接存储    │  │ LAL 引擎     │     │  │   │
│  │  │  │     ↓        │  │              │  │     ↓        │     │  │   │
│  │  │  │ Trace 指标   │  │ ④ Meter      │  │ 日志指标     │     │  │   │
│  │  │  │ (service_cpm,│  │   MAL 引擎   │  │ 日志记录     │     │  │   │
│  │  │  │  endpoint_   │  │     ↓        │  │              │     │  │   │
│  │  │  │  p99,        │  │ Meter 指标   │  │              │     │  │   │
│  │  │  │  apdex...)   │  │ (自定义)     │  │              │     │  │   │
│  │  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │  │   │
│  │  └─────────┼─────────────────┼─────────────────┼─────────────┘  │   │
│  │            │                 │                 │                 │   │
│  │            ▼                 ▼                 ▼                 │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  Storage 存储层 (H2 / MySQL / ES / BanyanDB)                │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                                                                   │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**管道汇总**：

| 管道 | 数据来源 | 谁上报 | 分析引擎 | 产物 |
|------|---------|--------|---------|------|
| **① Trace 管道** | 调用链 Span（Agent 拦截） | Agent | TraceAnalyzer → OAL | service_cpm、endpoint_p99、apdex、拓扑图 |
| **② Metrics 管道** | JVM 指标（Agent 定时采集）+ Meter 自定义指标（外部系统） | Agent + OTel/Prometheus/Micrometer | JVM 直接存储 / Meter → MAL | JVM 面板 + 自定义业务指标 |
| **③ Log 管道** | 日志（Agent 桥接 logback/log4j） | Agent | LogAnalyzer → LAL | 日志记录 + 日志模式指标 |

> ⚠️ **关键区分**：Trace 指标（OAL 产出）和 Meter 指标（MAL 产出）虽然都叫"指标"，但来源完全不同。Trace 指标是从 Trace 数据里**算出来的**（请求驱动），Meter 指标是从外部系统**采来的**（定时驱动）。详见第 6 章 6.4 节。

### 2. 上报协议对比

| 协议 | 传输方式 | 适用场景 | 默认端口 | 序列化 | 背压 |
|------|---------|---------|---------|--------|------|
| **gRPC** | 长连接 + 流式 | 默认，Agent 直连 OAP | 11800 | Protobuf | 内置流控 |
| **Kafka** | 消息队列 | 高吞吐、解耦 Agent 和 OAP | 9092 | Protobuf | Kafka 消费组 |
| **HTTP** | REST API | 轻量级、防火墙友好 | 12800 | JSON | 无 |

#### 2.1 灵魂拷问：Agent 数量多了，三种协议的压力怎么办？ ⭐⭐⭐

这是大规模部署（几百上千个 Agent）绕不开的问题。三种协议在"连接压力"和"负载均衡"上表现完全不同：

---

**① gRPC 直连：Agent 多了 OAP 压力大吗？怎么负载均衡？**

**会很大！** 因为 gRPC 是**长连接**，每个 Agent 都会和 OAP 建立一条 TCP 长连接。1000 个 Agent = 1000 条长连接，OAP 要同时维护这些连接。

```
压力来源：
  ├── 连接数压力：1000 Agent = 1000 条长连接，OAP 的 fd（文件描述符）会吃紧
  ├── 心跳压力：每个 Agent 每 30s 一次心跳，1000 Agent = 每秒 ~33 次心跳
  ├── 流式上报压力：每条连接持续上报数据，OAP 要并发处理
  └── 注册压力：所有 Agent 启动时同时注册，可能形成"注册风暴"
```

**怎么负载均衡？两种方案：**

```
方案A：客户端负载均衡（SkyWalking 默认）
  Agent 端配置多个 OAP 地址，自己轮询选一个连：

  agent.service_name=xxx
  collector.backend_service=oap1:11800,oap2:11800,oap3:11800
                                       ↑
                            Agent 启动时随机/轮询选一个 OAP 连接

  特点：Agent 主动选 OAP，不需要 LB 中间件，简单
  问题：连接分布可能不均，某个 OAP 可能被过多 Agent 选中

方案B：服务端负载均衡（L4/L7 负载均衡器）
  Agent -> Nginx/LB -> OAP 集群

  ┌────────┐     ┌─────────┐     ┌──────────┐
  │ Agent  │────>│  Nginx  │────>│ OAP 集群  │
  └────────┘     │ (L4 LB) │     │ (多个节点)│
                 └─────────┘     └──────────┘

  特点：Nginx 按连接数/轮询分发，连接分布均匀
  注意：gRPC 基于 HTTP/2，LB 要支持 HTTP/2（Nginx 1.13.10+）
```

**大规模建议**：
- Agent 数量 > 几百时，用方案 B（LB 分发）
- 同时把 OAP 拆成 Receiver + Aggregator 角色（见第 10 章 3.1），Receiver 无状态可任意扩展

---

**② Kafka：连接 Kafka 的 client 数量有限制吗？**

**有限制！这是 Kafka 模式容易被忽略的坑。**

Kafka 模式下，每个 OAP 实例是一个 Kafka 消费者。Kafka 的核心限制是：**一个 Topic 的 Partition 数量 = 该消费组内最大并行消费者数量**。

```
Kafka 的硬限制：
  Topic sw-trace 有 3 个 Partition
  ↓
  消费组（OAP 集群）内最多只能有 3 个消费者同时消费！
  ↓
  如果你部署了 5 个 OAP 节点：
    ├── OAP-1、OAP-2、OAP-3：各自消费一个 Partition ✅
    └── OAP-4、OAP-5：空闲！拿不到 Partition，白白部署 ⚠️

  解决：Partition 数 >= OAP 节点数
  要部署 5 个 OAP，sw-trace 至少要有 5 个 Partition
```

**还有连接数限制：**

```
Kafka Broker 的连接数上限（由 Broker 配置决定）：
  ├── max.connections：单个 Broker 的总连接数上限（默认无上限，但受 fd 限制）
  ├── max.connections.per.ip：单个 IP 的连接数上限（默认 0=不限）
  └── 默认每个 OAP 消费者会建多条连接（消费 + 心跳 + 元数据）

  实际经验：
    - 几十个 OAP 节点 + Kafka 集群：没问题
    - 上百个 OAP 节点：要调大 Broker 的 fd 和 max.connections
    - Kafka 模式下 Agent 不直连 OAP，所以 Agent 数量再多也不直接压 OAP
      （Agent 写 Kafka，OAP 消费 Kafka，压力被 Kafka 集群分摊）
```

**关键结论**：Kafka 模式下，**Agent 数量不再直接压 OAP，而是压 Kafka 集群**。Kafka 集群可水平扩展，所以 Agent 数量上限远高于 gRPC 直连。但 OAP 节点数不能超过 Partition 数。

---

**③ HTTP：有连接数和并发限制吗？**

**有，而且 HTTP 的连接管理最差**（短连接，频繁建连开销大）。

```
HTTP 短连接的问题：
  Agent 每次上报都要建立 TCP 连接 -> 三次握手 -> 发数据 -> 四次挥手
  ↓
  开销：
    ├── 每次上报都要建连，延迟高（gRPC 长连接没有这个开销）
    ├── TIME_WAIT 状态堆积：大量短连接会耗尽端口（默认 65535 - 1024）
    ├── OAP 端连接 accept 队列（somaxconn）可能被打满
    └── 序列化用 JSON，比 Protobuf 大 3-5 倍，带宽浪费

  限制：
    ├── OAP 端 Tomcat/Netty 线程池大小限制并发请求数
    ├── Linux 默认 fd 上限（ulimit -n 通常是 1024，生产要调大）
    └── 短连接端口耗尽（特别是单机大量 Agent 的场景）
```

**HTTP 模式适合什么场景？**
- Agent 数量少（几十个以内）
- 网络环境限制（防火墙只放行 80/443，封了 gRPC 的 11800）
- 跨语言 SDK（非 Java，没有 gRPC 依赖时）

**HTTP 怎么负载均衡？**
- 直接用 Nginx/HAProxy 做 L7 负载均衡（HTTP 天然支持，比 gRPC 简单）
- 或者用 K8s Service（自动负载均衡）

---

**三种协议在大规模部署的对比总结**

| 维度 | gRPC 直连 | Kafka | HTTP |
|------|----------|-------|------|
| **Agent 数量上限** | 几百（受 OAP 连接数限制） | 几千+（受 Kafka 集群容量限制） | 几十（受端口/fd 限制） |
| **OAP 连接压力** | 大（每 Agent 一条长连接） | 小（OAP 只连 Kafka，不连 Agent） | 中（短连接，但频繁建连） |
| **负载均衡方式** | 客户端轮询 / L4 LB | Kafka 分区自动分摊 | L7 LB（Nginx） |
| **水平扩展瓶颈** | OAP 节点的 fd 和连接数 | OAP 节点数 ≤ Partition 数 | OAP 端口数和 fd |
| **延迟** | 最低（长连接） | 较高（多一跳 Kafka） | 中（建连开销） |
| **适用规模** | 中小规模（< 500 Agent） | 大规模（500+ Agent） | 小规模或受限环境 |

> 💡 **选型建议**：
> - **Agent < 500**：gRPC 直连 + 客户端负载均衡（简单够用）
> - **Agent 500-2000**：gRPC 直连 + Nginx L4 负载均衡 + OAP 拆 Receiver/Aggregator
> - **Agent > 2000 或要求高可靠**：Kafka 桥接（压力分摊到 Kafka 集群，OAP 按需消费）
> - **网络受限/跨语言**：HTTP（但要注意端口和 fd 调优）

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