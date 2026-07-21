# Java 后端面试知识总结

6年Java工作经验相关知识总结，专注面试准备。

---

## 知识大纲

### 01 - Java基础

| 文件 | 核心内容 |
|------|---------|
| [01-集合框架.md](./01-java基础/01-集合框架.md) | Collection/Map接口、ArrayList/LinkedList、HashMap/ConcurrentHashMap、HashSet、TreeMap、LinkedHashMap、fail-fast |
| [02-并发编程.md](./01-java基础/02-并发编程.md) | 线程生命周期、线程池、synchronized、volatile、Lock、AQS、CAS、Atomic、ThreadLocal、死锁、JMM、happens-before |
| [03-IO与NIO.md](./01-java基础/03-IO与NIO.md) | BIO/NIO/AIO、字节流/字符流、序列化、Channel/Buffer/Selector、零拷贝、直接内存 |
| [04-Java8+新特性.md](./01-java基础/04-Java8+新特性.md) | Lambda、函数式接口、Stream、Optional、默认方法、新日期API、CompletableFuture |
| [05-其他核心知识点.md](./01-java基础/05-其他核心知识点.md) | 反射、动态代理、注解、泛型、异常、四大引用、双亲委派、SPI |

---

### 02 - JVM虚拟机

| 文件 | 核心内容 |
|------|---------|
| [00-JVM发展历程.md](./02-JVM虚拟机/00-JVM发展历程.md) | JVM家族发展史（Classic/Exact/HotSpot/JRockit/J9）、JDK时间线到JDK 23、主流JVM对比（HotSpot/OpenJ9/GraalVM/Azul）、关键技术演进总结、各JVM/GC名称由来 |
| [01-内存模型.md](./02-JVM虚拟机/01-内存模型.md) | 运行时数据区、堆/栈/方法区细节、永久代vs元空间、HotSpot对象探秘（创建/布局/访问）、TLAB、String.intern()变化 |
| [02-垃圾收集器与内存分配策略.md](./02-JVM虚拟机/02-垃圾收集器与内存分配策略.md) | 引用计数/可达性分析、三色标记法与漏标问题、四大GC算法、7大垃圾收集器详解（Serial/ParNew/Parallel/CMS/G1/ZGC）、ZGC三大黑科技、G1 Region分区、内存分配与回收策略、面试题整理 |
| [03-JDK性能监控与故障处理工具.md](./02-JVM虚拟机/03-JDK性能监控与故障处理工具.md) | jps/jstat/jinfo/jmap/jstack等命令行工具详解、jconsole/jvisualvm/JMC可视化工具、CPU飙高/OOM/死锁排查实战流程、高频面试题 |
| [04-Class类文件结构.md](./02-JVM虚拟机/04-Class类文件结构.md) | Class文件本质、魔数/版本号/常量池/访问标志、字段表/方法表、Code属性、字节码指令集、javap使用、完整面试题 |
| [05-虚拟机类加载机制.md](./02-JVM虚拟机/05-虚拟机类加载机制.md) | 类加载5阶段详解、三种类加载器、双亲委派模型原理与好处、三次破坏双亲委派场景、自定义类加载器、高频面试题 |
| [06-Java内存模型与线程.md](./02-JVM虚拟机/06-Java内存模型与线程.md) | JMM主内存/工作内存、三大特性（原子性/可见性/有序性）、volatile原理、happens-before8大规则、线程6种状态转换、7道面试题 |
| [07-性能调优实战.md](./02-JVM虚拟机/07-性能调优实战.md)  | GC日志分析、JVM参数调优、常见问题排查案例 |

---

### 03 - Spring全家桶

| 文件 | 核心内容 |
|------|---------|
| [01-Spring核心.md](./03-Spring全家桶/01-Spring核心.md)  | IoC容器、Bean生命周期、依赖注入、AOP原理、事务、Spring事务传播机制 |
| [02-SpringBoot.md](./03-Spring全家桶/02-SpringBoot.md)  | 自动配置原理、启动流程、Starter、配置文件加载顺序、条件注解 |
| [03-SpringCloud.md](./03-Spring全家桶/03-SpringCloud.md)  | 服务注册与发现、服务调用、配置中心、网关、熔断降级、分布式链路追踪 |

---

### 04 - 数据库（OLTP）

| 文件 | 核心内容 |
|------|---------|
| [01-MySQL基础.md](./04-数据库/01-MySQL基础.md)  | 存储引擎、数据类型、索引类型、约束、视图、触发器 |
| [02-索引与优化.md](./04-数据库/02-索引与优化.md)  | B+树索引、索引底层原理、聚簇索引/非聚簇索引、回表、覆盖索引、最左前缀原则、索引失效场景、Explain |
| [03-事务与锁.md](./04-数据库/03-事务与锁.md)  | ACID、隔离级别、MVCC、锁分类、行锁/表锁、Gap锁、意向锁、死锁 |
| [04-SQL优化.md](./04-数据库/04-SQL优化.md)  | 慢查询分析、SQL优化技巧、分页优化、Join优化、子查询优化 |

---

### 05 - OLAP数据库

| 文件 | 核心内容 |
|------|---------|
| [01-ClickHouse-基础与部署.md](./05-OLAP数据库/01-ClickHouse-基础与部署.md) | ClickHouse概述、20节点生产集群部署方案、ZooKeeper配置、10+种表引擎详解与选型 |
| [02-ClickHouse-原理与优化.md](./05-OLAP数据库/02-ClickHouse-原理与优化.md) | 副本分片内部原理、写入/查询流程详解、索引原理、物化视图、主键/分区/排序键设计、性能优化实践 |
| [03-ClickHouse-实战与运维.md](./05-OLAP数据库/03-ClickHouse-实战与运维.md) | 日志存储场景、可观测性指标存储、常见问题排查、故障处理、运维最佳实践 |

---

### 06 - 分布式系统

| 文件 | 核心内容 |
|------|---------|
| [01-分布式理论基础.md](./05-分布式系统/01-分布式理论基础.md)  | CAP、BASE、一致性协议（2PC、3PC、Paxos、Raft）、分布式ID |
| [02-缓存策略.md](./05-分布式系统/02-缓存策略.md)  | 缓存作用、缓存穿透/击穿/雪崩、缓存更新策略、缓存一致性、热点Key |
| [03-消息队列.md](./05-分布式系统/03-消息队列.md)  | MQ作用、消息丢失/重复/顺序性、消息堆积、幂等性 |
| [04-分布式事务.md](./05-分布式系统/04-分布式事务.md)  | 2PC、TCC、本地消息表、可靠消息最终一致性、最大努力通知、Seata |
| [05-微服务架构.md](./05-分布式系统/05-微服务架构.md)  | 服务治理、服务容错、网关、配置中心、链路追踪 |

---

### 07 - 中间件

| 文件 | 核心内容 |
|------|---------|
| [01-Kafka入门.md](./06-中间件/01-Kafka入门.md) | 初识Kafka、消息队列与JMS规范、生产者-消费者模式、消息中间件对比、ZooKeeper与KRaft模式、快速上手 |
| [02-Kafka基础.md](./06-中间件/02-Kafka基础.md) | 集群部署与启动（Broker/Controller选举）、创建主题（Topic/Partition/Replica/Leader-Follower/Log）、生产消息（三组件/发送方式/分区/ACK/幂等与事务/传输语义）、存储消息（文件格式/稀疏索引/HW-LEO-ISR/数据一致性）、消费消息（消费者组/Coordinator/分配策略/Offset管理） |
| [03-Kafka进阶.md](./06-中间件/03-Kafka进阶.md) | Controller选举与防脑裂(epoch)、Broker上下线与Leader重选举、数据偏移量定位(Segment/稀疏索引/跳跃表)、Topic删除、日志清理与压缩(delete/compact/墓碑)、页缓存、零拷贝(sendfile/mmap)、顺序写日志、KRaft模式 |
| [04-Nginx.md](./06-中间件/04-Nginx.md) | Nginx四大能力(HTTP服务器/反向代理/负载均衡/动静分离)、正向vs反向代理、高性能原理(epoll/master-worker)、worker进程控制(worker_processes/worker_connections/最大并发计算/CPU亲和性)、负载均衡策略(轮询/权重/ip_hash/url_hash)与配置示例(upstream/proxy_pass/server附加参数)、核心配置、与Tomcat配合 |

---

### 08 - 算法与数据结构

| 文件 | 核心内容 |
|------|---------|
| [高频面试题.md](./07-算法与数据结构/高频面试题.md)  | 链表、树、栈/队列、哈希、二分查找、双指针、滑动窗口、动态规划、回溯、贪心 |

---

### 09 - 设计模式

| 文件 | 核心内容 |
|------|---------|
| [常用设计模式.md](./08-设计模式/常用设计模式.md)  | 单例、工厂、抽象工厂、建造者、原型、适配器、装饰器、代理、策略、观察者、模板方法、责任链 |

---

### 10 - 系统设计与架构

| 文件 | 核心内容 |
|------|---------|
| [经典场景设计.md](./09-系统设计与架构/经典场景设计.md)  | 秒杀系统、限流、降级、熔断、分库分表、读写分离、分布式锁、幂等性设计、高可用架构 |

---

### 11 - 监控与可观测性

| 文件 | 核心内容 |
|------|---------|
| [01-Prometheus指标详解.md](./10-监控与可观测性/01-Prometheus指标详解.md) | 4大核心指标（Counter/Gauge/Histogram/Summary）、适用场景对比、P50/P95/P99计算、常见面试题 |
| [02-OpenTelemetry.md](./10-监控与可观测性/02-OpenTelemetry.md) | 可观测性统一标准(CNCF项目/厂商锁定痛点)、三大支柱(Traces/Metrics/Logs)、核心概念(Trace与Span/上下文传播/Resource/埋点/Baggage/语义约定)、架构(API+SDK+Collector)、与Prometheus/Jaeger关系 |

#### SkyWalking 系列（`10-监控与可观测性/`，共20章）

| 文件 | 核心内容 |
|------|---------|
| [03-SkyWalking-初识.md](./10-监控与可观测性/03-SkyWalking-初识.md) | APM本质与演进、SkyWalking定位与历史、架构全景（Agent+OAP+UI+Satellite+Rover）、与Pinpoint/Jaeger/Zipkin/CAT/OpenTelemetry矩阵对比 |
| [04-SkyWalking-快速上手.md](./10-监控与可观测性/04-SkyWalking-快速上手.md) | 环境准备、OAP单机部署、Java Agent接入、第一个Trace、UI初览、SpringBoot集成 |
| [05-SkyWalking-核心概念与数据模型.md](./10-监控与可观测性/05-SkyWalking-核心概念与数据模型.md) | Service/ServiceInstance/Endpoint三层模型、Trace/Span/Segment关系与区别、Tags/Logs/Events、OperationName与端点发现、Layer分层、Component组件定义 |
| [06-SkyWalking-指标体系与分类.md](./10-监控与可观测性/06-SkyWalking-指标体系与分类.md) | Service指标(Apdex/Cpm/SLA/RT/Throughput)、Instance JVM指标(内存/GC/线程/CPU)、Endpoint指标(QPS/P50/P95/P99)、Relation双向指标、Meter自定义指标(Counter/Gauge/Histogram)、百分位计算 |
| [07-SkyWalking-Trace传播协议.md](./10-监控与可观测性/07-SkyWalking-Trace传播协议.md) | sw8协议详解(Header编码/解码)、跨进程传播(ContextCarrier)、跨线程传播(ContextSnapshot)、跨语言传播、TraceId/SegmentId/SpanId生成规则、忽略端点与Trace忽略 |
| [08-SkyWalking-探针原理.md](./10-监控与可观测性/08-SkyWalking-探针原理.md) | Java Agent启动机制(premain)、字节码增强(ByteBuddy)、类加载隔离(AgentClassLoader/PluginClassLoader)、拦截点定义(AbstractClassEnhancePluginDefine)、ContextManager状态机 |
| [09-SkyWalking-插件体系.md](./10-监控与可观测性/09-SkyWalking-插件体系.md) | 内置插件全景(HTTP/RPC/DB/Cache/MQ)、插件增强四要素、可选插件vs Boot插件、自定义插件开发完整实战 |
| [10-SkyWalking-OAP架构.md](./10-监控与可观测性/10-SkyWalking-OAP架构.md) | 模块化设计(ModuleDefine/ModuleProvider)、集群角色(Mixed/Receiver/Aggregator)、水平扩展策略、启动流程(BootstrapFlow)、gRPC/HTTP服务 |
| [11-SkyWalking-数据采集与传输.md](./10-监控与可观测性/11-SkyWalking-数据采集与传输.md) | 上报协议(gRPC/Kafka/HTTP)、Agent注册流程、DataCarrier(RingBuffer/背压)、批量处理(TraceBuffer/LogBuffer)、连接管理 |
| [12-SkyWalking-数据计算与流式处理.md](./10-监控与可观测性/12-SkyWalking-数据计算与流式处理.md) | OAL引擎(语法/编译/执行)、L1聚合(Agent端)、L2聚合(OAP端/TraceAnalyzer/监听器)、三级降采样(L1/L2/L3)、TopN计算(最小堆)、慢查询检测、热力图 |
| [13-SkyWalking-存储引擎.md](./10-监控与可观测性/13-SkyWalking-存储引擎.md) | H2(内存/文件)、MySQL(按时间分表/TTL)、Elasticsearch(索引模板/分片/滚动索引)、BanyanDB(自研列式存储/LSM/压缩)、OpenSearch、TTL数据清理、存储选型对比矩阵 |
| [14-SkyWalking-UI与可视化.md](./10-监控与可观测性/14-SkyWalking-UI与可视化.md) | Dashboard(全局/服务/实例/端点)、拓扑图(服务依赖图)、Trace详情(调用树/时间轴)、日志面板(Trace-Log关联)、Profiling火焰图、自定义仪表盘 |
| [15-SkyWalking-告警与通知.md](./10-监控与可观测性/15-SkyWalking-告警与通知.md) | 告警规则引擎(yarldsl)、告警生命周期(触发→持续→恢复)、Webhook/钉钉/企微/飞书通知、动态配置(Apollo/Nacos/ZK)、与AlertManager集成 |
| [16-SkyWalking-OpenTelemetry对接.md](./10-监控与可观测性/16-SkyWalking-OpenTelemetry对接.md) | OTel协议兼容(OTLP Receiver)、三种混合架构方案、sw8 vs W3C TraceContext、迁移策略、与Prometheus/PromQL集成 |
| [17-SkyWalking-浏览器监控与ServiceMesh.md](./10-监控与可观测性/17-SkyWalking-浏览器监控与ServiceMesh.md) | Browser Agent(Web Vitals/错误收集/XHR追踪)、Service Mesh(Istio/Envoy ALS)、Kubernetes(SWCK Operator)、Satellite边车网关、eBPF Rover |
| [18-SkyWalking-MAL与日志分析.md](./10-监控与可观测性/18-SkyWalking-MAL与日志分析.md) | MAL(Meter Analysis Language)、LAL(Log Analysis Language/日志解析/结构化)、日志桥接(Logback/Log4j2→MDC注入)、MQE(Metrics Query Engine) |
| [19-SkyWalking-源码分析.md](./10-监控与可观测性/19-SkyWalking-源码分析.md) | Agent端(premain→PluginFinder→ByteBuddy→TracingContext→上报)、OAP端(模块启动→DataCarrier→TraceAnalyzer→OAL→存储)、端到端代码走读 |
| [20-SkyWalking-高级特性.md](./10-监控与可观测性/20-SkyWalking-高级特性.md) | 采样策略(全采样/强制/慢SQL)、Profiling(On-CPU线程栈/火焰图)、eBPF Rover、Event事件系统、Namespace多租户、TLS/mTLS、swctl CLI |
| [21-SkyWalking-生产实践.md](./10-监控与可观测性/21-SkyWalking-生产实践.md) | 大规模部署(1000+Agent)、OAP JVM调优(G1GC)、ES调优(索引/分片/refresh_interval)、常见问题排查(Agent不上报/拓扑不全/内存溢出)、OAP Telemetry自监控、与Prometheus+Grafana+Loki协同 |
| [22-SkyWalking-版本演进与生态.md](./10-监控与可观测性/22-SkyWalking-版本演进与生态.md) | v5→v10演进历史、v8(MAL/LAL)→v9(OTel/UI/MQE)→v10(BanyanDB/分层拓扑)关键变化、Apache治理、生态工具链、知名用户、技术趋势 |

---

### 12 - 云原生

| 文件 | 核心内容 |
|------|---------|
| [01-Docker基础与核心概念.md](./11-云原生/01-Docker基础与核心概念.md) | Docker概述、容器vs虚拟机、三大核心概念（镜像/容器/仓库）、架构、底层技术（Namespace/Cgroups/Union FS）、写时复制机制、常见面试题 |
| [02-Docker常用命令与实践.md](./11-云原生/02-Docker常用命令与实践.md) | 镜像/容器/网络/数据卷常用命令详解、docker run流程、exec vs attach、排错方法、资源限制、生产环境最佳实践 |
| [03-Dockerfile详解与镜像构建.md](./11-云原生/03-Dockerfile详解与镜像构建.md) | Dockerfile 14大指令详解、多阶段构建、.dockerignore、镜像优化、构建缓存原理、常见面试题 |
| [04-Kubernetes基础.md](./11-云原生/04-Kubernetes基础.md) | K8s概述与Docker关系、集群架构(控制平面apiserver/etcd/scheduler/controller-manager+工作节点kubelet/kube-proxy)、声明式API与控制循环、核心资源(Node/Pod/Label/ReplicaSet/Deployment/Service/Namespace/StatefulSet/DaemonSet/Ingress/ConfigMap/PV-PVC)、DaemonSet详解(每节点一个Pod+日志采集挂载/var/log+监控node-exporter+分层监控)、Service IP与端口关联(PodIP/ClusterIP/NodeIP三类IP+port/targetPort/nodePort三段链路)、节点亲和性原理(硬/软亲和性+调度器Filter/Score两阶段+与Taint/Pod亲和对比)、资源管理(requests/limits原理+Cgroups CPU限流与内存OOM+QoS三等级)、应用部署配置全景(资源清单/ConfigMap-Secret/RBAC/PVC/Ingress+Helm)、kubectl命令与资源清单、高频面试题(含Operator原理=CRD+自定义控制器) |

---

### 13 - 大数据

| 文件 | 核心内容 |
|------|---------|
| [01-HDFS.md](./12-大数据/01-HDFS.md) | HDFS分布式文件系统 |
| [02-YARN.md](./12-大数据/02-YARN.md) | YARN资源调度框架 |
| [03-Hadoop生态.md](./12-大数据/03-Hadoop生态.md) | Hadoop生态系统全景 |
#### Flink 系列（`12-大数据/Flink/`，共13章）

| 文件 | 核心内容 |
|------|---------|
| [01-初识Flink.md](./12-大数据/Flink/01-初识Flink.md) | Flink定位、流批一体、Lambda架构、Flink vs Spark、分层API |
| [02-快速上手.md](./12-大数据/Flink/02-快速上手.md) | 开发流程、批处理vs流处理代码差异、输出前缀子任务编号 |
| [03-部署.md](./12-大数据/Flink/03-部署.md) | Session/Per-Job/Application三种模式、Standalone/YARN/K8s、Standalone HA vs YARN HA |
| [04-运行时架构.md](./12-大数据/Flink/04-运行时架构.md) | JobManager四组件、四层调度图、并行度、算子链Operator Chain、任务槽与槽共享 |
| [05-DataStream-API基础.md](./12-大数据/Flink/05-DataStream-API基础.md) | 执行环境、Source、转换算子、keyBy原理、max vs maxBy、rebalance vs rescale、RichFunction生命周期、Sink |
| [06-时间与窗口.md](./12-大数据/Flink/06-时间与窗口.md) | 时间语义、水位线Watermark(生成/传递取最小/减1ms/空闲源)、窗口(滚动/滑动/会话/增量vs全窗口)、迟到数据三道防线 |
| [07-处理函数.md](./12-大数据/Flink/07-处理函数.md) | ProcessFunction、KeyedProcessFunction与定时器Timer、侧输出流、TopN案例 |
| [08-多流转换.md](./12-大数据/Flink/08-多流转换.md) | Union vs Connect、Window Join、Interval Join、Window CoGroup |
| [09-状态编程.md](./12-大数据/Flink/09-状态编程.md) | Keyed State vs Operator State、五种状态类型、状态TTL、状态后端HashMap vs RocksDB |
| [10-容错机制.md](./12-大数据/Flink/10-容错机制.md) | Checkpoint与Chandy-Lamport算法、Barrier对齐与非对齐、Savepoint区别、状态一致性、端到端精确一次、Flink+Kafka两阶段提交 |
| [11-Table-API与SQL.md](./12-大数据/Flink/11-Table-API与SQL.md) | 动态表与持续查询、更新查询vs追加查询、Append/Retract/Upsert编码、窗口TVF、UDF |
| [12-Flink-CEP.md](./12-大数据/Flink/12-Flink-CEP.md) | 复杂事件处理、NFA状态机、next/followedBy/followedByAny、超时与迟到处理 |
| [13-性能与调优.md](./12-大数据/Flink/13-性能与调优.md) | 反压Credit-Based流控、Flink内存模型、数据倾斜两阶段聚合、面试核心知识地图速记 |

---

### 14 - AI与大模型

> AI/Agent 专题。覆盖 LLM 基础与提示范式、RAG 与向量检索、Agent 核心原理（记忆/上下文/反思/能力体系）、多 Agent 架构、Agent 开发框架、Agent 通信协议生态（MCP/A2A/ACP/AG-UI）、ClaudeCode 与 Harness 原理、工程化与生产部署。

| 文件 | 核心内容 |
|------|---------|
| [01-LLM基础与提示工程.md](./13-AI与大模型/01-LLM基础与提示工程.md) | LLM定义/涌现能力、训练三段式(预训练/SFT/RLHF-DPO)、Token/上下文窗口/幻觉、能力边界、提示工程原则、推理范式(CoT/ToT/GoT/Self-Consistency/ReAct/Reflexion/Plan-Execute/Loop)对比 |
| [02-RAG与向量检索.md](./13-AI与大模型/02-RAG与向量检索.md) | RAG三阶段/三代演进(Naive->Advanced->Modular)、Chunking、Embedding、向量数据库选型(Milvus/Faiss/Pinecone/Qdrant等)、HNSW/IVF、混合检索、Rerank、Self-RAG/CRAG/RAPTOR/GraphRAG/LightRAG/Agentic RAG、LlamaIndex、RAGAS评测、上下文压缩 |
| [03-Agent核心原理.md](./13-AI与大模型/03-Agent核心原理.md) | Agent本质(Agent vs Chatbot vs Workflow)、Agent Loop、上下文工程(写入/选择/压缩/隔离/遗忘)、长短期记忆(工作/情景/语义/程序)、任务规划、工具调用(Function Calling与MCP关系)、自我反思、四层Agent能力体系(基础/核心/工程部署/安全合规) |
| [04-多Agent架构.md](./13-AI与大模型/04-多Agent架构.md) | 单Agent瓶颈、四种多Agent架构(层级监督者/网络协作/竞争辩论/编排流水线)对比、通信机制(直接调用/共享State/消息总线/A2A-ACP协议)、MetaGPT/AutoGen/CrewAI/AgentScope/LangGraph案例、设计取舍 |
| [05-Agent开发框架.md](./13-AI与大模型/05-Agent开发框架.md) | LangChain全家桶(LangChain/LangGraph/LangSmith/LangServe)、LangGraph State/Node/Edge/Checkpointer、AgentScope(阿里分布式)、AutoGen(对话式)、CrewAI(角色驱动)、Skill机制(渐进式披露)、框架选型矩阵 |
| [06-协议生态-MCP-A2A-ACP-AGUI.md](./13-AI与大模型/06-协议生态-MCP-A2A-ACP-AGUI.md) | SSE原理(与WebSocket/轮询对比)、MCP(Host/Client/Server架构/Tools-Resources-Prompts/stdio与Streamable HTTP/与Function Calling关系)、A2A(Google/Agent Card/Task/与MCP互补)、ACP(异步消息企业级)、AG-UI(Agent与前端交互)、协议三角互补矩阵 |
| [07-ClaudeCode与Harness原理.md](./13-AI与大模型/07-ClaudeCode与Harness原理.md) | Harness定义(运行时编排/模型是大脑Harness是身体)、Agent Loop真实运行、自动压缩Auto-Compact、子Agent隔离Task、多Agent并行Workflow、工具系统(MCP扩展)、Skill落地、权限沙箱、Hooks、计划模式、与Cursor/Cline/Devin对比 |
| [08-Agent工程化与生产部署.md](./13-AI与大模型/08-Agent工程化与生产部署.md) | 可视化编排(Dify/Coze/FastGPT/LangFlow)、LLM可观测(Trace/Token/延迟/LangSmith-Langfuse-Phoenix)、评测Eval(LLM-as-Judge/RAGAS/在线AB)、API网关、多模型路由(级联/降级/缓存)、Agent注册中心与灰度发布、安全合规(内容审核/权限RBAC/Prompt注入防护/PII脱敏/Agent特有风险)、生产部署架构 |

---

## 使用说明

- 每个文件先列出核心概念和常见面试题的**大纲**
- 具体内容留空，待每天学习时逐步补充完善
- 文件结构统一：「核心概念」 + 「常见面试题」
