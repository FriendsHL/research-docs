---
type: source
name: langmem
origin: "/Users/youren/myspace/langmem/ (langchain-ai/langmem, license 待 verify)"
ingested_at: 2026-05-16
ingested_commit: 153db82
ingested_commit_date: 2026-05-11
ingested_version_label: "langmem 0.0.30(`pyproject.toml`)"
contributed_to:
  - eval/langmem.md
  - concept/agent-memory.md
---

# Source — LangMem

> **LangChain 团队出品的 memory utilities**(`langchain-ai/langmem`)。**最轻量(3.7MB clone size)** + **跟 LangGraph Long-term Memory Store native 集成**。**3 大组件**:Core Memory API + Memory Tools(hot path)+ Background Memory Manager。

## 仓库结构概览(最简洁)

```
langmem/
├── src/langmem/                     # ⭐ Python 主包
│   ├── __init__.py                  # ⭐ 公开 API(10 个 export)
│   ├── errors.py
│   ├── utils.py
│   ├── reflection.py                # ⭐ ReflectionExecutor(背景反思)
│   ├── graph_rag.py                 # GraphRAG
│   ├── graphs/                      # Graph 工具
│   ├── knowledge/                   # ⭐ Knowledge extraction + memory tools
│   │   ├── extraction.py            # ⭐ Memory extraction
│   │   ├── tools.py                 # ⭐ create_manage_memory_tool + create_search_memory_tool
│   │   └── __init__.py
│   ├── prompts/                     # ⭐ Prompt optimization 5 模块
│   │   ├── optimization.py          # ⭐ create_prompt_optimizer + create_multi_prompt_optimizer
│   │   ├── gradient.py              # Gradient-based optimizer
│   │   ├── metaprompt.py            # Meta-prompt optimizer
│   │   ├── stateful.py              # Stateful optimizer
│   │   ├── stateless.py             # Stateless optimizer
│   │   ├── _layers.py
│   │   ├── prompt.py
│   │   ├── types.py
│   │   └── utils.py
│   └── short_term/                  # ⭐ Short-term memory
│       ├── summarization.py         # 短期 memory 摘要
│       └── __init__.py
├── docs/                            # MkDocs
├── examples/                        # 用户示例
├── tests/
├── langgraph.json                   # ⭐ LangGraph deployment 配置
├── Makefile
├── pyproject.toml(version = "0.0.30")
├── pytest.ini
└── uv.lock
```

## 关键 file:line 锚点

### 公开 API(`src/langmem/__init__.py:1-20`)

```python
from langmem.knowledge import (
    create_manage_memory_tool,        # ⭐ Memory 管理 tool(给 agent)
    create_memory_manager,             # Memory manager(高阶)
    create_memory_searcher,            # Memory searcher
    create_memory_store_manager,       # Memory store manager
    create_search_memory_tool,         # ⭐ Memory 搜索 tool(给 agent)
    create_thread_extractor,           # Thread extractor(从 conversation 提取)
)
from langmem.prompts.optimization import (
    Prompt,
    create_multi_prompt_optimizer,    # ⭐ Multi-prompt optimizer
    create_prompt_optimizer,          # ⭐ Prompt optimizer
)
from langmem.reflection import ReflectionExecutor  # ⭐ Background reflection
```

**10 个公开 API,3 大类**:
1. **Knowledge / Memory tools**(给 agent 用)
2. **Prompt optimization**(对照 [[../eval/dspy]] 15 / [[../eval/opik]] 7)
3. **Reflection**(background memory manager)

### Memory Tools(给 agent)`knowledge/tools.py`

`create_manage_memory_tool(...)` — `:25`(让 agent 主动 store memory)
`create_search_memory_tool(...)` — `:362`(让 agent 主动 search memory)
`_get_store(initial_store: BaseStore | None = None)` — `:489`(依赖 LangGraph BaseStore)
`_ensure_json_serializable(content: typing.Any)` — `:500`(JSON safety)
`_ToolWithRequired(StructuredTool)` — `:513`(自定 tool 类)
`_ensure_schema_contains_required(schema: dict)` — `:529`

**对照 [[letta]] Function-call memory ops**(`core_memory_append/replace + archival_memory_insert/search`)— **LangMem 同模式**:agent 通过 tool 主动操作 memory。

### Prompt Optimizer(`prompts/optimization.py`)

`create_prompt_optimizer(...)` 4 个重载(`:27 / :35 / :43 / :50`)— 不同 kind 的 optimizer:
- **gradient**(`prompts/gradient.py`)— Gradient-based
- **metaprompt**(`prompts/metaprompt.py`)— Meta-prompt
- **stateful**(`prompts/stateful.py`)— Stateful(有 history)
- **stateless**(`prompts/stateless.py`)— Stateless(无 history)

`MultiPromptOptimizer(... class) ` — `:272`(同时优化多个 prompt)
`create_multi_prompt_optimizer(...)` — `:471`

**4 种 optimizer kind 跟 [[../eval/dspy]] 15 算法 / [[../eval/opik]] 7 算法对比**:
- LangMem 4 种偏 "**应用场景分类**"(stateful / stateless / gradient / metaprompt)
- DSPy 15 / Opik 7 偏 "**算法实现分类**"(GEPA / MIPROv2 / FewShotBayesian 等)

### Reflection(`reflection.py`)

`MemoryItem(TypedDict)` — `:31`(reflection 产物)
`Executor(Protocol)` — `:53`(执行器接口)
`ReflectionExecutor(...)` — `:90, :102, :110` 3 个重载(可灵活配置)
`RemoteReflectionExecutor` — `:143`(远端执行)
`LocalReflectionExecutor` — `:254`(本地执行)
`PendingTask(NamedTuple)` — `:389`
`_process_queue` — `:401`(队列处理)

**Background reflection 机制**(README "Background memory manager"):
- 后台自动 extract + consolidate + update agent knowledge
- Remote vs Local 双执行器
- 跟 [[../harness/openclaw]] Memory Dreaming / [[../harness/hermes-agent]] `commit_memory_session` / [[../harness/claude-code]] `autoDream` 同方向(background memory consolidation)

### Short-term Memory(`short_term/summarization.py`)

短期 memory 摘要(对照 [[../concept/context-compaction]] 5 家路线)

### Knowledge Extraction(`knowledge/extraction.py`)

从对话提取 facts(对照 [[mem0]] single-pass LLM extraction / SkillForge MEMORY-LLM-SYNTHESIS)

### Quick Start(README)

```python
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore
from langmem import create_manage_memory_tool, create_search_memory_tool

store = InMemoryStore(index={"dims": 1536, "embed": "openai:text-embedding-3-small"})

agent = create_react_agent(
    "anthropic:claude-3-5-sonnet-latest",
    tools=[
        create_manage_memory_tool(namespace=("memories",)),
        create_search_memory_tool(namespace=("memories",)),
    ],
    store=store,
)
```

**`namespace=("memories",)`** — Tuple-based namespace(类 OS 文件路径)用于 memory scope。

### LangGraph BaseStore 集成

`tools.py:489 _get_store(initial_store: BaseStore | None = None)` — 直接用 LangGraph **`BaseStore`** 接口,**LangGraph Platform 默认包含**(README "Native integration with LangGraph's Long-term Memory Store, available by default in all LangGraph Platform deployments")。

**Production storage**:`AsyncPostgresStore`(LangGraph 提供)

### `langgraph.json` Deployment 配置

LangGraph Platform deployment 配置 — 跟 LangChain ecosystem 深度集成。

## 与 wiki 现有

### 印证
- **3 大组件结构**(Core API + Memory tools + Background reflection)跟 [[../harness/openclaw]] Memory Dreaming + [[../harness/hermes-agent]] `commit_memory_session` + [[mem0]] Memory.add 3 mode 一致行业共识
- **Prompt optimizer 4 种**(gradient / metaprompt / stateful / stateless)跟 [[../eval/dspy]] 15 / [[../eval/opik]] 7 算法路线
- **LangGraph BaseStore + AsyncPostgresStore** 跟 LangChain ecosystem 深度集成(对照 [[../eval/langchain]] 5 大产品)
- **`namespace=("memories",)`** 跟 [[mem0]] user_id/agent_id/run_id 三层 scope 类似但更通用(tuple-based)

### 矛盾
- **LangMem 0.0.30 很早期**(对照 [[mem0]] 2.0.2 / [[letta]] 0.16.8 / [[mempalace]] 成熟)— LangMem 是 LangChain ecosystem 的 memory 补丁,**不是独立 production memory framework**
- **LangMem 不发 LongMemEval / LoCoMo benchmark 分数**(README 无)— 对照 mem0 94.8 / MemPalace 96.6 / Letta MemGPT paper baseline

### 新增
- **`eval/langmem.md`**(新建)
- **扩 [[../concept/agent-memory.md]]**(待新建):LangMem "**LangGraph BaseStore 集成 + memory tools + background reflection + prompt optimization 一体**" 路线 — 跟其他 framework 不同的是 **LangMem 跟 LangChain ecosystem 深度绑定**

## 事实漂移 / 注意点

- **0.0.30 版本**:很早期(对照 mem0 v2.0.2 已成熟)
- **License 待 verify**(本仓库 LICENSE 文件未读,推测 MIT 跟 LangChain 一致)
- **LangMem 是 LangChain ecosystem 一部分,不是独立产品**:用户如果不在 LangGraph + LangChain ecosystem 内,LangMem 价值减半
- **Prompt optimizer 4 种 跟 [[../eval/dspy]] 15 算法关系**:LangMem 较少(4)+ 偏应用分类,DSPy / Opik 是算法分类
- **Background reflection 是 LangMem 独特**(对照 mem0 v2 single-pass ADD-only 同步路线)— **后台 reflection vs 同步 ADD 是两种工程选择**
- **`graph_rag.py`** 顶层文件 — GraphRAG 直接集成,跟 [[mempalace]] Temporal KG / [[zep+graphiti]] Graphiti 同方向
- **`langgraph.json`** — Deployment 配置文件,跟 LangGraph Platform 集成
- **`namespace=("memories",)`** tuple-based scope — 比 mem0 user_id/agent_id/run_id 更通用(任意深度 tuple)
- **`AsyncPostgresStore`** Production storage 推荐(对照 mem0 SQLite history + vector store)
