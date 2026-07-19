# 10 - SkyWalking OAP 后端架构

## 核心概念

### 1. OAP 是什么？

**OAP（Observability Analysis Platform，可观测性分析平台）** 是 SkyWalking 的后端核心，负责接收、分析、聚合、存储和查询所有可观测性数据。

```
OAP 的职责：
┌──────────────────────────────────────────────────────────────┐
│                         OAP Server                           │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐ │
│  │ 接收数据  │→│ 解析分析  │→│ 指标聚合  │→│ 存储持久化  │ │
│  │ (Receiver)│  │(Analyzer)│  │(OAL引擎) │  │(Storage)   │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────────┘ │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐  │
│  │ 查询数据  │  │ 告警判定  │  │ 集群管理/健康检查/遥测    │  │
│  │ (Query)  │  │ (Alarm)  │  │ (Cluster/Health/Telemetry)│  │
│  └──────────┘  └──────────┘  └──────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 2. 模块化架构设计

SkyWalking OAP 采用**模块化（Module）架构**，基于 Java SPI 机制实现插件化。

#### 2.1 ModuleDefine 与 ModuleProvider

```
┌──────────────────────────────────────────────────────────────┐
│                    模块化架构设计                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ModuleDefine（模块定义）                                      │
│  ├── name()：模块名称                                         │
│  ├── services()：模块提供的服务接口                             │
│  └── requiredModules()：依赖的其他模块                         │
│                                                              │
│  ModuleProvider（模块实现）                                     │
│  ├── name()：实现名称                                         │
│  ├── module()：所属模块                                       │
│  ├── prepare()：准备阶段（初始化配置）                          │
│  ├── start()：启动阶段（启动服务）                              │
│  └── notifyAfterCompleted()：所有模块启动完成后的回调            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**源码示例**：
```java
// 核心模块定义
public class CoreModule extends ModuleDefine {
    public static final String NAME = "core";

    public CoreModule() {
        super(NAME);
    }

    @Override
    public Class[] services() {
        // 核心模块提供的服务
        return new Class[] {
            RemoteClientManager.class,
            INamingControl.class,
            IComponentLibraryCatalogService.class,
            // ...
        };
    }
}

// 核心模块实现
public class CoreModuleProvider extends ModuleProvider {
    @Override
    public String name() {
        return "default";
    }

    @Override
    public Class<? extends ModuleDefine> module() {
        return CoreModule.class;
    }

    @Override
    public void prepare() {
        // 准备阶段：注册服务实现
        this.registerServiceImplementation(RemoteClientManager.class, ...);
    }

    @Override
    public void start() {
        // 启动阶段：启动后台线程、定时任务等
    }

    @Override
    public String[] requiredModules() {
        return new String[0]; // 核心模块不依赖其他模块
    }
}
```

#### 2.2 OAP 核心模块全景

```
oap-server/
├── server-core/              # 核心模块（数据模型、Source、Scope定义）
├── server-library/           # 共享库（模块系统、工具类、BanyanDB客户端）
├── server-receiver-plugin/   # 数据接收插件
│   ├── skywalking-trace-receiver-plugin    # Trace 接收
│   ├── skywalking-meter-receiver-plugin    # Meter 接收
│   ├── skywalking-log-receiver-plugin      # Log 接收
│   ├── skywalking-event-receiver-plugin    # Event 接收
│   ├── skywalking-browser-receiver-plugin  # 浏览器数据接收
│   ├── skywalking-jvm-receiver-plugin      # JVM 指标接收
│   ├── skywalking-profile-receiver-plugin  # Profiling 数据接收
│   ├── skywalking-management-receiver-plugin  # 管理接口
│   ├── otel-receiver-plugin                # OpenTelemetry OTLP 接收
│   ├── zipkin-receiver-plugin              # Zipkin 兼容接收
│   ├── envoy-metrics-receiver-plugin       # Envoy ALS 接收
│   ├── skywalking-ebpf-receiver-plugin     # eBPF 数据接收
│   └── ...                                 # 更多接收器
├── server-storage-plugin/    # 存储插件
│   ├── storage-banyandb-plugin             # BanyanDB 存储
│   ├── storage-elasticsearch-plugin        # Elasticsearch 存储
│   └── storage-jdbc-hikaricp-plugin        # JDBC 存储（MySQL/H2/PostgreSQL）
├── server-query-plugin/      # 查询插件（GraphQL）
├── server-alarm-plugin/      # 告警插件
├── server-cluster-plugin/    # 集群协调插件
│   ├── zookeeper                           # ZooKeeper
│   ├── kubernetes                          # K8s API
│   ├── nacos                               # Nacos
│   └── consul                              # Consul
├── server-configuration/     # 动态配置
│   ├── apollo                              # Apollo
│   ├── nacos                               # Nacos
│   ├── zookeeper                           # ZooKeeper
│   └── consul                              # Consul
├── server-fetcher-plugin/    # 数据拉取插件
│   ├── prometheus-fetcher-plugin           # Prometheus 拉取
│   └── kafka-fetcher-plugin                # Kafka 消费
├── analyzer/                 # 分析引擎
│   ├── agent-analyzer                      # Agent 数据分析
│   ├── meter-analyzer                      # Meter 数据分析
│   ├── log-analyzer                        # 日志分析
│   ├── event-analyzer                      # 事件分析
│   ├── gen-ai-analyzer                     # AI 数据分析
│   └── hierarchy                           # 层级关系分析
├── oal-grammar/              # OAL 语法定义
├── oal-rt/                   # OAL 运行时
├── mqe-grammar/              # MQE 查询语法
├── mqe-rt/                   # MQE 运行时
├── server-telemetry/         # OAP 自监控
├── server-testing/           # 测试工具
├── server-tools/             # 独立工具（Profile Exporter 等）
├── ai-pipeline/              # AI 管道
├── exporter/                 # 数据导出
└── server-starter/           # 启动入口
```

### 3. 集群部署架构

#### 3.1 集群角色

SkyWalking v9+ 支持三种集群角色：

| 角色 | 职责 | 使用场景 |
|------|------|---------|
| **Mixed**（默认） | 承担所有职责（Receiver + Aggregator） | 小规模部署、单机部署 |
| **Receiver** | 只负责接收数据，转发给 Aggregator | 大规模部署中的前置层 |
| **Aggregator** | 只负责聚合数据，写入存储 | 大规模部署中的计算层 |

#### 3.2 集群部署架构

```
                         ┌──────────────┐
                         │   UI (LB)    │
                         └──────┬───────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
     ┌────────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
     │ OAP Receiver  │  │OAP Receiver  │  │OAP Receiver  │
     │  (接收数据)    │  │  (接收数据)    │  │  (接收数据)    │
     └────────┬──────┘  └───────┬──────┘  └───────┬──────┘
              │                 │                 │
              │  gRPC 转发      │  gRPC 转发      │  gRPC 转发
              │                 │                 │
     ┌────────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
     │OAP Aggregator  │  │OAP Aggregator│  │OAP Aggregator│
     │  (聚合计算)     │  │  (聚合计算)    │  │  (聚合计算)    │
     └────────┬──────┘  └───────┬──────┘  └───────┬──────┘
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                       ┌────────▼──────┐
                       │  Storage 集群  │
                       │ (BanyanDB/ES) │
                       └───────────────┘
```

#### 3.3 水平扩展策略

**Receiver 层扩展**：
- 通过负载均衡（如 Nginx/K8s Service）将 Agent 上报请求分发到多个 Receiver
- 每个 Receiver 独立处理，不共享状态

**Aggregator 层扩展**：
- 通过集群协调（ZooKeeper/Nacos/K8s）分配数据分片
- 每个 Aggregator 负责一部分数据（按 Service 哈希分片）
- 数据分片确保同一个 Service 的指标聚合在同一个 Aggregator 上

### 4. 启动流程

```
OAP 启动流程（BootstrapFlow）：
  │
  ├── 1. 加载 application.yml 配置
  │     ├── 解析 cluster 配置（集群模式）
  │     ├── 解析 core 配置（存储选择）
  │     └── 解析各模块配置
  │
  ├── 2. 初始化 ModuleManager
  │     ├── 扫描 SPI 注册的 ModuleDefine
  │     ├── 扫描 SPI 注册的 ModuleProvider
  │     └── 根据配置选择 Provider
  │
  ├── 3. 模块准备（prepare 阶段）
  │     ├── 按依赖顺序调用每个 ModuleProvider.prepare()
  │     ├── 注册服务实现
  │     └── 验证依赖关系
  │
  ├── 4. 模块启动（start 阶段）
  │     ├── 按依赖顺序调用每个 ModuleProvider.start()
  │     ├── 启动 gRPC 服务（端口 11800）
  │     ├── 启动 HTTP 服务（端口 12800，GraphQL）
  │     └── 启动定时任务（指标聚合、TTL 清理等）
  │
  └── 5. 通知完成（notifyAfterCompleted）
        └── 所有模块启动完成后，通知各模块
```

### 5. 核心通信机制

#### 5.1 gRPC 服务（Agent 通信）

- **端口**：11800（默认）
- **协议**：gRPC（基于 Protobuf）
- **用途**：Agent 上报 Trace/Metrics/Logs/Profiling 数据
- **服务定义**：`apm-protocol/apm-network/src/main/proto/`

#### 5.2 HTTP 服务（UI 与 API 通信）

- **端口**：12800（默认）
- **协议**：HTTP + GraphQL
- **用途**：UI 查询数据、管理 API、健康检查
- **查询实现**：`server-query-plugin/`

#### 5.3 集群通信（OAP 之间）

- **协议**：gRPC
- **用途**：Receiver → Aggregator 数据转发、集群协调
- **实现**：`server-cluster-plugin/`

---

### 6. 灵魂拷问：OAP 为什么不内置 Kafka？数据怎么缓存？ ⭐⭐⭐

这是一个非常常见的问题：**既然数据上报有突发流量，为什么不引入 Kafka 做缓冲？没有 Kafka，OAP 怎么缓存数据？**

答案是：**OAP 不需要 Kafka 也能做数据缓存，因为它内置了高性能内存队列 DataCarrier。Kafka 是可选的，不是必需的。**

---

#### 6.1 先厘清："数据缓存"的两种场景

很多人把"数据缓存"等同于"上 Kafka"，这是误解。数据缓存其实有两个完全不同的场景：

| 场景 | 缓存什么 | 用什么 | 为什么要缓存 |
|------|---------|--------|------------|
| **A. OAP 内部瞬时缓冲** | 收到数据后到分析前的极短时间 | **DataCarrier（内存队列）** | 解耦接收线程和分析线程，应对微秒级突发 |
| **B. Agent -> OAP 之间的持久缓冲** | Agent 上报但 OAP 还没处理的数据 | **Kafka（可选）** | OAP 宕机时数据不丢，应对分钟级故障 |

**关键认知**：
- 场景A 是**必须**的，每个 OAP 节点都要做，DataCarrier 解决
- 场景B 是**可选**的，只有对可靠性要求极高时才上 Kafka

---

#### 6.2 OAP 内部的缓存机制：DataCarrier

OAP 收到数据后，不会立即分析，而是先丢进 **DataCarrier**（基于 Disruptor RingBuffer 的内存队列）：

```
Agent 上报数据（gRPC）
        │
        ▼
┌──────────────────────────────────────────────┐
│  Receiver 接收线程                              │
│  ├── 反序列化                                  │
│  └── 丢进 DataCarrier（内存队列）               │
│      ┌──────────────────────────────────┐    │
│      │  DataCarrier（Disruptor RingBuffer）│    │
│      │  ├── TraceBuffer（调用链缓冲）     │    │  ← 这就是"数据缓存"！
│      │  ├── MetricsBuffer（指标缓冲）     │    │
│      │  ├── LogBuffer（日志缓冲）         │    │
│      │  └── EventBuffer（事件缓冲）       │    │
│      └──────────────────────────────────┘    │
└──────────────────────────────────────────────┘
        │
        ▼（批量消费）
┌──────────────────────────────────────────────┐
│  Analyzer 分析线程                              │
│  ├── TraceAnalyzer -> OAL 聚合               │
│  ├── MeterAnalyzer -> MAL 聚合               │
│  └── LogAnalyzer -> LAL 分析                 │
└──────────────────────────────────────────────┘
```

**DataCarrier 为什么能替代 Kafka 做瞬时缓冲？**

| 对比维度 | DataCarrier | Kafka |
|---------|------------|-------|
| 延迟 | 微秒级（内存） | 毫秒级（磁盘+网络） |
| 吞吐 | 单机百万级 TPS | 集群百万级 TPS |
| 持久化 | ❌ 进程挂了数据丢 | ✅ 落盘持久化 |
| 运维成本 | ✅ 零（内置） | ❌ 高（要部署集群） |
| 适用场景 | OAP 内部瞬时解耦 | Agent -> OAP 持久缓冲 |

**结论**：对于 OAP 内部的"接收 -> 分析"解耦，DataCarrier 比 Kafka 更合适（更快、更轻）。详细原理见第 11 章 5.1 节。

---

#### 6.3 那 Kafka 在什么时候才需要？

只有当你有**强可靠性**需求时，才在 Agent -> OAP 之间引入 Kafka：

```
场景1：OAP 会频繁重启/升级
  没有Kafka：OAP重启期间，Agent上报失败，数据直接丢
  有Kafka：Agent写到Kafka，OAP重启后从Kafka消费，数据不丢

场景2：OAP 处理不过来（流量洪峰）
  没有Kafka：DataCarrier满 -> 触发背压 -> 丢弃数据（IF_POSSIBLE策略）
  有Kafka：Kafka削峰填谷，OAP按自己节奏消费，不丢数据

场景3：Agent 和 OAP 网络不稳定
  没有Kafka：网络抖动时Agent上报失败
  有Kafka：Agent写本地Kafka更可靠，OAP稳定时再拉
```

**Kafka 模式架构（可选）**：

```
┌──────────┐         ┌─────────┐         ┌──────────┐
│  Agent   │──gRPC──>│  Kafka  │<──消费──│   OAP    │
│（生产者） │         │ (缓冲)  │  (消费组)│ (Fetcher)│
└──────────┘         └─────────┘         └──────────┘

Agent 不直接连 OAP，而是写到 Kafka
OAP 用 kafka-fetcher-plugin 从 Kafka 消费数据
```

---

#### 6.4 SkyWalking 的设计哲学：Kafka 可选，不是必需

```
SkyWalking 的分层设计：

  默认模式（gRPC 直连）：
    Agent --gRPC--> OAP（内置DataCarrier做瞬时缓冲）
    特点：简单、低延迟、零额外依赖
    适用：90% 的中小规模场景

  可选模式（Kafka 桥接）：
    Agent --gRPC--> Kafka --消费--> OAP
    特点：高可靠、削峰填谷、解耦
    适用：对数据零丢失有强要求的场景

  为什么这样设计？
    1. 不强制依赖Kafka -> 部署简单，中小团队开箱即用
    2. 需要时再开启 -> 大厂/高可靠场景按需启用
    3. DataCarrier兜底 -> 即使不开Kafka，OAP内部也有缓冲能力
```

> 💡 **面试一句话**：OAP 不内置 Kafka 是因为它内置了 DataCarrier 内存队列来做瞬时缓冲，满足绝大多数场景的解耦需求。Kafka 是可选的"持久缓冲层"，只有对数据零丢失有强要求时才在 Agent 和 OAP 之间引入。详见第 11 章 Kafka 桥接模式。

---

## 常见面试题

### Q1: SkyWalking OAP 的模块化架构有什么好处？

1. **可插拔**：通过 SPI 机制，可以灵活替换存储、集群协调、动态配置等实现
2. **解耦**：模块之间通过服务接口通信，依赖关系清晰
3. **可测试**：每个模块可以独立测试，Mock 模块可以替换真实模块
4. **可扩展**：新增功能只需添加新模块，不影响现有模块
5. **生命周期管理**：统一的 prepare → start → notifyAfterCompleted 生命周期

### Q2: Mixed / Receiver / Aggregator 三种角色分别适用于什么场景？

| 角色 | 适用场景 | 部署规模 |
|------|---------|---------|
| Mixed | 开发测试、小规模生产（< 100 Agent） | 1-3 个节点 |
| Receiver + Aggregator | 大规模生产（> 100 Agent） | Receiver: 3-5 节点，Aggregator: 3-7 节点 |

**分离 Receiver 和 Aggregator 的好处**：
- Receiver 层无状态，可以任意扩展
- Aggregator 层有状态（数据分片），通过哈希分片扩展
- 接收和计算解耦，避免计算瓶颈影响数据接收

### Q3: OAP 如何保证集群中的数据一致性？

1. **数据分片**：通过哈希分片，每个 Service 的数据始终路由到同一个 Aggregator
2. **集群协调**：通过 ZooKeeper/Nacos/K8s 维护集群拓扑和分片信息
3. **最终一致性**：节点故障时，分片重新分配，新节点接管数据聚合
4. **存储层保证**：聚合后的指标写入存储，由存储层保证数据一致性

### Q4: prepare() 和 start() 两个阶段有什么区别？

| 阶段 | prepare() | start() |
|------|-----------|---------|
| 时机 | 所有模块的 prepare 先执行完 | 所有模块的 prepare 执行完后，再执行 start |
| 职责 | 注册服务实现、初始化配置 | 启动服务（gRPC/HTTP 监听、定时任务等） |
| 依赖 | 只能访问已注册的服务 | 可以访问所有模块的服务 |

**为什么分两个阶段？**
- prepare 阶段解决依赖注入问题（先注册服务，后启动服务）
- start 阶段启动外部监听（gRPC/HTTP），确保所有服务都已注册

### Q5: OAP 为什么不内置 Kafka？没有 Kafka 数据怎么缓存？

**核心答案**：OAP 不需要 Kafka 也能缓存数据，因为它内置了 **DataCarrier** 内存队列。要区分两种"缓存"：

| 场景 | 用什么 | 必需？ |
|------|--------|--------|
| OAP 内部瞬时缓冲（接收->分析解耦） | DataCarrier（内存队列） | ✅ 必需，已内置 |
| Agent -> OAP 持久缓冲（宕机不丢数据） | Kafka | ⚠️ 可选，按需启用 |

**为什么不内置 Kafka？**

1. **DataCarrier 已经够用**：微秒级延迟、单机百万 TPS，满足 OAP 内部解耦需求
2. **Kafka 太重**：要部署集群、运维成本高，不适合开箱即用
3. **定位差异**：DataCarrier 解决"瞬时缓冲"，Kafka 解决"持久缓冲"，职责不同

**什么时候才需要 Kafka？**

- OAP 频繁重启/升级，要求重启期间数据不丢
- 流量洪峰超过 OAP 处理能力，需要削峰填谷
- Agent 和 OAP 网络不稳定，需要可靠中间层

**结论**：默认 gRPC 直连 + DataCarrier 缓冲，覆盖 90% 场景；有强可靠性需求时才开启 Kafka 桥接模式。详见第 11 章 Kafka 桥接模式。

---

## 延伸阅读

- SkyWalking OAP 源码入口：`oap-server/server-starter/src/main/java/org/apache/skywalking/oap/server/starter/OAPServerBootstrap.java`
- 模块系统源码：`oap-server/server-library/library-module/src/main/java/org/apache/skywalking/oap/server/library/module/`