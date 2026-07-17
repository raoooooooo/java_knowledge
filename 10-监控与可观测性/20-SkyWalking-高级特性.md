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

| 策略 | 配置 | 说明 |
|------|------|------|
| **全采样** | `agent.sample_n_per_3_secs=-1` | 所有请求都记录，适合低流量 |
| **不采样** | `agent.sample_n_per_3_secs=0` | 不记录任何 Trace，只保留指标 |
| **速率限制** | `agent.sample_n_per_3_secs=N` | 每 3 秒最多 N 个 Trace |
| **强制采样** | — | 错误请求自动全采样（不受限流影响） |
| **慢请求采样** | OAP 端配置 | 超过慢阈值的请求强制采样 |

#### 1.3 采样策略配置

```properties
# Agent 端：采样率控制
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}

# 强制采样：以下情况始终采样
# 1. 错误请求（span.isError = true）
# 2. 慢请求（超过 OAP 端配置的慢阈值）
```

```yaml
# OAP 端：慢请求强制采样
agent-analyzer:
  default:
    traceSampleRateWatcher:
      # 慢请求阈值（毫秒）
      slowTraceSegmentThreshold: ${SW_SLOW_TRACE_SEGMENT_THRESHOLD:-1}
```

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
- 例如：部署后 P99 延迟上升 → 可视化标注部署时间点

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

| 采样策略 | 适用场景 | 配置 |
|---------|---------|------|
| 全采样 | 低流量（< 100 QPS） | `sample_n_per_3_secs=-1` |
| 速率限制 | 高流量（> 100 QPS） | `sample_n_per_3_secs=N` |
| 强制采样 | 错误/慢请求 | Agent 自动处理 |
| 不采样 | 只保留指标 | `sample_n_per_3_secs=0` |

**最佳实践**：
- 低流量：全采样
- 高流量：速率限制 + 错误强制采样 + 慢请求强制采样
- 极限流量：不采样 Trace，只保留指标

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