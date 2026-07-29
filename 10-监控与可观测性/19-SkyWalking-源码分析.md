# 19 - SkyWalking 源码分析

## 核心概念

### 1. 源码分析路线图

本章从**端到端**的角度，追踪一次 HTTP 请求在 SkyWalking 中的完整链路：

```mermaid
graph LR
    s1["① Agent 端：请求拦截 → Span 创建 → 上下文注入 → 出口上报"] --> s2["② 网络传输：gRPC 流式上报 → Protobuf 序列化"]
    s2 --> s3["③ OAP 端：DataCarrier 接收 → TraceAnalyzer 解析 → OAL 聚合"]
    s3 --> s4["④ 存储层：指标写入 → TTL 清理"]
    s4 --> s5["⑤ 查询层：GraphQL 查询 → UI 展示"]
```

源码分析路线图：一次 HTTP 请求的完整生命

### 2. Agent 端源码分析

#### 2.1 启动流程

```mermaid
graph TD
    premain["SkyWalkingAgent.premain(String agentArgs, Instrumentation instrumentation)"]

    premain --> step1["1. 加载配置<br/>SnifferConfigInitializer.initializeCoreConfig(agentArgs)"]
    step1 --> s11["读取 agent.config 文件"]
    step1 --> s12["读取系统属性（-D 参数）"]
    step1 --> s13["合并配置（系统属性优先）"]

    premain --> step2["2. 加载插件<br/>PluginFinder pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins())"]
    step2 --> s21["扫描 plugins/ 目录下的所有 JAR"]
    step2 --> s22["读取 META-INF/skywalking-plugin.def"]
    step2 --> s23["通过 PluginClassLoader 加载插件类"]
    step2 --> s24["构建 PluginFinder（插件查找器）"]

    premain --> step3["3. 初始化服务管理器<br/>ServiceManager.INSTANCE.boot()"]
    step3 --> s31["GRPCChannelManager：管理 gRPC 连接"]
    step3 --> s32["ServiceAndEndpointRegisterClient：服务注册"]
    step3 --> s33["TraceSegmentServiceClient：Trace 上报"]
    step3 --> s34["JVMService：JVM 指标采集"]
    step3 --> s35["SamplingService：采样控制"]

    premain --> step4["4. 注册 ClassFileTransformer<br/>new AgentBuilder.Default()"]
    step4 --> s41[".type(pluginFinder.buildMatch()) — 构建类匹配器"]
    s41 --> s42[".transform(new Transformer()) — 字节码转换"]
    s42 --> s43[".with(new Listener()) — 转换监听器"]
    s43 --> s44[".installOn(instrumentation) — 安装到 JVM"]

    premain --> step5["5. 启动完成，Agent 开始工作"]
```

#### 2.2 请求拦截与 Span 创建

当一个 HTTP 请求到达时，Tomcat 插件（或 SpringMVC 插件）拦截处理流程：

```
// 简化版源码流程（Tomcat 插件拦截器）

// ① beforeMethod：请求到达时
public void beforeMethod(EnhancedInstance objInst, Method method,
                         Object[] allArguments, ...) {
    HttpServletRequest request = (HttpServletRequest) allArguments[0];

    // 从 HTTP Header 中提取上下文
    ContextCarrier carrier = new ContextCarrier();
    CarrierItem items = carrier.items();
    while (items.hasNext()) {
        items.next().setHeadValue(
            request.getHeader(items.getHeadKey())  // 提取 sw8 或 traceparent
        );
    }

    // 创建 Entry Span
    AbstractSpan span = ContextManager.createEntrySpan(
        request.getMethod() + ":" + request.getRequestURI(),  // operationName
        carrier                                                 // 上下文
    );

    // 设置 Span 属性
    span.setComponent(ComponentsDefine.TOMCAT);
    span.setLayer(SpanLayer.HTTP);
    Tags.HTTP.METHOD.set(span, request.getMethod());
    Tags.URL.set(span, request.getRequestURL().toString());
}

// ② afterMethod：请求处理完成时
public Object afterMethod(EnhancedInstance objInst, Method method,
                          Object[] allArguments, ..., Object ret) {
    HttpServletResponse response = (HttpServletResponse) allArguments[1];

    // 设置状态码
    AbstractSpan span = ContextManager.activeSpan();
    Tags.HTTP_RESPONSE_STATUS_CODE.set(span, response.getStatus());

    // 判断是否有错误
    if (response.getStatus() >= 400) {
        span.errorOccurred();
    }

    // 停止 Span（Span 出栈）
    ContextManager.stopSpan();

    return ret;
}

// ③ handleMethodException：请求处理异常时
public void handleMethodException(EnhancedInstance objInst, Method method,
                                   Object[] allArguments, ..., Throwable t) {
    AbstractSpan span = ContextManager.activeSpan();
    span.errorOccurred();
    span.log(t);  // 记录异常到 Span Logs
}
```

#### 2.3 Span 出口与上报

当所有 Span 完成（Span 栈为空）后，Segment 自动完成并上报：

```
// ContextManager.stopSpan() 内部逻辑
public static void stopSpan() {
    AbstractTracerContext context = get();
    context.stopSpan(activeSpan);

    // 如果 Span 栈为空，说明 Segment 完成
    if (context.finish()) {
        // 序列化 Segment 为 Protobuf
        SegmentObject segment = context.toSegmentObject();

        // 放入 TraceBuffer（异步上报队列）
        TraceSegmentServiceClient.INSTANCE.send(segment);
    }
}
```

### 3. OAP 端源码分析

#### 3.1 模块启动流程

```mermaid
graph TD
    main["OAPServerBootstrap.main()"]

    main --> step1["1. 加载 application.yml<br/>ApplicationConfigLoader.load()"]
    main --> step2["2. 初始化 ModuleManager<br/>ModuleManager manager = new ModuleManager()"]
    main --> step3["3. 初始化模块（prepare 阶段）<br/>BootstrapFlow.start(manager, config)"]
    step3 --> m1["CoreModule → CoreModuleProvider"]
    step3 --> m2["StorageModule → BanyanDB/ES/JDBC Provider"]
    step3 --> m3["ReceiverModule → gRPC/Kafka Provider"]
    step3 --> m4["AnalyzerModule → AgentAnalyzer Provider"]
    step3 --> m5["QueryModule → GraphQL Provider"]
    step3 --> m6["AlarmModule → Alarm Provider"]
    step3 --> m7["ClusterModule → ZK/K8s/Nacos Provider"]
    main --> step4["4. 启动模块（start 阶段）"]
    step4 --> s41["启动 gRPC 服务（端口 11800）"]
    step4 --> s42["启动 HTTP 服务（端口 12800）"]
    step4 --> s43["启动定时任务"]
    main --> step5["5. 通知完成（notifyAfterCompleted）"]
```

#### 3.2 TraceAnalyzer：Segment 解析

```java
// 源码：TraceAnalyzer.java（简化版）
public void doAnalysis(SegmentObject segmentObject) {
    // 1. 创建 Span 监听器
    createSpanListeners();

    // 2. 通知 Segment 级别监听器
    notifySegmentListener(segmentObject);

    // 3. 遍历每个 Span，根据类型分发
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

    // 4. 所有监听器完成分析后，触发 build（生成 Source → 聚合指标）
    notifyListenerToBuild();
}
```

**监听器链**（源码中的实际监听器）：

| 监听器 | 触发点 | 生成的 Source |
|--------|--------|--------------|
| `FirstAnalysisListener` | 每个 Segment 的第一个 Span | EndpointMeta |
| `EntryAnalysisListener` | Entry Span | Service, Endpoint |
| `ExitAnalysisListener` | Exit Span | ServiceRelation |
| `LocalAnalysisListener` | Local Span | 附加分析 |
| `RPCAnalysisListener` | RPC 类型 Span | ServiceRelation（RPC） |
| `DatabaseSlowStatementBuilder` | DB 慢查询 Exit Span | DatabaseSlowStatement |
| `EndpointDependencyBuilder` | 端点间调用 | EndpointRelation |
| `VirtualDatabaseProcessor` | DB 类型 Exit Span | 虚拟数据库服务 |
| `VirtualCacheProcessor` | Cache 类型 Exit Span | 虚拟缓存服务 |
| `VirtualMQProcessor` | MQ 类型 Exit Span | 虚拟 MQ 服务 |
| `SampledTraceBuilder` | 被采样的 Segment | 完整 Trace 记录 |

#### 3.3 OAL 引擎执行

OAL 指标聚合流程：

```mermaid
graph TD
    s1["1. SourceReceiver 接收 Source 数据<br/>SourceReceiverImpl.receive(Source source)"]
    s1 --> s2["2. 将 Source 数据分发到对应的 OAL 聚合实例<br/>MetricsStreamProcessor.getInstance().in(source)"]
    s2 --> s3["3. 每个 OAL 聚合实例执行 combine()<br/>实时更新分钟级指标（L1 Metrics）"]
    s3 --> s4["4. Downsampling 定时任务"]
    s4 --> d1["每分钟：L1 指标 → 写入存储"]
    s4 --> d2["每小时：L1 指标 → 聚合为 L2 指标 → 写入存储"]
    s4 --> d3["每天：L2 指标 → 聚合为 L3 指标 → 写入存储"]
```

### 4. 端到端代码走读

以下是一次 HTTP 请求（`GET /order/123`）的完整代码追踪：

```mermaid
graph TB
    subgraph order_side["order-service 端"]
        s1["1. 用户请求到达 order-service（Tomcat 端口 8080）"]
        s2["2. Tomcat 插件拦截 → 创建 Entry Span<br/>operationName: GET:/order/123<br/>TracingContext 栈: [Entry Span]"]
        s3["3. OrderController 调用 OrderService.findById()<br/>无插件拦截 → 不创建 Span"]
        s4["4. OrderService 调用 UserService（通过 Feign HTTP 客户端）"]
        s5["5. Feign 插件拦截 → 创建 Exit Span<br/>operationName: GET:/user/123 ｜ peer: user-service:8080<br/>创建 ContextCarrier<br/>注入 sw8 Header 到 HTTP 请求<br/>TracingContext 栈: [Entry Span, Exit Span]"]
        s9["9. Feign Exit Span 完成 → stopSpan()<br/>TracingContext 栈: [Entry Span]"]
        s10["10. OrderController 返回响应"]
        s11["11. Tomcat Entry Span 完成 → stopSpan()<br/>TracingContext 栈: [] → Segment 完成"]
        s12["12. Segment 序列化 → 放入 TraceBuffer"]
        s13["13. 异步 gRPC 上报到 OAP"]
        s1 --> s2 --> s3 --> s4 --> s5
        s9 --> s10 --> s11 --> s12 --> s13
    end

    subgraph user_side["user-service 端"]
        s7["7. Tomcat 插件拦截 → 提取 sw8 Header → 创建 Entry Span<br/>关联到同一个 Trace"]
        s8["8. UserService 返回响应"]
        s7 --> s8
    end

    subgraph oap_side["14. OAP 端"]
        o1["gRPC 接收 → 放入 DataCarrier"]
        o2["Analyzer 消费 → TraceAnalyzer.doAnalysis()"]
        o2a["FirstAnalysisListener → 提取 Endpoint 元数据"]
        o2b["EntryAnalysisListener → 生成 Service/Endpoint Source"]
        o2c["ExitAnalysisListener → 生成 ServiceRelation Source"]
        o3["OAL 引擎 → 聚合指标"]
        o4["指标写入存储"]
        o5["UI 查询 → GraphQL → 展示"]
        o1 --> o2
        o2 --> o2a
        o2 --> o2b
        o2 --> o2c
        o2a --> o3
        o2b --> o3
        o2c --> o3
        o3 --> o4 --> o5
    end

    s5 -- "6. 发送 HTTP 请求到 user-service" --> s7
    s8 -- "响应返回" --> s9
    s13 --> o1
```

---

## 常见面试题

### Q1: 如果让你设计一个 APM 系统，你会怎么设计？

可以从 SkyWalking 的架构中总结出核心设计思路：

1. **数据采集层**：通过 Agent 字节码增强实现零侵入，覆盖主流框架
2. **数据传输层**：gRPC 流式上报 + Kafka 桥接，支持高吞吐
3. **数据分析层**：DSL 驱动的指标聚合（OAL/MAL/LAL），灵活可扩展
4. **数据存储层**：可插拔存储引擎，按场景选择（H2/MySQL/ES/BanyanDB）
5. **数据查询层**：GraphQL + MQE 灵活查询
6. **可视化层**：仪表盘 + 拓扑图 + 追踪详情 + 火焰图

### Q2: SkyWalking Agent 的 Span 栈管理是如何实现的？

通过 **TracingContext + ThreadLocal + Span 栈** 实现：

1. 每个线程有独立的 `TracingContext`（通过 ThreadLocal 管理）
2. `TracingContext` 内部维护一个 Span 栈（Stack）
3. `createEntrySpan()` / `createExitSpan()` → 压栈
4. `stopSpan()` → 出栈
5. 栈为空时 → Segment 完成 → 上报

**为什么用栈**：因为方法调用是嵌套的（先进后出），Span 的生命周期也是嵌套的。

### Q3: SkyWalking 源码中的 Compile 阶段做了什么？

OAL 脚本（.oal 文件）不是直接解释执行的，而是**编译为 Java 类**：

1. ANTLR4 解析 OAL 语法 → AST（抽象语法树）
2. AST → Java 源代码（`OALClassGenerator` 生成）
3. Java 源代码 → .class 字节码（`javac` 编译）
4. 运行时加载 .class 文件 → 反射创建聚合实例

**好处**：编译后的代码性能接近原生 Java 代码，远高于解释执行。

---

## 延伸阅读

- SkyWalking 源码仓库：[https://github.com/apache/skywalking](https://github.com/apache/skywalking)
- SkyWalking Java Agent 源码：[https://github.com/apache/skywalking-java](https://github.com/apache/skywalking-java)
- 核心类浏览入口：
  - Agent: `SkyWalkingAgent.java`
  - OAP Entry: `OAPServerBootstrap.java`
  - Analyzer: `TraceAnalyzer.java`
  - OAL: `OALParser.g4`