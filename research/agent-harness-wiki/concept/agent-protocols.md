---
type: concept
name: agent-protocols
status: stable
updated: 2026-05-17
mode: hybrid (B-mode for 21+ wiki verified anchors + doc-only for 5 external protocol specs)
related_harnesses:
  - claude-code
  - codex
  - agentscope
  - crewai
  - ragflow
  - dify
  - opencode
  - openclaw
  - hermes-agent
  - metagpt
related_evals:
  - ag2
  - agentops
  - cognee
  - helicone
  - langchain
  - langfuse
  - letta
  - llama_index
  - mempalace
  - opik
  - pathway
  - phoenix
  - zep-graphiti
sources:
  # Wiki 内 verified 引用(直接复用)
  - "[[../harness/claude-code]]"
  - "[[../harness/codex]]"
  - "[[../harness/agentscope]]"
  - "[[../harness/crewai]]"
  - "[[../concept/observability]]"
  - "[[../concept/agent-interop]]"
  # 外部协议 spec(2026-05-17 WebFetch)
  - "https://modelcontextprotocol.io/introduction"
  - "https://modelcontextprotocol.io/specification/2025-06-18"
  - "https://a2a-protocol.org/latest/"
  - "https://agentcommunicationprotocol.dev/"
  - "https://agntcy.org/"
  - "https://docs.ag-ui.com/"
  - "https://opentelemetry.io/docs/specs/otel/"
  - "https://opentelemetry.io/docs/specs/semconv/gen-ai/"
---

# Agent 协议栈 — 5 层(4 互通 + 1 贯穿)

> 2025-2026 围绕 agent 形成的**分层协议栈**:不是一个协议解决所有问题,而是 5 个协议各占一层,明示彼此 complementary 不竞争 — **MCP(tool 层) + A2A(agent 层) + AGNTCY(编排/网格层) + AG-UI(UI 层) + OTel(telemetry 侧向贯穿)**。这种 _"协议联盟健康姿态"_ 在标准之争史上罕见。

## 1. 定义与 5 层栈结构

```
┌─────────────────────────────────────────────────────────────────┐
│  AG-UI         Agent → User (UI)              CopilotKit + LG  │
├─────────────────────────────────────────────────────────────────┤
│  AGNTCY        编排/网格 (Internet of Agents)   Cisco + LF      │
├─────────────────────────────────────────────────────────────────┤
│  A2A           Agent → Agent (任务委托)         Google → LF     │
│                (ACP merged in,2026-05-17)                       │
├─────────────────────────────────────────────────────────────────┤
│  MCP           Agent → Tool/Resource/Prompt    Anthropic        │
├─────────────────────────────────────────────────────────────────┤
│  ⊥ OTel ⊥      Telemetry (侧向贯穿所有层)       CNCF graduated  │
│                GenAI semconv 已覆盖 MCP semantics                │
└─────────────────────────────────────────────────────────────────┘
```

**关键观察**:
- **MCP / A2A 是当前 wiki 内 verified 集成的主力**(MCP 18 页 / A2A 4 页)
- **AGNTCY / AG-UI wiki 内引用为 0**(本页是首次 anchor)
- **OTel 是 wiki 内引用最广的协议(21 页)** — 超过 MCP
- **协议联盟健康姿态 3 个证据**:
  1. ACP merge 进 A2A under Linux Foundation(2026-05-17 doc 明示)
  2. AGNTCY 明示 _"middleware above A2A/MCP"_,不试图替代
  3. OTel GenAI semconv 把 MCP semantics 纳入,跨协议协作

---

## 2. MCP — Model Context Protocol(Tool 层)

**[doc-only,2026-05-17 fetched modelcontextprotocol.io]**

> 📖 **本节深挖详见 [[../deep/mcp-spec-deep]]** — LSP 白话故事(2014 M×N 噩梦 / 2016 _前厨后厨分离_) + 6 primitives visibility 区别(只 Tool 进 function_calling) + Streamable HTTP 工程细节(Mcp-Session-Id / Resumability / 4 安全 MUST) + 5 失败模式 + 4 工程反直觉(tool result vs Sampling 主流 / skill+MCP 不互斥 / toolList 7 步注册 / LSP 第二次胜利)+ SkillForge R-HARNESS-1 3 阶段路线。

### 2.1 基础事实

- **Origin**:Anthropic 开源,**2024-11 发布**
- **Spec 当前版本**:`2025-06-18`(TypeScript schema `schema/2025-06-18/schema.ts`)
- **通信**:**JSON-RPC 2.0 + stateful connections**
- **传输**:`stdio` + `Streamable HTTP`(+ legacy SSE 已弃)
- **设计灵感**:**LSP (Language Server Protocol)** —— spec 明示 _"inspiration from Language Server Protocol"_
- **类比**:_"USB-C port for AI applications"_(产品营销史上最成功的协议类比之一)

### 2.2 3 角色架构

| 角色 | 职责 |
|---|---|
| **Host** | LLM 应用(发起连接)— 例:Claude / ChatGPT / Cursor / VS Code |
| **Client** | Host 内的连接器(per-server)|
| **Server** | 提供 context + capability 的服务 |

### 2.3 6 primitives(对称设计)

**Server 给 Client 的 3 primitives**:
- **Resources** — Context/数据(文件、数据库、URL)
- **Prompts** — 模板化 messages/workflow
- **Tools** — Functions 供 LLM 执行

**Client 给 Server 的 3 primitives**:
- **Sampling** — Server-initiated recursive LLM 调用
- **Roots** — Server 查询 URI/文件系统边界
- **Elicitation** — Server 请求用户额外信息

**对称性意义**:避免 _"server 啥都能做客户端只能听"_ 的不对等;允许 server 反向 query host 的 LLM。

### 2.4 安全 4 原则(spec 明示)

| 原则 | 含义 |
|---|---|
| **User Consent and Control** | 用户必须显式同意所有数据访问和操作 |
| **Data Privacy** | Host 必须在暴露用户数据前获得 consent;不得未授权外传 |
| **Tool Safety** | Tool 表示任意代码执行,annotation 默认不可信,使用前必须 consent |
| **LLM Sampling Controls** | 用户必须批准 sampling 请求,控制 prompt 和结果可见性 |

**spec 诚实之处**:_"MCP cannot enforce these security principles at the protocol level"_ — 协议层面无法 enforce,implementor MUST 落实。

### 2.5 capability negotiation

在 `initialize` 握手中协商 server 和 client 各自支持哪些 primitives — 与 LSP `initialize` 思路一致。

---

## 3. A2A — Agent2Agent Protocol(Agent 层)

**[doc-only,2026-05-17 fetched a2a-protocol.org/latest]**

> 📖 **本节深挖详见 [[../deep/google-a2a-spec]]** — 跟 MCP 根本错位 mental model("借用工具" vs "委托任务") + Agent Card 两层 card 设计(public + extended) + Task 8 状态机详细 + 8 JSON-RPC 方法 + 3 invocation 模式(同步/SSE/Webhook) + 完整 invoice walkthrough + 5 失败模式(IP 泄漏 / 长任务断线 / 恶意 Agent Card / Replay / 同步思维错误用 A2A)+ wiki 内 2 家落地路线对照(crewai agent-as-tool vs agentscope agent-as-actor) + SkillForge 3 阶段路线。

### 3.1 基础事实

- **Origin**:**Google 发起 → 转 Linux Foundation**,**v1.0 发布近期**(2025)
- **通信**:JSON-RPC over HTTP(社区知识;doc 未明确)
- **状态模型**:Streaming + Asynchronous + Multiturn
- **核心原则**:_"Agents interact without needing to share internal memory, tools, or proprietary logic"_ —— IP 保护

### 3.2 4 核心 primitives

| Primitive | 含义 |
|---|---|
| **Agent Card** | 描述 agent 能力 + identity 的 metadata(用于发现,类似 OpenAPI schema)|
| **Task** | Agent 间委派的工作单元(stateful,可 stream) |
| **Skill** | Agent 的具体能力/动作 |
| **Message** | 交互通信单元(支持 multiturn) |

### 3.3 跟 MCP 的定位(doc 明示)

> _"MCP: Provides agent-to-tool communication. A2A: Provides agent-to-agent communication. The protocols are complementary rather than competitive."_

### 3.4 ACP 合并

> _"IBM ACP has been incorporated into A2A under the Linux Foundation."_(2026-05-17 a2a-protocol.org 明示)

**ACP historical note**:
- **Origin**:IBM BeeAI 参考实现
- **传输**:REST-based HTTP(跟 A2A JSON-RPC 不同 — 这是合并前的历史差异)
- **特点**:MIME type + 同步/异步/streaming + offline discovery via 嵌入式 metadata
- **意义**:ACP → A2A 合并是 _"标准联盟健康"_ 信号,避免协议碎片化;**Linux Foundation 扮演 _"协议聚合 hub"_**

---

## 4. AGNTCY — Internet of Agents(编排/网格层)

**[doc-only,2026-05-17 fetched agntcy.org]**

### 4.1 基础事实

- **Origin**:**Cisco Outshift + Linux Foundation**;founding members:Google + Dell + Oracle + Red Hat
- **定位**:**middleware layer above A2A/MCP**(不替代,做跨协议 mesh)
- **状态**:active development,组件成熟度不一

### 4.2 5 大组件

| 组件 | 实现 |
|---|---|
| **Agent Discovery** | **OASF**(Open Agent Schema Framework)|
| **Agent Identity** | 加密验证 + 访问控制 |
| **Agent Messaging** | **SLIM**(Secure Low-latency Interactive Messaging)protocol |
| **Agent Observability** | 跨厂商监控 |
| **Protocol Integration** | 跨协议支持(A2A / MCP) |

### 4.3 跟 A2A/MCP 关系

- _"Internet of Agents"_ 是 AGNTCY 自身定位 — 跟 IETF 早期 _"协议族"_(IP / TCP / HTTP)思路同源
- **不竞争 A2A/MCP**,而是 _"协议之上的协议"_,做发现 + 身份 + 观测的统一抽象

---

## 5. AG-UI — Agent-User Interaction Protocol(UI 层)

**[doc-only,2026-05-17 fetched docs.ag-ui.com]**

### 5.1 基础事实

- **Origin**:**CopilotKit 主导**,LangGraph + CrewAI partnership 起步
- **定位**:_"Agent → User Interaction"_(协议栈第 4 层,补齐 MCP/A2A 之外的人机交互层)
- **传输**:**HTTP + WebSockets**(event-based)
- **状态模型**:**streamed event-sourced state**,read-only + read-write shared store

### 5.2 核心 events(bidirectional)

| Event 类 | 内容 |
|---|---|
| 流式 chat | Token-level 流式 |
| 多模态 | 文件 / 图像 / 音频 |
| **Generative UI** | 静态 typed component + 声明式约束 UI |
| Shared typed state | Event-sourced diffs + conflict resolution |
| Tool calls | Frontend + Backend 双向 |
| **Human-in-the-loop interrupts** | **协议层一等公民** |
| Custom events | 扩展性 |

### 5.3 集成 traction(惊人广度)

| 集成层级 | framework |
|---|---|
| **1st-party** | Microsoft Agent Framework / **Google ADK** / AWS Strands Agents / AWS Bedrock AgentCore / Mastra / Pydantic AI / Agno / LlamaIndex / AG2 |
| **Partnership** | LangGraph / CrewAI |
| **进行中** | OpenAI Agent SDK / Cloudflare Agents |
| **SDK** | Kotlin / Go / Dart / Java / Rust / .NET / Nim |

### 5.4 跟 MCP/A2A 关系(doc 明示)

> _"MCP=tool, A2A=agent, AG-UI=UI"_ — 协议栈完整

---

## 6. OTel — OpenTelemetry(Telemetry 层,侧向贯穿)

**[doc-only,2026-05-17 fetched opentelemetry.io/docs/specs/]**

### 6.1 基础事实

- **Origin**:**CNCF graduated**;OpenTracing + OpenCensus 合并(2019)
- **Spec 当前版本**:**1.56.0**
- **协议**:**OTLP**(OpenTelemetry Protocol)over gRPC/HTTP/JSON
- **3 信号类型**:**Traces / Metrics / Logs**

### 6.2 GenAI semconv(2025-2026 状态)

> ⚠️ **当前在 _Development_ 状态,未 stable**(2026-05-17)

- 已覆盖:Anthropic / Azure AI Inference / AWS Bedrock / OpenAI + **MCP semantics(!)**
- 3 层 span hierarchy:**LLM span / Agent span / Tool span**
- 关键 attribute:`gen_ai.operation.name` / `gen_ai.agent.id`(参考 `gen-ai-spans.md` + `gen-ai-agent-spans.md`)
- backward compat 通过 `OTEL_SEMCONV_STABILITY_OPT_IN` env var 切

### 6.3 侧向贯穿性质

OTel 是 5 个协议里**唯一不占某层独家**,而是**跨所有层做 observability**:

- 给 MCP server 的 tool call 加 span
- 给 A2A task 加 span
- 给 AGNTCY 编排链加 span
- 给 AG-UI event 加 span

**最强证据**:OTel GenAI semconv **已经把 MCP semantics 纳入**(2026-05-17 doc 明示) — _"协议之间相互纳入对方"_ 的协作姿态。

### 6.4 跟 Wide Events 2.0 的关系

详见 [[observability]]:
- **OTel 1.0** — 窄事件多信号(traces + metrics + logs 分开)
- **Wide Events 2.0** — 宽事件单信号(Langfuse 路线,Charity Majors 理论)
- **三分法**:OTel 1.0 / Wide Events 2.0 / Hybrid

---

## 7. 8 维设计空间

| 维度 | MCP | A2A | AGNTCY | AG-UI | OTel |
|---|---|---|---|---|---|
| **D1 层级** | tool | agent | 编排/网格 | UI | telemetry(贯穿) |
| **D2 通信** | JSON-RPC 2.0(stdio/Streamable HTTP) | JSON-RPC(HTTP)+ streaming | SLIM + 跨协议 | HTTP + WebSocket(event-based) | OTLP/gRPC/HTTP |
| **D3 发现** | Capability negotiation(initialize) | **Agent Card** endpoint | **OASF** schema | N/A | N/A |
| **D4 状态** | Stateful connection | Stateful task + streaming + multiturn | N/A | Event-sourced streaming | Stream events |
| **D5 trust/auth** | 4 明示原则(implementor MUST) | _"不共享 internal memory/tools/logic"_ + IP 保护 | **Agent Identity 加密** first-class | 跟随上层 | 跟随上层 |
| **D6 跨语言 SDK** | TS/Python/Java/Rust/Go/Swift 等 | 多 SDK | 多语言 | Kotlin/Go/Dart/Java/Rust/.NET/Nim 进行中 | CNCF 标准多 SDK |
| **D7 生态(wiki 引用数)** | **18 页**(事实标准) | **4 页**(普及中) | **0 页**(本页首次 anchor) | 间接 1 页 | **21 页(最广)** |
| **D8 安全模型** | 4 原则 + Tool Safety annotation 默认不可信 | Agent IP 保护 + Agent Identity 未明示 | 加密身份 first-class | 跟随上层 | Semconv 命名规范 |

---

## 8. Wiki 内集成方对照

### 8.1 MCP 集成 18 页阶梯

| 集成深度 | wiki 页 | 关键证据 |
|---|---|---|
| **First-class(自家)** | [[../harness/claude-code]] | Anthropic 自家;40+ tools + MCP server 一等 |
| **独立 crate/模块** | [[../harness/codex]] / [[../harness/agentscope]] / [[../harness/crewai]] | codex `mcp/` crate;agentscope `mcp/_mcp_function.py` _"MCP-as-local-function"_;crewai `mcp/` 子目录 |
| **Client + Server 双向** | [[../harness/ragflow]] / [[../harness/openclaw]] | ragflow MCP client+server;openclaw pi-sdk 内嵌 MCP |
| **平台级集成** | [[../harness/dify]] / [[../harness/opencode]] / [[../harness/hermes-agent]] | dify MCP 集成;opencode `acp/mcp/` packages;hermes MCP |
| **专用 MCP server 深度** | [[../eval/mempalace]] | **29 MCP tools(业内 memory framework MCP 最深)** |
| **MCP server 一键 install** | [[../eval/phoenix]] / [[../eval/helicone]] / [[../eval/opik]] / [[../eval/zep-graphiti]] / [[../eval/agentops]] | phoenix MCP 一键 Cursor install;zep-graphiti 2 MCP server(Python+Go);agentops Smithery |
| **集成提及** | [[../eval/cognee]] / [[../eval/langchain]] / [[../eval/pathway]] / [[../eval/ag2]] | — |

**结论**:**MCP 是 wiki 18 家集成的事实标准** — 几乎所有 harness/eval 都集成。这是协议层面 _"已经赢了"_ 的现象。

### 8.2 A2A 集成 4 页阶梯

| 集成深度 | wiki 页 | 关键证据 |
|---|---|---|
| **深度集成 + 服务发现** | [[../harness/agentscope]] | `a2a/` 5 文件 + `_nacos_resolver.py`(**Alibaba Nacos 服务发现**)+ `formatter/_a2a_formatter.py` 跨进程/跨主机 |
| **直接集成** | [[../harness/crewai]] | `a2a/` 子目录 + Google A2A protocol |
| **对照引用** | [[../eval/ag2]] / [[../concept/agent-state-separation]] | — |

**结论**:A2A wiki 内集成 **2 个 harness** — 比 MCP 少 1 个数量级,**正在普及阶段**。

### 8.3 AGNTCY / AG-UI wiki 内引用

| 协议 | wiki 内引用 | 说明 |
|---|---|---|
| AGNTCY | **0** | 本页是首次 anchor — 等 SkillForge / 任何 wiki harness 集成后可补 |
| AG-UI | 间接 1(待 verify) | docs.ag-ui.com 列 1st-party 集成含 [[../eval/ag2]] + Mastra + Pydantic AI 等;待逐一 verify |

### 8.4 OTel 集成 21 页阶梯(wiki 内引用最广)

| 集成深度 | wiki 页 |
|---|---|
| **独立 crate / first-class** | [[../harness/codex]] `otel/` crate / [[../harness/opencode]] OpenTelemetry first-class / [[../harness/agentscope]] **6 OTel trace decorator** / [[../eval/agentops]] OTel-based / [[../eval/opik]] 3 SDK via OTel / [[../eval/ag2]] **`opentelemetry/` 内置(40k+ stars 项目独家)** |
| **完整 stack 集成** | [[../eval/langfuse]] / [[../eval/phoenix]] **OpenInference semconv 主推** / [[../eval/helicone]] / [[../eval/letta]] / [[../eval/llama_index]] / [[../eval/langchain]] callback system / [[../eval/zep-graphiti]] `OTEL_TRACING.md` |
| **集成提及 / concept 引用** | [[../harness/metagpt]] / [[../harness/openclaw]] / [[../concept/observability]] / [[../concept/agent-eval]] / [[../concept/trace-based-eval]] / [[../concept/attribution]] / [[../concept/self-evolution]] |

**结论**:**OTel 是 wiki 内 verified 引用最广的协议(21 页)— 超过 MCP 18 页**。OTel 已经赢了 telemetry 标准之争,GenAI semconv 在 Development 中只是 _"AI agent 特化"_ 层。

---

## 9. Trade-off 横切

### 9.1 协议层 vs 实现层

每个协议都有 _"spec 标准化 vs 实现自由"_ trade-off:
- **MCP** 协议层只规定 JSON-RPC + 3+3 primitives;security 4 原则 _"protocol cannot enforce"_。
- **A2A** 协议层规定 Agent Card schema + Task model;具体 transport 留实现选。
- **AGNTCY** 故意 _"middleware above"_,留 A2A/MCP 实现层不动。
- **OTel** 协议层只定 OTLP wire format + Semconv attribute name;后端实现完全自由。

### 9.2 谁负责安全?(协议 vs implementor)

| 协议 | 安全责任分布 |
|---|---|
| MCP | 4 原则文档化,**implementor MUST 落实**(host 控制 consent flow) |
| A2A | 协议层 _"不共享 internal logic"_;**调用方 MUST 控制 Agent IP 暴露** |
| AGNTCY | Agent Identity 加密 first-class — 协议层多承担 |
| AG-UI | 跟随上层 host app |
| OTel | Semconv 命名规范,**不涉及 access control** |

**规律**:**底层协议(MCP/A2A)把安全责任推给 implementor;中间层协议(AGNTCY)在协议层多承担**。

### 9.3 协议成熟度阶梯(2026-05-17)

| 协议 | wiki 引用数 | 状态 | 行业 buy-in |
|---|---|---|---|
| OTel(基础) | 21 | **stable spec 1.56.0**(GenAI semconv Development) | CNCF graduated,跨行业 |
| MCP | 18 | **stable spec 2025-06-18** | Anthropic+OpenAI+Microsoft+所有 IDE |
| A2A | 4 | **v1.0 发布近期** | Google+LF+IBM(via ACP merge) |
| AG-UI | 间接 1 | 1st-party 集成完成 10+ framework | CopilotKit+LG+CrewAI partnership |
| AGNTCY | 0 | active development | Cisco+Google+Dell+Oracle+RH |

---

## 10. SkillForge 选型

> ⚠️ **SkillForge MCP / A2A / OTel / AG-UI / AGNTCY 集成现状未 verify** —— 用户对齐:暂标 `?`,后续读 `/Users/youren/myspace/skillforge` 源码确认。

### 10.1 SkillForge 现状

| 协议 | SkillForge 集成 |
|---|---|
| MCP | ? **未 verify** —— 社区知识 SkillForge 有 `mcp/` 目录(待 verify) |
| A2A | ? **未 verify** —— 大概率未集成(A2A 是 2025 新协议) |
| OTel | ? **未 verify** —— [[concept/observability]] 已分析 Wide Events 2.0 路线选择,但 SkillForge OTel 集成程度待 verify |
| AG-UI | ? **未 verify** —— SkillForge 是否有 frontend agent 交互需求待确认 |
| AGNTCY | ? **大概率未集成** |

### 10.2 条件性建议(待 verify 后升级为 R-PROTO-N)

| 协议 | 优先级 | 理由 |
|---|---|---|
| **MCP** | 🔴 **P0 必做** | wiki 18 家全集成,行业事实标准,**不集成 = 跟不上行业 baseline**。先 client(消费 MCP server)后 server(暴露 SkillForge tool 给 Claude/Cursor)|
| **OTel** | 🔴 **P0 必做** | wiki 21 家集成最广,跟现有 [[concept/observability]] 三分法选型直接相关。Wide Events 2.0(Langfuse 路线)兼容 OTel(GenAI semconv) |
| **A2A** | 🟡 **P1 应做** | [[harness/agentscope]] A2A + Nacos 跨进程是 _"SaaS multi-tenant agent team"_ 现成方案,跟 [[concept/agent-team-patterns]] R-TEAM-* 互补 |
| **AG-UI** | 🟢 **P2 观察** | 如 SkillForge 有 frontend agent 交互需求,AG-UI 是低风险标准选择(1st-party 集成已有 10+ framework)|
| **AGNTCY** | ⚪ **P3 不必跟** | middleware 层工程门槛高,要 Cisco-level multi-org 部署才需要 |

### 10.3 推荐路线(待 verify 后细化)

**先 P0 双协议(MCP + OTel)** → **再 P1(A2A)** → **远期 P2(AG-UI)** → **跳过 P3(AGNTCY)**。

跟 [[../backlog]] §5 协议条目联动 — 这页 ingest 完成后,deep dive 可以做 `deep/mcp-spec-deep` 或 `deep/google-a2a-spec`(后续单独 ingest)。

### 10.4 工程决策框架 — 3 个边界

> **关键 mental model**:_"agent 调 agent"_ 的协议选型,取决于**跨什么边界**。同一个问题在不同边界下答案完全不同。

```
┌─────────────────────────────────────────────────────────────────┐
│  边界 1:同进程内(同一个 CC / SkillForge 进程)                  │
│   → skill 封装 / TeamCreate / SubAgent / in-process actor 够用   │
│   → A2A overkill:JSON-RPC over HTTP 是在 OS pipe 之上加一层     │
│   ✅ 当前 wiki 多数 harness 在这里(CC Coordinator / agentscope)│
├─────────────────────────────────────────────────────────────────┤
│  边界 2:同信任域跨进程(同一公司 / 同一用户的多台机器)          │
│   → MCP-style RPC 或自家 IPC 协议够用                            │
│   → A2A 可选,但 _"假设跨信任域 + IP 保护"_ 设计是 over-spec      │
│   ⚠️ SkillForge SaaS 内部多 worker 协作可能在这里                │
├─────────────────────────────────────────────────────────────────┤
│  边界 3:跨信任域(跨组织 / 跨厂商 / 公开市场)                  │
│   → A2A 才是 native 设计场景                                    │
│   → 不能假设对方暴露 prompt / 不能 share memory / 需要 IP 保护  │
│   🎯 A2A 的真正主场,wiki 内目前 0 家在做                        │
└─────────────────────────────────────────────────────────────────┘
```

**决策规则**:**跨什么边界,选什么协议** —— 同进程别上 A2A,跨信任域别只用 skill 封装。

### 10.5 A2A 落地两条路线(wiki 内 verified)

A2A 协议没规定 _"集成方一定怎么暴露 agent 给本地 LLM"_,实际有两条路线分流:

| 路线 | 代表 | LLM 视角 | A2A 语义保留度 |
|---|---|---|---|
| **A. agent-as-tool**(类 MCP) | [[../harness/crewai]] `a2a/` | 看到一个 fine-grained tool,跟 MCP 一样塞进 function_calling | ⚠️ **降级** — 丢 long-running + INPUT_REQUIRED 语义 |
| **B. agent-as-actor**(跨进程 actor) | [[../harness/agentscope]] `a2a/` 5 文件 + Nacos | 看到一个 _"另一个 agent"_,通过显式 delegate / channel 触发 | ✅ **完整保留** A2A 语义,但 LLM 需要 multi-agent affordance |

**两条路线的本质差异**:
- **路线 A** 把 A2A 降级成 _"远程 tool"_ —— 简单,塞 function_calling 即可,但只能用 A2A `message/send` 同步模式,**`message/stream` SSE / `INPUT_REQUIRED` / Webhook push 三大杀器全废**
- **路线 B** 保留 A2A 完整语义 —— `Task` 8 状态机 / streaming / human-in-the-loop / IP 保护都能落地,但 LLM 不再把 A2A agent 当 _"tool"_,而是 _"协作伙伴"_(需要 multi-agent prompt scaffold)

**SkillForge 选型暗示**:如果只走路线 A,A2A 跟 MCP 差异不大,**P1 优先级正当**;如果想要路线 B 完整语义,跟 [[../concept/agent-team-patterns]] R-TEAM-* multi-agent 内部抽象**深度耦合**,需要把 SubAgent / Team / A2A 三个抽象统一。

### 10.6 校准:**CC Coordinator ≠ A2A**(消除误解)

Wiki [[../harness/claude-code]] 列了 `TeamCreateTool` + `SendMessageTool` + Coordinator Mode([code: `coordinator/coordinatorMode.ts:36`])作为第 3 种 multi-agent 模式。**容易误以为这是 A2A 协议落地** — 不是。

| 维度 | A2A 协议 | CC Coordinator Mode |
|---|---|---|
| 传输 | JSON-RPC over HTTP + SSE | **`<task-notification>` XML 内部消息**([code: `coordinatorMode.ts:145-156`]) |
| 发现 | `.well-known/agent-card.json` 公开 endpoint | **无 discovery**,worker 由 Coordinator 进程内 spawn |
| 信任模型 | 跨信任域 + Agent 不共享 internal logic | **同信任域**(同用户同机)|
| Task 状态机 | 8 状态(SUBMITTED/WORKING/INPUT_REQUIRED/...) | **无显式 task 状态机**,worker run→return→terminate |
| 部署边界 | 跨进程 / 跨主机 / 跨组织 | **同一个 CC 进程**(或本机 IPC) |

**结论**:CC Coordinator 跟 A2A **解决相同问题(agent 委托 agent),但 transport 完全不同,信任模型完全不同**。CC 选 in-process XML 因为它**不需要跨厂商互通**;A2A 选 JSON-RPC over HTTP 因为它**就是要跨厂商互通**。

跟 [[multi-agent-isolation]] §Coordinator Mode 校准一致:Coordinator 是 _"Hermes 模式 × N workers"_ — in-process 网状,不是协议层互通。

### 10.7 skill 封装 vs A2A 决策表(给 SkillForge 当前问题)

> **触发问题**:_"如果把 _'调用另一个 agent'_ 封装成 skill,A2A 协议还有用吗?"_

| 场景 | skill 封装够? | A2A 必要性 |
|---|---|---|
| CC TeamCreate worker(同进程) | ✅ 够 | ❌ 不必,同进程 |
| SkillForge 内部 SubAgent | ✅ 够 | ❌ 不必,同信任域 |
| SkillForge SaaS 内部多 worker 跨机器 | ✅ 够 | ⚠️ 可选,自家 IPC 可代替 |
| SkillForge **提供** agent 给外部 SaaS 客户 | ⚠️ 勉强 | 🟡 **应该接 A2A**(标准化 + IP 保护) |
| SkillForge **集成** 第三方 agent SaaS(Devin / Replit / Jules) | ❌ 不够 | 🔴 **必须接 A2A**(对方不会暴露 prompt + 需要 long-running) |
| 跨厂商 agent federation(Claude ↔ GPT) | ❌ 不够 | 🔴 **A2A 是 default** |
| Agent marketplace / 公开 agent 注册表 | ❌ 不够 | 🔴 **`.well-known/agent-card.json` 是 native 设计** |

**3 个 skill 封装搬不过来的 A2A 设计点**:

1. **Task 状态机 + INPUT_REQUIRED 反向 query** —— skill 是 _"call → return"_ 一发一收,中途反向问用户需要外部回调机制,不优雅。
2. **跨组织 IP 保护** —— skill 内部代码看得见远程 agent prompt,**违反 A2A _"双方不共享 internal memory/tools/logic"_ 核心原则**。如果 SkillForge 想做 _"我接你 SaaS 的 agent,你不能看到我的 prompt"_,skill 不行。
3. **Agent 公开发现** —— skill 只能调你 _"事先知道存在"_ 的 agent;A2A `.well-known/agent-card.json` 允许 _"新发现一家 agent 服务,通过 capability negotiation 接进来"_,marketplace 形态原生支持。

**给 SkillForge 的现在建议(不等 verify 也成立)**:

1. **现在不必急做 A2A**,但**做好 _"未来加 A2A 不痛"_ 的架构准备**:
   - Skill 抽象保留 _"远程 vs 本地"_ 切换点 — 现在 skill 调本地 agent,未来 skill 内部走 A2A client 调远程 agent,LLM 看到的 skill 接口不变
2. **Skill 元数据对齐 Agent Card 思路** — name / capability / examples / supportedContentTypes,**未来转 A2A 几乎零成本**
3. **Skill SDK 支持 INPUT_REQUIRED 语义** — 即使不走 A2A,内部 multi-agent 也能用 long-running affordance(对齐 [[../concept/agent-team-patterns]])

### 10.8 时间线判断 — A2A 何时变成 _"不接不行"_

| 时间 | A2A 状态 | SkillForge 应对 |
|---|---|---|
| **2026 现在** | wiki 2 家 verify 集成,探索期 | **不必急**,做架构准备(§10.7) |
| **2027-2028** | _"agent SaaS 跨厂商集成"_ 需求出现 | **A2A server 必选**(暴露给外部) |
| **2030+** | Agent marketplace 成熟,A2A registry 像 npm 一样基础设施 | **A2A client + server 双向**,但这是远未来,不左右当前路线 |

**类比**:**MCP 给 LLM ↔ Tool 标准化,A2A 给 Agent ↔ Agent 跨域标准化**。两个协议都不强制内部用,但都给跨域提供了 default 选择 — 公司不会因为 A2A 出来就把内部 function call 换成 HTTP/JSON-RPC,但跨公司协作 default 会用 A2A。

---

## 11. 学习视角

### 11.1 跟其他 concept 不同的是(differentiator)

- **Agent 协议栈是 2025-2026 协议联盟健康姿态最好的例子** — 5 个协议明示彼此 _"complementary"_ 不竞争,**还有 ACP merge 进 A2A、OTel GenAI semconv 把 MCP semantics 纳入两个合并案例**。反面是 W3C 推 Web Components 时各厂商互掐 / OpenStack 早期 API 碎片化。
- **MCP 的 USB-C 类比是协议传播史上最成功的产品营销** — 类比成功直接转化为 _"行业默认假设"_:任何新 LLM 应用都假定要支持 MCP,这是 _"协议网络效应"_ 的 textbook 案例。**1 年内从发布到事实标准**(2024-11 → 2026-05 wiki 18 家集成)。
- **OTel 是 _"侧向贯穿"_ 协议** —— 不占某层独家,而是跨所有层做 observability。GenAI semconv 把 MCP semantics 纳入正是这种 _"协议中协议"_ 的姿态 — wiki 21 家集成证明 OTel 的位置无可替代。
- **AGNTCY 是 _"协议之上的协议"_** —— 不是新通信协议,而是 _"跨协议 mesh"_。这种 "infrastructure protocol" 跟 IETF 早期 _"协议族"_ 思路(IP vs TCP vs HTTP)同源。

### 11.2 优势(strengths,横向比谁做得对)

- **MCP** 18+ wiki 内集成是 _"协议设计 → 产品落地"_ 1 年内完成的速度奇迹。**核心做对**:USB-C 类比 + 3+3 primitives 对称 + LSP-inspired 设计经验复用 + Anthropic 自己 dogfood([[../harness/claude-code]] first-class 集成)。
- **A2A** **Agent Card 抽象** 做得对 —— 把 _"agent 的服务发现"_ 标准化成 endpoint,跟 OpenAPI 风格一致,**借力 web 开发者已有 mental model**。
- **AG-UI** **1st-party 集成 10+ framework 是惊人 traction** —— 比 A2A wiki 内仅 2 集成对比强烈。**做对的是 _"协议先 partner 再 spec"_** — CopilotKit 拉 LangGraph/CrewAI 合作伙伴入伙,再发协议。
- **OTel** GenAI semconv **把 MCP semantics 纳入** —— 协议之间 _"我覆盖你的语义"_ 而不是 _"我替代你"_ 是协议合作的 elegant 姿态。
- **ACP → A2A 合并** 是 standardization 罕见健康姿态 —— Linux Foundation 在这里扮演 _"协议聚合 hub"_ 角色。比 _"各自为政各推一份"_ 的 W3C 模式更高效。

### 11.3 哪些点做得好(what_done_well,elegant 设计)

- **MCP 把 LSP 经验直接复用**(spec 明示 _"inspiration from Language Server Protocol"_) —— 不是另起炉灶,而是 _"agent 领域的 LSP"_。这是工程上 elegant 决策,降低 spec 设计风险 + 让 IDE 厂商(VS Code/Cursor)接入门槛低。
- **MCP 6 primitives 完整对称**:Server 给 3(Resources/Prompts/Tools)+ Client 给 3(Sampling/Roots/Elicitation)。**对称设计避免 _"server 啥都能做客户端只能听"_ 的不对等**,允许 server 反向 query client。
- **MCP 安全 4 原则文档明示** —— User Consent / Data Privacy / Tool Safety / LLM Sampling Controls。**把 _"security 不是 protocol 能 enforce 的,而是 implementor MUST 实现的"_ 写进 spec**,这种诚实的 _"协议局限"_ 表达罕见。
- **A2A _"不共享 internal memory/tools/logic"_** —— 协议层面就限定 _"agent 之间是黑盒"_,**避开了 _"暴露 prompt + tool 给对方 = 数据泄露"_ 的设计陷阱**。这跟 [[multi-agent-isolation]] _"完全隔离 vs 共享 parent context"_ 选择直接对应,A2A 协议站完全隔离一边。
- **AG-UI 把 _"human-in-the-loop interrupts"_ 当 first-class event** —— 不是后置约束,而是协议级别支持。**承认 _"agent 经常需要人介入"_,跟 [[../harness/codex]] `request_user_input` 协议事件思路一致**(都把人介入做成 protocol-level)。
- **AGNTCY 不试图替代 A2A/MCP** —— 协议联盟里最容易犯的错是 _"我要做下一代统一标准"_,AGNTCY 明示 _"middleware above A2A/MCP"_ 是 elegant 的协议政治姿态,提高 buy-in 概率。
- **OTel GenAI semconv 还在 Development 状态** —— 说明 _"AI agent 的 telemetry 标准化滞后于 agent 协议化"_。MCP 已 production-ready 18 月,GenAI semconv 还在草案阶段。这是 _"协议演化次序"_ 的有趣观察:**先有 agent → 才有 agent telemetry standardization**,跟现实中产品先行、规范后跟的常规一致。

---

## 12. 待回答的问题(lint 扫描)

- [ ] **SkillForge MCP / A2A / OTel / AG-UI 集成现状 verify** — 读 `/Users/youren/myspace/skillforge` 源码,把 §10 条件性建议升级成 R-PROTO-N 决策清单。
- [ ] **AG-UI 1st-party 集成 wiki 内 verify** — docs.ag-ui.com 列 1st-party 含 ag2 / Mastra / Pydantic AI / Agno / LlamaIndex 等,待逐一 verify wiki 已有页面是否提到 AG-UI(目前只 [[../eval/ragas]] 一处待验证)。
- [ ] **GenAI semconv 稳定后** — 跟踪 OTel GenAI semconv 从 Development → Stable 时间点,届时重读 spec 看是否影响 wiki [[concept/observability]] 三分法判断。
- [ ] **`deep/mcp-spec-deep`** — MCP `2025-06-18` spec 逐节深挖,backlog §5 已列。
- [ ] **`deep/google-a2a-spec`** — A2A v1.0 spec 逐节深挖,backlog §5 已列。
- [ ] **AGNTCY 进入 wiki 引用** — 等任何 harness 集成 AGNTCY 后再补这页 §8.3。
- [ ] **A2A 跟 OpenAPI 关系** — Agent Card 跟 OpenAPI schema 是否能互转(社区有讨论,未 verify)。

---

## 13. 引用链路

### Wiki 内反向链(本次更新)

- [[concept/agent-interop]] §定义 — 加 _"详见 [[agent-protocols]] 5 层栈"_(本次 reverse link)
- [[concept/observability]] — _"Wide Events 2.0 vs OTel 1.0 三分法"_ 跟本页 §6 OTel 协议层视角形成双向(本次 reverse link)

### 相关 concept

- [[multi-agent-isolation]] — A2A _"不共享 internal logic"_ 跟"完全隔离"路线对应
- [[code-mode-orchestration]] — Code Mode 跟 MCP tool 是两种 _"LLM 调工具"_ 范式
- [[hooks-design]] — AG-UI `human-in-the-loop interrupts` 跟 hook 设计正交但概念相邻
- [[agent-team-patterns]] — A2A 是 multi-agent team 的协议层基础

### 外部协议 spec(2026-05-17 fetched)

见 frontmatter `sources`。
