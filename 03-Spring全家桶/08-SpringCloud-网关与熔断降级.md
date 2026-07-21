# 08 - SpringCloud 微服务：网关、熔断降级限流、配置中心、链路追踪

> 📌 **一句话理解**：微服务架构里，每个服务都是一座孤岛，网关是统一的前台大门，熔断器是电路保险丝，限流是自来水阀门，配置中心是远程开关盒，链路追踪是请求的导航轨迹--它们一起让"一堆零散服务"变成"可治理的整体"。

---

## 核心概念

### 一、为什么需要 API 网关 ⭐⭐⭐

**没有网关时的痛点**

微服务把单体应用拆成了几十上百个服务，每个服务都有自己的 IP 和端口。如果让前端直接调后端：

- 前端要记几十个域名/IP，跨域问题满天飞
- 鉴权逻辑散落在每个服务里，重复且易漏
- 限流、熔断、日志监控没地方统一收口
- 服务拆分/合并后前端要跟着改

**网关是什么**

> **类比理解**：网关 = **大厦前台**。所有访客（请求）必须先到前台登记（鉴权），前台按你要去的楼层（路由）分流，必要时拦住不该进的人（限流/熔断），整个访问过程都有监控记录（日志）。

**网关的核心职责**

| 能力 | 通俗理解 | 典型实现 |
|---|---|---|
| 统一入口 | 所有外部请求只暴露一个域名/IP | Gateway / Nginx |
| 路由转发 | 按 URL/Header 把请求分发到对应服务 | Predicate 断言 |
| 鉴权认证 | 校验 token，没权限直接拒掉 | GlobalFilter + JWT |
| 限流熔断 | 后端扛不住时主动拒绝或降级 | Sentinel / Resilience4j |
| 日志监控 | 记录所有请求，便于审计和统计 | Filter + ELK |
| 协议转换 | 外部 HTTP 转 gRPC/WebSocket 等 | 自定义 Filter |
| 跨域处理 | 统一处理 CORS，避免每个服务都配 | CorsWebFilter |

---

### 二、Zuul vs Spring Cloud Gateway ⭐⭐⭐

**两大网关组件的演进史**

| 维度 | Zuul 1.x | Zuul 2.x | Spring Cloud Gateway |
|---|---|---|---|
| 出品方 | Netflix | Netflix | Spring 官方 |
| 底层 | Servlet（Tomcat）同步阻塞 | Netty 异步非阻塞 | WebFlux + Netty + Reactor |
| 线程模型 | 一个请求占一个线程 | 事件驱动 | Reactor 异步非阻塞 |
| 性能 | 一般（线程数受限） | 高 | 高 |
| 是否集成进 Spring Cloud | 是（已停更） | 否（Netflix 没集成） | 是（官方主推） |
| 状态 | ⚠️ 停更维护 | 未广泛使用 | 主流 |

> ⚠️ **重点纠偏**：很多人以为 Zuul 2.x 也是同步阻塞，这是错的。Zuul 2.x 已经改成 Netty 异步，但 Netflix 当时已不再积极维护 Spring Cloud Netflix 生态，Spring 官方选择另起炉灶做 Spring Cloud Gateway，所以 Zuul 2.x 在 Spring Cloud 体系里几乎没存在感。**Zuul 1.x 已停更，Gateway 是事实标准。**

**Gateway 为什么快**

- 基于 **WebFlux + Reactor + Netty**，全程异步非阻塞
- 网关线程不会因为后端慢响应而被占住，几个线程就能扛上万并发
- 不用 Servlet 容器（Tomcat），少了线程切换开销

> **类比理解**：Zuul 1.x 像"一个客人配一个服务员盯着"的传统餐厅，客人点菜慢服务员就只能傻等；Gateway 像"服务员+呼叫器"模式，客人慢慢看菜单，服务员同时招呼好几桌，谁点好了按铃就上菜。

---

### 三、Spring Cloud Gateway 核心概念 ⭐⭐⭐

Gateway 抽象出三个核心概念，面试必背：

| 概念 | 通俗理解 | 类比 |
|---|---|---|
| **Route（路由）** | 一条完整的转发规则：id + 目标 URI + 匹配条件 + 过滤器 | 前台的一本《访客分流手册》里的一条规则 |
| **Predicate（断言）** | 匹配请求的条件（Path/Header/Method 等），满足才走路由 | 前台问"您找哪位？哪个部门？" |
| **Filter（过滤器）** | 请求前/后做加工（加 header、改路径、鉴权、限流） | 前台给访客发访客牌、登记、带路 |

**1. Route 路由的组成**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service-route              # 路由唯一标识
          uri: lb://user-service              # 目标地址（lb:// 表示走服务发现负载均衡）
          predicates:
            - Path=/api/user/**               # 断言：路径匹配才走这条路由
          filters:
            - StripPrefix=2                   # 过滤器：转发前去掉前2段路径
```

**2. Predicate 断言（内置种类）**

| 断言 | 作用 | 示例 |
|---|---|---|
| Path | 路径匹配 | `Path=/api/user/**` |
| Method | HTTP 方法 | `Method=GET,POST` |
| Header | 请求头匹配 | `Header=X-Request-Id, \d+` |
| Host | 域名匹配 | `Host=**.example.com` |
| Query | 查询参数 | `Query=token, abc.` |
| Cookie | Cookie 匹配 | `Cookie=sessionId, xyz` |
| After/Before/Between | 时间窗口 | `After=2026-01-01...` |
| Weight | 按权重分流（灰度） | `Weight=group1, 8` |

**3. Filter 过滤器（两种）**

| 类型 | 作用域 | 说明 |
|---|---|---|
| **GatewayFilter** | 路由级 | 只对当前路由生效，配置在 route 的 filters 里 |
| **GlobalFilter** | 全局 | 对所有路由生效，实现 `GlobalFilter` 接口 |

```java
// 全局鉴权过滤器示例
@Component
public class AuthFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (StringUtils.isBlank(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 校验 token...
        return chain.filter(exchange);   // 放行到下一个过滤器
    }

    @Override
    public int getOrder() { return -100; }   // 数字越小优先级越高
}
```

---

### 四、Gateway 工作原理（流程图必背） ⭐⭐

**完整请求处理流程**

```text
   客户端请求
       │
       ▼
┌──────────────────────────┐
│  Gateway Handler Mapping │ ◀── 用 Predicate 匹配 Route
│  （路由匹配器）            │
└────────────┬─────────────┘
             │ 匹配成功
             ▼
┌──────────────────────────┐
│  Gateway Web Handler      │ ◀── 拿到匹配的 Route
│  （Web 处理器）            │
└────────────┬─────────────┘
             │ 取出 Filter Chain
             ▼
┌──────────────────────────┐
│  Filter Chain (pre)       │ ◀── 前置过滤器：鉴权、改请求、限流
│  前置过滤链                │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Proxy via Netty          │ ◀── 用 HttpClient（非阻塞）转发到后端
│  HttpClient 转发后端服务    │
└────────────┬─────────────┘
             │ 后端响应
             ▼
┌──────────────────────────┐
│  Filter Chain (post)      │ ◀── 后置过滤器：改响应、记录日志
│  后置过滤链                │
└────────────┬─────────────┘
             │
             ▼
        返回客户端
```

**关键点**

- **异步非阻塞**：全程基于 Reactor，转发后端用 Netty HttpClient，**后端再慢也不会阻塞网关线程**。网关线程在等响应时可以处理别的请求。
- **Filter 双向链**：前置（pre）在转发前执行，后置（post）在响应回来后执行，类似 Spring MVC 的 Interceptor。
- **Predicate 优先级**：多个 Route 都匹配时，按声明顺序取第一个。

> ⚠️ **易错点**：Gateway 默认不带服务发现，要配合 `spring-cloud-starter-loadbalancer` 才能用 `lb://service-name` 形式的 URI 做负载均衡。Spring Cloud 2020+ 起 Ribbon 被移除，改用 Spring Cloud LoadBalancer。

---

### 五、Gateway vs Nginx ⭐⭐

很多人会把两者混为一谈，其实定位不同：

| 维度 | Spring Cloud Gateway | Nginx |
|---|---|---|
| 定位 | 业务网关（应用层） | 反向代理 + 负载均衡（四/七层） |
| 语言 | Java | C |
| 性能 | 高（但比 Nginx 低） | 极高（C + epoll，单机几万并发） |
| 与 Spring Cloud 集成 | 原生（服务发现、配置中心） | 需自己改配置（upstream 手动维护） |
| 动态路由 | 支持（配合 Nacos 推送规则） | 默认静态，开源版改配置要 reload |
| 业务能力 | 强（鉴权、限流、熔断等业务过滤器） | 弱（主要做代理和负载均衡） |
| 静态资源 | 不擅长 | 强项 |

**生产实践：双层网关**

```text
   用户
    │
    ▼
┌─────────┐   静态资源、SSL卸载、四层负载、超高并发入口
│ Nginx   │   （外层：性能担当）
└────┬────┘
     │
     ▼
┌─────────┐   业务鉴权、动态路由、熔断限流、链路追踪打标
│ Gateway │   （内层：业务担当）
└────┬────┘
     │
     ▼
  微服务集群
```

> **类比理解**：Nginx = 大厦外面的保安亭（拦截大量访客、分流、安检）；Gateway = 大厦里面的前台（登记、带路、登记具体业务）。两者分工不同，常配合使用。

---

### 六、熔断降级限流：雪崩效应 ⭐⭐⭐

**雪崩效应怎么发生**

微服务间调用是链式的，一个服务出问题会级联拖垮整条链路：

```text
用户 -> 服务A -> 服务B -> 服务C
                    │
                    └─ C 挂了/慢了
                       │
                       └─ B 调 C 一直等待，线程被占住
                          │
                          └─ B 的线程池耗尽，B 也无法响应
                             │
                             └─ A 调 B 也卡住，A 线程池耗尽
                                │
                                └─ 整条链路宕机 = 雪崩
```

**类比理解**：电路里一个插座短路，如果不跳闸，电流会一直增大把整个电路烧毁。保险丝（熔断器）就是在电流异常时主动断开，保护整个电路。

**防御三板斧**

| 手段 | 作用 | 类比 |
|---|---|---|
| **超时控制** | 调用方设超时，不无限等待 | 等外卖 30 分钟没到就退款 |
| **熔断（Circuit Breaker）** | 失败率达到阈值时直接拒绝后续请求 | 保险丝跳闸 |
| **降级（Fallback）** | 主流程失败时返回兜底数据 | 餐厅没菜了给你泡面 |
| **限流（Rate Limiting）** | 控制单位时间请求数量 | 银行叫号机匀速放号 |
| **隔离（Bulkhead）** | 不同服务用独立线程池，互不影响 | 船舱隔板，一个进水不沉全船 |

---

### 七、熔断器三状态（必背状态图） ⭐⭐⭐

**三状态及转换**

```text
                    失败率/慢调用率达阈值
        ┌──────────────────────────────┐
        │                              │
        ▼                              │
┌───────────────┐                ┌───────────────┐
│   CLOSED      │                │    OPEN       │
│  （关闭）      │ ◀──────────── │   （打开）     │
│ 正常放行请求   │  半开探测成功  │ 直接拒绝/降级  │
│ 统计失败率     │                │ 等待冷却时间   │
└───────┬───────┘                └───────┬───────┘
        │                                │
        │          等待冷却时间到          │
        │  （比如 10 秒后进入半开）        │
        │                                ▼
        │                         ┌───────────────┐
        │                         │  HALF_OPEN    │
        │                         │  （半开）      │
        │                         │ 放少量请求探测  │
        │                         └───┬───────┬───┘
        │                             │       │
        │                     探测成功 │       │ 探测失败
        └─────────────────────────────┘       │
                                              ▼
                                         回到 OPEN
```

**三状态详解**

| 状态 | 行为 | 何时转换 |
|---|---|---|
| **CLOSED**（关闭） | 正常放行所有请求，同时统计失败率/慢调用率 | 失败率达阈值 -> OPEN |
| **OPEN**（打开） | 直接拒绝请求（快速失败），走降级逻辑，**不再调用后端** | 等待冷却时间（如 10s）-> HALF_OPEN |
| **HALF_OPEN**（半开） | 放行**少量请求**试探后端是否恢复 | 成功 -> CLOSED；失败 -> OPEN |

> ⚠️ **易错点**：OPEN 状态下不是无限期等待，而是有一个**冷却时间**（如 10 秒），到期后自动进入 HALF_OPEN，放少量请求试探后端是否恢复。HALF_OPEN 是试探状态，**不是完全打开也不是完全关闭**。

**核心参数（以 Resilience4j 为例）**

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failureRateThreshold: 50        # 失败率阈值 50%
        slowCallRateThreshold: 50       # 慢调用率阈值 50%
        slowCallDurationThreshold: 2s   # 慢调用判定：>2s 算慢
        waitDurationInOpenState: 10s    # OPEN 持续时间，10s 后进 HALF_OPEN
        slidingWindowSize: 10          # 滑动窗口大小
        minimumNumberOfCalls: 5        # 至少 5 次调用才统计
        permittedNumberOfCallsInHalfOpenState: 3  # HALF_OPEN 放 3 个试探请求
```

---

### 八、服务降级 ⭐⭐

**降级是什么**

主流程失败、超时、熔断时，返回兜底数据，保证核心功能可用，而不是直接给用户 500 错误。

**降级策略**

| 场景 | 兜底方案 |
|---|---|
| 推荐服务挂了 | 返回默认推荐列表（热门商品） |
| 用户头像服务挂了 | 返回默认头像 |
| 详情接口超时 | 返回缓存数据（Redis） |
| 库存查询失败 | 显示"库存查询中，稍后重试" |
| 评论服务熔断 | 隐藏评论区，不影响下单 |

**降级 vs 熔断 vs 限流**

| 手段 | 触发时机 | 目的 |
|---|---|---|
| 限流 | 请求量超过阈值 | 防止打垮后端 |
| 熔断 | 失败率超阈值 | 防止级联故障，快速失败 |
| 降级 | 主流程失败/熔断时 | 给用户兜底，保核心可用 |

> 三者关系：限流是预防，熔断是急救，降级是兜底。**熔断往往触发降级**，但降级不一定因为熔断（也可能是业务异常）。

---

### 九、限流算法（4种，必背图） ⭐⭐⭐

#### 1. 计数器（固定窗口）

```text
窗口1（0-1s）         窗口2（1-2s）        窗口3（2-3s）
┌────────────┐      ┌────────────┐      ┌────────────┐
│  ▲▲▲▲▲     │      │  ▲▲▲       │      │  ▲▲▲▲▲▲   │
│  阈值=5    │      │  阈值=5    │      │  阈值=5    │
└────────────┘      └────────────┘      └────────────┘
0s              1s              2s              3s
```

- **原理**：每个时间窗口内计数，超过阈值就拒绝。
- **优点**：简单。
- **缺点**：**临界点突刺问题**。窗口边界瞬间（如 0.99s 来 5 个 + 1.01s 来 5 个）一秒内放过 10 个请求，是阈值的 2 倍。

#### 2. 滑动窗口（解决临界突刺）

```text
   当前时刻
     │
     ▼
  ┌──┴──┬──┬──┬──┐
  │格子1 │格子2│格子3│格子4│   滑动窗口统计最近 N 个格子
  └──────┴──┴──┴──┘
  t-3s  t-2s t-1s  t
```

- **原理**：把时间切成更小的格子，统计**滑动覆盖的最近 N 个格子**的总数。
- **优点**：解决临界突刺，统计更平滑。Sentinel 默认用滑动窗口。
- **缺点**：实现比固定窗口复杂。

#### 3. 令牌桶（Token Bucket）

```text
        以固定速率 r 产生令牌
              │
              ▼
         ┌─────────┐
         │ 令牌桶  │  ◀── 桶满则丢弃新令牌
         │容量 C   │
         └────┬────┘
              │ 取到令牌
              ▼
         请求通过 ────► 没令牌则拒绝/排队
```

- **原理**：系统以固定速率往桶里放令牌，桶有最大容量；请求来时拿一个令牌，拿不到就拒绝。
- **特点**：**支持突发流量**。桶里可以攒令牌，突发请求一来可以一次性消耗多个令牌。
- **典型实现**：Guava `RateLimiter`。

#### 4. 漏桶（Leaky Bucket）

```text
   请求（任意速率）
     │  │  │
     ▼  ▼  ▼
   ┌─────────┐
   │  漏桶   │  ◀── 桶满则拒绝新请求
   │         │
   └────┬────┘
        │ 匀速漏出（固定速率 r）
        ▼
    后端处理
```

- **原理**：请求进桶，桶以**固定速率**漏出处理，超出桶容量则拒绝。
- **特点**：**强制匀速**，无论请求多快都按固定速率处理，**不支持突发**。
- **典型用途**：保护后端不被瞬时流量打垮，流量整形。

#### 5. 令牌桶 vs 漏桶（超高频对比）

| 维度 | 令牌桶 | 漏桶 |
|---|---|---|
| 处理速率 | 不固定（突发时可快） | 固定匀速 |
| 突发流量 | **支持**（桶里攒令牌） | **不支持**（强制匀速） |
| 桶里装什么 | 令牌 | 请求 |
| 谁匀速 | 产生令牌匀速 | 漏出请求匀速 |
| 类比 | 银行叫号机匀速放号，号攒着可以一次取多个 | 漏斗匀速漏水，水再多也只能匀速漏 |
| 适用场景 | API 网关限流（允许短暂突发） | 流量整形、保护后端匀速处理 |

> ⚠️ **核心区别一句话**：令牌桶允许"攒令牌"应对突发，漏桶强制"匀速出水"抹平突发。面试别答反了。

---

### 十、熔断降级组件对比 ⭐⭐

#### Hystrix vs Resilience4j vs Sentinel

| 维度 | Hystrix | Resilience4j | Sentinel |
|---|---|---|---|
| 出品方 | Netflix | Spring 官方推荐 | Alibaba |
| 状态 | ⚠️ 停更（2018 年起） | 活跃 | 活跃 |
| 隔离方式 | 线程池/信号量 | 信号量（默认）/舱壁 | 线程数 + 信号量 |
| 熔断 | 支持 | 支持 | 支持 |
| 限流 | 不支持（需配合） | 支持（RateLimiter） | 支持（核心能力） |
| 降级 | 支持 | 支持 | 支持 |
| 重试 | 不支持 | 支持 | 不支持（独立组件） |
| 系统自适应限流 | 不支持 | 不支持 | **支持**（CPU/Load） |
| 热点参数限流 | 不支持 | 不支持 | **支持** |
| 动态规则推送 | 不支持 | 不支持 | **支持**（控制台 + Nacos） |
| 控制台 | Dashboard（简陋） | 无 | Sentinel Dashboard（完善） |
| 编程模型 | 注解 + 包装类 | 函数式（Vavr） | 注解 + API |
| 与 Spring Cloud 集成 | 旧版默认 | 官方推荐 | 阿里系默认 |

#### Sentinel 的三大杀手锏（区别于其他组件）

1. **流量控制（限流）**：基于 QPS 或并发线程数限流，支持多种流控效果（直接拒绝、Warm Up 预热、匀速排队）。
2. **熔断降级**：支持慢调用比例、异常比例、异常数三种熔断策略。
3. **系统自适应限流**：根据 **CPU 使用率、系统 Load、平均 RT、入口 QPS、并发线程数**自动限流，不需要配规则，机器扛不住时自动保护。这是 Sentinel 独有的能力。

> ⚠️ **重点纠偏**：很多人以为 Sentinel 跟 Hystrix 一样只做熔断降级，其实 **Sentinel 的核心是"流量控制"**，熔断只是它能力的一部分。它最独特的是"系统自适应限流"--根据系统指标动态决策，不依赖人工配置规则。

#### Sentinel vs Hystrix 详细对比

| 维度 | Hystrix | Sentinel |
|---|---|---|
| 隔离方式 | 线程池隔离（默认）/ 信号量 | 并发线程数 + 信号量 |
| 熔断策略 | 失败率 | 慢调用比例 / 异常比例 / 异常数 |
| 限流能力 | 无 | 强（QPS、并发数、热点参数） |
| 流控效果 | 无 | 直接拒绝、Warm Up、匀速排队 |
| 系统自适应 | 无 | 有（CPU/Load/RT/QPS） |
| 规则持久化 | 配置文件 | Nacos / ZooKeeper / Apollo 推送 |
| 控制台 | Hystrix Dashboard | Sentinel Dashboard（实时监控 + 规则推送） |
| 编程侵入性 | 注解 + HystrixCommand | 注解 + SphU.entry() API |

---

### 十一、配置中心 ⭐⭐

**为什么需要配置中心**

- 微服务几十上百个，配置文件散落各处，改一次要重启
- 不同环境（dev/test/prod）配置不同，管理混乱
- 敏感配置（密码、密钥）不应硬编码进代码

#### Spring Cloud Config

- **存储**：基于 Git/SVN 等版本库
- **架构**：Config Server 从 Git 拉配置，Config Client 从 Server 拉
- **热刷新**：需配合 **Spring Cloud Bus（消息总线）** + Webhook，提交代码 -> Webhook 通知 Bus -> 广播到所有服务 -> `@RefreshScope` 刷新
- **缺点**：依赖 Git，热刷新链路长，无控制台

#### Nacos Config

- **存储**：配置存在 Nacos 内部（Derby/MySQL）
- **热刷新**：客户端用**长轮询（Long Polling）**监听配置变更，Nacos 推送变更后客户端自动刷新；加 `@RefreshScope` 注解的 Bean 会重新创建
- **控制台**：自带 Web 控制台，可视化编辑配置
- **集成**：与 Nacos 注册发现共用一套，开箱即用

#### 对比表

| 维度 | Spring Cloud Config | Nacos Config |
|---|---|---|
| 存储 | Git/SVN | 自带存储（Derby/MySQL） |
| 热刷新 | Bus + Webhook + RabbitMQ/Kafka | 长轮询，开箱即用 |
| 控制台 | 无（要靠 Git 工具） | 自带可视化控制台 |
| 命名空间/分组 | 弱 | 强（Namespace + Group + DataId） |
| 与服务发现共用 | 否 | 是 |
| 推送方式 | 事件广播 | 长轮询 + 推送 |

> **类比理解**：Spring Cloud Config 像"配置文件放仓库，改完要广播通知所有人重新读"；Nacos Config 像"配置存在数据库，客户端一直盯着，一变就刷新"。

**Nacos 长轮询原理（重点）**

```text
Client ──── HTTP 请求（带配置 MD5）───► Nacos Server
                                      │
                       配置没变？挂住请求（不立即返回）
                       最长 30s 超时
                                      │
            ┌──────────────┬──────────┴──────────┐
            │              │                     │
       超时返回        配置变了                立即返回
       （配置未变）    （MD5 不匹配）          新配置
            │              │
            ▼              ▼
       再次发起长轮询    收到变更，刷新本地配置
```

- 客户端把本地配置的 MD5 一起发给 Nacos
- Nacos 比对 MD5，相同就**挂住请求不立即返回**（长轮询，默认 30s 超时）
- 期间配置变更 -> Nacos 立即返回 -> 客户端拉新配置
- 超时且无变更 -> 返回空，客户端重新发起长轮询

> ⚠️ **易错点**：长轮询不是 WebSocket，本质是 HTTP 请求被服务端 hold 住不返回，到点或变更才返回。**比短轮询省请求次数，比 WebSocket 实现简单。**

---

### 十二、分布式事务 Seata（简述） ⭐⭐

详细原理在「05-分布式系统」模块，这里只列要点。

**为什么需要**

微服务下，一个业务操作跨多个服务，每个服务有自己的数据库，本地事务管不住，需要分布式事务保证一致性。

**四种事务模式**

| 模式 | 侵入性 | 原理 | 性能 | 场景 |
|---|---|---|---|---|
| **AT**（默认） | 无侵入 | 基于 undo log 两阶段：一阶段提交 + undo log，二阶段回滚或清理 | 高 | 大多数业务 |
| **TCC** | 强侵入（要写 try/confirm/cancel） | 业务层面两阶段提交 | 高 | 资金等强一致 |
| **SAGA** | 中等 | 长事务，正向 + 补偿 | 高 | 长流程业务 |
| **XA** | 弱 | 数据库 XA 协议，两阶段提交 | 低（锁资源） | 强一致数据库 |

**三大角色**

```text
   TM（事务管理器）          ◀── 开启全局事务，决策提交/回滚
       │
       │  开启/提交/回滚全局事务
       ▼
   TC（事务协调器）          ◀── Seata Server，维护全局事务状态
       │
       │  分支事务注册/上报状态
       ▼
   RM（资源管理器）          ◀── 每个微服务里的本地数据库代理
```

- **TC**：独立部署的 Seata Server，全局事务的大管家
- **TM**：开启全局事务的那个服务（事务发起方）
- **RM**：每个参与的服务，负责本地分支事务

---

### 十三、链路追踪 ⭐⭐

**核心概念**

| 概念 | 含义 | 类比 |
|---|---|---|
| **Trace** | 一次完整的请求链路 | 一个快递单号，从下单到送达全程 |
| **Span** | 一次调用段（一个服务内的处理） | 快递单号里的每一段物流（出库、运输、派送） |
| **TraceId** | 全链路唯一标识 | 贯穿整条链路的根 ID |
| **SpanId** | 当前 Span 的标识 | 当前段的 ID |
| **ParentSpanId** | 父 Span 的 ID | 谁调的我，便于构建调用树 |

**一次调用的 Trace/Span 关系**

```text
Trace (TraceId=abc)
  │
  ├─ Span1: Gateway  (SpanId=s1, parent=null)
  │    │
  │    ├─ Span2: Service-A  (SpanId=s2, parent=s1)
  │    │    │
  │    │    └─ Span3: Service-B  (SpanId=s3, parent=s2)
  │    │
  │    └─ Span4: Service-C  (SpanId=s4, parent=s1)
```

每个 Span 记录起止时间、服务名、方法、标签等，串联起来就是完整调用链。

#### 组件演进（务必准确）

| 组件 | 状态 | 说明 |
|---|---|---|
| **Spring Cloud Sleuth**（Brave/ZKin） | ⚠️ 停更 | Spring Cloud 2022.0（SpringBoot 3.x）起移除 |
| **Micrometer Tracing** | 主流 | Spring 官方替代，统一 API，支持 Brave（Zipkin）和 OpenTelemetry（OTLP）两种 bridge |
| **SkyWalking** | 活跃 | Apache 顶级项目，字节码增强，无侵入，自带 OAP + 存储 |
| **OpenTelemetry** | 标准统一 | CNCF 标准，Traces + Metrics + Logs 三合一 |

> ⚠️ **重点纠偏**：
> 1. **Sleuth 不是"被改名"，是被"移除"**。SpringBoot 3.x + Spring Cloud 2022.0 起，Sleuth 不再维护，要用 Micrometer Tracing。
> 2. **Micrometer Tracing 不是新框架**，它是 Spring 对底层追踪 API 的薄封装，底层可插拔接 OTel（OpenTelemetry）或 Brave（Zipkin）。
> 3. **SkyWalking 和 OpenTelemetry 不互斥**：SkyWalking 既可作为 OTel 的后端，也可独立用 agent 字节码增强方案。

#### 三种方案对比

| 方案 | 接入方式 | 语言支持 | 存储 | 特点 |
|---|---|---|---|---|
| **Micrometer Tracing + Zipkin** | SDK 嵌入 | 主要 Java | Zipkin 自带 / ES | Spring 官方推荐，轻量 |
| **Micrometer Tracing + OTel** | SDK 嵌入，OTLP 上报 | 多语言 | Jaeger/Tempo/SkyWalking OAP | 标准化，多语言统一 |
| **SkyWalking Agent** | 字节码增强，无侵入 | 多语言（Java 主） | 自带 OAP + 多存储 | 无侵入，功能丰富（含告警/拓扑） |

> **类比理解**：Sleuth 像老式 BP 机，能传简讯但功能单一且要淘汰；Micrometer Tracing 像智能手机的"统一通讯录框架"，可以接不同的运营商（OTel/Zipkin）；SkyWalking 像自带全套硬件的运营商（基站+话费系统+客服），开箱即用。

---

### 十四、微服务组件选型总表 ⭐⭐

Netflix OSS 停更后，Spring Cloud 生态的现代化替代：

| 能力 | Netflix（停更） | 现代替代 | 说明 |
|---|---|---|---|
| 服务注册发现 | Eureka | **Nacos** / Consul | Nacos 同时支持 AP/CP 切换 |
| 负载均衡 | Ribbon | **Spring Cloud LoadBalancer** | Ribbon 在 Spring Cloud 2020 移除 |
| 声明式调用 | Feign | **OpenFeign** | Spring Cloud OpenFeign 维护中 |
| 熔断降级 | Hystrix | **Resilience4j / Sentinel** | Hystrix 2018 停更 |
| 网关 | Zuul | **Spring Cloud Gateway** | Zuul 1.x 停更 |
| 配置中心 | Spring Cloud Config | **Nacos Config** | Nacos 一站式 |
| 链路追踪 | Sleuth + Zipkin | **Micrometer Tracing**（+ OTel/SkyWalking） | Sleuth 在 SpringBoot 3.x 移除 |
| 分布式事务 | 无 | **Seata** | 阿里开源 |

> **一句话记忆**：Netflix 全家桶 -> Alibaba 全家桶 + Spring 官方现代化方案。Eureka→Nacos、Ribbon→LoadBalancer、Hystrix→Resilience4j/Sentinel、Zuul→Gateway、Sleuth→Micrometer Tracing。

---

## 常见面试题

### 1. Spring Cloud Gateway 的核心概念有哪些？工作原理是什么？（高频）

**回答思路：**
> Gateway 有三个核心概念：
>
> 1. **Route（路由）**：一条转发规则，由 id + uri + predicate + filter 组成。
> 2. **Predicate（断言）**：匹配请求的条件，如 Path、Header、Method、Cookie、Host 等，满足才走这条路由。
> 3. **Filter（过滤器）**：对请求做前置/后置加工，分路由级（GatewayFilter）和全局级（GlobalFilter）两种。
>
> **工作原理**：客户端请求 -> Gateway Handler Mapping 用 Predicate 匹配 Route -> Gateway Web Handler 取出 Filter Chain -> 前置过滤器执行（鉴权、改请求、限流）-> 用 Netty HttpClient 异步转发到后端 -> 后端响应 -> 后置过滤器执行（改响应、记日志）-> 返回客户端。
>
> 全程基于 WebFlux + Reactor + Netty，**异步非阻塞**，后端再慢也不会阻塞网关线程，这是它比 Zuul 1.x 快的根本原因。

---

### 2. Gateway 和 Zuul 的区别？为什么选 Gateway？

**回答思路：**
> | 维度 | Zuul 1.x | Spring Cloud Gateway |
> |---|---|---|
> | 底层 | Servlet（Tomcat）同步阻塞 | WebFlux + Netty + Reactor 异步非阻塞 |
> | 线程模型 | 一个请求占一个线程 | Reactor，少量线程扛高并发 |
> | 出品 | Netflix（停更） | Spring 官方（主推） |
> | 集成 | 老版 Spring Cloud Netflix | Spring Cloud 原生 |
>
> **为什么选 Gateway**：
> 1. Zuul 1.x 已停更，Netflix 不再积极维护
> 2. Zuul 2.x 虽然改成 Netty 异步，但 Netflix 没集成进 Spring Cloud 生态
> 3. Gateway 性能更好（异步非阻塞，不阻塞线程）
> 4. Gateway 与 Spring Cloud 生态集成更紧密（服务发现、配置中心、链路追踪原生支持）
> 5. Gateway 功能更丰富（Predicate、Filter、动态路由等设计更现代）

---

### 3. Gateway 和 Nginx 的区别？怎么配合？

**回答思路：**
> - **Nginx**：C 语言写的反向代理 + 负载均衡，性能极高，主要做四/七层代理、静态资源、SSL 卸载。不擅长业务逻辑，配置基本静态（开源版改配置要 reload）。
> - **Gateway**：Java 写的业务网关，集成 Spring Cloud 生态，支持动态路由、业务鉴权、熔断限流、链路追踪打标等业务能力。性能不如 Nginx，但业务能力强。
>
> **生产实践**：双层网关架构。外层 Nginx 挡流量、做 SSL 卸载、负载均衡到多个 Gateway 实例；内层 Gateway 做业务鉴权、动态路由、熔断限流。Nginx 是性能担当，Gateway 是业务担当，各司其职。

---

### 4. 什么是熔断？熔断器的三种状态？怎么转换？（高频）

**回答思路：**
> **熔断**是一种保护机制，当下游服务失败率或慢调用率达到阈值时，熔断器打开，**直接拒绝后续请求**（快速失败 + 走降级），不再调用后端，防止级联故障拖垮上游。
>
> **三状态**：
> 1. **CLOSED（关闭）**：正常放行请求，同时统计失败率。
> 2. **OPEN（打开）**：失败率达阈值时进入，直接拒绝请求走降级，不再调用后端。
> 3. **HALF_OPEN（半开）**：OPEN 等待冷却时间（如 10s）后进入，放少量请求试探后端是否恢复。
>
> **转换**：
> - CLOSED -> OPEN：失败率/慢调用率达阈值
> - OPEN -> HALF_OPEN：等待冷却时间到
> - HALF_OPEN -> CLOSED：探测请求成功
> - HALF_OPEN -> OPEN：探测请求失败
>
> 类比保险丝：CLOSED 正常用电，OPEN 短路跳闸，HALF_OPEN 试合闸看还短不短路。

---

### 5. 什么是雪崩效应？如何防止？

**回答思路：**
> **雪崩效应**：微服务调用链中，下游服务（C）慢或挂 -> 上游（B）调用时线程被占住等待 -> B 线程池耗尽 -> B 也无法响应 -> A 调 B 也卡住 -> 整条链路宕机。
>
> **防止手段（三板斧+）**：
> 1. **超时控制**：调用方设超时（如 2s），不无限等待。
> 2. **熔断**：失败率达阈值时主动熔断，快速失败。
> 3. **降级**：主流程失败时返回兜底数据，保核心可用。
> 4. **限流**：控制单位时间请求数，防止打垮后端。
> 5. **隔离（舱壁模式）**：不同服务用独立线程池，互不影响，一个服务出问题不拖垮其他。
>
> 类比：电路短路如果不跳闸会烧毁整个电路，保险丝（熔断器）在异常时主动断开保护整个电路。

---

### 6. 常见限流算法有哪些？令牌桶和漏桶的区别？（高频）

**回答思路：**
> **四种常见算法**：
>
> 1. **计数器（固定窗口）**：每个时间窗口内计数，超阈值拒绝。简单但有**临界突刺问题**（窗口边界瞬间放过 2 倍流量）。
> 2. **滑动窗口**：把时间切成小格子，统计最近 N 个格子总数，解决临界突刺。Sentinel 默认用此。
> 3. **令牌桶**：固定速率往桶里放令牌，桶满丢弃；请求拿令牌通过。**支持突发流量**（桶里攒令牌），Guava RateLimiter 用此。
> 4. **漏桶**：请求进桶，固定速率漏出处理，桶满拒绝。**强制匀速**，不支持突发。
>
> **令牌桶 vs 漏桶**：
>
> | 维度 | 令牌桶 | 漏桶 |
> |---|---|---|
> | 突发流量 | 支持（攒令牌） | 不支持（匀速出水） |
> | 处理速率 | 可变 | 固定 |
> | 桶里装 | 令牌 | 请求 |
> | 类比 | 银行叫号机匀速放号 | 漏斗匀速漏水 |
>
> **一句话**：令牌桶允许"攒令牌应对突发"，漏桶强制"匀速出水抹平突发"。API 网关常用令牌桶（允许短暂突发），流量整形常用漏桶（保护后端匀速处理）。

---

### 7. Hystrix、Resilience4j、Sentinel 的区别？Sentinel 的优势？

**回答思路：**
> | 维度 | Hystrix | Resilience4j | Sentinel |
> |---|---|---|---|
> | 状态 | 停更 | Spring 官方推荐 | 活跃 |
> | 隔离 | 线程池/信号量 | 信号量/舱壁 | 线程数+信号量 |
> | 限流 | 无 | 有 | 强（核心能力） |
> | 动态规则 | 无 | 无 | Nacos 推送 |
> | 控制台 | 简陋 | 无 | 完善 |
> | 系统自适应 | 无 | 无 | 有 |
>
> **Sentinel 三大优势**：
> 1. **流量控制为核心**：支持 QPS、并发线程数、热点参数限流，多种流控效果（直接拒绝、Warm Up 预热、匀速排队）。
> 2. **系统自适应限流**：根据 CPU 使用率、系统 Load、平均 RT、入口 QPS、并发线程数**自动限流**，不需配规则，机器扛不住时自动保护。这是独有的。
> 3. **动态规则 + 控制台**：规则可通过 Sentinel Dashboard 实时推送，配合 Nacos 持久化，无需重启。
>
> **选型建议**：阿里系生态选 Sentinel（与 Nacos/Dubbo 无缝集成），Spring 官方纯净系选 Resilience4j（函数式轻量）。

---

### 8. Spring Cloud Sleuth 为什么被淘汰？现在用什么做链路追踪？

**回答思路：**
> **Sleuth 被淘汰的原因**：
> 1. Spring Cloud 2022.0（对应 SpringBoot 3.x）起，Sleuth **被移除**（不是改名，是被废弃）。
> 2. Sleuth 强绑定 Brave/Zipkin，无法支持 OpenTelemetry 等新标准。
> 3. Spring 官方推动 Observability 统一化，将追踪能力下沉到 Micrometer 体系。
>
> **现代替代方案**：
> 1. **Micrometer Tracing**：Spring 官方替代，是 Sleuth 的精神继承者。它本身是薄封装，底层可插拔接 Brave（Zipkin）或 OpenTelemetry（OTLP）。
> 2. **OpenTelemetry**：CNCF 标准，Traces + Metrics + Logs 三合一，多语言统一。
> 3. **SkyWalking**：Apache 顶级项目，字节码增强无侵入，自带 OAP + 存储 + 告警 + 拓扑图。
>
> **关系**：Micrometer Tracing 是 Spring 的统一 API 层，底层可选 OTel 或 Zipkin bridge；SkyWalking 既可以作为 OTel 的后端，也可以独立用 Java Agent 方案。

---

### 9. Spring Cloud Config 和 Nacos Config 的区别？

**回答思路：**
> | 维度 | Spring Cloud Config | Nacos Config |
> |---|---|---|
> | 存储 | Git/SVN 版本库 | Nacos 自带存储（Derby/MySQL） |
> | 热刷新 | 需 Bus + Webhook + MQ | 长轮询，开箱即用 |
> | 控制台 | 无 | 自带可视化控制台 |
> | 命名空间/分组 | 弱 | 强（Namespace + Group + DataId） |
> | 与服务发现共用 | 否 | 是（一套 Nacos 搞定） |
>
> **Nacos 长轮询原理**：客户端发 HTTP 请求（带本地配置 MD5）给 Nacos，Nacos 比对 MD5，相同则**挂住请求不返回**（最长 30s），期间配置变更则立即返回，超时则返回空。客户端收到响应后重新发起长轮询。
>
> **Nacos 优势**：配置 + 注册一体、热刷新简单、控制台好用、命名空间/分组管理多环境方便。生产环境大多选 Nacos。

---

### 10. Zuul 和 Gateway 都说"异步非阻塞"，到底有什么区别？

**回答思路：**
> **Zuul 1.x 是同步阻塞**（基于 Servlet + Tomcat，一个请求一个线程），不是异步。
>
> **Zuul 2.x 是异步非阻塞**（基于 Netty），但 Netflix 没有把它集成进 Spring Cloud 生态，且 Zuul 2.x 开源时间晚，Spring 官方已经自己做了 Gateway，所以 Zuul 2.x 在 Spring Cloud 体系里几乎没人用。
>
> **Spring Cloud Gateway 是异步非阻塞**（WebFlux + Reactor + Netty），是 Spring 官方主推。
>
> 所以"Zuul 和 Gateway 都异步非阻塞"这个说法是错的。准确说法是：**Zuul 1.x 同步阻塞已停更，Zuul 2.x 异步但没进 Spring Cloud，Gateway 是异步非阻塞的主流方案。**

---

## 本章学习建议

1. **先记大图**：网关 = 前台、熔断器 = 保险丝、限流 = 阀门、令牌桶 = 叫号机、漏桶 = 漏斗。先有直观类比再啃细节。
2. **必背三个图**：Gateway 工作原理流程图、熔断器三状态转换图、令牌桶 vs 漏桶图。面试一画图分就到手。
3. **组件停更清单要记牢**：Eureka→Nacos、Ribbon→LoadBalancer、Hystrix→Resilience4j/Sentinel、Zuul→Gateway、Sleuth→Micrometer Tracing。这是 Spring Cloud 现代化选型的核心。
4. **重点纠错**：Zuul 1.x 是同步阻塞（不是异步）、Sleuth 是被移除（不是改名）、Sentinel 核心是流量控制（不只是熔断）、令牌桶支持突发而漏桶不支持。
5. **结合实践**：限流算法最好手写一遍（Guava RateLimiter 一行就能跑），熔断器用 Resilience4j 配置一次就有感觉。
6. **关联其他章节**：链路追踪的 TraceId 在日志排查时极其重要，配合「02-JVM虚拟机/03-JDK性能监控与故障处理工具」的线上排查流程一起看；分布式事务详细原理在「05-分布式系统/04-分布式事务」。

> 💡 **学习心法**：Spring Cloud 的本质是"把单体应用里那些隐式的调用关系，显式化成可治理的网络"。网关、熔断、限流、配置、追踪，每一项都在解决"拆开之后才暴露的问题"。学的时候多想一句"单体里这个问题存在吗？为什么拆开就有了？"--理解就深一层。

---

## 资料勘误与重点提醒

1. **Zuul 2.x 经常被误传为"同步阻塞"**：实际上 Zuul 2.x 已改成 Netty 异步，只是没集成进 Spring Cloud 生态，社区几乎不用。准确表述是"Zuul 1.x 同步阻塞已停更，Zuul 2.x 异步但未进 Spring Cloud"。
2. **Sleuth 的命运**：很多旧资料说"Sleuth 改名为 Micrometer Tracing"，这是不准确的。准确表述是 Sleuth 在 SpringBoot 3.x / Spring Cloud 2022.0 起**被移除**，由 Micrometer Tracing 体系替代，且新方案支持 OTel/Zipkin 双 bridge。
3. **Sentinel 的定位**：常被简单等同于"Hystrix 替代品"，但 Sentinel 的核心能力是**流量控制**（限流、热点参数、系统自适应），熔断只是其中一部分。系统自适应限流（基于 CPU/Load）是它独有的杀手锏，务必记住。
4. **令牌桶 vs 漏桶**：超高频考点，务必记牢"令牌桶支持突发（攒令牌），漏桶强制匀速（匀速出水）"，别答反。两者桶里装的东西也不同：令牌桶装令牌，漏桶装请求。
5. **熔断器 HALF_OPEN 易被忽略**：HALF_OPEN 是"试探状态"，放少量请求探测后端，成功才回 CLOSED，失败回 OPEN。不是"完全打开一半流量"，别理解错。
6. **Nacos Config 是长轮询不是 WebSocket**：本质是 HTTP 请求被服务端 hold 住不返回，到点或变更才返回。比短轮询省请求，比 WebSocket 实现简单。
7. **Gateway 默认不带负载均衡**：要用 `lb://service-name` 形式 URI，需引入 `spring-cloud-starter-loadbalancer`（Spring Cloud 2020+ Ribbon 已被移除）。
8. **分布式事务补充**：Seata AT 模式默认基于 undo log 两阶段，**一阶段就提交本地事务并释放锁**（不像 XA 一直锁资源），性能好但有短暂不一致窗口（事务协调器宕机期间的脏写问题需要业务侧规避）。详细原理见「05-分布式系统/04-分布式事务」。
