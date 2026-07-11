# ClickHouse 实战与运维

---

## 一、日志存储场景实战

### 1.1 日志场景特点

| 特点 | 挑战 |
|------|------|
| 写入量巨大 | 每天 TB 级，每秒百万行 |
| 保留时间长 | 30天-1年 |
| 查询模式固定 | 按时间 + 服务/主机 + 关键词 |
| 写入一次，查询多次 | 不需要更新 |

---

### 1.2 日志表设计最佳实践

#### 标准版设计

```sql
CREATE TABLE logs.logs_local ON CLUSTER prod_cluster
(
    `timestamp` DateTime64(3) COMMENT '日志时间(毫秒)',
    `dt` Date MATERIALIZED toDate(timestamp) COMMENT '日期(用于分区)',
    `cluster` LowCardinality(String) COMMENT 'K8s集群',
    `namespace` LowCardinality(String) COMMENT '命名空间',
    `app` LowCardinality(String) COMMENT '应用名',
    `pod_name` String COMMENT 'Pod名',
    `hostname` String COMMENT '主机名',
    `level` LowCardinality(String) COMMENT '日志级别',
    `trace_id` String COMMENT '链路ID',
    `message` String COMMENT '日志内容',
    `labels` Map(String, String) COMMENT '自定义标签'
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (cluster, namespace, app, timestamp)
TTL timestamp + INTERVAL 30 DAY
SETTINGS 
    index_granularity = 8192,
    min_bytes_for_wide_part = '10M';
```

#### 设计要点解析

| 设计 | 为什么这么做 |
|------|------------|
| `DateTime64(3)` | 精确到毫秒，比 DateTime 更精确 |
| `LowCardinality(String)` | 低基数字符串自动转字典，压缩比提升 5-10 倍 |
| `ORDER BY (cluster, namespace, app, timestamp)` | 查询都是按服务查，把等值查询列放前面 |
| `TTL 30天` | 自动清理过期日志，不用手动删 |
| `min_bytes_for_wide_part` | 小 Part 用紧凑格式，减少文件数 |

---

### 1.3 Kafka 实时接入方案

#### 架构图

```
应用输出日志 → Filebeat → Kafka → ClickHouse Kafka引擎 → 物化视图 → 日志表
```

#### 步骤1：创建 Kafka 消费表

```sql
CREATE TABLE logs.kafka_logs_raw ON CLUSTER prod_cluster
(
    `raw` String
)
ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka-01:9092,kafka-02:9092,kafka-03:9092',
    kafka_topic_list = 'app-logs',
    kafka_group_name = 'clickhouse-logs-consumer',
    kafka_format = 'JSONAsString',
    kafka_num_consumers = 4,  -- 每个分片4个消费者
    kafka_max_block_size = 65536;
```

#### 步骤2：创建物化视图做解析

```sql
CREATE MATERIALIZED VIEW logs.logs_mv ON CLUSTER prod_cluster
TO logs.logs_local
AS
SELECT
    toDateTime64(JSONExtractString(raw, 'timestamp'), 3) AS timestamp,
    JSONExtractString(raw, 'cluster') AS cluster,
    JSONExtractString(raw, 'namespace') AS namespace,
    JSONExtractString(raw, 'app') AS app,
    JSONExtractString(raw, 'pod_name') AS pod_name,
    JSONExtractString(raw, 'hostname') AS hostname,
    JSONExtractString(raw, 'level') AS level,
    JSONExtractString(raw, 'trace_id') AS trace_id,
    JSONExtractString(raw, 'message') AS message,
    cast(JSONExtractKeysAndValues(raw, 'labels', 'String'), 'Map(String, String)') AS labels
FROM logs.kafka_logs_raw;
```

> ✅ **优势：** 完全实时接入，秒级延迟，不用 Flink/Spark 额外组件！

---

### 1.4 日志查询优化

#### 优化1：关键词搜索加速

```sql
-- 建布隆过滤器索引，加速 LIKE 查询
CREATE INDEX idx_message ON logs.logs_local(message)
TYPE ngrambf_v1(3, 256, 0, 8192)
GRANULARITY 4;

-- 建完后需要刷新
ALTER TABLE logs.logs_local MATERIALIZE INDEX idx_message;
```

**效果：** `message LIKE '%error%'` 查询速度提升 **10-50倍**

#### 优化2：常用字段建跳数索引

```sql
-- trace_id 建布隆索引，加速链路追踪查询
CREATE INDEX idx_trace_id ON logs.logs_local(trace_id)
TYPE bloom_filter()
GRANULARITY 1;
```

---

### 1.5 日志分级存储

#### 冷热分离架构

```
热数据（7天内）→ SSD 高性能盘 → 快速查询
温数据（7-30天）→ SATA 大容量盘 → 偶尔查询
冷数据（30天以上）→ 对象存储S3 → 归档（可选）
```

#### 实现方式：TTL + 移动分区

```sql
-- 7天后移动到 SATA 盘（需要配置存储策略）
ALTER TABLE logs.logs_local 
MODIFY TTL timestamp + INTERVAL 7 DAY TO VOLUME 'sata',
           timestamp + INTERVAL 30 DAY DELETE;
```

---

## 二、可观测性指标存储实战

### 2.1 指标场景特点

| 对比项 | 日志 | 指标 |
|--------|------|------|
| 数据量 | 极大（文本） | 中等（数值） |
| 查询类型 | 关键词搜索 | 聚合统计（avg/max/p95） |
| 保留时间 | 30天 | 1年以上 |
| 写入频率 | 秒级 | 10秒-1分钟 |

---

### 2.2 Prometheus + ClickHouse 存储方案

#### 为什么用 ClickHouse 存指标？

- Prometheus 本地存储：单机容量有限，长期存储难
- M3DB/Thanos：复杂，运维成本高
- **ClickHouse：** 压缩比高（10:1），查询快，运维简单

#### 指标表设计

```sql
CREATE TABLE metrics.metrics_local ON CLUSTER prod_cluster
(
    `timestamp` DateTime CODEC(DoubleDelta) COMMENT '指标时间',
    `name` LowCardinality(String) COMMENT '指标名',
    `value` Float64 CODEC(Gorilla) COMMENT '指标值',
    -- 标签（维度）
    `cluster` LowCardinality(String),
    `namespace` LowCardinality(String),
    `service` LowCardinality(String),
    `instance` String,
    `job` LowCardinality(String),
    `additional_labels` Map(LowCardinality(String), String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (name, cluster, namespace, service, timestamp)
TTL timestamp + INTERVAL 365 DAY
SETTINGS index_granularity = 8192;
```

#### 特殊压缩算法

| 压缩算法 | 适用列 | 压缩比 |
|---------|-------|-------|
| `DoubleDelta` | 时间戳 | 20:1 |
| `Gorilla` | 监控数值 | 12:1 |
| `LZ4` | 默认通用 | 5:1 |

> ✅ **效果：** 相同数据比 Prometheus 少占 **3-5倍** 空间！

---

### 2.3 指标预聚合（物化视图）

#### 问题：原始指标秒级数据，查1个月的趋势太慢？

**解决方案：** 用物化视图做分钟级聚合

```sql
-- 分钟级聚合视图
CREATE MATERIALIZED VIEW metrics.metrics_1m ON CLUSTER prod_cluster
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (name, cluster, service, ts)
AS
SELECT
    toStartOfMinute(timestamp) AS ts,
    name,
    cluster,
    namespace,
    service,
    count() AS sample_count,
    sumState(value) AS sum_val,
    minState(value) AS min_val,
    maxState(value) AS max_val,
    avgState(value) AS avg_val,
    quantileState(0.5)(value) AS p50_val,
    quantileState(0.95)(value) AS p95_val,
    quantileState(0.99)(value) AS p99_val
FROM metrics.metrics_local
GROUP BY ts, name, cluster, namespace, service;
```

#### 查询聚合视图

```sql
-- 查过去24小时某服务的P95延迟
SELECT
    ts,
    quantileMerge(0.95)(p95_val) AS p95
FROM metrics.metrics_1m
WHERE name = 'http_request_duration_seconds'
  AND service = 'user-service'
  AND ts >= now() - INTERVAL 24 HOUR
GROUP BY ts
ORDER BY ts;
```

> ✅ **效果：** 查询速度提升 **100倍**，因为数据只有原始的 1%

---

### 2.4 多层聚合架构

```
原始指标（10秒粒度）
      ↓ MV聚合
分钟级聚合（6倍压缩）
      ↓ MV聚合
小时级聚合（60倍压缩）
      ↓ MV聚合
天级聚合（24倍压缩）
```

**查询路由：**
- 查最近1小时 → 查原始表
- 查最近7天 → 查分钟级聚合
- 查最近30天 → 查小时级聚合
- 查年度趋势 → 查天级聚合

---

## 三、常见问题排查与处理

### 3.1 高频问题 TOP 10

---

#### 问题1：Too many parts (300). Merges are processing significantly slower than inserts.

**症状：** 写入报错，查询变慢

**原因：** 写入太频繁，产生太多小 Part，合并线程追不上

**解决方案：**

| 方案 | 操作 |
|------|------|
| 调大写入批次 | 1000行/批 → 5万-10万行/批 |
| 降低写入并发 | 10个并发 → 3-5个并发 |
| 手动触发合并 | `OPTIMIZE TABLE logs.logs_local FINAL;` |
| 调大合并线程 | `background_pool_size` 从 16 调到 32 |

**紧急处理：**
```sql
-- 查看当前Part数量
SELECT count() FROM system.parts WHERE table = 'logs_local' AND active;

-- 手动强制合并
OPTIMIZE TABLE logs.logs_local FINAL PARTITION '20240101';
```

---

#### 问题2：Memory limit (for query) exceeded

**症状：** 查询 OOM，直接报错

**原因：** 查询扫描的数据太多，内存不够

**解决方案：**

1. **缩小查询范围**
   ```sql
   -- ❌ 查一年，肯定OOM
   WHERE timestamp >= '2024-01-01'
   
   -- ✅ 只查需要的时间范围
   WHERE timestamp >= '2024-01-15' AND timestamp < '2024-01-16'
   ```

2. **开启外部排序/分组**
   ```xml
   <!-- users.xml -->
   <max_bytes_before_external_group_by>20000000000</max_bytes_before_external_group_by>
   <max_bytes_before_external_sort>20000000000</max_bytes_before_external_sort>
   ```
   > 超过20G就溢写到磁盘，慢但不会挂

3. **用聚合下推**
   ```sql
   -- 分布式查询，先在每个分片聚合，减少数据传输
   SET distributed_group_by_no_merge = 0;  -- 默认就是0
   ```

---

#### 问题3：查询超时 / 太慢

**症状：** 查询跑几分钟还没返回

**诊断步骤：**

```sql
-- 1. 看查询日志，分析慢查询
SELECT 
    query_duration_ms / 1000 AS seconds,
    read_rows,
    read_bytes / 1024 / 1024 AS read_mb,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%logs_local%'
ORDER BY query_duration_ms DESC
LIMIT 5;
```

**常见慢查询原因：**

| 原因 | 解决方案 |
|------|---------|
| 没走分区裁剪 | WHERE 条件加分区键 |
| 排序键顺序不对 | 调整 ORDER BY，把常用过滤列放前面 |
| 没跳数索引 | 给高频查询列建跳数索引 |
| SELECT * | 只查需要的列 |

---

#### 问题4：副本不同步

**症状：** 查两个副本返回的数据量不一样

**诊断：**
```sql
-- 查看各副本的队列大小
SELECT
    database,
    table,
    replica_name,
    queue_size,  -- 待同步的队列大小
    absolute_delay
FROM system.replicas
WHERE is_readonly = 0;
```

**处理：**

1. **等一会** - 通常是临时落后，过会自动同步
2. **触发恢复** - `SYSTEM RESTART REPLICA db.table;`
3. **极端情况** - 直接删除有问题的副本，让它重新全量同步

```sql
-- 删除副本本地数据，触发重新同步
SYSTEM DROP REPLICA 'replica_name' FROM ZKPATH '/clickhouse/...';
```

---

#### 问题5：ZooKeeper 连接超时 / Session 过期

**症状：** 各种 ZK 相关报错，写入失败

**原因：**
- ZK 机器负载高，响应慢
- 网络抖动
- ZK 节点数太多（超百万）

**处理：**

1. **检查 ZK 状态**
   ```bash
   echo mntr | nc zk-01 2181
   ```

2. **清理 ZK 垃圾**
   - 旧的快照文件
   - 无用的 ClickHouse 节点

3. **优化 ZK 配置**
   - 事务日志单独挂 SSD
   - 把 ZK 集群和 CH 节点物理分开

---

#### 问题6：磁盘满了

**症状：** 写入报错，`Disk full`

**紧急处理：**

1. **删最老的分区**
   ```sql
   ALTER TABLE logs.logs_local DROP PARTITION '20240101';
   ```

2. **触发合并释放空间**
   ```sql
   OPTIMIZE TABLE logs.logs_local FINAL;
   ```

3. **检查临时文件**
   ```sql
   SELECT sum(bytes_on_disk) FROM system.temporary_tables;
   ```

**预防：**
- 设置磁盘配额 `max_data_size_for_all_tables`
- 设置 TTL 自动过期
- 监控磁盘使用率，>80% 报警

---

#### 问题7：ALTER TABLE 卡住

**症状：** 加列/改结构一直不完成

**原因：** 有查询在跑，拿不到表锁

**处理：**

```sql
-- 1. 查看正在运行的查询
SELECT query_id, query, elapsed FROM system.processes;

-- 2. Kill 掉占用表的长查询
KILL QUERY WHERE query_id = 'xxxx-xxxx-xxxx';

-- 3. 如果还不行，重启CH
systemctl restart clickhouse-server
```

---

#### 问题8：分布式查询结果不准

**症状：** 查分布式表和本地表结果不一样

**常见原因：**

| 原因 | 检查 |
|------|------|
| 某个分片挂了 | `SELECT * FROM system.clusters` |
| 副本不同步 | 查各副本数据量对比 |
| 分片键写错了 | 检查分布式表分片键 |
| 重复写入 | `SELECT count(*) FROM distributed_table FINAL` |

---

#### 问题9：高并发下查询抖动

**症状：** 平时查询100ms，高峰时突然变成10秒

**原因：**
- Mark 缓存失效，频繁读磁盘
- 合并任务和查询抢 IO

**优化：**

```xml
<!-- 调大Mark缓存 -->
<mark_cache_size>5368709120</mark_cache_size>  <!-- 5G -->

<!-- 合并限流，避免影响查询 -->
<merge_tree>
    <max_replicated_merges_in_queue>8</max_replicated_merges_in_queue>
    <number_of_free_entries_in_pool_to_lower_max_size_of_merge>8</number_of_free_entries_in_pool_to_lower_max_size_of_merge>
</merge_tree>
```

---

#### 问题10：数据重复

**症状：** 统计结果比真实值大 2倍

**原因：**
- 写入时重试，导致重复写
- Kafka 消费时重复消费

**解决方案：**

1. **用 ReplacingMergeTree 自动去重**
   ```sql
   ENGINE = ReplacingMergeTree(version)
   ```

2. **查询时去重**
   ```sql
   SELECT count(DISTINCT uuid) FROM ...
   ```

3. **幂等写入**
   - 给每条日志加唯一 UUID
   - 写入前判断是否已存在（但会影响性能）

---

## 四、运维最佳实践

### 4.1 监控指标大盘

必须监控的核心指标：

| 指标名 | 告警阈值 | 说明 |
|-------|---------|------|
| `ClickHouseMetrics_MemoryUsage` | > 80% 内存 | 内存使用率 |
| `parts_to_insert` | > 300 | Part数量，太多会报错 |
| `replicas_queue_size` | > 1000 | 副本同步队列，太大说明同步落后 |
| `readonly_replicas` | > 0 | 只读副本数，大于0说明有问题 |
| `failed_query_rate` | > 5% | 查询失败率 |
| `disk_usage` | > 80% | 磁盘使用率 |

---

### 4.2 备份策略

#### 方案1：快照备份（推荐）

```bash
# 用 clickhouse-backup 工具
clickhouse-backup create 20240115_backup

# 上传到 S3
clickhouse-backup upload 20240115_backup
```

#### 方案2：导出 SQL

```sql
-- 导出小表
SELECT * INTO OUTFILE 'backup.csv' FROM table;
```

**备份频率：**
- 每天一次全量备份
- 保留最近7天的备份
- 核心数据异地备份

---

### 4.3 扩容流程

#### 加新分片步骤：

```
1. 部署新的 ClickHouse 节点，配置加入集群
2. 修改 config.xml 的 remote_servers 配置
3. 在新节点上建表（和其他分片结构一致）
4. 热加载配置（不用重启！）
   SYSTEM RELOAD CONFIG;
5. 验证：SELECT * FROM system.clusters;
6. 逐步迁移老数据到新分片（后台慢慢迁）
```

> ✅ **好消息：** ClickHouse 可以在线扩容，不影响线上服务！

---

### 4.4 日常 Checklist

| 频率 | 检查项 |
|------|-------|
| 每小时 | 磁盘使用率、ZK 状态、Part 数量 |
| 每天 | 慢查询 Top10、失败查询、副本同步延迟 |
| 每周 | 数据分布是否均匀、各分片容量对比 |
| 每月 | 性能基线对比、是否需要扩容 |

---

## 常见面试题

### 1. 生产环境 ClickHouse 怎么保证高可用？

**答案：**
1. **多副本** - 每个分片至少 3 副本，推荐 5 副本
2. **ZK 集群独立部署** - 3 节点 ZK，不与 CH 混部
3. **分布式表查询路由** - 任意节点都能查，无单点
4. **副本间异步同步** - 一个副本挂了不影响其他

### 2. 日志场景和指标场景的表设计有什么不同？

**答案：**

| 维度 | 日志场景 | 指标场景 |
|------|---------|---------|
| 排序键 | 服务+时间 | 指标名+标签+时间 |
| 压缩 | 默认 LZ4 | 时间用 DoubleDelta，值用 Gorilla |
| TTL | 30天 | 365天 |
| 索引 | message 建 ngram 索引 | 不需要额外索引 |
| 聚合 | 一般不预聚合 | 多层预聚合（分/时/天） |

### 3. 怎么排查慢查询？

**答案：**

1. **看 query_log** - 看扫描了多少行，读了多少数据
2. **看执行计划** - `EXPLAIN SELECT ...` 看是否走了索引
3. **检查分区裁剪** - 是否只扫了需要的分区
4. **检查排序键** - WHERE 条件的列是否在 ORDER BY 前面
5. **Profile 事件** - 看各个阶段的耗时分布

### 4. 副本同步的原理是什么？同步延迟高怎么办？

**答案：**

**原理：** 通过 ZK 协调，只同步元数据（块编号、校验和），实际数据点对点拉取。

**延迟高处理：**
1. 检查 ZK 是否正常，是否有压力
2. 检查网络带宽（跨机房同步容易延迟高）
3. 调大 `background_pool_size` 增加同步线程
4. 检查是否有大的 Merge 在跑，占用了 IO
5. 实在不行，删除有问题副本，让它全量重同步

### 5. 数据丢失过吗？怎么避免？

**答案：**

ClickHouse 本身很可靠，数据丢失一般都是运维操作导致：

1. **误删分区** - 操作前备份，权限控制
2. **磁盘损坏** - 多副本，不用 RAID0（生产用 RAID10）
3. **ZK 数据丢失** - ZK 自己也要多副本 + 定期备份
4. **版本升级坏盘** - 升级前一定要备份元数据

> ✅ 有 5 副本 + ZK 3 副本，想丢数据都难！
