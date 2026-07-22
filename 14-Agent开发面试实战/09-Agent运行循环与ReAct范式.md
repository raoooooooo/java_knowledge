# 09 - Agent 运行循环与 ReAct 范式

> 本章是 Agent 的灵魂：Agent 为什么能「自主」跑起来。04 章的 Function Calling 是零件，本章把零件装成一台能循环运转的机器。理解 Agent Loop，Agent 原理就通了，面试「Agent 怎么跑的」必考。

---

## 一、Agent Loop 是什么

### 1.1 核心定义

所有 Agent 框架（LangChain/LangChain4j/Spring AI）的内核都是同一个循环：

```
while 未完成 且 未超轮数:
    1. 组装上下文（系统提示 + 记忆 + 历史 + 工具说明 + 当前状态）
    2. LLM 推理，输出：思考 / 调用哪个工具 / 最终答案
    3. 若调用工具 -> 执行工具 -> 把结果作为 Observation 放回上下文 -> 回到 1
    4. 若给出最终答案 -> 结束
```

- 这是 04 章 Function Calling 循环的泛化：工具调用是循环的引擎。
- **模型决定下一步干什么**：这是「控制权交给 LLM」的本质，也是 Agent 非确定性的来源。

### 1.2 和 Function Calling 循环的关系

- 04 章的「模型要调工具->执行->回灌->继续」就是 Agent Loop。
- 区别：Function Calling 聚焦「单次调用循环」，Agent Loop 在其上加了**记忆、规划、终止判断、多步**等完整能力。
- 用 Spring AI 的 `tools()` 其实框架已自动跑了 Agent Loop，你不用手写。但面试要能手写出来讲清原理。

> 💡 **面试金句**：Agent Loop = 组装上下文 -> 调模型 -> 若调工具则执行回灌继续循环 -> 若给最终答案则结束。工具调用是循环引擎，模型决定下一步是 Agent 自主性的来源。

---

## 二、ReAct 范式 ★ 必考

### 2.1 ReAct（Reasoning + Acting）

- **思想**：让模型交替输出 `Thought（思考）` -> `Action（调用工具）` -> `Observation（观察结果）` -> `Thought...` 直到得出答案。
- **意义**：把「推理」和「行动」耦合，模型不再空想，而是边想边查边算。几乎所有 Agent 的鼻祖范式。

```
Thought: 我需要查今天的天气
Action: search_weather("上海")
Observation: 晴，35℃
Thought: 用户问穿什么，基于晴天高温建议夏装
Final Answer: 建议穿短袖
```

### 2.2 ReAct 的循环本质

- ReAct 就是 Agent Loop 的一种具体范式：Thought 对应 LLM 推理，Action 对应工具调用，Observation 对应回灌结果。
- 现代 Function Calling 把 Action/Observation 结构化了（不再靠正则解析文本），但 Thought-Action-Observation 的循环思想不变。

### 2.3 ReAct 的局限

- **一步一停**：每步都要调一次模型，慢、贵。
- **线性推理**：一条路走到黑，错了无法回退。
- 复杂任务容易在中途卡死或跑偏。

---

## 三、Plan-and-Execute 与 Reflexion ★

### 3.1 Plan-and-Execute（先规划后执行）

- **思想**：先把任务拆成子任务列表（Plan），再逐个执行（Execute），执行中可重规划。
- **优势**：比 ReAct 一步一停更高效（减少中途卡死），适合复杂长任务。
- **劣势**：规划可能不准，遇意外要重规划。

```
Plan: 1.查订单 2.查物流 3.生成回复
Execute step1 -> 结果
Execute step2 -> 结果
（发现物流异常，重规划）-> Plan 更新
Execute step3 -> 最终回复
```

### 3.2 Reflexion（自我反思）

- **思想**：执行后让模型复盘失败，把「反思」写进记忆，下次重试参考。
- **循环**：Actor 执行 -> Evaluator 评判 -> Self-Reflector 反思生成经验 -> 下轮带着经验再试。
- **意义**：让 Agent「从错误中学习」。

### 3.3 三范式对比

| 范式 | 核心 | 是否调工具 | 是否反思 | 适用 |
|------|------|-----------|---------|------|
| ReAct | 边想边做 | 是 | 否 | 通用工具调用 |
| Plan-Execute | 先拆解后执行 | 是 | 可重规划 | 复杂长任务 |
| Reflexion | 做错复盘再做 | 是 | 是 | 自我改进 |

> 💡 **面试金句**：ReAct 是「边想边做」，Plan-Execute 是「先想好再做」，Reflexion 是「做错了复盘再做」。常组合：Plan-Execute 规划执行，Reflexion 在失败时反思修正。

---

## 四、手写最简 Agent Loop（Java）★ 面试加分

> 能手写这段，面试官就知道你真懂 Agent。

```java
public class SimpleAgentLoop {
    private final ChatModel model;
    private final List<Tool> tools;
    private final int maxSteps;          // 防死循环

    public String run(String userQuestion) {
        List<Message> messages = new ArrayList<>();
        messages.add(SystemMessage.of("你是助手，可用工具回答问题"));
        messages.add(UserMessage.of(userQuestion));

        for (int step = 0; step < maxSteps; step++) {
            // 1. 调模型（带工具）
            ChatResponse resp = model.call(messages, tools);
            AssistantMessage am = resp.assistantMessage();

            // 2. 判断是否调工具
            if (am.hasToolCalls()) {
                messages.add(am);                    // 记录模型决策
                for (ToolCall tc : am.toolCalls()) {
                    try {
                        String result = execute(tc.name(), tc.args());
                        messages.add(ToolMessage.of(tc.id(), result));  // 回灌
                    } catch (Exception e) {
                        // 工具失败返回可读错误，让模型修正重试（而非崩）
                        messages.add(ToolMessage.of(tc.id(),
                            "{\"error\":\"工具执行失败:" + e.getMessage() + "，请调整参数重试\"}"));
                    }
                }
                continue;  // 继续循环
            } else {
                // 3. 无工具调用 = 最终答案
                return am.text();
            }
        }
        // 4. 超过最大步数兜底
        return "抱歉，处理该问题需要过多步骤，请细化问题或稍后重试。";
    }
}
```

### 关键点（面试要讲）

1. **工具调用是循环引擎**：有 tool_calls 就继续，没有就结束。
2. **结果回灌**：工具结果作为 ToolMessage 放回上下文给模型继续推理。
3. **错误不崩**：工具异常 catch 住，返回可读错误让模型修正重试（Agent 自我修正的基础）。
4. **防死循环**：maxSteps 兜底，防止模型无限调工具。
5. **上下文累积**：每步的消息都保留，模型能看到完整历史。

---

## 五、循环终止与防死循环 ★ 生产关键

### 5.1 终止条件

| 条件 | 说明 |
|------|------|
| 模型输出最终答案（无工具调用） | 正常终止 |
| 达到最大步数 | 兜底防死循环 |
| 超时 | 单次/总时长超时 |
| 重复检测 | 连续调同一工具同一参数 = 卡住 |
| 用户中断 | 人在环取消 |

### 5.2 死循环的常见形态

1. **反复调同一工具**：模型认为信息不够，反复查同一个。
2. **工具失败不修正**：工具一直报错，模型一直重试不调参。
3. **陷入思考不行动**：模型一直输出 Thought 不调工具也不给答案。
4. **工具结果误解**：模型误解结果，反复确认。

### 5.3 防护措施

```java
// 1. 最大步数兜底（最基础，必须有）
int maxSteps = 10;

// 2. 重复检测：记录近期调用，重复则终止或提示模型换策略
Set<String> recentCalls = new HashSet<>();  // toolName + argsHash

// 3. 超时：整个循环设总超时
long deadline = System.currentTimeMillis() + 60_000;

// 4. 工具失败重试限制：同一工具失败超 N 次放弃
Map<String, Integer> failCount = new HashMap<>();
```

> ⚠️ **maxSteps 是底线**：没有步数限制的 Agent 一定会死循环烧钱。生产必须有，且要监控平均步数（步数飙升 = 质量下降信号）。

---

## 六、错误处理与重试

### 6.1 分层错误处理

| 层 | 错误 | 处理 |
|----|------|------|
| 工具层 | 工具执行失败（参数错/依赖挂） | catch 返回可读错误，让模型修正重试 |
| 模型层 | API 失败（限流/超时/5xx） | 指数退避重试，超限降级 |
| 循环层 | 死循环/超步数 | 兜底终止，返回友好提示 |
| 格式层 | 模型输出格式错（解析失败） | 带「格式错请按schema」的修正 prompt 重试 |

### 6.2 让模型自我修正（Agent 特有）

- 传统重试：原样重试。
- Agent 重试：把错误信息喂回模型，让它**据错误修正**。
- 例：工具返回「订单号不存在，格式应为 ORD 开头」-> 模型下次会调整订单号格式。

> 💡 **面试金句**：Agent 错误处理和传统不同--不是简单重试，而是把错误信息回灌让模型自我修正。工具失败返回可读错误（不是 null/异常），模型能据此调参重试，这是 Agent 自我修正的基础。

---

## 七、生产踩坑

1. **没设最大步数**：死循环烧钱。必须有 maxSteps 兜底。
2. **工具异常直接崩**：异常冒泡打断循环。catch 住返回可读错误让模型修正。
3. **没监控步数**：步数飙升是质量下降信号，要监控平均步数。
4. **上下文无限增长**：每步消息全留，长任务上下文爆。要压缩/滑动窗口（见 10 章）。
5. **重复调用不检测**：模型卡住反复调同一工具。加重复检测。
6. **超时设置不当**：LLM 慢 + 多步循环，总时长可能几分钟。设总超时 + 告警。
7. **错误信息不可读**：返回 stacktrace 模型看不懂无法修正。返回结构化可读错误。

---

## 八、常见面试题

1. **Agent 的运行循环是怎样的？**
   while 未完成：组装上下文 -> 调模型 -> 若调工具则执行后把结果作为 Observation 回灌继续循环 -> 若给最终答案则结束。工具调用是循环引擎，模型决定下一步是 Agent 自主性来源。设最大步数防死循环。

2. **ReAct 是什么？**
   Reasoning + Acting，让模型交替输出 Thought-Action-Observation 循环，边推理边调工具。是几乎所有 Agent 的鼻祖范式。局限是一步一停慢、线性推理不能回退。

3. **ReAct、Plan-Execute、Reflexion 区别？**
   ReAct 边想边做；Plan-Execute 先规划再执行（适合复杂长任务，减少中途卡死）；Reflexion 做错复盘再做（自我改进）。常组合：Plan-Execute 规划执行，Reflexion 失败时反思修正。

4. **Agent 怎么防死循环？**
   多重：最大步数兜底（必须有）、超时、重复调用检测（连续调同工具同参数=卡住）、工具失败重试次数限制。监控平均步数，飙升=质量下降信号。

5. **工具失败了 Agent 怎么处理？**
   不是简单重试，是把错误信息回灌让模型自我修正。catch 异常返回可读错误（如「订单号不存在，格式应为 ORD 开头」），模型据此调参重试。返回 null/异常崩循环是错的。

6. **Agent 循环什么时候终止？**
   模型输出最终答案（无工具调用）/ 达到最大步数 / 超时 / 重复检测命中 / 用户中断。正常是模型判断「信息够了」给最终答案。

7. **手写一个最简 Agent Loop？**
   （讲思路）循环里：调模型带工具 -> 有 tool_calls 则执行后结果作为 ToolMessage 回灌 continue -> 无 tool_calls 则返回文本。外层 maxSteps 兜底，工具异常 catch 返回可读错误。Spring AI 的 tools() 已自动实现这套循环。

8. **为什么说工具调用是 Agent 循环的引擎？**
   只要有工具调用，循环就继续（模型还想查/算）；模型给出纯文本最终答案且无工具调用，循环结束。所以工具调用驱动循环，模型决定调不调决定循环走多久。

---

## 九、资料勘误与重点提醒

1. **「Agent = 调用 LLM」是误解**：单次调 LLM 不是 Agent，Agent 必须有循环（多步、工具、自主决策）。资料偶尔把「调一次大模型」叫 Agent，不准确。
2. **「ReAct 已过时」不严谨**：资料可能暗示 ReAct 被淘汰。实际 Function Calling 是 ReAct 的结构化升级（Thought-Action-Observation 思想不变），ReAct 仍是 Agent 理论根基。
3. **「循环不需要步数限制」是严重错误**：资料示例常省略 maxSteps。生产没步数限制一定死循环烧钱，必须有且监控。
4. **「工具失败就重试」是传统思维**：Agent 应把错误回灌让模型自我修正，不是原样重试。资料常只讲「重试」不讲「回灌错误让模型改」。
5. **「Plan-Execute 一定比 ReAct 好」不严谨**：Plan-Execute 适合复杂长任务，简单任务 Plan-Execute 反而多一次规划开销。按任务复杂度选。
6. **Reflexion 不是「每次都反思」**：资料常不说代价。Reflexion 每次失败要额外 LLM 调用做反思，贵。通常只在关键任务/失败时触发，不是每次循环都反思。

---

> 下一章「10-上下文工程实战」：Agent Loop 跑起来后，上下文怎么管才不爆、不丢关键信息。
