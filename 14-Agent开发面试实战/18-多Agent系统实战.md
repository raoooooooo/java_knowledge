# 18 - 多 Agent 系统实战

> 单 Agent 在复杂任务上力不从心（上下文爆炸、角色混杂）。多 Agent 架构把任务拆给多个专职 Agent 协作。13 章讲了四种架构原理，本章讲 Java 工程实现、通信、成本控制。第四篇收尾。

---

## 一、什么时候用多 Agent ★

### 1.1 单 Agent 的瓶颈

| 单 Agent 瓶颈 | 多 Agent 如何缓解 |
|---------------|------------------|
| 一个 Agent 承担太多角色，Prompt 臃肿易跑偏 | 专职分工，每个 Agent Prompt 聚焦 |
| 上下文随任务增长爆炸 | 各 Agent 独立上下文，只回传结论 |
| 单一规划视角易出错 | 多视角/多专家协作、辩论、互查 |
| 单点失败 | 可并行、可替换 |

### 1.2 但不是越多越好

- 每个 Agent 有 Prompt/上下文/调用成本，过多反而难协调、慢、贵。
- 简单任务用单 Agent + 工具往往更优。**别为多而多**。

> 💡 **面试金句**：多 Agent 解决单 Agent 的上下文爆炸/角色混杂/单点规划瓶颈，但不是越多越好--每个 Agent 有成本，过多难协调慢贵。任务可清晰拆解、需多专家视角、单 Agent 过载才用多 Agent，简单任务单 Agent + 工具更优。

---

## 二、四种多 Agent 架构工程实现 ★

### 2.1 层级/监督者（Hierarchical/Supervisor）

- 一个 Supervisor Agent 接收任务、拆解、分派子 Agent、收集结果。

```java
// Supervisor Agent 负责分发
class SupervisorAgent {
    List<Agent> subAgents;  // 搜索Agent/分析Agent/写作Agent

    String run(String task) {
        List<String> subtasks = plan(task);      // 拆解
        Map<String, String> results = new HashMap<>();
        for (String sub : subtasks) {
            Agent agent = route(sub);             // 分派对应Agent
            results.put(sub, agent.run(sub));     // 子Agent独立上下文
        }
        return aggregate(results);                // 汇总
    }
}
```

- Java 实现：Supervisor 用 Function Calling 调子 Agent（子 Agent 当工具）。
- 适合：任务可清晰拆解、子任务独立（研究-分析-写作流水线）。

### 2.2 网络/协作（Network/Collaborative）

- Agent 间点对点自由通信，无固定中心。
- Java 实现：用共享 State 或消息总线，Agent 互相调用。
- 适合：多轮讨论/协商、创意发散。坑：易失控（死循环）、难调试。

### 2.3 竞争/辩论（Debate/Competitive）

- 多 Agent 对同一问题给不同方案，互相批判，裁判选最优。

```java
// 两个 Agent 给方案，裁判 Agent 评审
String planA = agentA.propose(problem);
String planB = agentB.propose(problem);
String best = judgeAgent.evaluate(problem, planA, planB);
```

- 适合：高可靠推理/决策（代码评审、安全审查）。坑：调用多、慢、贵。

### 2.4 编排/流水线（Orchestrator-Worker/Pipeline）

- 编排者分发任务给一组（通常同构）Worker 并行，再聚合。
- Pipeline：串行（A 输出是 B 输入）。Map-Reduce：并行同构后合并。

```java
// Map-Reduce：长文档切片分给多 Worker 总结再合并
List<String> chunks = split(doc);
List<String> summaries = chunks.parallelStream()
    .map(chunk -> workerAgent.run("总结：" + chunk))
    .toList();
String finalSummary = mergerAgent.run("合并摘要：" + summaries);
```

- 适合：大批量同质任务、可并行拆分。可控性高、易控成本。

### 2.5 对比速记

| 架构 | 拓扑 | 协调 | 控制度 | 典型 |
|------|------|------|--------|------|
| 层级 | 树(有中心) | 中心调度 | 高 | 可拆解流水任务 |
| 网络 | 图(去中心) | 消息传递 | 低 | 多轮讨论协商 |
| 辩论 | 对抗+裁判 | 投票裁决 | 中 | 高可靠推理 |
| 编排 | 管道/星型 | 编排者驱动 | 高 | 大批量同质并行 |

---

## 三、Agent 间通信机制

| 机制 | 说明 | Java 实现 |
|------|------|----------|
| **直接调用** | Agent A 调 Agent B | 方法调用/Function Calling |
| **共享状态** | 读写同一 State | Graph 的 State（17 章） |
| **消息总线** | 往共享频道发消息 | 消息中间件（RocketMQ）/内存队列 |
| **标准协议** | 跨框架/厂商通信 | A2A / ACP（13 章协议生态） |

- **共享 State** 是 Graph 编排的核心：节点读写同一 State 传递信息，天然支持 Checkpoint。
- **A2A/ACP 协议** 解决跨框架互通（呼应 13 章）。

---

## 四、成本控制与可观测 ★

### 4.1 多 Agent 成本爆炸

- 辩论/网络架构 token 消耗是单 Agent 的数倍。
- 必须评估 ROI，控成本。

### 4.2 成本控制手段

| 手段 | 说明 |
|------|------|
| **架构选型** | 能用编排/层级别用辩论/网络 |
| **限制 Agent 数** | 不是越多越好 |
| **子 Agent 用小模型** | 简单子任务用便宜模型 |
| **缓存** | 相似子任务复用结果 |
| **设预算** | 整体 token 预算上限，超了停 |
| **最大轮数** | 防无限对话 |

### 4.3 可观测（呼应 20 章）

- 多 Agent 调用链路深，**必须有 Trace**，否则出 Bug 无法定位是哪个 Agent。
- 每个 Agent 调用记录输入/输出/耗时/成本，串成调用树。
- 监控：总 token、各 Agent 占比、平均轮数、人工接管率。

---

## 五、Java 多 Agent 框架对比

| 框架 | 多 Agent 模式 | 特点 |
|------|--------------|------|
| **Spring AI Alibaba Graph** | 图编排多 Agent | 国产、Spring 集成、状态图 |
| **LangChain4j** | AiServices 组合 | 灵活、偏线性，复杂要自组合 |
| **自研** | 自定义 | 最可控、要造轮子 |
| **A2A 协议** | 跨框架互通 | 标准化、适合跨厂商 |

- 国内 Spring Boot 项目：Spring AI Alibaba Graph 最顺手。
- 跨框架/跨厂商互通：A2A 协议。

---

## 六、生产踩坑

1. **过度多 Agent**：简单任务上多 Agent，慢贵还可能更差。单 Agent + 工具够就别多。
2. **Supervisor 单点瓶颈**：层级架构 Supervisor 是瓶颈。可分层级、子 Agent 间直接通信减中转。
3. **网络架构死循环**：Agent 无限对话。设最大轮数/收敛条件。
4. **辩论成本爆炸**：多 Agent 多轮 token 数倍消耗。评估 ROI、限制轮数、子 Agent 用小模型。
5. **不可观测**：多 Agent 链路深出 Bug 难定位。必须有 Trace。
6. **共享状态竞争**：并行 Agent 读写共享 State 要线程安全。
7. **子 Agent 结论污染主上下文**：子 Agent 回传太多细节。只回传结论。

---

## 七、常见面试题

1. **什么时候用多 Agent？什么时候单 Agent 够？**
   多 Agent 解决单 Agent 上下文爆炸/角色混杂/单点规划瓶颈。任务可清晰拆解、需多专家视角、单 Agent 过载才用；简单线性任务单 Agent + 工具更优。不是越多越好，每个 Agent 有成本。

2. **四种多 Agent 架构？Java 怎么实现？**
   层级（Supervisor 调度子 Agent）、网络（点对点消息）、辩论（多方案+裁判）、编排（并行 Worker+聚合）。Java 用 Spring AI Alibaba Graph 编排，或自研。Supervisor 用 Function Calling 调子 Agent。

3. **多 Agent 怎么通信？**
   直接调用（方法/Function Calling）、共享状态（Graph 的 State）、消息总线（MQ）、标准协议（A2A/ACP）。Graph 共享 State 最常用，A2A 解决跨框架互通。

4. **多 Agent 怎么控成本？**
   架构选型（能用层级/编排别用辩论/网络）、限制 Agent 数、子 Agent 用小模型、缓存复用、设 token 预算、最大轮数防死循环。辩论/网络 token 是单 Agent 数倍，要评估 ROI。

5. **Supervisor 架构的瓶颈？怎么缓解？**
   Supervisor 是单点和规划瓶颈。可分层级（多级 Supervisor）、子 Agent 间直接通信减中转、限制层级深度。

6. **多 Agent 怎么防失控？**
   设最大轮数/超时、收敛条件、共享精简状态、Trace 可观测、限制 Agent 数、整体 token 预算上限。

7. **辩论架构为什么能提升质量？**
   多视角交叉验证能发现单 Agent 的偏见和错误，竞争逼近更优解，裁判聚合降低幻觉。代价是调用多慢贵，适合高可靠场景。

8. **Java 多 Agent 怎么选框架？**
   Spring Boot 国内项目用 Spring AI Alibaba Graph（图编排多 Agent，Spring 集成）；LangChain4j 偏线性复杂要自组合；跨框架互通用 A2A 协议；最可控可自研。

---

## 八、资料勘误与重点提醒

1. **「多 Agent 一定比单 Agent 强」是误区**：资料常把多 Agent 当更优解。实际有协调/成本/延迟代价，简单任务单 Agent + 工具更优。按任务复杂度选，别为多而多。
2. **「辩论/网络架构通用」不严谨**：资料常把辩论/网络当推荐架构。实际它们 token 消耗是单 Agent 数倍，只适合高可靠/多轮协商场景，要评估 ROI。
3. **「多 Agent 不需要特殊可观测」是坑**：多 Agent 链路深，出 Bug 难定位是哪个 Agent。资料少强调 Trace 必要性。必须有调用树。
4. **「Agent 间随便通信」漏了线程安全**：并行 Agent 读写共享 State 要线程安全，资料示例常忽略并发问题。
5. **「子 Agent 回传越多越好」是坑**：子 Agent 回传海量细节会污染主上下文。只回传结论是隔离原则，资料少强调。
6. **A2A 协议少在 Java 资料出现**：A2A 是跨框架 Agent 互通标准，Java 资料常不提。这是多 Agent 互通的趋势，值得关注。

---

> 第四篇 框架与编排完结。你已懂 Java 两大框架（LangChain4j/Spring AI）、国内生态（Alibaba）、复杂编排（Graph）、多 Agent。下一篇「第五篇 工程化与生产」把你带回主场--这是 Java 工程师的核心优势区。
