# 14 - LangChain4j 详解

> LangChain4j 是 LangChain 的 Java 实现，社区活跃、轻量灵活。本章讲它的核心组件、Spring Boot 集成、与 Spring AI 的选型对比。面试「你用哪个 Java AI 框架」必考。

---

## 一、LangChain4j 是什么

### 1.1 定位

- **LangChain 的 Java 版**：把 Python LangChain 的核心能力（LLM 调用、工具、记忆、RAG、Agent）移植到 Java。
- **轻量、灵活、社区活跃**：不依赖 Spring，可独立用，也能很好集成 Spring Boot。
- **目标**：让 Java 生态有和 Python 对等的 AI 应用开发能力。

### 1.2 和 Python LangChain 的关系

- 概念相通（Chain/Agent/Tool/Memory/Retriever），但**不是逐行移植**，按 Java 习惯重新设计（用接口、Builder、注解）。
- API 比 Python 版更类型安全、更规整。
- 功能上 Python 版更全更快（新特性先在 Python 出），LangChain4j 有滞后但核心跟得上。

---

## 二、核心组件

| 组件 | 作用 | 对应概念 |
|------|------|---------|
| **ChatLanguageModel** | 调 LLM 对话/生成 | LLM 封装 |
| **EmbeddingModel** | 文本向量化 | Embedding |
| **EmbeddingStore** | 向量存储与检索 | 向量库 |
| **AiServices** | 声明式 Agent 接口（类似 Feign） | Agent 抽象 |
| **@Tool** | 声明工具 | Function Calling |
| **ChatMemory** | 会话记忆 | Memory |
| **RetrievalAugmentor** | RAG 检索增强 | RAG |
| **DocumentSplitter** | 文档切分 | Chunking |

---

## 三、AiServices：声明式 Agent ★ 核心

### 3.1 像调接口一样用 Agent

- LangChain4j 最有特色的设计：**定义一个 Java 接口，框架生成实现**，像 MyBatis Mapper / OpenFeign 那样。
- 把「组装 Prompt + 调模型 + 工具循环 + 记忆」全隐藏在接口背后。

```java
// 1. 定义接口
interface Assistant {
    String chat(String userMessage);
}

// 2. 用 AiServices 构建实现
Assistant agent = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)              // 接哪个模型
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))  // 记忆
    .tools(orderTools, weatherTools)       // 工具
    .systemMessage("你是电商助手")          // 系统提示
    .build();

// 3. 像调普通方法一样用
String answer = agent.chat("帮我查下订单ORD123的物流");
// 框架自动：组装上下文->调模型->要调工具则执行回灌->循环->返回
```

### 3.2 结构化输出

```java
interface SentimentAnalyzer {
    Sentiment analyze(String text);   // 直接返回 Java 对象
}
SentimentAnalyzer analyzer = AiServices.builder(SentimentAnalyzer.class)
    .chatLanguageModel(model).build();
Sentiment s = analyzer.analyze("这手机太差了");   // 自动结构化
```

- 返回类型是 Java 对象，框架自动让模型输出 JSON 并反序列化。

### 3.3 @Tool 声明工具

```java
class OrderTools {
    @Tool("根据订单号查物流状态")
    String getLogistics(@P("订单号") String orderId) { ... }

    @Tool("申请退款")
    String refund(@P("订单号") String orderId) { ... }  // 危险操作应加确认
}

AiServices.builder(Assistant.class)
    .tools(new OrderTools())   // 注册
    .build();
```

---

## 四、Spring Boot 集成实战

### 4.1 依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
</dependency>
<!-- 国内模型用 OpenAI 兼容协议 -->
```

### 4.2 配置（通义千问）

```yaml
langchain4j:
  open-ai:
    chat-model:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
      model-name: qwen-plus
      temperature: 0.7
    embedding-model:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
      model-name: text-embedding-v3
```

### 4.3 完整 RAG Agent 示例

```java
@Configuration
public class AgentConfig {
    @Bean
    public Assistant agent(ChatLanguageModel model,
                            EmbeddingStore<TextSegment> store,
                            EmbeddingModel embeddingModel) {
        // RAG 检索增强
        RetrievalAugmentor rag = DefaultRetrievalAugmentor.builder()
            .queryEmbedder(TextEmbedder.builder().embeddingModel(embeddingModel).build())
            .contentRetriever(EmbeddingStoreContentRetriever.builder()
                .embeddingStore(store).embeddingModel(embeddingModel).maxResults(5).build())
            .build();

        return AiServices.builder(Assistant.class)
            .chatLanguageModel(model)
            .retrievalAugmentor(rag)               // RAG
            .chatMemoryProvider(id -> MessageWindowChatMemory.withMaxMessages(20))
            .tools(new OrderTools())
            .systemMessage("你是电商助手，基于知识库回答并标注来源")
            .build();
    }
}
```

- 一个 AiServices 调用串起：LLM + RAG + 记忆 + 工具，是 LangChain4j 的精髓。

---

## 五、与 Spring AI 选型对比 ★ 高频面试题

| 维度 | LangChain4j | Spring AI |
|------|-------------|-----------|
| **出身** | 社区（LangChain Java 版） | Spring 官方 |
| **依赖** | 不依赖 Spring，可独立 | 深度集成 Spring |
| **集成度** | 自己的 starter | Spring Boot 自动配置 |
| **抽象风格** | AiServices 接口式（像 Feign） | ChatClient Builder 式 |
| **扩展机制** | 灵活组合组件 | Advisor 链（类似 AOP/拦截器） |
| **生态** | 社区活跃，跟进 LangChain | Spring 官方背书，增长快 |
| **国内支持** | 需自己接通义 | Spring AI Alibaba 官方扩展 |
| **适合** | 不用 Spring / 想要灵活 / 已熟悉 LangChain | Spring Boot 项目 / 企业级 / 国内项目 |

### 选型建议

- **Spring Boot 项目、国内项目**：优先 Spring AI（+ Spring AI Alibaba），官方背书、国内生态好。
- **不用 Spring 或想要 LangChain 对等能力**：LangChain4j。
- **已熟悉 Python LangChain 团队**：LangChain4j 概念对得上，迁移快。
- 两者可以混用（都是 Java 库），但一般选一个主框架。

> 💡 **面试金句**：Spring AI 是 Spring 官方、深度集成 Spring Boot、Advisor 链扩展、Spring AI Alibaba 国内生态好；LangChain4j 是社区、不依赖 Spring、AiServices 接口式灵活。Spring Boot 国内项目优先 Spring AI。

---

## 六、生产踩坑

1. **API 版本变动快**：LangChain4j 迭代快，API 可能在版本间变，锁定版本看文档。
2. **AiServices 的接口限制**：返回类型要能反序列化，复杂泛型可能出问题，用简单 POJO。
3. **@Tool 的参数类型**：复杂对象参数模型易填错，尽量用基本类型 + 枚举。
4. **记忆跨会话**：默认内存记忆重启就丢，要换持久化实现（Redis/JDBC）。
5. **国内模型兼容**：通义/DeepSeek 用 OpenAI 兼容端点，但某些特性（如结构化输出）可能不完全兼容，测试验证。
6. **starter 冲突**：langchain4j-spring-boot-starter 和 spring-ai 不能混用同一能力，选一个。

---

## 七、常见面试题

1. **LangChain4j 是什么？和 Python LangChain 什么关系？**
   LangChain 的 Java 实现，把核心能力（LLM/工具/记忆/RAG/Agent）移植到 Java。概念相通但非逐行移植，按 Java 习惯重新设计（接口/Builder/注解），更类型安全。功能比 Python 版有滞后但核心跟得上。

2. **AiServices 是什么？**
   LangChain4j 的声明式 Agent 抽象：定义 Java 接口，用 AiServices.builder 生成实现，像 Feign/MyBatis Mapper。把组装 Prompt+调模型+工具循环+记忆全隐藏在接口背后，还能直接返回 Java 对象（结构化输出）。

3. **LangChain4j 和 Spring AI 怎么选？**
   Spring AI 是 Spring 官方、深度集成 Spring Boot、Advisor 链扩展、国内有 Spring AI Alibaba；LangChain4j 是社区、不依赖 Spring、AiServices 接口式灵活。Spring Boot 国内项目优先 Spring AI；不用 Spring 或要 LangChain 对等能力选 LangChain4j。

4. **LangChain4j 怎么集成 Spring Boot？**
   用 langchain4j-spring-boot-starter + langchain4j-open-ai-spring-boot-starter，配置 base-url 指向通义/DeepSeek 兼容端点，@Bean 注入 ChatLanguageModel，用 AiServices 构建 Agent。国内模型走 OpenAI 兼容协议。

5. **LangChain4j 怎么做 RAG？**
   RetrievalAugmentor + EmbeddingStoreContentRetriever：Query 向量化 -> 检索 EmbeddingStore Top-K -> 拼进 Prompt。配合 AiServices 的 retrievalAugmentor 一行接入。可加 Rerank、查询改写。

6. **@Tool 怎么用？工具太多怎么办？**
   方法加 @Tool("描述") + @P("参数描述")，注册到 AiServices.tools()。工具太多模型选错，要分组/按需加载/路由分流，控制单次 10~20 个。

7. **AiServices 怎么实现结构化输出？**
   接口方法返回 Java 对象（POJO/Record），框架自动让模型输出 JSON 并反序列化成该对象。比手写 Prompt 约束+解析更稳。

8. **你为什么选 LangChain4j/Spring AI？**
   （按实际答）参考：Spring Boot 项目 + 国内部署优先 Spring AI（官方背书 + Alibaba 扩展 + Advisor 链好做横切）；如果是非 Spring 或要 LangChain 概念对等选 LangChain4j。

---

## 八、资料勘误与重点提醒

1. **「LangChain4j = Python LangChain 翻译」不严谨**：不是逐行移植，按 Java 习惯重新设计（接口/Builder/注解）。资料偶尔说成「直接翻译」导致对 API 风格预期错。
2. **「LangChain4j 和 Spring AI 二选一」要带场景**：资料常直接说哪个更好。实际取决于是否用 Spring Boot、是否国内项目、团队是否熟悉 LangChain。没有绝对优劣。
3. **「AiServices 能做一切」有边界**：复杂流程编排（条件路由/循环/人在环）AiServices 力不从心，要 LangGraph 思想或 Spring AI Alibaba Graph（见 17 章）。资料常把 AiServices 当万能。
4. **国内模型兼容性**：资料示例常默认 OpenAI。通义/DeepSeek 用兼容端点，但结构化输出/并行调用等特性可能不完全兼容，要测试。资料少提兼容性差异。
5. **「LangChain4j 功能和 Python 版同步」不严谨**：新特性通常 Python 先出，LangChain4j 有滞后。资料偶尔暗示完全同步，实际有版本差。
6. **starter 不能混用**：langchain4j 和 spring-ai 的同类 starter 不宜同时引入同一能力（两个 ChatModel 自动配置冲突）。资料常不提醒这点。

---

> 下一章「15-Spring AI 详解」：Spring 官方 AI 框架，本系列主力，Advisor 链机制是核心。
