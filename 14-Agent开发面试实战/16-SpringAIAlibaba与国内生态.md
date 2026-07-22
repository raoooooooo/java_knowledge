# 16 - Spring AI Alibaba 与国内生态

> 国内项目绕不开合规、私有化、国产模型。Spring AI Alibaba 是阿里基于 Spring AI 的扩展，深度整合通义/百炼，覆盖 Graph/RAG/Observability。本章讲国内场景怎么落地。面试「国内怎么选型」必考。

---

## 一、Spring AI Alibaba 是什么

### 1.1 定位

- **阿里基于 Spring AI 的扩展项目**：在 Spring AI 标准抽象之上，提供国内模型（通义）、国内中间件、Graph 编排、RAG、可观测等增强。
- **官方背书 + 本土化**：Spring 官方背书 Spring AI 标准框架，Alibaba 背书国内扩展，双保险。
- **地位**：国内 Java AI 开发的首选组合（Spring AI + Spring AI Alibaba + 通义）。

### 1.2 为什么需要它

- Spring AI 原生对接 OpenAI 系，国内模型需要适配（虽然通义提供 OpenAI 兼容端点，但原生 SDK 功能更全）。
- 国内中间件（Nacos/OSS/RocketMQ）集成需要本土化封装。
- Graph 编排、RAG 工程、可观测等开箱即用模块，省自研。
- 合规和私有化部署有官方支持。

> 💡 **面试金句**：Spring AI Alibaba 是阿里基于 Spring AI 的扩展，深度整合通义/百炼，提供 Graph 编排/RAG 工程/可观测等开箱即用模块。国内项目用「Spring AI + Alibaba + 通义」是官方背书 + 本土化首选组合。

---

## 二、通义千问 / 百炼 / DashScope ★

### 2.1 三个名词理清

| 名词 | 是什么 |
|------|--------|
| **通义千问 Qwen** | 阿里的大语言模型（系列：qwen-plus/turbo/max） |
| **百炼（Bailian）** | 阿里的 AI 应用平台，一站式建 Agent/RAG/知识库 |
| **DashScope** | 阿里的模型服务 API 平台（通义/Embedding/Rerank 都在这调） |

- 关系：通义是模型，DashScope 是调模型的 API 平台，百炼是基于模型的上层应用平台。
- 开发者主要对接 DashScope（API）或百炼（平台）。

### 2.2 DashScope 接入方式

**方式一：OpenAI 兼容端点（最简，Spring AI 原生）**

```yaml
spring:
  ai:
    openai:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode
      chat:
        options:
          model: qwen-plus
```

- 通义提供 OpenAI 兼容协议，Spring AI 的 OpenAI starter 直接能用，改 base-url 即可。
- 优点：一套代码切模型。缺点：部分高级特性可能不支持。

**方式二：Spring AI Alibaba 原生 SDK（功能全）**

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    alibaba:
      dashscope:
        api-key: ${DASHSCOPE_API_KEY}
        chat:
          options:
            model: qwen-plus
```

- 用阿里原生 SDK，功能更全（如多模态、特定参数、Rerank 模型 gte-rerank 原生支持）。
- Spring AI Alibaba 封装成 Spring AI 的 ChatModel/EmbeddingModel，统一抽象不变。

### 2.3 通义模型矩阵

| 模型 | 用途 |
|------|------|
| qwen-plus / qwen-max | 通用对话（plus 性价比，max 最强） |
| qwen-turbo | 轻量快速、降本 |
| qwen-long | 长上下文 |
| text-embedding-v3 | Embedding |
| gte-rerank | Rerank 重排 |
| qwen-vl | 多模态（视觉） |

---

## 三、Spring AI Alibaba 的扩展能力

### 3.1 Graph 编排（重点，呼应 17 章）

- 提供**有状态图编排**能力（类似 Python LangGraph）：State/Node/Edge/条件路由/循环。
- 比 Spring AI 原生 Advisor 链更适合复杂流程（多分支/循环/人在环）。
- 详见 17 章。

### 3.2 RAG 工程

- 开箱即用的 RAG 模块：文档解析/切分/Embedding/入库/检索/Rerank 全套。
- `DocumentTransformer`（如 MarkdownDocumentTransformer 语义切分）、`DashScopeRerankModel` 等。

### 3.3 可观测

- 集成 OTel/SkyWalking 的 AI 埋点，Trace/Token/延迟自动上报。
- 配合阿里云 ARMS 或自建 Langfuse。

### 3.4 多 Agent

- 提供 Multi-Agent 编排能力（呼应 18 章）。

### 3.5 国内中间件集成

- Nacos（配置/注册）、OSS（对象存储/快照）、RocketMQ（异步任务）、OOS 等。

---

## 四、国产模型选型对比

| 模型 | 出身 | 强项 | 适用 |
|------|------|------|------|
| **通义 Qwen** | 阿里 | 生态（Alibaba 扩展）、开源可私有化、多模态 | 国内首选、企业落地 |
| **DeepSeek** | 深度求索 | 推理强、价格极低、开源 | 推理任务、降本 |
| **GLM/智谱** | 智谱 | 合规、可私有化、多模态 | 政企合规 |
| **文心/星火** | 百度/讯飞 | 生态、合规 | 特定行业 |

### 选型建议

- **国内通用 + 生态**：通义（Spring AI Alibaba 原生支持）。
- **推理任务 + 降本**：DeepSeek（极便宜）。
- **政企合规 + 私有化**：GLM 或通义开源版自部署。
- **多模型混合**：用 Spring AI 抽象，不同任务路由不同模型（见 22 章）。

---

## 五、合规与私有化部署 ★

### 5.1 合规要点

- **数据不出域**：敏感场景（金融/政企/医疗）数据不能出，必须私有化或用合规云。
- **内容合规**：输入输出过审（涉政/暴恐/违规），国内必须。
- **备案**：面向公众的生成式 AI 服务需算法备案（网信办）。
- **日志留存**：调用日志按要求留存可审计。

### 5.2 私有化部署方案

- **开源模型自部署**：Qwen 开源版 / DeepSeek / GLM 开源版 + 自建推理服务（vLLM，见 24 章）。
- **合规云**：阿里云/华为云等合规区域部署。
- **混合**：非敏感走云端 API，敏感走私有化，模型路由分流（22 章）。

### 5.3 私有化的工程挑战

- 推理性能：要 vLLM/TGI 加速，否则慢。
- 运维：自部署模型要运维（GPU 资源、版本、监控）。
- 效果：开源版可能略逊于闭源旗舰，要评测。
- 成本：GPU 贵，小团队可能不如直接买 API。

> 💡 **面试金句**：国内合规要求数据不出域、内容过审、算法备案、日志留存。私有化部署用开源模型（Qwen/DeepSeek/GLM）+ vLLM 推理加速，敏感场景必选，代价是运维和 GPU 成本。非敏感可走合规云 API 降本。

---

## 六、实战：通义 RAG Agent（Spring AI Alibaba）

```java
@Configuration
public class AgentConfig {
    @Bean
    public ChatClient chatClient(ChatClient.Builder builder, VectorStore vs) {
        return builder
            .defaultSystem("你是企业助手，基于知识库回答并标注来源")
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory()).build(),
                QuestionAnswerAdvisor.builder(vs).build()
            )
            .build();
    }

    @Bean
    public VectorStore vectorStore(EmbeddingModel embedding) {
        // 用 Pgvector 或 Milvus
        return PgVectorStore.builder(jdbcTemplate).dimensions(1024).build();
    }

    @Bean
    public RerankModel rerankModel() {
        return new DashScopeRerankModel(apiKey, "gte-rerank");  // 通义 Rerank
    }
}
```

- 国产全家桶：通义 LLM + 通义 Embedding + 通义 Rerank + Spring AI Alibaba 框架，一套打通。

---

## 七、生产踩坑

1. **兼容端点功能不全**：通义 OpenAI 兼容端点方便但部分特性（结构化输出/并行调用/特定参数）可能不支持。要原生 SDK 兜底。
2. **模型版本别名漂移**：用 `qwen-plus` 别名厂商更新会漂移，生产用固定版本号。
3. **私有化推理慢**：自部署开源模型不做加速会慢，用 vLLM/TGI。
4. **GPU 成本**：私有化要 GPU，小团队可能不如买 API，评估 ROI。
5. **合规备案**：面向公众的 AI 服务要算法备案，别忽略监管。
6. **内容审核遗漏**：国内必须过审，别只靠模型自律，要加审核中间件。
7. **多模态计费**：图片/视频输入按分辨率算 token，成本易超。

---

## 八、常见面试题

1. **Spring AI Alibaba 是什么？为什么要用它？**
   阿里基于 Spring AI 的扩展，深度整合通义/百炼/DashScope，提供 Graph/RAG/可观测等开箱即用模块和国内中间件集成。国内项目用它获得官方背书 + 本土化 + 合规支持，是 Spring AI + 通义的首选组合。

2. **通义千问、百炼、DashScope 是什么关系？**
   通义千问是模型（Qwen 系列）；DashScope 是调模型的 API 平台；百炼是基于模型的上层应用平台（建 Agent/RAG/知识库）。开发者对接 DashScope（API）或百炼（平台）。

3. **怎么接入通义千问？**
   两种：OpenAI 兼容端点（Spring AI OpenAI starter 改 base-url，最简但功能可能不全）或 Spring AI Alibaba 原生 SDK（功能全，原生支持 Rerank/多模态等）。推荐国内项目用 Alibaba 原生。

4. **国产模型怎么选？**
   通义（生态好、Alibaba 扩展原生支持、可私有化）国内首选；DeepSeek（推理强、极便宜）降本推理；GLM（合规、可私有化）政企。多模型可路由分流（22 章）。

5. **国内合规要注意什么？**
   数据不出域（敏感场景私有化）、内容过审（涉政/暴恐）、算法备案（网信办）、日志留存审计。面向公众的 AI 服务必须备案，别忽略监管。

6. **怎么做私有化部署？**
   开源模型（Qwen/DeepSeek/GLM 开源版）+ vLLM 推理加速自建。敏感场景必选，代价是运维和 GPU 成本。小团队评估 ROI，可能不如买合规云 API。

7. **Spring AI Alibaba 的 Graph 是什么？**
   有状态图编排能力（类似 LangGraph）：State/Node/Edge/条件路由/循环，比 Spring AI 原生 Advisor 链更适合复杂流程编排。见 17 章。

8. **你项目国内怎么落地的？**
   （参考）Spring AI + Spring AI Alibaba + 通义（DashScope）。LLM 用 qwen-plus，Embedding 用 text-embedding-v3，Rerank 用 gte-rerank，向量库 Pgvector/Milvus。Advisor 链做记忆+RAG+安全。敏感场景部分私有化（Qwen 开源 + vLLM）。

---

## 九、资料勘误与重点提醒

1. **「通义只能用兼容端点」不全面**：通义既有 OpenAI 兼容端点（简单）也有原生 DashScope SDK（功能全）。资料常只讲一种。需要 Rerank/多模态等用原生 SDK。
2. **「私有化一定更安全更好」不严谨**：私有化数据不出域更安全，但开源版效果可能略逊旗舰、运维成本高、需 GPU。资料常只讲安全不讲代价。要评估 ROI。
3. **「百炼和 DashScope 是一回事」是混淆**：DashScope 是模型 API 平台，百炼是上层应用平台（建 Agent/知识库）。资料偶尔混用。
4. **合规备案少被提及**：资料讲技术不讲监管。面向公众的生成式 AI 服务需算法备案（网信办），这是国内上线的硬门槛，不是可选项。
5. **「用别名模型方便」是坑**：资料示例常用 `qwen-plus` 别名。生产要固定版本号防漂移，否则厂商更新导致输出变化、评测失效。
6. **多模态计费**：资料少提图片/视频输入的 token 成本。多模态按分辨率算 token，成本易超，要监控。

---

> 下一章「17-复杂流程编排」：简单 Advisor 链不够用时，怎么编排复杂流程（条件路由/循环/人在环）。
