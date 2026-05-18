---
type: eval-framework
name: langmem
status: active
language: python

# === 版本时效 ===
last_ingested_version: "langmem 0.0.30 @ commit 153db82 (2026-05-11)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/langmem]]"

# === 9 维度 ===
target: |
  **LangChain 团队出品的 memory utilities**(`langchain-ai/langmem`,0.0.30)— **跟 LangGraph Long-term Memory Store native 集成**,**LangChain ecosystem 的 memory 补丁**(而非独立 production framework)。

  **3 大组件**(README):
  1. **Core memory API**(works with any storage)
  2. **Memory management tools**(给 agent 在 hot path 用)
  3. **Background memory manager**(`ReflectionExecutor` 自动 extract/consolidate/update)

  跟其他 memory framework 对比:
  - [[mem0]] = pure memory framework(Single-pass ADD-only)
  - [[letta]] = MemGPT Agent OS(virtual context + function-call memory ops)
  - [[mempalace]] = Verbatim + hierarchical scope(零 API call)
  - **LangMem = LangChain ecosystem memory utilities**(深度绑定 LangGraph BaseStore)

  跟 [[../eval/langchain]] 关系:**LangMem 是 LangChain 5 大产品生态 + Deep Agents + LangGraph + LangSmith 的"memory 补丁"**,不在 LangChain 主仓库
  [verified-from-source]

dataset: |
  - 无内置 dataset(用户自己 conversation)
  - 通过 `create_thread_extractor`(`__init__.py:export`)从对话提取
  - 通过 `Memory tools` agent 主动 store
  - 通过 `ReflectionExecutor` background 自动提取
  - **Storage**:依赖 LangGraph `BaseStore`(`InMemoryStore` dev / `AsyncPostgresStore` production)
  [verified-from-source]

scorer: |
  ### 没有内置 benchmark / metric(README 无 LongMemEval / LoCoMo 分数)
  对照其他 memory framework:
  - [[mem0]] 公布 LongMemEval 94.8 / LoCoMo 91.6
  - [[mempalace]] 公布 LongMemEval R@5 96.6%
  - [[letta]] MemGPT paper baseline
  - **LangMem 不公布 benchmark**

  ### Memory Tools(agent-facing)
  - `create_manage_memory_tool(namespace=("memories",))` — 让 agent 主动 store
  - `create_search_memory_tool(namespace=("memories",))` — 让 agent 主动 search
  - `namespace=tuple-based scope`(类 OS 文件路径,任意深度)

  ### Background Memory Manager(`reflection.py`)
  - `ReflectionExecutor` 2 种(`:254 LocalReflectionExecutor` + `:143 RemoteReflectionExecutor`)
  - 自动 extract + consolidate + update memory
  - `MemoryItem`(`reflection.py:31 TypedDict`)+ `Executor`(`:53 Protocol`)+ `PendingTask`(`:389 NamedTuple`)
  [verified-from-source]

reproducibility: |
  - `pyproject.toml:version = "0.0.30"`
  - **`langgraph.json`** Deployment 配置(跟 LangGraph Platform 集成)
  - **`uv.lock`** 锁依赖
  - **依赖 LangGraph**(`langgraph.prebuilt.create_react_agent` + `langgraph.store.memory.InMemoryStore`)
  - 用户场景:LangChain + LangGraph ecosystem 内
  [verified-from-source]

coverage: |
  ### Public API(`__init__.py` 10 export)
  - **`create_manage_memory_tool`** + **`create_search_memory_tool`** — Memory tools(agent-facing)
  - **`create_memory_manager`** — Memory manager(高阶)
  - **`create_memory_searcher`** — Memory searcher
  - **`create_memory_store_manager`** — Memory store manager
  - **`create_thread_extractor`** — Thread extractor(对话提取)
  - **`Prompt`** + **`create_prompt_optimizer`** + **`create_multi_prompt_optimizer`** — Prompt optimization
  - **`ReflectionExecutor`** — Background reflection

  ### Prompt Optimization 4 种(`prompts/`)
  - **gradient**(`prompts/gradient.py`)
  - **metaprompt**(`prompts/metaprompt.py`)
  - **stateful**(`prompts/stateful.py`)
  - **stateless**(`prompts/stateless.py`)

  跟 [[../eval/dspy]] 15 算法 / [[../eval/opik]] 7 算法对照:**LangMem 4 种偏应用分类**,DSPy / Opik 偏算法分类

  ### Short-term Memory(`short_term/summarization.py`)
  短期 memory 摘要(对照 [[../concept/context-compaction]] 5 家路线)

  ### Graph RAG(`graph_rag.py`)
  顶层 single file — GraphRAG 直接集成(对照 [[mempalace]] Temporal KG / [[zep+graphiti]] Graphiti)

  ### 不覆盖
  - **没有 LongMemEval / LoCoMo benchmark 分数**(对照 mem0 / MemPalace)
  - **没有 Multi-Level Memory(user/session/agent)三层显式**(对照 mem0)
  - **没有 Wings/Rooms/Drawers hierarchical**(对照 MemPalace)
  - **没有 git 版本控制 memory**(对照 [[letta]] block_manager_git)
  - **没有 Read-only block**(对照 Letta READ_ONLY_BLOCK_EDIT_ERROR)
  - **没有 Inner Thoughts / Heartbeat**(对照 Letta MemGPT)
  - **没有 SaaS / dashboard / production loop**(对照 mem0 server + dashboard)
  - **依赖 LangChain + LangGraph ecosystem**(不能脱离)
  [verified-from-source]

production_loop: |
  ### Agent-driven loop(典型用法,README)
  ```python
  agent = create_react_agent(
      "anthropic:claude-3-5-sonnet-latest",
      tools=[
          create_manage_memory_tool(namespace=("memories",)),  # ⭐ Agent 主动 store
          create_search_memory_tool(namespace=("memories",)),  # ⭐ Agent 主动 search
      ],
      store=store,  # InMemoryStore dev / AsyncPostgresStore production
  )
  ```

  ### Background loop(ReflectionExecutor)
  ```
  Conversation → ReflectionExecutor.add(task)
       ↓
  Background queue(LocalReflectionExecutor 本地 / RemoteReflectionExecutor 远端)
       ↓
  Auto extract + consolidate + update memory store
       ↓
  Next agent.invoke → 自动看到 updated memory
  ```

  对照其他 framework:
  - **mem0 single-pass ADD-only**(同步 1 LLM call)
  - **LangMem Reflection**(异步 background)
  - **OpenClaw Memory Dreaming**(cron-based promotion)
  - **SkillForge MEMORY-LLM-SYNTHESIS + DREAM-CONSOLIDATION**(后台 + 人审 gate)

  **LangMem 走异步 background reflection 路线**
  [verified-from-source]

automation: |
  - **`langgraph.json` Deployment 配置** — LangGraph Platform native 集成
  - **`AsyncPostgresStore` production storage**(LangGraph 提供)
  - **`ReflectionExecutor` 双执行器**(Local + Remote)— 灵活部署
  - **`namespace=tuple` scope** — 任意深度 namespace
  - **No CLI / dashboard**(对照 mem0 dashboard / MemPalace `mempalace wake-up` CLI)
  [verified-from-source]

cost: |
  - **依赖 LangChain LLM provider**(`anthropic:claude-3-5-sonnet-latest` 等)
  - **`AsyncPostgresStore`** vector index dimensions(`dims=1536` for openai embeddings)
  - **Background reflection 异步**:不阻塞 hot path,但 LLM call 累积
  - **没有内置 cost tracking**(对照 [[helicone]] cost tracking 一等公民)
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:LangMem 是 **LangChain ecosystem 的 memory 补丁**,**对 SkillForge 借鉴价值最低**(4 memory framework 里)

  对照其他 framework 对 SkillForge 价值排序:
  - [[mem0]] **HIGH**:业内标杆 + Single-pass ADD-only + Entity linking + Multi-signal retrieval + **OpenClaw plugin 联动**
  - [[mempalace]] **HIGH**:LongMemEval 业内最高分 + Wings/Rooms/Drawers 三层 + 零 API call
  - [[letta]] **MEDIUM-HIGH**:MemGPT Agent OS + git 版本控制 memory block + READ_ONLY_BLOCK + Inner Thoughts
  - **[[langmem]] LOW-MEDIUM**:LangChain ecosystem 内 memory utilities,跟 SkillForge Java/Spring 栈不直接借鉴

  ## 可借鉴(给 SkillForge memory 工程细节)

  ### 高优(1-2 周)
  1. **`namespace=tuple` 任意深度 scope** — 对照 SkillForge memory 当前 scope 划分(可能 flat),改 tuple-based namespace 更通用
  2. **`ReflectionExecutor` 异步 Local + Remote 双执行器** — SkillForge MEMORY-LLM-SYNTHESIS 可加 remote 执行器(对照当前可能 Spring `@Async` 本地)
  3. **`create_manage_memory_tool + create_search_memory_tool` 双 tool** 让 agent 主动操作 memory(对照 [[letta]] function-call memory ops 同模式)— SkillForge 是否暴露 memory tool 给 agent 主动用?

  ### 中优(1 个月)
  4. **`MemoryItem TypedDict`**(`reflection.py:31`)— Pydantic typed memory item(对照 SkillForge MEMORY-LLM-SYNTHESIS 4 类 proposal schema)
  5. **`graph_rag.py` GraphRAG 集成** — SkillForge memory 加 graph 维度(对照 MemPalace Temporal KG / Graphiti)
  6. **`short_term/summarization.py` 短期 memory 摘要** — SkillForge 短期 memory(session-level)是否有独立 summarization 模块?

  ### 不必跟
  - **LangGraph BaseStore + AsyncPostgresStore 集成**(SkillForge Java 栈,LangGraph Python ecosystem 不直接用)
  - **`langgraph.json` deployment config**(SkillForge 不是 LangGraph deployment)
  - **`Prompt Optimizer` 4 种**(SkillForge 走自家 PromptImproverService 路线 + 借鉴 DSPy/Opik)
  - **LangMem 0.0.30 早期版本**(对照 mem0 v2.0.2 / MemPalace 成熟,LangMem 较新)

  ## SkillForge 已领先
  - **Production trace + 飞轮闭环**(LangMem 没有)
  - **MEMORY-LLM-SYNTHESIS + 人审 gate**(LangMem background reflection 无人审)
  - **MEMORY-DREAM-CONSOLIDATION embedding dedup + archived_reason**(LangMem 简单 store)
  - **multi-channel + multi-tenant**(LangMem 是 LangGraph 单 agent 内 memory)

  ## 关键决策建议

  **LangMem 借鉴价值低**,主要原因:
  - LangMem 跟 LangChain ecosystem 深度绑定,SkillForge Java/Spring 栈不能直接用
  - LangMem 0.0.30 早期(对照 mem0 / MemPalace 成熟)
  - LangMem 没有 benchmark 分数(对照 mem0 / MemPalace 公开 LongMemEval / LoCoMo)

  **真正可借鉴 3 条**:
  1. `namespace=tuple` 任意深度 scope(对照 SkillForge memory scope 划分)
  2. `ReflectionExecutor` Local + Remote 双执行器(SkillForge MEMORY-LLM-SYNTHESIS 异步分布式)
  3. `create_manage_memory_tool + create_search_memory_tool` agent-facing memory tools(对照 Letta function-call memory ops 同模式)

learning_takeaway: |
  ## 跟其他 memory framework 不同的是 (differentiator)
  1. **LangChain ecosystem 内 memory utilities**(`langchain-ai/langmem`,LangChain 5 大产品 + Deep Agents + LangGraph + LangSmith + LangMem)
  2. **跟 LangGraph BaseStore native 集成** — LangGraph Platform 默认包含
  3. **3 大组件结构清晰**:Core API + Memory tools + Background reflection
  4. **`namespace=tuple` 任意深度 scope** — 对照 [[mem0]] user/session/agent 3 层固定,LangMem 更通用
  5. **`ReflectionExecutor` Local + Remote 双执行器** — 异步 background 反思
  6. **Prompt optimization 一体内置**(4 种 kind:gradient / metaprompt / stateful / stateless)— **Memory + Prompt optimization 同 framework**(对照 [[mem0]] 只 memory,[[../eval/dspy]] 只 prompt optimization)
  7. **依赖 LangChain + LangGraph**(无法独立)
  8. **0.0.30 早期版本**(对照其他 memory framework 成熟)
  9. **`graph_rag.py` GraphRAG 内置** — 单文件实现(对照 MemPalace Temporal KG / Graphiti 独立 framework)
  10. **不公布 benchmark 分数**(对照 mem0 LongMemEval 94.8 / MemPalace 96.6)
  11. **`langgraph.json` Deployment-first** — 为 LangGraph Platform 部署设计

  ## 优势 (strengths)
  1. **跟 LangGraph 深度集成** — LangChain 生态用户零成本接入
  2. **3.7MB clone size** — 业内最轻 memory framework
  3. **Public API 10 export 清晰**(`__init__.py:1-20`)
  4. **`ReflectionExecutor` 异步 background** + Local / Remote 双执行器
  5. **`Prompt Optimizer` 4 种 kind 整合** — Memory + Prompt 一体
  6. **`namespace=tuple`** 通用 scope(类 OS 文件路径)
  7. **`AsyncPostgresStore` production-ready**(LangGraph 提供)
  8. **`uv.lock`** 依赖锁定
  9. **`MemoryItem TypedDict` schema-first**
  10. **`short_term/summarization.py`** 短期 memory 单独模块(concern 分离)

  ## 哪些点做得好 (what_done_well)
  1. **`__init__.py` 10 export** — 一目了然 public API
  2. **`namespace=("memories",)` tuple-based scope** — 比 mem0 user/session/agent 3 层更通用
  3. **`ReflectionExecutor` Protocol + 2 实现**(Local + Remote)— Strategy pattern 工程化
  4. **`Prompt Optimizer 4 种 kind`(gradient / metaprompt / stateful / stateless)按"是否需要 history" 分** — 工程分类清晰
  5. **`MemoryItem TypedDict + Executor Protocol + PendingTask NamedTuple`** — Python type-first 工程文化
  6. **`langgraph.json`** Deployment 配置文件 — LangGraph Platform 一等公民
  7. **`create_*_tool / create_*_manager / create_*_searcher` 一致命名** — factory function naming 一致
  8. **`graph_rag.py` 单文件实现** — 不堆 framework,简单概念简单实现
  9. **`short_term/` + `knowledge/` + `prompts/` + `reflection.py` 4 模块拆分** — short-term / long-term / prompt-opt / reflection 4 concern 分离
  10. **`uv.lock + pyproject.toml`** Astral uv 生态(跟 DSPy / Promptfoo 一致)
---

# LangMem(LangChain 团队,0.0.30)

## 一句话定位

**LangChain ecosystem 内 memory utilities**(`langchain-ai/langmem`)— **跟 LangGraph Long-term Memory Store native 集成**。**3 大组件**:Core memory API + Memory tools(给 agent)+ Background reflection executor。**LangChain 生态的 memory 补丁,不是独立 production framework**。

## 4 memory framework 对照

| Framework | 定位 | Benchmark(LongMemEval) | SkillForge 借鉴优先级 |
|---|---|---|---|
| **mem0 v2** | Pure memory framework + Single-pass ADD-only | 94.8 | **HIGH** |
| **MemPalace** | Verbatim + Wings/Rooms/Drawers + 零 API call | 96.6 raw / 98.4 hybrid | **HIGH** |
| **Letta(MemGPT)** | Agent OS + virtual context + function-call ops | (MemGPT paper baseline) | **MEDIUM-HIGH** |
| **LangMem** | LangChain ecosystem memory utilities | 不公布 | **LOW-MEDIUM** |

## 3 大组件(README)

1. **Core memory API** — `create_memory_manager / create_memory_searcher / create_memory_store_manager / create_thread_extractor`
2. **Memory tools(给 agent)** — `create_manage_memory_tool + create_search_memory_tool`(agent 主动操作 memory)
3. **Background memory manager** — `ReflectionExecutor`(Local + Remote 双执行器,异步 extract/consolidate/update)

## SkillForge 真正借鉴 3 条

1. **`namespace=tuple` 任意深度 scope** — 对照 SkillForge memory scope
2. **`ReflectionExecutor` Local + Remote 双执行器** — SkillForge MEMORY-LLM-SYNTHESIS 异步分布式
3. **`create_manage_memory_tool + create_search_memory_tool`** agent-facing memory tools — 对照 Letta function-call memory ops 同模式

**LangMem 借鉴价值偏低**(跟 LangChain ecosystem 深度绑定),**主要价值是了解 LangChain ecosystem 怎么做 memory**。

## 待回答
- License 具体(LICENSE 文件未读)
- Prompt optimizer 4 种 kind 跟 DSPy 15 算法关系
- `graph_rag.py` GraphRAG 实现细节
- LangMem 跟 LangGraph Long-term Memory Store 关系(LangGraph 主仓库还是 LangMem 独立?)
- ReflectionExecutor Remote 执行器协议
