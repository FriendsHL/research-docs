---
name: cognee
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: topoteretes/cognee
version: 1.0.9
commit: 4ca1d0c2bbbb46924acb1f5f6cd805214805ca16
license: Apache 2.0
last_ingested_at: 2026-05-17
contributed_to:
  - "[[eval/cognee]]"
  - "[[concept/rag-design-space]]"
  - "[[concept/agent-memory]]"
---

# Source — Cognee 1.0.9 @ commit 4ca1d0c2

> topoteretes/cognee — **"Memory control plane for AI Agents"**(Apache 2.0,75 MB)。slogan: _"Cognee - The Brain behind your Agents"_。**graph + embedding + cognitive science** 三者融合,跟 SkillForge memory 路线最近的 framework。

---

## 1. Top-level 仓库结构

```
cognee/
├── cognee/                 # ⭐ 核心 Python 包
├── cognee-mcp/             # MCP server (独立子项目)
├── cognee-frontend/        # Next.js UI
├── cognee-starter-kit/     # 用户起步模板
├── distributed/            # Modal + workers + queues
├── kuzu/                   # Kuzu graph DB 集成
├── evals/                  # 评测脚本
├── examples/               # configurations + custom_pipelines + database_examples + demos + guides + pocs
├── deployment/             # 部署
├── alembic/                # 数据库 migration
├── docker-compose.yml + Dockerfile
└── AGENTS.md + CLAUDE.md   # dogfood
```

---

## 2. `cognee/` 核心包 — 22 子模块 [verified]

```
cognee/
├── api/                # FastAPI + 12 router (add/cognify/memify/search/delete/users/datasets/responses/visualize/settings/sync/update/checks)
├── cli/                # cognee-cli 命令行
├── infrastructure/     # 基础设施 (databases / llm / embeddings / loaders / storage / context / session / locks)
├── modules/            # ⭐ 领域逻辑 (25 子模块, 见 §3)
├── tasks/              # ⭐ 可复用任务 (17 子目录, 见 §4)
├── memify_pipelines/   # ⭐ "memify" 7 个内置 pipeline
├── memory/             # 记忆条目 (entries.py)
├── eval_framework/     # ⭐ 内置评测 framework (含 BEAM benchmark)
├── pipelines/          # Pipeline 抽象
├── shared/             # 共享工具
├── alembic/            # 数据库 migration
├── exceptions/
├── tests/
├── skill.md            # ⭐ Anthropic Skill 标准文件 (业内独家)
├── version.py          # 1.0.9
├── low_level.py
├── base_config.py
├── context_global_variables.py
├── get_token.py
├── root_dir.py
└── run_migrations.py
```

**业内独家**:Cognee 在仓库内放 `skill.md`(Anthropic Skill 标准格式),里面写 _"Use this skill whenever the user asks about Cognee, AI memory, persistent agent memory, ..."_ — **Cognee 把自己包装成可被 Claude/agent 直接发现 + 调用的 skill**(对照 [[harness/dify]] 的 6 `.claude/skills/` 是 Dify 自家开发用,Cognee 这是给用户在 Claude Code / 其他 harness 里安装时直接调用)。

---

## 3. `cognee/modules/` — 25 领域模块 [verified]

```
agent_memory/        # ⭐ Agent memory 抽象 (decorator + runtime + sanitization)
chunking/            # Chunking
cloud/               # Cloud 集成
cognify/             # ⭐ "cognify" 把数据 → KG (config)
data/                # 数据
engine/              # 引擎
graph/               # Graph 操作
ingestion/           # Ingestion
memify/              # ⭐ "memify" 主流程 (memify.py)
metrics/             # Metrics
notebooks/           # Notebook 集成
observability/       # 可观测
ontology/            # ⭐ RDF/XML Ontology resolver (业内罕见)
pipelines/           # Pipeline
recall/              # ⭐ Recall (methods + types 抽象)
retrieval/           # ⭐ 22 retriever (见 §5)
run_custom_pipeline/ # 自定义 pipeline runner
search/              # Search
session_lifecycle/   # ⭐ Session 生命周期
settings/
storage/
sync/                # 同步
users/               # 用户 + 权限
visualization/       # 可视化
```

---

## 4. `cognee/tasks/` — 17 task 类别 [verified]

```
chunks/              # Chunk 处理
cleanup/             # 清理
codingagents/        # ⭐ 编码 agent 专用
completion/          # 补全
documents/           # 文档处理
entity_completion/   # 实体补全
graph/               # 图操作
ingestion/           # Ingest
memify/              # memify task
schema/              # Schema
storage/             # 存储
summarization/       # 摘要
temporal_awareness/  # ⭐ 时间感知
temporal_graph/      # ⭐ 时间图谱
translation/         # 翻译
web_scraper/         # 网页爬取
```

**Temporal 一等公民**(`temporal_awareness/` + `temporal_graph/`)— 跟 [[eval/zep-graphiti]] Graphiti Temporal KG 同思路。

---

## 5. `cognee/modules/retrieval/` — 22 Retriever 类型 [code: cognee/modules/retrieval/]

```
base_retriever.py
chunks_retriever.py                                # Chunk-level
coding_rules_retriever.py                          # Coding rules 专用
completion_retriever.py
cypher_search_retriever.py                         # ⭐ Neo4j Cypher 查询
graph_completion_context_extension_retriever.py    # 图补全 + 上下文扩展
graph_completion_cot_retriever.py                  # ⭐ 图补全 + CoT
graph_completion_decomposition_retriever.py        # ⭐ 图补全 + 分解
graph_completion_retriever.py                      # 图补全
graph_summary_completion_retriever.py              # 图摘要 + 补全
jaccard_retrival.py                                # Jaccard 相似度
lexical_retriever.py                               # 词法
natural_language_retriever.py                      # 自然语言查询
register_retriever.py                              # 注册
registered_community_retrievers.py                 # 社区 retriever
summaries_retriever.py
temporal_retriever.py                              # ⭐ 时间检索
triplet_retriever.py                               # ⭐ Triplet (SPO 三元组)
+ context_providers/ + entity_extractors/ + exceptions/ + utils/
```

**22 retriever vs Microsoft GraphRAG 4 search mode** — Cognee 把 KG retrieval 进一步细分,5 种 graph_completion 变体(基础 / 上下文扩展 / CoT / 分解 / 摘要)显式建模 RAG composition pattern。

---

## 6. `cognee/memify_pipelines/` — 7 个核心 Pipeline [code: cognee/memify_pipelines/]

```
apply_feedback_weights.py                          # ⭐ 反馈权重 (用户反馈 → memory 权重)
apply_frequency_weights.py                         # 频率权重
consolidate_entity_descriptions.py                 # 实体描述合并 (dedup-like)
create_triplet_embeddings.py                       # Triplet (S-P-O) embedding
memify_default_tasks.py                            # 默认任务
persist_agent_trace_feedbacks_in_knowledge_graph.py  # ⭐ Agent trace + feedback → KG
persist_sessions_in_knowledge_graph.py             # ⭐ Session → KG
```

**核心创新**:Cognee 把 _"agent 跑出来的 trace"_ 和 _"用户对 agent 输出的 feedback"_ 都**自动持久化进 KG**(`persist_agent_trace_feedbacks_in_knowledge_graph`)— 这是闭环 self-improving memory。

**对照** SkillForge:
- SkillForge 有 `t_session` + `t_message` + `LlmTraceEntity`(observability),但**没有把 trace → KG 自动 persist**
- SkillForge MEMORY-LLM-SYNTHESIS 是 LLM 提议 memory + 人审 gate,**用户 feedback 信号没显式 routed 进 memory pipeline**
- 这是 SkillForge ATTRIBUTION-AGENT v3 + MEMORY v3 协同的关键缺口 — Cognee 给参考

---

## 7. `cognee/eval_framework/` — 内置评测 [code: cognee/eval_framework/]

```
analysis/
answer_generation/
benchmark_adapters/                                # ⭐ 多 benchmark 适配
corpus_builder/
evaluation/
metrics_dashboard.py
modal_eval_dashboard.py                            # Modal serverless dashboard
modal_run_eval.py                                  # Modal 运行 eval
run_beam_eval.py                                   # ⭐ BEAM benchmark
run_eval.py
eval_config.py
```

**`run_beam_eval.py`** 内置 BEAM benchmark 跑流(参 [[eval/mem0]] BEAM 64.1% / mem0 数据)— Cognee 直接跑 BEAM, 跟 mem0 / Letta 等 memory framework 公开 benchmark 对齐。

---

## 8. `cognee/infrastructure/databases/` — 8 类 [code: cognee/infrastructure/databases/]

```
cache/                  # 缓存
dataset_database_handler/
dataset_queue/
graph/                  # Graph DB (Kuzu + Neo4j + ...)
hybrid/                 # ⭐ Hybrid (graph + vector)
relational/             # 关系型 (Postgres / SQLite)
unified/                # ⭐ Unified (一致接口跨多 backend)
vector/                 # Vector DB
```

**`unified/` + `hybrid/` 两层抽象** — Cognee 让 graph + vector + relational 不仅可单用,还能 unified 一致接口 / hybrid 混合查询(对照 [[eval/zep-graphiti]] Graphiti 4 driver 是 graph-only)。

---

## 9. 关键 verified 锚点

| 路径 | 主题 |
|---|---|
| `cognee/skill.md` | **Anthropic Skill 标准 wrapper 让 Cognee 可被 agent 发现** |
| `cognee/modules/agent_memory/{decorator,runtime,sanitization}.py` | **decorator-based agent memory API**(`@cognify` 风) |
| `cognee/modules/memify/memify.py` | "memify" 主流程 |
| `cognee/memify_pipelines/persist_agent_trace_feedbacks_in_knowledge_graph.py` | **trace + feedback → KG 自动持久化** |
| `cognee/modules/retrieval/{graph_completion_cot,decomposition,context_extension,summary_completion}_retriever.py` | 5 种 graph_completion 变体 |
| `cognee/modules/retrieval/cypher_search_retriever.py` | Neo4j Cypher 查询支持 |
| `cognee/modules/retrieval/temporal_retriever.py` + `cognee/tasks/temporal_graph/` | Temporal KG 一等公民 |
| `cognee/modules/ontology/{rdf_xml,matching_strategies,base_ontology_resolver}.py` | **RDF/XML Ontology resolver**(业内罕见) |
| `cognee/infrastructure/databases/{unified,hybrid}/` | unified + hybrid 抽象 |
| `cognee/eval_framework/run_beam_eval.py` | BEAM benchmark 内置 |
| `cognee-mcp/` | 独立 MCP server 子项目 |
| `cognee-frontend/` | Next.js UI |
| `kuzu/` | Kuzu graph DB 集成 |

---

## 10. Recent activity

最新 commit `fix(frontend): sync package lock (#2833)` — 持续积极开发。GitHub topic `cognee` 显示 community 活跃。Discord `r/AIMemory` 专门 subreddit。Trendshift badge 13955。

LlamaIndex 已有官方集成包 `llama-index-graph-rag-cognee`(印证 §[[source/llama_index]] §4 graph_rag 类别只 1 个集成就是 Cognee — Cognee 是 LlamaIndex 选的 KG memory partner)。

---

## 11. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/cognee/](file:///Users/youren/myspace/cognee/) 75 MB
- GitHub: `topoteretes/cognee` Apache 2.0
- LlamaIndex 官方集成:`llama-index-graph-rag-cognee`
- 衍生 wiki:[[eval/cognee]] / [[concept/rag-design-space]] / [[concept/agent-memory]]
