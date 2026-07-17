# 21 - SkyWalking 生产实践

## 核心概念

### 1. 大规模部署架构

#### 1.1 1000+ Agent 的部署架构

```
┌──────────────────────────────────────────────────────────────────┐
│  大规模部署架构（1000+ Agent）                                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  应用层（1000+ Agent）                                      │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐     ┌────────┐        │   │
│  │  │App 1   │ │App 2   │ │App 3   │ ... │App N   │        │   │
│  │  │+ Agent │ │+ Agent │ │+ Agent │     │+ Agent │        │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘     └───┬────┘        │   │
│  └──────┼──────────┼──────────┼───────────────┼──────────────┘   │
│         │          │          │               │                   │
│         │          │     Kafka Cluster（可选）  │                   │
│         │          │          │               │                   │
│         ▼          ▼          ▼               ▼                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  OAP Receiver 层（3-5 节点，无状态）                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │    │
│  │  │Receiver 1│ │Receiver 2│ │Receiver 3│                 │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘                 │    │
│  └───────┼─────────────┼─────────────┼───────────────────────┘    │
│          │             │             │                             │
│          │        gRPC 转发（数据分片）                              │
│          │             │             │                             │
│          ▼             ▼             ▼                             │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  OAP Aggregator 层（5-7 节点，有状态）                       │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │    │
│  │  │Aggreg. 1 │ │Aggreg. 2 │ │Aggreg. 3 │                 │    │
│  │  │(分片 0-49)│ │(分片50-99)│ │(分片100+)│                 │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘                 │    │
│  └───────┼─────────────┼─────────────┼───────────────────────┘    │
│          │             │             │                             │
│          └─────────────┼─────────────┘                             │
│                        │                                           │
│                        ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Storage 集群（BanyanDB/ES）                               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │    │
│  │  │ Node 1   │ │ Node 2   │ │ Node 3   │                 │    │
│  │  └──────────┘ └──────────┘ └──────────┘                 │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.2 JVM 调优建议

```bash
# OAP Server JVM 参数（oapService.sh）
JAVA_OPTS="
  -Xms4g -Xmx4g                               # 堆内存 4G（建议 4-8G）
  -XX:+UseG1GC                                # 使用 G1 GC
  -XX:MaxGCPauseMillis=200                    # GC 暂停时间 ≤ 200ms
  -XX:G1HeapRegionSize=8m                     # G1 Region 大小 8MB
  -XX:+ParallelRefProcEnabled                 # 并行引用处理
  -XX:+UnlockExperimentalVMOptions
  -XX:G1NewSizePercent=5                      # 年轻代初始占比 5%
  -XX:InitiatingHeapOccupancyPercent=45       # 45% 堆占用时触发并发标记
  -XX:+DisableExplicitGC                      # 禁用 System.gc()
  -XX:+HeapDumpOnOutOfMemoryError             # OOM 时生成 heap dump
  -XX:HeapDumpPath=/var/logs/skywalking/      # heap dump 路径
"
```

### 2. ES 调优建议

#### 2.1 索引模板优化

```json
{
  "template": "sw_*",
  "settings": {
    "number_of_shards": "3",
    "number_of_replicas": "1",
    "refresh_interval": "30s",
    "translog.durability": "async",
    "translog.sync_interval": "30s",
    "merge.policy.max_merged_segment": "5gb"
  }
}
```

| 参数 | 建议值 | 理由 |
|------|--------|------|
| `number_of_shards` | 3-5 | 分片过多增加协调开销，过少不利于分布式 |
| `number_of_replicas` | 1 | 高可用基础，读性能翻倍 |
| `refresh_interval` | 30s | 降低刷新频率，减少磁盘 I/O（APM 不需要实时搜索） |
| `translog.durability` | async | 异步写 translog，提升写入性能 |
| `merge.policy.max_merged_segment` | 5gb | 减少 merge 次数 |

#### 2.2 ES 集群规划

```
Agent 数量 → ES 集群规模建议：

< 100 Agent → 3 节点（1 master + 2 data）
100-500 Agent → 5 节点（3 master + 2 data）
500-1000 Agent → 7 节点（3 master + 4 data）
> 1000 Agent → 考虑 BanyanDB 或 ES 独立集群
```

### 3. 常见问题排查

#### 3.1 Agent 不上报数据

```
排查步骤：
1. 检查 Agent 日志
   └── 查找 "SkyWalking agent starting" 日志
   └── 查找 "Register Service" 日志
   └── 查找连接错误日志

2. 检查网络连通性
   └── telnet oap-server 11800

3. 检查配置
   └── agent.service_name 是否配置
   └── collector.backend_service 是否配置正确

4. 检查 OAP 日志
   └── 是否有服务注册成功的日志

5. 常见原因
   ├── OAP 地址配置错误
   ├── 防火墙阻止 11800 端口
   ├── Agent 版本与 OAP 版本不兼容
   └── 服务名包含特殊字符
```

#### 3.2 拓扑图不完整

```
排查步骤：
1. 检查是否所有服务都安装了 Agent
2. 检查跨线程传播是否正确配置
   └── 异步线程池是否使用了 ContextSnapshot
3. 检查 ignore_suffix 配置
   └── 某些端点是否被忽略
4. 检查采样率
   └── 采样率太低导致部分调用关系丢失

常见原因：
├── 异步线程未使用 ContextSnapshot
├── MQ 消费者未安装 Agent
└── 数据库/缓存没有 Agent（显示为虚拟服务）
```

#### 3.3 OAP 内存溢出

```
排查步骤：
1. 检查 DataCarrier 是否积压
   └── 查看 OAP 指标：oap_analysis_latency

2. 检查 GC 日志
   └── Full GC 频率是否过高

3. 检查存储连接
   └── ES/BanyanDB 是否可访问

4. 解决方案
   ├── 增加堆内存（-Xmx）
   ├── 降低采样率（减少数据处理量）
   ├── 检查 ES 集群健康状态
   ├── 启用 G1 GC
   └── 分离 Receiver 和 Aggregator
```

### 4. 监控自监控（OAP Telemetry）

#### 4.1 OAP 自身的指标

```yaml
# config/telemetry/telemetry-config.yaml
telemetry:
  selector: ${SW_TELEMETRY:prometheus}
  prometheus:
    host: ${SW_TELEMETRY_PROMETHEUS_HOST:0.0.0.0}
    port: ${SW_TELEMETRY_PROMETHEUS_PORT:1234}
```

**OAP 暴露的指标**：

| 指标 | 含义 | 告警建议 |
|------|------|---------|
| `oap_analysis_latency` | 分析延迟 | > 1s 告警 |
| `oap_metrics_data_persistence_latency` | 存储延迟 | > 500ms 告警 |
| `oap_heap_used` | 堆内存使用 | > 80% 告警 |
| `oap_gc_count` | GC 次数 | > 10/min 告警 |
| `oap_grpc_connect_count` | gRPC 连接数 | 监控趋势 |
| `oap_datacarrier_used_percent` | DataCarrier 使用率 | > 80% 告警 |

### 5. 与 Prometheus + Grafana + Loki 协同方案

```
完整可观测性方案：

┌──────────────────────────────────────────────────────────────────┐
│  可观测性技术栈                                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  SkyWalking   │  │  Prometheus  │  │  Loki                │   │
│  │  (Trace +     │  │  (Metrics)   │  │  (Logs)              │   │
│  │   APM指标)    │  │              │  │                      │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
│         │                 │                      │                │
│         └─────────────────┼──────────────────────┘                │
│                           │                                       │
│                           ▼                                       │
│                   ┌──────────────┐                                │
│                   │   Grafana    │                                │
│                   │  (统一展示)   │                                │
│                   └──────────────┘                                │
│                                                                   │
│  分工：                                                           │
│  ├── SkyWalking：分布式追踪 + 应用层指标 + 服务拓扑               │
│  ├── Prometheus：基础设施指标 + 业务指标 + 告警                   │
│  ├── Loki：日志聚合 + 全文搜索                                   │
│  └── Grafana：统一仪表盘 + 告警管理                              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 常见面试题

### Q1: 大规模部署 SkyWalking 需要注意什么？

1. **分离 Receiver 和 Aggregator**：避免接收瓶颈影响分析
2. **使用 Kafka 桥接**：解耦 Agent 和 OAP，削峰填谷
3. **合理设置采样率**：高流量下控制采样率，避免存储爆炸
4. **ES 集群优化**：合理设置分片数、副本数、refresh_interval
5. **OAP JVM 调优**：使用 G1 GC，堆内存 4-8G
6. **监控自监控**：OAP 自身的健康状态也需要监控

### Q2: Agent 不上报数据的常见原因有哪些？

1. **网络问题**：OAP 端口 11800 不可达（防火墙/安全组）
2. **配置错误**：`agent.service_name` 为空或 `collector.backend_service` 配置错误
3. **版本不兼容**：Agent 版本与 OAP 版本不匹配
4. **服务名特殊字符**：服务名包含 `-`、`.` 等特殊字符可能导致注册失败
5. **Agent 加载失败**：查看应用启动日志，确认 Agent 是否成功加载

### Q3: 如何监控 SkyWalking 自身的健康状态？

通过 OAP Telemetry 模块暴露 Prometheus 指标，然后用 Prometheus + Grafana 监控：

1. 配置 `telemetry-config.yaml` 启用 Prometheus Exporter
2. Prometheus 抓取 OAP 的 `/metrics` 端点
3. Grafana 展示 OAP 的健康指标（分析延迟、存储延迟、GC、内存等）
4. 配置告警规则（如分析延迟 > 1s 告警）

---

## 延伸阅读

- SkyWalking 性能调优：[https://skywalking.apache.org/docs/main/latest/en/guides/performance-tuning/](https://skywalking.apache.org/docs/main/latest/en/guides/performance-tuning/)
- SkyWalking Telemetry：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-telemetry/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-telemetry/)