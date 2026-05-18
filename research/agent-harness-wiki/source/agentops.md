---
type: source
name: agentops
origin: "/Users/youren/myspace/agentops/ (AgentOps-AI, MIT)"
ingested_at: 2026-05-16
ingested_commit: a855a92
ingested_commit_date: 2026-03-19
ingested_version_label: "agentops PyPI(`pyproject.toml`,以 commit 为准)"
contributed_to:
  - eval/agentops.md
  - concept/observability.md
  - concept/attribution.md
---

# Source — AgentOps

> **专门做 AI Agent observability + DevTool platform**(MIT,`AgentOps-AI/agentops`)。slogan: **"Observability and DevTool platform for AI Agents"**。**业内独家:专门 agent-level observability**(不是 LLM-level / trace-level / metric-level)。
>
> **完全开源**(README 明示 "AgentOps app is open source under the MIT license"):SDK + Dashboard + API + ClickHouse + OTel Collector + Grafana 全 stack。
>
> 集成 9 agentic framework + 5 LLM provider + **mem0 集成**(`instrumentation/providers/mem0/`)— AgentOps + mem0 协同 ecosystem。

## 仓库结构概览

```
agentops/
├── agentops/                          # ⭐ Python SDK 主包(PyPI: agentops)
│   ├── __init__.py
│   ├── client/                        # Client
│   ├── config.py
│   ├── enums.py
│   ├── exceptions.py
│   ├── helpers/                       # Dashboard helpers 等
│   ├── instrumentation/               # ⭐ instrumentation 系统
│   │   ├── OpenTelemetry.md           # ⭐ OTel 集成 best practices 文档
│   │   ├── README.md
│   │   ├── common/                    # 通用
│   │   ├── utilities/                 # 工具
│   │   ├── agentic/                   # ⭐ 9 agentic framework 集成
│   │   │   ├── ag2/                   # AG2(原 AutoGen)
│   │   │   ├── agno/                  # Agno
│   │   │   ├── crewai/                # CrewAI
│   │   │   ├── google_adk/            # Google ADK
│   │   │   ├── haystack/              # Haystack
│   │   │   ├── langgraph/             # LangGraph
│   │   │   ├── openai_agents/         # OpenAI Agents SDK
│   │   │   ├── smolagents/            # smolagents
│   │   │   └── xpander/               # xpander
│   │   └── providers/                 # ⭐ 5 LLM provider + 1 memory
│   │       ├── anthropic/
│   │       ├── google_genai/
│   │       ├── ibm_watsonx_ai/
│   │       ├── mem0/                  # ⭐⭐ mem0 集成(AgentOps + mem0 协同)
│   │       └── openai/
│   ├── integration/                   # 高阶集成
│   ├── legacy/                        # 旧版兼容
│   ├── logging/                       # 自家 logging
│   ├── sdk/                           # ⭐ 主 SDK
│   │   ├── attributes.py
│   │   ├── core.py                    # ⭐ TraceContext class + 完整 OTel 集成
│   │   ├── decorators/                # ⭐ @session / @agent / @operation / @task
│   │   │   ├── factory.py
│   │   │   ├── utility.py
│   │   │   └── __init__.py
│   │   ├── descriptors/
│   │   ├── exporters.py               # AuthenticatedOTLPExporter
│   │   ├── processors.py              # InternalSpanProcessor
│   │   ├── README.md
│   │   └── types.py
│   ├── semconv/                       # ⭐⭐ AgentOps semantic conventions(归因核心)
│   │   ├── README.md
│   │   ├── agent.py
│   │   ├── core.py
│   │   ├── enum.py
│   │   ├── instrumentation.py
│   │   ├── langchain.py
│   │   ├── message.py
│   │   ├── meters.py
│   │   ├── resource.py
│   │   ├── span_attributes.py
│   │   ├── span_kinds.py              # ⭐ 12 SpanKind(归因维度)
│   │   ├── status.py
│   │   ├── tool.py
│   │   └── workflow.py
│   └── validation.py
├── app/                               # ⭐ 完整 Dashboard + Backend stack
│   ├── api/                           # Backend API
│   ├── dashboard/                     # ⭐ Next.js Dashboard(Cypress E2E + Bun)
│   ├── clickhouse/                    # ⭐ ClickHouse schema + migrations
│   ├── deploy/                        # 部署
│   ├── e2e/                           # End-to-end 测试
│   ├── graphana/                      # ⭐ Grafana 集成
│   ├── landing/                       # 落地页
│   ├── opentelemetry-collector/       # ⭐ OTel Collector 配置
│   ├── compose.yaml                   # Docker compose self-host
│   ├── handbook/                      # Handbook
│   ├── justfile                       # justfile(类 make)
│   └── ...
├── docs/                              # 文档
├── examples/                          # 用户示例
├── tests/                             # 测试
├── llms.txt                           # ⭐ LLM-readable doc(类 robots.txt)
├── pyproject.toml
└── uv.lock                            # ⭐ uv 包管理(Astral 生态)
```

## 关键 file:line 锚点

### 12 SpanKind(归因核心维度,`agentops/semconv/span_kinds.py:7-19`)

```python
class AgentOpsSpanKindValues(Enum):
    WORKFLOW = "workflow"
    SESSION = "session"
    TASK = "task"
    OPERATION = "operation"
    AGENT = "agent"
    TOOL = "tool"
    LLM = "llm"
    CHAIN = "chain"
    TEXT = "text"
    GUARDRAIL = "guardrail"      # ⭐ Guardrail 一等公民 span 类型
    HTTP = "http"
    UNKNOWN = "unknown"
```

**对照其他 observability 项目**:
- Langfuse `observation_type` 主要 `GENERATION / SPAN / EVENT`
- Phoenix OpenInference semconv:`LLM / TOOL / RETRIEVER / CHAIN / AGENT / EMBEDDING / GUARDRAIL`
- **AgentOps 12 类**含 `WORKFLOW + SESSION + TASK + OPERATION + AGENT` 4 个 agent-level 维度(业内最细 agent 拆分)

### Legacy SpanKind 7 细分(`span_kinds.py:23-43`)

```python
class SpanKind:  # Legacy
    # Agent action kinds
    AGENT_ACTION = "agent.action"      # Agent performing an action
    AGENT_THINKING = "agent.thinking"  # Agent reasoning/planning
    AGENT_DECISION = "agent.decision"  # Agent making a decision

    # LLM interaction kinds
    LLM_CALL = "llm.call"              # LLM API call

    # Workflow kinds
    WORKFLOW_STEP = "workflow.step"    # Step in a workflow
    # ...
```

**业内独家**:**agent action 3 子类**(`AGENT_ACTION / AGENT_THINKING / AGENT_DECISION`)— 区分 agent 在做什么(行动 vs 思考 vs 决策),**对归因关键**(知道 agent 是"想错了"还是"做错了"还是"决策错了")。

### 4 个核心装饰器(README + `sdk/decorators/__init__.py`)

```python
from agentops.sdk.decorators import session, agent, operation, task

@session                # ⭐ Root span(整个 session)
def my_workflow():
    return result

@agent                  # ⭐ Agent class span
class MyAgent:
    def __init__(self, name):
        self.name = name

@operation              # ⭐ Operation-level span
def some_op():
    pass

@task                   # ⭐ Task-level span
def some_task():
    pass
```

**对照其他 framework**:
- mem0 `@track`(只 LLM 调用 trace)
- Phoenix `@trace`(通用 OTel)
- Langfuse `@observe()`(generation / span / event)
- Letta(无装饰器,靠 function-call memory ops)
- **AgentOps 4 装饰器是 agent 语义层**(session → agent → operation → task)

### `TraceContext` 主类(`agentops/sdk/core.py:40-56`)

```python
class TraceContext:
    def __init__(
        self,
        span: Span,
        token: Optional[context_api.Token] = None,
        is_init_trace: bool = False
    ):
        self.span = span
        self.token = token
        self.is_init_trace = is_init_trace  # ⭐ 标识 auto-started trace
        self._end_state = StatusCode.UNSET   # 默认未知 end state

    def __enter__(self) -> "TraceContext":
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        """Automatically sets the trace status based on whether an exception occurred"""
        # ...
```

**业内独家**:**`is_init_trace` flag** 区分 framework 自动启动的 trace vs 用户显式启动的(便于过滤)。

### OpenTelemetry 集成 best practices(`instrumentation/OpenTelemetry.md`)

业内独家把 OTel 集成写成完整工程文档,4 大核心 pattern:

1. **`weakref.WeakKeyDictionary()`** 存 span context — 避免内存泄漏
2. **Explicit parent context** — `trace.start_as_current_span(name, context=parent_context, kind, attributes)`
3. **`_get_parent_context(trace_obj)` helper** — Try trace_root_contexts first, fallback to current context
4. **`format_trace_id(span_context.trace_id)`** debug trace continuity

**对照 Langfuse / Opik / Phoenix 通用 OTel ingestion**:**AgentOps 工程化 OTel 集成模板最详细**(适合 agent framework 集成方做参考)。

### 9 Agentic Framework 集成(`instrumentation/agentic/`)

| Framework | 集成 | 备注 |
|---|---|---|
| **ag2** | ✅ | AG2(原 AutoGen)|
| **agno** | ✅ | Agno |
| **crewai** | ✅ | CrewAI |
| **google_adk** | ✅ | Google ADK(SkillForge 待 verify 集成的方向)|
| **haystack** | ✅ | Haystack |
| **langgraph** | ✅ | LangGraph |
| **openai_agents** | ✅ | OpenAI Agents SDK |
| **smolagents** | ✅ | smolagents |
| **xpander** | ✅ | xpander |

**业内 agent framework 集成最广**(对照 Langfuse 23+ 集成偏 LLM provider,AgentOps 9 个是 agent framework 专属)。

### 5 LLM Provider + 1 Memory(`instrumentation/providers/`)

```
anthropic / google_genai / ibm_watsonx_ai / mem0 / openai
```

**重大发现**:**`providers/mem0/` 文件夹存在**(`__init__.py + common.py + instrumentor.py + memory.py`)— **AgentOps + mem0 是协同 ecosystem**!

对照 [[../source/mem0]] 重大发现 1(`mem0/openclaw/dream-gate.ts`):
- **mem0 + OpenClaw 联动**(mem0 给 OpenClaw plugin)
- **mem0 + AgentOps 联动**(AgentOps instrument mem0)
- **mem0 是 memory framework 协同 hub**

### 完整 Self-host stack(`app/`)

| 组件 | 作用 |
|---|---|
| **`app/dashboard/`** | Next.js + Cypress + Bun + Jest Dashboard |
| **`app/api/`** | Backend API |
| **`app/clickhouse/`** | ClickHouse schema + migrations(对照 [[../eval/langfuse]] / [[../eval/opik]] / [[../eval/phoenix]] 都用 ClickHouse)|
| **`app/opentelemetry-collector/`** | OTel Collector 配置(业内 SDK 走 OTel collector 路线最完整)|
| **`app/grafana/`** | Grafana 集成(metric 可视化,对照 Phoenix Prometheus)|
| **`app/e2e/`** | E2E 测试 |
| **`app/compose.yaml`** | Docker compose self-host stack |
| **`app/justfile`** | `just` 命令(类 Make,Rust 生态)|

### Quick Start(README)

```python
import agentops

# 1. Init
agentops.init(<API_KEY>)

# 2. 跑 agent 代码(自动 instrumentation)

# 3. End session
agentops.end_session('Success')

# Dashboard:app.agentops.ai
```

**2 行代码集成**(类 [[../eval/helicone]] AI Gateway baseURL 切换 + Langfuse `@observe()`)。

### README "First class Developer Experience"(装饰器示例)

```python
# Session span(root for all other spans)
from agentops.sdk.decorators import session, agent, operation, task

@session
def my_workflow():
    return result

@agent
class MyAgent:
    pass

@operation
def some_op():
    pass

@task
def some_task():
    pass
```

### 5 大功能(README)

1. **📊 Replay Analytics and Debugging** — Step-by-step agent execution graphs
2. **💸 LLM Cost Management** — Track spend with LLM foundation model providers
3. **🤝 Framework Integrations** — Native CrewAI / AG2 / Agno / LangGraph / 等
4. **⚒️ Self-Host** — 完整 OSS stack
5. **Session replays** + **Summary Analytics** + **Agent Debugging**

### MCP Server

README 标 Smithery badge(`@AgentOps-AI/agentops-mcp` MCP server)— 对照 [[../eval/phoenix]] / [[helicone]] / [[mempalace]] MCP 集成业内主流。

### 商业模式

- **MIT 完全开源 SDK + App**
- **Cloud SaaS**(`app.agentops.ai`,商业版)
- **Self-host 完整 stack**(免费)
- 对照 Langfuse / Opik / Helicone 类似 OSS + Cloud 模式

### `llms.txt`(LLM-readable doc)

业内独家:**仓库顶层 `llms.txt`**(类 `robots.txt` 但给 LLM agent)— 让 Claude / GPT 等 agent 读项目时有标准 entry point。

## 与 wiki 现有

### 印证
- **ClickHouse 主分析库** 跟 Langfuse / Opik / Phoenix / Helicone 一致(业内事实标准)
- **OpenTelemetry first-class** 跟 Phoenix 一致(都是 OTel-based)
- **MIT + Cloud SaaS 商业模式** 跟 Langfuse / Helicone / Opik 一致 YC 路线
- **mem0 instrumentation** 印证 mem0 是 memory framework 协同 hub(跟 OpenClaw 联动 + AgentOps 联动)

### 矛盾
- **AgentOps 是 agent-level observability**(SpanKind 12 类含 4 个 agent-level)vs **Phoenix / Langfuse 是 LLM-level observability**(主要 GENERATION / SPAN / EVENT)— **不同抽象层级**
- **AgentOps 9 agentic framework instrumentation** vs Langfuse 23+ LLM framework 集成 — AgentOps 是 agent framework 偏好,Langfuse 是 LLM 框架偏好

### 新增
- **`eval/agentops.md`**(新建)— **agent-level observability** 专属
- **扩 [[../concept/observability]]**:加 AgentOps "agent-level OTel"路线 vs LLM-level / trace-level / wide-events 已有 3 路线
- **(可选)新建 [[../concept/attribution]]**:把归因深度对照写成独立 concept 页(对照 SkillForge ATTRIBUTION-AGENT + tau-bench + Phoenix Score.explanation + AgentOps SpanKind)

## 事实漂移 / 注意点

- **commit `a855a92` @ 2026-03-19** — 跟其他 ingest 项目稍旧(其他多 2026-05)但 stable
- **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 是 legacy**(`span_kinds.py:23-43`)— v2 用 `AGENT` 一类,**对归因细分仍可借鉴**(SkillForge 可加这 3 个 sub-kind)
- **AgentOps 12 SpanKind vs OpenInference 7 SpanKind** — AgentOps 多 `WORKFLOW / SESSION / TASK / OPERATION / TEXT / HTTP / UNKNOWN` 5 类
- **`@session / @agent / @operation / @task` 4 装饰器** vs Langfuse `@observe()` 1 装饰器 — AgentOps 强制 agent 语义层级
- **`is_init_trace` flag**(`core.py:43`)— 业内独家区分 framework 自动 vs 用户显式
- **`weakref.WeakKeyDictionary()`** OTel context 存储(`OpenTelemetry.md`)— 业内 OTel 集成工程模板
- **`xpander`** framework(`instrumentation/agentic/xpander/`)— 小众 agent framework,wiki 未 ingest 单独研究
- **`google_adk` instrumentation** — Google ADK 集成,对应 README 提及 "Google ADK" 是最新集成之一
- **mem0 instrumentation file 详情**:`__init__.py + common.py + instrumentor.py + memory.py` 4 文件,**AgentOps 给 mem0 全 instrument**
