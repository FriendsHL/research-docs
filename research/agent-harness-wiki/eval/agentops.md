---
type: eval-framework
name: agentops
status: active
language: python + typescript

# === 版本时效 ===
last_ingested_version: "agentops @ commit a855a92 (2026-03-19)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - 2026-03-19 之后的新 framework instrumentation(2 个月未跟进)

sources:
  - "[[../source/agentops]]"

# === 9 维度 ===
target: |
  **业内独家:专门 agent-level observability platform**(MIT,AgentOps-AI)。slogan: **"Observability and DevTool platform for AI Agents"**。

  跟其他 observability 路线对比:
  - [[../eval/langfuse]] / [[../eval/opik]] / [[../eval/phoenix]] = **LLM-level / trace-level / wide events** observability
  - [[../eval/helicone]] = **AI Gateway-first** observability
  - **AgentOps = agent-level OTel observability**(12 SpanKind 含 4 个 agent-level:WORKFLOW / SESSION / TASK / OPERATION / AGENT)

  **关键差异**:**AgentOps 强制 agent 语义层级**(`@session / @agent / @operation / @task` 4 装饰器),其他 framework 走通用 trace 抽象。

  跟 [[../eval/langchain]] 关系:LangChain ecosystem 集成 AgentOps via instrumentation,但 AgentOps 跨 9 个 agent framework(不限 LangChain)。

  跟 [[mem0]] 关系:**AgentOps `instrumentation/providers/mem0/` 完整 instrument mem0**(对照 mem0 + OpenClaw 联动,**mem0 是 memory framework 协同 hub**)。
  [verified-from-source]

dataset: |
  - **Session-level dataset**:每个 session = 1 trace + N spans
  - **Replay**:`app.agentops.ai/sessions/<id>` 可视回放(README "Replay Analytics and Debugging")
  - **不提供内置 benchmark dataset**(对照 [[../eval/opik]] 8 内置 benchmark)
  - 用户场景:agent framework 用户的 production session 数据
  [verified-from-source]

scorer: |
  ### 12 SpanKind(归因维度核心,`agentops/semconv/span_kinds.py:7-19`)

  ```python
  class AgentOpsSpanKindValues(Enum):
      WORKFLOW = "workflow"      # 工作流(顶层)
      SESSION = "session"        # 会话
      TASK = "task"              # 任务
      OPERATION = "operation"    # 操作
      AGENT = "agent"            # ⭐ Agent 实体(独家维度)
      TOOL = "tool"              # 工具调用
      LLM = "llm"                # LLM 调用
      CHAIN = "chain"            # Chain(LangChain 风格)
      TEXT = "text"              # 纯文本
      GUARDRAIL = "guardrail"    # ⭐ 护栏(对照 [[../eval/opik]] Guardrails 独立 service)
      HTTP = "http"              # HTTP 调用
      UNKNOWN = "unknown"        # 未知
  ```

  ### Legacy SpanKind 7 细分(归因独家,`span_kinds.py:23-43`)

  ```python
  # Agent action 3 子类(业内独家归因维度)
  AGENT_ACTION = "agent.action"       # ⭐ Agent performing an action
  AGENT_THINKING = "agent.thinking"   # ⭐ Agent reasoning/planning
  AGENT_DECISION = "agent.decision"   # ⭐ Agent making a decision

  # LLM interaction
  LLM_CALL = "llm.call"

  # Workflow
  WORKFLOW_STEP = "workflow.step"
  ```

  **归因价值**:**`AGENT_THINKING` vs `AGENT_ACTION` vs `AGENT_DECISION` 三态区分** — 用户知道 agent "想错了"vs"做错了"vs"决策错了"

  对照 [[../eval/tau-bench]] `FaultAuthor × FaultType` typed enum:
  - tau-bench:fault 角度(谁错了)
  - **AgentOps:action 角度(在做什么)**
  - 互补,组合用更完整

  ### 没有内置 LLM-as-judge metric
  对照 [[../eval/ragas]] 27 / [[../eval/promptfoo]] 50 assertion — AgentOps 是 **observability 不是 eval framework**。
  [verified-from-source]

reproducibility: |
  - **OpenTelemetry first-class**(`agentops/instrumentation/OpenTelemetry.md` 完整 best practices 文档)
  - **`weakref.WeakKeyDictionary()`** 存 span context — 避免内存泄漏
  - **`TraceContext.is_init_trace` flag**(`sdk/core.py:43`)— 区分 framework 自动 vs 用户显式
  - **ClickHouse + OTel Collector + Grafana** 完整 self-host stack(`app/`)
  - **`compose.yaml` Docker compose** self-host
  - **`justfile`** 命令(类 Make,Rust 生态)
  - **uv 包管理**(`uv.lock`,Astral 生态,跟 DSPy / MemPalace / Promptfoo 一致)
  - **Cypress + Jest E2E**(`app/dashboard/`)
  [verified-from-source]

coverage: |
  ### 5 大功能(README)
  1. **📊 Replay Analytics and Debugging** — Step-by-step agent execution graphs
  2. **💸 LLM Cost Management** — Track spend
  3. **🤝 Framework Integrations** — Native 9 agentic framework
  4. **⚒️ Self-Host** — 完整 OSS stack
  5. **Session replays + Summary Analytics + Agent Debugging**

  ### 9 Agentic Framework Instrumentation(业内最广 agent framework 集成)
  - **ag2**(原 AutoGen)/ **agno** / **crewai** / **google_adk** / **haystack** / **langgraph** / **openai_agents** / **smolagents** / **xpander**

  ### 5 LLM Provider + 1 Memory Instrumentation
  - **anthropic** / **google_genai** / **ibm_watsonx_ai** / **openai** + **mem0**(memory framework instrument!)

  ### 4 核心装饰器(`agentops/sdk/decorators/`)
  - `@session` — Root span
  - `@agent` — Agent class span
  - `@operation` — Operation-level span
  - `@task` — Task-level span

  ### MCP Server(README Smithery badge)
  `@AgentOps-AI/agentops-mcp` — 让 Claude / Cursor 等 MCP client 一键查 AgentOps trace

  ### 不覆盖
  - **没有归因 agent**(对照 SkillForge ATTRIBUTION-AGENT V3)— AgentOps 提供 span 维度,**归因决策靠人在 Dashboard 看**
  - **没有 cross-session pattern 聚类**(对照 SkillForge V1 PROD-LABEL-CLUSTER)
  - **没有 A/B + canary**(对照 SkillForge V2 SKILL-CANARY-ROLLOUT)
  - **没有 prompt optimizer**(对照 [[../eval/opik]] 7 / [[../eval/dspy]] 15)
  - **没有 27+ RAG metric**(对照 [[../eval/ragas]])
  [verified-from-source]

production_loop: |
  ### Observability-first loop(README)
  ```
  Agent code with @session / @agent / @operation / @task 装饰器
       ↓
  OpenTelemetry SDK 自动 instrument(WeakKeyDictionary context propagation)
       ↓
  OTel Collector → ClickHouse(`app/clickhouse/`)
       ↓
  Dashboard(Next.js)+ Grafana 可视化
       ↓
  人工在 Dashboard 看 trace tree + replay session + 找 root cause
  ```

  **没有自动归因决策**(对照 SkillForge V3 ATTRIBUTION-AGENT)— **AgentOps 提供归因 raw materials**(12 SpanKind 维度细分 + replay UI),**归因决策靠人**。

  跟 wiki 已有归因 layer 对照:
  - **Layer A trace observability + 人工探索**(AgentOps 在此)
  - **Layer B per-evaluation rationale**(Phoenix Score.explanation 在此)
  - **Layer D fault typed enum**(tau-bench / DSPy 在此)
  - **Layer E production 自动归因 + 因果链**(SkillForge V3 独家)
  [verified-from-source]

automation: |
  - **2 行代码集成**:`agentops.init(API_KEY)` + `agentops.end_session('Success')`
  - **9 agentic framework auto-instrumentation**(用户不改代码)
  - **OTel Collector + Grafana** 标准 stack
  - **MCP Server**(Smithery)
  - **Cypress E2E** + **Jest** 测试自动化
  - **`@session / @agent / @operation / @task`** 4 装饰器最简(对照 mem0 `@track` 1 装饰器)
  [verified-from-source]

cost: |
  - **LLM Cost Management 一等公民**(README 第 2 大功能)
  - 跟 5 LLM provider instrumentation 集成自动 track spend
  - ClickHouse 列存 + OTel Collector 标准 stack 经济
  - **Cloud SaaS**(`app.agentops.ai`)+ **Self-host 免费**
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:AgentOps 是 **SkillForge ATTRIBUTION-AGENT 的 raw materials layer 参考**

  SkillForge ATTRIBUTION-AGENT V3 飞轮 [verified-from-source]:
  - **V80 完整 state machine**(proposal_pending → proposal_approved → candidate_created → ab_running → ab_passed → canary_started → promoted/rolled_back → verified)
  - **`surface_type`**(skill / prompt / behavior_rule)+ confidence + cooldown_expires_at
  - **OBS-2 / OBS-4 trace 模型**(`t_llm_trace` / `t_llm_span` + `root_trace_id`)

  AgentOps 走 **agent-level OTel observability + 12 SpanKind + 9 framework instrument** — **Layer A(trace 收集 + UI 归因)**,**没有 Layer C/D/E**(SkillForge 已超过这 3 层)。

  ## 可借鉴(给 SkillForge OBS / ATTRIBUTION-AGENT 加强)

  ### 高优(1-2 周)
  1. **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind**(`span_kinds.py:25-27`)— **业内独家 agent action 3 态归因**。SkillForge OBS-2 `t_llm_span.kind` 当前(`llm` / `tool` / `event`)可加 agent-level 细分(`agent.thinking` / `agent.action` / `agent.decision`)— **对 ATTRIBUTION-AGENT 归因输入更精细**(知道 agent 是想错 vs 做错 vs 决策错)
  2. **12 SpanKind 含 `GUARDRAIL`**(`span_kinds.py:17`)— SkillForge SafetyHook(N3 已交付)+ behavior_rule 可加 guardrail span type,**让 ATTRIBUTION-AGENT 能识别 guardrail 触发的失败**
  3. **`is_init_trace` flag**(`sdk/core.py:43`)— SkillForge OBS-4 trace 区分 framework 自动 vs 用户显式

  ### 中优(1 个月)
  4. **`@session / @agent / @operation / @task` 4 装饰器层级**(`sdk/decorators/__init__.py`)— SkillForge 当前 OBS-1 是 session × trace 合并视图,**可显式 task / operation 层级**(对照当前可能是隐式)
  5. **`weakref.WeakKeyDictionary()`** OTel context propagation(`OpenTelemetry.md`)— SkillForge Java 跟 Python 等价(`WeakHashMap`)
  6. **MCP Server**(Smithery)— SkillForge dashboard 可暴露 MCP server 让 IDE 一键查 trace(对照 [[../eval/phoenix]] / [[mempalace]] MCP 集成)

  ### 不必跟
  - **Cloud SaaS 商业模式**(SkillForge 不走 SaaS)
  - **9 agentic framework instrument**(SkillForge 自家 agent loop,不集成外部 framework)
  - **OTel Collector + Grafana stack**(SkillForge 自家 dashboard 已有,不需重做)
  - **完整 ClickHouse self-host**(SkillForge 当前 Postgres,加 ClickHouse 增运维,**> 10K traces/day 再考虑**)

  ## SkillForge 已领先
  - **V80 自动归因 state machine**(AgentOps 无)
  - **V1+V2+V3+V4 飞轮闭环**(AgentOps 无)
  - **多 surface 优化**(skill/prompt/behavior_rule,AgentOps 不做 optimization)
  - **MEMORY-LLM-SYNTHESIS + 人审 gate**(AgentOps 不做 memory)

  ## 关键决策建议

  **AgentOps 真正可借鉴 3 条**:
  1. **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind** — 给 ATTRIBUTION-AGENT 归因输入更精细(对照 tau-bench `FaultType` 4 类 + AgentOps `AGENT_*` 3 sub-kind,**两者组合用最完整**)
  2. **`GUARDRAIL` span 类型** — SkillForge SafetyHook + behavior_rule 加 guardrail span 标识
  3. **`@operation` / `@task` 显式层级** — OBS-1 session × trace 合并可加 task / operation 显式层级(对照当前可能隐式)

  **真正不该跟 AgentOps 走的路**(已领先,不要回退):
  - SkillForge ATTRIBUTION-AGENT 自动归因 → AgentOps 只 observability + 人工探索
  - SkillForge V1-V4 飞轮闭环 → AgentOps 没有 production loop

learning_takeaway: |
  ## 跟其他 observability 不同的是 (differentiator)
  1. **业内独家 agent-level OTel observability**(12 SpanKind 含 4 个 agent-level:WORKFLOW / SESSION / TASK / OPERATION / AGENT)
  2. **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind** — **业内独家 agent action 三态归因维度**
  3. **9 Agentic framework instrumentation**(ag2 / agno / crewai / google_adk / haystack / langgraph / openai_agents / smolagents / xpander)— **业内 agent framework 集成最广**
  4. **`mem0` instrumentation 集成**(`instrumentation/providers/mem0/`)— **mem0 + AgentOps 协同**(对照 mem0 + OpenClaw 联动,**mem0 是 memory framework 协同 hub**)
  5. **`is_init_trace` flag**(`sdk/core.py:43`)— 区分 framework 自动 vs 用户显式
  6. **`weakref.WeakKeyDictionary()`** OTel context propagation 工程模板(`OpenTelemetry.md`)
  7. **完整开源 stack**(SDK + Dashboard + API + ClickHouse + OTel Collector + Grafana + compose.yaml)
  8. **`@session / @agent / @operation / @task` 4 装饰器强制 agent 语义层级**(对照 mem0 / Phoenix 通用 trace 抽象)
  9. **`GUARDRAIL` 一等公民 SpanKind**(对照 [[../eval/opik]] Guardrails 独立 service)
  10. **MCP Server**(Smithery `@AgentOps-AI/agentops-mcp`)
  11. **`llms.txt` 顶层 LLM-readable doc**(类 robots.txt 但给 LLM)— 业内罕见
  12. **`justfile`**(Rust 生态)替 Makefile

  ## 优势 (strengths)
  1. **`agentops.init() + agentops.end_session()` 2 行代码集成** — dev 体验业内最简(对照 Helicone baseURL 切换 / mem0 `Memory.add`)
  2. **9 framework + 5 LLM + 1 memory(mem0) instrumentation 自动化** — 用户 0 代码改动
  3. **OpenTelemetry.md 详细工程文档**(WeakKeyDictionary + explicit parent context + _get_parent_context helper + format_trace_id debug)— 业内 OTel 集成模板
  4. **`TraceContext` class 含 `__enter__ / __exit__`** + 自动 status 设置 + `is_init_trace` flag — Python with-statement 工程化
  5. **`AuthenticatedOTLPExporter`**(`exporters.py`)+ `InternalSpanProcessor`(`processors.py`)— OTel pipeline 工程化
  6. **Cypress E2E + Jest** 完整 Dashboard 测试
  7. **`app/grafana/`** + `app/opentelemetry-collector/` 完整 OSS stack
  8. **MIT 完全开源**(SDK + App 都 MIT,对照 Phoenix ELv2 部分限制)
  9. **uv 包管理 + ruff**(`ruff.toml`)— Astral 现代 Python stack
  10. **`@AgentOps-AI/agentops-mcp` Smithery MCP server** — IDE 一键集成

  ## 哪些点做得好 (what_done_well)
  1. **12 SpanKind enum 含 GUARDRAIL** — 业内独家把 guardrail 作为 first-class span 类型(对照其他 observability framework 把 guardrail 隐藏在 metric / event 里)
  2. **Legacy SpanKind 3 sub-kind**(`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION`)保留 — **deprecated 但有归因价值**,体面 legacy 处理(对照 [[../eval/langchain]] `langchain_classic/`)
  3. **`AgentOpsSpanKindValues(Enum)` + `class SpanKind`** 双类(`span_kinds.py:7,22`)— Pythonic 旧版兼容
  4. **`mem0` instrumentation 4 文件**(`__init__.py + common.py + instrumentor.py + memory.py`)— mem0 instrument 工程化(对照 [[../source/mem0]] `mem0/openclaw/` 4 文件类似深度)
  5. **`OpenTelemetry.md` Best practices 文档** — 业内独家把 OTel 集成 best practice 写文档(对照 Phoenix `OTEL_TRACING.md` 类似)
  6. **`weakref.WeakKeyDictionary()`** OTel context 存储 — 避免内存泄漏,Python idiomatic
  7. **`TraceContext.__exit__` 自动 status 设置** — exception 时自动 ERROR,正常 OK
  8. **`@session / @agent / @operation / @task` 4 装饰器一致命名** — agent 语义层级清晰
  9. **`llms.txt`** 顶层 LLM-readable doc(类 robots.txt)— 业内独家 LLM-friendly project entry point
  10. **`justfile`** 替代 Makefile — Rust 生态新工具
---

# AgentOps(AgentOps-AI, MIT, 业内独家 agent-level observability)

## 一句话定位

**业内独家专门 agent-level observability platform**(MIT)。**12 SpanKind 含 4 个 agent-level**(WORKFLOW / SESSION / TASK / OPERATION / AGENT)+ **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind 业内独家归因维度** + **9 agentic framework + 5 LLM provider + mem0 instrumentation** + 完整开源 stack(SDK + Dashboard + ClickHouse + OTel Collector + Grafana)。

## 跟 SkillForge 归因 layer 对照

```
Layer E: Production 自动归因 + 因果链 actionable
  → SkillForge V3 ATTRIBUTION-AGENT(独家)

Layer D: Eval-output / fault 归因(typed enum)
  → tau-bench FaultAuthor × FaultType

Layer C: Memory / KG fact 归因(provenance)
  → Graphiti Episodes + Validity windows

Layer B: Per-evaluation LLM rationale
  → Phoenix Score.explanation

Layer A: Trace observability + 人工探索归因
  → AgentOps(agent-level)← 你在这层
  → Phoenix / Langfuse / Opik(LLM-level)
  → Helicone(Gateway-first)
```

## 3 大 SkillForge actionable 借鉴

1. **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind**(`span_kinds.py:25-27`)— SkillForge OBS-2 `t_llm_span.kind` 加 agent-level 细分,**对 ATTRIBUTION-AGENT 归因输入更精细**
2. **`GUARDRAIL` 一等公民 SpanKind**(`span_kinds.py:17`)— SkillForge SafetyHook + behavior_rule 加 guardrail span 标识
3. **`@operation` / `@task` 显式层级**(`sdk/decorators/__init__.py`)— OBS-1 session × trace 合并加 task / operation 显式层级

## 重大发现:mem0 是 memory framework 协同 hub

**`agentops/instrumentation/providers/mem0/`** 完整 instrument mem0(4 文件:`__init__.py + common.py + instrumentor.py + memory.py`)。

对照 [[../source/mem0]] 重大发现 1(`mem0/openclaw/dream-gate.ts`):
- mem0 + OpenClaw 联动
- **mem0 + AgentOps 联动**
- **mem0 是 LLM agent ecosystem memory framework 协同 hub**

## 待回答
- AgentOps Dashboard "Replay" 具体怎么实现(对照 [[../eval/langfuse]] session replay)
- `xpander` framework 实际是什么(小众 agent framework,wiki 未深查)
- AgentOps Cloud SaaS vs Self-host 功能差异
- `app/api/` Backend 是 Python 还是 Node(对照 README app/ Bun lock)
- AgentOps + mem0 instrumentation 具体抓什么 trace event(`memory.py` 实现)
