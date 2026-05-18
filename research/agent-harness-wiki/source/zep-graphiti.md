---
type: source
name: zep-graphiti
origin: "/Users/youren/myspace/zep/ + /Users/youren/myspace/graphiti/ (getzep/zep + getzep/graphiti)"
ingested_at: 2026-05-16
ingested_commit: "zep: faf2ace (2026-04-08) + graphiti: 9a2d6d0 (2026-05-14)"
ingested_commit_date: 2026-05-14
ingested_version_label: "Zep(本仓库主要为 examples/integrations,主 source 转 Cloud SaaS) + Graphiti 0.29.0(独立 OSS framework)"
contributed_to:
  - eval/zep-graphiti.md
  - concept/agent-memory.md
---

# Source — Zep + Graphiti(Temporal Knowledge Graph)

> **Zep AI 出品**(getzep,商业 SaaS Zep Cloud + 开源 Graphiti library)。
>
> ⚠️ **重要**:`getzep/zep` 主仓库现在是 **examples + integrations + legacy**,**Zep 主 source 转 Zep Cloud SaaS**(`getzep.com`)。**Graphiti 是独立 Python OSS framework**(`getzep/graphiti` 0.29.0,arxiv 2501.13956)。
>
> **Build Temporal Context Graphs for AI Agents** — 业内独家"**context graph 跟踪 facts 怎么随时间变**",每个 fact 有 **validity window**(when it became true / when superseded)。

## 仓库结构概览(2 个 repo)

### Zep repo(`/Users/youren/myspace/zep/`)

```
zep/
├── assets/
├── benchmarks/                      # ⭐ Zep 公开 benchmark
│   ├── locomo/
│   └── longmemeval/
│       ├── analyze_dataset_sessions.py
│       ├── pyproject.toml
│       ├── README.md
│       ├── session_analysis_summary.txt
│       ├── Zep Test Harness/
│       ├── zep_longmem_eval.py
│       └── zep_memgpt_eval.ipynb     # ⭐ Zep vs MemGPT eval
├── examples/                        # 用户示例
│   ├── go/
│   ├── python/
│   └── typescript/
├── integrations/python/
├── legacy/                          # ⭐ 旧 Zep Go source code(归档)
│   ├── docker-compose.ce.yaml
│   ├── Dockerfile.ce
│   ├── go.work + go.work.sum
│   ├── Makefile.ce
│   ├── src/                         # Go source
│   ├── zep/ + zep.yaml
├── mcp/zep-mcp-server/              # ⭐ Zep MCP server(Go)
│   ├── cmd / internal / pkg
│   ├── docker-compose.yml + Dockerfile
│   └── go.mod + go.sum
├── ontology/                        # ⭐ Ontology definitions
│   └── default_ontology.py
├── zep-eval-harness/                # ⭐ Eval harness 工具
│   └── config/ + data/ + runs/
└── pyproject.toml(Note: "**This repository is currently a work in progress**")
```

### Graphiti repo(`/Users/youren/myspace/graphiti/`)

```
graphiti/
├── graphiti_core/                   # ⭐ Python 主包(0.29.0)
│   ├── __init__.py
│   ├── graphiti.py                  # ⭐ 主 Graphiti class
│   ├── graphiti_types.py            # Pydantic types
│   ├── nodes.py                     # ⭐ Entity node
│   ├── edges.py                     # ⭐ Relationship edge(temporal)
│   ├── graph_queries.py             # Graph queries
│   ├── decorators.py
│   ├── helpers.py
│   ├── tracer.py
│   ├── errors.py
│   ├── search/                      # ⭐ Hybrid search(semantic + keyword + graph traversal)
│   ├── llm_client/                  # LLM client
│   ├── embedder/                    # Embedding
│   ├── cross_encoder/               # Cross encoder reranking
│   ├── driver/                      # ⭐ Multi-DB driver
│   │   ├── neo4j/                   # ⭐ Neo4j(default)
│   │   ├── falkordb/                # FalkorDB
│   │   ├── kuzu/                    # Kuzu(embedded)
│   │   ├── neptune/                 # AWS Neptune
│   │   ├── graph_operations/
│   │   ├── operations/
│   │   └── search_interface/
│   ├── models/                      # Pydantic models
│   ├── namespaces/                  # Multi-tenant namespace
│   ├── migrations/                  # Schema migrations
│   ├── prompts/                     # LLM prompt templates
│   ├── telemetry/                   # OpenTelemetry
│   ├── utils/
│   │   └── ontology_utils/          # ⭐ Ontology utilities
│   └── py.typed
├── server/                          # FastAPI server
├── mcp_server/                      # ⭐ Graphiti MCP server(Python)
├── tests/
├── examples/                        # 用户示例
├── signatures/                      # 签名
├── spec/
├── images/                          # 文档图
├── pyproject.toml(version = "0.29.0")
├── uv.lock
├── Dockerfile + docker-compose.yml + docker-compose.test.yml
├── conftest.py + pytest.ini
├── Makefile
├── depot.json + ellipsis.yaml
├── OTEL_TRACING.md                  # OpenTelemetry 文档
├── Zep-CLA.md                       # CLA(Zep AI 项目共享)
├── AGENTS.md + CLAUDE.md
├── CODE_OF_CONDUCT.md + CONTRIBUTING.md + SECURITY.md
└── README.md(arxiv 2501.13956)
```

## 关键 file:line 锚点

### Graphiti 学术 backbone(README + arxiv)

**arxiv paper**:**2501.13956**(2025-01 发布)— "Graphiti: Temporal Knowledge Graphs for AI Agents"

### Context Graph 定义(README "What is a Context Graph?")

> "A **context graph** is a temporal graph of entities, relationships, and facts — like *"Kendra loves Adidas shoes (as of March 2026)."* Unlike traditional knowledge graphs, each fact in a context graph has a **validity window**: when it became true, and when (if ever) it was superseded. Entities evolve over time with updated summaries. Everything traces back to **episodes** — the raw data that produced it."

**核心 4 概念**:
1. **Entities**(nodes)— People, products, policies, concepts,有 summaries 会随时间更新
2. **Facts / Relationships**(edges)— Triplets(Entity → Relationship → Entity)+ **temporal validity windows**
3. **Episodes** — Raw data(对话/文档/事件)产生 entities + facts 的 provenance
4. **Ontology** — 实体 / 关系 types(prescribed + learned)

### Multi-DB Driver(`graphiti_core/driver/`)

**4 个 graph DB driver**:
- **`neo4j/`** — Neo4j(default,业内主流 graph DB)
- **`falkordb/`** — FalkorDB(Redis 兼容 graph DB)
- **`kuzu/`** — Kuzu(embedded graph DB,本地)
- **`neptune/`** — AWS Neptune

跟 [[mempalace]] SQLite-backed KG(单 DB)/ [[mem0]] vector store 多 backend 对照:**Graphiti 是 graph-first multi-driver**。

### Hybrid Search(`graphiti_core/search/`)

README 明示:**hybrid retrieval(semantic + keyword + graph traversal)**

跟其他对比:
- [[mem0]] v2:multi-signal(semantic + BM25 + entity matching)
- [[mempalace]] hybrid v4:keyword + temporal-proximity + preference
- **Graphiti**:semantic + keyword + **graph traversal**(独家:跨节点遍历检索)

### Cross Encoder(`graphiti_core/cross_encoder/`)

业内独家:**Cross encoder reranking**(对照 [[mem0]] reranker / [[mempalace]] LLM rerank)— Cross encoder 是 NLP 学术方法(query × candidate pair 直接 encode),比 LLM rerank 更精细 + 更便宜。

### Ontology 系统(`graphiti_core/utils/ontology_utils/`)

业内独家:**Ontology 一等公民**:
- **Prescribed ontology**(用户预定义实体 + 关系 type)
- **Learned ontology**(自动从数据学习新 type)
- `zep/ontology/default_ontology.py` Zep 自带 default ontology

### MCP Server(`graphiti/mcp_server/` + `zep/mcp/zep-mcp-server/`)

**Graphiti 跟 Zep 都有自家 MCP server**:
- `graphiti/mcp_server/` — Python MCP server
- `zep/mcp/zep-mcp-server/` — Go MCP server

README "TIP":
> "Check out the new MCP server for Graphiti! Give Claude, Cursor, and other MCP clients powerful context graph-based memory with temporal awareness."

跟 [[../eval/phoenix]] / [[helicone]] / [[mempalace]] MCP server 一致业内主流。

### Episode 系统(`graphiti_core/`)

**Episodes 是 Graphiti 独家概念**:
- Raw data(对话 message / 文档 / event)
- 产生 entities + facts 的 provenance
- **每个 fact 都可追溯到产生它的 episode**(对照 [[mempalace]] verbatim 存储所有原文)

### Multi-tenant Namespace(`graphiti_core/namespaces/`)

业内 framework 罕见的 multi-tenant first-class — 对照 [[../eval/langfuse]] main+secondary 双队列 multi-tenant。

### Migrations(`graphiti_core/migrations/`)

Schema migrations 一等公民(对照 Letta alembic / Langfuse Drizzle)。

### Zep `legacy/` 是旧 Zep Open Source Edition(CE)

`zep/legacy/`:
- `Dockerfile.ce` + `docker-compose.ce.yaml` + `Makefile.ce` — Community Edition
- `src/` Go source
- `zep/ + zep.yaml` — 配置

**含义**:Zep CE 已 legacy,**Zep 现在 Cloud-only**。**Graphiti 是 OSS 接续**。

### Zep Eval Harness(`zep/zep-eval-harness/`)

Zep 自家 eval harness(对照 [[../eval/opik]] Pytest 集成)— Cloud 评测工具。

### Zep Benchmarks(`zep/benchmarks/`)

- `locomo/` — LoCoMo benchmark
- `longmemeval/zep_longmem_eval.py` — Zep on LongMemEval
- **`longmemeval/zep_memgpt_eval.ipynb`** — **Zep vs MemGPT 对比**(直接对照 [[letta]] MemGPT)
- `session_analysis_summary.txt`

### Zep SDK(3 语言)

- **Python**:`pip install zep-cloud`
- **TypeScript/JS**:`npm install @getzep/zep-cloud`
- **Go**:`go get github.com/getzep/zep-go/v2`

**3 语言 SDK**(对照 [[../eval/opik]] Python + TS + Ruby via OTel)

### OpenTelemetry 集成(`graphiti/OTEL_TRACING.md`)

Graphiti 跟 `OTEL_TRACING.md` 独立文档解释 OpenTelemetry 集成 — 对照 [[../eval/phoenix]] OpenInference / [[../eval/langfuse]] OTel ingestion。

## 与 wiki 现有

### 印证
- **Hybrid retrieval(semantic + keyword + graph traversal)** 跟 [[mempalace]] hybrid v4(keyword + temporal + preference)+ [[mem0]] v2 multi-signal(semantic + BM25 + entity)同方向
- **MCP server** 跟 Phoenix / Helicone / MemPalace 一致
- **Multi-DB driver**(Neo4j / FalkorDB / Kuzu / Neptune)跟 [[mem0]] multi-vector-store 同思路
- **`integrations/`** + `examples/{go,python,typescript}/` 跟 LangChain 16 partner 一致

### 矛盾
- **Zep 主仓库已转 examples + integrations**(Cloud-only) — 跟 Langfuse / Opik / Phoenix 完全 OSS 路线对立
- **Graphiti = Zep 接续 OSS 部分** — 但限 Temporal KG framework,不含完整 memory store
- **Temporal validity windows** 跟其他 memory framework 完全不同:
  - mem0:ADD-only 累加(时间戳但无 validity window)
  - MemPalace:Temporal KG + validity windows(本地 SQLite,跟 Graphiti 同模式)
  - Graphiti:Temporal KG + validity windows + graph DB(Neo4j 等)
  - Letta:Memory blocks 无 temporal validity
  - SkillForge:可能时间戳但无 validity window?(待 verify)

### 新增
- **`eval/zep-graphiti.md`**(新建)— Zep + Graphiti 合一(两 repo 同 ecosystem)
- **扩 [[../concept/agent-memory.md]]**(待新建):**Temporal KG with validity windows** 路线(Graphiti / MemPalace)vs **ADD-only 累加**(mem0)vs **Agent OS memory blocks**(Letta)vs **Background reflection**(LangMem)

## 事实漂移 / 注意点

- **Zep 主仓库 README 顶部明示"work in progress"**,主要价值是 examples + integrations + legacy + benchmarks
- **Zep CE 已 legacy**:Zep 现在 Cloud-only 商业 SaaS(`getzep.com`)
- **Graphiti 0.29.0** 是接续 OSS 的活产物 — 跟 mem0 / MemPalace / Letta 同档活跃
- **arxiv 2501.13956**(2025-01)Graphiti paper
- **`Zep-CLA.md`** Contributor License Agreement — Zep AI 项目共享 CLA
- **4 graph DB driver**(Neo4j / FalkorDB / Kuzu / Neptune)— Neo4j default,**Kuzu embedded 适合 SkillForge 本地场景**
- **Cross encoder reranking** 业内独家(跨 query × candidate pair encode,比 LLM rerank 便宜 + 精细)
- **Episode 概念**(raw data → entities + facts 的 provenance)— **跟 [[mempalace]] verbatim 存储** + [[mem0]] LLM extracted facts 都不同:Graphiti 存所有,但提取出 graph + 保留 provenance
- **3 语言 SDK** + 跟 Cloud SaaS 商业模式
- **MCP server 2 个**(Python + Go)— 业内独家(对照 Phoenix / Helicone / MemPalace 各 1 个 MCP server)
- **Multi-tenant namespace 一等公民**(`namespaces/`)— 对照 Langfuse main+secondary 双队列
- **OpenTelemetry 独立文档**(`OTEL_TRACING.md`)
