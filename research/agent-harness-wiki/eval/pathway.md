---
name: pathway
type: eval
status: active
language: python+rust
last_ingested_version: dynamic
last_ingested_at: 2026-05-17
last_commit: 510c6daf92bd76e21b7279cee5f588b1b64e7b0f
community_known_newer: false
license_note: "BSL 1.1 — 商用需 NavAlgo 授权"
known_post_ingest_features: []

sources:
  - "[[source/pathway]]"
  - "/Users/youren/myspace/pathway/"

agent_loop: |
  无 agent loop. Pathway 是 stream processing + LLM/RAG ETL framework, 不是 agent runtime.
  集成方式: 用户 agent (LangChain/AutoGen/Anthropic) 调用 Pathway-built RAG pipeline 作为 tool / MCP server.

tool_protocol: |
  ⭐ MCP server 内置 (`python/pathway/xpacks/llm/mcp_server.py`) — RAG pipeline 直接对外作为 MCP server.
  跟 [[source/cognee]] `cognee-mcp/`, [[source/mempalace]] 29 MCP tools 同思路.

context_mgmt: |
  N/A — Pathway 是 ETL/RAG 后端, 不管 agent context.

memory: |
  Pathway memory = vector_store + document_store (`xpacks/llm/`).
  关键差异: ⭐ **Differential Dataflow 增量计算** — 文档库变化时只重算 delta, 不重 build.
  这是 pathway 50k stars 真正护城河.

multi_agent: |
  N/A.

eval_loop: |
  无内置 eval framework. `integration_tests/rag_evals/` 测试套件 (跟 Ragas/Promptfoo 集成).

self_evolution: |
  Differential Dataflow 提供 incremental compute, 数据变化自动反映到 RAG output, 但不是"agent self-evolution"意义上.

observability: |
  `python/pathway/web_dashboard/` 自家 dashboard.
  `integration_tests/monitoring/` 监控集成.

hooks: |
  Pipeline-level hooks (callback / connector lifecycle).

safety: |
  License BSL 1.1 enforcement (integration_tests/license/).
  没有 sandbox / moderation 独立模块.

provider: |
  `xpacks/llm/llms.py` 多 LLM provider wrapper (OpenAI / Anthropic / Ollama / Mistral / ...).
  Connector 层覆盖 Kafka / Iceberg / Postgres / S3 / SharePoint / GDrive / Airbyte.

channels: |
  Python library + REST server (`xpacks/llm/servers.py`) + MCP server + Web dashboard + CLI (cli.py).

persistence: |
  `src/persistence/` Rust 持久化层 + `python/pathway/persistence/` Python wrapper.
  Snapshot + replay 支持(streaming first-class).

cancellation: |
  Async runtime cancellation (`src/async_runtime.rs`), pipeline-level.

skillforge_takeaway: |
  Pathway 路线 (streaming + incremental + Rust engine) 跟 SkillForge (Java + 业务 SaaS) **不直接重叠**,
  但有 3 条值得参考的高价值点.

  🟡 **R-PW-1 — Differential Dataflow 增量计算思想**(memory dedup / dream consolidation 灵感)
     SkillForge MEMORY-DREAM-CONSOLIDATION 当前是定期跑 dedup batch (`MemoryConsolidator.java`),
     如果 memory 变化频繁 (high-write workload), 可借鉴 differential dataflow "只算 delta" 思想.
     工作量: 不直接借代码 (Rust + License 限制), 借设计思想 — < 1 周 design 评估.

  🟡 **R-PW-2 — Temporal + Statistical stdlib 设计**(SkillForge trace / metric 时序)
     `python/pathway/stdlib/{temporal,statistical}/` 把时间窗口聚合作为 stdlib 一等模块.
     SkillForge OBS-2 LlmTraceEntity / 长 session 统计需要时序聚合 (P50/P95/P99 latency, error rate window),
     当前可能在 dashboard 层 ad-hoc 算, 远期可参考 Pathway 抽象成 framework primitive.
     工作量: 远期, 取决于 SkillForge 可观测复杂度.

  🟢 **R-PW-3 — RAG xpack as MCP server pattern**(参考)
     `xpacks/llm/mcp_server.py` — Pathway-built RAG pipeline 直接对外作为 MCP server.
     SkillForge skill marketplace 可参考 "把 skill 包装成 MCP server" 让其他 agent 调用模式.
     跟 [[source/cognee]] R-COG-6 同思路.
     工作量: < 半周设计.

  **❌ 不该跟**:
  - Rust + Differential Dataflow 自己实现 — Java/Spring 栈不兼容,License BSL 1.1 商用受限
  - 9 stdlib 模块全套 — SkillForge 不需要 stream processing framework
  - Adaptive RAG / Multimodal RAG template — 直接用 Pathway 集成 (不自家重写)

  **总结**: Pathway 是 6 个 RAG ingest 里 SkillForge **最远的参考** —
  设计思想可借 (incremental / temporal), 代码不可移植 (语言 + license).
---

# Eval — Pathway (Live Data Framework)

> Pathway 是 **Python + Rust + Differential Dataflow** 的 streaming/ETL/RAG framework(BSL 1.1,NavAlgo SAS,50k+ stars)。本次 ingest 13 维分析 + 跟 SkillForge 的 3 条参考价值点。**关键 caveat**:BSL 1.1 license,商用需 NavAlgo 授权。

---

## 1. 项目定位(一句话)

**"Live Data Framework"** — Python 写、Rust 跑(Differential Dataflow 引擎),用一份 code 处理 batch + streaming,内置 LLM/RAG xpack。**业内 streaming-RAG 唯一框架**(其他都是 batch + 手工增量)。

---

## 2. 为什么 Pathway 对 RAG 设计空间重要

业内 RAG framework 默认假设是 **"文档库是静态的"**:Index 一次,query 多次。但 production 场景:
- 文档库每天新增(论文 / 新闻 / customer ticket / Slack 消息)
- 文档库经常修改(wiki page / 政策更新)
- 文档库偶尔删除(GDPR / 过期数据)

传统 RAG framework 的做法:**定期重 build 整个 index**(慢、贵、有 gap)。
Pathway 做法:**Differential Dataflow 增量计算** — 文档变化只重算 delta,index 实时反映。

这是 Pathway 50k stars 真正护城河,**不是营销话术**,是 Rust + Frank McSherry 的 Naiad 论文(NSDI 2013)级 incremental compute 基建。

---

## 3. 核心技术 — Differential Dataflow

```
传统 RAG:
  Day 1: Index(全库)   → 1h
  Day 2: 加 100 文档    → Index(全库)→ 1h (浪费)
  Day 3: 删 5 文档      → Index(全库)→ 1h (浪费)

Pathway:
  Day 1: Index(全库)   → 1h
  Day 2: 加 100 文档    → ΔIndex(100 文档 + 受影响 chunk) → 1 min
  Day 3: 删 5 文档      → ΔIndex(invalidate 5 文档 chain) → 30s
```

**实现**:
- `src/timestamp.rs` — 每条数据带 logical timestamp
- `external/differential-dataflow/` — Microsoft Research vendored
- 文档 → chunk → embedding → vector_store 整个链路 incremental
- 用户 query 也 cache + delta 更新

**生产价值**:每天 sync Notion/Slack/GDrive 数据的 enterprise RAG,不需要每天重 build 索引。

---

## 4. xpacks/llm 完整 RAG 工具栈

11 个核心组件覆盖完整 RAG pipeline(详见 [[source/pathway]] §3):

```
Ingestion:    parsers.py + splitters.py
Indexing:     embedders.py + vector_store.py + document_store.py
Retrieval:    rerankers.py
Generation:   llms.py + prompts.py + question_answering.py
Serving:      servers.py (REST/WS) + mcp_server.py
```

**比 LlamaIndex/Dify 更小 但更紧凑** — 因为 Pathway 强调 "ETL framework first,RAG xpack second"。

---

## 5. 跟其他 5 个 RAG ingest 项目对照

| | Pathway | Dify | LlamaIndex | RAGFlow | MS GraphRAG | Cognee |
|---|---|---|---|---|---|---|
| **核心定位** | Streaming ETL + RAG xpack | Production AI Platform | RAG framework 标杆 | Deep doc understanding RAG | KG paradigm 论文实现 | Memory control plane |
| **Indexing 策略** | ⭐ Differential Dataflow 增量 | Batch | Batch | Batch + 21 workflow update 路径 | Batch | Batch + memify pipeline |
| **Language stack** | ⭐ Python + Rust | Python (Flask) | Python | Python + Go | Python | Python |
| **License** | ⚠️ BSL 1.1 | Apache 2.0 + 商业限制 | MIT | Apache 2.0 | MIT | Apache 2.0 |
| **MCP server** | ✅(`mcp_server.py`) | partial | partial | ✅ | ❌ | ✅(独立子项目) |
| **Stars** | 50k+ | 114k+ | 46.5k | 80.6k | popular demo | growing |
| **Stream RAG** | ⭐ 唯一 | ❌ | ❌ | partial(2025-10 ingestion pipeline)| ❌ | partial(sync module) |

**结论**:Pathway 是 6 个 RAG ingest 里**唯一 streaming-first**。其他都是 batch + 增量补丁。

---

## 6. R-PW 借鉴清单

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-PW-1** | Differential Dataflow **设计思想**(memory dedup 增量化)| < 1 周 design 评估 | 中 — 写多场景受益 | 🟡 P2 |
| **R-PW-2** | Temporal + Statistical stdlib 设计 | 远期 | 中 — trace/metric 时序 | 🟡 P2 |
| **R-PW-3** | RAG xpack as MCP server pattern | < 半周 | 中 — skill marketplace 模式 | 🟢 P3 |

**总结**:Pathway 是 6 个 RAG ingest 里 SkillForge **最远的参考** — 设计思想可借(incremental / temporal),代码不可移植(Rust + BSL 1.1)。

---

## 7. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[concept/rag-design-space]] | **streaming RAG 维度独家贡献者** | Differential Dataflow incremental compute |
| [[harness/dify]] / [[eval/llama_index]] / [[harness/ragflow]] / [[eval/ms-graphrag]] / [[eval/cognee]] | 互补对照 | 6 RAG framework 全景里 streaming 角度补全 |
| [[concept/observability]] | 远期对照 | Temporal/statistical stdlib 给 SkillForge OBS metric/trace 时序聚合参考 |
| [[concept/agent-memory]] | 远期对照 | Differential Dataflow 思想对 memory dedup pipeline 启示 |

---

## 8. 已知 post-ingest 风险

- BSL 1.1 license 商用限制未深 verify 具体条款
- Rust engine 实际 API surface 跟 Python API 一一对应程度未 verify
- `web_dashboard/` 实际能力未试运行
- 4 RAG template(Adaptive / Multimodal / Private / Unstructured-to-SQL)只看 URL,未深读实现
- `differential-dataflow/` vendored 版本是否最新未 verify(Frank McSherry 主仓库可能更新)

---

## 9. 引用

- [[source/pathway]] 完整源码锚点
- GitHub: `pathwaycom/pathway` 50k+ stars / **BSL 1.1**
- 官网:pathway.com / docs:pathway.com/developers
- Naiad paper(NSDI 2013):Differential Dataflow theoretical base
- 跨页对照:[[harness/dify]] / [[eval/llama_index]] / [[harness/ragflow]] / [[eval/ms-graphrag]] / [[eval/cognee]] / [[concept/rag-design-space]]
