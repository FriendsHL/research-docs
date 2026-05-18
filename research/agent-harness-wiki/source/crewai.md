---
name: crewai
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: crewai
version: 1.14.5a6
commit: a95d26763f4766b1a4f7c19c039133d1202dbdaa
license: MIT
stars: 30k+
last_ingested_at: 2026-05-17
contributed_to:
  - "[[harness/crewai]]"
  - "[[concept/agent-team-patterns]]"
---

# Source — CrewAI v1.14.5a6 @ commit a95d2676

> joaomdmoura/crewAI — **业内最 production-ready agent team framework**(30k+ stars,MIT)。**Crew = team of agents** 抽象 + `Process` enum(sequential / hierarchical)+ Task DAG + 5-package monorepo(cli / crewai / crewai-core / crewai-files / crewai-tools)。clone @ `/Users/youren/myspace/crewai/` 354 MB。

---

## 1. 仓库结构 — 5 子包 monorepo

```
crewai/
├── lib/
│   ├── cli/             # crewai-cli (命令行)
│   ├── crewai/          # ⭐ 主包 (本次 ingest 重点)
│   ├── crewai-core/     # core abstractions
│   ├── crewai-files/    # 文件相关 utility
│   ├── crewai-tools/    # 工具集成
│   └── devtools/        # 开发工具
├── docs/
├── pyproject.toml + uv.lock
└── README.md
```

**5-package modular monorepo**:cli / crewai / crewai-core / crewai-files / crewai-tools — 比 LlamaIndex 23 类少但比 Dify monolith 多,中等粒度。

---

## 2. `lib/crewai/src/crewai/` — 26 子模块 [verified]

```
crewai/
├── a2a/                 # ⭐ Google A2A protocol 集成
├── agent/               # agent 抽象 (core.py + planning_config + internal/)
├── agents/              # agent executor + parser + planner_observer + step_executor + tools_handler
├── auth/                # 认证
├── cli/                 # 命令行
├── constants.py
├── context.py
├── core/
├── crew.py              # ⭐ Crew class (2305 行,核心)
├── crews/               # 多 crew 协调
├── events/              # event system
├── experimental/        # 实验功能
├── flow/                # ⭐ Flow (event-driven workflow,跟 Crew 并行)
├── hooks/
├── knowledge/           # knowledge base
├── lite_agent_output.py / lite_agent.py   # 轻量 agent
├── llm.py / llms/       # LLM 抽象
├── mcp/                 # MCP 支持
├── memory/              # ⭐ memory (unified_memory + encoding/recall_flow + memory_scope + storage + analyze)
├── plus_api.py          # CrewAI Plus SaaS
├── process.py           # ⭐ Process enum (sequential/hierarchical/consensual)
├── project/             # project scaffolding
├── rag/                 # RAG
├── security/            # security_config
├── settings.py
├── task.py              # ⭐ Task class (1468 行)
└── mypy.py              # mypy plugin
```

---

## 3. 核心抽象 — Crew + Task + Process + Agent

### 3.1 `Process` enum [code: process.py:1-11]

```python
class Process(str, Enum):
    """Class representing the different processes that can be used to tackle tasks"""
    sequential = "sequential"
    hierarchical = "hierarchical"
    # TODO: consensual = 'consensual'
```

**3 个流派 + 1 个 TODO**:
- `sequential` — Task 按顺序串行,前任务 output 喂给后任务(默认)
- `hierarchical` — Manager agent(`manager_llm` 或 `manager_agent`)分配 task 给 worker agents,可并行
- `consensual` — TODO 未实现(consensus 协议,可能后续支持)

### 3.2 `Crew` class 核心字段 [code: crew.py:159-300+]

```python
class Crew(FlowTrackable, BaseModel):
    """Represents a group of agents, defining how they should collaborate
    and the tasks they should perform."""
    
    # Core
    name: str | None = "crew"
    agents: list[BaseAgent] = []                  # ⭐ team members
    tasks: list[Task] = []                         # ⭐ task list
    process: Process = Process.sequential          # ⭐ 协作模式
    
    # Hierarchical mode
    manager_llm: str | BaseLLM | None = None       # ⭐ manager LLM
    manager_agent: BaseAgent | None = None          # ⭐ 自定义 manager agent
    
    # Capabilities
    memory: bool | Memory | MemoryScope | MemorySlice | None = False
    embedder: EmbedderConfig | None = None
    cache: bool = True
    function_calling_llm: str | LLM | None = None
    planning: bool = ...                            # 是否生成 plan
    
    # Callbacks
    step_callback: SerializableCallable | None = None
    task_callback: SerializableCallable | None = None
    before_kickoff_callbacks: list[SerializableCallable] = []
    after_kickoff_callbacks: list[SerializableCallable] = []
    
    # Output / streaming
    stream: bool = False
    chat_llm: BaseLLM | None = None
    
    # Telemetry / config
    usage_metrics: UsageMetrics | None = None
    max_rpm: int | None = None                      # ⭐ rate limiting
    share_crew: bool | None = False                 # 是否上报 crewai SaaS
    security_config: SecurityConfig | None = None   # fingerprinting
    config: dict | Json[dict] | None = None
```

**关键决策点**:
- `manager_llm` vs `manager_agent` — 简单 hierarchical 给个 LLM 就行,复杂场景给完整 agent
- `embedder` — 跟 memory 强耦合,memory enable 时必须配
- `max_rpm` — 全 crew rate limit(防 LLM API 限流)
- `step_callback` / `task_callback` — 2 个 lifecycle hook 粒度

### 3.3 `Task` class 核心字段 [code: task.py:113-220+]

```python
class Task(BaseModel):
    # Core
    description: str                                 # ⭐ task 描述
    expected_output: str                              # ⭐ 期望 output 形态
    agent: BaseAgent | None = None                    # ⭐ 谁执行
    
    # Context
    context: list[Task] | None = ...                  # ⭐ 前置 task output 当 context
    
    # Async
    async_execution: bool | None = False              # 异步执行
    
    # Structured output (3 种)
    output_json: type[BaseModel] | None = None        # JSON
    output_pydantic: type[BaseModel] | None = None    # pydantic instance
    response_model: type[BaseModel] | None = None     # ⭐ native provider structured output
    
    # File output
    output_file: str | None = None
    create_directory: bool | None = True
    
    # Tools (task-level 限制)
    tools: list[BaseTool] | None = []
    
    # Callbacks
    callback: SerializableCallable | None = None
    
    # Tracking
    used_tools: int = 0
    tools_errors: int = 0
    delegations: int = 0                              # ⭐ 委派次数(hierarchical 用)
    
    # CrewAI trigger payload control
    allow_crewai_trigger_context: bool | None = None   # 控制 trigger payload injection
                                                       # None: 只 inject first task
                                                       # True: 总 inject
                                                       # False: 永不 inject
    
    # Output
    output: TaskOutput | None = None
```

**关键设计**:
- **Task DAG via `context: list[Task]`** — 不是 string template,是 _"前 task 实例作为 context"_(类型安全)
- **3 种 structured output**:`output_json`(老式)+ `output_pydantic`(pydantic 实例)+ `response_model`(native provider feature,如 OpenAI structured outputs)
- **Task-level tool 限制**:`tools: list[BaseTool]` 这个 task 只能用这些 tool(防 hierarchical mode 下 worker 越权)
- **`delegations: int`** 跟踪委派次数 — hierarchical 模式 manager 把 task delegate 给 worker 时累加,用于 audit

---

## 4. 关键 sub-module 深度

### 4.1 `flow/` — Event-driven workflow(跟 Crew 并行的另一抽象)

```
flow/
├── flow.py
├── flow_config.py / flow_context.py / flow_serializer.py
├── flow_trackable.py
├── flow_wrappers.py
├── async_feedback / human_feedback.py
├── constants.py
```

**Crew vs Flow**:
- **Crew** = team of agents 跑 tasks(static collaboration)
- **Flow** = event-driven workflow(`@start` / `@listen` / `@router` decorators)

两个是**正交抽象**。Crew 内可被 Flow 调用,Flow 内也可包含 Crew。

### 4.2 `memory/` — 8 模块完整 memory layer

```
memory/
├── unified_memory.py    # ⭐ 统一抽象
├── encoding_flow.py     # 写入 flow
├── recall_flow.py       # 读取 flow
├── memory_scope.py      # ⭐ scope 控制(short_term/long_term/entity)
├── storage/             # backend (Chroma/Qdrant/Postgres/...)
├── analyze.py           # 分析
└── types.py / utils.py
```

**3 维 memory scope**(via `MemoryScope`):short_term / long_term / entity — 跟 LangMem `namespace=tuple` 任意深度 + Letta `block` 类似但更结构化。

### 4.3 `a2a/` — Google A2A protocol 集成

```
a2a/
├── auth/ + config.py + errors.py
├── extensions/
├── task_helpers.py
├── templates.py
├── types.py
├── updates/
├── utils/
└── wrapper.py
```

CrewAI **2026 主动集成 Google A2A protocol** — cross-org agent 通信。跟 [[harness/agentscope]] `a2a/` 路径同思路。

### 4.4 `agents/` — Agent execution layer

```
agents/
├── crew_agent_executor.py       # ⭐ agent 实际执行
├── parser.py                     # output parsing
├── planner_observer.py           # planning 观察
├── step_executor.py              # step-level execution
├── tools_handler.py              # tool 调用 wrapping
├── agent_adapters / agent_builder / cache / constants
```

**Agent execution layer 跟 Agent core 分离** — Agent 是 entity definition,AgentExecutor 是 runtime。类似 [[harness/dify]] 5 agent runner 分离设计。

### 4.5 `mcp/` — MCP support(`__init__.py` + 实现)

支持 MCP 协议,跟 Anthropic / opencode 同思路。

---

## 5. 关键 verified 锚点

| 路径 | 主题 |
|---|---|
| `process.py:1-11` | 3 process enum(sequential / hierarchical / consensual TODO) |
| `crew.py:159-300+` | Crew class 完整定义(2305 行总)|
| `task.py:113-220+` | Task class 完整定义(1468 行总)|
| `agents/crew_agent_executor.py` | agent 执行 runtime |
| `agents/tools_handler.py` | tool 调用 wrapping |
| `agents/planner_observer.py` | planning 观察 |
| `memory/unified_memory.py + memory_scope.py` | 3 维 memory scope |
| `memory/encoding_flow.py + recall_flow.py` | 读写分离 |
| `flow/flow.py + flow_wrappers.py + human_feedback.py` | Event-driven workflow + human-in-the-loop |
| `a2a/wrapper.py + types.py` | Google A2A protocol |
| `mcp/` | MCP 支持 |
| `plus_api.py` | CrewAI Plus SaaS API |
| `share_crew + security_config` | telemetry + security fingerprinting |

---

## 6. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/crewai/](file:///Users/youren/myspace/crewai/) 354 MB
- GitHub: `joaomdmoura/crewAI` 30k+ stars / MIT
- 文档:docs.crewai.com
- 衍生 wiki:[[harness/crewai]] / [[concept/agent-team-patterns]]
