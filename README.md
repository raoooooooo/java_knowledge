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
| [06-Java高频面试题集.md](./01-java基础/06-Java高频面试题集.md) | Java基础(装箱缓存/值传递/反射/异常/String不可变)、集合(ArrayList扩容/HashMap底层1.7-1.8/ConcurrentHashMap分段锁演进/fail-fast)、并发(synchronized锁升级/volatile不保证原子性/AQS双向队列/CAS与ABA/线程池七参四策略/ThreadLocal内存泄漏/死锁四条件/JMM三大特性happens-before)、JVM(运行时数据区/堆分代/GC Roots三色标记/四大GC算法/CMS-G1-ZGC/类加载5阶段双亲委派及三次破坏/OOM排查/CPU飙高排查)、资料勘误(Integer缓存非常量池/String底层byte[]/HashMap树化双条件/AQS双向链表/元空间在本地内存等25+处) |

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

#### Spring核心系列（`03-Spring全家桶/`）

| 文件 | 核心内容 |
|------|---------|
| [01-Spring核心-IoC与Bean生命周期.md](./03-Spring全家桶/01-Spring核心-IoC与Bean生命周期.md) | IoC思想(控制反转什么)、容器体系(BeanFactory vs ApplicationContext懒加载vs预实例化)、BeanDefinition图纸、Bean完整生命周期13步(BeanFactoryPostProcessor->实例化->属性填充->Aware->BeanPostProcessor前置->@PostConstruct->afterPropertiesSet->init-method->BeanPostProcessor后置/AOP在此->销毁)、Bean注册方式(@Component/@Bean/@Import三用法/FactoryBean)、核心扩展点 |
| [02-Spring核心-循环依赖与三级缓存.md](./03-Spring全家桶/02-Spring核心-循环依赖与三级缓存.md) | 循环依赖类型与能否解决(构造器/Setter/字段/prototype)、三级缓存源码(singletonObjects/earlySingletonObjects/singletonFactories)、A->B->A完整时序图、为什么三级而非二级(AOP延迟代理+循环依赖折中)、@Lazy解法、Spring Boot 2.6默认禁循环依赖、@Async不实现getEarlyBeanReference的坑 |
| [03-Spring核心-AOP原理.md](./03-Spring全家桶/03-Spring核心-AOP原理.md) | AOP术语(JoinPoint/Pointcut/Advice/Aspect/Weaving)、5种通知与执行顺序(Spring 5.2.7前后@After位置差异)、JDK动态代理vs CGLIB(FastClass机制)、选择策略(Spring Boot 2.0+默认CGLIB)、AnnotationAwareAspectJAutoProxyCreator实现原理、拦截器链责任链、代理失效场景(this内部调用/static/final/private)、AopContext.currentProxy()解法、Spring AOP vs AspectJ |
| [04-Spring核心-事务管理.md](./03-Spring全家桶/04-Spring核心-事务管理.md) | 声明式事务原理(@Transactional+TransactionInterceptor+AOP)、三大接口(PlatformTransactionManager/TransactionDefinition/TransactionStatus)、7种传播行为(REQUIRED/REQUIRES_NEW/NESTED重点)、隔离级别、默认回滚规则(RuntimeException+Error回滚checked不回滚)、5大失效场景(非public/自调用/异常吞/rollbackFor不匹配/MyISAM)、多线程事务失效 |
| [05-SpringBoot自动配置与启动原理.md](./03-Spring全家桶/05-SpringBoot自动配置与启动原理.md) | 约定优于配置、@SpringBootApplication拆解、自动配置全链路(@EnableAutoConfiguration->AutoConfigurationImportSelector)、spring.factories->AutoConfiguration.imports版本演变(2.7引入/3.0移除自动配置/其他扩展点仍用factories)、条件注解、SpringApplication.run启动流程、应用类型推断、refresh()关键步骤、内嵌Tomcat在onRefresh启动、Runner执行时机 |
| [06-SpringBoot-Starter与配置体系.md](./03-Spring全家桶/06-SpringBoot-Starter与配置体系.md) | Starter机制与组成、官方(spring-boot-starter-xxx)vs第三方(xxx-spring-boot-starter)命名规范、自定义Starter完整步骤、配置文件properties vs YAML、配置加载优先级(命令行>系统属性>环境变量>外部>内部)、Profile多环境、@Value vs @ConfigurationProperties(松散绑定/JSR303校验)、@PropertySource不支持YAML、@RefreshScope热刷新 |
| [07-SpringCloud-注册发现与服务调用.md](./03-Spring全家桶/07-SpringCloud-注册发现与服务调用.md) | SpringCloud定位与版本命名(地铁站名->2020.0日历版)、Netflix停更现状(Eureka1.x维护/Hystrix停更/Ribbon停更LoadBalancer替代/Zuul->Gateway)、Eureka(AP/peer复制/自我保护85%阈值)、Nacos(临时AP-Distro/永久CP-Raft/默认AP/推送vs拉取)、OpenFeign原理(FactoryBean->JDK代理->SynchronousMethodHandler)、OpenFeign vs Dubbo、客户端vs服务端负载均衡 |
| [08-SpringCloud-网关与熔断降级.md](./03-Spring全家桶/08-SpringCloud-网关与熔断降级.md) | API网关职责、Zuul1.x同步阻塞/Zuul2.x异步未进SpringCloud/Gateway基于WebFlux+Netty异步非阻塞、Gateway核心概念(Route/Predicate/Filter)与工作原理、熔断器三状态(CLOSED/OPEN/HALF_OPEN)、雪崩效应、4种限流算法(令牌桶支持突发vs漏桶匀速)、Hystrix/Resilience4j/Sentinel对比(Sentinel核心是流量控制+系统自适应限流)、Nacos长轮询配置热刷新、Seata四种模式与三角色、Sleuth被移除->Micrometer Tracing |
| [09-Spring事务高频场景题.md](./03-Spring全家桶/09-Spring事务高频场景题.md) | 长事务核心场景(事务内调第三方接口超时->连接占用/锁不释放/连接池耗尽/超时回滚致数据不一致/timeout失效)、同类场景(事务内发MQ消息丢失/调RPC/循环调接口/CPU耗时/REQUIRES_NEW双连接/Redis缓存操作)、@Transactional timeout为何不生效、通用解决框架(缩小事务边界/编程式事务/异步化MQ/本地消息表)、事务内能否做X判断表、资料勘误(timeout是错觉/DB回滚无法撤销MQ/REQUIRES_NEW连接占用) |

---

### 04 - 数据库

> 数据库大类：含关系型 OLTP（MySQL）与搜索型/文档型 NoSQL（Elasticsearch）。

| 文件 | 核心内容 |
|------|---------|
| [01-MySQL基础.md](./04-数据库/01-MySQL基础.md)  | 存储引擎、数据类型、索引类型、约束、视图、触发器 |
| [02-索引与优化.md](./04-数据库/02-索引与优化.md)  | B+树索引、索引底层原理、聚簇索引/非聚簇索引、回表、覆盖索引、最左前缀原则、索引失效场景、Explain |
| [03-事务与锁.md](./04-数据库/03-事务与锁.md)  | ACID、隔离级别、MVCC、锁分类、行锁/表锁、Gap锁、意向锁、死锁 |
| [04-SQL优化.md](./04-数据库/04-SQL优化.md)  | 慢查询分析、SQL优化技巧、分页优化、Join优化、子查询优化 |
| [01-MySQL高频面试题集.md](./04-数据库/01-MySQL高频面试题集.md)  | MySQL 九大主题高频面试题速查：基础架构/索引/事务/锁/日志/SQL优化/存储引擎/主从复制/其他，每题要点答案+易错点勘误 |
#### Elasticsearch 系列（`04-数据库/`）

| 文件 | 核心内容 |
|------|---------|
| [05-Elasticsearch基础与架构.md](./04-数据库/05-Elasticsearch基础与架构.md) | ES定位(搜索引擎+文档NoSQL)、概念模型(Index/Doc/Mapping/Shard/Replica对照MySQL)、节点角色(Master/Data/Coordinating/Ingest/7.10+数据层data_hot/warm/cold/frozen/content)、冷热分层架构、专用Master、主分片与副本、路由公式(hash(routing)%主分片数)、集群健康状态、选主与脑裂(zen2 term机制/7.0+自动规避/假死场景)、Cluster State与两阶段发布、Discovery选主、分片分配(Allocation Deciders/Awareness/Filter/Balancer)、Recovery机制(阶段/限流/滚动重启)、线程池模型、内存模型(Heap组成/Page Cache/Circuit Breaker)、生产部署与容量规划、监控指标体系、安全机制(RBAC/TLS/审计)、ES vs MySQL、MySQL同步ES(Canal/binlog) |
| [06-Elasticsearch索引与读写原理.md](./04-数据库/06-Elasticsearch索引与读写原理.md) | 倒排索引(Term Dictionary/Term Index FST/Posting List压缩FOR+RoaringBitmap)、Doc Values列式正排、分词器Analyzer三段式(char filter/tokenizer/token filter)与ik中文分词、写入分词vs搜索分词、Mapping(text vs keyword/动态vs显式/字段类型不可改)、写入流程(refresh/flush/translog/NRT/segment不可变+merge+标记删除)、读取流程(query then fetch两阶段)、相关性打分(TF-IDF->BM25) |
| [07-Elasticsearch查询与调优.md](./04-数据库/07-Elasticsearch查询与调优.md) | Query DSL(query vs filter缓存)、bool四子句(must/filter/should/must_not)、term查text搜不到坑、聚合(Bucket/Metric/Pipeline)、深分页(from+size/scroll/search_after/PIT)、索引/查询/写入/JVM调优、集群运维(健康状态/未分配原因/Recovery/ILM脑裂)、写放大回顾(5~10x/与BanyanDB对比)、ES vs ClickHouse vs MySQL选型、资料勘误(type已弃用等) |
| [08-Elasticsearch运维与故障排查.md](./04-数据库/08-Elasticsearch运维与故障排查.md) | 排查方法论与工具箱、集群Red/Yellow排查(allocation/explain未分配原因)、节点OOM/假死、磁盘水位线三水位(flood只读锁需手动解除)、GC停顿(Long GC/heap dump)、Thread Pool rejected(write/search/get)、写入慢/拒绝(bulk/refresh/merge)、慢查询(慢日志/profile API)、Heap占用高(各cache/jmap+MAT)、Circuit Breaker触发、Recovery慢、Mapping爆炸/Cluster State膨胀、热点分片/数据倾斜(hot_threads)、网络分区/Master假死、滚动升级与节点维护(voting_config_exclusions/drain)、快照备份恢复(snapshot/restore)、ILM故障、运维向面试题、资料勘误(只读锁不自动解除/reject勿调大队列/breaker勿调大阈值等) |

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
| [05-MyBatis.md](./06-中间件/05-MyBatis.md) | MyBatis定位(半自动ORM)与JDBC/Hibernate对比、核心组件架构(SqlSessionFactory/SqlSession/Executor/四大Handler)与查询执行时序、Mapper接口绑定原理(JDK动态代理/MapperProxy/namespace+id)、#{}预编译防注入vs${}拼接、动态SQL标签(if/choose/where/set/foreach)、一级缓存(SqlSession级/Spring下几乎失效/@Transactional内短暂生效)与二级缓存(namespace级/需Serializable/多表脏读/分布式建议关闭下沉Redis)、插件原理(JDK动态代理责任链拦截四大对象)、PageHelper分页(ThreadLocal/物理vs逻辑分页)、Spring集成(@MapperScan/MapperFactoryBean/SqlSessionTemplate)、N+1问题、资料勘误 |

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

### 15 - Agent开发面试实战

> Java 工程师视角的 AI Agent 开发面试指南。与「13-AI与大模型」互补：13 章讲原理，本系列讲 Java 实战(Spring AI / LangChain4j) + 高频面试题 + 生产踩坑。从入门到精通，6 篇 28 章。

| 文件 | 核心内容 |
|------|---------|
| [00-总览与学习路径.md](./14-Agent开发面试实战/00-总览与学习路径.md) | 系列定位(Java视角/面试导向/与13章互补)、28章大纲、6-8周学习路径、面试能力地图(必考/高频/中频考点分层)、Java Agent技术栈全景、名词约定 |
| [01-Agent开发全景与面试能力地图.md](./14-Agent开发面试实战/01-Agent开发全景与面试能力地图.md) | 岗位现状(Java面试AI占比)、Java工程师为何学Agent(AI工程化优势)、与传统后端差异(确定性vs非确定性)、面试能力地图、高频面试题 |
| [02-大模型API调用与参数调优.md](./14-Agent开发面试实战/02-大模型API调用与参数调优.md) | 主流模型API对比(OpenAI/Anthropic/Qwen/DeepSeek/GLM)、核心参数(temperature/top_p/max_tokens/seed)、流式SSE与Java处理、Token计费、多模态、Java客户端实战(Spring AI/OkHttp/WebFlux)、高频面试题 |
| [03-提示工程实战与工程化.md](./14-Agent开发面试实战/03-提示工程实战与工程化.md) | Prompt结构化模板(角色/任务/格式/示例)、Few-shot/CoT、JSON/XML结构化输出、Prompt模板版本化管理、Prompt注入与防御、高频面试题 |
| [04-结构化输出与函数调用.md](./14-Agent开发面试实战/04-结构化输出与函数调用.md) | JSON Schema与Structured Output、Function Calling/Tool Use流程、并行工具调用、与MCP关系、Spring AI @Tool与LangChain4j @Tool实战、高频面试题 |

#### 第二篇 · RAG 检索增强（`14-Agent开发面试实战/`）

| 文件 | 核心内容 |
|------|---------|
| [05-RAG基础与离线建库.md](./14-Agent开发面试实战/05-RAG基础与离线建库.md) | RAG全流程、文档加载与切分(固定/语义/递归/父子块)、Embedding选型(BGE/通义/Cohere)、向量库选型(Milvus/Pgvector/ES/Qdrant)、Spring AI VectorStore与LangChain4j实战 |
| [06-RAG检索与召回优化.md](./14-Agent开发面试实战/06-RAG检索与召回优化.md) | 稠密/稀疏/混合检索、Rerank重排(bge-reranker)、查询改写/HyDE/查询路由、元数据过滤、Top-K调优、上下文压缩(LLMLingua)、高频面试题 |
| [07-RAG进阶架构.md](./14-Agent开发面试实战/07-RAG进阶架构.md) | Self-RAG/CRAG自适应纠错、RAPTOR递归摘要、GraphRAG/LightRAG图增强、Agentic RAG(Agent化RAG)、多跳检索与多源融合 |
| [08-RAG评测与生产实战.md](./14-Agent开发面试实战/08-RAG评测与生产实战.md) | RAGAS四维指标、Bad case分析方法论、知识库更新策略(增量/全量)、生产RAG踩坑(切分丢语义/召不回/超限/引用溯源)、高频面试题 |

#### 第三篇 · Agent 核心（`14-Agent开发面试实战/`）

| 文件 | 核心内容 |
|------|---------|
| [09-Agent运行循环与ReAct范式.md](./14-Agent开发面试实战/09-Agent运行循环与ReAct范式.md) | Agent Loop工程实现(伪代码+Java)、ReAct/Plan-Execute/Reflexion代码级对比、循环终止与防死循环(最大步数/超时/重复检测)、错误处理重试、手写最简Agent Loop |
| [10-上下文工程实战.md](./14-Agent开发面试实战/10-上下文工程实战.md) | 上下文窗口管理、Auto-Compact自动压缩、关键信息锚定、子Agent上下文隔离、长上下文工程(Lost in the Middle/1M窗口)、高频面试题 |
| [11-记忆系统设计.md](./14-Agent开发面试实战/11-记忆系统设计.md) | 短期/长期/工作/情景/语义/程序记忆、向量记忆库设计与检索、记忆写入策略、记忆遗忘与反思、Spring AI会话记忆实战、高频面试题 |
| [12-工具调用工程化.md](./14-Agent开发面试实战/12-工具调用工程化.md) | 工具设计原则(描述清晰/参数简单/错误可读)、工具治理(数量/分组/按需加载)、错误处理与自我修正、动态工具发现与MCP集成、Spring AI/LangChain4j @Tool实战 |
| [13-自我反思与评估体系.md](./14-Agent开发面试实战/13-自我反思与评估体系.md) | Self-Correction/Critic-Actor、Verifier/Guardrail客观校验、LLM-as-Judge、评测数据集与回归测试、Java侧评测工程实践 |

#### 第四篇 · 框架与编排（`14-Agent开发面试实战/`）

| 文件 | 核心内容 |
|------|---------|
| [14-LangChain4j详解.md](./14-Agent开发面试实战/14-LangChain4j详解.md) | LangChain4j定位与核心组件、与Python LangChain对比、AiServices/Tools/Memory/RAG模块、Spring Boot集成实战、选型对比 |
| [15-SpringAI详解.md](./14-Agent开发面试实战/15-SpringAI详解.md) | Spring AI核心抽象(ChatClient/Advisor/Embedding/VectorStore/ToolCallback)、Advisor链机制(类似拦截器)、Tool Calling、RAG与VectorStore、Spring Boot生产配置、与LangChain4j选型 |
| [16-SpringAIAlibaba与国内生态.md](./14-Agent开发面试实战/16-SpringAIAlibaba与国内生态.md) | Spring AI Alibaba定位、通义千问/百炼/DashScope接入、Graph/RAG/Observability扩展、国内合规与私有化部署 |
| [17-复杂流程编排.md](./14-Agent开发面试实战/17-复杂流程编排.md) | LangGraph状态图思想(State/Node/Edge/条件路由/循环)、人在环/断点恢复/回放、Java流程编排方案(Spring AI Alibaba Graph/自研状态机)、并发分支编排 |
| [18-多Agent系统实战.md](./14-Agent开发面试实战/18-多Agent系统实战.md) | 四种多Agent架构工程实现、Agent间通信与协调、成本控制与可观测、Java多Agent框架对比 |

#### 第五篇 · 工程化与生产（`14-Agent开发面试实战/`）

| 文件 | 核心内容 |
|------|---------|
| [19-Agent工程化架构.md](./14-Agent开发面试实战/19-Agent工程化架构.md) | 无状态运行时+有状态存储、会话持久化与断点恢复、Agent as a Service/API网关、异步长任务与队列、高可用与水平扩展 |
| [20-可观测性实战.md](./14-Agent开发面试实战/20-可观测性实战.md) | LLM可观测与传统APM差异、Trace/Token成本/延迟/错误率/质量、Langfuse/LangSmith/OTel GenAI接入、Spring AI Observation埋点 |
| [21-安全与护栏.md](./14-Agent开发面试实战/21-安全与护栏.md) | Prompt注入防护(指令数据隔离/输入审核)、内容审核(输入/输出/工具调用)、PII脱敏与数据不出域、权限RBAC与最小权限、沙箱与外发管控 |
| [22-模型路由与降级.md](./14-Agent开发面试实战/22-模型路由与降级.md) | 多模型策略(按任务/级联/降级)、成本优化(缓存/Prompt压缩/批处理)、私有化部署与混合云、模型版本管理与灰度 |
| [23-Agent评测与灰度发布.md](./14-Agent开发面试实战/23-Agent评测与灰度发布.md) | 评测体系(离线/在线/LLM-as-Judge)、黄金数据集与回归、在线A/B与灰度、注册中心与版本管理回滚 |
| [24-性能与成本优化.md](./14-Agent开发面试实战/24-性能与成本优化.md) | Token经济学与成本模型、Prompt压缩与上下文裁剪、批处理API/缓存/语义缓存、流式与并发、推理加速(vLLM/量化/蒸馏) |

#### 第六篇 · 场景与面试冲刺（`14-Agent开发面试实战/`）

| 文件 | 核心内容 |
|------|---------|
| [25-经典场景实战.md](./14-Agent开发面试实战/25-经典场景实战.md) | 智能客服/企业知识库/代码助手/数据分析Agent(NL2SQL)/工作流自动化、系统设计题(设计企业知识库问答系统/智能客服) |
| [26-AI编码助手与开发提效.md](./14-Agent开发面试实战/26-AI编码助手与开发提效.md) | Cursor/Claude Code/Copilot原理与对比、用AI提效(写代码/Review/测试/文档)、MCP Server开发实战、自建团队AI工具链 |
| [27-高频面试题精讲上.md](./14-Agent开发面试实战/27-高频面试题精讲上.md) | LLM基础类(幻觉/训练/参数/上下文)、RAG类(流程/优化/评测/进阶)、Prompt类(设计/注入/工程化)、Function Calling/Tool类，题库+答案+话术 |
| [28-高频面试题精讲下.md](./14-Agent开发面试实战/28-高频面试题精讲下.md) | Agent类(Loop/记忆/反思/多Agent)、MCP/协议类、工程化类(架构/可观测/安全/评测)、系统设计题套路、面试复盘与学习路线 |

---

### 16 - 计算机基础

> 计算机网络与操作系统基础，偏面试高频。与「01-java基础」「06-中间件」互补：NIO/epoll/零拷贝等知识点在各自技术栈里点到为止，底层机制在此统一讲透。

| 文件 | 核心内容 |
|------|---------|
| [01-计算机网络.md](./16-计算机基础/01-计算机网络.md) | 网络分层模型(OSI七层vs TCP/IP四层/数据封装单位)、TCP vs UDP、三次握手(为什么不是两次/四次/SYN洪泛)、四次挥手(为什么四次/TIME_WAIT与2MSL/CLOSE_WAIT过多)、TCP可靠传输(序列号/超时重传/滑动窗口/流量控制rwnd/拥塞控制慢开始-拥塞避免-快重传-快恢复)、HTTP方法与状态码/Cookie-Session-Token、HTTP演进(1.0/1.1/2/3队头阻塞/多路复用/QUIC)、HTTPS混合加密与TLS握手/CA证书、输入URL到页面显示、TCP粘包/正反向代理/CDN/CORS跨域/DNS协议、资料勘误(三次握手根因/UDP快不绝对/HTTP2未根治队头阻塞等) |
| [02-操作系统.md](./16-计算机基础/02-操作系统.md) | 进程vs线程vs协程(资源分配/调度单位/上下文切换开销/协程用户态切换)、IPC通信(管道/消息队列/共享内存最快/信号量/信号/Socket)、进程调度算法(FCFS/SJF/优先级/时间片轮转/多级反馈队列)、死锁四条件与处理(预防/银行家避免/检测解除)、虚拟内存与分页(TLB/多级页表/段页式)、页面置换算法(OPT/FIFO-Belady异常/LRU哈希表+双向链表/LFU/Clock)、线程同步(互斥锁vs自旋锁/读写锁/条件变量/信号量)、IO模型四概念区分与五大IO模型、IO多路复用select/poll/epoll(LT vs ET/epoll为什么快)、Reactor三种模式、用户态内核态与系统调用、零拷贝sendfile/mmap、文件系统inode/硬链接软链接/VFS、僵尸孤儿进程、资料勘误(阻塞非阻塞vs同步异步两维度/ET必须配非阻塞IO/协程阻塞坑等) |

---

### 17 - 场景题

> 跨主题、面试导向的场景题问答，聚焦"给定业务场景会出现什么问题、怎么解决、怎么选型"。与各知识点章节（01-java基础/04-数据库/05-分布式/06-中间件）及「09-系统设计与架构/经典场景设计」互补：那里讲原理与单点技术，本系列讲场景化的问题分析与方案权衡。

| 文件 | 核心内容 |
|------|---------|
| [01-高并发与并发场景.md](./17-场景题/01-高并发与并发场景.md) | 库存扣减防超卖(DB行锁FOR UPDATE/乐观锁版本号/Redis预扣减+Lua原子/分布式锁)、秒杀系统(削峰MQ/限流/缓存预热/异步下单/分段锁)、订单超时取消(定时扫表/DelayQueue/RocketMQ延迟/时间轮)、限流(计数器/滑动窗口/令牌桶/漏桶 单机Sentinel vs 分布式Redis+Lua)、线程池实战(参数设置/突发流量/隔离/优雅关闭/禁用Executors)、分布式计数与排行榜(ZSet)、接口幂等(token/唯一索引/状态机/防重表)、精度问题(BigDecimal/LongAdder)、热点key、SimpleDateFormat线程安全、资料勘误(Redis预扣减必须Lua/乐观锁CAS/Redis过期通知不可靠) |
| [02-缓存与消息队列场景.md](./17-场景题/02-缓存与消息队列场景.md) | 缓存穿透(布隆过滤器/空值)、击穿(互斥锁/逻辑过期)、雪崩(随机过期/熔断降级/多级缓存)、DB-缓存一致性(先更新DB再删缓存/延迟双删/为什么删而非更新)、热key大key、双写强一致(Cache Aside/afterCommit)；MQ消息丢失(三端保障-生产确认/持久化/手动ACK)、重复消费(幂等)、顺序性(同key同队列)、堆积(扩消费者/临时Topic/批量)、延迟消息(RocketMQ延迟等级/死信)、可靠投递；资料勘误(推荐先更新DB再删缓存/延迟双删原理/布隆无漏判有误判/顺序仅同队列内/手动ACK配幂等/Kafka单partition天然有序vs RocketMQ锁queue) |
| [03-分布式与数据库场景.md](./17-场景题/03-分布式与数据库场景.md) | 分布式锁实现与选型(Redis setnx+NX PX原子加锁+Lua释放校验value/Redlock争议/Zookeeper临时顺序节点CP/数据库唯一索引)、分布式锁三大坑(锁过期业务没完-Redisson看门狗30s租约10s续期仅未指定leaseTime生效/释放误删-UUID+Lua原子/主从切换丢锁)、分布式事务选型(2PC-XA强一致阻塞/Seata AT全局串行/TCC空回滚悬挂幂等/SAGA补偿/本地消息表/可靠消息最终一致/CAP与BASE)、分布式ID(UUID无序不做主键/雪花算法时钟回拨/号段模式Leaf/Redis incr)、分库分表(何时分-垂直水平-分片键-跨库Join聚合分页难题)、分库分表后全局唯一ID(雪花算法时钟回拨处理)、MySQL死锁排查(show engine innodb status/行锁交叉更新/间隙锁RR级别)、慢SQL优化(explain type-key-rows/索引失效/回表/覆盖索引/大分页游标WHERE id>last_id/延迟关联)、千万级大表优化(索引-冷热分离-归档-读写分离-分库分表/分批DELETE/count计数表)、高并发改同一行(乐观锁版本号vs悲观锁FOR UPDATE会阻塞vs分布式锁vs条件CAS WHERE stock>0)、主从延迟读旧数据(强制走主库/缓存过渡/半同步仍异步回放)、连接池配置(HikariCP公式core*2+spindle/maxLifetime小于wait_timeout/minimumIdle=maximumPoolSize)、资料勘误(setnx+expire非原子/看门狗leaseTime条件/Redlock争议/间隙锁RC消除/深分页游标/UUID主键页分裂/连接池非越大越好等) |
| [04-系统设计实战.md](./17-场景题/04-系统设计实战.md) | 系统设计题六步法套路(STAR)、短链(302重定向/Base62/发号器)、Feed流(推/拉/推拉结合)、IM(读写扩散/消息有序/可靠投递/已读未读)、抢红包(二倍均值法)、排行榜(ZSet/分桶)、点赞(去重/计数/异步落库)、秒杀(分层防护)、电商订单(状态机/分布式事务/超时取消)、附近的人(Redis GEO/Geohash/Haversine)、资料勘误(301vs302/推模式适用场景/写扩散爆炸/二倍均值不均匀/ZSet分桶/Redis过期通知不可靠) |

---

## 使用说明

- 每个文件先列出核心概念和常见面试题的**大纲**
- 具体内容留空，待每天学习时逐步补充完善
- 文件结构统一：「核心概念」 + 「常见面试题」
