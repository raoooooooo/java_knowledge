# 22 - SkyWalking 版本演进与生态

## 核心概念

### 1. 版本演进历史

#### 1.1 v1-v5（2015-2017）：早期探索

| 版本 | 年份 | 关键变化 |
|------|------|---------|
| v1 | 2015 | 吴晟开源，纯 Java Agent + 内存存储 |
| v2 | 2016 | 引入后端存储概念 |
| v3 | 2016 | 支持 MySQL 存储 |
| v5 | 2017 | 加入 Apache 孵化器，重构通信协议 |

#### 1.2 v6（2018-2019）：OAP 架构诞生

```
v6 重大变化：
├── OAP（Observability Analysis Platform）架构引入
├── 模块化设计（ModuleDefine + ModuleProvider）
├── gRPC 通信协议
├── 引入 OAL 引擎
├── 支持 Elasticsearch 存储
└── 从孵化器毕业成为 Apache 顶级项目（2019.4）
```

#### 1.3 v7（2019-2020）：生态扩展

```
v7 重大变化：
├── 支持多种语言 Agent（Python、Node.js、Go）
├── Service Mesh 集成（Istio/Envoy）
├── 浏览器监控（Browser Agent）
├── Profiling 性能剖析
└── 动态配置中心
```

#### 1.4 v8（2020-2022）：语言引擎成熟

```
v8 重大变化：
├── MAL（Meter Analysis Language）引入
├── LAL（Log Analysis Language）引入
├── Prometheus Fetcher
├── Kafka Fetcher
├── 事件系统（Event）
├── 告警引擎重构
└── Meter 指标体系完善
```

#### 1.5 v9（2022-2024）：现代化与标准化

```
v9 重大变化：
├── UI 重构（RocketBot → 现代化 UI）
├── 原生 OpenTelemetry 支持（OTLP Receiver）
├── 移除 H2 集群模式（生产环境只用 ES/BanyanDB/MySQL）
├── MQE（Metrics Query Engine）引入
├── eBPF Rover 引入
├── GenAI 监控
├── Layer 分层重构
├── Hierarchy 层级关系分析
└── Helm Charts 简化部署
```

#### 1.6 v10（2024+）：自研数据库时代

```
v10 重大变化：
├── BanyanDB 成为默认存储引擎
├── 分层拓扑（Layer Topology）
├── 新的 Agent 协议（减少带宽 50%+）
├── Satellite 边车网关
├── SWCK Operator 生产可用
├── Helm Charts 重大简化
└── 性能大幅提升（写入性能 3x，查询性能 2x）
```

### 2. 社区与生态

#### 2.1 Apache 基金会治理

SkyWalking 在 Apache 基金会的治理下运作：

```
Apache 治理结构：
├── Board（董事会）：Apache 软件基金会最高决策机构
├── PMC（Project Management Committee）：项目管理委员会
│   ├── PMC Chair：项目主席
│   └── PMC Members：PMC 成员
├── Committers：提交者（有代码写入权限）
└── Contributors：贡献者（提交 PR 的任何人）
```

**如何成为 Committer**：
1. 持续贡献代码/文档/社区
2. 由现有 PMC 成员提名
3. PMC 投票通过

#### 2.2 生态工具链

| 工具 | 作用 | 集成方式 |
|------|------|---------|
| **Grafana** | 统一仪表盘展示 | SkyWalking DataSource 插件 |
| **Arthas** | 在线诊断 | 与 SkyWalking Profiling 互补 |
| **Prometheus** | 基础设施指标 | Prometheus Fetcher 拉取 + Exporter 暴露 |
| **AlertManager** | 告警管理 | SkyWalking Webhook → AlertManager |
| **Loki** | 日志聚合 | 通过 Grafana 统一展示 |
| **Apache APISIX** | API 网关 | 内置 SkyWalking 插件 |
| **Apache ShardingSphere** | 分布式数据库 | 内置 SkyWalking Agent |
| **Soul Gateway** | API 网关 | 内置 SkyWalking 插件 |

#### 2.3 知名用户

- **华为**：大规模微服务集群全链路监控
- **腾讯**：微信支付、腾讯云内部使用
- **阿里巴巴**：部分业务线使用
- **滴滴**：核心服务监控
- **字节跳动**：部分服务监控
- **永辉超市**：全链路追踪
- **中国电信**：云平台监控

### 3. 技术趋势

#### 3.1 从 APM 到可观测性

```
传统 APM（只关注应用性能）
  → 可观测性（Traces + Metrics + Logs 统一）

SkyWalking 的演进：
├── v6-7：APM（Trace + 基础指标）
├── v8：+ Metrics（MAL + Meter）
├── v9：+ Logs（LAL + 日志关联）+ OpenTelemetry
└── v10：+ eBPF（Rover）+ 自研数据库
```

#### 3.2 OpenTelemetry 融合

SkyWalking 正在从"独立的 APM 系统"向"OTel 生态的可观测性后端"演进：

```
短期（v9-v10）：sw8 和 OTLP 双协议共存
中期（v10-v11）：OTLP 成为主要协议，sw8 逐步废弃
长期（v11+）：SkyWalking 成为 OTel 生态的核心分析平台
```

#### 3.3 自研数据库方向

BanyanDB 的推出标志着 SkyWalking 从"依赖外部存储"到"自研 APM 专用数据库"的战略转变：

- **v9 之前**：默认 ES（运维成本高，性能一般）
- **v10**：默认 BanyanDB（自研，APM 场景优化，零运维）
- **未来**：BanyanDB 可能成为独立的 APM 数据库产品

---

## 常见面试题

### Q1: SkyWalking v8 → v9 → v10 有哪些关键变化？

| 版本 | 关键变化 | 面试关注点 |
|------|---------|-----------|
| v8 → v9 | OTel 支持、UI 重构、MQE、eBPF Rover、移除 H2 集群 | 现代化、标准化 |
| v9 → v10 | BanyanDB 默认、分层拓扑、新协议、性能提升 | 自研数据库、性能优化 |

**面试重点**：
- v9：OpenTelemetry 集成是最大的变化，标志着 SkyWalking 从"自有协议"向"行业标准"转变
- v10：BanyanDB 默认是最大的变化，标志着 SkyWalking 从"依赖 ES"到"自研数据库"的转变

### Q2: SkyWalking 社区的治理模式是怎样的？

Apache 基金会治理，PMC 管理项目，Committers 有代码写入权限。任何人都可以成为 Contributor（提交 PR），持续贡献可以成为 Committer。

**特点**：
- 社区驱动，不是单一公司控制
- 决策透明（邮件列表公开讨论）
- 版本发布遵循 Apache Release 流程

### Q3: SkyWalking 未来的技术趋势是什么？

1. **OpenTelemetry 深度融合**：OTLP 成为主要协议
2. **BanyanDB 自研数据库**：降低存储成本，提升性能
3. **eBPF 无侵入监控**：覆盖更多无法安装 Agent 的场景
4. **AI 辅助分析**：GenAI 监控、智能告警
5. **云原生深化**：Kubernetes Operator、Helm Charts 简化部署

---

## 延伸阅读

- SkyWalking 官方博客：[https://skywalking.apache.org/blog/](https://skywalking.apache.org/blog/)
- Apache 基金会治理：[https://www.apache.org/foundation/governance/](https://www.apache.org/foundation/governance/)
- SkyWalking GitHub：[https://github.com/apache/skywalking](https://github.com/apache/skywalking)
- 版本发布说明：[https://skywalking.apache.org/downloads/](https://skywalking.apache.org/downloads/)