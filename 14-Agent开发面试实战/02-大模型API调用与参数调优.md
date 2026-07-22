# 02 - 大模型 API 调用与参数调优

> 本章是动手起点：怎么用 Java 调大模型 API、核心参数怎么调、流式怎么处理、Token 怎么算钱。面试官最爱从这里切入问「你实际用过哪个模型、怎么调的、参数怎么选」。

---

## 一、主流大模型 API 对比（2026 中）

| 模型 | 出身 | 特点 | 国内可用 | 典型用途 |
|------|------|------|---------|---------|
| **GPT 系列** | OpenAI | 能力强、生态标杆 | 需代理/合规问题 | 通用、复杂推理（海外业务） |
| **Claude** | Anthropic | 长上下文、代码强、Agent 友好 | 需代理 | 编码 Agent、长文档（海外） |
| **Gemini** | Google | 多模态强 | 受限 | 多模态（海外） |
| **通义千问 Qwen** | 阿里 | 国产第一梯队、开源可私有化、百炼平台 | ✅ 原生 | 国内首选、企业落地 |
| **DeepSeek** | 深度求索 | 推理强、价格极低、开源 | ✅ 原生 | 推理任务、降本 |
| **GLM/智谱** | 智谱 | 国产、合规、可私有化 | ✅ 原生 | 国内、政企合规 |
| **Llama / Qwen 开源版** | Meta/阿里 | 自部署、数据不出域 | ✅ 自建 | 私有化、敏感场景 |

> 💡 **国内项目首选**：通义千问（百炼平台）/ DeepSeek / GLM。合规、便宜、可私有化。海外业务才考虑 OpenAI/Claude。
>
> 💡 **选型口诀**：能力上限看 GPT/Claude，性价比看 DeepSeek/通义，合规私有化看 Qwen 开源版/GLM。

### 1.1 API 调用的统一范式

各家 API 长得不一样，但核心都是：

```
POST https://api.xxx.com/v1/chat/completions
Header: Authorization: Bearer <API_KEY>
Body:
{
  "model": "qwen-plus",
  "messages": [
    {"role": "system", "content": "你是助手"},
    {"role": "user", "content": "你好"}
  ],
  "temperature": 0.7,
  "stream": false
}
```

- **messages**：消息列表，role 有 `system`（系统指令，定人设）/ `user`（用户）/ `assistant`（模型历史回复）/ `tool`（工具结果）。
- **OpenAI 兼容协议**已成事实标准：通义/DeepSeek/GLM/月之暗面基本都兼容 OpenAI 的 `/v1/chat/completions` 格式，换 `base_url` 和 `api_key` 即可。**这是 Java 侧能一套代码切多模型的基础**。

---

## 二、核心参数详解 ★ 高频面试题

### 2.1 temperature（温度）

- **作用**：控制输出随机性。范围通常 0~2（默认 0.7~1.0）。
- **原理**：模型在每个位置输出一个词的概率分布，temperature 对 logits（未归一化的分数）除以 T 再 softmax。T 越小分布越尖锐（越确定），T 越大越平缓（越随机）。
  - `temperature=0`：近乎贪心解码（选概率最高的），输出最稳定。
  - `temperature 高`：低概率词也有机会被选，更有创造力但易跑偏。
- **选型**：
  - 事实问答/代码/分类/JSON 抽取 -> `0~0.3`（要稳）。
  - 通用对话/总结 -> `0.5~0.7`。
  - 创意写作/头脑风暴 -> `0.8~1.2`。

### 2.2 top_p（核采样）

- **作用**：另一种控制随机性的方式。只从累计概率达到 p 的候选词里采样（把长尾低概率词砍掉）。
- `top_p=0.9`：只考虑累计概率前 90% 的词。
- **和 temperature 的关系**：**一般只调一个**，不同时调（OpenAI 官方建议）。temperature 是「整体调尖锐度」，top_p 是「截断长尾」。

> ⚠️ **面试金句**：temperature 和 top_p 都是控制采样随机性，**通常二选一**。temperature 调整整条概率分布的尖锐度，top_p 截断长尾低概率词。生产事实类任务 temperature 设 0~0.3 求稳，创意类设高。

### 2.3 max_tokens（最大输出长度）

- 限制模型**输出**的 token 数（不是输入）。输出超过就截断。
- ⚠️ 注意和上下文窗口区分：上下文窗口是「输入+输出」的总上限。

### 2.4 stop（停止序列）

- 遇到指定字符串就停止输出。常用于：让模型输出结构化内容时，遇到结束标记停下；或在 Function Calling 里截断。

### 2.5 seed（随机种子）

- 设固定 seed + `temperature=0`，尽量让输出可复现。⚠️ 但因实现（批处理浮点、KV-cache），不保证 100% 一致，只能说「近似确定」。

### 2.6 presence_penalty / frequency_penalty

- 惩罚已出现/高频出现的词，减少重复。长文本生成时防「车轱辘话」。

### 2.7 参数选型速查表

| 场景 | temperature | top_p | 备注 |
|------|-------------|-------|------|
| 事实问答/抽取/分类 | 0~0.3 | 1.0（默认） | 要稳、可复现 |
| 代码生成 | 0~0.2 | 1.0 | 要准 |
| 通用对话 | 0.5~0.7 | 1.0 | 平衡 |
| 创意写作 | 0.8~1.2 | 0.9 | 要发散 |
| JSON 结构化输出 | 0~0.1 | 1.0 | 格式稳定，配合 JSON Mode |

---

## 三、流式输出（SSE）★ 必考

### 3.1 为什么要流式

- 一次生成几百 token 要几秒到几十秒，等完整结果用户体验极差。
- 流式：模型边生成边返回 token，前端边渲染（打字机效果），**首 token 时间（TTFT）** 大幅改善体感。

### 3.2 SSE 原理（呼应 06 章）

- 基于 HTTP，服务器单向推送事件流，`Content-Type: text/event-stream`。
- 每个chunk是 `data: {...}\n\n`，最后以 `data: [DONE]` 结束。
- LLM 流式天然是「服务器持续推 token」的单向场景，SSE 轻量、走 HTTP、自动重连、对前端友好，是流式输出事实标准（OpenAI/Anthropic/通义都用）。

### 3.3 Java 处理流式

**WebFlux 响应式（推荐，天然适合流）**：

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String msg) {
    return webClient.post()
        .uri("/v1/chat/completions")
        .header("Authorization", "Bearer " + apiKey)
        .body(BodyInserters.fromValue(Map.of(
            "model", "qwen-plus",
            "messages", List.of(Map.of("role","user","content",msg)),
            "stream", true            // 开流式
        )))
        .retrieve()
        .bodyToFlux(String.class)      // 每个chunk作为一个元素
        .takeUntil("[DONE]"::equals);  // 遇到结束标记停止
}
```

**Spring AI 流式（最简，强烈推荐）**：

```java
@RestController
public class ChatController {
    private final ChatClient chatClient;
    public ChatController(ChatClient.Builder builder) { this.chatClient = builder.build(); }

    @GetMapping(value = "/chat/stream", produces = "text/event-stream")
    public Flux<String> stream(@RequestParam String msg) {
        return chatClient.prompt().user(msg).stream().content();
        // Spring AI 内部已处理 SSE 分帧，一行搞定
    }
}
```

> 💡 **面试金句**：流式用 SSE（不是 WebSocket），因为 LLM 生成是服务器单向推 token 的场景，SSE 走 HTTP 轻量、自动重连、对前端友好。Java 侧用 WebFlux 的 Flux 响应式天然处理流，Spring AI 还封装了一行式流式 API。

### 3.4 流式踩坑

- **背压**：下游消费慢会积压，WebFlux 响应式自带背压，但要注意缓冲区。
- **超时**：流式连接要保持，网关/代理别把超时设太短（默认 30s 可能不够），Nginx 要关 `proxy_buffering`。
- **错误中断**：流到一半出错，要能向前端发一个错误事件再关闭，别静默断开。
- **token 计费**：流式仍按总 token 计费，不要以为流式就便宜。

---

## 四、Token 与计费 ★ 必考

### 4.1 Token 是什么

- 模型处理的最小单位，介于「词」和「字」之间。
- 英文：约 1 token ≈ 0.75 词（4 字符 ≈ 1 token）。
- 中文：1 个汉字 ≈ 1~2 token（视分词器，Qwen/GLM 对中文优化较好，汉字 token 率高于 GPT）。
- **输入和输出都算 token**，计费 = 输入 token 单价 + 输出 token 单价（输出通常贵 3~5 倍）。

### 4.2 怎么估算 token

- 粗估：中文 1 字 ≈ 1.5 token，英文 1 词 ≈ 1.3 token。
- 精确：用模型对应的 tokenizer（Qwen 的 `tiktoken`、各家 SDK 的 `countTokens`）。
- API 返回里有 `usage.prompt_tokens` / `completion_tokens` / `total_tokens`，**以这个为准计费**。

### 4.3 Token 经济学（成本控制意识）

- prompt 越长越贵：塞 10 万 token 上下文，光输入就花一次大钱。
- 历史对话累积：多轮对话要把历史塞回去，轮次越多越贵（这是上下文压缩的动因）。
- RAG 检索回来的片段也是 token，别一股脑塞 50 段。
- 缓存能省钱：相同 prompt 用 prompt caching（部分模型支持），或自己用语义缓存。

> 💡 **面试金句**：输入输出都算 token，输出贵 3~5 倍。成本控制三板斧：压缩上下文（减少输入）、用小模型（降单价）、缓存（复用结果）。详见 24 章。

---

## 五、Java 客户端实战

### 5.1 Spring AI（推荐，最简）

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<!-- 国产模型用 OpenAI 兼容协议，配 base-url 即可 -->
```

```yaml
spring:
  ai:
    openai:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode   # 通义百炼兼容模式
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
```

```java
@Service
public class ChatService {
    private final ChatClient chatClient;
    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是资深Java架构师，回答简洁")
            .build();
    }

    public String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

### 5.2 LangChain4j（轻量，灵活）

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
</dependency>
```

```java
ChatLanguageModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("DASHSCOPE_API_KEY"))
    .baseUrl("https://dashscope.aliyuncs.com/compatible-mode/v1")
    .modelName("qwen-plus")
    .temperature(0.7)
    .build();

String answer = model.generate("用一句话解释什么是RAG");
```

### 5.3 裸 OkHttp（理解底层，面试能讲原理）

```java
String body = """
{"model":"qwen-plus",
 "messages":[{"role":"user","content":"%s"}],
 "temperature":0.7,"stream":false}""".formatted(msg);

Request req = new Request.Builder()
    .url("https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions")
    .header("Authorization", "Bearer " + apiKey)
    .post(RequestBody.create(body, MediaType.parse("application/json")))
    .build();
try (Response resp = client.newCall(req).execute()) {
    System.out.println(resp.body().string());
}
```

> ⚠️ **API Key 严禁硬编码**：用环境变量/配置中心/Secret Manager。提交到 Git 是事故。CLAUDE.md 铁律2。

### 5.4 多模型切换（OpenAI 兼容协议的威力）

因为通义/DeepSeek/GLM 都兼容 OpenAI 协议，一套 Spring AI 代码，改 `base-url` 和 `model` 就能切模型。这是后面 22 章「模型路由降级」的基础。

---

## 六、生产踩坑

1. **超时设置**：LLM 响应慢，HTTP 客户端超时要设长（30~120s），流式更长。OkHttp/WebClient 默认超时不够。
2. **限流**：模型厂商有 QPM/TPM 限制（每分钟请求数/token 数），超了 429。自己要加客户端限流 + 重试退避。
3. **重试要退避**：429/5xx 重试用指数退避，别猛重试加剧限流。
4. **API Key 管理**：环境变量或配置中心，不同环境不同 key，定期轮换。
5. **流式代理问题**：Nginx 要 `proxy_buffering off`、`proxy_read_timeout` 调大，否则流被缓冲或断开。
6. **多模态/视觉**：图片也要算 token（按分辨率），别以为只算文本。
7. **模型版本固定**：生产用具体版本（`qwen-plus-20260715`）而非别名（`qwen-plus`），避免厂商更新导致输出漂移。这是评测回归的前提。

---

## 七、常见面试题

1. **temperature 和 top_p 区别？怎么选？**
   都是控制采样随机性，通常二选一。temperature 调整概率分布尖锐度（小=确定，大=随机），top_p 截断长尾低概率词（只从累计概率 p 的候选采样）。事实/代码/抽取设 temperature 低（0~0.3）求稳，创意设高。

2. **为什么 LLM 流式用 SSE 而不是 WebSocket？**
   LLM 生成是服务器单向持续推 token 的场景，SSE 走 HTTP 轻量、浏览器原生自动重连、对前端友好，足够用；WebSocket 是双向实时（聊天室/游戏），对单向推 token 太重。详见 06 章。

3. **temperature=0 输出就完全确定吗？**
   不一定。受实现影响（批处理浮点非确定性、KV-cache、批内 padding），同一输入不同次仍可能有微小差异，近似确定非绝对。设 seed 可提高可复现性但不保证 100%。

4. **Token 怎么算？输入输出都算钱吗？**
   都算。输入和输出分别计费，输出通常贵 3~5 倍。中文 1 字 ≈ 1~2 token，英文 1 词 ≈ 1.3 token。API 返回 usage 字段是计费依据。多轮对话历史也是输入 token，越长越贵。

5. **你们项目用哪个模型？为什么？**
   （如实答，参考）国内首选通义千问（百炼平台）或 DeepSeek：合规可商用、价格低、能力强。通义胜在生态（Spring AI Alibaba 原生支持）和私有化（开源版可自部署）；DeepSeek 胜在推理强+极便宜。海外业务用 GPT/Claude。

6. **怎么切换模型？**
   因为通义/DeepSeek/GLM 都兼容 OpenAI 协议，一套 Spring AI 代码改 base-url 和 model 即可切。抽象出模型调用层，配合配置中心动态切换，是模型路由降级的基础。

7. **流式输出在 Java 里怎么实现？有什么坑？**
   WebFlux 用 Flux 响应式天然处理，Spring AI 封装了一行式 stream()。坑：网关/代理要关 buffering、超时调大；背压控制；中途出错要发错误事件再关闭；流式仍按总 token 计费。

8. **一次 LLM 调用 RT 突然 30s，怎么排查？**
   先看是首 token 慢（TTFT）还是生成慢：TTFT 慢多是模型排队/负载，生成慢多是输出长（max_tokens 大）。看 Trace 里 prompt token 数是否暴涨（上下文太长）、是否触发了限流重试。客户端超时设够、加监控。

---

## 八、资料勘误与重点提醒

1. **「temperature 和 top_p 同时调更好」是误区**：不少博客建议两个都调。OpenAI 官方明确建议**二选一**，同时调效果难预测。面试答「二选一」更准。
2. **「temperature=0 就完全确定」不严谨**：资料常说「temperature=0 输出确定」。准确说法是「近似确定」，因浮点/KV-cache 仍可能有微小差异，设 seed 提高复现但不保证 100%。
3. **「流式输出省钱」是错的**：流式只是改善体感（首 token 快），总 token 和计费不变，甚至略多（流式分帧开销）。
4. **模型版本别用别名**：资料示例常写 `model: gpt-4` 这种别名，生产要用固定版本号。厂商更新别名会导致输出漂移，评测回归失效。
5. **国内模型不一定走 OpenAI 协议**：通义/DeepSeek/GLM 提供 OpenAI 兼容端点，但也有自家原生 SDK（如 DashScope 原生协议）。Java 侧建议用兼容端点统一，Spring AI/LangChain4j 都支持。
6. **上下文窗口 ≠ max_tokens**：资料常混。上下文窗口是「输入+输出」总上限，max_tokens 是「输出」上限。max_tokens 设太大可能超窗口报错。

---

> 下一章「03-提示工程实战与工程化」：怎么写好 Prompt、怎么工程化管理 Prompt、怎么防注入。
