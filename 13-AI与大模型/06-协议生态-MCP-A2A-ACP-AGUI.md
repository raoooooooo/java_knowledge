# 协议生态（MCP / A2A / ACP / AG-UI / SSE）

> 2024-2025 年 Agent 领域最关键的变化是**协议标准化**：MCP 让 Agent 标准化接工具/数据、A2A/ACP 让 Agent 间标准化通信、AG-UI 让 Agent 与前端标准化交互。本章是面试重点。

> ⚠️ 这些协议都在快速演进，版本细节请以官方最新 spec 为准。本章描述截至 2026 年中的主流认知。

---

## 一、SSE 原理（底层传输基础）

> 用户明确要求 SSE 原理。MCP、A2A 都用到 SSE，先打基础。

### 1.1 什么是 SSE（Server-Sent Events）

- **定义**：基于 HTTP 的**服务器向客户端单向推送**技术。服务器用一条长连接持续把事件流推给浏览器/客户端。
- **MIME 类型**：`text/event-stream`。
- **消息格式**：
  ```
  data: 第一条消息\n
  \n
  data: 第二条消息\n
  \n
  event: update\n
  data: {"k":"v"}\n
  \n
  ```
  每条消息由若干字段（`data`/`event`/`id`/`retry`）组成，消息间用空行分隔。

### 1.2 关键机制

- **单向**：只有 Server -> Client，Client 要发消息得另发 HTTP 请求。
- **自动重连**：浏览器原生断线自动重连。
- **断点续传**：客户端把最后收到的 `id` 通过 `Last-Event-ID` 请求头发回，服务器据此续传，不丢消息。
- **纯文本**：默认只能传文本（UTF-8），二进制需 Base64。

### 1.3 SSE vs WebSocket vs 轮询

| 特性 | 轮询 | SSE | WebSocket |
|------|------|-----|-----------|
| 方向 | 客户端拉 | 服务器推（单向） | 双向 |
| 协议 | HTTP | HTTP | 独立协议（握手后升级） |
| 复杂度 | 低 | 低 | 中 |
| 自动重连 | 无 | 内置 | 需自己实现 |
| 二进制 | 是 | 否（需编码） | 是 |
| 适合 | 简单低频 | 服务器推送事件流 | 双向实时通信（聊天/游戏） |

> ⚠️ **为什么 AI Agent 爱用 SSE**：LLM 流式生成天然是「服务器持续推 token」的场景，SSE 轻量、走 HTTP、自动重连、对前端友好，是流式输出的事实标准（OpenAI/Anthropic 流式 API 都用 SSE）。

---

## 二、MCP（Model Context Protocol）★

### 2.1 是什么、谁搞的

- **全称**：Model Context Protocol（模型上下文协议）。
- **提出**：Anthropic，2024 年 11 月开源。
- **定位口号**：「AI 应用的 USB-C」--一个标准接口，让任何 LLM/Agent 能接任何工具/数据源，解耦「模型」与「上下文来源」。

### 2.2 解决什么痛点

- **M×N 问题**：M 个 Agent 框架 × N 个工具，要写 M×N 份集成。MCP 把它变成 M+N：Agent 实现 MCP Client，工具实现 MCP Server，即插即用。
- **工具碎片化**：每个框架各自定义工具协议，无法复用。MCP 统一了。

### 2.3 架构：Host / Client / Server

```
┌──────────────────────────────┐
│  Host（宿主应用，如 Claude Code）│
│   ┌─────────┐  ┌─────────┐   │
│   │ Client1 │  │ Client2 │   │   <-- 每个 Server 对应一个 Client
│   └────┬────┘  └────┬────┘   │
└────────┼─────────────┼──────┘
         │ 协议        │ 协议
   ┌─────▼─────┐ ┌─────▼─────┐
   │ MCP Server│ │ MCP Server│   <-- 各自暴露能力
   │(文件系统) │ │(数据库)   │
   └───────────┘ └───────────┘
```

- **Host（宿主）**：运行 Agent 的应用（Claude Desktop、Claude Code、Cursor、IDE 插件）。管理 Client、安全、用户授权。
- **Client（客户端）**：Host 内部，每个 Client 与一个 Server 一对一连接，负责协议通信。
- **Server（服务端）**：独立进程/服务，暴露能力给 LLM。可由任何人开发（官方有大量预置 Server：GitHub、文件系统、数据库、Slack 等）。

### 2.4 三大能力原语（Primitives）

| 原语 | 含义 | 由谁控制 | 典型用途 |
|------|------|---------|---------|
| **Tools（工具）** | 可被 LLM 调用执行的函数 | 模型决定调用 | 查数据库、发请求、调 API |
| **Resources（资源）** | 可读取的数据/文件 | 应用/用户控制读取 | 读文件、查配置、获取上下文 |
| **Prompts（提示模板）** | 预置的提示词模板 | 用户触发 | 标准化任务模板 |

> 后续 spec 还引入：**Sampling**（Server 反向请求 Host 的 LLM 推理）、**Roots**（限定 Server 可访问的文件范围）、**Elicitation**（Server 向用户请求额外输入）。

### 2.5 传输层（Transport）

- **stdio（标准输入输出）**：Server 作为本地子进程，通过 stdin/stdout 通信。**本地、低延迟、简单**，是最常用方式。
- **Streamable HTTP**（2025-03-26 spec 引入，**取代旧的 HTTP+SSE 传输**）：基于 HTTP，可选升级为 SSE 流。支持远程、无状态友好、可经代理/网关、可负载均衡。
  - ⚠️ **演进说明**：早期 MCP 远程传输是「HTTP+SSE」（一条 SSE 长连接收消息 + 一条 HTTP POST 发消息），存在难负载均衡、连接状态重等问题。新 spec 用 **Streamable HTTP** 统一为单端点：客户端 POST 请求，服务器可选择返回普通 JSON 或升级为 SSE 流，更灵活更适合生产。

### 2.6 一次工具调用的流程

1. Host 启动，加载各 MCP Server，Client 与之握手，发现其 Tools/Resources/Prompts。
2. Host 把可用 Tools 的描述喂给 LLM（作为函数定义）。
3. LLM 决定调用某 Tool -> Host 经 Client 把调用请求发给对应 Server。
4. Server 执行工具，返回结果。
5. 结果回填 LLM 上下文，LLM 继续推理。

> 本质：MCP 把「工具发现+调用」标准化，最终模型调用工具仍走 Function Calling。**MCP 是协议层，Function Calling 是模型层**，二者配合不冲突。

### 2.7 MCP vs Function Calling

| 维度 | Function Calling | MCP |
|------|------------------|-----|
| 层次 | 模型层（厂商特性） | 协议层（开放标准） |
| 作用 | 模型输出结构化调用 | 标准化工具发现/调用/通信 |
| 复用 | 每个应用各自集成 | Server 写一次处处可用 |
| 关系 | MCP Server 暴露的工具，最终靠 Function Calling 让模型调 |

---

## 三、A2A（Agent2Agent Protocol）

### 3.1 是什么、谁搞的

- **全称**：Agent2Agent Protocol（Agent 间协议）。
- **提出**：Google，2025 年 4 月发布，开放，众多厂商参与。
- **定位**：让**不同框架、不同厂商的 Agent 能互相发现、互相通信、协作完成任务**的开放标准。类比「Agent 之间的 HTTP」。

### 3.2 解决什么痛点

- Agent 生态碎片化：你的 Agent 用 LangGraph，同事的用 AutoGen，客户的用 Dify，彼此无法互通。
- A2A 让它们用统一协议对话，不强制都用同一框架。

### 3.3 核心概念

- **Agent Card**：每个 Agent 在 `https://<host>/.well-known/agent.json` 暴露一张「名片」，说明自己的能力、技能、认证、端点。客户端靠它发现 Agent（类比微服务的服务发现）。
- **Task（任务）**：协作的基本单位，有生命周期（submitted/working/input-required/completed/failed/canceled 等）。
- **Message / Part**：任务内多轮消息，每条消息含若干 Part（文本/文件/数据等结构化片段）。
- **Artifact（产物）**：任务最终产出（如生成的文件、图片、结构化数据）。
- **传输**：基于 **HTTP + JSON-RPC 2.0**（JSON-RPC，基于 JSON 的远程过程调用协议），流式场景用 **SSE** 推送任务进度。

### 3.4 A2A vs MCP（高频面试题）★

| 维度 | MCP | A2A |
|------|-----|-----|
| 通信对象 | Agent <-> 工具/数据源 | Agent <-> Agent |
| 定位 | Agent 接外部能力的「USB-C」 | Agent 间协作的「HTTP」 |
| 由谁 | Anthropic | Google |
| 抽象 | Tools/Resources/Prompts | Agent Card/Task/Message/Artifact |

> ⚠️ 关键金句：**MCP 和 A2A 是互补不是竞争**。MCP 解决「Agent 怎么用工具」，A2A 解决「Agent 之间怎么对话」。一个 Agent 可以同时：用 MCP 接工具，用 A2A 和别的 Agent 协作。完整链路是 `Agent A --(A2A)--> Agent B --(MCP)--> 工具/数据库`。

---

## 四、ACP（Agent Communication Protocol）

### 4.1 是什么

- **全称**：Agent Communication Protocol（Agent 通信协议）。
- **提出**：由 Agent Communication Protocol 组织（多家参与）推动，约 2025 年发布。
- **定位**：又一个 Agent 间通信标准，强调**异步消息**和**企业级集成**。
- **特点**：
  - 基于 **HTTP/REST + AsyncAPI 规范**描述接口。
  - 支持**同步和异步**消息，异步可对接消息中间件（Kafka/RabbitMQ），适合长任务、跨组织。
  - Agent 通过消息互相调用，强调可观测、可治理。
- **vs A2A**：A2A 偏 Google 生态、JSON-RPC、同步+流式为主；ACP 偏异步、消息总线、企业集成。两者目标相似、实现侧重不同，业界常有「谁成标准」之争。

> ⚠️ 命名冲突提醒：业界还有 Cisco 的 **Agent Connect Protocol（也叫 ACP）**，用于联络中心场景，是另一回事。面试若被问，可澄清「Agent 间通信的 ACP」与「Cisco 联络中心 ACP」不同。

---

## 五、AG-UI（Agent-User Interaction Protocol）

### 5.1 是什么

- **全称**：Agent-User Interaction Protocol（Agent-用户交互协议）。
- **提出**：CopilotKit 主导，2025 年开源。
- **定位**：标准化 **Agent 与前端 UI 之间的交互**。让任意 Agent 后端能把过程（思考、工具调用、状态变化、流式 token）实时、标准地推给任意前端。

### 5.2 解决什么痛点

- 每个 Agent 框架向前端推流的方式都不同，前端要为每个框架写适配。
- AG-UI 定义一组**标准事件**（如 `text-message-chunk` 文本块、`tool-call` 工具调用、`state-snapshot` 状态快照、`human-input-request` 请求人工输入等），通过事件流（SSE/WS（WebSocket））推给前端，前端用通用组件渲染。

### 5.3 协议三角（这是面试的金句图）

```
        工具/数据源
            ▲
            │ MCP（Agent<->工具）
            │
   Agent ◄───┴──► Agent          ◄── A2A / ACP（Agent<->Agent）
            │
            │ AG-UI（Agent<->前端UI）
            ▼
          用户/前端
```

| 协议 | 通信两端 | 解决 |
|------|---------|------|
| **MCP** | Agent <-> 工具/数据源 | Agent 怎么用工具 |
| **A2A / ACP** | Agent <-> Agent | Agent 间怎么协作 |
| **AG-UI** | Agent <-> 用户/前端 | Agent 怎么和用户交互 |

> 三者**互补**，覆盖了 Agent 生态的三个方向：向下接工具（MCP）、横向连 Agent（A2A/ACP）、向上对用户（AG-UI）。

---

## 六、协议对比总表

| 协议 | 由谁 | 对象 | 传输/格式 | 核心抽象 | 状态 |
|------|------|------|----------|---------|------|
| **MCP** | Anthropic | Agent<->工具 | stdio / Streamable HTTP | Tools/Resources/Prompts | 主流、生态最大 |
| **A2A** | Google | Agent<->Agent | HTTP+JSON-RPC (+SSE) | Agent Card/Task/Artifact | 快速增长 |
| **ACP** | ACP 组织 | Agent<->Agent | HTTP/REST + 异步消息中间件 | AsyncAPI/消息 | 企业异步方向 |
| **AG-UI** | CopilotKit | Agent<->前端 | 事件流(SSE/WS) | 标准事件 | 前端交互方向 |
| **SSE** | W3C（World Wide Web Consortium，万维网联盟）标准 | 底层传输 | HTTP text/event-stream | 事件流 | 通用基础（MCP/A2A/AG-UI 多基于它） |

---

## 七、常见面试题

1. **MCP 是什么？解决什么问题？**
   Anthropic 2024.11 提出的开放协议，标准化 Agent 与工具/数据源的连接，把 M×N 集成变成 M+N。架构是 Host/Client/Server，能力原语是 Tools/Resources/Prompts。

2. **MCP 的传输方式有哪些？为什么从 HTTP+SSE 改成 Streamable HTTP？**
   stdio（本地子进程）和 Streamable HTTP（远程）。旧 HTTP+SSE 双连接难负载均衡、状态重，新 Streamable HTTP 单端点可灵活返回 JSON 或升级 SSE 流，更适合生产。

3. **MCP 和 Function Calling 什么关系？**
   MCP 是协议层（标准化工具发现/调用/通信），Function Calling 是模型层（模型输出结构化调用）。MCP Server 暴露的工具最终靠 Function Calling 被模型调用，二者配合不冲突。

4. **A2A 和 MCP 区别？它们竞争吗？**
   不竞争、互补。MCP 是 Agent<->工具，A2A 是 Agent<->Agent。完整链路 `Agent A --A2A--> Agent B --MCP--> 工具`。

5. **A2A 怎么发现 Agent？**
   Agent 在 `/.well-known/agent.json` 暴露 Agent Card（能力/技能/认证/端点），客户端据此发现和调用。

6. **ACP 和 A2A 有什么区别？**
   A2A 偏同步+流式（JSON-RPC/SSE）；ACP 偏异步、企业级，可对接 Kafka/RabbitMQ 消息中间件做长任务跨组织协作。

7. **AG-UI 解决什么？和 MCP/A2A 怎么分工？**
   AG-UI 标准化 Agent<->前端交互，用标准事件流让任意前端渲染任意 Agent 的过程。三者互补：MCP 向下接工具、A2A/ACP 横向连 Agent、AG-UI 向上对用户。

8. **SSE 为什么适合 AI 流式输出？**
   LLM 流式生成是服务器持续推 token 的单向场景，SSE 走 HTTP、轻量、自动重连、对前端友好，比 WebSocket 更简单，是流式输出事实标准。

---

> 本章是理解 Agent 互联互通生态的关键。「07-ClaudeCode」会看到 Claude Code 真实使用 MCP/Skill/Harness 的实现。
