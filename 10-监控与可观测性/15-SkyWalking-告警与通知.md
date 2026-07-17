# 15 - SkyWalking 告警与通知

## 核心概念

### 1. 告警架构全景

```
┌──────────────────────────────────────────────────────────────────┐
│                    告警引擎架构                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OAP 告警模块 (server-alarm-plugin)                        │   │
│  │                                                           │   │
│  │  ① 告警规则定义 (alarm-settings.yml)                       │   │
│  │     ├── 规则名称 + 指标名称 + 阈值 + 周期 + 静默期          │   │
│  │     └── 支持动态配置中心（Apollo/Nacos/ZK）热更新            │   │
│  │                                                           │   │
│  │  ② 告警规则引擎 (AlarmRulesEngine)                         │   │
│  │     ├── 定时检查（每 1 分钟）                               │   │
│  │     ├── 指标查询 → 阈值判定 → 触发告警                       │   │
│  │     └── 告警生命周期管理（触发 → 持续 → 恢复）              │   │
│  │                                                           │   │
│  │  ③ 告警钩子 (AlarmHook)                                    │   │
│  │     ├── Webhook（HTTP POST JSON）                          │   │
│  │     ├── gRPC Hook（自定义 gRPC 服务）                       │   │
│  │     └── Prometheus AlertManager（转接）                     │   │
│  │                                                           │   │
│  │  ④ 通知渠道（通过 Webhook 实现）                           │   │
│  │     ├── 钉钉机器人                                           │   │
│  │     ├── 企业微信机器人                                       │   │
│  │     ├── 飞书机器人                                           │   │
│  │     ├── Slack                                              │   │
│  │     └── 邮件（通过外部服务）                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 2. 告警规则配置（alarm-settings.yml）

```yaml
# config/alarm-settings.yml
rules:
  # 服务平均响应时间告警
  service_resp_time_rule:
    # 指标名称（对应 OAL 定义的指标）
    metrics-name: service_resp_time
    # 操作符：>, <, >=, <=, ==
    op: ">"
    # 阈值（毫秒）
    threshold: 1000
    # 持续周期数（告警触发前需要连续满足条件的周期数）
    period: 3
    # 每个周期的时间（分钟）
    count: 1
    # 静默期（分钟，告警恢复后 N 分钟内不再触发）
    silence-period: 10
    # 告警消息模板
    message: "服务 {name} 的响应时间超过 {threshold}ms，当前值: {value}ms"

  # 服务成功率告警
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 95
    period: 2
    count: 1
    silence-period: 5
    message: "服务 {name} 的成功率低于 {threshold}%，当前值: {value}%"

  # 端点 P99 延迟告警
  endpoint_p99_rule:
    metrics-name: endpoint_p99
    op: ">"
    threshold: 5000
    period: 3
    count: 1
    silence-period: 10
    message: "端点 {name} 的 P99 延迟超过 {threshold}ms，当前值: {value}ms"

  # 数据库慢查询告警
  database_slow_rule:
    metrics-name: database_access_resp_time
    op: ">"
    threshold: 200
    period: 3
    count: 1
    silence-period: 10
    message: "数据库 {name} 平均响应时间超过 {threshold}ms"

  # 服务实例 JVM 内存告警
  instance_jvm_memory_rule:
    metrics-name: instance_jvm_heap_used
    op: ">"
    threshold: 80
    period: 2
    count: 1
    silence-period: 10
    message: "服务实例 {name} 堆内存使用率超过 {threshold}%"

# Webhook 配置
webhooks:
  - url: http://your-webhook-server/alarm
    # 钉钉机器人示例
    # url: https://oapi.dingtalk.com/robot/send?access_token=xxx

# gRPC Hook 配置
grpchook:
  target: your-grpc-server:port
```

### 3. 告警生命周期

```
┌──────────────────────────────────────────────────────────────────┐
│                    告警生命周期                                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   [正常] ────── 指标超过阈值 ──────→ [触发]                        │
│     ↑                                  │                          │
│     │                                  │ 发送告警通知               │
│     │                                  ▼                          │
│     │                              [持续中]                        │
│     │                                  │                          │
│     │                                  │ 指标回到正常               │
│     │                                  ▼                          │
│     └────────── 发送恢复通知 ─────── [恢复]                         │
│                                         │                         │
│                                    静默期 N 分钟                    │
│                                         │                         │
│                                         ▼                         │
│                                     [正常]                         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**关键参数说明**：

| 参数 | 含义 | 示例 |
|------|------|------|
| `period` | 连续满足条件的周期数 | 3（连续 3 个周期超标才触发） |
| `count` | 每个周期的时长（分钟） | 1（每 1 分钟检查一次） |
| `silence-period` | 静默期（分钟） | 10（告警恢复后 10 分钟内不再触发） |

**为什么需要 period > 1？**
- 防止瞬时抖动导致的误告警
- 例如：`period: 3, count: 1` 表示连续 3 分钟超标才触发告警

### 4. 告警消息格式

#### 4.1 Webhook 消息格式

```json
[
  {
    "scopeId": 1,
    "scope": "SERVICE",
    "name": "order-service",
    "id0": "2",
    "id1": "",
    "ruleName": "service_resp_time_rule",
    "alarmMessage": "服务 order-service 的响应时间超过 1000ms，当前值: 1500ms",
    "startTime": 1690000000000,
    "tags": []
  }
]
```

#### 4.2 钉钉机器人通知示例

```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "SkyWalking 告警",
    "text": "## SkyWalking 告警通知\n\n"
      + "**告警规则**：service_resp_time_rule\n\n"
      + "**服务名称**：order-service\n\n"
      + "**告警信息**：服务 order-service 的响应时间超过 1000ms，当前值: 1500ms\n\n"
      + "**告警时间**：2024-07-17 10:05:00\n\n"
      + "[查看详情](http://skywalking-ui:8080)"
  }
}
```

### 5. 动态配置（Configuration Discovery）

#### 5.1 什么是动态配置？

动态配置允许在**不重启 OAP 的情况下**，动态修改告警规则、采样策略等配置。

#### 5.2 支持的配置中心

| 配置中心 | 配置方式 | 适用场景 |
|---------|---------|---------|
| **Apollo** | 携程开源配置中心 | 大型企业 |
| **Nacos** | 阿里开源配置中心 | 阿里云生态 |
| **ZooKeeper** | Apache 分布式协调 | 传统架构 |
| **Consul** | HashiCorp 服务网格 | 云原生 |
| **etcd** | CNCF 分布式 KV | Kubernetes 生态 |

#### 5.3 可动态配置的项

```yaml
# 动态配置项
configuration:
  # 告警规则
  alarm-settings:
    # 动态加载 alarm-settings.yml

  # 采样策略
  trace-sampling-settings:
    # 动态调整采样率

  # 慢阈值
  slow-threshold:
    # 动态调整慢 SQL / 慢 HTTP 阈值

  # 日志分析规则
  lal-settings:
    # 动态调整 LAL 规则
```

### 6. 告警与 Prometheus AlertManager 集成

```
SkyWalking 告警 → Prometheus AlertManager

1. OAP 检测到告警
2. 通过 Webhook 发送到 AlertManager
3. AlertManager 处理告警（分组、抑制、静默）
4. AlertManager 发送通知到各种渠道
   ├── Email
   ├── PagerDuty
   ├── Slack
   └── Webhook（自定义）
```

---

## 常见面试题

### Q1: SkyWalking 告警引擎的工作原理是什么？

1. **规则定义**：在 `alarm-settings.yml` 中定义告警规则（指标、阈值、周期、静默期）
2. **定时检查**：每 N 分钟检查一次指标是否超过阈值
3. **触发条件**：连续 `period` 个周期超标 → 触发告警
4. **生命周期**：触发 → 持续 → 恢复
5. **通知**：通过 Webhook/gRPC Hook 发送告警通知

### Q2: 为什么需要 period（连续周期数）参数？

防止 **瞬时抖动导致的误告警**。例如：
- 某服务因为 GC 导致短暂延迟升高，1 分钟后恢复正常
- 如果 `period=1`，这次抖动会触发告警
- 如果 `period=3`，需要连续 3 分钟超标才触发，过滤掉瞬时抖动

### Q3: 动态配置有什么好处？支持哪些配置中心？

**好处**：
- 不重启 OAP 即可修改告警规则
- 紧急情况下快速调整采样率
- 配置变更可追溯（配置中心版本管理）

**支持的配置中心**：Apollo、Nacos、ZooKeeper、Consul、etcd

### Q4: 如何实现告警的静默期？

通过 `silence-period` 参数。告警恢复后，在 N 分钟内**不再触发同一规则的告警**，避免"告警风暴"（频繁触发和恢复）。

示例：`silence-period: 10` → 告警恢复后，10 分钟内即使指标再次超标也不触发告警。

---

## 延伸阅读

- SkyWalking 告警配置文档：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-alarm/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/backend-alarm/)
- 动态配置文档：[https://skywalking.apache.org/docs/main/latest/en/setup/backend/dynamic-config/](https://skywalking.apache.org/docs/main/latest/en/setup/backend/dynamic-config/)