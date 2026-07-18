# 08 - SkyWalking 探针原理（Java Agent）

## 核心概念

### 1. Java Agent 机制概述

SkyWalking Java Agent 是整个 SkyWalking 生态中最核心的组件之一，它利用 JVM 提供的 **Instrumentation 机制**，在运行时动态修改字节码，实现对业务代码的零侵入监控。

#### 1.1 JVM Instrumentation 基础

Java 从 JDK 1.5 开始提供 `java.lang.instrument` 包，支持两种 Agent 加载方式：

| 加载方式 | 入口方法 | 时机 | 使用场景 |
|---------|---------|------|---------|
| **静态加载** | `premain(String agentArgs, Instrumentation inst)` | JVM 启动时，main 方法执行前 | SkyWalking 的默认方式 |
| **动态加载** | `agentmain(String agentArgs, Instrumentation inst)` | JVM 运行中，通过 Attach API 动态附加 | Arthas 热诊断工具 |

**SkyWalking 使用静态加载方式**，通过 `-javaagent` 启动参数指定。

#### 1.2 Agent 的 MANIFEST.MF

```properties
# skywalking-agent.jar 的 META-INF/MANIFEST.MF
Manifest-Version: 1.0
Premain-Class: org.apache.skywalking.apm.agent.SkyWalkingAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

- `Premain-Class`：指定 Agent 入口类
- `Can-Redefine-Classes`：允许重新定义类
- `Can-Retransform-Classes`：允许重新转换类（SkyWalking 使用 `retransform` 而非 `redefine`）

### 2. Agent 启动流程

#### 2.1 整体启动流程

```
java -javaagent:skywalking-agent.jar -jar app.jar

1. JVM 加载 Agent
2. 调用 SkyWalkingAgent.premain(agentArgs, instrumentation)
3. Agent 初始化
4. 注册 ClassFileTransformer
5. 加载业务类时自动增强
6. 业务方法被拦截，生成 Span 数据
```

#### 2.2 详细启动流程（源码分析）

```
SkyWalkingAgent.premain()
  │
  ├── 1. 初始化配置（SnifferConfigInitializer）
  │     ├── 读取 agent.config 配置文件
  │     ├── 读取系统属性（-DSW_AGENT_NAME 等）
  │     └── 合并配置（系统属性优先级 > 配置文件）
  │
  ├── 2. 加载插件（PluginFinder）
  │     ├── 扫描 skywalking-agent.jar 中的插件
  │     ├── 解析每个插件的 skywalking-plugin.def 文件
  │     └── 构建 PluginFinder（插件查找器）
  │
  ├── 3. 初始化核心组件
  │     ├── ServiceManager：管理 Agent 内部服务（GRPCChannelManager 等）
  │     ├── ContextManager：Trace 上下文管理器
  │     └── TracingContext：当前线程的 Trace 上下文
  │
  ├── 4. 使用 ByteBuddy 创建 AgentBuilder
  │     └── new AgentBuilder.Default()
  │           .type(pluginFinder.buildMatch())   // 匹配需要增强的类
  │           .transform(new Transformer())       // 转换匹配到的类
  │           .with(new Listener())               // 监听转换结果
  │           .installOn(instrumentation)         // 安装到 JVM
  │
  └── 5. 启动完成，Agent 开始在后台监听类加载
```

### 3. 类加载隔离

#### 3.1 为什么需要类加载隔离？

类加载隔离是所有 APM 探针的**核心技术难题**，不做隔离的 Agent 在生产环境中 100% 会出问题。

**场景 1：依赖版本冲突（最常见）**

```
业务应用的 classpath：
  ├── gRPC 1.30（业务 2019 年上线，一直没升级）
  ├── Netty 4.1.42
  └── Protobuf 3.5

SkyWalking Agent 的 classpath：
  ├── gRPC 1.50（Agent 2023 年版本，需要新特性）
  ├── Netty 4.1.90
  └── Protobuf 3.21

如果不隔离：
  JVM 先加载到业务的 gRPC 1.30 → Agent 调用 gRPC 1.50 新增的方法
  → NoSuchMethodError → 业务应用启动失败！
```

**场景 2：类重复加载问题**

Agent 会增强很多业务类（如 `@RestController`、`@Service`），如果 Agent 和业务用同一个 ClassLoader：

```
同一个类被加载两次：
  1. AppClassLoader 加载业务类 UserController
  2. Agent 增强后，ByteBuddy 又生成一个 UserController$Enhanced

结果：
  instanceof 判断失效 → UserController.class != UserController$Enhanced.class
  强转失败 → ClassCastException
  Spring 依赖注入失败 → 应用启动崩溃
```

**场景 3：SPI 服务发现污染**

很多框架用 Java SPI 机制（如 JDBC Driver、SLF4J），如果不隔离：

```
Agent 的 SLF4J binding ←→ 业务的 SLF4J binding
  ↓
SPI 服务发现时，两个 binding 都被加载
  ↓
SLF4J 报 "multiple bindings" 警告，随机选一个
  ↓
Agent 的日志打到业务的日志文件里，或者反过来
```

**类加载隔离的本质**：让 Agent 的依赖和业务的依赖分别放在不同的「命名空间」里，互不干扰。

#### 3.2 AgentClassLoader 设计

```
┌────────────────────────────────────────────────────────┐
│                   JVM 类加载器层次                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Bootstrap ClassLoader（加载 rt.jar，JDK 核心类）        │
│        │                                               │
│        ├── ExtClassLoader（JDK 8）/ PlatformClassLoader │
│        │                                               │
│        └── AppClassLoader（加载业务应用的类）             │
│              │                                         │
│              └── AgentClassLoader（加载 Agent 的类）      │
│                    ├── gRPC 1.50（Agent 自己的版本）     │
│                    ├── ByteBuddy（Agent 自己的版本）      │
│                    ├── Protobuf 3.x（Agent 自己的版本）   │
│                    └── 所有插件类（PluginClassLoader）     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**关键设计**：
1. AgentClassLoader 以 AppClassLoader 为父加载器（可以访问业务类）
2. Agent 的第三方依赖放在 AgentClassLoader 中（与业务依赖隔离）
3. 插件类使用独立的 PluginClassLoader（插件之间隔离）

#### 3.3 Pinpoint 也用了类加载隔离吗？

**用了，而且比 SkyWalking 更复杂**。

Pinpoint 作为最早开源的 APM 探针（2014 年），在类加载隔离上踩了很多坑，它的隔离方案比 SkyWalking 更激进：

| 对比维度 | SkyWalking | Pinpoint |
|---------|-----------|---------|
| **核心类加载器 | AgentClassLoader | IsolatedClassLoader |
| **插件隔离** | PluginClassLoader（每个插件独立） | BootstrapPlugin + AgentPluginClassLoader |
| **Bootstrap 类注入 | ✅ 用 Instrumentation.appendToBootstrapClassLoaderSearch | ✅ 同样用 Bootstrap 注入 |
| **类加载顺序** | 双亲委派（先查自己，再查父加载器 | 完全打破双亲委派（**先自己加载，找不到再问父要**） |
| **隔离粒度** | Agent vs 业务 | Agent vs 业务 vs 插件之间 |

**Pinpoint 为什么要打破双亲委派？**

Pinpoint 诞生于 2014 年，当时 ByteBuddy 还不成熟，Pinpoint 选择了 ASM 做字节码增强，同时遇到了更极端的类加载冲突问题：

```
问题：某些中间件自己有类加载器（如 Tomcat 的 WebAppClassLoader）

  Tomcat WebAppClassLoader
    ├── 每个 WebApp 有自己的 classpath
    └── 不遵循双亲委派（先查自己，再查父）

如果 Agent 用正常的双亲委派：
  Agent 的类放在父加载器里 → WebApp 里的类看不到 Agent 的类
  → 插件无法增强 WebApp 里的类

Pinpoint 的解决方案：
  把 Agent 的核心类注入到 Bootstrap ClassLoader 里
  → 所有类加载器都能看到
  → 不管中间件的自定义类加载器也能访问 Agent 的类
```

**代价**：把 Agent 的类注入 Bootstrap 后，优先级极高，一旦出问题就是 JVM 直接 crash。所以 Pinpoint 的 Agent 稳定性问题一度是社区吐槽的重点。

#### 3.4 OpenTelemetry Java Agent 的类加载隔离方案

OTel 走了第三条路：**不做 ClassLoader 级别的隔离，而是用 Maven Shade 插件把所有依赖「重命名打包」**。

**OTel 的做法：影子类（Shaded Classes）**

```
OTel Agent 构建时：
  ├── 把 gRPC 1.50 的包名从 io.grpc → io.opentelemetry.shaded.grpc
  ├── 把 Netty 4.1.90 的包名从 io.netty → io.opentelemetry.shaded.netty
  ├── 把 Protobuf 3.21 的包名从 com.google.protobuf → io.opentelemetry.shaded.protobuf
  └── 所有内部依赖全部重命名

结果：
  业务的 io.grpc 1.30 和 OTel 的 io.opentelemetry.shaded.grpc 1.50
  是完全不同的两个类，不存在版本冲突！
```

**三种方案对比**：

| 方案 | 代表产品 | 原理 | 优点 | 缺点 |
|------|---------|------|------|------|
| **自定义 ClassLoader** | SkyWalking | AgentClassLoader 隔离依赖 | 1. 不需要修改字节码<br>2. 插件可以独立加载<br>3. 调试方便 | 1. 类加载器层次复杂<br>2. 跨加载器调用需要反射 |
| **打破双亲委派 + Bootstrap 注入** | Pinpoint | 核心类注入 Bootstrap，插件隔离 | 1. 适配所有自定义类加载器（Tomcat 等）<br>2. 插件访问权限最大 | 1. 风险极高，一旦出错 JVM 直接 crash<br>2. 调试困难 |
| **Maven Shade 影子类** | OpenTelemetry | 构建时重命名所有依赖包名 | 1. 不涉及类加载器黑科技<br>2. 原理简单，出问题好排查<br>3. 兼容性最好 | 1. Agent JAR 包变大（所有依赖都打进去）<br>2. 构建时间变长<br>3. 堆栈里的类名是 `xxx.shaded.xxx`，可读性差 |

**OTel 为什么选影子类方案？**

OTel 的定位是「标准」，不是「APM 产品」。它需要适配所有 Java 应用（包括各种老应用、奇葩类加载器的应用），**兼容性是第一位**。

```
OTel 的设计哲学：
  我不管你的应用用了什么类加载器
  我也不管你的应用依赖了什么版本的库
  我把我自己的所有依赖全部重命名
  → 永远不会和你的业务代码冲突
```

**代价**：OTel Agent 的 JAR 包有 20MB+（SkyWalking Agent 只有 10MB 左右），因为所有依赖都打进去了。

> ⚠️ **面试题延伸**：三种方案没有绝对的好坏，只是设计哲学不同——SkyWalking 选「保守稳定」，Pinpoint 选「极致适配」，OTel 选「通用兼容」。

### 4. 字节码增强（ByteBuddy）

#### 4.1 为什么选择 ByteBuddy？

| 对比维度 | ByteBuddy（SkyWalking） | ASM（Pinpoint） | Javassist |
|---------|------------------------|-----------------|-----------|
| API 易用性 | ★★★★★ 流式 API | ★★☆☆☆ 需理解字节码指令 | ★★★★☆ 源码级 API |
| 性能 | ★★★★☆ 好 | ★★★★★ 最好 | ★★★☆☆ 一般 |
| 类型安全 | ✅ 编译期检查 | ❌ 运行时错误 | ❌ 运行时错误 |
| 社区活跃度 | ★★★★★ 高 | ★★★☆☆ 中 | ★★★☆☆ 中 |
| 学习曲线 | 低 | 高 | 中 |

**ByteBuddy vs ASM 深度对比**：

| 维度 | ByteBuddy | ASM |
|------|-----------|-----|
| **抽象层级** | 高级 API，面向"类/方法"语义 | 低级 API，面向"字节码指令" |
| **编程模型** | 声明式（DSL 链式调用，描述"做什么"） | 命令式（Visitor 模式，手动操作字节码） |
| **代码示例** | `new ByteBuddy().subclass(Foo.class).method(named("bar")).intercept(...)` | `mv.visitVarInsn(ALOAD, 0); mv.visitMethodInsn(INVOKESTATIC, ...);` |
| **类型安全** | 编译期类型检查，重命名/重构友好 | 字符串常量拼字节码，重构易出错 |
| **字节码知识要求** | 几乎不需要 | 需要深入理解 JVM 字节码指令集、栈帧、常量池 |
| **性能差异** | 接近 ASM（底层也生成字节码），略慢 5-10% | 最接近手写字节码，零开销 |
| **调试难度** | 生成的代码可读，断点调试友好 | 字节码级错误难排查，`VerifyError` 常见 |
| **代表用户** | SkyWalking、Mockito、Hibernate | Pinpoint、CGLIB、Groovy |
| **适用场景** | 业务逻辑拦截、AOP、APM 探针 | 底层框架、字节码分析工具、性能极致场景 |

**选型结论**：
- 选 **ByteBuddy**：大多数场景，特别是 APM 探针（需要快速开发插件、团队不需要字节码专家）
- 选 **ASM**：极致性能场景、字节码分析工具（如 FindBugs/SpotBugs）、需要精确控制字节码的场景

> ⚠️ Pinpoint 选择 ASM 有其历史原因——Pinpoint 诞生于 2014 年，当时 ByteBuddy 还不够成熟（v1.0 于 2014 年发布）。SkyWalking 在 2017 年重构时选用了已经成熟的 ByteBuddy，代表了业界的趋势。

#### 4.2 增强流程

```
1. 类加载时 JVM 触发 ClassFileTransformer
      │
2. AgentBuilder 根据匹配规则，判断是否需要增强该类
      │
3. 如果匹配：
   ├── 创建 DynamicType.Builder
   ├── 根据插件定义，选择拦截点
   ├── 使用 ByteBuddy 生成增强后的字节码
   └── 返回新的字节码给 JVM
      │
4. JVM 使用增强后的字节码定义类
      │
5. 当业务方法被调用时，拦截器自动生效
```

#### 4.3 增强示例

```java
// 原始类（Spring MVC Controller）
@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);  // ← 需要被拦截
    }
}

// 增强后（等价于）
@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        // === Agent 注入的拦截逻辑 ===
        Span span = ContextManager.createEntrySpan(
            "GET:/user/{id}",     // operationName
            null,                 // carrier
            ContextCarrier        // 从请求中提取
        );
        span.setComponent(ComponentsDefine.SPRING_MVC);
        span.setLayer(SpanLayer.HTTP);
        span.setTag("http.method", "GET");
        // === 拦截逻辑结束 ===

        try {
            User user = userService.findById(id);  // 原始业务逻辑
            span.setTag("http.status_code", "200");
            return user;
        } catch (Exception e) {
            span.errorOccurred();
            span.log(e);
            throw e;
        } finally {
            ContextManager.stopSpan();  // 结束 Span
        }
    }
}
```

### 5. 拦截点定义（AbstractClassEnhancePluginDefine）

#### 5.1 插件定义结构

每个插件都继承自 `AbstractClassEnhancePluginDefine`，需要定义以下四个关键要素：

```java
public class SpringMVCInstrumentation extends AbstractClassEnhancePluginDefine {

    // 1. 需要增强的类匹配规则
    @Override
    protected ClassMatch enhanceClass() {
        return ClassMatch.byAnnotation("org.springframework.stereotype.Controller")
            .or(ClassMatch.byAnnotation("org.springframework.web.bind.annotation.RestController"));
    }

    // 2. 拦截的方法匹配规则
    @Override
    protected InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoint() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    // 匹配带有 @RequestMapping / @GetMapping 等注解的方法
                    return isAnnotatedWith(
                        named("org.springframework.web.bind.annotation.RequestMapping")
                        .or(named("org.springframework.web.bind.annotation.GetMapping"))
                        .or(named("org.springframework.web.bind.annotation.PostMapping"))
                        // ...
                    );
                }

                @Override
                public String getMethodsInterceptor() {
                    return "org.apache.skywalking.apm.plugin.spring.mvc.SpringMVCInterceptor";
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }

    // 3. 验证类是否存在（witness classes）
    @Override
    protected String[] witnessClasses() {
        return new String[] {
            "org.springframework.web.servlet.DispatcherServlet"
        };
    }

    // 4. 插件名称
    @Override
    public String getPluginName() {
        return "spring-mvc";
    }
}
```

#### 5.2 四个关键要素

| 要素 | 方法 | 作用 |
|------|------|------|
| **类匹配** | `enhanceClass()` | 指定哪些类需要增强（按名称、注解、父类、接口） |
| **方法匹配** | `getInstanceMethodsInterceptPoint()` / `getStaticMethodsInterceptPoint()` | 指定类中哪些方法需要拦截 |
| **拦截器** | `getMethodsInterceptor()` | 指定拦截逻辑的实现类 |
| **见证类** | `witnessClasses()` | 可选，验证依赖是否存在（避免类找不到异常） |

#### 5.3 方法拦截器

```java
public class SpringMVCInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method,
                             Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) {
        // 1. 从 HTTP 请求中提取 sw8 上下文
        HttpServletRequest request = (HttpServletRequest) allArguments[0];
        ContextCarrier carrier = new ContextCarrier();
        CarrierItem items = carrier.items();
        while (items.hasNext()) {
            CarrierItem item = items.next();
            item.setHeadValue(request.getHeader(item.getHeadKey()));
        }

        // 2. 创建 Entry Span
        Span span = ContextManager.createEntrySpan(
            request.getMethod() + ":" + request.getRequestURI(),
            carrier
        );

        // 3. 设置 Span 属性
        span.setComponent(ComponentsDefine.SPRING_MVC);
        span.setLayer(SpanLayer.HTTP);
        span.setTag("http.method", request.getMethod());
        span.setTag("url", request.getRequestURL().toString());
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method,
                              Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) {
        // 1. 设置 HTTP 状态码
        HttpServletResponse response = (HttpServletResponse) allArguments[1];
        ContextManager.getActiveSpan()
            .setTag("http.status_code", String.valueOf(response.getStatus()));

        // 2. 判断是否有错误
        if (response.getStatus() >= 500) {
            ContextManager.getActiveSpan().errorOccurred();
        }

        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method,
                                       Object[] allArguments, Class<?>[] argumentsTypes,
                                       Throwable t) {
        // 记录异常
        ContextManager.activeSpan().log(t);
    }
}
```

### 6. 上下文管理（ContextManager）

#### 6.1 核心类图

```
ContextManager（线程安全、入口类）
  │
  ├── createEntrySpan()    → 创建 Entry Span（接收请求）
  ├── createExitSpan()     → 创建 Exit Span（发起调用）
  ├── createLocalSpan()    → 创建 Local Span（内部操作）
  ├── activeSpan()         → 获取当前活跃 Span
  ├── stopSpan()           → 停止当前 Span
  ├── capture()            → 捕获 ContextSnapshot（跨线程）
  ├── continued()          → 恢复 ContextSnapshot（跨线程）
  └── getGlobalTraceId()   → 获取全局 TraceId
```

#### 6.2 TracingContext 状态机

```
          ┌──────────────────────────────────────┐
          │          TracingContext 状态机         │
          └──────────────────────────────────────┘

  [初始化] ──→ createEntrySpan() ──→ [活跃]
    │                                    │
    │                              createExitSpan()
    │                              createLocalSpan()
    │                                    │
    │                              stopSpan()
    │                                    │
    │                                    ▼
    │                              [活跃] (Span 栈非空)
    │                                    │
    │                              stopSpan() ← 最后一个 Span
    │                                    │
    │                                    ▼
    └──────────────────────────── [完成] → 上报 Segment
```

**关键规则**：
1. 每个线程有独立的 `TracingContext`（通过 ThreadLocal 管理）
2. Span 以栈的方式管理（先进后出），`stopSpan()` 关闭当前栈顶的 Span
3. 当所有 Span 都关闭后，整个 Segment 完成，异步上报到 OAP

---

## 常见面试题

### Q1: SkyWalking Agent 是如何做到"零侵入"的？

核心技术：**JVM Instrumentation + ByteBuddy 字节码增强**

1. `-javaagent` 启动时，JVM 在加载 `main` 类之前先加载 Agent 的 `premain` 方法
2. `premain` 中注册 `ClassFileTransformer`
3. 当业务类被类加载器加载时，JVM 触发 `ClassFileTransformer.transform()`
4. ByteBuddy 根据插件定义修改字节码，在目标方法前后插入拦截逻辑
5. 整个过程**不修改源码、不修改 class 文件、不修改类加载器**，只在运行时动态修改字节码

### Q2: 为什么 Agent 需要类加载隔离？如何实现的？

**原因**：Agent 依赖的第三方库（gRPC、ByteBuddy、Protobuf）可能与业务应用版本冲突。例如业务使用 gRPC 1.30，Agent 使用 gRPC 1.50，如果共用类加载器会导致 `NoSuchMethodError`。

**实现**：
- Agent 的核心类和第三方依赖放在 `AgentClassLoader` 中
- `AgentClassLoader` 以 `AppClassLoader` 为父加载器（可以访问业务类）
- 插件类使用独立的 `PluginClassLoader`（插件之间隔离）
- 遵循双亲委派模型，但 Agent 的类优先从自己的 ClassLoader 加载

### Q3: premain 和 agentmain 有什么区别？

| 对比维度 | premain（静态加载） | agentmain（动态加载） |
|---------|-------------------|---------------------|
| 加载时机 | JVM 启动时，main 之前 | JVM 运行中任意时刻 |
| 启动方式 | `-javaagent:xxx.jar` | Attach API 动态附加 |
| 使用场景 | APM 探针（SkyWalking） | 热诊断工具（Arthas） |
| 类增强 | `retransform`（推荐） | `retransform` |
| 限制 | 无 | 已经加载的类需要 retransform |

SkyWalking 使用 premain 方式，确保在业务类加载之前就注册好 Transformer。

### Q4: 所有 APM 探针都要做类加载隔离吗？三种主流方案有什么区别？

**是的，所有生产级 APM 都必须做类加载隔离**，否则一定会遇到依赖版本冲突、类重复加载、SPI 污染等问题。但实现思路有三种完全不同的路线：

| APM | 隔离方案 | 核心原理 | 设计哲学 | 风险等级 |
|------|---------|---------|---------|---------|
| SkyWalking | **自定义 ClassLoader** | AgentClassLoader 隔离 Agent 依赖与业务依赖 | 保守稳定 | ★★☆☆☆ |
| Pinpoint | **打破双亲委派 + Bootstrap 注入** | 核心类注入 Bootstrap ClassLoader，插件独立加载器 | 极致适配 | ★★★★☆ |
| OpenTelemetry | **Maven Shade 影子类** | 构建时重命名所有依赖包名，从根源避免冲突 | 通用兼容 | ★☆☆☆☆ |

**一句话总结三种路线：

- **SkyWalking**：我建一堵墙（ClassLoader），你在墙那边，我在墙这边，互不干扰
- **Pinpoint**：我爬到最高的地方（Bootstrap），所有人都能看到我，我也能看到所有人
- **OTel**：我改个名字（shaded），你认不出我，就不会和我冲突了

> ⚠️ OTel 的影子类方案风险最低，但代价是 Agent JAR 包变大（20MB+），且堆栈可读性差。SkyWalking 的 ClassLoader 方案最平衡，Pinpoint 的方案兼容性最强但风险最高。

### Q5: 如果 Agent 配置错误，会不会导致业务应用启动失败？

**不会**。SkyWalking Agent 的设计原则是**"Agent 失败不影响业务"**：

1. 所有 Agent 初始化代码都有 try-catch 保护
2. 如果 Agent 配置错误（如 OAP 地址不可达），Agent 会降级运行（不创建 Span）
3. 业务线程不会因为 Agent 异常而中断
4. Span 创建和上报都是异步的，不会阻塞业务线程

---

## 延伸阅读

- Java Agent 官方规范：[java.lang.instrument (Java SE 11)](https://docs.oracle.com/en/java/javase/11/docs/api/java.instrument/java/lang/instrument/package-summary.html)
- ByteBuddy 官方文档：[https://bytebuddy.net/](https://bytebuddy.net/)
- SkyWalking Agent 源码入口：`apm-agent-core/src/main/java/org/apache/skywalking/apm/agent/SkyWalkingAgent.java`