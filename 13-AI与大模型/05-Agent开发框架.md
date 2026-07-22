# Agent 开发框架

> 本章对比主流 Agent 开发框架：LangChain 全家桶（LangChain/LangGraph/LangSmith/LangServe）、AgentScope、AutoGen、CrewAI，以及 Anthropic 的 Skill 机制。选型是面试高频。

---

## 一、LangChain 生态全景

LangChain Inc.（现 LangChain 公司）围绕 LLM 应用形成一套「全家桶」，分工明确：

| 组件 | 定位 | 阶段 |
|------|------|------|
| **LangChain** | 通用 LLM 应用开发框架 | 开发 |
| **LangGraph** | 基于图的有状态 Agent 编排 | 开发（生产级 Agent） |
| **LangSmith** | 观测/追踪/评测/Prompt 管理 | 运维（DevOps，开发运维一体化） |
| **LangServe / LangGraph Platform** | 把应用部署为 API / 托管运行时 | 部署 |

> 用户列表中的 `LANGSUIT` 最可能指 **LangSmith**（也可能含 LangServe）。本章一并覆盖，请确认。

### 1.1 LangChain

- **定位**：最早最全的 LLM 应用框架，把「LLM + 提示 + 记忆 + 工具 + 检索 + 链」抽象成可组合组件。
- **核心概念**：
  - **Chain（链）**：把多个步骤串起来（如 `prompt | llm | output_parser`）。
  - **LCEL（LangChain Expression Language，LangChain 表达式语言）**：用管道符 `|` 声明式编排，支持流式、批处理、异步、回溯（trace）。
  - **Agent / AgentExecutor**：早期用 `AgentExecutor` 跑 ReAct 循环。⚠️ 官方已**不推荐 AgentExecutor**，转而用 LangGraph 构建可控 Agent。
  - **Memory**：对话记忆抽象（已弱化，记忆迁移到 LangGraph State）。
  - **Retriever**：RAG 检索抽象。
- **评价**：生态最全、上手快，但抽象层多、较重、历史上 API 变动频繁。适合快速原型和 RAG。

### 1.2 LangGraph ★（当前官方主推）

- **定位**：把 Agent 当作**状态机/图**来编排，强调可控、可观测、可持久化，是生产级 Agent 的首选。
- **核心抽象**：
  - **State（状态）**：一个共享的有序字典/对象，贯穿整个图，节点读写它。这是 LangGraph 的灵魂。
  - **Node（节点）**：一个函数/Agent，接收 State、返回 State 更新。
  - **Edge（边）**：节点间的转移，可以是固定边或**条件边**（由一个函数根据 State 决定下一个节点，实现路由/循环）。
  - **Checkpointer**：每一步状态可持久化（如存到 SQLite/Postgres），支持**中断恢复、人在环（Human-in-the-loop）、回放**。
- **为什么取代 AgentExecutor**：AgentExecutor 是「黑盒 ReAct 循环」，难控制、难调试、难加人审；LangGraph 把流程显式化成图，可精确控制循环、分支、并发、暂停。
- **多 Agent**：官方总结 Supervisor / Hierarchical / Network 三种模式（见「04-多Agent架构」），每个 Agent 是图中的一个节点。

```python
# LangGraph 形象示例（伪代码）
graph = StateGraph(State)
graph.add_node("researcher", research_agent)
graph.add_node("writer", write_agent)
graph.add_conditional_edges("researcher", lambda s: "writer" if s.ready else "researcher")
graph.set_entry_point("researcher")
app = graph.compile(checkpointer=MemorySaver())  # 支持中断恢复
```

### 1.3 LangSmith（推测为 LANGSUIT）

- **定位**：LLM 应用的**可观测 + 评测 + Prompt 管理**平台，类比 LLM 时代的 SkyWalking + JUnit。
- **能力**：
  - **Trace 追踪**：把一次 Agent 调用展开成树（LLM 调用、工具调用、检索），每步的输入/输出/耗时/token 成本全记录。
  - **Eval 评测**：定义数据集，自动跑评测，支持 LLM-as-a-Judge（用 LLM 当裁判打分）自动打分，对比不同 Prompt/模型版本。
  - **Prompt Hub/管理**：版本化管理 Prompt，A/B 实验。
  - **Playground**：在线调试 Prompt/链。
- **意义**：LLM 应用「非确定性」，传统监控不够，必须靠全链路 Trace + 回放评测才能稳定迭代。LangSmith 是 LangChain 生态的商业化核心。
- **同类**：Langfuse（开源）、Arize Phoenix、Helicone。

### 1.4 LangServe / LangGraph Platform

- **LangServe**：把 LangChain Runnable 一键部署为 FastAPI REST API（已较旧）。
- **LangGraph Platform / Deployment**：托管运行 LangGraph 应用，提供持久化、流式、Cron、人在环等运行时能力。是 LangChain 公司的部署商业化方向。

---

## 二、AgentScope（阿里）

- **定位**：阿里通义实验室开源的**全链路多 Agent 框架**，强调分布式、高可扩展、对开发者友好。
- **核心特性**：
  - **Message 通信**：Agent 间基于消息通信，支持同步/异步、点对点/广播。
  - **Agent 抽象**：`ReActAgent`、`UserAgent` 等，可自定义。
  - **Pipeline / Hub**：提供现成的消息流转模式（如顺序、广播、请求-响应）和工具/模型 Hub。
  - **分布式部署**：原生支持把 Agent 分布到多机，适合大规模多 Agent。
  - **可视化**：Studio 可视化编排与调试。
- **对比 LangChain**：AgentScope 更聚焦「多 Agent 协作」和「分布式」，对多 Agent 场景的抽象更直接；LangChain 生态更大但偏通用。

---

## 三、AutoGen（微软）

- **定位**：**对话式多 Agent** 框架，核心是让多个 Agent 通过「对话」协作完成任务。
- **核心抽象**：
  - **ConversableAgent**：可对话的 Agent，能接收/回复消息、调工具。
  - **GroupChat + GroupChatManager**：群聊，多个 Agent 在一个对话里按规则轮流发言，Manager 决定下一个发言者。
  - **UserProxyAgent**：代表人类的 Agent，可执行代码、提供输入，实现人在环。
- **强项**：多 Agent 对话协作、代码执行（写完代码自动跑）。
- **版本**：AutoGen 0.4+ 进行了大重构（Async、事件驱动、可扩展），向生产级演进。
- **对比**：AutoGen 偏「对话驱动多 Agent」，LangGraph 偏「图状态机」，CrewAI 偏「角色」。

---

## 四、CrewAI

- **定位**：**角色驱动**的多 Agent 框架，像组建一个「团队/剧组（Crew）」。
- **核心抽象**：
  - **Agent**：有 Role（角色）、Goal（目标）、Backstory（背景故事），让 Agent 更「人设化」。
  - **Task**：具体任务，可指定给某 Agent，有期望输出。
  - **Crew**：把多个 Agent + Task 组成团队，指定 Process（`sequential` 顺序 / `hierarchical` 层级）。
  - **Tools**：Agent 可用工具。
- **强项**：上手极简、角色人设让协作更自然，适合内容/研究类多 Agent。
- **对比**：比 LangGraph 简单但可控性弱；比 AutoGen 更结构化（角色而非纯对话）。

---

## 五、Skill 机制（Anthropic）★

> 用户列表中的 `SKILL`。这是 2025 年 Anthropic 提出的重要概念，Claude Code 等 Agent 已内置。

### 5.1 什么是 Skill

- **定义**：一种**可复用、按需加载的能力包**。把某一领域的能力（指令 + 工作流 + 脚本 + 资源文件）打包，Agent 在需要时才加载进上下文，用完即释放。
- **类比**：函数/插件 vs Skill 的区别--Skill 是「带上下文和资源的工作流包」，不只是函数调用。

### 5.2 为什么需要 Skill（渐进式披露 / Progressive Disclosure）

- **痛点**：把所有工具/知识一次性塞进上下文 -> 上下文爆炸 + 注意力稀释（模型选错工具）。
- **Skill 思路**：平时只给 Agent 一个「Skill 索引」（名字 + 一句话描述），Agent 判断需要时才把对应 Skill 的完整指令和资源加载进来。
- **好处**：
  - 省 token（按需而非全量）。
  - 提准确率（减少干扰）。
  - 可扩展（加 Skill 不改主提示）。
  - 可复用/可分享（Skill 可打包分发）。

### 5.3 Skill 与工具/记忆的关系
- **工具（Tool）**：执行动作的函数。Skill 可以**包含**工具调用，但 Skill 更上层--它是一套「能力+流程+资源」。
- **记忆（Memory）**：存事实/经验。Skill 是「程序性记忆」（怎么做），记忆库是「陈述性记忆」（是什么）。

---

## 六、框架选型对比矩阵

| 框架 | 出身 | 强项 | 适合场景 | 多 Agent 模式 |
|------|------|------|---------|--------------|
| **LangChain** | LangChain 公司 | 生态全、组件多、RAG | 快速原型/通用 LLM 应用 | 弱（已转 LangGraph） |
| **LangGraph** | LangChain 公司 | 状态机图、可控可观测、持久化 | 生产级 Agent、复杂流程 | Supervisor/层级/网络 |
| **LangSmith** | LangChain 公司 | Trace+Eval+Prompt 管理 | LLM 应用可观测与评测 | - |
| **AutoGen** | 微软 | 对话式多 Agent、代码执行 | 多 Agent 讨论、代码任务 | GroupChat 群聊 |
| **CrewAI** | 社区/公司 | 角色驱动、上手快 | 内容/研究团队协作 | 顺序/层级 |
| **AgentScope** | 阿里 | 分布式多 Agent、可视化 | 大规模分布式多 Agent | 消息通信 |
| **Dify/Coze** | 平台 | 可视化拖拽、低代码 | 业务团队快速搭 Agent | 平台内置 |
| **Skill** | Anthropic | 按需加载能力包 | 给 Agent 扩展领域能力 | - |

---

## 七、常见面试题

1. **LangChain 和 LangGraph 什么关系？为什么现在主推 LangGraph？**
   LangChain 是通用框架，LangGraph 是其生态中专门做「有状态 Agent 编排」的图框架。LangGraph 取代旧的 AgentExecutor，因为后者是黑盒 ReAct 循环难控，LangGraph 用显式图+共享 State 支持循环/分支/并发/持久化/人在环，更适合生产。

2. **LangGraph 的 State 有什么用？**
   State 是贯穿图的共享状态，节点读写它来传递信息。基于 State 可做 Checkpoint 持久化，从而支持中断恢复、人在环、回放调试。

3. **LangSmith 解决什么问题？**
   LLM 应用非确定性，需要全链路 Trace（看每步输入输出/耗时/成本）+ 自动评测（数据集+LLM-as-Judge）+ Prompt 版本管理，才能稳定迭代。LangSmith 就是这个可观测+评测平台。

4. **AutoGen 和 CrewAI 的区别？**
   AutoGen 以「对话」为核心，多个 Agent 在群聊里协作，适合开放式讨论和代码任务；CrewAI 以「角色」为核心，结构化定义 Role/Goal/Task，适合内容/研究类流水协作。

5. **什么是 Skill？为什么需要它？**
   Skill 是按需加载的能力包（指令+流程+脚本+资源）。通过「渐进式披露」只在需要时加载，避免一次性塞满上下文导致爆炸和注意力稀释，同时可扩展可复用。

6. **选型：要做一个生产级、需人在环和断点恢复的复杂 Agent，选哪个？**
   LangGraph。它的显式图 + State + Checkpointer 原生支持人在环、中断恢复、回放，可控可观测，是生产首选。

7. **多 Agent 框架怎么选？**
   要分布式大规模 -> AgentScope；要对话协作 -> AutoGen；要角色流水 -> CrewAI；要可控生产级图编排 -> LangGraph。简单任务其实单 Agent + 工具更优。

---

> 本章框架与「06-协议生态」的 MCP（工具标准）、「07-ClaudeCode」的 Skill 机制强相关。
