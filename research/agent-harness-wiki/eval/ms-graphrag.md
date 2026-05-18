---
name: ms-graphrag
type: eval
status: active
language: python
last_ingested_version: latest @ commit de531f0a
last_ingested_at: 2026-05-17
last_commit: de531f0a697d2f35c4f85cb8511141507278404e
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/ms-graphrag]]"
  - "https://arxiv.org/pdf/2404.16130"
  - "/Users/youren/myspace/ms-graphrag/"

agent_loop: |
  无 agent loop. Microsoft GraphRAG 是 data pipeline (indexing + query),不是 agent 框架.
  通过 CLI / Python API 调用, 跟 agent harness 集成由用户实现.

tool_protocol: |
  不直接提供 tool 协议. Query API 返回 string answer + citation, 由用户 wrap 成 tool 给 agent.

context_mgmt: |
  无 chat / context. 单 query → 单 answer 模式.
  Community report (社区摘要) 是 KG-level context primitive — 取代传统 chunk-level context.

memory: |
  KG-as-memory 路线. Indexing 阶段把所有文档抽出 entity + relationship + community,
  存为 GraphML / parquet (storage 8 sub-package). Query 时按 4 mode 检索.
  Cross-session 不 evolve (一次性 index, 增量 update 走 21 workflow 的 update_* 部分).

multi_agent: |
  无.

eval_loop: |
  README 提到 evaluation in docs, 但没看到内置 eval framework.
  生产用户跑 Ragas / Promptfoo / 自家 benchmark 在 GraphRAG 之上.

self_evolution: |
  `prompt_tune/` 内置自动 prompt 调优 (defaults / generator / loader / prompt / template),
  针对 GraphRAG 抽 entity/relationship/community 这套 prompt 自动 fine-tune.
  跟 DSPy 15 optimizer / Opik 7 algorithm 不同 — Microsoft GraphRAG 是 task-specific tuning.

observability: |
  `callbacks/` + `logger/` 完整 logging system.
  Indexing pipeline 21 workflow 每步可观测 (factory.py 管理 workflow 注册).

hooks: |
  Indexing pipeline workflow-level hook (callbacks/).

safety: |
  README 含 `RAI_TRANSPARENCY.md` (Responsible AI Transparency) — Microsoft 标配 RAI 评估.

provider: |
  `graphrag-llm` sub-package 独立 LLM 抽象 + middleware + metrics + cache.
  支持 OpenAI / Azure OpenAI 系 (主), 其他 provider 用户自配.

channels: |
  CLI (`__main__.py`) + Python API + `unified-search-app/` Streamlit demo app.
  不做 SaaS / web UI 服务.

persistence: |
  `graphrag-storage` sub-package — parquet / GraphML / 文件系统.
  支持 CosmosDB (Azure) — commit de531f0a 新增 `CosmosTableProvider with namespace partitioning`.

cancellation: |
  ? 未深 verify.

skillforge_takeaway: |
  Microsoft GraphRAG 是 **KG RAG paradigm 起点论文实现**.
  对 SkillForge 是"参考 + 警示"两层意义.

  🟡 **R-GR-1 — 4 Search Mode 抽象**(KG RAG 核心创新)
     `query/structured_search/{basic,local,global,drift}_search/` 4 种检索粒度策略
     (basic vanilla / local entity neighborhood / global community summary / drift hybrid).
     SkillForge 如果走 KG memory v3, 这 4 mode 是 retrieval 设计的 reference.
     drift_search 是 Microsoft 2025 新创新, dynamic reasoning over flexible traversal.
     工作量: 远期参考, 不直接打到生产.

  🟡 **R-GR-2 — Community Report Primitive**(KG-summary intermediate)
     `data_model/community_report.py` + `index/operations/summarize_communities/` —
     不是简单 entity-level KG, 而是 **community detection (Leiden) + LLM 自动写社区摘要报告**.
     Global search 直接查 community reports, 绕过 entity-level.
     SkillForge ATTRIBUTION-AGENT 因果链有 graph 雏形, 远期升级 v3 可参考此抽象层.

  🟡 **R-GR-3 — 21 Workflow + 13 Operation 细粒度 Indexing Pipeline**(工程化)
     `index/workflows/{create,extract,update}_*.py` 21 个 + `index/operations/` 13 个.
     每步独立可观测 + 增量更新 (update_*). SkillForge skill marketplace 入库流程可参考.
     工作量: 远期, 不直接打.

  🟢 **R-GR-4 — Task-specific Prompt Tuning System**(参考)
     `prompt_tune/` 自动给 entity/relationship/community 抽取 prompt 调优.
     SkillForge 自家 PROMPT-OPTIMIZE 路线 (V?? 待 verify) 可借鉴此 task-specific 思路.
     vs DSPy 15 optimizer / Opik 7 algorithm 是通用优化, GraphRAG 是垂直任务专门优化.

  **⚠️ R-GR-CALIBRATION — KG RAG 未必更好**
     ICLR'26 GraphRAG-Benchmark paper "When to use Graphs in RAG: A Comprehensive Analysis"
     明确警示: "GraphRAG frequently underperforms vanilla RAG on many real-world tasks."
     SkillForge **不要默认走 KG 路线**, 先在 vanilla RAG (好 chunking + reranker) 上做扎实再考虑 KG.
     Microsoft GraphRAG 是 paradigm reference, 不是 production recommendation.

  **❌ 不该跟**:
  - Microsoft 不 officially support — 不能 production 直接依赖
  - 8 sub-package modular 切分 — SkillForge monolith Spring 不需要
  - Community report LLM 摘要成本高 (论文都警告 indexing 贵) — SkillForge 短期不划算
---

# Eval — Microsoft GraphRAG @ commit de531f0a

> Microsoft GraphRAG 是 **KG RAG paradigm 起点论文实现**(arxiv 2404.16130),提供 4 种检索模式(basic / local / global / drift)和 community report 中间表示。Microsoft Research 出品但 **README 明确不是 officially supported product**(demonstration code)。

---

## 1. 项目定位(一句话)

**Microsoft Research demo + 论文实现**(MIT,arxiv 2404.16130),提出 _"hierarchical KG + community report + global/local 检索"_ 三件套,作为 LLM 处理 narrative private data 的 reference。

---

## 2. 为什么这个项目重要(尽管 demo 性质)

Microsoft GraphRAG 论文(arxiv 2404.16130)是 **KG RAG paradigm 的事实起点**:

| 维度 | 之前 RAG 主流 | GraphRAG 提出 |
|---|---|---|
| 数据表示 | flat chunk + vector | **Entity + Relationship + Community 三层图** |
| Indexing | embedding + index | **抽 entity → cluster community → LLM 写社区摘要报告** |
| Query | top-k chunk retrieval | **4 mode**:basic / local entity hop / **global community report 汇总** / drift 混合 |
| 适合 query | 具体事实 | **跨主题 / "What are the main themes?" 这种 holistic question** |
| 代价 | 单次 embedding cost | **Indexing 极贵**(每个 community 都 LLM 写一遍报告,大数据集很慢) |

**这个 paradigm 之后**:LightRAG / nano-graphrag / HippoRAG / RAGFlow's graphrag/light/general / LlamaIndex Property Graph 全都是这个 paradigm 的变体或简化。

**ICLR'26 calibration**(GraphRAG-Benchmark paper):_"GraphRAG frequently underperforms vanilla RAG on many real-world tasks."_ — 意思是 **paradigm 重要,但生产场景未必更好**。

---

## 3. 核心技术 — Community Report 是关键创新

```
Document
   │
   ▼
TextUnit (chunk)
   │
   ├─→ Entity (LLM extract)
   │      │
   │      └─→ Relationship (entity-to-entity)
   │              │
   │              ▼
   │       Cluster Graph (Leiden / Louvain)
   │              │
   │              ▼
   │       Community (聚类)
   │              │
   │              ▼ LLM
   │       ⭐ Community Report (LLM-written summary)
   │
   └─→ Covariate (上下文属性,可选)
```

**4 种 query**:
1. **basic_search**:vanilla RAG fallback,Chunk → vector → top-k
2. **local_search**:Question → Entity match → 邻域 hop → 上下文构建
3. **global_search**:Question → relevant Community Report → 汇总 → 答案
4. **drift_search**(2025 新):**Dynamic** Reasoning and **Inference** with **Flexible Traversal** — 先 community context 提供 high-level 上下文,再 local refinement 精化(Microsoft 自创概念)

**核心洞察**:Global search 跳过 entity-level,**直接查 Community Report**(LLM 预先写的社区自然语言摘要)。这让"What are the main themes?"这种 holistic question 变成简单文本检索。

---

## 4. 跟其他 KG RAG 框架对照(同 §8 of [[source/ms-graphrag]] 但视角不同)

跟当前 wiki 已 ingest 项目对照:

| | MS GraphRAG | LightRAG | RAGFlow `graphrag/` | LlamaIndex Property Graph | Zep+Graphiti |
|---|---|---|---|---|---|
| **抽象层** | Data pipeline | Same paradigm 简化 | 集成两风格 | KG 作为 18+ index 类型之一 | Temporal KG memory framework |
| **Community-level** | ✅ 核心创新 | ❌(简化掉)| general/ 流派含 | Sub-retrievers 可定制 | ❌ |
| **Drift / dynamic** | ✅ 2025 新 | ❌ | ❌ | ❌ | ❌ |
| **学术 backbone** | arxiv 2404.16130 | EMNLP2025 | (集成)| (集成)| arxiv 2501.13956 |
| **Production-ready** | ❌(demo)| ⚠️ 学术 demo | ✅ | ✅ | ✅(Cloud)|

---

## 5. R-GR 借鉴清单(详见 frontmatter)

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-GR-1** | 4 Search Mode 抽象(basic/local/global/drift)| 远期 | 中 — KG 路线 reference | 🟡 P2 |
| **R-GR-2** | Community Report Primitive | 远期 | 中 — ATTRIBUTION-AGENT v3 可借 | 🟡 P2 |
| **R-GR-3** | 21 Workflow + 13 Operation 细粒度 Indexing | 远期 | 中 — skill marketplace 入库 | 🟡 P2 |
| **R-GR-4** | Task-specific Prompt Tuning | 远期 | 中 — 自家 PROMPT-OPTIMIZE 路线借鉴 | 🟢 P3 |
| ⚠️ **R-GR-CALIBRATION** | **KG RAG 未必更好**(ICLR'26 paper)| — | **高 — 防 SkillForge 盲跟 KG 路线** | 🔴 必读 |

---

## 6. 跟现有 wiki 关系

| 当前页 | 关系 |
|---|---|
| [[eval/zep-graphiti]] | 互补 — Graphiti 是 Temporal KG memory(时间窗 + episode provenance),GraphRAG 是 hierarchical community KG 静态索引。两个不冲突,可叠加 |
| [[harness/ragflow]] | RAGFlow `graphrag/general/` 直接抄 MS GraphRAG 实现,**production-ready 版本** |
| [[eval/llama_index]] | LlamaIndex Property Graph 是 KG-as-index-type,跟 MS GraphRAG community-level 抽象层不同 |
| [[concept/rag-design-space]] | **关键贡献者** — 4 search mode + community report 是 KG RAG 设计空间核心 |
| [[concept/agent-memory]] | KG memory 路线参考,但 ICLR'26 paper 警示不要默认 KG |

---

## 7. 已知 post-ingest 风险

- `graphrag-llm` sub-package middleware/metrics/cache 未深 verify
- `prompt_tune/` 具体调优算法未深读
- `unified-search-app/` Streamlit demo 实际能力未试运行
- DRIFT search 论文/blog 未深读,只看到代码模块名
- `breaking-changes.md` 版本不兼容性未追溯

---

## 8. 引用

- [[source/ms-graphrag]] 完整源码锚点
- **Arxiv**:[2404.16130](https://arxiv.org/pdf/2404.16130) — "From Local to Global: A Graph RAG Approach to Query-Focused Summarization"
- ICLR'26 calibration:GraphRAG-Benchmark paper
- 跨页对照:[[eval/zep-graphiti]] / [[harness/ragflow]] / [[eval/llama_index]] / [[concept/rag-design-space]]
