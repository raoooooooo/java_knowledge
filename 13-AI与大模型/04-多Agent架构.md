# 多 Agent 架构

> 单 Agent 在复杂任务上力不从心（上下文爆炸、角色混杂、单点规划瓶颈）。多 Agent 架构把任务拆给多个专职 Agent 协作。本章讲四种主流架构、通信机制与典型案例。

---

## 一、核心概念

### 1.1 为什么需要多 Agent

| 单 Agent 瓶颈 | 多 Agent 如何缓解 |
|---------------|------------------|
| 一个 Agent 承担太多角色，提示词臃肿、易跑偏 | 专职分工，每个 Agent 提示词聚焦 |
| 上下文随任务增长爆炸 | 各 Agent 独立上下文，只回传结论 |
| 单一规划视角，易出错 | 多视角/多专家协作、辩论、互查 |
| 单点失败 | 可并行、可替换 |

### 1.2 多 Agent 系统三要素

1. **架构（Topology）**：Agent 之间怎么组织（层级/网络/竞争...）。
2. **通信（Communication）**：Agent 之间怎么传消息（直接调用/共享黑板/消息总线/标准协议 A2A（Agent2Agent）/ACP（Agent Communication Protocol））。
3. **协调（Coordination）**：谁指挥、谁先动、怎么聚合（监督者/规则/投票）。

---

## 二、四种主流多 Agent 架构 ★

> 用户明确要求「四种多 Agent 架构」。下面是业界公认的四类典型拓扑。

### 2.1 架构一：层级 / 监督者架构（Hierarchical / Supervisor）

```
            [Supervisor 监督者 Agent]
           /        |         \
     [Agent A]  [Agent B]   [Agent C]
     (搜索)     (分析)      (写作)
```

- **结构**：一个 Supervisor 负责接收任务、拆解、分派给子 Agent、收集结果、决定下一步或汇总。
- **协调方式**：中心化，Supervisor 全权调度。
- **优点**：控制清晰、易实现、可追踪、责任明确。
- **缺点**：Supervisor 是瓶颈和单点，子 Agent 间无直接沟通。
- **典型**：LangGraph 的 Supervisor 模式、大多数「编排型」Agent 平台。
- **适用**：任务可清晰拆解、子任务相对独立的场景（如「研究-分析-写作」流水线）。

### 2.2 架构二：网络 / 协作架构（Network / Collaborative）

```
   [Agent A] <----> [Agent B]
      ^   \         /   ^
      |    v       v    |
   [Agent C] <--> [Agent D]
```

- **结构**：Agent 之间点对点自由通信，无固定中心，谁需要谁就找谁对话。
- **协调方式**：去中心化，靠共享状态/消息传递协调。
- **优点**：灵活、可涌现复杂协作、无单点。
- **缺点**：易失控（死循环/无限对话）、成本高、难调试、难追踪。
- **典型**：AutoGen 的 GroupChat、CrewAI 的角色协作、MetaGPT 的「软件公司」模拟。
- **适用**：需要多轮讨论/协商、创意发散的场景。

### 2.3 架构三：竞争 / 辩论架构（Debate / Competitive）

```
   [Agent A 立场1] --\
                     --> [Judge 裁判 Agent] --> 最优答案
   [Agent B 立场2] --/
```

- **结构**：多个 Agent 对同一问题给出不同方案/观点，互相辩论/批判，最后由裁判 Agent（或投票）选出最优。
- **协调方式**：对抗+评审，靠「竞争」逼近更优解。
- **优点**：能减少单 Agent 偏见和幻觉，提升答案质量和可靠性（多视角交叉验证）。
- **缺点**：调用次数多、慢、贵。
- **典型**：Multi-Agent Debate 论文范式、 Society of Minds、Critic-Actor 模式。
- **适用**：高可靠要求的推理/决策（如代码评审、安全审查、复杂判断）。

### 2.4 架构四：编排 / 流水线架构（Orchestrator-Worker / Pipeline / Router）

```
   [Orchestrator]
     | 路由分发
     v
  [Worker1] [Worker2] [Worker3]   <- 并行同构 worker
     \        |        /
      v       v       v
   [结果汇总/Aggregator]
```

- **结构**：编排者把任务分发给一组（通常同构）Worker 并行处理，再聚合结果。包含两种子模式：
  - **Pipeline（流水线）**：串行，A 的输出是 B 的输入（研究->起草->润色）。
  - **Router / Map-Reduce**：并行同构，分而治之（如把长文档切片分给多个 Worker 总结再合并）。
- **协调方式**：编排者驱动，确定性较高（接近 Workflow）。
- **优点**：可并行、可扩展、易控成本、结果可预测。
- **缺点**：灵活性低于自由协作，适合结构化任务。
- **典型**：Anthropic Workflow 的「并行/聚合」范式、Map-Reduce 式文档处理。
- **适用**：大批量同质任务、可并行拆分的工作。

### 2.5 四种架构对比速记

| 架构 | 拓扑 | 协调 | 并行性 | 控制度 | 典型场景 |
|------|------|------|--------|--------|---------|
| 层级/监督者 | 树（有中心） | 中心调度 | 中 | 高 | 可拆解的流水任务 |
| 网络/协作 | 图（去中心） | 消息传递 | 高 | 低 | 多轮讨论/协商 |
| 竞争/辩论 | 对抗+裁判 | 投票/裁决 | 中 | 中 | 高可靠推理/审查 |
| 编排/流水线 | 管道或星型 | 编排者驱动 | 高（可并行） | 高 | 大批量同质任务 |

> ⚠️ 面试注意：这四类不是非此即彼，真实系统常**混合**。比如外层 Supervisor + 内部某个节点用 Debate；或 Pipeline 某环节用 Network 协作。

---

## 三、Agent 间通信机制

| 机制 | 说明 | 典型 |
|------|------|------|
| **直接调用** | 一个 Agent 直接把消息作为输入调另一个 | 函数式编排 |
| **共享状态/黑板** | 多个 Agent 读写同一个共享状态对象 | LangGraph 的共享 State |
| **消息总线/群聊** | Agent 往共享频道发消息，其他人按需响应 | AutoGen GroupChat |
| **标准协议** | 跨进程/跨厂商的 Agent 间通信协议 | **A2A、ACP**（见第6章） |

- **共享 State** 是 LangGraph 的核心抽象：图节点（Agent）读写同一个 State 对象，靠 State 传递信息，天然支持检查点/回放/人在环。
- **A2A/ACP 协议** 解决的是「不同框架/不同厂商的 Agent 怎么互通」--类比微服务的 RPC（Remote Procedure Call，远程过程调用）标准。

---

## 四、典型案例

### 4.1 MetaGPT（软件公司模拟）
模拟一个软件公司：Product Manager -> Architect -> Engineer -> QA，各司其职，按 SOP（Standard Operating Procedure，标准作业流程）协作产出软件。**层级+流水线**架构的代表。

### 4.2 AutoGen（微软）
对话式多 Agent 框架。核心是 `GroupChat`：多个 Agent 在一个群聊里按规则轮流发言，由一个 `GroupChatManager` 决定下一个发言者。**网络/协作**架构代表。

### 4.3 CrewAI
角色驱动：定义多个有 Role/Goal/Backstory 的 Agent，组成 Crew，按 Process（sequential/hierarchical）执行任务。偏**协作+层级**。

### 4.4 AgentScope（阿里）
面向多 Agent 应用开发的开源框架，提供消息通信、Agent 抽象、可视化，支持分布式部署。详见「05-Agent开发框架」。

### 4.5 LangGraph 的多 Agent 模式
LangGraph 官方总结三种多 Agent 模式：**Supervisor（监督者）/ Hierarchical（层级）/ Network（网络）**，本质就是上面架构一、二及它们的组合。

---

## 五、设计取舍与陷阱

- **不是越多 Agent 越好**：每个 Agent 都有 prompt/上下文/调用成本，过多反而难协调、慢、贵。简单任务用单 Agent + 工具往往更优。
- **上下文隔离 vs 信息共享**：隔离防污染但可能丢全局信息；共享全但易爆炸。折中：共享精简 State，子 Agent 内部细节隔离。
- **防死循环**：网络架构必须设最大轮数/收敛条件，否则 Agent 可能无限对话。
- **可观测**：多 Agent 调用链路深，必须有 Trace（见第8章），否则出 bug 无法定位是哪个 Agent 的问题。
- **成本控制**：辩论/网络架构 token 消耗是单 Agent 的数倍，需评估 ROI（Return on Investment，投资回报率）。

---

## 六、常见面试题

1. **什么时候该用多 Agent，什么时候单 Agent 就够？**
   任务可清晰拆解、需多专家视角、单 Agent 上下文/角色过载 -> 多 Agent。简单、线性、低成本优先 -> 单 Agent + 工具。

2. **四种多 Agent 架构分别是？各自适用场景？**
   层级（可拆解流水任务）、网络协作（多轮讨论协商）、竞争辩论（高可靠推理审查）、编排流水线（大批量同质并行）。详见正文对比表。

3. **多 Agent 怎么通信？**
   直接调用、共享状态/黑板、消息总线/群聊、标准协议（A2A/ACP）。LangGraph 用共享 State，AutoGen 用 GroupChat。

4. **Supervisor 架构的瓶颈是什么？怎么缓解？**
   Supervisor 是单点和规划瓶颈。可分层级（多级 Supervisor）、子 Agent 间直接通信减少中转、限制层级深度。

5. **多 Agent 辩论架构为什么能提升答案质量？**
   多视角交叉验证能发现单 Agent 的偏见和错误，竞争逼近更优解，裁判聚合降低幻觉。

6. **多 Agent 系统怎么防止失控和成本爆炸？**
   设最大轮数/超时、收敛条件、共享精简状态、Trace 可观测、限制 Agent 数量、对高消耗架构评估 ROI。

---

> 本章与「05-Agent开发框架」的 LangGraph/AutoGen/CrewAI/AgentScope 强相关，可对照阅读。
