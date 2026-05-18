---
type: harness
name: langgraph
status: active
language: python+typescript
last_ingested_version: 1.2.0
last_ingested_commit: 076e2a3
last_ingested_at: 2026-05-18
community_known_newer: false
known_post_ingest_features: []
sources:
  - "/Users/youren/myspace/langgraph/AGENTS.md"
  - "/Users/youren/myspace/langgraph/libs/langgraph/langgraph/graph/state.py"
  - "/Users/youren/myspace/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py"
  - "/Users/youren/myspace/langgraph/libs/langgraph/langgraph/pregel/"
  - "/Users/youren/myspace/langgraph/libs/prebuilt/langgraph/prebuilt/"
  - "[[../source/langgraph]]"

agent_loop: |
  [verified-from-source]
  **Pregel-style superstep execution + channels message passing**(`libs/langgraph/langgraph/pregel/` 完整模块)— Google graph processing 范式应用在 agent
  **编译/运行分离**:`StateGraph` (`graph/state.py:130`) → `.compile()` → `CompiledStateGraph` (`:1391`)
  **4 generic type parameter**:`class StateGraph(Generic[StateT, ContextT, InputT, OutputT])` — **业内 type-safe agent state machine 最强**
  `prebuilt/chat_agent_executor.py` 高层 ReAct agent + `tool_node.py` 工具执行节点 + `tool_validator.py` 校验

tool_protocol: |
  [verified-from-source]
  `prebuilt/tool_node.py` + `tool_call_stream.py` + `tool_call_transformer.py` + `tool_validator.py` — tool execution layer
  跟 LangChain ecosystem `Runnable` protocol + Tool abstraction 紧耦合(详见 [[../eval/langchain]])
  MCP 集成走 [[../eval/langmem]] / LangChain partners

context_mgmt: |
  [verified-from-source]
  `runtime.py` + `config.py` + `stream/` 模块负责 runtime context 注入
  **Checkpoint 是 persistence 不是 compaction** — LangGraph 假设状态全 checkpoint,不做主动 compaction(对照 [[../harness/claude-code]] 5 级 compaction)

memory: |
  [verified-from-source]
  **Checkpoint 抽象独立 package + 3 backends**(`libs/checkpoint/` base + `checkpoint-postgres/` + `checkpoint-sqlite/` + `checkpoint-conformance/` test suite)
  `checkpoint/base/__init__.py` 860 行 `BaseCheckpointSaver` 完整抽象
  **persistence first-class** — wiki 内对照 [[../harness/openhuman]] SQLite + [[../harness/openclaw]] state / [[../harness/hermes-agent]] PostgreSQL,LangGraph **唯一把 checkpoint 当独立 package + 合规性测试**
  与 [[../eval/langmem]] long-term memory 自然集成(`langmem` package native binding `BaseStore`)

multi_agent: |
  [verified-from-source]
  `prebuilt/` 高层 API:`chat_agent_executor.py` + `tool_node.py` + `interrupt.py`
  StateGraph 本身是 multi-actor 模型 — Pregel _"actor as node"_ 范式
  跟 [[../harness/crewai]] / [[../harness/ag2]] / [[../harness/metagpt]] 4 种 multi-agent 路线对照:LangGraph 是 _"显式状态图 + Pregel"_ 路线

eval_loop: |
  [verified-from-source]
  **`checkpoint-conformance` 独立 package 给 3rd-party checkpointer 实现做合规性测试** — **业内独家**;`libs/checkpoint-conformance/` 是 conformance test suite,任何新写的 checkpointer(如自家 Redis checkpointer)都能跑同一套测试验证
  跟 LangSmith eval / Langfuse 集成走 LangChain 生态

self_evolution: |
  N/A — LangGraph 不涉及 self-evolution

observability: |
  [verified-from-source]
  `callbacks.py` 共享 LangChain ecosystem callback 系统(LangSmith / Langfuse / Opik / Helicone / Phoenix 都通过 callback 接 LangGraph)
  跟 [[../eval/langchain]] callback system **业内事实标准** anchor

hooks: |
  [verified-from-source]
  `prebuilt/interrupt.py` first-class interrupt hook
  Pregel-level node interrupt 支持
  callback 系统也算 hook 路线

safety:
  sandbox: "? 未深查"
  tool_acl: "[verified-from-source] `prebuilt/tool_validator.py` tool 参数校验"
  bash_blocklist: "N/A — LangGraph 框架层不直接执行 shell"
  path_traversal: "?"
  anti_runaway: "[verified-from-source] Pregel-level recursion limit + interrupt + checkpoint rollback"
  prompt_injection: "?"
  secret_scan: "?"
  auth: "[verified-from-source] LangGraph Server / Platform 通过 sdk-py/sdk-js 走 API key auth"

provider: |
  [verified-from-source]
  **复用 LangChain ecosystem 100+ provider**(`init_chat_model("provider:model")` 字符串路由,详见 [[../eval/langchain]])

channels: |
  [verified-from-source]
  **`libs/cli`** — 官方 CLI
  **`libs/sdk-py` + `sdk-js`** — Python + JS/TS SDK 接 LangGraph Server REST API
  **LangGraph Server / Platform**(闭源 SaaS,部署 hosted graph)
  **Pregel `channels/` 内部消息传递** — LastValue / Topic / EphemeralValue / BinaryOperatorAggregate

persistence: |
  [verified-from-source]
  **Checkpoint 独立 package + 3 backends**:memory / sqlite / postgres
  conformance test suite 给 3rd-party 实现兜底
  **wiki 内 persistence 最完整工程化**(对照 hermes-agent PostgreSQL + pgvector / openclaw / openhuman SQLite + Obsidian 双写)

cancellation: |
  [verified-from-source]
  `prebuilt/interrupt.py` + Pregel-level interrupt + 通过 checkpoint rollback 恢复

skillforge_takeaway: |
  ## 可借鉴

  ### 🟡 R-LG-1 P2 — Checkpoint as first-class concern + 3 backends + conformance test 模式(2-3 周)
  LangGraph 把 checkpoint 抽出独立 package + 3 backends(memory/sqlite/postgres)+ `checkpoint-conformance` test suite 给 3rd-party 兜底 — **业内独家 _"persistence first-class"_ 工程姿态**。
  SkillForge 当前 PostgreSQL + pgvector persistence,可借鉴**抽出 conformance test suite + 多 backend SPI**(对照 [[../harness/hermes-agent]] 7 sandbox backends SPI 思路同源)。
  实施路径:`skillforge-checkpoint` 独立 module + `BaseCheckpointSaver` 接口 + sqlite/postgres 双实现 + conformance test。

  ### 🟡 R-LG-2 P2 — StateGraph 4 generic type parameter 类型安全(1-2 周)
  `StateGraph(Generic[StateT, ContextT, InputT, OutputT])` — type-safe agent state machine。Java/Kotlin generic-heavy 自然适配,SkillForge agent loop 可借鉴 4 类型参数划分(状态 / 上下文 / 输入 / 输出)。

  ### 🟢 R-LG-3 P3 — Pregel execution model POC(长期评估)
  agent loop alternative paradigm — 跟 ReAct / event loop / actor model 4 种范式并列。短期不必跟,长期 multi-agent + parallel execution 可评估。

  ### 🟢 R-LG-4 P3 — prebuilt 双层 SDK 模式(可选)
  `prebuilt/` 包是 LangGraph 内的高层 ReAct agent SDK — 底层 StateGraph 灵活但门槛高,prebuilt 给快速上手。SkillForge 远期可考虑双层(底层 agent loop + 高层 prebuilt template)。

  ## SkillForge 现状对照

  - Checkpoint as independent module:? 未 verify SkillForge persistence 是否抽出 SPI
  - StateGraph generic typing:? 待 verify

  ## 风险 / 不抄
  - **LangChain ecosystem 紧耦合**:Runnable protocol + callback + init_chat_model — SkillForge 已有自家 SPI,不抄
  - **LangGraph Server / Platform 闭源** — 商业 SaaS,不做参考

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)
  - **Pregel execution model** — 业内罕见在 agent framework 上用 Google graph processing 范式,跟 ReAct loop / event loop / actor model 形成第 4 种 agent execution paradigm
  - **StateGraph 4 generic type parameter** — `Generic[StateT, ContextT, InputT, OutputT]` 业内 type-safe agent state machine 最强工程实现
  - **Checkpoint 独立 package + 3 backends + conformance test suite** — 业内唯一把 persistence 当独立工程关注点
  - **8 包 monorepo + 清晰 dependency graph**(AGENTS.md:34-53) — checkpoint base 给 4 上游用,prebuilt 给 langgraph,sdk-py 给 langgraph+cli,sdk-js standalone — **解耦设计教科书**

  ## 优势(strengths)
  - **`checkpoint-conformance` 独立 package** — 给 3rd-party checkpointer 实现兜底,行业事实标准姿态
  - **StateGraph → CompiledStateGraph 编译/运行分离** — 类似 LLVM IR 思路,跟 [[../harness/codex]] V8 isolate 编译 JS 一脉相承
  - **LangChain ecosystem callback 复用**(`callbacks.py`)— 不重新发明 observability,直接接 LangSmith/Langfuse/Opik/Phoenix/Helicone
  - **prebuilt 高层 SDK + 底层 StateGraph 双层抽象** — 业界少有的双层 framework 设计

  ## 哪些点做得好(what_done_well)
  - **`AGENTS.md:34-53` dependency map 直接 ASCII 渲染** — 让 LLM agent 立刻看懂 monorepo 结构,工程文档诚实
  - **Pregel channel-based message passing**(`channels/` LastValue/Topic/EphemeralValue/BinaryOperatorAggregate)— 把不同 reduction 语义抽成 channel type,elegant
  - **`prebuilt/interrupt.py` first-class interrupt 模块** — 跟 LangGraph human-in-the-loop 协议级支持(对照 A2A INPUT_REQUIRED / AG-UI interrupts)
  - **`init_chat_model("provider:model")` 字符串风格 provider** — 借力 LangChain ecosystem 100+ provider 零配置
  - **不重新发明 observability** — 完全靠 LangChain callback 接生态,**工程克制**
---

# langgraph

## 一句话定位

**LangChain 出品 stateful agent graph framework**(v1.2.0,2026-05-12)— **业内 _"Pregel + 状态图"_ agent execution model 旗手**,8 包 monorepo + Checkpoint 独立 package + 3 backends + conformance test suite,核心 `StateGraph[StateT, ContextT, InputT, OutputT]` type-safe state machine,prebuilt 包给高层 ReAct agent SDK。

## 核心架构

```
8 包 monorepo:
  checkpoint (base) ─→ checkpoint-postgres / checkpoint-sqlite / prebuilt / langgraph
  prebuilt ─→ langgraph
  sdk-py ─→ langgraph + cli
  sdk-js (standalone)
  checkpoint-conformance (test suite for 3rd-party)

langgraph 核心:
  pregel/      Pregel-style superstep execution(Google graph processing 范式)
  graph/state.py:130  StateGraph(Generic[StateT, ContextT, InputT, OutputT]) 1964 行
  graph/state.py:1391 CompiledStateGraph(编译产物)
  channels/    LastValue / Topic / EphemeralValue / BinaryOperatorAggregate
  callbacks.py LangChain ecosystem callback 复用

prebuilt 包(高层 SDK):
  chat_agent_executor.py + tool_node.py + interrupt.py + tool_validator.py
```

## 跟 SkillForge 对照

详见 frontmatter `skillforge_takeaway`。最高 ROI 是 **R-LG-1 Checkpoint as first-class + 3 backends + conformance test**(2-3 周)。

## 待回答

- pregel/ 模块内部细节(superstep / barrier / message dispatch)
- safety 5 维(sandbox / path_traversal / prompt_injection / secret_scan / bash_blocklist)
- LangGraph Server / Platform 闭源 SaaS 形态
- StateGraph 编译期 schema validation 细节

## 引用链路

### 相关 concept
- [[../concept/agent-team-patterns]] — Pregel + StateGraph 是第 4 种 multi-agent execution paradigm
- [[../concept/runtime-context-injection]] — `runtime.py + config.py` runtime context
- [[../concept/observability]] — LangChain callback 是事实标准 anchor

### 对照 harness
- [[../eval/langchain]] — LangChain framework 母体,LangGraph 继承 Runnable + callback + provider
- [[../eval/langmem]] — LangGraph BaseStore native binding memory framework
- [[../harness/claude-code]] / [[../harness/codex]] — agent loop 不同 paradigm
