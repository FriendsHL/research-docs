---
name: ms-graphrag
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: microsoft/graphrag
version: latest (semversioner-managed)
commit: de531f0a697d2f35c4f85cb8511141507278404e
license: MIT
arxiv: "2404.16130"
last_ingested_at: 2026-05-17
contributed_to:
  - "[[eval/ms-graphrag]]"
  - "[[concept/rag-design-space]]"
---

# Source — Microsoft GraphRAG @ commit de531f0a

> microsoft/graphrag — **KG RAG paradigm 起点论文实现**(arxiv 2404.16130)。Microsoft Research 出品,demonstration code 非 officially supported product。clone @ `/Users/youren/myspace/ms-graphrag/` 31 MB(小,因为是 demo 实现)。

---

## 1. Top-level 仓库结构(monorepo 8 sub-package)

```
ms-graphrag/
├── packages/
│   ├── graphrag/                   # ⭐ 主包 (本次 ingest 重点)
│   ├── graphrag-cache/             # 缓存
│   ├── graphrag-chunking/          # Chunking
│   ├── graphrag-common/            # 公共
│   ├── graphrag-input/             # 输入 loader
│   ├── graphrag-llm/               # LLM 抽象 (含 metrics/middleware/types/cache)
│   ├── graphrag-storage/           # 存储
│   └── graphrag-vectors/           # 向量
├── unified-search-app/             # 独立 Demo 全栈 App (Streamlit + RAG)
├── docs/                           # 完整 mkdocs 文档
│   ├── config/ data/ get_started.md
│   ├── index/ prompt_tuning/ query/
│   ├── examples_notebooks/
│   └── visualization_guide.md
├── tests/                          # 测试
├── breaking-changes.md             # 版本不兼容变更显式管理
├── mkdocs.yaml / pyproject.toml / uv.lock
└── README.md
```

**8 sub-package 模块化路径**:cache / chunking / common / input / llm / storage / vectors + 主包。**比 LlamaIndex 23 类别精简**,但比 RAGFlow 整体打包**更分散**(每个 sub-package 独立 PyPI 可单用)。

---

## 2. `packages/graphrag/graphrag/` — 主包 14 模块 [verified]

```
packages/graphrag/graphrag/
├── api/                            # 公共 API
├── cache/                          # 缓存
├── callbacks/                      # 回调
├── cli/                            # 命令行
├── config/                         # 配置
├── data_model/                     # ⭐ KG 数据模型 (entity/relationship/community/...)
├── graphs/                         # 图结构
├── index/                          # ⭐ 索引构建 (21 workflow + 13 operation)
├── logger/                         # 日志
├── prompt_tune/                    # ⭐ Prompt 自动调优
├── prompts/                        # Prompt 模板
├── query/                          # ⭐ 4 种检索模式
├── tokenizer/                      # Token 计数
└── utils/                          # 工具
```

---

## 3. 核心创新:`query/structured_search/` 4 种检索模式 [code: packages/graphrag/graphrag/query/structured_search/]

```
structured_search/
├── basic_search/      # vanilla RAG fallback
├── local_search/      # entity 邻域 hop (具体事实)
├── global_search/     # community-level 汇总 (主题摘要)
├── drift_search/      # ⭐ DRIFT (Dynamic Reasoning and Inference with Flexible Traversal)
└── base.py            # 基类
```

**这是 GraphRAG 业内独家创新点**:不只是 "KG retrieval",而是**4 种检索粒度策略让用户按 query 性质选**:

| 模式 | 适合 query | 工程实现 |
|---|---|---|
| **basic_search** | 简单事实查询 | Vanilla RAG fallback |
| **local_search** | "What does X do?" 具体实体邻域 | Entity → relationship → entity neighborhood hop |
| **global_search** | "What are the main themes?" 跨数据集主题 | Community-level summarization 汇总 |
| **drift_search** | 复杂混合 query | 先 community context + 再 local refinement 动态混合(Microsoft 新创新) |

**对照** LlamaIndex 18+ Query Engine 是 RAG composition primitive,Microsoft GraphRAG 4 search mode 是 **KG-specific retrieval strategy** — 不同抽象层。

---

## 4. `data_model/` KG 数据模型 [code: packages/graphrag/graphrag/data_model/]

```
community_report.py       # ⭐ Community 报告 (KG-level summary primitive)
community.py              # ⭐ Community (聚类节点)
covariate.py              # Covariate (协变量/上下文属性)
data_reader.py
document.py               # Document (原始文档)
entity.py                 # ⭐ Entity (KG 节点)
identified.py + named.py  # 基类: 有 ID / 有 name 的对象
relationship.py           # ⭐ Relationship (KG 边)
schemas.py
text_unit.py              # Text unit (chunk)
types.py
row_transformers.py + dfs.py
```

**核心 KG schema 5 类型** + **community report** 作为中间 KG-summary primitive:
- 原始 layer:Document → TextUnit
- 抽取 layer:Entity / Relationship / Covariate
- 聚合 layer:Community → CommunityReport

**Community + CommunityReport 是 Microsoft GraphRAG 跟其他 KG RAG 的关键差异**:不是 entity-level 简单图谱,**而是社区检测 + LLM 自动写社区摘要报告**。Global search 直接查 community reports,绕过 entity-level 查询。

---

## 5. `index/workflows/` — 21 Indexing Pipeline [code: packages/graphrag/graphrag/index/workflows/]

按 phase 分组:

**Create phase**(7 workflow):
```
create_base_text_units.py        # 切分基础 chunk
create_communities.py            # 社区检测
create_community_reports.py + create_community_reports_text.py  # 社区报告生成
create_final_documents.py
create_final_text_units.py
finalize_graph.py
```

**Extract phase**(3 workflow):
```
extract_covariates.py            # 抽 covariate (协变量)
extract_graph.py                 # 抽 entity + relationship
extract_graph_nlp.py             # NLP 辅助抽图
```

**Update phase**(8 workflow,增量更新):
```
update_clean_state.py
update_communities.py
update_community_reports.py
update_covariates.py
update_entities_relationships.py
update_final_documents.py
update_text_embeddings.py
update_text_units.py
```

**Other**(3 workflow):
```
factory.py + generate_text_embeddings.py + load_input_documents.py + load_update_documents.py + prune_graph.py
```

**21 个 workflow → 完整 indexing pipeline,每个 stage 独立可观测**。这是 Microsoft 论文 to production 的工程化 — 把 "build KG → cluster → summarize" 3 步抽成 21 个细粒度 workflow。

---

## 6. `index/operations/` — 13 核心算子 [code: packages/graphrag/graphrag/index/operations/]

```
build_noun_graph/             # 名词图谱构建
cluster_graph.py              # ⭐ Leiden / Louvain 风社区聚类
compute_edge_combined_degree.py
embed_text/
extract_covariates/
extract_graph/                # entity + relationship 抽取
finalize_community_reports.py
finalize_entities.py
finalize_relationships.py
prune_graph.py
snapshot_graphml.py           # 导出 GraphML 标准格式
summarize_communities/        # ⭐ LLM 写社区摘要 (核心创新)
summarize_descriptions/       # 实体 description 摘要
```

**`cluster_graph.py` + `summarize_communities/`** 是 Microsoft GraphRAG 核心创新 2 个算子:
1. **Cluster graph** 把抽出来的 entity-relationship 图做社区检测(Leiden algorithm 用 graspologic / networkx)
2. **Summarize communities** 让 LLM 给每个社区写自然语言摘要(community report),作为 global_search 的检索目标

---

## 7. `prompt_tune/` — 自动 Prompt 调优 [code: packages/graphrag/graphrag/prompt_tune/]

```
defaults.py
generator/
loader/
prompt/
template/
types.py
```

**业内独家自带 prompt tuning system**(README 强烈推荐:_"We strongly recommend to fine-tune your prompts following the Prompt Tuning Guide"_)— 因为 KG 抽取质量极依赖 prompt,Microsoft 把这套自动化进框架。

**对照** SkillForge 自家 PROMPT-OPTIMIZE 路线 / DSPy 15 optimizer / Opik 7 algorithm:Microsoft GraphRAG `prompt_tune` 是 task-specific(只针对 GraphRAG 抽 entity/relationship/community 这套 prompt),不是通用 prompt optimizer。

---

## 8. 跟其他 KG RAG 框架对照

| | Microsoft GraphRAG | LightRAG (HKUDS) | RAGFlow `graphrag/general+light` | LlamaIndex Property Graph |
|---|---|---|---|---|
| **核心创新** | 4 search mode (drift_search 最新)+ Community report | Simple + Fast dual-level retrieval | 两流派内置(general 抄 MS / light 抄 LightRAG) | Property Graph 作为 18+ index 类型之一 |
| **学术 backbone** | arxiv 2404.16130 | EMNLP2025 | (集成,不自家原创) | (集成,不自家原创) |
| **stars** | (流行度高 demo)| 27-35k | (80.6k 主仓库) | (46.5k 主仓库) |
| **License** | MIT | MIT | Apache 2.0 | MIT |
| **Stack** | 8 sub-package Python | 单包 Python | Python + Go | Python |
| **Maintenance** | Microsoft Research | HKUDS 学术组 | InfiniFlow 公司 | RunLlama 公司 |
| **Production-ready** | "Not official Microsoft product"(demo)| 学术 demo | ✅ 产品 | ✅ 产品 |

⚠️ **关键 calibration**:_"Microsoft GraphRAG 是 paradigm 起点,但 README 明确不是 officially supported product"_。生产用 RAGFlow / LlamaIndex 集成或自己 fork 更靠谱。

---

## 9. 关键 verified 锚点(总结)

| 路径 | 主题 |
|---|---|
| `packages/graphrag/graphrag/query/structured_search/{basic,local,global,drift}_search/` | **4 search mode 核心创新** |
| `packages/graphrag/graphrag/data_model/{entity,relationship,community,community_report,covariate,text_unit,document}.py` | KG schema 7 类型 |
| `packages/graphrag/graphrag/index/workflows/{create,extract,update}_*.py` | 21 indexing workflow |
| `packages/graphrag/graphrag/index/operations/cluster_graph.py` | Leiden 社区检测 |
| `packages/graphrag/graphrag/index/operations/summarize_communities/` | LLM 社区报告生成 |
| `packages/graphrag/graphrag/index/operations/extract_graph/` | entity + relationship 抽取 |
| `packages/graphrag/graphrag/prompt_tune/` | 自动 prompt 调优 |
| `packages/graphrag/graphrag/query/context_builder/{community,local,entity_extraction,conversation_history,dynamic_community_selection}.py` | Context builder 多策略 |
| `unified-search-app/` | Streamlit 演示 app |
| `arxiv.org/pdf/2404.16130` | 学术 backbone |

---

## 10. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/ms-graphrag/](file:///Users/youren/myspace/ms-graphrag/) 31 MB
- GitHub: `microsoft/graphrag` / MIT(non-supported demo)
- **Arxiv**:[2404.16130](https://arxiv.org/pdf/2404.16130) — KG RAG paradigm 论文
- Microsoft Research Blog:_"GraphRAG: Unlocking LLM discovery on narrative private data"_
- 衍生 wiki:[[eval/ms-graphrag]] / [[concept/rag-design-space]]
