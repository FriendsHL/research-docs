---
name: cognee
type: eval
status: active
language: python
last_ingested_version: 1.0.9
last_ingested_at: 2026-05-17
last_commit: 4ca1d0c2bbbb46924acb1f5f6cd805214805ca16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/cognee]]"
  - "/Users/youren/myspace/cognee/"

agent_loop: |
  Cognee 不是 agent runtime. 是 "memory control plane" library + MCP server, 
  agent loop 由用户的 agent framework (LangChain/LangGraph/AutoGen/Anthropic) 提供.
  `cognee/modules/agent_memory/` 提供 decorator-based memory API (decorator.py + runtime.py + sanitization.py)
  让用户 agent 通过 @decorator 接 Cognee memory.

tool_protocol: |
  `cognee-mcp/` MCP server 把 Cognee 作为 MCP tool 暴露 (SSE / HTTP / stdio 3 协议).
  `cognee/skill.md` 业内独家 — 仓库内放 Anthropic Skill 标准文件,
  agent 可直接 install + call.

context_mgmt: |
  Cognee 不做 in-context window 压缩. context 走 KG retrieval (22 retriever 各种 mode).
  对照: 长 context 通过 `graph_completion_*_retriever` 5 种变体提取 KG context.

memory: |
  ⭐ **核心定位**. 25 modules 几乎都围绕 memory:
  - `modules/agent_memory/` decorator API
  - `modules/cognify/` 数据 → KG (核心动词 "cognify")
  - `modules/memify/` + `memify_pipelines/` 7 个 — feedback weight / frequency weight / entity dedup / triplet embedding / agent trace persist / session persist
  - `modules/recall/` + `modules/retrieval/` 22 retriever
  - `modules/ontology/` RDF/XML ontology grounding
  - `tasks/temporal_awareness/` + `tasks/temporal_graph/` 时间维度
  - `infrastructure/databases/unified+hybrid/` graph + vector + relational 统一抽象

multi_agent: |
  无. 用户 agent framework 提供.

eval_loop: |
  ⭐ **内置 eval_framework** (`cognee/eval_framework/`):
  - run_beam_eval.py 跑 BEAM benchmark (跟 mem0/Letta 对齐)
  - corpus_builder / answer_generation / evaluation / benchmark_adapters 完整流程
  - Modal serverless 集成 (modal_eval_dashboard + modal_run_eval)
  - DeepEval evaluation engine 集成

self_evolution: |
  ⭐ **闭环 self-improving memory**:
  - `apply_feedback_weights.py` 用户反馈 → memory 权重
  - `persist_agent_trace_feedbacks_in_knowledge_graph.py` agent 跑出来的 trace + feedback 自动入 KG
  - `apply_frequency_weights.py` 频率权重
  - `consolidate_entity_descriptions.py` 实体描述合并 (dedup)
  这是业内最完整的 "feedback loop → memory evolution" pipeline.

observability: |
  `modules/observability/` 内置. Trace + metrics + dashboard.
  Modal serverless 集成 deploy eval dashboard.

hooks: |
  Pipeline-level hooks (modules/pipelines/).
  Session lifecycle (modules/session_lifecycle/).

safety: |
  `cognee/modules/agent_memory/sanitization.py` — memory 写入前 sanitize.
  `cognee/modules/users/` 含用户 + 权限模型 (multi-tenant memory isolation).

provider: |
  `cognee/infrastructure/llm/` — `LLMGateway.py` + `extraction/` + `structured_output_framework/` + `prompts/` + `tokenizer/`.
  支持 OpenAI / Anthropic / 等多 provider.

channels: |
  Python library (主) + FastAPI (api/) + CLI (cognee-cli) + MCP server (cognee-mcp/) + Next.js UI (cognee-frontend/).
  4 channel 覆盖.

persistence: |
  `infrastructure/databases/` 8 类:cache / graph (Kuzu+Neo4j+...) / hybrid / relational / unified / vector.
  Alembic migrations. Kuzu graph DB 子目录独立.

cancellation: |
  ? 未深 verify.

skillforge_takeaway: |
  Cognee 跟 SkillForge memory **路线最近** — 都做 agent memory + KG + feedback loop.
  这是 6 个 RAG ingest 里跟 SkillForge memory v3 决策最直接相关的 framework.

  🔴 **R-COG-1 — Agent Trace + Feedback 自动入 KG**(架构启示, 直接打到 SkillForge)
     `cognee/memify_pipelines/persist_agent_trace_feedbacks_in_knowledge_graph.py` 
     把 agent 跑出来的 trace + 用户 feedback **自动持久化进 KG**.
     SkillForge 现状: 有 `t_session` + `t_message` + `LlmTraceEntity` (observability),
     但 trace 没自动进 memory pipeline; MEMORY-LLM-SYNTHESIS 是 LLM 提议 + 人审 gate, 
     **用户 feedback 信号没显式 routed 进 memory pipeline**.
     这是 SkillForge ATTRIBUTION-AGENT v3 + MEMORY v3 协同的关键缺口.
     工作量: 1-2 周 (加 V?? `t_memory_proposal_from_trace_feedback` + pipeline).

  🔴 **R-COG-2 — Decorator-based Agent Memory API**(集成抽象)
     `cognee/modules/agent_memory/{decorator,runtime,sanitization}.py` —
     Python `@cognify` 风 decorator 让用户 agent 函数自动有 memory.
     SkillForge skill 当前是 prompt + tool list, **skill 内的 LLM call 不自动有 memory**.
     如果 SkillForge skill 加 "memory-aware" 抽象 (@with_memory(scope='session') 等),
     Cognee decorator 模式是 reference.
     工作量: 1 周.

  🟡 **R-COG-3 — 22 Retriever 显式建模**(参考)
     `cognee/modules/retrieval/` 22 种 retriever, 5 种 graph_completion 变体 
     (基础 / 上下文扩展 / CoT / 分解 / 摘要).
     SkillForge MemoryService 当前 hybrid FTS+Vector+RRF 单一路径, 远期可参考显式建模.
     工作量: 远期参考.

  🟡 **R-COG-4 — Unified + Hybrid 数据库抽象**(架构)
     `cognee/infrastructure/databases/{unified,hybrid}/` 让 graph + vector + relational 一致接口.
     SkillForge 当前 PostgreSQL + 向量 plugin (pgvector?), 没有显式 unified 抽象.
     工作量: 远期, 看 SkillForge 是否需要多 backend.

  🟡 **R-COG-5 — RDF/XML Ontology Resolver**(KG memory 路线深度)
     `cognee/modules/ontology/` 独立 ontology 模块 (业内罕见).
     SkillForge 如果走 KG memory v3, ontology grounding 是必备 (防 entity drift).
     工作量: 远期, 只在 KG 路径上需要.

  🟢 **R-COG-6 — `skill.md` 自带 Anthropic Skill wrapper**(参考)
     Cognee 在仓库内放 `cognee/skill.md` (Anthropic Skill 标准格式) — Cognee 可被 Claude/agent 直接 install + call.
     SkillForge skill marketplace 可参考 "项目自带 skill.md" 的发布模式
     (让用户的项目自带 SkillForge 兼容 skill wrapper).

  🟢 **R-COG-7 — 内置 BEAM benchmark 跑通流**(eval 参考)
     `cognee/eval_framework/run_beam_eval.py` 跑通 BEAM (跟 mem0 64.1% 数据对齐).
     SkillForge V85 EVAL-DYNAMIC-USER-SIM 是自家 eval, 商业化需要外部 benchmark 对照.
     工作量: 1-2 周 BEAM 接 SkillForge eval framework.

  **❌ 不该跟**:
  - 全栈 Cognee MCP + frontend + 8 DB backend — SkillForge 已有自家 stack
  - "memify / cognify" 自创动词 — 概念清晰, 不需要 rebrand
---

# Eval — Cognee 1.0.9 (Memory Control Plane for AI Agents)

> Cognee 是 **"Brain behind your Agents"** 自我定位的 memory framework(Apache 2.0,topoteretes/cognee)。跟 SkillForge memory 路线**最直接对照** — 都做 agent memory + KG + feedback loop + self-improving。本次 ingest 13 维分析 + R-COG-1~R-COG-7 借鉴清单。

---

## 1. 项目定位(一句话)

**Memory control plane for AI Agents** — 把"用户文档 + agent trace + 用户 feedback"统一变成可检索 + 可演化的 KG memory,通过 22 retriever 让 agent 取用,通过 7 memify pipeline 让 memory 自我改进。

---

## 2. 为什么 Cognee 对 SkillForge 特别重要

SkillForge memory 当前路线(verified from [[concept/agent-memory]]):
- ✅ MemoryEntity 三态(observation/reflection/optimized)
- ✅ Hybrid FTS+Vector+RRF retrieval(`MemoryService.java:853-908`)
- ✅ LLM-SYNTHESIS proposal + 人审 gate
- ✅ DREAM-CONSOLIDATION dedup + archive(V66+)
- ❌ **Agent trace → memory 自动 pipeline**
- ❌ **User feedback → memory weight 自动 pipeline**
- ❌ **多种 retrieval pattern 显式建模**
- ❌ **Decorator-based skill-internal memory API**

**Cognee 正好补上 4 个 ❌ 中的前 2 个**(R-COG-1 + R-COG-2,共 2-3 周工作量)。这两条直接打到 SkillForge ATTRIBUTION-AGENT v3 + MEMORY v3 协同。

---

## 3. 核心创新 — "Memify Pipelines" 7 个工具流

```
┌──────────────────────────────────────────────────┐
│  User document → Cognify → KG (entity+relationship│
│                            +community+ontology)   │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
              ┌──────────────┐
              │ Agent uses   │  ← 22 retriever (graph_completion / cypher / temporal / triplet / ...)
              │ memory in    │
              │ task         │
              └──────┬───────┘
                     │
                     │ trace + user feedback
                     ▼
┌──────────────────────────────────────────────────┐
│  Memify Pipelines (cognee/memify_pipelines/):     │
│   1. apply_feedback_weights                       │  ← 反馈 → memory 权重调整
│   2. apply_frequency_weights                       │  ← 使用频率 → 权重
│   3. consolidate_entity_descriptions               │  ← 实体描述合并 (dedup)
│   4. create_triplet_embeddings                     │  ← Triplet (S-P-O) embedding
│   5. memify_default_tasks                          │  ← 默认 task 流
│   6. persist_agent_trace_feedbacks_in_KG  ⭐       │  ← agent trace + feedback → KG
│   7. persist_sessions_in_KG               ⭐       │  ← session → KG
└──────────────────────────────────────────────────┘
                     │
                     ▼
              Memory 自我演化 + 越用越好
```

**核心闭环**:_user data → KG → agent retrieval → trace + feedback → memify pipeline → KG improvement_。这是业内最完整的"self-improving memory"工程化实现。

---

## 4. 22 Retriever 显式建模(对照 LlamaIndex 18 / MS GraphRAG 4)

详见 [[source/cognee]] §5。亮点:**5 种 graph_completion 变体**:
- `graph_completion_retriever.py` — 基础 KG 补全
- `graph_completion_context_extension_retriever.py` — 邻域上下文扩展
- `graph_completion_cot_retriever.py` — CoT 引导
- `graph_completion_decomposition_retriever.py` — 子问题分解
- `graph_summary_completion_retriever.py` — 摘要级

加上 `cypher_search_retriever.py`(Neo4j Cypher 原生)+ `temporal_retriever.py`(时间维度)+ `triplet_retriever.py`(SPO 三元组)+ `natural_language_retriever.py` 等,**22 个 retriever 在 RAG composition 上比 LlamaIndex 更细分**。

---

## 5. 跟其他 memory framework 对照

| | Cognee | mem0 | Letta (MemGPT) | MemPalace | LangMem | Zep+Graphiti |
|---|---|---|---|---|---|---|
| **核心定位** | Memory control plane + KG | LLM-extracted facts ADD-only | MemGPT virtual context + function call | Verbatim hierarchical (Wings/Rooms/Drawers) | LangChain ecosystem reflection | Temporal KG |
| **KG 一等公民** | ✅(graph + ontology) | ⚠️ entity-level | ❌ block-based | ❌ doc-based | ❌ tool-based | ✅(temporal episode)|
| **Feedback loop** | ✅ memify 7 pipeline | partial | partial | ❌ | partial | partial |
| **Self-improving** | ✅(memify 闭环) | (LongMemEval 算法)| partial | partial | (reflection) | partial |
| **22 retriever** | ✅ | ❌(单 retrieval)| limited | ❌(MCP tools)| limited | hybrid 3 mode |
| **Ontology** | ✅ RDF/XML | ❌ | ❌ | ❌ | ❌ | partial(ontology) |
| **Temporal** | ✅(2 task category)| ❌ | partial | ❌ | ❌ | ✅(核心)|
| **Skill wrapper** | ✅(skill.md 业内独家)| ❌ | ❌ | ❌(29 MCP tool)| ❌ | ❌(2 MCP server)|

**结论**:Cognee 在 **breadth × depth** 维度都强 — 22 retriever + 7 memify + ontology + temporal + skill wrapper 全有,但**任何单一维度不一定最强**(temporal 不如 Graphiti / hierarchical 不如 MemPalace / extracted facts 不如 mem0)。

**对 SkillForge 启示**:Cognee 是 _"全能型 memory framework"_,跟 SkillForge _"全能型 platform"_ 路线契合。**R-COG-1 + R-COG-2 是 SkillForge 直接借鉴最高 ROI 的两条**。

---

## 6. R-COG 借鉴清单总览

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-COG-1** | **Agent trace + feedback 自动入 KG**(`persist_agent_trace_feedbacks_in_knowledge_graph`) | 1-2 周 | 🔴 高 — 直接打 ATTRIBUTION-AGENT v3 缺口 | 🔴 P1 |
| **R-COG-2** | **Decorator-based agent memory API**(`@cognify`)| 1 周 | 🔴 高 — skill 加 memory 抽象 | 🔴 P1 |
| **R-COG-3** | 22 Retriever 显式建模 | 远期 | 中 | 🟡 P2 |
| **R-COG-4** | Unified + Hybrid DB 抽象 | 远期 | 中 | 🟡 P2 |
| **R-COG-5** | RDF/XML Ontology Resolver | 远期(KG 路径)| 中 | 🟡 P2 |
| **R-COG-6** | `skill.md` 自带 Skill wrapper | < 半周 | 中 — marketplace 模式 | 🟢 P3 |
| **R-COG-7** | 内置 BEAM benchmark 跑通流 | 1-2 周 | 中 — eval 对照 | 🟢 P3 |

---

## 7. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[concept/agent-memory]] | **核心扩展** — Cognee 是 5 路线之外的 6th "Memory control plane + memify" 路线 | feedback-driven self-improving memory pipeline |
| [[eval/mem0]] / [[eval/letta]] / [[eval/mempalace]] / [[eval/langmem]] / [[eval/zep-graphiti]] | 互补 | 跟 5 个 memory framework 横向对照 |
| [[eval/llama_index]] | 互相集成 | `llama-index-graph-rag-cognee` 官方集成包印证关系 |
| [[source/cognee]] / [[concept/rag-design-space]] | 衍生 + 后续合并 | — |

---

## 8. 已知 post-ingest 风险

- 22 retriever 各自 implementation 只看模块名未深读
- 7 memify pipeline 内部算法未 verify (feedback_weights / frequency_weights 具体怎么算)
- MCP server (`cognee-mcp/`) 暴露的 tool 清单未 verify
- Ontology RDF/XML resolver 跟用户自定义 ontology 集成路径未 verify
- BEAM 跑出来的 Cognee 自家成绩未在 README 公布

---

## 9. 引用

- [[source/cognee]] 完整源码锚点
- GitHub: `topoteretes/cognee` Apache 2.0
- LlamaIndex 官方集成:`llama-index-graph-rag-cognee`
- Discord:r/AIMemory subreddit
- 跨页对照:[[concept/agent-memory]] 6 路线扩展 / [[eval/mem0]] / [[eval/letta]] / [[eval/mempalace]] / [[eval/langmem]] / [[eval/zep-graphiti]]
