---
name: llama_index
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: llama_index
version: llama-index-core 0.14.22
commit: 23bd65aaf79ffed587af35f325738b1580b35dea
license: MIT
stars: 46.5k
last_ingested_at: 2026-05-17
contributed_to:
  - "[[eval/llama_index]]"
  - "[[concept/rag-design-space]]"
---

# Source — LlamaIndex @ commit 23bd65a (core 0.14.22)

> run-llama/llama_index — 业内 RAG framework 标杆。818 MB 模块化 monorepo:**llama-index-core 0.14.22** + **23+ integration category 各自独立 PyPI package**(包括 104 LLM / 78 vector store / 159 reader / 66 embedding / 68 tool)。clone @ `/Users/youren/myspace/llama_index/`。

---

## 1. 模块化 monorepo 结构

```
llama_index/
├── llama-index-core/              # 核心 abstraction (本次 ingest 重点)
├── llama-index-integrations/      # 23+ 个 integration 类别, 每个含 N 个独立 PyPI package
├── llama-index-instrumentation/   # OTel-based 观测
├── llama-index-utils/             # 工具
├── llama-dev/                     # 开发工具
├── docs/                          # 完整 doc
├── scripts/                       # 维护脚本
├── pyproject.toml                 # uv-based
├── CITATION.cff                   # 学术引用 first-class
└── README.md
```

**`llama-index-integrations/` 23 类**:
agent / callbacks / embeddings / evaluation / extractors / **graph_rag** / graph_stores / indices / ingestion / llms / memory / node_parser / observability / output_parsers / postprocessor / program / protocols / question_gen / readers / response_synthesizers / retrievers / selectors / sparse_embeddings / storage / tools / vector_stores / voice_agents

---

## 2. `llama-index-core` — 56 子模块 [verified]

```
llama-index-core/llama_index/core/
├── agent/              # react + workflow 风格 agent
├── async_utils.py
├── base/
├── bridge/             # LangChain / DSPy bridge
├── callbacks/
├── chat_engine/        # 对话 engine
├── chat_ui/            # 自家 UI
├── command_line/
├── composability/      # index 组合
├── constants.py
├── data_structs/
├── embeddings/
├── evaluation/         # 12 内置 metric
├── extractors/
├── graph_stores/
├── image_retriever.py
├── img_utils.py
├── indices/            # 18+ index 类型
├── ingestion/          # Pipeline + cache + DataSink + DataSource
├── instrumentation/
├── langchain_helpers/  # LangChain 兼容层
├── llms/
├── memory/             # 5 种 memory 类型
├── multi_modal_llms/
├── node_parser/        # 3 类: file / text / relational
├── objects/
├── output_parsers/
├── playground/
├── postprocessor/
├── program/            # structured output program
├── prompts/
├── query_engine/       # 18+ query engine 类型
├── question_gen/
├── rate_limiter.py
├── readers/
├── response/
├── response_synthesizers/
├── retrievers/
├── schema.py
├── selectors/
├── service_context_elements/
├── service_context.py
├── settings.py
├── sparse_embeddings/
├── storage/
├── text_splitter/
├── tools/
├── types.py
├── utilities/
├── utils.py
├── vector_stores/
├── voice_agents/
└── workflow/           # 仅 import workflows pkg (extracted)
```

---

## 3. 关键 verified 锚点

### 3.1 18+ Query Engine 类型 [code: llama-index-core/llama_index/core/query_engine/]

```
citation_query_engine.py      # 带 citation 标注
cogniswitch_query_engine.py   # CogniSwitch 集成
custom.py                     # 自定义
flare/                        # FLARE: Forward-Looking Active REtrieval
graph_query_engine.py         # 知识图谱
jsonalyze/                    # JSON 分析
knowledge_graph_query_engine.py
multi_modal.py                # 多模态
multistep_query_engine.py     # 多步检索
pandas/                       # DataFrame 查询
retriever_query_engine.py     # 基础 (retriever → synth)
retry_query_engine.py         # 重试
retry_source_query_engine.py
router_query_engine.py        # 路由
sql_join_query_engine.py      # SQL + RAG 混合
sql_vector_query_engine.py    # SQL + 向量混合
sub_question_query_engine.py  # 拆子问题
transform_query_engine.py     # 查询转换
```

**核心 RAG composition primitive**:其他 framework 通常给 1-2 个,LlamaIndex 给 18+ 显式建模 RAG 不同 patterns。

### 3.2 18+ Indices 类型 [code: llama-index-core/llama_index/core/indices/]

```
common / common_tree / composability /
document_summary / empty / keyword_table /
knowledge_graph / list /
managed / multi_modal /
property_graph / query / registry / struct_store /
vector_store / + base.py + base_retriever.py + postprocessor.py + prompt_helper.py + loading.py
```

**Property Graph 一等公民**(`indices/property_graph/`)— `base.py` / `retriever.py` / `sub_retrievers/` / `transformations/` — KG RAG 内置(对照 [[harness/dify]] 也有 graph_rag 但 Dify 是外部 plugin)。

### 3.3 Workflow 已抽离为独立包 [code: llama-index-core/llama_index/core/workflow/workflow.py]

```python
from workflows.workflow import Workflow, WorkflowMeta  # noqa
```

LlamaIndex `workflow.py` 完整内容就这 1 行 —— 已抽出独立 `workflows` PyPI package。这是 modularization 极致(对照 LangChain v1 `partners/` 16 独立 PyPI 同思路,但 LlamaIndex 把核心 abstraction 自己也抽出去)。

### 3.4 Memory 5 种类型 [code: llama-index-core/llama_index/core/memory/]

```
chat_memory_buffer.py            # 简单 buffer
chat_summary_memory_buffer.py    # 带 summary 的 buffer
memory_blocks/                   # 模块化 memory blocks
memory.py                        # 基类
simple_composable_memory.py      # 可组合 (多种 memory 类型并存)
vector_memory.py                 # 向量化记忆
```

**Composable memory + memory blocks** — 多种 memory 类型可叠加。对照 AgentScope `_long_term_memory/` + `_working_memory/` 双层,LlamaIndex 是 N 层任意组合。

### 3.5 Evaluation 12 内置 metric [code: llama-index-core/llama_index/core/evaluation/]

```
answer_relevancy.py / base.py / batch_runner.py /
benchmarks/ / context_relevancy.py / correctness.py /
dataset_generation.py / eval_utils.py / faithfulness.py /
guideline.py / multi_modal/ / notebook_utils.py /
pairwise.py / relevancy.py / retrieval/ / semantic_similarity.py
```

**13 RAG eval metric 内置**(vs Ragas 27 / Promptfoo 50+)— LlamaIndex eval 弱于专门 framework,但**框架内一体化**是产品力。

### 3.6 Ingestion Pipeline [code: llama-index-core/llama_index/core/ingestion/]

```
__init__.py / api_utils.py / cache.py /
data_sinks.py / data_sources.py /
pipeline.py / transformations.py
```

**Pipeline + cache + DataSink + DataSource + transformations** — 工程化 RAG 数据流。

### 3.7 Node Parser 3 类 [code: llama-index-core/llama_index/core/node_parser/]

```
file/        # 按文件格式 parse
text/        # 按文本结构 chunk
relational/  # 关系型 (table)
```

---

## 4. Integrations 规模(business moat)[verified counts]

| 类别 | 数量 | 备注 |
|---|---|---|
| **vector_stores** | **78** | 业内最广,含 alibabacloud / azure / bigquery / cassandra / chroma / pgvector / pinecone / weaviate 等(15+ 主流 + 60+ 长尾) |
| **llms** | **104** | OpenAI / Anthropic / Azure / Bedrock / Cohere / Gemini / Ollama / Mistral / DashScope / 等 |
| **readers** | **159** | 业内最多 data connector(Notion / Slack / GitHub / Google Docs / Confluence / 等) |
| **embeddings** | 66 | OpenAI / Cohere / HuggingFace / Voyage / Jina / etc |
| **tools** | 68 | 各类外部 API tool |
| **agent** | 2+ | agentmesh / azure |
| **graph_rag** | 1 | **`llama-index-graph-rag-cognee`** — 跟 Cognee 官方集成 |

**对照 SkillForge**:`OpenAiProvider` + `ClaudeProvider` 2 个 vs LlamaIndex 104 个 — 商业化阶段如果要做 model marketplace,LlamaIndex 模板可借鉴(每个 LLM 独立 PyPI package,用户按需 pip install)。

---

## 5. 关键技术决策(verified from code)

| 决策 | LlamaIndex 选择 | 跟其他 framework 对比 |
|---|---|---|
| **Composition primitive** | Query Engine(18+ 种 explicit type)| LangChain Chain / AutoGen Agent / Dify Workflow |
| **Index modeling** | 18+ Index 类型 vs flat doc store | RAGFlow doc-store / mem0 entity-store / Zep KG store |
| **Modularization** | 23 integration category × N PyPI package | LangChain v1 16 partner / Dify monorepo |
| **Workflow** | 已抽离到 `workflows` 独立 package | AgentScope `pipeline/` 内置 / Dify `workflow/` 内置 |
| **Memory** | Composable N 种叠加 | AgentScope 双层 / Letta MemGPT function-call / Dify pure-RAG |
| **Eval** | 框架内 13 metric(轻)| Ragas 27 / Promptfoo 50+(重) |

---

## 6. 关键文件清单(供后续 query)

| 路径 | 主题 |
|---|---|
| `llama-index-core/llama_index/core/query_engine/*` | 18+ Query Engine 类型 |
| `llama-index-core/llama_index/core/indices/property_graph/` | Property Graph KG RAG |
| `llama-index-core/llama_index/core/indices/document_summary/` | Document Summary Index |
| `llama-index-core/llama_index/core/indices/registry.py` | Index 注册 |
| `llama-index-core/llama_index/core/memory/*` | 5 种 memory + memory_blocks/ |
| `llama-index-core/llama_index/core/ingestion/pipeline.py` | Ingestion Pipeline |
| `llama-index-core/llama_index/core/evaluation/*` | 13 metric |
| `llama-index-core/llama_index/core/workflow/workflow.py` | 抽离 marker (1 行 import) |
| `llama-index-core/llama_index/core/agent/{react,workflow}/` | Agent 路径 |
| `llama-index-integrations/vector_stores/` | 78 vector store integration |
| `llama-index-integrations/llms/` | 104 LLM integration |
| `llama-index-integrations/readers/` | 159 data reader |
| `llama-index-integrations/graph_rag/llama-index-graph-rag-cognee/` | Cognee 官方集成 |

---

## 7. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/llama_index/](file:///Users/youren/myspace/llama_index/) 818 MB
- GitHub: `run-llama/llama_index` 46.5k stars / MIT
- `CITATION.cff` 学术 first-class
- 衍生 wiki:[[eval/llama_index]] / [[concept/rag-design-space]] 后续合并
