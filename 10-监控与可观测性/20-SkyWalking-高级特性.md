# 20 - SkyWalking 高级特性

## 核心概念

### 1. 采样策略

#### 1.1 为什么需要采样？

在高并发场景下，全量采集 Trace 会带来：
- **存储成本**：海量 Trace 数据占用大量磁盘
- **网络开销**：Agent 上报大量数据占用带宽
- **OAP 压力**：处理和分析全量数据消耗 CPU

**采样就是在可观测性和资源消耗之间做平衡。**

#### 1.2 采样策略类型

采样策略按**决策位置**分两大类，必须先分清（混淆这点是面试最常见的错误）：

**Agent 端前置采样**（上报前决策，能省带宽）：

| 策略 | 配置 | 说明 |
|------|------|------|
| **全采样** | `agent.sample_n_per_3_secs=-1` | 默认值，所有 Trace 都上报，适合低流量 |
| **不采样** | `agent.sample_n_per_3_secs=0` | 计数器恒为 0，不记录任何 Trace，只保留指标 |
| **速率限制** | `agent.sample_n_per_3_secs=N` | 每 3 秒最多采样 N 个 Trace，`AtomicInteger` CAS 抢名额 |
| **动态采样率** | OAP 下发 `trace.sample_rate` | 配置中心热更新，Agent 每 20s 拉取，无需重启 |

**OAP 端后置采样**（数据已到 OAP，只省存储不省带宽）：

| 策略 | 配置 | 说明 |
|------|------|------|
| **慢请求强制采样** | `trace-sampling-policy-settings.yml` 的 `duration` | Trace 耗时超过阈值则强制保留，优先级最高 |
| **错误段强制采样** | `SW_FORCE_SAMPLE_ERROR_SEGMENT=true`（默认开） | `span.isError=true` 的段强制保留 |
| **采样率哈希丢弃** | `trace-sampling-policy-settings.yml` 的 `rate` | `traceId.hashCode()%10000` 落在区间内才保留 |

> ⚠️ **资料勘误**：旧资料常说"错误请求/慢请求在 Agent 端强制采样"，这是**错误**的。Agent 采样决策发生在请求**开始时**（`ContextManagerExtendService.createTraceContext`），此时既不知道是否出错、也不知道耗时多久，机制上不可能基于 error/slow 反向强制采样。错误/慢请求的强制保留是 **OAP 端后置**行为。详见文末「资料勘误与重点提醒」。

#### 1.3 采样策略配置

**① Agent 端：静态前置采样**

```properties
# agent.config（启动时读取，改了要重启）
# -1: 关闭采样，全量上报（默认）；0: 不采样；N>0: 每3秒最多N个Trace
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}
```

实现类 `org.apache.skywalking.apm.agent.core.sampling.SamplingService`：`AtomicInteger` 计数器 + 每 3 秒定时清零，每个 Trace 在 `createTraceContext` 创建时 CAS 抢名额，抢到才创建 `TracingContext`，否则创建 `IgnoredTracerContext`（所有 span 变 noop）。

**② Agent 端：动态采样率下发（无需重启）**

```yaml
# 配置中心（如 Nacos）dataId = configuration-discovery.default.agentConfigurations
# 值是一段 YAML，按 service 分组，每 service 一组 KV：
configurations:
  order-service:
    trace.sample_rate: 1000            # 采样率（每3秒最多1000个）
    trace.ignore_path: /api/health/*   # 忽略路径
  payment-service:
    trace.sample_rate: 500             # 不同 service 独立配置
    trace.ignore_path: /api/metrics/*
```

Agent 每 20 秒通过 gRPC `fetchConfigurations` 主动向 OAP 拉取，OAP 背后对接配置中心转发；`SamplingRateWatcher` 收到后实时覆盖本地 `agent.config`，热更新生效。

**采样率与实例的匹配关系**（按两个维度理解）：

| 维度 | 匹配 key | 说明 |
|------|---------|------|
| **service 级** | `service` 名 | Agent 在 `ConfigurationSyncRequest` 里带 `Config.Agent.SERVICE_NAME`，OAP 按此查表返回对应 KV |
| **instance 级** | 同 service 共享 | 同 service 的所有实例**共享同一份配置**，无法给单个实例单独配 |

```
order-service 的 3 个实例（Pod1/Pod2/Pod3）
        ↓ 全部收到同一份 trace.sample_rate: 1000
   每个实例各自用 AtomicInteger 按 1000/3s 独立限流
        ↓
   合计可能 3000 个/3s（下发的是"每实例配额"，非"全服务总量"）
```

**跨进程采样一致性**：分布式调用时，上游若已采样，下游必须延续，否则链路断裂：

```
order-service(已采样) ──RPC──> payment-service(本该被丢弃)
                                        ↓
                  ContextCarrier 携带采样标记
                                        ↓
            payment-service 收到 carrier，调用 SamplingService.forceSampled()
                                        ↓
                  强制延续采样，链路完整（forceSampled 唯一用途）
```

**③ OAP 端：后置采样策略**

```yaml
# config/trace-sampling-policy-settings.yml（静态文件）
default:
  # 采样率：traceId.hashCode()%10000 < rate 才保留（0~10000）
  rate: 10000
  # 慢阈值(ms)：duration >= duration 强制保留（-1 关闭）
  duration: -1
# 可按服务单独覆盖
service:
  order-service:
    rate: 5000
    duration: 2000
```

```yaml
# application.yml
agent-analyzer:
  default:
    traceSamplingPolicySettingsFile: ${SW_TRACE_SAMPLING_POLICY_SETTINGS_FILE:trace-sampling-policy-settings.yml}
    # 错误段强制保留（OAP 端后置，默认开）
    forceSampleErrorSegment: ${SW_FORCE_SAMPLE_ERROR_SEGMENT:true}
```

> ⚠️ 旧资料里的 `traceSampleRateWatcher` / `SW_SLOW_TRACE_SEGMENT_THRESHOLD` 在 8.6.0 起已废弃，分别合并为 `TraceSamplingPolicyWatcher` 类与 `traceSamplingPolicy` 配置 key。

#### 1.4 两套采样的设计哲学

为什么 SkyWalking 要有 Agent 前置 + OAP 后置**两套**采样？根本原因在于**信息可获得性的时序差**：

```
请求生命周期：
请求开始 ────────────────────────────────────────> 请求结束
   │                                                  │
   ▼                                                  ▼
Agent 决策点                                    OAP 决策点
此时：                                          此时：
· 不知道会不会出错                              · 已知 isError
· 不知道耗时多久                                · 已知 duration
· 只能做"无脑"计数/比例丢弃                     · 可基于 error/slow 智能保留
· 省：网络带宽 + OAP 处理压力                   · 省：存储空间
```

- **Agent 前置采样**省的是**带宽**（数据根本没发出去），但只能用固定比例/计数
- **OAP 后置采样**省的是**存储**（数据已到 OAP），能基于 error/duration 做智能判断

两套不是冗余，而是**互补**：Agent 把洪峰削下去，OAP 在剩余数据里再做精筛。如果只靠 OAP 端采样，带宽照样被打爆；如果只靠 Agent 端采样，错误和慢请求会被随机丢弃，丢失关键排障信息。

#### 1.5 动态采样率下发链路

完整的"配置中心 -> OAP -> Agent"下发链路：

```
┌─────────────┐         ┌──────────────────────────────────────┐    ┌──────────────┐
│ Nacos/Apollo│         │            OAP 端                      │    │    Agent      │
│ /ZK/etcd    │         │                                       │    │              │
│             │  ①注册  │  AgentConfigurationsWatcher           │    │              │
│ dataId:     │<────────│  (item: configuration-discovery        │    │              │
│ configurat- │  监听    │   .default.agentConfigurations)       │    │              │
│ ion-discov- │  变更   │            ▲                          │    │              │
│ ery.default │────────>│            │ ②整段YAML解析成KV         │    │              │
│ .agentConf │         │            ▼                          │    │              │
│             │         │  ConfigurationDiscoveryServiceHandler │    │              │
└─────────────┘         │  gRPC: fetchConfigurations()         │    │              │
                        │            ▲                          │    │              │
                        │            │ ③Agent每20s主动pull       │    │              │
                        └────────────┼──────────────────────────┘    │              │
                                     │                                │              │
                                     └───────────────────────────────>│              │
                                      ④返回 Commands(KV list)          │              │
                                                                      │  CommandService│
                                                                      │  解析          │
                                                                      │      ▼         │
                                                                      │  SamplingRate  │
                                                                      │  Watcher       │
                                                                      │      ▼         │
                                                                      │  实时切换采样率 │
                                                                      └──────────────┘
```

关键澄清：

- **Agent 不直连配置中心**，只通过 gRPC 连 OAP 的 `ConfigurationDiscoveryServiceHandler`
- **OAP 也不存配置**，只是转发层--配置真正存放在 Nacos/Apollo 等
- 下发的是**结构化 KV 列表**（如 `trace.sample_rate`、`trace.ignore_path`），不是单个数字，也不是"远程文件"
- 是 **Agent 主动 pull**（每 20s），不是 OAP push--配置中心变更后，最长 20s 生效
- UUID 机制做增量同步：配置没变时 OAP 直接返回空，省带宽

**拉取间隔可配置（纯配置，无需改源码）**

20s 只是默认值，对应 `Config.Collector.GET_AGENT_DYNAMIC_CONFIG_INTERVAL`（默认 20，单位秒）。改它**不用动代码**，四种渠道任选其一：

| 渠道 | 配置方式 | 适用场景 |
|------|---------|---------|
| agent.config 文件 | `collector.get_agent_dynamic_config_interval=10` | 本地/虚拟机 |
| 环境变量 | `SW_AGENT_COLLECTOR_GET_AGENT_DYNAMIC_CONFIG_INTERVAL=10` | **K8s 首选** |
| 系统属性 | `-Dskywalking.collector.get_agent_dynamic_config_interval=10` | JVM 启动参数 |
| -javaagent 选项 | `-javaagent:skywalking-agent.jar=collector.get_agent_dynamic_config_interval=10` | 启动时注入 |

> ⚠️ **配太小的两个坑**（源码挖出）：
> 1. **单线程执行器**：`ConfigurationDiscoveryService` 用 `newSingleThreadScheduledExecutor`，`scheduleAtFixedRate` 不并发。若单次 gRPC 耗时超过配置间隔，任务排队，实际周期 = `max(配置间隔, 单次调用耗时)`。配 1s 但调用耗 3s，实际还是 3s 一次。
> 2. **gRPC 超时 30s > 轮询间隔**：`fetchConfigurations` 的 deadline 复用全局 `GRPC_UPSTREAM_TIMEOUT`（默认 30s）。OAP 不响应时单次调用阻塞最长 30s，期间多次轮询排队，OAP 恢复后瞬间连发，反而脉冲打 OAP。
>
> 所以若要配很短（如 5s 以下），建议**同时把 `GRPC_UPSTREAM_TIMEOUT` 调小**（如 3s）。注意它是全局参数，trace/heartbeat/profile 上行 gRPC 都用，调小需权衡。源码对该间隔**无下限校验**，配 0 也不报错，但不推荐。
>
> 实践建议：默认 20s 够用；压测快速调参可配 5s + 超时调 3s；生产稳定期可放宽到 20~60s。


### 2. 性能剖析（Profiling）

#### 2.1 工作原理

SkyWalking Profiling 通过 **On-CPU 线程栈采样** 生成火焰图：

```
Profiling 流程：

1. OAP 下发 Profiling 任务
   ├── 目标服务：order-service
   ├── 持续时间：5 分钟
   ├── 采样间隔：10ms
   └── 监控类型：On-CPU

2. Agent 执行 Profiling
   ├── 每 10ms 采样一次目标线程的调用栈
   ├── 聚合相同调用栈的计数
   └── 5 分钟后上报结果

3. OAP 聚合和存储
   ├── 合并多个 Agent 的采样数据
   └── 生成火焰图

4. UI 展示
   └── 火焰图 + 热点分析
```

#### 2.2 与 JProfiler/Arthas 的对比

| 对比维度 | SkyWalking Profiling | JProfiler | Arthas |
|---------|---------------------|-----------|--------|
| 侵入性 | 极低（Agent 自动） | 需 Attach | 需 Attach |
| 性能开销 | < 1% CPU | 5-10% CPU | 1-3% CPU |
| 适用场景 | 生产环境持续监控 | 开发/测试环境 | 在线诊断 |
| 自动化 | ✅ 可配置自动触发 | ❌ 手动触发 | ❌ 手动触发 |
| 火焰图 | ✅ 内置 | ✅ 支持 | ❌ 不支持 |

### 3. eBPF Rover（无侵入内核级监控）

#### 3.1 什么是 eBPF？

eBPF（extended Berkeley Packet Filter）是 Linux 内核的一项技术，允许在**内核空间**安全地运行沙箱程序，无需修改内核源码或加载内核模块。

#### 3.2 Rover 的监控能力

```
Rover 监控能力：

┌──────────────────────────────────────────────────────────────┐
│  Rover（eBPF 探针）                                          │
│                                                              │
│  ├── 网络监控                                                │
│  │   ├── TCP 连接追踪（建立/关闭/重传）                       │
│  │   ├── HTTP/gRPC 协议分析                                  │
│  │   ├── 网络延迟和吞吐量                                    │
│  │   └── TLS 握手耗时                                        │
│  │                                                           │
│  ├── 进程监控                                                │
│  │   ├── CPU 使用率（按进程/线程）                            │
│  │   ├── 内存使用（RSS/VSS）                                 │
│  │   └── 磁盘 I/O 和文件操作                                 │
│  │                                                           │
│  ├── 系统监控                                                │
│  │   ├── 系统调用追踪                                        │
│  │   ├── DNS 查询监控                                        │
│  │   └── 文件系统事件                                        │
│  │                                                           │
│  └── 特点                                                    │
│      ├── 零侵入：无需修改应用代码                             │
│      ├── 零 Agent：无需安装任何 Agent                         │
│      └── 内核级精度：数据直接从内核获取                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4. 事件系统（Event）

#### 4.1 Event 是什么？

SkyWalking Event 是**离散的事件通知**，用于标记系统中发生的重要事件：

```
Event 类型：
├── 部署事件：应用发布/回滚
├── 配置变更事件：配置中心更新
├── 扩缩容事件：K8s Pod 创建/销毁
├── 故障事件：节点宕机/网络中断
└── 自定义事件：业务事件（如大促开始）
```

#### 4.2 Event 的使用

```bash
# 通过 swctl 发送事件
swctl event report \
  --service=order-service \
  --name="Deployment" \
  --message="v2.0.0 deployed" \
  --type=Normal
```

**Event 的作用**：
- 在指标图上标记事件发生时间点
- 帮助关联"部署"和"指标异常"的因果关系
- 例如：部署后 P99 延迟上升 -> 可视化标注部署时间点

### 5. 命名空间与多租户

#### 5.1 Namespace 隔离

```yaml
# 不同环境使用不同的 namespace
# 环境 A：开发环境
storage:
  elasticsearch:
    namespace: dev

# 环境 B：生产环境
storage:
  elasticsearch:
    namespace: prod
```

**Namespace 的作用**：
- 数据隔离：不同 namespace 的数据存储在不同的 ES 索引中
- 多租户：不同团队使用不同的 namespace
- 环境隔离：dev/staging/prod 数据互不干扰

### 6. TLS/mTLS 安全传输

```yaml
# Agent 端：启用 TLS
agent:
  # gRPC TLS
  grpc_channel:
    ssl_trusted_ca_path: /path/to/ca.crt

# OAP 端：启用 mTLS
receiver-sharing-server:
  default:
    gRPCSslEnabled: ${SW_RECEIVER_SHARING_GRPC_SSL_ENABLED:true}
    gRPCSslKeyPath: ${SW_RECEIVER_SHARING_GRPC_SSL_KEY_PATH:/path/to/server.key}
    gRPCSslCertChainPath: ${SW_RECEIVER_SHARING_GRPC_SSL_CERT_CHAIN_PATH:/path/to/server.crt}
    gRPCSslTrustedCAPath: ${SW_RECEIVER_SHARING_GRPC_SSL_TRUSTED_CA_PATH:/path/to/ca.crt}
```

### 7. CLI 工具（swctl）

```bash
# 安装 swctl
# 下载：https://skywalking.apache.org/downloads/

# 查看服务列表
swctl service list

# 查看服务指标
swctl metrics linear --name=service_cpm --service=order-service

# 查看拓扑
swctl dependency service --name=order-service

# 查看 Trace
swctl trace ls --service=order-service

# 触发 Profiling
swctl profile create --service=order-service --duration=5
```

---

## 常见面试题

### Q1: 采样策略有哪些？如何选择？

采样分两端，**面试时务必先点出"Agent 前置 + OAP 后置"两层，这是加分项**：

| 采样层 | 策略 | 配置 | 适用场景 |
|--------|------|------|---------|
| **Agent 前置** | 全采样 | `sample_n_per_3_secs=-1` | 低流量（< 100 QPS） |
| **Agent 前置** | 速率限制 | `sample_n_per_3_secs=N` | 高流量，省带宽 |
| **Agent 前置** | 动态下发 | `trace.sample_rate`（配置中心） | 需热更新、不重启 |
| **OAP 后置** | 慢阈值强制 | `duration` | 慢请求排障必采 |
| **OAP 后置** | 错误段强制 | `forceSampleErrorSegment=true` | 错误请求排障必采（默认开） |
| **OAP 后置** | 哈希采样率 | `rate` | 存储成本控制 |
| **不采样** | 只保留指标 | `sample_n_per_3_secs=0` | 极限流量 |

**最佳实践**：
- 低流量：Agent 全采样
- 高流量：Agent 速率限制（省带宽）+ OAP 错误/慢请求强制采样（保关键）+ OAP 哈希采样率（控存储）
- 极限流量：Agent 不采样，只保留指标

**常见误区**：错误请求和慢请求的强制采样是 **OAP 端**行为，不是 Agent 端。Agent 端做不到（决策在请求开始时，此时还不知道是否出错/耗时）。

### Q2: SkyWalking Profiling 和 JProfiler 有什么区别？

核心区别：**SkyWalking Profiling 适合生产环境，JProfiler 适合开发调试**。

SkyWalking Profiling 使用 On-CPU 采样（< 1% CPU 开销），可以持续运行在生产环境。JProfiler 使用 Instrumentation（5-10% CPU 开销），适合开发环境深度分析。

### Q3: eBPF Rover 解决了什么问题？

**解决的问题**：对于无法安装 Agent 的场景（如 C++ 遗留系统、第三方服务），Rover 通过 eBPF 从内核层面采集监控数据，实现**真正的零侵入**。

**与 Agent 的关系**：Agent 看到的是应用内部（方法调用），Rover 看到的是网络和系统层面（TCP 连接、CPU、内存）。两者互补，Agent + Rover = 完整覆盖。

---

## 延伸阅读

- SkyWalking Profiling：[https://skywalking.apache.org/docs/main/latest/en/concepts-and-designs/profiling/](https://skywalking.apache.org/docs/main/latest/en/concepts-and-designs/profiling/)
- SkyWalking Rover：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/rover/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/rover/)
- swctl CLI：[https://skywalking.apache.org/docs/main/latest/en/setup/cli/](https://skywalking.apache.org/docs/main/latest/en/setup/cli/)
- Trace Sampling 文档：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/trace-sampling/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/trace-sampling/)
- Configuration Discovery：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/configuration-discovery/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/configuration-discovery/)

---

## 资料勘误与重点提醒

### 1. 错误/慢请求强制采样在 OAP 端，不在 Agent 端（高频误区）

**资料常见说法**："错误请求、慢请求在 Agent 端强制全采样，不受限流影响。"

**源码事实**：错误。Agent 端采样决策发生在 `ContextManagerExtendService.createTraceContext()`，即**请求刚开始的那一刻**。此时：
- 既不知道是否抛异常（`errorOccurred` 在异常发生后才置位）
- 也不知道耗时多久（duration 在请求结束才确定）

所以 Agent 端**机制上不可能**基于 error/slow 反向强制采样。被丢弃的 Trace 落到 `IgnoredTracerContext`（所有 span 变 noop），后续即便出错也救不回来。

错误/慢请求的强制保留是 **OAP 端后置**行为，在 `SegmentAnalysisListener.parseSegment()` 实现，决策优先级：**慢阈值 > 错误段 > 采样率哈希丢弃**。

### 2. `traceSampleRateWatcher` / `SW_SLOW_TRACE_SEGMENT_THRESHOLD` 已废弃（8.6.0 起）

这两个旧名字在现行源码中已不存在，被合并为统一的 `traceSamplingPolicy`：

| 旧（8.6.0 前） | 新（现行） |
|----------------|-----------|
| `traceSampleRateWatcher` 类 | `TraceSamplingPolicyWatcher` 类 |
| `SW_SLOW_TRACE_SEGMENT_THRESHOLD` 环境变量 | `SW_TRACE_SAMPLING_POLICY_SETTINGS_FILE`（静态文件）/ `agent-analyzer.default.traceSamplingPolicy`（动态 key） |
| 单独的 sampleRate + slowThreshold | 一个 yaml 策略文件，含 `rate` + `duration` |

`TraceSamplingPolicyWatcher` **同时管**慢阈值（`duration`）和采样率（`rate`），优先级 **慢阈值 > 采样率**。它和 receiver 的 `AgentConfigurationsWatcher`（下发给 Agent）是两套独立链路，别混淆。

### 3. `forceSampled()` 不是"错误强制采样"

`SamplingService.forceSampled()` 的**唯一用途是跨进程传播**：上游已采样的 Trace，下游必须延续采样，避免链路在跨服务调用时断裂。全仓库唯一调用点是 `ContextManager.createEntrySpan`（从 ContextCarrier 还原上下文时）。与"错误请求"无关。

### 4. 动态下发的正确类名与链路

| 错误名字（资料常见） | 正确名字（源码） |
|---------------------|----------------|
| `ConfigServer` | Agent: `ConfigurationDiscoveryService`；OAP: `ConfigurationDiscoveryServiceHandler` |
| `fetchConfigItems` | `fetchConfigurations`（gRPC 方法） |
| "读远程文件" | gRPC 拉取结构化 KV 列表，非文件 |
| Agent 直连配置中心 | Agent 只连 OAP，OAP 转发配置中心 |

### 5. 两套采样的本质区别

| 维度 | Agent 前置采样 | OAP 后置采样 |
|------|--------------|-------------|
| 决策时机 | 请求开始时 | 数据到 OAP 后 |
| 算法 | AtomicInteger 计数器 CAS | `traceId.hashCode()%10000` 哈希 |
| 能省什么 | **网络带宽** | **存储空间** |
| 能否基于 error/slow | ❌ 不能（时序上拿不到） | ✅ 能 |
| 实现类 | `SamplingService` | `SegmentAnalysisListener` + `TraceSamplingPolicyWatcher` |

### 6. 其他 *LatencyThresholdsAndWatcher 不影响 Trace 采样

`agent-analyzer` 模块下还有 `DBLatencyThresholdsAndWatcher`、`CacheReadLatencyThresholdsAndWatcher`、`CacheWriteLatencyThresholdsAndWatcher` 等，名字里带 "Threshold" 容易被误以为和采样相关。它们**只影响是否生成慢操作记录**（如慢 SQL 列表、慢缓存操作），**不影响 Trace segment 是否被丢弃**，别被名字误导。

### 7. 面试加分点

讲采样时按这个层次展开，体现深度：
1. 先讲**为什么有两层**（信息时序差：决策时 vs 结束后）
2. 再讲**各自省什么**（Agent 省带宽、OAP 省存储）
3. 然后讲**动态下发链路**（配置中心 -> OAP -> Agent，每 20s pull）
4. 最后点出**常见误区**（错误/慢强制采样是 OAP 端，Agent 做不到）

> 源码依据：本节勘误基于对 `apache/skywalking`（OAP server）与 `apache/skywalking-java`（Java Agent）源码的查证。关键类：Agent 端 `SamplingService`/`ConfigurationDiscoveryService`/`SamplingRateWatcher`；OAP 端 `ConfigurationDiscoveryServiceHandler`/`AgentConfigurationsWatcher`/`SegmentAnalysisListener`/`TraceSamplingPolicyWatcher`。
