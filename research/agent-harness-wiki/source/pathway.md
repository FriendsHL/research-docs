---
name: pathway
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: pathwaycom/pathway
version: dynamic (semversioner)
commit: 510c6daf92bd76e21b7279cee5f588b1b64e7b0f
license: BSL 1.1 (NavAlgo SAS) - 非完全 OSS
stars: 50k+
last_ingested_at: 2026-05-17
contributed_to:
  - "[[eval/pathway]]"
  - "[[concept/rag-design-space]]"
---

# Source — Pathway @ commit 510c6daf

> pathwaycom/pathway — **"Live Data Framework"** — Python ETL framework for streaming + LLM/RAG。**Rust engine + Differential Dataflow**(Microsoft Research Naiad 论文级)+ Python API。**BSL 1.1 license**(NavAlgo SAS,非完全 OSS,Business Source License)。clone @ `/Users/youren/myspace/pathway/` 323 MB。

---

## 1. Top-level 仓库结构

```
pathway/
├── python/pathway/        # ⭐ Python API
├── src/                   # ⭐ Rust 核心引擎
├── external/
│   └── differential-dataflow/  # ⭐ Frank McSherry's Differential Dataflow (vendored)
├── integration_tests/     # ⭐ 14+ category 集成测试 (含 rag_evals)
├── examples/              # 示例
├── docs/                  # 文档
├── tests/                 # 单测
├── build.rs / Cargo.toml / Cargo.lock / rust-toolchain.toml  # Rust 配置
├── pyproject.toml         # Python 配置 (uv-friendly)
├── LICENSE.txt            # BSL 1.1
└── README.md
```

**核心架构特征**:**Python frontend + Rust engine + Differential Dataflow base**。Python 写 pipeline,实际计算在 Rust;Differential Dataflow 提供 _incremental compute_(数据变化只重算 delta)。

---

## 2. `python/pathway/` Python 包结构

```
python/pathway/
├── __init__.py
├── __main__.py
├── _engine_finder.py         # Rust engine 发现 + 加载
├── cli.py
├── debug/
├── demo/
├── engine.pyi                # ⭐ Rust engine Python type stubs
├── internals/                # 内部 API
├── io/                       # I/O connector
├── persistence/              # 持久化
├── reducers.py
├── schema.py
├── stdlib/                   # ⭐ 9 子模块 (graphs/indexing/ml/ordered/stateful/statistical/temporal/utils/viz)
├── tests/
├── third_party/
├── udfs.py                   # User-defined function
├── universes.py
├── web_dashboard/            # 自家 Web dashboard
└── xpacks/                   # ⭐ 扩展包 (LLM/RAG 在这里)
```

---

## 3. `python/pathway/xpacks/llm/` — LLM/RAG xpack 完整工具栈 [verified]

```
xpacks/llm/
├── document_store.py        # ⭐ Document store
├── embedders.py             # Embedding wrapper
├── llms.py                  # LLM wrapper
├── mcp_server.py            # ⭐ MCP server 内置
├── parsers.py               # Document parser
├── prompts.py               # Prompt templates
├── question_answering.py    # ⭐ QA 主入口
├── rerankers.py             # Rerank
├── servers.py               # REST/WS server
├── splitters.py             # Text splitter
├── vector_store.py          # ⭐ Vector store
├── constants.py
├── README.md
├── tests/
└── _parser_utils.py + _typing.py + _utils.py + utils.py
```

**完整 RAG 工具栈 in single xpack**:11 个核心组件覆盖 ingestion(parsers/splitters)→ indexing(embedders/vector_store/document_store)→ retrieval(rerankers)→ generation(llms/prompts/question_answering)→ serving(servers/mcp_server)。

**MCP server 内置**(`mcp_server.py`)— Pathway-built RAG pipeline 可直接对外作为 MCP server,被任何 agent 调用。

---

## 4. `python/pathway/stdlib/` — 标准库 9 子模块 [verified]

```
stdlib/
├── graphs/         # 图算法 (shortest path / pagerank 等)
├── indexing/       # 索引 (用于 RAG)
├── ml/             # ML 操作
├── ordered/        # 有序数据
├── stateful/       # 有状态计算
├── statistical/    # ⭐ 统计 (时序窗口聚合等)
├── temporal/       # ⭐ 时序 (Tumbling window / Session window / Sliding window)
├── utils/
└── viz/            # 可视化
```

**`temporal/` + `statistical/` 双时序 first-class** — 业内独家把"时间窗口聚合 + 统计算子"作为 stdlib 一等模块(对照 SkillForge 自家 trace / metric 时序需求)。

---

## 5. `src/` Rust 核心引擎 [verified]

```
src/
├── async_runtime.rs        # Async runtime
├── connectors/             # ⭐ Connector (Kafka/Iceberg/Postgres/...)
├── deepcopy.rs
├── engine/                 # ⭐ 核心 engine
├── env.rs
├── external_integration/   # 外部集成
├── fs_helpers.rs
├── lib.rs
├── mat_mul.rs              # 矩阵乘 (ML 用)
├── persistence/            # 持久化
├── pipe.rs
├── python_api/             # Python binding
├── python_api.rs
├── retry.rs
└── timestamp.rs            # ⭐ 时间戳 (核心 differential dataflow 概念)
```

**`timestamp.rs`** 是 Differential Dataflow 核心概念 — 每条数据带时间戳,引擎用时间戳计算 delta(增量)而非全量重算。

---

## 6. `external/differential-dataflow/` — vendored 业内顶级 incremental compute 库

**Frank McSherry's Differential Dataflow** 是 Microsoft Research **Naiad 论文**(NSDI 2013)的开源实现。核心思想:

```
传统 batch:  Input → Pipeline → Output (整批重算)
Differential: Input + delta → Pipeline → Output + delta (只算变化)
```

**对 RAG 的意义**:
- 文档库变化(新增 / 删除 / 修改)只重算受影响的 chunk + embedding + index
- 实时 indexing 不需要重 build 整个 KG
- 用户 query 历史变化只更新 cache 受影响部分

**业内独家**:大部分 RAG framework(LlamaIndex / Dify / RAGFlow)用 standard batch + manual incremental code,Pathway 是 **framework-level differential dataflow** 直接提供原生增量 — 这是 Pathway 50k stars 的真正护城河。

---

## 7. `integration_tests/` — 14+ category 集成测试 [verified]

```
integration_tests/
├── airbyte/         # Airbyte connector
├── common/
├── db_connectors/   # 数据库连接
├── gdrive/          # Google Drive
├── iceberg/         # Apache Iceberg
├── kafka/           # ⭐ Kafka 流
├── license/         # BSL license enforcement
├── monitoring/
├── rag_evals/       # ⭐ RAG eval 集成
├── s3/
├── sharepoint/      # ⭐ SharePoint
├── stdlib/
├── webserver/
├── wordcount/
└── xpack/
```

**Real connector breadth**:Airbyte / Kafka / Iceberg / Postgres / S3 / SharePoint / GDrive — 数据源覆盖企业级常见。

---

## 8. README 推荐 RAG templates(verified from URLs)

- _Unstructured data to SQL on-the-fly_
- _Private RAG with Ollama and Mistral AI_
- **_Adaptive RAG_** (业内主流"adaptive-rag" 模式实现)
- **_Multimodal RAG with gpt-4o_**

**对照** [[eval/llama_index]] 18+ Query Engine / [[harness/dify]] 14 RAG submodule:Pathway 不提供 18+ pattern abstraction,**而是给 4 个完整 production template** 让用户起步。

---

## 9. 关键 verified 锚点(总结)

| 路径 | 主题 |
|---|---|
| `python/pathway/xpacks/llm/` | 11 文件完整 RAG 工具栈 + MCP server |
| `python/pathway/stdlib/{temporal,statistical}/` | 时序 + 统计 first-class |
| `src/timestamp.rs` | Differential Dataflow 时间戳核心 |
| `src/engine/` | Rust engine |
| `external/differential-dataflow/` | vendored Microsoft Research Naiad 论文实现 |
| `integration_tests/rag_evals/` | RAG eval 集成测试 |
| `integration_tests/{kafka,iceberg,sharepoint,gdrive}/` | 企业级 connector 测试 |
| `python/pathway/web_dashboard/` | 自家 Web dashboard |
| `LICENSE.txt` | **BSL 1.1**(NavAlgo SAS)非完全 OSS,商用需 license |

---

## 10. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/pathway/](file:///Users/youren/myspace/pathway/) 323 MB
- GitHub: `pathwaycom/pathway` 50k+ stars
- 官网:pathway.com / 文档:pathway.com/developers
- **BSL 1.1 注意**:不是 Apache/MIT,商用需 NavAlgo 授权
- Differential Dataflow:Microsoft Research Naiad 论文 NSDI 2013
- 衍生 wiki:[[eval/pathway]] / [[concept/rag-design-space]]
