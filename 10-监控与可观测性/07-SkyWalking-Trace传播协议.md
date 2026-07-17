# 07 - SkyWalking Trace 传播协议（sw8）

## 核心概念

### 1. 为什么需要 Trace 传播协议？

在分布式系统中，同一个请求会经过多个服务节点。要让这些节点产生的 Span 能正确关联到同一个 Trace，必须有一个**跨进程传播协议**来传递 Trace 上下文。

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│ Service A │ ───→ │ Service B │ ───→ │ Service C │
└──────────┘       └──────────┘       └──────────┘
     │                   │                   │
     │ 在 HTTP Header     │ 从 HTTP Header     │ 从 HTTP Header
     │ 中注入 sw8 上下文   │ 中提取 sw8 上下文   │ 中提取 sw8 上下文
     │                   │                   │
     ▼                   ▼                   ▼
  Segment-A           Segment-B           Segment-C
  (同一个 traceId: abc123.1.1690000000000)
```

### 2. sw8 协议详解

**sw8**（SkyWalking v8 protocol）是 SkyWalking 自研的跨进程传播协议，从 v1 演进到 v8。

#### 2.1 sw8 Header 格式

```
Header Key: sw8

Header Value: {Base64编码的字符串}

编码格式（用 - 分隔的 8 个字段）：
1-TRACE_ID-3-PARENT_SERVICE-5-PARENT_SERVICE_INSTANCE-7-PARENT_ENDPOINT-9-IP_OF_TARGET
```

**解码后的字段结构**：

| 位置 | 字段 | 含义 | 示例 |
|------|------|------|------|
| 0 | `sample` | 采样标志（0/1） | `1` |
| 1 | `traceId` | 全局唯一 Trace ID | `e8a2f3b4c5d6e7f8a9b0c1d2e3f4a5b6.1.1690000000000` |
| 2 | `parentService` | 父服务名称 | `order-service` |
| 3 | `parentServiceInstance` | 父服务实例标识 | `order-service-instance-1` |
| 4 | `parentEndpoint` | 父端点名称 | `POST:/order/create` |
| 5 | `targetAddress` | 目标地址（对端 IP:Port） | `192.168.1.1:8080` |

**编码过程**：

```
1. 将 8 个字段用 "-" 连接
2. 对连接后的字符串进行 Base64 编码
3. 结果作为 HTTP Header 值
```

#### 2.2 sw8 Header 的三个变体

| Header | 编码前缀 | 用途 |
|--------|---------|------|
| **sw8** | `1-` | 标准跨进程传播（HTTP/Dubbo/gRPC Headers） |
| **sw8-x** | `0-` | 扩展协议，携带额外信息（如 TTL） |
| **sw8-correlation** | (无前缀) | 跨进程传播自定义业务字段 |

#### 2.3 sw8 协议的生成过程

```java
// 伪代码：Agent 在发起下游调用时生成 sw8 上下文

// 1. 创建 Exit Span
Span exitSpan = ContextManager.createExitSpan(
    "/user/findById",      // operationName
    "192.168.1.1:8080",    // peer
    ContextCarrier          // 上下文载体
);

// 2. ContextCarrier 自动填充
ContextCarrier {
    traceId: "e8a2f3b4c5d6e7f8a9b0c1d2e3f4a5b6.1.1690000000000",
    segmentId: "a1b2c3d4...",
    spanId: 1,
    parentService: "order-service",
    parentServiceInstance: "order-service-instance-1",
    parentEndpoint: "POST:/order/create",
    peer: "192.168.1.1:8080"
}

// 3. 编码为 sw8 Header
String sw8Value = encode(ContextCarrier);
// → "MS1lOGEyZjNiNGM1ZDZlN2Y4YTliMGMxZDJlM2Y0YTViNi4xLjE2OTAwMDAwMDAwMDAtM29yZGVyLXNlcnZpY2UtNW9yZGVyLXNlcnZpY2UtaW5zdGFuY2UtMS03UE9TVDovb3JkZXIvY3JlYXRlLTktMTkyLjE2OC4xLjE6ODA4MA=="

// 4. 注入到 HTTP Header
httpRequest.setHeader("sw8", sw8Value);
```

### 3. ContextCarrier vs ContextSnapshot

Sw8 协议有两种上下文传播方式：

| 传播方式 | 数据载体 | 适用场景 | 传播方式 |
|---------|---------|---------|---------|
| **ContextCarrier** | `ContextCarrier` 类 | 跨进程传播（RPC/HTTP/MQ） | 序列化到 Header/Metadata |
| **ContextSnapshot** | `ContextSnapshot` 类 | 跨线程传播（异步任务） | 内存传递（不序列化） |

#### 3.1 ContextCarrier（跨进程传播）

```java
// 场景：HTTP 调用下游服务

// ===== 调用方（Client）=====
// 在 Exit Span 创建时，ContextCarrier 自动填充
ContextCarrier carrier = new ContextCarrier();
Span exitSpan = ContextManager.createExitSpan("/api/order", carrier, "service-b:8080");

// 将 carrier 注入到 HTTP Header
httpRequest.setHeader("sw8", carrier.serialize());

// ===== 被调用方（Server）=====
// 从 HTTP Header 中提取 carrier
String sw8Header = httpRequest.getHeader("sw8");
ContextCarrier carrier = ContextCarrier.deserialize(sw8Header);

// 创建 Entry Span，关联到上游 Trace
Span entrySpan = ContextManager.createEntrySpan("/api/order", carrier);
```

#### 3.2 ContextSnapshot（跨线程传播）

```java
// 场景：异步任务（线程池）

// ===== 主线程 =====
// 捕获当前上下文快照
ContextSnapshot snapshot = ContextManager.capture();

// 提交异步任务
executorService.submit(() -> {
    // ===== 工作线程 =====
    // 恢复上下文
    ContextManager.continued(snapshot);

    // 在工作线程中创建 Local Span
    Span localSpan = ContextManager.createLocalSpan("asyncTask");

    // 业务逻辑...
    doSomething();

    // 停止 Span
    ContextManager.stopSpan();
});
```

**为什么跨线程需要 ContextSnapshot？**

```
主线程                               工作线程
  │                                    │
  │ ContextManager.capture()           │
  │  ↓ 快照当前 Trace 上下文             │
  │  executorService.submit() ────────→│
  │                                    │ ContextManager.continued(snapshot)
  │                                    │  ↓ 恢复 Trace 上下文
  │                                    │ doSomething()
  │                                    │  ↓ 在工作线程中创建 Span
  │                                    │  ↓ Span 与主线程的 Span 关联
  │                                    │
  ▼                                    ▼
  主线程的 Segment  ├── Span-0 (Local, 异步任务)
```

### 4. 跨语言传播

sw8 协议是语言无关的，所有语言的 SkyWalking Agent 都实现了相同的编码/解码逻辑。这使得：

- Java Agent 发起的请求 → 可以被 Python Agent 正确接收和关联
- Go Agent 发起的请求 → 可以被 Node.js Agent 正确接收和关联
- 任何语言的 Agent 产生的 Trace → 都可以被 OAP 正确组装

```
┌──────────────┐     sw8     ┌──────────────┐     sw8     ┌──────────────┐
│ Java Agent   │ ─────────→  │ Go Agent     │ ─────────→  │ Python Agent │
│ (OrderService)│             │(UserService) │             │(NotifyService)│
└──────────────┘             └──────────────┘             └──────────────┘
         │                          │                           │
         ▼                          ▼                           ▼
    Segment-A                   Segment-B                   Segment-C
    traceId 相同: abc123.1.1690000000000
```

### 5. TraceId / SegmentId / SpanId 生成规则

#### 5.1 TraceId 生成

```
TraceId 格式：{global_id}.{thread_id}.{timestamp}

示例：e8a2f3b4c5d6e7f8a9b0c1d2e3f4a5b6.1.1690000000000

组成部分：
- global_id: 全局唯一 UUID（去掉连字符）
- thread_id: 线程编号（从 1 开始递增）
- timestamp: 毫秒级时间戳
```

**为什么 TraceId 包含时间戳？**

1. 排序友好：按时间排序查询时性能更好
2. 可读性：从 TraceId 可以直观看到请求发生的时间
3. 存储优化：BanyanDB/ES 可以利用时间戳前缀做分区

#### 5.2 SegmentId 生成

```
SegmentId 格式：{UUID}

示例：a1b2c3d4-e5f6-7890-abcd-ef1234567890

每个服务实例在处理一个请求时生成一个唯一的 SegmentId。
不同服务实例的 SegmentId 互不相关，通过 traceId 关联。
```

#### 5.3 SpanId 生成

```
SpanId 格式：整数（从 0 开始递增）

在一个 Segment 内：
- Span-0：第一个 Span（通常是 Entry Span）
- Span-1：第二个 Span
- Span-2：第三个 Span
- ...

parentSpanId = -1 表示根 Span
```

### 6. 忽略端点与 Trace 忽略

#### 6.1 忽略端点（ignore_suffix）

通过 `agent.ignore_suffix` 配置，可以忽略特定 URL 后缀的请求（如静态资源），避免产生无意义的 Trace：

```properties
agent.ignore_suffix=.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.woff,.woff2,.html,.htm
```

**原理**：Agent 在拦截 HTTP 请求时，检查请求 URL 的后缀，如果匹配 `ignore_suffix`，则跳过 Span 创建。

#### 6.2 Trace 忽略（sample rate）

通过 `agent.sample_n_per_3_secs` 配置，可以控制采样率：

```properties
# 每 3 秒最多采样 N 个 Trace
# -1：全采样（不限制）
# 0：不采样
# N：每 3 秒最多采样 N 个 Trace
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}
```

**采样策略**：
- 全采样（-1）：记录所有请求，适合低流量场景
- 强制采样：对于错误请求，即使超过采样上限也会记录
- 慢请求采样：OAP 端可以配置慢请求阈值，对慢请求强制采样

---

## 常见面试题

### Q1: sw8 协议和 W3C TraceContext 标准有什么区别？

| 对比维度 | sw8（SkyWalking） | W3C TraceContext |
|---------|-------------------|-----------------|
| 标准化程度 | SkyWalking 自研 | W3C 国际标准 |
| Header Key | `sw8` | `traceparent` + `tracestate` |
| 编码格式 | Base64（单 Header） | 明文（两个 Header） |
| 携带信息 | TraceId + 父服务信息 | TraceId + SpanId + TraceFlags |
| 生态兼容 | 仅 SkyWalking 生态 | 所有遵循 W3C 标准的工具 |
| 性能 | 更紧凑（单 Header） | 稍大（两个 Header） |

**趋势**：SkyWalking v9+ 同时支持 sw8 和 W3C TraceContext，通过 OTLP Receiver 可以接收 W3C 标准的 Trace。

### Q2: ContextCarrier 和 ContextSnapshot 各自用在什么场景？为什么不能混用？

- **ContextCarrier**：用于**跨进程**传播，需要序列化/反序列化（因为要经过网络传输）
- **ContextSnapshot**：用于**跨线程**传播，内存中直接传递对象引用（因为同一进程内）

**不能混用的原因**：
- ContextCarrier 需要序列化开销，不适合高频率的线程切换
- ContextSnapshot 没有序列化能力，不能通过网络传输
- 如果跨线程使用了 ContextCarrier，会浪费序列化开销
- 如果跨进程使用 ContextSnapshot，数据无法传输

### Q3: 异步线程池场景下，如何保证 Trace 不丢失？

```java
// 正确的做法：使用 ContextSnapshot
@Autowired
private ThreadPoolTaskExecutor executor;

public void processOrder() {
    // 1. 在业务线程中捕获上下文
    ContextSnapshot snapshot = ContextManager.capture();

    // 2. 提交异步任务
    executor.submit(() -> {
        // 3. 在工作线程中恢复上下文
        ContextManager.continued(snapshot);

        try {
            // 4. 创建 Local Span
            Span span = ContextManager.createLocalSpan("async-process-order");
            // 业务逻辑...
        } finally {
            ContextManager.stopSpan();
        }
    });
}
```

**常见的错误做法**：
- ❌ 不做任何处理 → 工作线程的 Span 丢失，Trace 断链
- ❌ 使用 ContextCarrier → 序列化开销大，且语义不对
- ❌ 使用 ThreadLocal 手动传递 → 线程池复用导致上下文污染

### Q4: TraceId 为什么要包含时间戳？

1. **排序友好**：按时间戳排序查询时，B+Tree 索引效率更高
2. **可读性**：从 TraceId 直接看出请求发生时间，排查问题时更直观
3. **存储分区**：BanyanDB 和 ES 可以利用时间戳前缀做数据分区，提升查询性能
4. **全局唯一性**：UUID + 时间戳 + 线程号，三重保障唯一性

---

## 延伸阅读

- SkyWalking Java Agent 源码（ContextManager 类）：`apm-agent-core/src/main/java/org/apache/skywalking/apm/agent/core/context/ContextManager.java`
- W3C TraceContext 标准：[https://www.w3.org/TR/trace-context/](https://www.w3.org/TR/trace-context/)
- OpenTelemetry 传播协议：[https://opentelemetry.io/docs/specs/otel/context/api-propagators/](https://opentelemetry.io/docs/specs/otel/context/api-propagators/)