---
name: llama_index
type: framework
status: active
language: python
last_ingested_version: llama-index-core 0.14.22
last_ingested_at: 2026-05-17
last_commit: 23bd65aaf79ffed587af35f325738b1580b35dea
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/llama_index]]"
  - "/Users/youren/myspace/llama_index/"

# 13 维 (framework 视角, 部分维度对 RAG framework 不强相关)

agent_loop: |
  agent/ 模块含 react + workflow 两路径.
  Workflow agent 是 v1 推荐 (workflow 已抽离独立 PyPI package).
  跟纯 agent runtime (Hermes/CC/Codex) 不同 — LlamaIndex agent 是 framework primitive,
  实际跑在用户 app 内, 不是独立 runtime.

tool_protocol: |
  `tools/` core + 68 tool integration.
  FunctionTool wraps Python function with auto JSON schema.
  Recent commit "add DocumentBlock and VideoBlock to FunctionTool._parse_tool_output (#21678)"
  显示 tool result 支持多模态 block.

context_mgmt: |
  ChatMemoryBuffer + ChatSummaryMemoryBuffer + VectorMemory + SimpleComposableMemory 4 种 + memory_blocks/ 模块化.
  Composable: 多种 memory 类型可叠加.

memory: |
  框架级 memory primitives, 跟专用 memory framework (mem0/Letta/MemPalace) 不同 —
  LlamaIndex memory 是 chat history 风, 不做 entity/reflection/summarization 长期 memory.

multi_agent: |
  `agent/workflow/` workflow-based agent + AgentMesh integration.
  没有 SubAgent / Coordinator / Fork 之类抽象 (vs Claude Code).
  Workflow 内嵌 agent 节点是 LlamaIndex 多 agent 模式.

eval_loop: |
  **13 内置 metric** (`core/evaluation/`):
    answer_relevancy / context_relevancy / correctness / faithfulness / guideline /
    pairwise / relevancy / semantic_similarity + retrieval/ + multi_modal/ + dataset_generation / batch_runner / benchmarks/
  vs Ragas 27 / Promptfoo 50+ — LlamaIndex eval 框架内一体化, 不是专门 eval framework, 但够用.

self_evolution: |
  不做.

observability: |
  `llama-index-instrumentation/` 独立 PyPI 含 OTel-based 观测.
  `core/instrumentation/` + `core/callbacks/`.
  跟 Langfuse / Phoenix / Helicone 等专门 obs 协同 (integrations/observability).

hooks: |
  `core/callbacks/` callback handler 风, 不是 Claude Code 风 lifecycle hook.

safety: |
  ? 未深 verify. 没看到 sandbox / moderation 独立模块 (vs Dify).
  Output parser 系统提供 schema 校验防 malformed output.

provider: |
  **104 LLM integration** (vector_stores 78 / readers 159 / embeddings 66 / tools 68 也是顶级).
  每个独立 PyPI: pip install llama-index-llms-openai 等. 业内仅次 LangChain.

channels: |
  Python library 主入口 + `chat_ui/` 自家 UI + `command_line/` CLI + Notebook (`notebook_utils.py`).

persistence: |
  `core/storage/` + 78 vector store integration. Storage abstraction over DB.
  CITATION.cff 学术 first-class.

cancellation: |
  ? 未深 verify. Workflow primitives 可能有 cancel, 但不是 Hermes/CC 那种 Esc-level cancellation.

skillforge_takeaway: |
  LlamaIndex 是 **RAG framework 业内标杆**, 比 LangChain 更专注 RAG/index/composition.

  🔴 **R-LI-1 — Query Engine 18+ 种 explicit type 抽象**(架构启示)
     SkillForge MemoryService 当前 hybrid FTS+Vector+RRF 单一 retrieval 路径.
     LlamaIndex 把 RAG composition 显式建模为 18 种 query engine (citation/router/sub_question/sql_join/multistep/transform/...).
     这套 explicit type 是 RAG framework 业内标杆 — SkillForge memory v3 重构时可参考"显式 retrieval pattern 而非 if-else 堆"思路.
     工作量: 远期参考.

  🟡 **R-LI-2 — Composable memory blocks**(参考)
     `core/memory/memory_blocks/` 模块化 memory blocks + `simple_composable_memory.py` 多种 memory 类型可叠加.
     SkillForge MemoryEntity 三态 (observation/reflection/optimized) 是按 lifecycle 分,
     LlamaIndex blocks 是按功能 (buffer / summary / vector / external) 分.
     两套正交可叠加. 工作量: < 1 周.

  🟡 **R-LI-3 — Property Graph 一等公民**(KG RAG 路径)
     `core/indices/property_graph/` (base / retriever / sub_retrievers / transformations).
     SkillForge memory 当前无 KG, 但 ATTRIBUTION-AGENT 因果链有 graph 雏形.
     如果走 KG memory v3, LlamaIndex property_graph 是 reference impl.
     **GraphRAG-Benchmark ICLR'26 paper 警示: KG RAG 在很多 real-world task 反而 underperform vanilla**, 不必盲跟.

  🟢 **R-LI-4 — Ingestion Pipeline + Cache**(参考)
     `core/ingestion/{pipeline,cache,data_sinks,data_sources,transformations}.py`
     工程化 RAG 数据流 + 增量缓存. 远期 SkillForge skill marketplace + memory 自动入库可参考.

  🟢 **R-LI-5 — Modular monorepo (23 integration × N PyPI)**(参考)
     每个 integration 独立 PyPI package (vector_stores 78 / llms 104 / readers 159).
     SkillForge 当前 monolith Spring, 商业化阶段如果要做 provider marketplace 可参考.

  **❌ 不该跟**:
  - Memory chat history 风 — SkillForge 走 entity/skill/agent memory, 不是 chat history
  - 18 query engine 全部抽象 — 过度工程, SkillForge 1-2 个 retrieval pattern 够用
  - Workflow engine extraction 到独立包 — SkillForge 内置即可
---

# Eval — LlamaIndex (core 0.14.22)

> LlamaIndex 是 RAG framework 业内标杆。818 MB 模块化 monorepo,**llama-index-core 0.14.22** + 23+ integration category(104 LLM / 78 vector store / 159 reader / 66 embedding / 68 tool)。本页 narrative 重点:**为什么 LlamaIndex 把 RAG 显式建模为 18 种 Query Engine + 18 种 Index** 是 RAG framework 设计的范式起点。

---

## 1. 项目定位(一句话)

**专注 RAG / index / composition 的 Python framework**(46.5k stars / MIT),业内 RAG 标杆。比 LangChain 更专注 RAG 单一垂直,比 Haystack 更 modular,比 Dify 更 library-friendly。

---

## 2. 核心范式:18+ × 18+ = RAG 显式建模

```
                  ┌─────────────────────────────┐
                  │  18+ Index 类型              │
                  │  vector / list / keyword /   │
                  │  tree / document_summary /   │
                  │  property_graph (KG) /        │
                  │  knowledge_graph /            │
                  │  struct_store / multi_modal / │
                  │  managed / empty / ...        │
                  └────────────┬────────────────┘
                               │
                               ▼
                  ┌─────────────────────────────┐
                  │  Retriever (从 Index 拿候选) │
                  └────────────┬────────────────┘
                               │
                               ▼
                  ┌─────────────────────────────┐
                  │  Postprocessor (filter/rerank)│
                  └────────────┬────────────────┘
                               │
                               ▼
                  ┌─────────────────────────────┐
                  │  18+ Query Engine 类型       │
                  │  retriever / citation /      │
                  │  router / sub_question /     │
                  │  multistep / multi_modal /   │
                  │  transform / retry /         │
                  │  flare / sql_join /          │
                  │  sql_vector / graph /        │
                  │  knowledge_graph / pandas /  │
                  │  jsonalyze / cogniswitch     │
                  └────────────┬────────────────┘
                               │
                               ▼
                  ┌─────────────────────────────┐
                  │  Response Synthesizer        │
                  └─────────────────────────────┘
```

**核心洞察**:LlamaIndex 把 RAG 当作 **"index storage primitive × query composition primitive"** 的笛卡尔积空间显式建模。其他 framework(LangChain Chain / Dify Workflow / RAGFlow pipeline)是 "blob of chain steps",LlamaIndex 是 **强类型 composition algebra**。

**对 SkillForge memory v3 启示**:当前 `MemoryService.java` 是 ad-hoc 函数堆(`hybridRecallL1` / `vectorRecall` / `ftsRecall`),如果想做更复杂 retrieval pattern(multistep / sub_question / router),LlamaIndex 18+ Query Engine 抽象给出**显式建模的 reference**。

---

## 3. 三大独家技术点

### 3.1 Property Graph 一等公民(`core/indices/property_graph/`)

`base.py` + `retriever.py` + `sub_retrievers/` + `transformations/` + `utils.py` — **KG-as-index 内置抽象**。

跟 [[eval/zep-graphiti]] 对照:Graphiti 是独立 KG memory framework,LlamaIndex 把 KG 当作**index 类型**之一(跟 vector_store / keyword_table / list 平级),意味着同一 RAG pipeline 可以同时混用 vector + KG。

⚠️ **GraphRAG-Benchmark ICLR'26 calibration**:_"GraphRAG frequently underperforms vanilla RAG on many real-world tasks"_。LlamaIndex Property Graph 内置但不强推 — 你可以用,但不必默认用。

### 3.2 Modular Monorepo 23 × N PyPI(`llama-index-integrations/`)

业内最广 integration 生态:
- **78 vector store**(从 alibabacloud / azure / bigquery 到 pgvector / pinecone / weaviate)
- **104 LLM**(从 OpenAI / Anthropic 到 DashScope / Ollama)
- **159 reader**(数据连接器,Notion / Slack / GitHub / Confluence / Google Docs 等)
- **66 embedding** + **68 tool** + 1 **graph_rag-cognee** 官方集成

**用户路径**:`pip install llama-index-core` + 按需 `pip install llama-index-vector-stores-pinecone` 等。**vs Dify monorepo / LangChain v1 16 partner package** — LlamaIndex 极致 modular(每个 integration 独立 PyPI)。

### 3.3 Workflow 已抽离独立包(`workflow.py` 1 行 import)

```python
[code: llama-index-core/llama_index/core/workflow/workflow.py]
from workflows.workflow import Workflow, WorkflowMeta
```

LlamaIndex 把 workflow primitive 抽出来到独立 `workflows` PyPI package — 可独立用,可被其他 framework 复用。Modularization 极致。

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[eval/langchain]] | 并列 | LangChain 是"agent + framework 大而全",LlamaIndex 是"RAG + index 单一深" |
| [[harness/dify]] | 互补 | Dify 14 RAG sub-module + 4 retrieval method 是 RAG **工程化**,LlamaIndex 18+ × 18+ 是 RAG **抽象建模** |
| [[concept/agent-memory]] | 扩展 | LlamaIndex Composable Memory 是 5 路线之外的第 7 路线 "Composable Block Memory" |
| [[concept/rag-design-space]] | **主贡献者** | 18+ × 18+ 矩阵是 RAG 设计空间的骨架 |
| [[eval/zep-graphiti]] | 校准 | Graphiti 是 KG memory 框架,LlamaIndex Property Graph 是 KG index 类型 — 两种抽象层 |

---

## 5. R-LI 借鉴清单

详见 frontmatter `skillforge_takeaway`。总结:

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-LI-1** | Query Engine 18+ 种 explicit type 抽象 | 远期 | 中 — 架构启示 | 🟡 P2 |
| **R-LI-2** | Composable memory blocks(功能维度叠加)| < 1 周 | 中 | 🟡 P2 |
| **R-LI-3** | Property Graph 一等公民(KG RAG 路径)| 中期 | 待定 — ICLR'26 paper 警示 KG 未必 better | 🟢 P3 |
| **R-LI-4** | Ingestion Pipeline + Cache | 远期 | 低 — 工程化 | 🟢 P3 |
| **R-LI-5** | Modular monorepo(provider PyPI 化) | 远期 | 商业化阶段 | 🟢 P3 |

**对 SkillForge 的真实价值**:LlamaIndex 给 SkillForge memory v3 提供**抽象层 reference**(显式 Query Engine + Index 类型),但**不必照抄**(SkillForge memory 是 entity-based,不是 doc-based)。最高 ROI 是 R-LI-2 (composable memory blocks) 给 SkillForge 现有三态 memory 加功能维度。

---

## 6. 已知 post-ingest 风险

- llama-index-core 内 56 子模块只看了 10+ 个核心,其他 40+ 未深读
- 23 integration category × N package 只抽样 4-5 个(vector_stores/llms/readers/graph_rag),其他未 verify
- `workflows` 独立 PyPI package 内部细节未读
- ATTRIBUTION-AGENT × LlamaIndex Property Graph 对照可深度展开

---

## 7. 引用

- [[source/llama_index]] 完整源码锚点
- GitHub: `run-llama/llama_index` 46.5k stars / MIT
- `CITATION.cff` 学术 first-class
- 跨页对照:[[eval/langchain]] / [[harness/dify]] / [[concept/agent-memory]] / [[concept/rag-design-space]]
- ICLR'26 calibration:[[concept/rag-design-space]](GraphRAG-Benchmark 后续 ingest)
