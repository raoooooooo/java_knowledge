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

SkyWalking Agent 作为基础架构组件，其依赖的库（如 gRPC、Protobuf、ByteBuddy 等）可能与业务应用的依赖**版本冲突**。例如：

- 业务应用使用 gRPC 1.30 → Agent 使用 gRPC 1.50
- 如果共用一个类加载器，会导致 `NoSuchMethodError` 等运行时错误

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

### 4. 字节码增强（ByteBuddy）

#### 4.1 为什么选择 ByteBuddy？

| 对比维度 | ByteBuddy（SkyWalking） | ASM（Pinpoint） | Javassist |
|---------|------------------------|-----------------|-----------|
| API 易用性 | ★★★★★ 流式 API | ★★☆☆☆ 需理解字节码指令 | ★★★★☆ 源码级 API |
| 性能 | ★★★★☆ 好 | ★★★★★ 最好 | ★★★☆☆ 一般 |
| 类型安全 | ✅ 编译期检查 | ❌ 运行时错误 | ❌ 运行时错误 |
| 社区活跃度 | ★★★★★ 高 | ★★★☆☆ 中 | ★★★☆☆ 中 |
| 学习曲线 | 低 | 高 | 中 |

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

### Q4: 如果 Agent 配置错误，会不会导致业务应用启动失败？

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