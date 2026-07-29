# 14 - SkyWalking UI 与可视化

## 核心概念

### 1. UI 架构概览

SkyWalking UI（RocketBot）是基于 React + Ant Design 构建的单页应用（SPA），通过 **GraphQL API** 与 OAP 通信。

```mermaid
graph TB
    subgraph browser["Browser（React SPA）"]
        subgraph rocketbot["RocketBot UI"]
            dash["Dashboard（仪表盘）"]
            topo["Topology（拓扑图）"]
            trace["Trace（追踪详情）"]
            log["Log（日志面板）"]
            alarm["Alarm（告警面板）"]
            profiling["Profiling（性能剖析）"]
            settings["Settings（设置）"]
        end
    end

    subgraph oap["OAP Server（端口 12800）"]
        query_plugin["server-query-plugin（GraphQL 查询引擎）"]
    end

    rocketbot -- "GraphQL API（HTTP POST /graphql）" --> query_plugin
```

### 2. Dashboard（仪表盘）

#### 2.1 全局仪表盘

展示所有服务的整体健康状态：

```mermaid
graph TB
    subgraph dash["SkyWalking Dashboard"]
        kpi1["服务总数: 15"]
        kpi2["Apdex: 0.95"]
        kpi3["平均RT: 45ms"]
        kpi4["成功率: 99.5%"]

        subgraph svc_list["服务列表（Apdex ｜ Cpm ｜ RT）"]
            s1["order-service — 0.98 ｜ 120 ｜ 35ms"]
            s2["user-service — 0.95 ｜ 85 ｜ 42ms"]
            s3["inventory-service — 0.92 ｜ 45 ｜ 58ms"]
            s4["payment-service — 0.99 ｜ 30 ｜ 28ms"]
            s5["gateway-service — 1.00 ｜ 200 ｜ 15ms"]
        end

        subgraph slow_top["慢服务 Top 5（按平均响应时间）"]
            t1["inventory-service → 58ms"]
            t2["user-service → 42ms"]
            t3["order-service → 35ms"]
        end
    end
```

#### 2.2 服务仪表盘

选定一个服务后，展示该服务的详细指标：

```mermaid
graph TB
    subgraph svc_dash["order-service 仪表盘"]
        subgraph apdex_box["Apdex"]
            a1["0.98"]
        end
        subgraph pct["响应时间百分位"]
            p1["P50: 15ms ｜ P75: 25ms"]
            p2["P90: 35ms ｜ P95: 50ms"]
            p3["P99: 100ms"]
        end
        subgraph ep_list["端点列表（QPS ｜ RT ｜ P99 ｜ SLA）"]
            e1["GET:/order/{id} — 50 ｜ 10ms ｜ 20ms ｜ 100%"]
            e2["POST:/order/create — 30 ｜ 25ms ｜ 50ms ｜ 99.5%"]
            e3["PUT:/order/update — 20 ｜ 15ms ｜ 30ms ｜ 100%"]
            e4["DELETE:/order/{id} — 10 ｜ 12ms ｜ 25ms ｜ 100%"]
        end
        subgraph slow_ep["慢端点 Top 5"]
            se1["POST:/order/create → P99: 50ms"]
            se2["PUT:/order/update → P99: 30ms"]
        end
        subgraph jvm["实例 JVM 指标"]
            j1["堆内存: 512MB / 1024MB (50%)"]
            j2["GC 次数: Young 15/min, Old 0/min"]
            j3["GC 耗时: Young 50ms, Old 0ms"]
            j4["线程数: 活跃 45, 峰值 120"]
        end
    end
```

### 3. Topology（拓扑图）

#### 3.1 服务拓扑图

自动发现和展示服务之间的调用关系：

```mermaid
graph TD
    gateway["Gateway<br/>（gateway）"]
    gateway --> order["Order Service"]
    gateway --> user["User Service"]
    gateway --> inventory["Inventory Service"]
    order --> mysql["MySQL<br/>（虚拟）"]
    user --> redis1["Redis<br/>（虚拟）"]
    inventory --> redis2["Redis<br/>（虚拟）"]
```

**拓扑图展示的信息**：
- 节点大小 → 调用量（Cpm）
- 连线粗细 → 调用频率
- 节点颜色 → 健康状态（绿=正常，黄=警告，红=异常）
- 连线标注 → 平均响应时间 + 调用量

#### 3.2 拓扑图交互

- **点击节点**：查看该服务的详细指标
- **点击连线**：查看两个服务间的调用指标（Client/Server 双向视角）
- **拖拽节点**：手动调整布局
- **时间范围选择**：查看不同时间段的拓扑
- **层级筛选**：只显示特定 Layer 的服务（GENERAL/DB/CACHE/MQ）

### 4. Trace（追踪详情）

#### 4.1 Trace 列表

```mermaid
graph TB
    subgraph trace_list["Trace 列表（时间范围: 2024-07-17 10:00 - 10:15 ｜ 状态: 成功 + 错误 ｜ 最小耗时: 100ms）"]
        t1["abc...001.1690000000000 — 150ms ｜ ✅ 成功 ｜ 12 个 Span"]
        t2["abc...002.1690000000000 — 80ms ｜ ✅ 成功 ｜ 8 个 Span"]
        t3["abc...003.1690000000000 — 500ms ｜ ❌ 错误 ｜ 15 个 Span"]
        t4["abc...004.1690000000000 — 120ms ｜ ✅ 成功 ｜ 10 个 Span"]
    end
```

#### 4.2 Trace 详情——调用树

Trace: abc...001.1690000000000 ｜ 总耗时: 150ms

```mermaid
graph TD
    subgraph seg_gw["Gateway（10ms）"]
        g1["[Entry] GET /api/order → 10ms"]
        g2["[Exit] POST /order → OrderService → 8ms"]
    end
    subgraph seg_order["OrderService（140ms）"]
        o1["[Entry] POST /order → 140ms"]
        o2["[Exit] GET /user/123 → UserService → 50ms"]
        o3["[Exit] SELECT * FROM order WHERE... → MySQL → 30ms"]
        o4["[Exit] SET order:123 → Redis → 2ms"]
    end
    subgraph seg_user["UserService（50ms）"]
        u1["[Entry] GET /user/123 → 50ms"]
        u2["[Exit] SELECT * FROM user WHERE id=? → MySQL → 45ms"]
    end
    g2 -- "跨进程传播" --> o1
    o2 -- "跨进程传播" --> u1
```

时间线视图（0ms - 150ms）：

```mermaid
gantt
    title Trace 时间线视图
    dateFormat x
    axisFormat %L ms
    section 调用链
    Gateway（10ms）: 0, 10
    OrderService（140ms）: 10, 150
    UserService（50ms）: 20, 70
    MySQL（30ms）: 20, 50
    Redis（2ms）: 20, 22
```

#### 4.3 Span 详情

点击一个 Span，可以查看详细信息：

Span 详情：

```mermaid
graph TD
    span["Span ID: 0 ｜ 操作名: GET:/user/123"]
    span --> type["类型: Entry"]
    span --> comp["组件: SpringMVC（ID: 14）"]
    span --> layer["层级: Http"]
    span --> cost["耗时: 50ms"]
    span --> status["状态: ✅ 成功"]
    span --> tags["Tags"]
    tags --> t1["http.method: GET"]
    tags --> t2["http.status_code: 200"]
    tags --> t3["url: http://user-service:8080/user/123"]
    tags --> t4["http.params: id=123"]
    span --> logs["Logs"]
    logs --> l1["[10:00:00.100] start_processing"]
    logs --> l2["[10:00:00.150] end_processing"]
    span --> events["附件事件: 无"]
```

### 5. Log（日志面板）

#### 5.1 Trace-Log 关联

SkyWalking 支持将 Trace 与日志关联。点击一个 Span，可以查看该 Span 关联的日志：

Trace-Log 关联原理：

```mermaid
graph LR
    s1["1. Agent 将 TraceId 注入到日志 MDC（Mapped Diagnostic Context）"] --> s2["2. 日志框架（Logback/Log4j2）在日志中输出 TraceId"]
    s2 --> s3["3. OAP 端存储日志，按 TraceId 索引"]
    s3 --> s4["4. UI 根据 TraceId 查询关联日志"]
```

**Logback 配置示例**：
```xml
<!-- logback.xml -->
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <!-- %tid 是 SkyWalking 注入的 TraceId -->
        <pattern>[%tid] %d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

#### 5.2 日志查询

```mermaid
graph TB
    subgraph log_query["日志查询（服务: order-service ｜ 端点: POST:/order/create ｜ TraceId: abc...001.1690000000000）"]
        l1["[abc...001] 10:00:00.100 INFO OrderController - 创建订单"]
        l2["[abc...001] 10:00:00.120 INFO OrderService - 验证库存"]
        l3["[abc...001] 10:00:00.150 ERROR OrderService - 库存不足！"]
        l4["[abc...001] 10:00:00.200 INFO OrderController - 订单失败"]
        l1 --> l2 --> l3 --> l4
    end
```

### 6. Profiling（性能剖析）

#### 6.1 线程栈采样

SkyWalking Profiling 通过定期采样线程栈，生成火焰图：

```mermaid
graph TD
    main["main（100%）"]
    main --> filter["doFilter（95%）"]
    filter --> dispatcher["DispatcherServlet（90%）"]
    dispatcher --> controller["OrderController.create（80%）"]
    controller --> service["OrderService.createOrder（75%）"]
    service --> check["InventoryService.checkStock（60%）"]
    check --> redis["Redis.get（40%）"]
    redis --> mysql["MySQL.select（30%）"]
    mysql --> json["JSON.serialize（10%）"]
    hotspot["热点分析：checkStock 方法占用 60% 的 CPU 时间，建议优化"]
    check --> hotspot
```

order-service 线程栈火焰图（采样 100 次，持续 5 分钟），自上而下为调用栈，宽度对应 CPU 时间占比。

### 7. Custom Dashboard（自定义仪表盘）

SkyWalking v9+ 支持自定义仪表盘，通过 Widget 配置：

```yaml
# 自定义仪表盘配置
dashboard:
  name: "我的仪表盘"
  widgets:
    - name: "订单服务 QPS"
      type: "line"
      metrics: "service_cpm"
      conditions:
        - service: "order-service"
    - name: "订单服务 P99 延迟"
      type: "line"
      metrics: "endpoint_p99"
      conditions:
        - service: "order-service"
        - endpoint: "POST:/order/create"
```

---

## 常见面试题

### Q1: SkyWalking 的拓扑图是如何生成的？

拓扑图通过分析 **Exit Span 和 Entry Span 的匹配关系** 自动生成：

1. 服务 A 创建一个 Exit Span（调用服务 B）
2. 服务 B 创建一个 Entry Span（接收服务 A 的请求）
3. OAP 通过 traceId 匹配 A 的 Exit Span 和 B 的 Entry Span
4. 统计 A → B 的调用量、延迟、成功率
5. UI 根据这些关系数据渲染拓扑图

### Q2: Trace-Log 关联是如何实现的？

通过 **TraceId 注入日志 MDC** 实现：

1. SkyWalking Agent 在创建 Span 时，将 TraceId 注入到 SLF4J 的 MDC 中
2. 日志框架（Logback/Log4j2）在输出日志时，自动包含 TraceId
3. Agent 或 OAP 收集日志，按 TraceId 索引
4. UI 中点击一个 Span，根据 TraceId 查询关联的日志

### Q3: 性能剖析（Profiling）的原理是什么？

SkyWalking Profiling 使用 **On-CPU 线程栈采样**：

1. OAP 下发 Profiling 任务到 Agent（指定目标服务、持续时间）
2. Agent 在指定时间内，定期（如每 10ms）采样目标线程的调用栈
3. 采样数据上报到 OAP
4. OAP 聚合所有采样数据，生成火焰图
5. 火焰图中宽度越大的方法，占用的 CPU 时间越多

**优势**：相比传统 Profiler（如 JProfiler），SkyWalking Profiling 对生产环境影响极小（< 1% CPU 开销）。

---

## 延伸阅读

- SkyWalking UI 源码：[https://github.com/apache/skywalking-booster-ui](https://github.com/apache/skywalking-booster-ui)
- GraphQL API 文档：[https://skywalking.apache.org/docs/main/latest/en/api/query-protocol/](https://skywalking.apache.org/docs/main/latest/en/api/query-protocol/)