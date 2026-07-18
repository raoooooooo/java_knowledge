# 05 - SkyWalking 核心概念与数据模型

## 核心概念

### 1. 三层服务模型（Service Hierarchy）

SkyWalking 使用 **服务 → 服务实例 → 端点** 三层模型来组织被监控的应用：

```
┌─────────────────────────────────────────────────────────────────┐
│  skywalking-user-service（Service = 服务）                       │
│  ├── user-service-instance-1（ServiceInstance = 服务实例）        │
│  │     ├── GET /user/{id}（Endpoint = 端点）                    │
│  │     ├── POST /user/create（Endpoint = 端点）                 │
│  │     └── PUT /user/update（Endpoint = 端点）                  │
│  │                                                              │
│  └── user-service-instance-2（ServiceInstance = 服务实例）        │
│        ├── GET /user/{id}（Endpoint = 端点）                    │
│        ├── POST /user/create（Endpoint = 端点）                 │
│        └── PUT /user/update（Endpoint = 端点）                  │
│                                                                  │
│  skywalking-order-service（Service = 服务）                      │
│  └── order-service-instance-1（ServiceInstance = 服务实例）       │
│        ├── POST /order/create（Endpoint = 端点）                │
│        └── GET /order/{id}（Endpoint = 端点）                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.1 Service（服务）

- **定义**：提供相同功能的一组服务实例的集合，通常对应一个微服务或一个应用
- **标识**：`agent.service_name` 配置的值
- **作用**：服务级别的指标聚合（Apdex、Cpm、RT、SLA、Throughput）
- **源码对应**：`oap-server/server-core/.../source/Service.java`

```java
// 源码：Service.java - 服务级别的 Source 定义
@ScopeDeclaration(id = DefaultScopeDefine.SERVICE, name = "Service", catalog = DefaultScopeDefine.SERVICE_CATALOG)
public class Service extends Source {
    @ScopeDefaultColumn.VirtualColumnDefinition(fieldName = "entityId", type = String.class)
    private String entityId;
    private String name;
    // 服务级别的指标字段
    private int apdex;
    private int cpm;
    private int throughput;
    private long avgResponseTime;
    private int errorRate;
    // ...
}
```

#### 1.2 ServiceInstance（服务实例）

- **定义**：服务的一个具体运行实例，通常对应一个 JVM 进程或一个 Pod
- **标识**：`agent.service_name` + `IP:Port` 或 Pod Name
- **作用**：实例级别的指标（JVM 内存、GC、线程、CPU 使用率）
- **源码对应**：`oap-server/server-core/.../source/ServiceInstance.java`

```java
// 源码：ServiceInstance.java - 实例级别的 Source 定义
@ScopeDeclaration(id = DefaultScopeDefine.SERVICE_INSTANCE, name = "ServiceInstance")
public class ServiceInstance extends Source {
    private String entityId;
    private String name;
    private int serviceId;
    // 实例级别的属性
    private String serviceName;
    private String endpointName;
}
```

#### 1.3 Endpoint（端点）

- **定义**：服务暴露的一个具体操作接口，通常是 HTTP URI 或 RPC 方法签名
- **标识**：HTTP Method + URI Path（如 `GET:/user/{id}`），或 RPC 方法全限定名
- **作用**：端点级别的指标（QPS、延迟、错误率、状态码分布）
- **源码对应**：`oap-server/server-core/.../source/Endpoint.java`

```java
// 源码：Endpoint.java - 端点级别的 Source 定义
@ScopeDeclaration(id = DefaultScopeDefine.ENDPOINT, name = "Endpoint")
public class Endpoint extends Source {
    private String entityId;
    private String name;
    private int serviceId;
    private String serviceName;
}
```

### 2. Trace 数据模型

SkyWalking 的 Trace 数据模型与 Google Dapper 论文的设计有相似之处，但也有自己的特色。核心区别在于引入了 **Segment** 概念。

#### 2.1 Span（操作单元）

Span 是追踪数据的最小单位，代表一次具体的操作。SkyWalking 定义了三种 Span 类型：

| Span 类型 | 含义 | 典型场景 | 源码枚举 |
|-----------|------|---------|---------|
| **Entry** | 入口 Span，服务接收外部请求 | HTTP 请求处理、gRPC 服务端接收 | `SpanType.Entry` |
| **Exit** | 出口 Span，服务发起外部调用 | HTTP 客户端调用、MySQL 查询、Redis 操作 | `SpanType.Exit` |
| **Local** | 本地 Span，服务内部的本地方法调用 | 本地方法执行、异步任务处理 | `SpanType.Local` |

**Span 包含的关键信息**：

```
Span {
    spanId: 0,                    // 在 Segment 内的唯一编号
    parentSpanId: -1,             // 父 Span 编号（-1 表示根 Span）
    startTime: 1690000000000,     // 开始时间（毫秒）
    endTime: 1690000000015,       // 结束时间（毫秒）
    operationName: "GET:/user/123",  // 操作名
    spanType: Entry,              // Span 类型
    spanLayer: Http,              // 层级（Http/RPC/Database/Cache/MQ）
    componentId: 14,              // 组件 ID（14=SpringMVC, 2=MySQL, 7=Redis）
    peer: "192.168.1.1:3306",    // 对端地址
    isError: false,               // 是否出错
    tags: [                       // 自定义标签（Key-Value）
        {key: "http.method", value: "GET"},
        {key: "http.status_code", value: "200"},
        {key: "db.statement", value: "SELECT * FROM user WHERE id=?"}
    ],
    logs: [                       // Span 日志（时间点事件）
        {time: 1690000000005, data: [{key: "event", value: "start"}]},
        {time: 1690000000012, data: [{key: "event", value: "end"}]}
    ],
    refs: [                       // 跨 Segment 引用
        {traceId: "xxx", parentSegmentId: "yyy", parentSpanId: 0, ...}
    ]
}
```

#### 2.2 Segment（段）

**Segment 是 SkyWalking 独有的概念**，代表一个服务实例在一次请求中产生的所有 Span 的集合。

```
一个 Segment 的特征：
1. 属于一个特定的 Trace（通过 traceId 关联）
2. 属于一个特定的服务实例（在一个 JVM 进程中产生）
3. 包含一个或多个 Span（至少有一个 Entry Span）
4. 有唯一的 segmentId（UUID 格式）
5. 在 Agent 端组装完成后，作为一个整体上报给 OAP
```

**为什么需要 Segment？**

- **减少上报次数**：多个 Span 打包成一个 Segment 一次性上报，减少网络开销
- **保持服务内上下文**：同一个服务内的 Span 天然具有时序关系，在 Segment 内可以完整保留
- **简化 OAP 处理**：OAP 收到的数据以 Segment 为单位，不需要重新拼接同一服务内的 Span 关系

**源码对应**：`apm-protocol/apm-network/src/main/proto` 中的 `SegmentObject` 定义

#### 2.3 Trace（追踪链）

**Trace 是跨服务的完整调用链**，由多个 Segment 通过引用关系串联而成。

```
Trace 的组装过程：

1. Agent 端：每个服务实例生成自己的 Segment，包含该服务内的所有 Span
2. 上报时：Segment 携带 refs 引用信息（traceId + parentSegmentId + parentSpanId）
3. OAP 端：根据 traceId 将所有 Segment 关联起来，根据 refs 确定父子关系
4. 查询时：UI 根据 traceId 查询所有 Segment，组装成完整的调用树
```

**Trace 的关键标识**：

| 标识 | 说明 | 示例 |
|------|------|------|
| traceId | 全局唯一，标识一次完整的请求 | `e8a2f3b4c5d6e7f8a9b0c1d2e3f4a5b6.1.1690000000000` |
| segmentId | 每个服务实例的 Segment 唯一标识 | `a1b2c3d4e5f6...`（UUID） |
| spanId | Segment 内 Span 的编号 | `0`（根 Span），`1`，`2`... |
| parentSegmentId | 父 Segment 的 ID | 用于跨服务关联 |

### 3. Layer（分层）与 Component（组件）

这两个概念常被混淆，先明确它们的区别：

> **Layer 是"你是什么类型的服务"**——跨语言、跨技术栈的分类，不关心具体实现。
> **Component 是"你用了什么具体的库/框架"**——与语言生态绑定，Java Agent 拦截到的是 SpringMVC，Python Agent 拦截到的是 Django。

比如：MySQL 和 PostgreSQL 的 Component 不同，但 Layer 都是 `DB`；SpringMVC 和 Django 的 Component 不同，但 Layer 都是 `GENERAL`。

#### 3.1 Layer（层级分类）

Layer 是**跨语言、跨技术栈**的服务分类，每种 Layer 有不同的指标和监控重点：

| Layer | 含义 | 指标特点 | 跨语言示例 |
|-------|------|---------|-----------|
| **GENERAL** | 通用服务（默认） | 标准 HTTP/RPC 指标 | Java(SpringMVC)、Python(Django)、Go(Gin)、Node.js(Express) |
| **DB** | 数据库 | 连接数、慢查询、QPS | MySQL、PostgreSQL、MongoDB、TiDB、Oracle |
| **MQ** | 消息队列 | 生产/消费速率、积压 | Kafka、RocketMQ、RabbitMQ、Pulsar |
| **CACHE** | 缓存 | 命中率、内存使用 | Redis、Memcached、Caffeine |
| **VIRTUAL_MQ** | 虚拟 MQ（无 Agent 的 MQ） | 通过 Exit Span 推断 | 调用方有 Agent，但 MQ 本身没有安装 Agent |
| **VIRTUAL_DATABASE** | 虚拟数据库（无 Agent 的 DB） | 通过 Exit Span 推断 | 调用方有 Agent，但 DB 本身没有安装 Agent |
| **FAAS** | 函数即服务 | 冷启动、调用次数 | AWS Lambda、阿里云函数计算 |
| **GATEWAY** | API 网关 | 路由转发、限流 | Spring Cloud Gateway、Kong、APISIX、Nginx |

**Layer 的跨语言本质**：不管你的服务是 Java 写的还是 Go 写的，只要它是一个数据库，它在 SkyWalking 中的 Layer 就是 `DB`。这是 SkyWalking 能在**多语言混合架构**中统一展示拓扑图的基础——拓扑图中用同一个数据库图标展示 MySQL 和 PostgreSQL 两个不同节点。

#### 3.2 Component（组件定义）

Component 是**具体框架/库的标识**，与语言生态绑定。每个被 Agent 拦截的库都有一个唯一的 `componentId`，Agent 在创建 Span 时自动设置。

**Component 是语言相关的**——不同语言的 Agent 有各自独立的 Component 定义：

| 语言 | HTTP Server 组件 | RPC 组件 | DB 组件 |
|------|-----------------|---------|---------|
| **Java** | SpringMVC(14)、Tomcat(38)、Undertow | Dubbo(3)、gRPC(23)、Feign | MySQL(2)、PgSQL(22)、MongoDB |
| **Python** | Django、Flask、FastAPI | gRPC-Python | mysqlclient、psycopg2 |
| **Go** | Gin、Echo、net/http | gRPC-Go、Kitex | go-sql-driver、pgx |
| **Node.js** | Express、Koa、Fastify | gRPC-Node | mysql2、pg |

**Java Agent 中部分 Component ID 示例**（最成熟，内置 100+ 组件）：

| Component ID | 组件 | 类型 | 所属 Layer |
|-------------|------|------|-----------|
| 14 | SpringMVC | HTTP Server | GENERAL |
| 38 | Tomcat | HTTP Server | GENERAL |
| 2 | MySQL | 数据库 | DB |
| 22 | PostgreSQL | 数据库 | DB |
| 7 | Redis | 缓存 | CACHE |
| 30 | Jedis | 缓存客户端 | CACHE |
| 3 | Dubbo | RPC | GENERAL |
| 23 | gRPC | RPC | GENERAL |
| 41 | Kafka | 消息队列 | MQ |
| 21 | Spring Cloud Gateway | 网关 | GATEWAY |

> ⚠️ **Component 不等于 Layer**：一个 Component 属于哪个 Layer 是固定的（MySQL 永远是 DB 层），但同一个 Layer 下可以有多种 Component（DB 层下有 MySQL、PostgreSQL、MongoDB 等）。Layer 决定了 SkyWalking 用什么样的指标模板来监控这个服务，Component 决定了 Span 上标记的具体技术栈名称。

### 4. Tags、Logs 与 Events

#### 4.1 Tags（标签）

Span 上的 Key-Value 元数据，用于描述 Span 的上下文信息：

```java
// 源码：SpanTags.java - 预留的 Span 标签 Key
public class SpanTags {
    public static final String HTTP_RESPONSE_STATUS_CODE = "http.status_code";
    public static final String RPC_RESPONSE_STATUS_CODE = "rpc.status_code";
    public static final String DB_STATEMENT = "db.statement";
    public static final String DB_TYPE = "db.type";
    public static final String CACHE_TYPE = "cache.type";
    public static final String CACHE_OP = "cache.op";
    public static final String CACHE_CMD = "cache.cmd";
    public static final String CACHE_KEY = "cache.key";
    public static final String MQ_QUEUE = "mq.queue";
    public static final String MQ_TOPIC = "mq.topic";
    public static final String TRANSMISSION_LATENCY = "transmission.latency";
    public static final String LOGIC_ENDPOINT = "x-le";  // 逻辑端点
}
```

#### 4.2 Logs（Span 日志）

附着在 Span 上的时间点事件，记录 Span 生命周期中的关键节点：

```json
{
  "logs": [
    {
      "time": 1690000000005,
      "data": [
        {"key": "event", "value": "start_processing"},
        {"key": "queue_size", "value": "100"}
      ]
    }
  ]
}
```

#### 4.3 AttachedEvents（附加事件）

附着在 Span 上的特殊标记（源码中独立于 Logs），用于标记异常：

- `error` 事件：标记 Span 发生了错误
- 与 `isError` 字段配合使用

### 5. OperationName（操作名）与端点发现

#### 5.1 OperationName 生成规则

| 协议 | 生成规则 | 示例 |
|------|---------|------|
| HTTP | `{HTTP_METHOD}:{URI_PATTERN}` | `GET:/user/{id}` |
| RPC（Dubbo） | `{InterfaceName}.{methodName}` | `com.example.UserService.findById` |
| Database | `{DB_TYPE}/{operation}` | `MySQL/select` |
| Cache | `{CACHE_TYPE}/{operation}` | `Redis/get` |

#### 5.2 端点发现机制

```
Agent 端处理流程：
1. 拦截请求 → 提取 OperationName
2. 本地缓存 OperationName（避免重复注册）
3. 首次出现的 OperationName → 注册到 OAP
4. OAP 端创建 Endpoint 元数据 → 持久化存储
```

---

## 常见面试题

### Q1: 为什么 SkyWalking 要引入 Segment 概念？直接上报 Span 不行吗？

引入 Segment 的设计带来了几个关键优势：

1. **减少网络开销**：一个服务内的多个 Span 打包成一个 Segment 一次性上报，而不是每个 Span 单独上报。在高并发场景下，这能显著减少网络请求次数。

2. **保持服务的本地上下文**：同一服务内的 Span 天然具有时序关系，在 Segment 内可以完整保留。OAP 不需要重新拼接同一服务内的 Span 关系，只需要处理跨服务的 Segment 关联。

3. **简化 OAP 的处理逻辑**：OAP 以 Segment 为单位进行分析，而不是以 Span 为单位。一个 Segment 代表一个服务实例的一次完整处理，直接对应 Endpoint 级别的指标聚合。

### Q2: service、serviceInstance、endpoint 三层模型有什么好处？

**类比**：像公司组织架构

- **Service** = 部门（如"研发部"），代表一组功能
- **ServiceInstance** = 部门员工（如"张三"），具体的执行者
- **Endpoint** = 员工负责的具体工作项（如"需求评审"）

**实际价值**：

1. **多维度聚合**：可以在不同粒度上聚合指标
   - Service 级别：整体服务质量（Apdex、SLA）
   - Instance 级别：单个实例的健康状况（JVM 指标）
   - Endpoint 级别：具体接口的性能（QPS、延迟）

2. **弹性伸缩友好**：K8s 环境下实例频繁启停，Service 级别的指标保持稳定，Instance 级别的指标反映扩容/缩容效果

3. **故障定位**：从 Service 级别发现异常 → 下钻到 Instance 级别找到具体实例 → 下钻到 Endpoint 级别找到具体接口

### Q3: Entry、Exit、Local 三种 Span 类型分别代表什么？

| 类型 | 方向 | 含义 | 示例 |
|------|------|------|------|
| Entry | 入站 | 服务接收外部请求 | Tomcat 接收 HTTP 请求、Dubbo Provider 接收 RPC 调用 |
| Exit | 出站 | 服务发起外部调用 | HttpClient 调用下游、JDBC 查询 MySQL、Jedis 操作 Redis |
| Local | 内部 | 服务内部的本地方法调用 | 同步方法执行、异步任务、本地缓存操作 |

**为什么区分这三种类型？**

- **Entry** 和 **Exit** 成对出现，用于构建服务拓扑图（A 的 Exit → B 的 Entry → 服务 A 依赖 B）
- **Local** 独立于服务拓扑，只影响当前服务的内部调用链展示
- 告警规则可以基于不同 Span 类型配置（如：只告警 Entry Span 的错误）

### Q4: 如何理解 Layer 分层？Layer 和 Component 有什么区别？

Layer 是 SkyWalking 对服务的**技术栈分类**，**跨语言、跨技术栈**，不关心具体实现。Component 是**具体框架/库的标识**，与语言生态绑定。

**简单记法**：Layer 回答"你是什么？"（数据库、缓存、MQ...），Component 回答"你是谁？"（MySQL、Redis、Kafka...）。MySQL 和 PostgreSQL 的 Component 不同，但 Layer 都是 `DB`。

**为什么需要 Layer？**

1. **跨语言统一拓扑**：Java 服务调 MySQL、Go 服务调 PostgreSQL——在拓扑图中它们都是"数据库"图标，Layer 提供统一视图
2. **不同 Layer 的指标不同**：DB 需要监控连接数和慢查询，MQ 需要监控生产和消费速率，通用服务需要监控 HTTP 状态码
3. **拓扑图展示更清晰**：UI 上可以用不同图标区分不同类型的服务（数据库图标、缓存图标、消息队列图标）
4. **虚拟服务推断**：当数据库没有被 Agent 探针覆盖时，SkyWalking 可以通过分析 Exit Span 的 `peer` 地址和 `componentId`，推断出虚拟的数据库服务节点（Layer 自动设为 `VIRTUAL_DATABASE`）

---

## 延伸阅读

- SkyWalking Scope 定义源码：`oap-server/server-core/src/main/java/org/apache/skywalking/oap/server/core/source/`
- SpanTags 常量定义：`oap-server/server-analyzer/agent-analyzer/.../trace/parser/SpanTags.java`