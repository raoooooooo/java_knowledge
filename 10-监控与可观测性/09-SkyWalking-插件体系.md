# 09 - SkyWalking 插件体系

## 核心概念

### 1. 插件全景图

SkyWalking Java Agent 的插件体系覆盖了 Java 生态中几乎所有主流框架和中间件：

```
┌─────────────────────────────────────────────────────────────────┐
│              SkyWalking Java Agent 插件体系全景                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HTTP 服务端                                              │  │
│  │  Tomcat 7/8/9/10 | Jetty | Undertow | SpringMVC         │  │
│  │  Spring WebFlux | Struts2 | Play | RESTEasy             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HTTP 客户端                                              │  │
│  │  HttpClient 3/4/5 | OkHttp | Spring RestTemplate         │  │
│  │  WebClient | Feign | HttpURLConnection                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  RPC 框架                                                 │  │
│  │  Dubbo 2/3 | gRPC | Motan | SofaRPC | Armeria           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  数据库                                                   │  │
│  │  MySQL | PostgreSQL | Oracle | H2 | MongoDB              │  │
│  │  ShardingSphere | MyBatis | Hibernate                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  缓存                                                     │  │
│  │  Redis(Jedis/Lettuce/Redisson) | Memcached               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  消息队列                                                   │  │
│  │  Kafka | RocketMQ | RabbitMQ | ActiveMQ | Pulsar         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  网关与微服务                                               │  │
│  │  Spring Cloud Gateway | Resilience4j | Sentinel          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  其他                                                     │  │
│  │  Quartz | ElasticJob | XXL-Job | ThreadPool | Logback    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. 插件分类

#### 2.1 内置插件（Built-in Plugins）

打包在 `skywalking-agent.jar` 中，默认启用。包括最常用的框架和中间件。
**位置**：`skywalking-agent.jar` 内的 `plugins/` 目录

#### 2.2 可选插件（Optional Plugins）

打包在 `skywalking-agent.jar` 中，但**默认不启用**。需要手动移动到 `plugins/` 目录才能激活。
**位置**：`skywalking-agent.jar` 内的 `optional-plugins/` 目录

**为什么需要可选插件？**
- 某些插件可能与业务代码冲突（如 Spring 注解拦截）
- 某些插件开销较大（如方法级追踪）
- 让用户按需选择

#### 2.3 引导插件（Bootstrap Plugins）

需要加载到 Bootstrap ClassLoader 的插件，用于拦截 JDK 核心类（如 `HttpURLConnection`）。
**位置**：`skywalking-agent.jar` 内的 `bootstrap-plugins/` 目录

**为什么需要 Bootstrap 插件？**
- Java 核心类库（如 `java.net.HttpURLConnection`）由 Bootstrap ClassLoader 加载
- 普通插件由 AgentClassLoader 加载，无法访问 Bootstrap 类
- Bootstrap 插件通过 `Instrumentation.appendToBootstrapClassLoaderSearch()` 注入到 Bootstrap ClassLoader

### 3. 插件增强四要素

每个插件必须定义以下四个关键要素：

```
┌──────────────────────────────────────────────────────────────┐
│                  插件增强四要素                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. witnessClasses（见证类）                                  │
│     └── 验证目标框架/库是否真的存在                            │
│     └── 如果没有，跳过增强（避免 ClassNotFoundException）      │
│                                                              │
│  2. classNameMatch（类匹配）                                  │
│     └── 指定需要增强的目标类                                   │
│     └── 支持：按名称 / 按注解 / 按父类 / 按接口               │
│                                                              │
│  3. methodsInterceptor（方法拦截器）                           │
│     └── 指定需要拦截的方法                                     │
│     └── 指定拦截逻辑的实现类                                   │
│                                                              │
│  4. 上下文传播（ContextCarrier / ContextSnapshot）             │
│     └── 跨进程：注入/提取 ContextCarrier                       │
│     └── 跨线程：捕获/恢复 ContextSnapshot                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4. 插件发现与加载机制

#### 4.1 插件配置文件

每个插件在 `META-INF/` 下有一个 `skywalking-plugin.def` 文件：

```properties
# 格式：插件定义类的全限定名
org.apache.skywalking.apm.plugin.spring.mvc.define.SpringMVCInstrumentation
org.apache.skywalking.apm.plugin.spring.mvc.define.RestControllerInstrumentation
org.apache.skywalking.apm.plugin.spring.mvc.define.ControllerAdviceInstrumentation
```

#### 4.2 插件加载流程

```
Agent 启动
  │
  ├── PluginFinder 扫描所有 META-INF/skywalking-plugin.def
  │     ├── 读取每个插件的全限定类名
  │     ├── 通过 PluginClassLoader 加载插件类
  │     └── 调用每个插件的 define() 方法
  │
  ├── 构建匹配器
  │     ├── 收集所有插件的 classNameMatch
  │     ├── 合并为 ElementMatcher
  │     └── 传递给 AgentBuilder
  │
  └── AgentBuilder 在类加载时触发匹配
        ├── 匹配成功 → 应用插件增强
        └── 匹配失败 → 跳过
```

### 5. 自定义插件开发实战

#### 5.1 场景：监控一个自定义 RPC 框架

假设有一个自定义 RPC 框架 `my-rpc`，需要开发 SkyWalking 插件来监控它的调用。

```java
// 自定义 RPC 框架的接口
package com.example.rpc;

public interface MyRpcClient {
    Response invoke(String serviceName, String methodName, Request request);
}

public class MyRpcServer {
    public Response handleRequest(Request request) { ... }
}
```

#### 5.2 步骤一：定义插件（Client 端）

```java
package com.example.rpc.plugin;

import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.*;
import org.apache.skywalking.apm.agent.core.plugin.match.*;
import static net.bytebuddy.matcher.ElementMatchers.*;

public class MyRpcClientInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    // 见证类：确认 my-rpc 框架存在
    @Override
    protected String[] witnessClasses() {
        return new String[] { "com.example.rpc.MyRpcClient" };
    }

    // 匹配实现了 MyRpcClient 接口的所有类
    @Override
    protected ClassMatch enhanceClass() {
        return ClassMatch.byInterface("com.example.rpc.MyRpcClient");
    }

    // 拦截 invoke 方法
    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoint() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named("invoke");
                }

                @Override
                public String getMethodsInterceptor() {
                    return "com.example.rpc.plugin.MyRpcClientInterceptor";
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }
}
```

#### 5.3 步骤二：实现拦截器（Client 端）

```java
package com.example.rpc.plugin;

import org.apache.skywalking.apm.agent.core.context.*;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.*;

public class MyRpcClientInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method,
                             Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) {
        String serviceName = (String) allArguments[0];
        String methodName = (String) allArguments[1];

        // 创建 Exit Span
        ContextCarrier carrier = new ContextCarrier();
        Span span = ContextManager.createExitSpan(
            serviceName + "/" + methodName,  // operationName
            carrier,                          // 上下文载体
            "my-rpc-server:9090"              // 对端地址
        );
        span.setComponent(MyRpcComponentDefine.MY_RPC);
        span.setLayer(SpanLayer.RPC);

        // 将 carrier 注入到 RPC 请求的 Attachments 中
        CarrierItem items = carrier.items();
        while (items.hasNext()) {
            CarrierItem item = items.next();
            ((Request) allArguments[2]).setAttachment(item.getHeadKey(), item.getHeadValue());
        }
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method,
                              Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) {
        Response response = (Response) ret;
        if (!response.isSuccess()) {
            ContextManager.activeSpan().errorOccurred();
        }
        ContextManager.stopSpan();
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method,
                                       Object[] allArguments, Class<?>[] argumentsTypes,
                                       Throwable t) {
        ContextManager.activeSpan().log(t);
    }
}
```

#### 5.4 步骤三：定义插件（Server 端）

```java
public class MyRpcServerInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    @Override
    protected ClassMatch enhanceClass() {
        return ClassMatch.byClassName("com.example.rpc.MyRpcServer");
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoint() {
        return new InstanceMethodsInterceptPoint[] {
            new DeclaredInstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    return named("handleRequest");
                }

                @Override
                public String getMethodsInterceptor() {
                    return "com.example.rpc.plugin.MyRpcServerInterceptor";
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }
}
```

#### 5.5 步骤四：注册插件

在 `META-INF/skywalking-plugin.def` 中添加：

```properties
com.example.rpc.plugin.MyRpcClientInstrumentation
com.example.rpc.plugin.MyRpcServerInstrumentation
```

#### 5.6 步骤五：打包部署

```bash
# 1. 编译插件
mvn clean package

# 2. 将插件 JAR 放入 SkyWalking Agent 的 plugins/ 目录
cp target/my-rpc-plugin-1.0.jar /path/to/skywalking-agent/plugins/

# 3. 重启应用，插件自动生效
```

### 6. 插件拦截器类型

| 拦截器类型 | 接口 | 使用场景 |
|-----------|------|---------|
| **实例方法环绕拦截** | `InstanceMethodsAroundInterceptor` | 最常用，拦截实例方法 |
| **静态方法环绕拦截** | `StaticMethodsAroundInterceptor` | 拦截静态方法 |
| **构造方法拦截** | `ConstructorInterceptPoint` | 拦截构造方法 |

**环绕拦截的三个回调**：

```java
public interface InstanceMethodsAroundInterceptor {
    // 方法执行前
    void beforeMethod(EnhancedInstance objInst, Method method,
                      Object[] allArguments, Class<?>[] argumentsTypes,
                      MethodInterceptResult result);

    // 方法执行后（正常返回）
    Object afterMethod(EnhancedInstance objInst, Method method,
                       Object[] allArguments, Class<?>[] argumentsTypes,
                       Object ret);

    // 方法抛出异常
    void handleMethodException(EnhancedInstance objInst, Method method,
                                Object[] allArguments, Class<?>[] argumentsTypes,
                                Throwable t);
}
```

---

## 常见面试题

### Q1: 可选插件和内置插件有什么区别？什么时候用可选插件？

| 对比维度 | 内置插件 | 可选插件 |
|---------|---------|---------|
| 默认状态 | 启用 | 禁用 |
| 位置 | `plugins/` | `optional-plugins/` |
| 激活方式 | 自动 | 手动移动到 `plugins/` |
| 使用场景 | 常见框架，冲突风险低 | 特殊场景，可能冲突 |

**需要可选插件的常见场景**：
- Spring 注解拦截器：某些业务代码使用了自定义注解，可能与通用拦截逻辑冲突
- 方法级追踪：性能开销较大，只在需要时启用
- 特定框架版本：某些框架版本有特殊行为，需要专用插件

### Q2: Bootstrap 插件和普通插件有什么区别？

| 对比维度 | 普通插件 | Bootstrap 插件 |
|---------|---------|---------------|
| 类加载器 | AgentClassLoader | Bootstrap ClassLoader |
| 目标类 | 业务类（如 Spring/Dubbo 类） | JDK 核心类（如 HttpURLConnection） |
| 加载方式 | 标准 AgentBuilder 增强 | `appendToBootstrapClassLoaderSearch()` |
| 使用场景 | 拦截第三方框架 | 拦截 JDK 核心 API |

**为什么需要 Bootstrap 插件？**
因为 JDK 核心类（如 `java.net.HttpURLConnection`）由 Bootstrap ClassLoader 加载，AgentClassLoader 无法访问，必须通过特殊方式注入。

### Q3: 如何开发一个自定义 SkyWalking 插件？简述关键步骤。

1. **定义插件类**：继承 `AbstractClassEnhancePluginDefine`，实现类匹配、方法匹配
2. **实现拦截器**：实现 `InstanceMethodsAroundInterceptor`，在 before/after/exception 中处理 Span
3. **处理上下文传播**：Exit Span 通过 `ContextCarrier` 注入上下文，Entry Span 提取上下文
4. **注册插件**：在 `META-INF/skywalking-plugin.def` 中声明插件类
5. **打包部署**：将插件 JAR 放入 Agent 的 `plugins/` 目录

### Q4: witnessClasses 的作用是什么？为什么需要它？

`witnessClasses`（见证类）的作用是**验证目标框架/库是否真的存在于类路径中**。

**为什么需要？**
- 如果 Agent 尝试增强一个不存在的类（如用户没有引入 SpringMVC），会抛出 `ClassNotFoundException` 或 `NoClassDefFoundError`
- Agent 的原则是"不要影响业务"，所以添加 `witnessClasses` 作为前置检查
- 如果见证类不存在 → 跳过该插件的增强（安全降级）
- 如果见证类存在 → 继续增强流程

---

## 延伸阅读

- SkyWalking 插件开发指南：[https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/plugin-development-guide/](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/plugin-development-guide/)
- ByteBuddy 官方 API：[https://bytebuddy.net/#/tutorial](https://bytebuddy.net/#/tutorial)