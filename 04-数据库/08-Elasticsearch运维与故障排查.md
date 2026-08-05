# Elasticsearch 运维与故障排查

> 本章是 ES 线上运维实战：把 05 章的架构认知落地为「**现象 -> 排查 -> 定位 -> 解决 -> 预防**」的故障排查流程。覆盖线上高频场景，目标是你能独立值班、半夜被叫醒也能按流程止血。前置依赖：05 章（架构）、06 章（读写原理）、07 章（查询调优）。

---

## 一、排查方法论与工具箱

### 1.1 故障排查四步法

```mermaid
graph LR
    A["① 现象采集<br/>监控告警 / 用户反馈 / 业务报错<br/>搞清「什么坏了、影响多大」"]
    --> B["② 快速止血<br/>先恢复可用（限流/扩容/重启/降级）<br/>再查根因，别在告警时深挖"]
    --> C["③ 根因定位<br/>用 API + 日志 + 监控缩小范围<br/>集群级 → 节点级 → 分片级 → 请求级"]
    --> D["④ 长效预防<br/>调参/加监控/改架构/补容量<br/>避免同类问题复发"]
```

> ⚠️ **止血优先于根因**：线上 Red 且业务写入全挂时，第一目标是恢复，不是查为什么。先做能快速恢复的动作（清磁盘/加节点/限流/重启坏节点），稳住后再慢慢定位。在告警现场深挖根因是运维大忌。

### 1.2 常用排查 API 速查表

| 场景 | 命令 |
|------|------|
| 集群健康 | `GET _cluster/health` |
| 待处理任务 | `GET _cluster/pending_tasks` |
| 节点概览 | `GET _cat/nodes?v&h=name,role,heap,ram,disk,cpu,load` |
| 节点详细统计 | `GET _nodes/stats` |
| 索引概览 | `GET _cat/indices?v&health=red` |
| 分片分布 | `GET _cat/shards?v` |
| 未分配分片 | `GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state` |
| 未分配原因 | `GET _cluster/allocation/explain` |
| 磁盘使用 | `GET _cat/allocation?v` |
| 线程池 | `GET _cat/thread_pool?v&h=node,name,active,queue,rejected,completed` |
| recovery 进度 | `GET _cat/recovery?v&active_only=true` |
| 热点线程 | `GET _nodes/hot_threads` |
| 各 breaker | `GET _nodes/stats/breaker` |
| 当前任务 | `GET _cat/tasks?v` |
| 字段映射 | `GET <index>/_mapping` |
| 索引设置 | `GET <index>/_settings` |
| 慢查询日志 | 看 `<cluster>_index_search_slowlog.json` |
| 慢索引日志 | 看 `<cluster>_index_indexing_slowlog.json` |

### 1.3 日志位置与级别

- 日志路径：`path.logs` 配置（默认 `<es-home>/logs/`），主日志 `<clustername>.log`。
- 调高日志级别排查疑难问题（临时，排查完恢复，否则日志量爆炸）：
  ```
  PUT _cluster/settings
  { "transient": { "logger.org.elasticsearch.cluster.routing.allocation": "DEBUG" } }
  ```
- GC 日志：`jvm.options` 里配 `-Xlog:gc*`（JDK 9+）/ `-XX:+PrintGCDetails`（JDK 8），位置在日志目录。
- 关键日志关键字：`OutOfMemoryError`、`circuit_breaking_exception`、`EsRejectedExecutionException`、`unassigned`、`master not discovered`、`Long GC`、`high disk watermark`。

---

## 二、集群 Red / Yellow 排查 ★

### 2.1 现象

- 监控：`cluster.status` 变 `red`（有主分片未分配，数据缺失，最严重）或 `yellow`（主分片正常，有副本未分配）。
- 业务：red 可能导致部分写入/查询失败；yellow 通常不影响读写但有单点风险。

### 2.2 排查流程

```bash
# 1. 确认整体健康
GET _cluster/health
# 重点看：status / number_of_unassigned_shards / unassigned_shards / active_shards_percent

# 2. 找出哪些分片未分配
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node&s=state
# state=UNASSIGNED 的就是问题分片，unassigned.reason 给初步原因

# 3. 查未分配的详细原因（最关键的一步）
GET _cluster/allocation/explain
# 或指定分片：
GET _cluster/allocation/explain
{ "index": "logs-2026.07.22", "shard": 0, "primary": false }
```

### 2.3 定位：未分配原因对照表

| `unassigned.reason` | 含义 | 解决 |
|---------------------|------|------|
| `NODE_LEFT` | 持有该分片的节点掉线 | 等节点回来 / 恢复节点；回不来则等副本提升或新建副本 |
| `NODE_CRASHED` | 节点崩溃 | 同上，排查崩溃原因（见第三章） |
| `CLUSTER_RECOVERED` | 集群刚恢复中 | 等 recovery 完成 |
| `REINITIALIZED` | 分片重新初始化 | 等恢复 |
| `DANGLING_INDEX_IMPORTED` | 悬挂索引（节点本地有但 cluster state 没有） | 决定导入或删除 |
| `ALLOCATION_FAILED` | 分配尝试失败（常见磁盘满/分配异常） | 看 decider 拒绝原因 |
| `NODE_DATA` / `NODE_ATTRIBUTES` | 没有符合角色的节点 | 检查节点角色/属性是否匹配（如 tier_preference 指向 hot 但无 hot 节点） |
| `DECIDERS` | 被 decider 拒绝 | `_cluster/allocation/explain` 看 `decisions` 字段，哪个 decider 说 NO |

`_cluster/allocation/explain` 返回的 `decisions` 会列出每个候选节点被哪个 decider 拒绝（如 `disk_threshold` 说磁盘超 high 水位线），这是定位的金钥匙。

### 2.4 解决

- **磁盘超水位线**：清磁盘（删过期索引/ILM）/ 调水位线（临时）/ 加节点（见第四章）。
- **节点掉线**：恢复节点；若确认不回来，等副本提升为主、再补副本。
- **副本与主同节点**：加节点让有地方放副本。
- **强制分配**（慎用，可能丢数据）：`POST _cluster/reroute` 用 `allocate_stale_primary`/`allocate_replica`，仅在确认主分片数据可丢时用 `allocate_stale_primary`。

```bash
# 临时调高磁盘水位线救急（治标，根因要清磁盘）
PUT _cluster/settings
{ "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%" } }
```

### 2.5 预防

- 磁盘水位告警设在 75%，别等 85% 才处理。
- 副本数 ≥1，data 节点 ≥2，单节点必然 yellow。
- ILM 自动清理过期索引，防止磁盘被吃满。
- 监控 `unassigned_shards > 0`。

---

## 三、节点掉线 / OOM / 假死 ★

### 3.1 现象

- 监控：某节点从 `_cat/nodes` 消失，或 `GET _cluster/health` 报节点数减少。
- 日志：节点日志有 `OutOfMemoryError`、`Long GC`、节点被 master 剔除（`node disconnected`）。
- 集群：触发分片重分配（recovery），可能短时 yellow。

### 3.2 排查

```bash
# 1. 确认哪些节点在线
GET _cat/nodes?v&h=name,role,heap,ram,disk,cpu,load

# 2. 看掉线节点的日志（去那台机器）
#    grep -E "OutOfMemoryError|Long GC|fatal|SIGTERM" <es>/logs/<cluster>.log

# 3. 如果是 OOM，看是哪类 OOM
#    - Java heap space：堆不够 / breaker 没拦住 / 大聚合
#    - Direct buffer memory：Netty direct memory 不够
#    - Metaspace：类加载泄漏（少见）

# 4. 看节点退出前的 GC 日志（频繁 Full GC 是 OOM 前兆）
#    grep -E "Full GC|concurrent mode failure" <es>/logs/gc.log
```

### 3.3 定位与根因

| 现象 | 根因 | 处理 |
|------|------|------|
| `OutOfMemoryError: Java heap space` | 堆不够 / 大查询打爆 / breaker 阈值过高 / segment 元数据占用大（分片多） | 加堆（受 31GB 限制）/ 优化查询 / 降 breaker 阈值 / 减分片 |
| `Long GC` / Full GC 频繁 | 堆太大 GC 停顿长 / 老年代膨胀泄漏 | 检查堆大小、查泄漏（heap dump） |
| `circuit_breaking_exception` 先于 OOM | breaker 正常工作 | 见第十章 |
| 节点被 `node disconnected` 剔除 | GC STW 或网络抖动致心跳超时 | 排查 GC/网络 |
| `master not discovered yet` | master-eligible 不足 quorum | 补 master 节点（见第十四章） |

### 3.4 解决

- **止血**：重启 OOM 节点（先处理堆积的请求，限流降低写入压力再启），ES 重启后会 recovery。
- **根因**：
  - 堆不够且未到 31GB：加堆到 31GB。
  - 已到 31GB 仍 OOM：**减分片数**（segment 元数据占堆）、优化查询（避免大聚合深分页）、降 breaker 阈值提前熔断。
  - 泄漏：dump heap（`jmap -dump:format=b,file=heap.hprof <pid>`）用 MAT 分析大对象。
- **临时降级**：`PUT _cluster/settings` 调小 breaker 阈值让请求更早被拒、保护节点不 OOM。

### 3.5 预防

- 堆 ≤31GB、≤物理内存 50%、`bootstrap.memory_lock: true`。
- 监控 `heap_used_percent > 75%` 告警、`Full GC > 0` 告警。
- 控制每节点分片数（每 GB 堆 ≤20 分片），避免 segment 元数据吃堆。
- breaker 阈值合理（parent 95%、fielddata 40%、request 60%）。
- 给 JVM 配 `-XX:+ExitOnOutOfMemoryError` 或 `-XX:+HeapDumpOnOutOfMemoryError`，OOM 时 dump 后退出，被管理（K8s）重启拉起。

---

## 四、磁盘水位线 / 磁盘满 ★

### 4.1 现象

- 监控：节点 `disk.used_percent` 飙升。
- 日志：`high disk watermark exceeded`、`flood stage disk watermark exceeded`。
- 业务：flood(95%) 触发后索引被设 **只读**（`index.blocks.read_only_allow_delete`），写入直接报错。

### 4.2 三水位线机制

| 水位线 | 默认 | 触发动作 |
|--------|------|---------|
| `low` | 85% | 不再往该节点**分配新分片** |
| `high` | 90% | 开始把该节点分片**迁出**（若别处有空间） |
| `flood_stage` | 95% | 索引被强制**只读**（`blocks.read_only_allow_delete`），只能删不能写 |

```bash
# 看各节点磁盘
GET _cat/allocation?v
# node | shard | disk.indices | disk.used | disk.avail | disk.total | disk.percent
```

### 4.3 解决（救急顺序）

```bash
# 1. 删过期索引（最快降盘）
DELETE logs-2026.06.*
DELETE logs-2026.07.0?

# 2. 临时调高水位线救急（治标，腾出空间后调回）
PUT _cluster/settings
{ "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%" } }

# 3. 解除只读锁（清完磁盘后必须手动解，不会自动恢复）
PUT _all/_settings
{ "index.blocks.read_only_allow_delete": null }

# 4. force_merge 已删除文档占用的空间（删除是标记删，merge 才物理释放）
POST logs-2026.07.01/_forcemerge?max_num_segments=1
```

> ⚠️ **只读锁不会自动解除**：磁盘降到 95% 以下后，`read_only_allow_delete` 不会自动清除，**必须手动 PUT 解锁**，否则业务一直写不进。这是高频踩坑点。

### 4.4 预防

- 磁盘告警设 75%，留足处理时间。
- ILM 自动 TTL 清理过期索引，磁盘水位是 ILM 的触发条件之一。
- `path.data` 配多盘分散 IO 和存储。
- 单节点磁盘别太大（恢复慢、风险集中），靠加节点扩容而非堆单盘。

---

## 五、GC 停顿 / Long GC

### 5.1 现象

- 监控：`jvm.gc.old.collection_time` 飙升、`heap_used_percent` 高位震荡。
- 日志：`Long GC` 警告、频繁 Full GC。
- 集群：节点 STW 致心跳超时被 master 剔除（表现为节点掉线，见第三章）。

### 5.2 排查

```bash
# 1. 看各节点 GC
GET _nodes/stats/jvm
# 重点：gc.collectors.old.collection_count/time（Full GC）、young.collection_time

# 2. 看 GC 日志
#    grep -E "Full GC|Pause Young|concurrent mode failure" <es>/logs/gc.log
#    判断：是 Young GC 还是 Old/Full GC、频率、单次耗时
```

### 5.3 定位

- **频繁 Young GC 但快**：写入/查询并发高，正常，调小单次延迟可用 G1/ZGC。
- **Full GC 频繁**：老年代膨胀，可能泄漏或大对象常驻（大 fielddata cache、大 query cache、segment 元数据多）。
- **单次 GC 耗时长（>1s）**：堆太大或用 CMS 老收集器。ES 默认 G1（JDK 17+），单次停顿较短。

### 5.4 解决

- 堆太大致 GC 慢：**降堆**（前提是有 page cache 空间），G1 单次停顿优于 CMS。
- 老年代膨胀：查 `GET _nodes/stats/indices/fielddata`、`GET _nodes/stats/indices/query_cache`，哪个 cache 占用大就限制其 size。
- segment 元数据占堆：减分片数、force_merge。
- 泄漏：heap dump 用 MAT 分析支配树，找大对象。

### 5.5 预防

- 堆 ≤31GB，G1/ZGC 收集器。
- 限制 fielddata cache：`indices.fielddata.cache.size: 40%`。
- 监控 Full GC >0 告警、GC 耗时占比。

---

## 六、Thread Pool Rejected（写入 / 查询拒绝）★

### 6.1 现象

- 监控：`thread_pool.write.rejected` / `thread_pool.search.rejected` / `thread_pool.get.rejected` >0 且持续。
- 业务：客户端收到 429 `EsRejectedExecutionException`，写入/查询被拒。

### 6.2 排查

```bash
# 看各节点各线程池 active/queue/rejected
GET _cat/thread_pool?v&h=node,name,active,queue,rejected,completed

# 哪个池 rejected 高就是哪个环节过载：
#   write rejected   -> 写入过载（bulk 太多/太大）
#   search rejected  -> 查询过载（复杂查询/深分页）
#   get rejected     -> 点查过载
```

### 6.3 解决

> ⚠️ **不要一味调大队列**：队列越大请求积压越久、延迟越高，把「快速失败」变「慢速失败」，可能拖垮上层超时。reject 是过载保护，根本解法是降负载或扩容。

- **客户端限流**：降低并发，bulk 控制合理 batch（5~15MB）和并发数（如每节点 2~4 个并发 bulk）。
- **写入侧**：调大 `refresh_interval`（如 30s）减 segment 生成、批量写时副本设 0、translog 异步。
- **查询侧**：优化慢查询（见第八章）、用 filter 缓存、避免深分页、限制 `size`。
- **扩容**：加 data 节点分担、加协调节点卸载协调压力。
- **必要时调队列**（谨慎，配合客户端超时调整）：
  ```bash
  PUT _cluster/settings
  { "transient": { "thread_pool.write.queue_size": 1500 } }
  ```
- 客户端必须实现**重试 + 退避**（exponential backoff），429 是预期内的过载信号，不该报错给用户。

### 6.4 预防

- 监控 rejected >0 持续告警。
- 容量规划留 30% 余量，洪峰不触 reject。
- 客户端连接池和重试策略就绪。

---

## 七、写入慢 / 写入拒绝

### 7.1 现象

- 监控：`indexing.index_time` 飙升（单条写入耗时高）、bulk RT 高、write rejected。
- 业务：写入延迟大、bulk 部分失败、消费堆积。

### 7.2 排查

```bash
# 1. 看写入耗时和拒绝
GET _nodes/stats/indices/indexing
# indexing.index_total / index_time_in_millis 算单条延迟

# 2. 看是不是 merge 跟不上（segment 堆积）
GET _nodes/stats/indices/merge
GET _cat/segments/<index>?v   # segment 数过多=merge 慢

# 3. 看磁盘 IO 是否饱和
GET _nodes/stats/fs   # disk_io 读写量
GET _cat/allocation?v  # 磁盘水位

# 4. 看是不是 refresh 太频繁
GET <index>/_settings/index.refresh_interval
```

### 7.3 定位：写入慢的常见根因

| 根因 | 信号 | 解法 |
|------|------|------|
| 单条写没用 bulk | indexing 请求多但小 | 改 bulk 批量 |
| bulk 太大 | 单次 RT 高、内存压力 | 降到 5~15MB/批 |
| refresh 太频繁（默认1s） | segment 多、merge 压力大 | 调大 refresh_interval（30s~60s） |
| 副本写入开销 | 写入 RT 受副本拖累 | 批量导入时副本设 0，写完恢复 |
| translog 同步刷盘 | 每次写都 fsync | `index.translog.durability: async` |
| 磁盘 IO 饱和 | disk_io 高、IO wait 高 | SSD、加盘、加节点 |
| segment 堆积 merge 跟不上 | segments.count 高 | 调 merge 并发、低峰 force_merge |
| 字段太多索引太多 | 写放大严重（见 07 章写放大） | 关不必要的 index/doc_values |
| mapping 动态推断开销 | 字段持续增长 | 关 dynamic |

### 7.4 批量导入优化配置

```bash
# 写入密集期的临时配置（牺牲近实时和副本换写入）
PUT logs-*/_settings
{ "index": {
    "refresh_interval": "60s",
    "number_of_replicas": 0,
    "translog.durability": "async" } }

# 写完后恢复
PUT logs-*/_settings
{ "index": {
    "refresh_interval": "1s",
    "number_of_replicas": 1,
    "translog.durability": "request" } }
POST logs-*/_forcemerge?max_num_segments=1
```

> ⚠️ **副本设 0 期间是单点**：批量导入时临时关副本可大幅提速，但此期间该分片只有一份，节点故障会丢数据。只用于可重放的数据导入，写完必须恢复副本。

### 7.5 segment 管理与 force_merge ★

Segment 是 ES 存储的基本单元，太多小 segment 会导致：查询慢（要开很多文件）、堆占用高（元数据）、文件句柄浪费。

#### 7.5.1 merge 策略参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `index.merge.policy.max_merged_segment` | 5GB | 单个 merged segment 上限 |
| `index.merge.policy.segments_per_tier` | 10 | 每层 segment 数 |
| `index.merge.scheduler.max_thread_count` | Math.min(3, 核数/2) | merge 并发线程数 |

- 写入密集场景可适当调大 `max_thread_count`，加速 merge 避免 segment 堆积
- 但别调太大，merge 吃磁盘 IO，会影响正常读写

#### 7.5.2 force_merge（强制合并）

手动触发 segment 合并，把多个小 segment 合并成少量大 segment，同时物理删除被标记删除的文档。

```bash
# 合并为 1 个 segment（最大化压缩，但非常耗 IO）
POST logs-2026.07.01/_forcemerge?max_num_segments=1

# 只合并已被删除文档占比超过某个阈值的 segment
POST my_index/_forcemerge?only_expunge_deletes=true
```

**适用场景**：
- 只读索引（历史日志）合并降存储和降 segment 数
- 大量删除后释放磁盘空间
- ILM warm/cold 阶段自动执行

**注意事项**：
- ⚠️ **非常耗 IO 和 CPU**，生产必须在低峰期执行
- ⚠️ 只对**不再写入**的索引做（冷数据），热索引做了还会生成新 segment
- ⚠️ 不要对 hot 层的活跃索引做 force_merge，越合并越慢
- 合并过程中集群可能变慢，设好 `indices.recovery.max_bytes_per_sec` 限流

### 7.6 预防

- 永远用 bulk，按 5~15MB 分批。
- 写密集场景 refresh_interval 调大、translog async。
- SSD + 多盘。
- 监控 indexing 延迟、merge 耗时、segment 数。

---

## 八、查询慢 / 慢查询

### 8.1 现象

- 监控：`search.query_time` 飙升、query rejected、用户反馈搜索慢。
- 日志：慢查询日志 `index_search_slowlog` 有记录。

### 8.2 排查

```bash
# 1. 看查询耗时和 QPS
GET _nodes/stats/indices/search
# query_total / query_time_in_millis 算单次延迟

# 2. 配置慢查询日志（先开慢日志阈值）
```

#### 8.2.1 慢日志完整配置

慢日志分 **查询慢日志**（search slowlog）和 **索引慢日志**（indexing slowlog），按级别（warn/info/debug/trace）分别设阈值：

```bash
# 查询慢日志
PUT logs-*/_settings
{
  "index.search.slowlog.threshold.query.warn": "5s",     // query 阶段超 5s 打 warn
  "index.search.slowlog.threshold.query.info": "2s",
  "index.search.slowlog.threshold.query.debug": "500ms",
  "index.search.slowlog.threshold.query.trace": "100ms",

  "index.search.slowlog.threshold.fetch.warn": "1s",     // fetch 阶段超 1s 打 warn
  "index.search.slowlog.threshold.fetch.info": "500ms",
  "index.search.slowlog.level": "info"                   // 记录级别，info 及以上才落盘
}

# 索引慢日志
PUT logs-*/_settings
{
  "index.indexing.slowlog.threshold.index.warn": "10s",
  "index.indexing.slowlog.threshold.index.info": "5s",
  "index.indexing.slowlog.source": "1000"                // 最多记录 1000 字符的 source
}
```

慢日志文件位置：`path.logs` 目录下，文件名形如：
- `<cluster>_index_search_slowlog.json`（查询慢日志）
- `<cluster>_index_indexing_slowlog.json`（索引慢日志）

> 💡 慢日志是**按索引级别**配置的，可以只对怀疑慢的索引开，避免全集群开了日志量爆炸。上线前建议设 warn 5s，有问题再调低。

#### 8.2.2 用 profile API 精确定位慢查询

找到慢查询后，用 `?profile=true` 查看每个子句耗时分布：

```bash
GET my_index/_search?profile=true
{
  "query": {
    "bool": {
      "must": [{ "match": { "title": "手机" } }],
      "filter": [{ "range": { "price": { "gte": 100 } } }]
    }
  }
}
```

profile 返回结果里每个查询类型、每个分片的耗时、打分耗时、advance 次数都有，直接定位是哪一步慢。

#### 8.2.3 排查步骤

```bash
# 1. 看整体查询耗时
GET _nodes/stats/indices/search

# 2. 开慢日志抓慢查询（配好阈值等日志）
#    去日志目录 grep slowlog

# 3. 拿慢查询 DSL，用 profile 分析
#    在 Kibana Dev Tools 里跑 ?profile=true

# 4. 根据 profile 结果定位是哪类慢（wildcard / 深分页 / 大聚合 / script）

```bash
# 1. 看查询耗时和 QPS
GET _nodes/stats/indices/search
# query_total / query_time_in_millis 算单次延迟

# 2. 看慢查询日志（先确保开了慢日志）
PUT logs-*/_settings
{ "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "1s" }
# 然后看 <es>/logs/<cluster>_index_search_slowlog.json

# 3. 用 profile API 分析单条慢查询的耗时分布
GET logs-*/_search?profile=true
{ "query": { ... } }
# profile 返回每个 query 子句、每个分片的耗时，定位是哪一步慢
```

### 8.3 定位：慢查询常见根因

| 根因 | 特征 | 解法 |
|------|------|------|
| `wildcard` 前缀通配 | `*keyword*` 全 term 扫描 | 改 match/match_phrase 或 completion suggester |
| 深分页 `from+size` | from 大、size 大 | 改 search_after |
| 大聚合高 cardinality | terms 聚合桶多、cardinality | 限定范围、预聚合 rollup/transform、加 precision_threshold |
| `script` 脚本查询 | 每条文档跑脚本 | 改用 runtime field 预计算或 ingested 字段 |
| 返回大 `_source` | 取回完整文档大 | `_source` 过滤只取需要的字段 |
| 分片太多 fan-out | 查询打到所有分片 | 用 routing 只查目标分片、合理分片数 |
| text 字段聚合开 fielddata | fielddata 吃堆且慢 | 用 keyword 子字段聚合 |
| 索引无时间范围过滤 | 查全量数据 | 加 filter 限定时间/范围 |

### 8.4 解决与预防

- 慢查询治理：开慢日志 -> profile 定位 -> 改写查询（filter 替代 query、search_after 替代深分页、删 wildcard）。
- 必要的聚合预计算：transform/rollup 或在写入时 ingest 预聚合。
- 查询带 routing 减少 fan-out。
- 监控 P99 查询延迟，慢查询自动告警。
- 限制单查询返回量（`max_result_window`、`search.max_buckets`）。

---

## 九、Heap 占用高排查 ★

### 9.1 现象

- 监控：`heap_used_percent` 长期 >75%、逼近 85%。
- 关联：可能触发 GC 频繁（第五章）、breaker 触发（第十章）、最终 OOM（第三章）。

### 9.2 排查：堆里到底是什么

```bash
# 1. 总览
GET _nodes/stats/jvm
# mem.heap_used_percent

# 2. 看各 cache 占用
GET _nodes/stats/indices/fielddata   # fielddata cache（text 聚合）
GET _nodes/stats/indices/query_cache # filter 结果缓存
GET _nodes/stats/indices/request_cache # shard 请求缓存
GET _nodes/stats/segments            # segment 内存占用（term index 等常驻）

# 3. 看分片数和 segment 数（每分片有元数据常驻堆）
GET _cat/shards?v&h=node | wc -l
GET _cat/segments?v&s=node

# 4. 看 cluster state 大小（mapping 爆炸会让它虚大）
GET _cluster/state/version?filter_path=metadata
POST _cluster/state/version?human   # 看 cluster state 体积
```

### 9.3 定位：堆占用高的常见来源

| 来源 | 信号 | 解法 |
|------|------|------|
| **fielddata cache** | text 字段聚合、cache size 无限 | 限制 `indices.fielddata.cache.size: 40%`、用 keyword 聚合 |
| **query cache** | 大量 filter 查询 | 默认堆 10%，一般不是大头 |
| **segment 元数据** | 分片/segment 多 | 减分片数、force_merge 降 segment |
| **cluster state** | mapping 字段多/索引多 | 关 dynamic、减索引、ILM 清理 |
| **translog buffer** | 写入密集 | 一般不大 |
| **indexing buffer** | 默认堆 10% | 按需调 `indices.memory.index_buffer_size` |
| **泄漏** | 老年代持续涨不回落 | heap dump + MAT 分析 |

### 9.4 heap dump 分析（定位泄漏）

```bash
# dump 堆（会 STW，影响生产，低峰做或单节点做）
jmap -dump:format=b,file=heap.hprof <pid>
# 或用 ES 的导出（部分版本支持）
# jcmd <pid> GC.heap_dump heap.hprof

# 用 Eclipse MAT 打开 hprof：
#   - Dominator Tree 看大对象支配树
#   - Leak Suspects 报告看疑似泄漏点
#   - 重点看是否某类对象异常多（如 SegmentReader、查询缓存条目）
```

> ⚠️ **dump 会触发 STW**：生产环境 dump 会让节点暂停服务数秒~数十秒，应在低峰或针对单节点做，且 dump 文件大（≈堆大小，31GB），确保磁盘够。

### 9.5 解决

- cache 类：限制大小（fielddata 40%）让 ES 自动淘汰。
- segment/分片类：减分片数、force_merge、ILM 清理。
- cluster state 类：关 dynamic mapping、减索引数。
- 泄漏类：按 MAT 结果修代码（自定义插件/脚本）或升级版本。
- 临时：重启节点释放堆（治标，泄漏会再现）。

### 9.6 预防

- 监控 heap >75% 预警、>85% 高危。
- 限制 fielddata cache size。
- 控制分片数（每 GB 堆 ≤20）。
- 定期清理过期索引。

---

## 十、Circuit Breaker 触发

### 10.1 现象

- 业务：查询返回 `circuit_breaking_exception`（如 `Data too large, ... would field circuit breaking`）。
- 监控：`breakers.*.tripped` 计数上涨。

### 10.2 排查

```bash
# 看各 breaker 使用量和触发次数
GET _nodes/stats/breaker
# parent / fielddata / request / accounting / in_flight_requests
# 重点关注 tripped_count >0 的

# 通常是 request 或 fielddata breaker 跳闸
```

### 10.3 定位与解决

| Breaker 跳闸 | 触发场景 | 解法 |
|--------------|---------|------|
| **fielddata** | text 字段聚合、大 cardinality | 改用 keyword 子字段聚合；限制 fielddata cache size |
| **request** | 大 terms 聚合、深分页、script | 缩小聚合范围、加 precision_threshold、避免 script |
| **parent** | 子 breaker 总和超 95% | 优化大查询、加节点 |
| **in_flight_requests** | 在途请求积压 | 降并发、限流 |

```bash
# 临时调高 request breaker（治标，根本要优化查询）
PUT _cluster/settings
{ "transient": { "indices.breaker.request.limit": "70%" } }
```

> ⚠️ **调大 breaker 阈值是拆保险丝**：频繁触发应优化查询或扩容，不是调大阈值。调大了迟早 OOM（breaker 失效 -> heap 爆 -> 节点挂）。

### 10.4 预防

- 避免对高 cardinality 字段做大 terms 聚合。
- text 聚合一律走 keyword 子字段。
- 监控 breaker tripped >0 告警。

---

## 十一、Recovery 慢

### 11.1 现象

- 节点重启/加入后集群长时间不 green，`relocating_shards` / recovery 持续很久。
- `GET _cat/recovery?v&active_only=true` 显示 recovery 进度缓慢。

### 11.2 排查

```bash
# 1. 看 recovery 进度
GET _cat/recovery?v&active_only=true
# index | shard | time | type | stage | files | files_percent | bytes | bytes_percent

# 2. 卡在哪个 stage（INDEX 阶段拷贝数据最慢）

# 3. 看限流配置
GET _cluster/settings?include_defaults=true&filter_path=indices.recovery.*
```

### 11.3 根因

- **分片太大**：单分片几十 GB，拷贝慢。这是「单分片 <50GB」的根因。
- **限流太紧**：`indices.recovery.max_bytes_per_sec` 默认约 40mb/s，生产偏小。
- **并发 recovery 太多**：一次重启大量节点或大量分片同时恢复。
- **网络/磁盘瓶颈**：recovery 吃满带宽或 IO。
- **translog 太大**：flush 前积攒大量 translog，recovery 要回放很久。

### 11.4 解决

```bash
# 1. 调大 recovery 带宽（生产常用 250mb~500mb，视带宽）
PUT _cluster/settings
{ "transient": { "indices.recovery.max_bytes_per_sec": "500mb" } }

# 2. 调大并发 recovery
PUT _cluster/settings
{ "transient": {
    "cluster.routing.allocation.node_concurrent_recoveries": 4,
    "cluster.routing.allocation.node_initial_primaries_recoveries": 8 } }

# 3. 降低 translog 积压（提前 flush）
POST <index>/_flush
```

### 11.5 预防

- 单分片 <50GB（控制分片数 + ILM rollover）。
- 生产调大 `max_bytes_per_sec`。
- 滚动重启一次一个节点，避免大量分片同时 recovery。
- 低峰期做节点维护。

---

## 十二、Mapping 爆炸 / Cluster State 膨胀

### 12.1 现象

- 监控：`pending_tasks` 飙升、Master CPU 高、建索引慢、堆虚高。
- 表现：单个索引字段数成千上万、cluster state 体积大。

### 12.2 排查

```bash
# 1. 看哪些索引字段数异常多
GET _cat/indices?v&h=index,docs.count,store.size&s=store.size:desc
GET <index>/_mapping   # 看字段数

# 2. 看 cluster state 大小
GET _cluster/state?filter_path=metadata.indices
POST _cluster/state?human   # 看整体体积（注意可能很大）

# 3. 看是否有大量字段名是动态生成的（如把 userId 当字段名 -> 每个用户一个字段）
```

### 12.3 根因

- **动态映射 + 字段爆炸**：写入日志把 `tag.<动态值>`、`field_<随机>` 当字段，每个值一个字段，字段数爆炸。
- **mapping 爆炸**：字段数从几十涨到几万，cluster state 中 mapping 占绝大部分，发布慢、每节点堆被占。

### 12.4 解决

```bash
# 1. 关动态映射（治根）
PUT _template/logs_template
{ "index_patterns": ["logs-*"],
  "mappings": { "dynamic": false } }   # 或 strict（超出定义的字段直接拒绝写入）

# 2. 已爆炸的索引：reindex 到规范 mapping 的新索引
POST _reindex
{ "source": { "index": "logs-bad" }, "dest": { "index": "logs-good" } }

# 3. 清理无用字段：用 _reindex 的 script 过滤，或删旧索引
DELETE logs-bad

# 4. 对「以值为字段」的场景改用 nested 或 object 而非字段展开
```

### 12.5 预防

- 生产 `dynamic: false`（不索引新字段）或 `strict`（拒绝未定义字段）。
- 监控索引字段数，超阈值告警。
- 字段命名规范，禁止把动态值当字段名（用 nested 结构存键值对）。

---

## 十三、热点分片 / 数据倾斜

### 13.1 现象

- 监控：某节点 CPU/IO 显著高于其他、某分片读写 QPS 异常高。
- 表现：集群整体不忙但个别节点过载（木桶效应）。

### 13.2 排查

```bash
# 1. 热点线程：哪个线程在吃 CPU
GET _nodes/hot_threads?threads=10&interval=10s
# 看 stack，判断是在 indexing 还是 search

# 2. 看分片级别负载（需监控按 shard 维度采集）
GET _cat/shards?v&h=index,shard,docs,store,node&s=docs:desc
# 文档数/大小悬殊 = 数据倾斜

# 3. 看是否 routing 集中（同一 routing 全落一个分片）
```

### 13.3 根因

- **routing 倾斜**：按 user_id 路由，某大用户数据量远超他人，单分片过载（hot shard）。
- **分片数不均**：主分片数太少，单分片承载过多。
- **时间热点**：时序数据当天分片读写集中（正常，但 hot 节点要够）。

### 13.4 解决

- **hot shard**：把热点索引分到更多主分片（需 reindex）、热数据单独放 hot 层足够节点。
- **routing 倾斜**：大用户单独索引、或 routing 加散列打破集中。
- **分片不均**：调 `cluster.routing.allocation.balance.*` 让分片均衡分布。

```bash
# 调均衡策略（让分片更均匀分布）
PUT _cluster/settings
{ "transient": {
    "cluster.routing.allocation.balance.shard": "0.4",
    "cluster.routing.allocation.balance.index": "0.3",
    "cluster.routing.allocation.balance.disk_usage": "0.4" } }  # 7.x 支持按磁盘使用均衡
```

### 13.5 预防

- 监控按 shard 采集负载，识别 hot shard。
- 大索引分片数合理，别太少。
- hot 层节点数与热点数据量匹配。

---

## 十四、网络分区 / 脑裂 / Master 假死

### 14.1 现象

- `master not discovered yet`、集群选不出 master。
- 集群「假死」：能查询但建索引/改 mapping/分配分片全卡（pending_tasks 堆积）。

### 14.2 根因（7.0+ 仍可能）

- master-eligible 节点 <3，或一次挂掉超过半数 master。
- master 之间网络分区。
- master 节点 GC STW 或负载过高致心跳超时。

### 14.3 排查

```bash
# 1. 集群健康
GET _cluster/health
# number_of_pending_tasks 持续高位 = Master 通道堵

# 2. 看 master 是否存在、谁在选主
GET _cat/master
GET _cluster/state?filter_path=master_node,nodes

# 3. 看 pending 任务
GET _cluster/pending_tasks

# 4. 看日志
# grep -E "master not discovered|failed to publish|disconnected" <master>/logs/<cluster>.log
```

### 14.4 解决

- **master 挂了凑不齐 quorum**：尽快恢复 master 节点（重启）；若永久丢失且只剩 1 个 master，用 `POST _cluster/voting_config_exclusions` 调整投票配置（**慎用，会降低容错**）。
- **Master 过载假死**：看是否数据节点 GC 拖累（应专用 master 隔离）、是否 cluster state 过大（减分片/减索引）、pending 任务是否被某个慢任务阻塞。
- **临时止血**：重启 master 节点（一次一个，等选主稳定）。

```bash
# 临时调大 master 任务处理（治标）
PUT _cluster/settings
{ "transient": { "cluster.routing.allocation.cluster_concurrent_rebalance": "2" } }
```

### 14.5 预防

- 专用 master ≥3（奇数）、物理隔离、不存数据。
- master 与 data 分离，避免 data GC/IO 拖累 master。
- 监控 pending_tasks、master 节点 CPU。
- 控制分片数和 cluster state 体积。

---

## 十五、滚动升级与节点维护

### 15.1 滚动升级流程（不中断服务）

```bash
# 0. 备份！升级前必须 snapshot（见第十六章）
PUT _snapshot/backup_repo/snapshot_pre_upgrade?wait_for_completion=true

# 1. 关闭分片自动分配
PUT _cluster/settings
{ "transient": { "cluster.routing.allocation.enable": "primaries" } }

# 2. 停止索引写入刷新（可选，让数据落盘）
POST _flush

# 3. 一次停一个节点：停止 ES 进程
# 4. 升级 ES 版本（替换软件包）
# 5. 启动节点，等它加入集群、分片恢复
GET _cat/health
GET _cat/recovery?v&active_only=true

# 6. 集群 green 后，恢复分片分配
PUT _cluster/settings
{ "transient": { "cluster.routing.allocation.enable": "all" } }

# 7. 等集群 green，处理下一个节点
```

### 15.2 下线 master 节点

下线专用 master 要小心 quorum：3 个下 1 个没事，3 个下 2 个会假死。

```bash
# 排除投票配置（告诉集群这个 master 即将离开，重新计算 quorum）
POST _cluster/voting_config_exclusions?node_names=es-master-3

# 等 60s 让配置生效，再停节点
# 节点停止后，可清除排除项：
DELETE _cluster/voting_config_exclusions
```

### 15.3 下线 / 缩容 data 节点

```bash
# 让分片从该节点迁走
PUT _cluster/settings
{ "transient": { "cluster.routing.allocation.exclude._name": "es-data-3" } }

# 等该节点分片数变 0
GET _cat/allocation?v

# 分片清零后安全停节点
```

### 15.4 注意事项

- 滚动升级期间高版本不向低版本分配分片（NodeVersion decider），所以**必须先升 master 再升 data**，且不能跨大版本滚动（如 7.x->8.x 需全文重索引或 reindex，不能滚动）。
- 升级前必备份、必在测试环境验证。
- 业务侧准备双写或只读降级预案。

---

## 十六、快照备份与恢复

> 快照是 ES 数据唯一的可靠备份手段，也是升级、迁移、灾难恢复的底气。**没有快照就没有底气说数据安全**。

### 16.1 注册仓库

```bash
# 文件系统仓库（需 path.repo 配置 + 所有节点共享或各节点本地）
PUT _snapshot/backup_repo
{ "type": "fs",
  "settings": { "location": "/mnt/backup/es", "compress": true } }

# 对象存储仓库（S3/OSS，需装插件）
PUT _snapshot/s3_repo
{ "type": "s3",
  "settings": { "bucket": "my-backup", "base_path": "es-snapshots", "compress": true } }
```

### 16.2 创建快照

```bash
# 全集群快照
PUT _snapshot/backup_repo/snapshot_20260722?wait_for_completion=false
# 异步，用 GET _snapshot/backup_repo/snapshot_20260722 查进度

# 指定索引快照
PUT _snapshot/backup_repo/snapshot_logs
{ "indices": "logs-*", "ignore_unavailable": true, "include_global_state": false }
```

- `include_global_state: true` 会备份 cluster state（含模板、ILM 策略），恢复时还原；跨集群恢复一般设 false 避免覆盖目标集群配置。
- 增量：快照是增量的，只存变化的部分，可定期全量 + 频繁增量。

### 16.3 恢复

```bash
# 恢复到原索引名（原索引要先关闭或删除）
POST _snapshot/backup_repo/snapshot_20260722/_restore
{ "indices": "logs-2026.07.22", "rename_pattern": "logs-(.+)", "rename_replacement": "restored-$1" }

# 恢复时改名避免覆盖现网
```

- 恢复时目标索引不能已存在且在用，用 `rename_pattern` 改名。
- 恢复期间索引不可写（恢复完成后可用）。

### 16.4 ILM 自动备份

ILM 可配 `snapshot` action，索引到一定阶段自动打快照，配合 frozen 层做 searchable snapshot。

### 16.5 预防与演练

- **定期快照**：生产每天全量 + 高频增量，验证可恢复。
- **定期恢复演练**：在隔离环境恢复快照验证完整性，别等真出事才发现快照坏了。
- 快照仓库做异地冗余（S3 跨 region 复制）。
- 升级、大变更前必打快照。

---

## 十七、ILM 故障

### 17.1 现象

- 滚动索引没按预期 rollover（数据全堆在一个索引）。
- ILM policy 报错、索引停留在某阶段。
- `GET <index>/_ilm/explain` 显示 policy 状态异常。

### 17.2 排查

```bash
# 看索引的 ILM 状态
GET logs-*/_ilm/explain
# 重点：managed、phase、action、step、step_info

# 看 ILM 策略
GET _ilm/policy/logs_policy

# 看 rollover 是否被触发（看索引创建时间和大小）
GET _cat/indices/logs-*?v&h=index,docs.count,store.size,creation.date
```

### 17.3 常见故障

| 故障 | 原因 | 解决 |
|------|------|------|
| 不 rollover | rollover 条件没满足（max_age/max_size/max_docs 都没到）| 检查 policy 条件；用 `POST <alias>/_rollover` 手动触发测试 |
| 索引未托管（managed=false） | 索引没绑定 policy、或别名不是 write alias | 绑定 policy + 设置 `is_write_index: true` 的别名 |
| 卡在某 step | 该 step 报错（如 shrink 节点不足、force_merge 失败）| 看 step_info，针对性处理 |
| policy 报错 | 语法错或引用了不存在的节点属性 | 修 policy |
| delete 阶段不删 | age 从 rollover 算，索引没 rollover 则 age 不增长 | 确保 rollover 正常 |

### 17.4 预防

- ILM policy 上线前在测试环境验证全流程（hot->warm->cold->delete）。
- 监控 ILM 报错、索引 phase 异常停留。
- rollover 别名正确配 `is_write_index: true`。

---

## 十八、运维速查手册

> 尚硅谷视频实操部分大量使用 Kibana Dev Tools 与 \_cat API。下面整理生产最常用的速查表，面试被问到「怎么查 XX」也能快速说出来。

### 18.1 \_cat 系列命令速查

所有 \_cat 命令都支持 `?v`（显示表头）、`?help`（看支持哪些列）、`?h=col1,col2`（指定列）、`?s=col:desc`（排序）。

| 命令 | 作用 | 常用场景 |
|------|------|---------|
| `_cat/health?v` | 集群健康总览 | 第一眼看集群状态 |
| `_cat/nodes?v` | 节点列表 | 看节点角色、堆、磁盘、CPU |
| `_cat/indices?v&s=store.size:desc` | 索引列表（按大小排序） | 找大索引、看索引健康 |
| `_cat/shards?v` | 分片分布 | 看分片在哪个节点、状态 |
| `_cat/allocation?v` | 各节点分片数与磁盘 | 看磁盘使用率、分片分布均不均 |
| `_cat/thread_pool?v&h=node,name,active,queue,rejected` | 线程池状态 | 排查 rejected、过载 |
| `_cat/recovery?v&active_only=true` | 正在进行的 recovery | 看恢复进度 |
| `_cat/pending_tasks` | Master 待处理任务 | 排查 Master 过载 |
| `_cat/aliases?v` | 别名列表 | 查别名指向哪些索引 |
| `_cat/templates?v` | 索引模板列表 | 查有哪些模板 |
| `_cat/plugins?v` | 插件列表 | 查各节点装了什么插件 |
| `_cat/segments/<index>?v` | 索引 segment 详情 | 排查 segment 数 |
| `_cat/fielddata?v` | fielddata 占用 | 排查 text 聚合占堆 |
| `_cat/nodeattrs?v` | 节点自定义属性 | 查 rack/tier 属性 |

### 18.2 常用排查命令组合

```bash
# 看集群状态 + pending tasks（Master 是否堵）
GET _cluster/health
GET _cluster/pending_tasks

# 看节点负载（哪些节点忙）
GET _cat/nodes?v&h=name,role,heap.percent,ram.percent,disk.used_percent,cpu,load_1m&s=cpu:desc

# 找最大的 10 个索引
GET _cat/indices?v&h=index,health,docs.count,store.size&s=store.size:desc&v=true

# 找未分配的分片
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason,node&s=state

# 看哪些线程池在拒绝
GET _cat/thread_pool?v&h=node,name,active,queue,rejected&s=rejected:desc

# 热点线程（谁在吃 CPU）
GET _nodes/hot_threads?threads=10&interval=5s
```

### 18.3 Kibana Dev Tools 小技巧

- **Ctrl/Cmd + Enter**：执行当前光标所在的请求
- **Ctrl/Cmd + / **：注释/取消注释
- **自动补全**：输入 DSL 时按 Tab 补全
- **历史记录**：右侧 History 面板查看历史执行
- **多个请求**：一个文件里写多个请求，光标在哪执行哪个（用空行分隔）

---

## 十九、常见面试题（运维向）

1. **ES 集群 Red 怎么排查？**
   先 `GET _cluster/health` 确认，`GET _cat/shards?v` 找 UNASSIGNED 分片，`GET _cluster/allocation/explain` 看被哪个 decider 拒绝。常见原因：磁盘超水位线、节点掉线、副本与主同节点。止血优先（清磁盘/加节点/调水位线），再查根因。

2. **节点 OOM 怎么排查和预防？**
   看节点日志 `OutOfMemoryError`、GC 日志判 Full GC 频繁、`GET _nodes/stats` 看 cache 占用。根因：堆不够/大查询打爆/breaker 没拦住/segment 元数据多。预防：堆≤31GB、控分片数、breaker 阈值合理、监控 heap>75%。

3. **磁盘满了 ES 自动只读怎么解除？**
   flood(95%) 触发 `index.blocks.read_only_allow_delete`，**只读锁不会自动解除**，清完磁盘后必须 `PUT <index>/_settings {"index.blocks.read_only_allow_delete": null}` 手动解锁。预防：ILM 清理、磁盘告警 75%、监控水位线。

4. **write/search rejected（429）怎么办？要不要调大队列？**
   reject 是过载保护，看 `GET _cat/thread_pool` 是哪个池。**不盲目调大队列**（队列大延迟高，把快速失败变慢速失败）。根本解法：客户端限流降并发、优化查询/写入、扩容。客户端必须重试退避。

5. **ES 慢查询怎么定位？**
   开慢查询日志（`index.search.slowlog.threshold.query`）找慢查询，再用 `?profile=true` 看 profile 输出定位是哪个子句、哪个分片慢。常见慢根因：wildcard、深分页、大聚合高 cardinality、script、大 _source。对应改写查询。

6. **堆占用高怎么定位是什么占了堆？**
   `GET _nodes/stats/indices/{fielddata,query_cache,request_cache}` 看各 cache、`GET _cat/segments` 看 segment 数、看 cluster state 大小。常见大头：fielddata（text 聚合）、segment 元数据（分片多）、cluster state（mapping 爆炸）。定位泄漏用 `jmap` dump + MAT。dump 会 STW，低峰做。

7. **Circuit Breaker 频繁触发怎么办？**
   `GET _nodes/stats/breaker` 看哪个 breaker tripped。通常是 fielddata（text 聚合）或 request（大聚合/深分页）。优化查询（keyword 聚合、缩小范围、加 precision_threshold）而非调大阈值（调大是拆保险丝，迟早 OOM）。

8. **滚动升级一个节点怎么做？**
   ① `allocation.enable=primaries` 暂停分配；② `POST _flush` 落盘；③ 停节点升级；④ 启动等加入集群；⑤ `allocation.enable=all` 恢复；⑥ green 后处理下一个。升级前必打快照，先升 master 再升 data，不能跨大版本滚动。

9. **节点重启后 recovery 很慢怎么办？**
   看分片是否太大（应 <50GB）、`indices.recovery.max_bytes_per_sec` 是否太小（生产调到 250~500mb）、是否一次重启太多节点。解决：调大 recovery 带宽和并发、单分片控制大小、一次一个节点重启。

10. **mapping 爆炸怎么处理？**
    `GET <index>/_mapping` 看字段数是否异常多（动态值当字段名导致）。治根：`dynamic: false/strict` 关动态映射；已爆炸的 reindex 到规范 mapping 的新索引；「以值为字段」改用 nested 结构。预防：监控索引字段数、字段命名规范。

11. **怎么下线一个 ES 节点不丢数据？**
    data 节点：`cluster.routing.allocation.exclude._name=<node>` 让分片迁走，等该节点分片数=0 再停。master 节点：先 `POST _cluster/voting_config_exclusions` 调整 quorum 再停（否则可能凑不齐 quorum 假死）。

12. **ES 怎么备份？**
    注册快照仓库（fs/s3），`PUT _snapshot/repo/snapshot_name` 打快照（增量、可异步）。恢复用 `POST _snapshot/repo/snapshot/_restore`，可 rename 避免覆盖。生产每天全量+频繁增量，定期在隔离环境演练恢复。升级/大变更前必打快照。

13. **ELK 各组件作用是什么？数据流怎么走？**
    E=Elasticsearch（存储检索）、L=Logstash（采集过滤转换，input→filter→output）、K=Kibana（可视化）。典型数据流：Filebeat（端采集日志）→ Logstash（过滤解析）→ Elasticsearch（存）→ Kibana（展示）。小场景 Filebeat 可直接写 ES 省掉 Logstash。

14. **Java 客户端有哪些？区别？**
    TransportClient（TCP，7.x弃8.x移）、RestHighLevelClient（HTTP，7.x主力）、Elasticsearch Java Client（构建者模式，8.x推荐）。生产7.x用 RestHighLevelClient，8.x用新 Java Client。面试常问：Transport 走 9300、REST 走 9200，官方推动 HTTP 方向。

15. **force_merge 什么时候用？有什么注意事项？**
    对只读/冷索引手动合并 segment，降 segment 数、释放被删文档占的空间。注意：① 非常耗 IO，低峰做；② 只对不再写入的冷索引做，热索引做了白做还会新增；③ max_num_segments=1 最省空间但代价最大；④ ILM 可自动触发。

16. **ES 慢查询怎么排查？完整步骤？**
    ① 开慢查询日志（search.slowlog.threshold）抓慢查询 DSL；② 用 `?profile=true` 看每个子句/分片耗时；③ 定位根因：wildcard/深分页/大聚合/script/大_source/分片太多；④ 对应优化：改match/用search_after/缩小聚合范围/加filter/用routing。

17. **索引别名有什么用？为什么生产建议用别名？**
    索引的软链接。用处：① reindex 时无感切换（原子删旧加新），业务零停机；② 一个别名指向多个索引，一次查多天数据；③ write alias + rollover 自动滚动建索引。业务代码永远访问别名，不直接访问真实索引名。

---

## 二十、资料勘误与重点提醒

1. **「只读锁会自动解除」是错的**：很多资料/教程说「磁盘降下来只读会自动恢复」。实际 `read_only_allow_delete` 触发后**即使磁盘下降也不会自动清除**，必须 `PUT` 手动解除。线上因这个误解导致业务长时间写不进的案例很多。

2. **「rejected 就调大队列」是常见误导**：不少博客建议「write rejected 就调大 `thread_pool.write.queue_size`」。这是治标且有害：队列越大积压越久、延迟越高，把快速失败变慢速失败，还可能拖垮上层超时。正确做法是降负载/扩容/优化，客户端重试退避。reject 是**保护机制不是缺陷**。

3. **「breaker 频繁触发就调大阈值」同理是错的**：调大 breaker 等于拆保险丝，迟早 OOM。应优化查询（缩小聚合范围、用 keyword 聚合、避免深分页 script）。

4. **dump heap 不会「无影响」**：资料常忽略 `jmap -dump` 会触发 Full GC + STW，生产节点 dump 会让服务暂停数秒到数十秒，且 dump 文件≈堆大小（31GB）。应在低峰或单节点做，确保磁盘足够。

5. **滚动升级「直接停节点重启」是错的**：必须先 `cluster.routing.allocation.enable=primaries` 暂停分配，否则重启时该节点分片被迁走、起来又迁回，造成无谓 recovery 风暴。资料常省略这步。

6. **「下线 master 直接停」会假死**：3 个 master 直接停 1 个没事，但若操作不当一次停 2 个或没调整 voting config，会凑不齐 quorum 假死。下线 master 应先 `voting_config_exclusions`。资料常不强调 quorum 风险。

7. **快照「恢复时自动覆盖」是错的**：恢复到已存在且在用的索引会失败，要用 `rename_pattern/rename_replacement` 改名恢复，验证后再切换别名。直接恢复到现网索引风险极大。

8. **「ILM 设了就高枕无忧」是误区**：ILM 常因 rollover 别名配置错误（没 `is_write_index: true`）、policy 条件没满足、某 step 报错而停滞。必须 `_ilm/explain` 监控状态，定期验证全流程。资料常只讲配置不讲排障。

9. **跨大版本不能滚动升级**：7.x->8.x 不能滚动（有 breaking change），需先升到 7.x 最新、用 upgrade assistant 检查、reindex 兼容、再全停升级。资料常说「滚动升级」不讲大版本限制。
10. **force_merge 不是越多越好，也不是越合并查询越快**：很多人误以为 segment 越少越快，有事没事就 force_merge 到 1 个 segment。实际上：① merge 非常耗 IO，生产会拖慢正常读写；② 少量 segment（如 3~5 个）对查询性能影响很小，完全没必要强行合并到 1 个；③ 热索引 force_merge 了也白搭，新 segment 还会不断生成。**force_merge 只适合只读/冷索引做存储优化**。
11. **慢查询日志级别开太低会打爆磁盘**：教程常教「开 trace 级别看所有查询」。但 trace 级别所有查询都记，高 QPS 下日志量爆炸，分分钟把日志盘写满。生产建议**默认开 warn 级别（如 5s）**，排查问题时临时对单索引调低，排查完立刻调回去。
12. **「_cat 命令就是用来开发用的，生产别看」是误解**：不少人觉得 _cat 是"玩具级"接口，生产要用 _nodes/stats 之类的 JSON API。实际上 _cat 系列是**生产运维排查的第一工具**——人眼可读、快速定位问题，比翻 JSON 高效得多。生产排查第一反应就应该是 _cat/health、_cat/nodes、_cat/shards 这三件套。
13. **「refresh_interval: -1 就是永远不 refresh」要注意恢复**：批量导入时设 `-1` 关闭 refresh 很常见，但很多人**导入完忘了改回来**，导致数据一直搜不到。生产操作要记着恢复，或者用 transient 临时配置（集群重启后自动恢复），比 persistent 安全。
14. **「磁盘只读锁会自动解除」是经典错误认知**：flood_stage 触发的 `read_only_allow_delete` 锁，**即使磁盘降到水位线以下也不会自动清除**，必须手动 PUT `index.blocks.read_only_allow_delete: null` 解锁。因这个误解导致业务长时间写不进的事故很多。（与第3条呼应，此处再次强调因为它是最常踩的坑）

---

> 本章把架构认知落地为排障流程。配合 05 章架构、06 章读写原理、07 章查询调优，构成 ES 从理论到运维的完整闭环：懂原理（05/06）-> 会用会调（07）-> 出事能救（08）。
