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
| 07-性能调优实战.md | GC日志分析、JVM参数调优、常见问题排查案例 |

---

### 03 - Spring全家桶

| 文件 | 核心内容 |
|------|---------|
| 01-Spring核心.md | IoC容器、Bean生命周期、依赖注入、AOP原理、事务、Spring事务传播机制 |
| 02-SpringBoot.md | 自动配置原理、启动流程、Starter、配置文件加载顺序、条件注解 |
| 03-SpringCloud.md | 服务注册与发现、服务调用、配置中心、网关、熔断降级、分布式链路追踪 |

---

### 04 - 数据库（OLTP）

| 文件 | 核心内容 |
|------|---------|
| 01-MySQL基础.md | 存储引擎、数据类型、索引类型、约束、视图、触发器 |
| 02-索引与优化.md | B+树索引、索引底层原理、聚簇索引/非聚簇索引、回表、覆盖索引、最左前缀原则、索引失效场景、Explain |
| 03-事务与锁.md | ACID、隔离级别、MVCC、锁分类、行锁/表锁、Gap锁、意向锁、死锁 |
| 04-SQL优化.md | 慢查询分析、SQL优化技巧、分页优化、Join优化、子查询优化 |

---

### 05 - OLAP数据库

| 文件 | 核心内容 |
|------|---------|
| [01-ClickHouse-基础与部署.md](./05-OLAP数据库/01-ClickHouse-基础与部署.md) | ClickHouse概述、20节点生产集群部署方案、ZooKeeper配置、10+种表引擎详解与选型 |
| 02-ClickHouse-原理与优化.md | 副本分片内部原理、写入/查询流程详解、索引原理、物化视图、主键/分区/排序键设计、性能优化实践 |
| 03-ClickHouse-实战与运维.md | 日志存储场景、可观测性指标存储、常见问题排查、故障处理、运维最佳实践 |

---

### 06 - 分布式系统

| 文件 | 核心内容 |
|------|---------|
| 01-分布式理论基础.md | CAP、BASE、一致性协议（2PC、3PC、Paxos、Raft）、分布式ID |
| 02-缓存策略.md | 缓存作用、缓存穿透/击穿/雪崩、缓存更新策略、缓存一致性、热点Key |
| 03-消息队列.md | MQ作用、消息丢失/重复/顺序性、消息堆积、幂等性 |
| 04-分布式事务.md | 2PC、TCC、本地消息表、可靠消息最终一致性、最大努力通知、Seata |
| 05-微服务架构.md | 服务治理、服务容错、网关、配置中心、链路追踪 |

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
| 高频面试题.md | 链表、树、栈/队列、哈希、二分查找、双指针、滑动窗口、动态规划、回溯、贪心 |

---

### 09 - 设计模式

| 文件 | 核心内容 |
|------|---------|
| 常用设计模式.md | 单例、工厂、抽象工厂、建造者、原型、适配器、装饰器、代理、策略、观察者、模板方法、责任链 |

---

### 10 - 系统设计与架构

| 文件 | 核心内容 |
|------|---------|
| 经典场景设计.md | 秒杀系统、限流、降级、熔断、分库分表、读写分离、分布式锁、幂等性设计、高可用架构 |

---

### 11 - 监控与可观测性

| 文件 | 核心内容 |
|------|---------|
| [01-Prometheus指标详解.md](./10-监控与可观测性/01-Prometheus指标详解.md) | 4大核心指标（Counter/Gauge/Histogram/Summary）、适用场景对比、P50/P95/P99计算、常见面试题 |

---

### 12 - 云原生

| 文件 | 核心内容 |
|------|---------|
| [01-Docker基础与核心概念.md](./11-云原生/01-Docker基础与核心概念.md) | Docker概述、容器vs虚拟机、三大核心概念（镜像/容器/仓库）、架构、底层技术（Namespace/Cgroups/Union FS）、写时复制机制、常见面试题 |
| [02-Docker常用命令与实践.md](./11-云原生/02-Docker常用命令与实践.md) | 镜像/容器/网络/数据卷常用命令详解、docker run流程、exec vs attach、排错方法、资源限制、生产环境最佳实践 |
| [03-Dockerfile详解与镜像构建.md](./11-云原生/03-Dockerfile详解与镜像构建.md) | Dockerfile 14大指令详解、多阶段构建、.dockerignore、镜像优化、构建缓存原理、常见面试题 |
| 04-Docker网络.md | 网络驱动（bridge/host/overlay/macvlan/none）、DNS解析、端口映射、跨主机通信、网络安全、CNI |
| 05-Docker存储.md | Storage Driver、Volume、Bind Mount、tmpfs、存储驱动选型、持久化方案、数据备份恢复 |
| 06-Docker安全.md | 容器安全风险、用户权限、Capability、Seccomp、AppArmor、镜像安全扫描、运行时安全、最佳实践 |
| 07-Docker Compose.md | Compose 配置文件、多容器编排、服务发现、网络、卷管理、常用命令、生产环境部署 |
| 08-Kubernetes基础.md | K8s 架构、核心概念（Pod/Node/Namespace/Service/Deployment）、对象管理、常用命令、资源清单 |

---

## 使用说明

- 每个文件先列出核心概念和常见面试题的**大纲**
- 具体内容留空，待每天学习时逐步补充完善
- 文件结构统一：「核心概念」 + 「常见面试题」
