# 15 - Spring AI 详解

> Spring AI 是 Spring 官方的 AI 应用框架，已成 Java AI 开发事实标准。本章是本系列重点--Advisor 链机制（类似 AOP/拦截器）是 Spring AI 的灵魂，也是 Java 工程师最容易上手的点。面试「你用 Spring AI 怎么做的」必考。

---

## 一、Spring AI 是什么

### 1.1 定位

- **Spring 官方 AI 应用框架**：把 Spring 的设计哲学（约定优于配置、自动配置、抽象接口）带到 AI 应用。
- **目标**：提供统一的 AI 抽象（ChatClient/Embedding/VectorStore/Tool），屏蔽模型厂商差异，像 Spring Data 屏蔽数据库差异一样。
- **地位**：2025 年起已成 Java AI 开发事实标准，尤其配合 Spring AI Alibaba 覆盖国内场景。

### 1.2 核心设计

- **统一抽象 + 可插拔实现**：定义 `ChatModel`/`EmbeddingModel`/`VectorStore` 接口，各厂商提供实现（OpenAI/通义/Milvus/Pgvector...）。换模型/换库只改配置。
- **Spring Boot 自动配置**：starter 引入即自动装配，`@Autowired` 就能用。
- **Advisor 链**：类似 AOP/拦截器，在 LLM 调用前后插入横切逻辑（记忆/检索/安全/日志）。

> 💡 **面试金句**：Spring AI 是 Spring 官方 AI 框架，核心是统一抽象（ChatClient/Embedding/VectorStore/Tool 屏蔽厂商差异）+ Advisor 链（类似 AOP 的横切机制）。换模型换库只改配置，已成 Java AI 事实标准。

---

## 二、核心抽象 ★

| 抽象 | 作用 | 关键类 |
|------|------|--------|
| **ChatClient** | LLM 调用入口（Builder 式） | `ChatClient.builder().prompt().user().call()` |
| **ChatModel** | 底层模型封装 | OpenAiChatModel 等 |
| **EmbeddingModel** | 文本向量化 | OpenAiEmbeddingModel 等 |
| **VectorStore** | 向量库统一接口 | MilvusVectorStore / PgVectorStore |
| **ToolCallback** | 工具调用封装 | @Tool 注解自动生成 |
| **Advisor** | 横切逻辑（拦截器） | `BaseAdvisor` / `RequestResponseAdvisor` |
| **ChatMemory** | 会话记忆 | MessageWindowChatMemory |
| **Document** | 文档/片段抽象 | Document + Metadata |

### 2.1 ChatClient（最常用）

```java
// Builder 式，链式调用
String answer = chatClient.prompt()
    .system("你是助手")
    .user(question)
    .tools(orderTools)
    .call()
    .content();

// 流式
Flux<String> stream = chatClient.prompt().user(question).stream().content();

// 结构化输出
Sentiment s = chatClient.prompt().user(text).call().entity(Sentiment.class);
```

---

## 三、Advisor 链机制 ★ 核心

### 3.1 什么是 Advisor

- **类比 AOP/拦截器**：在 LLM 调用前后插入横切逻辑，不改业务代码。
- 和 Spring AOP/Servlet Filter 一个思路，Java 工程师秒懂。
- 解决：记忆注入、RAG 检索、安全审核、日志 Trace 等横切关注点。

### 3.2 内置 Advisor

| Advisor | 作用 |
|---------|------|
| **MessageChatMemoryAdvisor** | 自动注入/管理会话记忆 |
| **QuestionAnswerAdvisor** | 自动 RAG 检索注入上下文 |
| **SafeGuardAdvisor** | 输入/输出安全审查 |
| **SimpleLoggerAdvisor** | 日志记录 |

### 3.3 工作流程

```
请求 -> [Advisor1前置] -> [Advisor2前置] -> ... -> LLM 调用
                                                    ↓
响应 <- [Advisor2后置] <- [Advisor1后置] <- ... ←---
```

- 前置 Advisor 改请求（如注入记忆/检索片段），后置 Advisor 改响应（如审核/日志）。

### 3.4 使用示例

```java
// 链式配置多个 Advisor
ChatClient client = chatClientBuilder
    .defaultSystem("你是助手")
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build(),       // 记忆
        questionAnswerAdvisor,                                        // RAG
        SafeGuardAdvisor.builder().addBlockedWords(...).build()      // 安全
    )
    .build();

// 调用时自动走 Advisor 链：记忆注入->RAG检索->安全检查->调模型->安全审核->返回
String answer = client.prompt().user(question)
    .advisors(a -> a.param(CONVERSATION_ID, sessionId))
    .call().content();
```

### 3.5 自定义 Advisor

```java
public class TraceAdvisor implements BaseAdvisor {
    @Override
    public AdvisedRequest before(AdvisedRequest req) {
        // 前置：记开始时间、注入上下文
        return req;
    }
    @Override
    public AdvisedResponse after(AdvisedResponse resp) {
        // 后置：记录 Trace、token、延迟
        return resp;
    }
}
```

- 想加任何横切逻辑（限流/审计/Prompt 改写/缓存），写一个 Advisor 挂上即可。

> 💡 **面试金句**：Advisor 链是 Spring AI 的灵魂--类似 AOP/拦截器，在 LLM 调用前后插横切逻辑。记忆注入、RAG 检索、安全审核、日志 Trace 全做成 Advisor，业务代码干净。Java 工程师天然理解这个模式。

---

## 四、Tool Calling

### 4.1 @Tool 声明

```java
@Component
public class OrderTools {
    @Tool(description = "查订单物流状态")
    public String getLogistics(@ToolParam(description="订单号") String orderId) { ... }
}

// 注册
chatClient.prompt().user(q).tools(orderTools).call();
```

- 详见 04/12 章。Spring AI 自动生成 JSON Schema、处理调用循环。

### 4.2 ToolCallback 抽象

- @Tool 注解的方法会被包装成 `ToolCallback`。
- MCP Server 的工具也转成 ToolCallback，统一调用。所以本地工具和 MCP 工具用起来一样。

---

## 五、RAG 与 VectorStore

### 5.1 VectorStore 统一接口

```java
// 写入（自动调 Embedding）
vectorStore.add(List.of(new Document("文本", Map.of("source","doc1"))));

// 检索
List<Document> docs = vectorStore.similaritySearch(
    SearchRequest.builder().query("问题").topK(5).build());
```

- 换库只改 Bean 配置：MilvusVectorStore / PgVectorStore / ElasticsearchVectorStore / QdrantVectorStore。

### 5.2 QuestionAnswerAdvisor（RAG 一键接入）

```java
Advisor rag = QuestionAnswerAdvisor.builder(vectorStore).build();
ChatClient client = builder.defaultAdvisors(rag).build();
// 之后每次调用自动：Query->检索->拼上下文->调模型
```

- 一个 Advisor 搞定 RAG，业务无感。

---

## 六、Spring Boot 生产配置

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    openai:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
      embedding:
        options:
          model: text-embedding-v3
```

```java
@Configuration
public class AiConfig {
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, ChatMemory memory, VectorStore vs) {
        return builder
            .defaultSystem("你是企业助手")
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(memory).build(),
                QuestionAnswerAdvisor.builder(vs).build()
            )
            .build();
    }

    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder().maxMessages(20).build();
    }
}
```

---

## 七、与 LangChain4j 选型（呼应 14 章）

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 风格 | ChatClient Builder + Advisor 链 | AiServices 接口式 |
| 横切 | Advisor（AOP 式） | 手动组合组件 |
| Spring 集成 | 原生 | starter 接入 |
| 国内生态 | Spring AI Alibaba 官方 | 需自接 |
| 适合 | Spring Boot 企业项目 | 非 Spring / 灵活组合 |

- **推荐**：Spring Boot 项目（尤其国内）优先 Spring AI。横切逻辑（记忆/RAG/安全/Trace）用 Advisor 最干净，符合 Spring 习惯。

---

## 八、生产踩坑

1. **Advisor 顺序敏感**：前置 Advisor 按链顺序执行，顺序错可能影响（如安全检查要在检索后？还是前？）。想清楚顺序。
2. **ChatMemory 默认内存**：重启丢。生产换 Redis/JDBC 持久化实现。
3. **VectorStore 换库要重建**：不同库的向量索引不通用，换库要重新 Embedding 入库。
4. **Advisor 改了请求没返回**：前置 Advisor 改了 AdvisedRequest 要返回，后置改 AdvisedResponse 要返回，别漏。
5. **版本变动**：Spring AI 迭代快，Advisor API 可能在版本间变（如 1.0 前后接口调整），锁版本看文档。
6. **结构化输出兼容**：通义等模型的 JSON Mode/Structured Output 支持可能有差异，测试验证。
7. **流式 + Advisor**：流式下 Advisor 后置处理要注意（响应是流不是整体），部分 Advisor 在流式下行为不同。

---

## 九、常见面试题

1. **Spring AI 是什么？核心设计？**
   Spring 官方 AI 框架。核心是统一抽象（ChatClient/Embedding/VectorStore/Tool 屏蔽厂商差异，换模型换库只改配置）+ Advisor 链（类似 AOP 的横切机制）。已成 Java AI 事实标准。

2. **Advisor 链是什么？有什么用？**
   类似 AOP/拦截器，在 LLM 调用前后插横切逻辑。记忆注入（MessageChatMemoryAdvisor）、RAG（QuestionAnswerAdvisor）、安全（SafeGuardAdvisor）、日志 Trace 全做成 Advisor，业务代码干净。前置改请求，后置改响应。

3. **Spring AI 怎么做 RAG？**
   用 QuestionAnswerAdvisor + VectorStore：Advisor 自动做 Query->检索->拼上下文。VectorStore 统一接口屏蔽 Milvus/Pgvector/ES 差异。换库只改 Bean 配置。

4. **Spring AI 怎么做记忆？**
   ChatMemory 抽象（MessageWindowChatMemory 滑动窗口）+ MessageChatMemoryAdvisor 自动注入管理。按 conversationId 隔离会话。可换 Redis/JDBC 持久化跨重启。

5. **ChatClient 怎么用？**
   Builder 链式：chatClient.prompt().system().user().tools().call().content()。支持流式（stream()）和结构化输出（entity(Class)）。默认配置用 defaultSystem/defaultAdvisors/defaultTools 设。

6. **Spring AI 和 LangChain4j 怎么选？**
   Spring AI 是 Spring 官方、ChatClient+Advisor 链、Spring Boot 原生、国内有 Alibaba 扩展；LangChain4j 是社区、AiServices 接口式、不依赖 Spring。Spring Boot 企业项目（尤其国内）优先 Spring AI。

7. **自定义 Advisor 怎么写？**
   实现 BaseAdvisor（或 RequestResponseAdvisor），重写 before（改请求）和 after（改响应）。像写 AOP 切面，用于限流/审计/Prompt 改写/缓存等横切逻辑。

8. **ToolCallback 是什么？**
   工具调用的统一封装。@Tool 注解的方法自动包装成 ToolCallback，MCP Server 的工具也转成 ToolCallback。所以本地工具和 MCP 工具调用方式统一，注册给 ChatClient.tools() 即可。

---

## 十、资料勘误与重点提醒

1. **「Spring AI 就是调 LLM 的封装」不全面**：它的核心价值是统一抽象 + Advisor 链，不只是封装 API。资料常只讲调 LLM 忽略 Advisor 横切能力，而那才是工程化关键。
2. **「Advisor 等于 AOP」要讲清差异**：类比 AOP 但作用在 LLM 调用链，前置改请求后置改响应，不是通用方法切面。资料偶尔等同导致误解。
3. **「ChatMemory 默认够用」是坑**：默认内存实现重启丢。资料示例常用默认不换持久化，生产必须换 Redis/JDBC。多会话/重启场景必换。
4. **Advisor 顺序**：资料少提顺序敏感性。安全审核放检索后还是前、记忆注入和 RAG 谁先，顺序影响结果，要设计。
5. **版本 API 变动**：Spring AI 迭代快，Advisor 接口在 1.0 前后变过。资料代码可能过时，以官方最新文档为准。
6. **「VectorStore 换库无感」不完全**：接口统一，但向量索引不通用，换库要重新 Embedding 入库。资料常说「改配置即可」漏了重建成本。

---

> 下一章「16-Spring AI Alibaba 与国内生态」：国内场景怎么用，通义/百炼/DashScope 接入。
