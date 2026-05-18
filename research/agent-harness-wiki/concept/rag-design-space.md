---
name: rag-design-space
type: concept
status: B-mode source-grounded (depth-first synthesis)
updated: 2026-05-17
sources:
  - "[[harness/dify]]"
  - "[[eval/llama_index]]"
  - "[[harness/ragflow]]"
  - "[[eval/ms-graphrag]]"
  - "[[eval/cognee]]"
  - "[[eval/pathway]]"
  - "[[eval/ragas]]"
  - "[[concept/agent-memory]]"
---

# RAG Design Space — 6 项目横向 + 8 维设计空间 + ICLR'26 paper calibration

> **要点**:这页 synthesis 是 6 个 RAG ingest(Dify / LlamaIndex / RAGFlow / Microsoft GraphRAG / Cognee / Pathway)+ 1 个 paper calibration(ICLR'26 GraphRAG-Benchmark)的合成。Depth-first 写法:不堆 catalog,**讲 RAG 设计空间的 8 维**、**每个维度上业内做了什么 trade-off**、**ICLR'26 paper 警示**、**SkillForge memory v3 应该往哪走**。
>
> 一个关键反直觉发现:**SkillForge memory 不是"RAG over documents",是"RAG over agent state"** — 这意味着 80% 的 RAG framework design 对 SkillForge **不直接 applicable**,但**抽象层(retrieval pattern / KG / temporal)+ 工程(sandbox / incremental / feedback loop)**是直接借鉴的。

---

## 1. 写这页之前我们需要校准的 3 件事

### 1.1 SkillForge 的 RAG 不是普通 RAG

普通 RAG 框架(LlamaIndex / RAGFlow / Dify / Pathway / MS GraphRAG)的核心抽象是 **"Document → Chunk → Index → Retrieve → Synthesize"**,数据源是 _unstructured documents_。

SkillForge 是 **"agent state(MemoryEntity 三态 + SkillEntity + SubAgentRunEntity + LlmTraceEntity)→ structured records → Index → Retrieve → 注入 agent loop"**,数据源是 _agent 跑出来的 structured state_。

**所以**:6 个 RAG project 的 _"docling/mineru/paddleocr deep document understanding"_ 对 SkillForge 不直接 applicable(因为 SkillForge 不 ingest 客户的 PDF),但 **"hybrid retrieval / KG / temporal / feedback loop"** 在两种 RAG 上都成立。

### 1.2 ICLR'26 GraphRAG-Benchmark paper 警示

GraphRAG-Bench/GraphRAG-Benchmark ICLR'26 论文 _"When to use Graphs in RAG: A Comprehensive Analysis for Graph Retrieval-Augmented Generation"_ 明确警示:

> _"GraphRAG frequently underperforms vanilla RAG on many real-world tasks."_

**翻译成 SkillForge 决策语言**:
- **不要默认走 KG memory v3**
- 先把 vanilla RAG(好 chunking + 好 embedding + RRF rerank)做扎实
- KG memory 只在 _"用户 query 需要 multi-hop reasoning / 全局主题汇总"_ 时才需要
- SkillForge MemoryEntity 三态(observation/reflection/optimized)+ 当前 hybrid FTS+Vector+RRF 已经是 vanilla RAG 优秀实现

### 1.3 RAG framework 不必和 memory framework 二选一

很多人误以为 _"我用 mem0,就不用 LlamaIndex 了"_。事实是:
- **mem0/Letta/MemPalace/Cognee** 是 _agent memory framework_(管 entity / reflection / summarization / KG)
- **LlamaIndex/RAGFlow/Dify** 是 _RAG framework over documents_(管 doc → chunk → index)
- **两者协同**:Cognee 把 LlamaIndex 选为 graph_rag 唯一官方集成包(`llama-index-graph-rag-cognee`)就是证据

---

## 2. RAG 设计空间 8 维

每个维度上,**业内做了什么 trade-off** + **SkillForge 当前位置**。

### 2.1 维度 1 — Document Parsing(进入 RAG 之前的"数据清洗")

```
       ┌─ 浅:plain text / 直接 PDF → text ─────── 大部分 framework
       │
parsing │  ┌─ 中:专用 parser per format ───────── LlamaIndex 159 reader
       │  │
       └──┤  ┌─ 深:OCR + layout + table struct ── ⭐ RAGFlow (16 parser + 10 vision)
          │  │  + mineru + paddleocr + docling
          └──┘
```

**业内 trade-off**:深度 parsing 极贵(每 PDF 都跑 OCR + layout + LLM),但 production 用户数据是垃圾必须深。RAGFlow 80.6k stars 真正护城河就在这里。

**SkillForge 当前位置**:**N/A**(SkillForge 不 ingest 客户文档,memory 是 structured entity 不需要 parsing)。

### 2.2 维度 2 — Chunking Strategy

```
chunking ┌─ Fixed size ──────────────── LangChain default / LlamaIndex SentenceSplitter
        │
        ├─ Recursive / Semantic ────── LlamaIndex Semantic Splitter
        │
        ├─ Template by file type ───── ⭐ RAGFlow 15 模板 (book/paper/laws/resume/...)
        │
        └─ Tree-organized RAPTOR ───── LlamaIndex + RAGFlow rag/raptor.py
```

**业内 trade-off**:Fixed simple 但语义切碎,Template 准但工程多,RAPTOR 树形多层粒度。

**SkillForge 当前位置**:**N/A**(memory 是 entity 不切 chunk)。

### 2.3 维度 3 — Index 类型(怎么存)

```
index type ┌─ Vector (default) ─────────── 全员都有
          │
          ├─ Keyword / BM25 ──────────── LlamaIndex + Dify + RAGFlow
          │
          ├─ Hybrid (vector+keyword) ─── ⭐ Dify HYBRID_SEARCH / SkillForge MemoryService hybrid
          │
          ├─ Knowledge Graph ────────── ⭐ MS GraphRAG / Cognee / LlamaIndex Property Graph / RAGFlow graphrag 双流派
          │
          ├─ Tree (document_summary) ── LlamaIndex DocumentSummaryIndex / RAPTOR
          │
          └─ Temporal KG (validity windows) ── ⭐ Graphiti / Cognee temporal_graph
```

**业内 trade-off**:Vector 通用但语义有限,KG 强大但 indexing 极贵(MS GraphRAG paper 警告),Temporal KG 适合时间相关 query。

**SkillForge 当前位置** [verified-from-source]:**Hybrid (FTS + Vector + RRF)**(`MemoryService.java:853-908` hybridRecallL1 + rrfMerge,RRF_K=60)— **业内 Hybrid 主流配置**。⚠️ **ICLR'26 calibration:不必盲跟 KG**。

### 2.4 维度 4 — Retrieval Pattern(怎么取)

```
retrieval pattern ┌─ Top-K (single shot) ───── 全员默认
                 │
                 ├─ Sub-question decomposition ─ LlamaIndex SubQuestionQueryEngine / Cognee graph_completion_decomposition
                 │
                 ├─ Multi-step / iterative ──── LlamaIndex MultistepQueryEngine
                 │
                 ├─ Router (多 retriever 选) ── LlamaIndex RouterQueryEngine / Cognee 22 retriever
                 │
                 ├─ Citation-aware ──────────── LlamaIndex CitationQueryEngine / RAGFlow "grounded citations"
                 │
                 ├─ HyDE / Query transform ─── LlamaIndex TransformQueryEngine / FLARE
                 │
                 ├─ 4-mode KG (basic/local/    ⭐ MS GraphRAG 独家
                 │  global/drift)
                 │
                 ├─ 5-variant graph_completion ⭐ Cognee 独家
                 │  (基础/扩展/CoT/分解/摘要)
                 │
                 └─ Adaptive RAG ───────────── Pathway template
```

**业内 trade-off**:18+ pattern 显式建模(LlamaIndex / Cognee)vs 几个核心 pattern(Dify 4 retrieval method + RAGFlow 单一 hybrid)。SkillForge 路线:**显式抽象超前于实际 query 需求是过度工程**。

**SkillForge 当前位置** [verified-from-source]:**Top-K + Hybrid**(单 retrieval pattern)。**建议 V3**:加 sub-question decomposition + router 选 2-3 pattern,不抄 18+ 全套。

### 2.5 维度 5 — Reranking & Fusion(怎么排)

```
rerank ┌─ Model-based (Cohere/BGE) ─── 全员可集成
       │
       ├─ Weighted score fusion ───── ⭐ Dify WEIGHTED_SCORE + LlamaIndex
       │
       ├─ RRF (Reciprocal Rank Fusion) ⭐ SkillForge MemoryService:853-908 + 业内主流
       │
       ├─ Cross-encoder ────────────── ⭐ Zep+Graphiti / Cognee
       │
       └─ LLM-as-judge rerank ─────── LlamaIndex LLMRerank (慢但准)
```

**业内 trade-off**:RRF 简单稳定免训练,Cross-encoder 准但慢,LLM-as-judge 最准但贵。

**SkillForge 当前位置** [verified-from-source]:**RRF (RRF_K=60)**(`MemoryService.java:853-908`)— **业内 default 优秀实现**。

### 2.6 维度 6 — Indexing 时态(批 vs 增量 vs 流)

```
indexing time ┌─ Batch (一次性 build) ──────── 大部分 framework
             │
             ├─ Incremental batch ─────────── RAGFlow 21 workflow 含 update_*(2025-10 ship)
             │
             ├─ Streaming (event-driven) ── ⭐ Pathway 业内唯一 framework-level Differential Dataflow
             │
             └─ Hybrid (实时部分 + 批量后台) Cognee sync module + memify pipeline
```

**业内 trade-off**:Batch 简单贵,Streaming 复杂但实时。

**SkillForge 当前位置**:**Incremental batch**(MemoryEntity 写入即可见,DREAM-CONSOLIDATION 定期跑)— 跟 RAGFlow 同模式。**远期可借 R-PW-1 Differential Dataflow 设计思想**(memory dedup 高频时)。

### 2.7 维度 7 — Generation & Synthesis(怎么用 retrieval 结果)

```
synthesis ┌─ Compact mode (拼一起塞 LLM) ── 默认
         │
         ├─ Refine mode (多轮 refine) ──── LlamaIndex Refine
         │
         ├─ Tree summarize ────────────── LlamaIndex TreeSummarize
         │
         ├─ Accumulator (per chunk) ───── LlamaIndex Accumulate
         │
         ├─ Citation in answer ────────── ⭐ RAGFlow grounded citations + Phoenix Score.explanation
         │
         └─ Adaptive (按 query 复杂度选) ── Pathway template
```

**业内 trade-off**:Compact 简单丢 nuance,Tree summarize 准但贵,Citation 用户友好。

**SkillForge 当前位置**:**Compact** + memory 注入 system prompt(SkillForge memory 跟 RAG synthesis 不同 — memory 直接成为 LLM context 一部分,不需要 synthesis)。

### 2.8 维度 8 — Feedback Loop & Self-Improvement

```
feedback loop ┌─ ❌ 无(static index) ────── 大部分 RAG framework
             │
             ├─ 手动 reindex on data change Dify / RAGFlow / LlamaIndex
             │
             ├─ Memory dedup batch ────── ⭐ SkillForge DREAM-CONSOLIDATION (V66+)
             │
             ├─ User rating → memory weight Mem0 part / Letta part
             │
             └─ ⭐ Agent trace + user      ⭐ Cognee memify_pipelines 业内最完整闭环
               feedback → KG (auto)         (persist_agent_trace_feedbacks_in_KG +
                                            persist_sessions_in_KG + apply_feedback_weights)
```

**业内 trade-off**:Static index 简单 / 静态,Auto-feedback 闭环复杂但 self-improving。

**SkillForge 当前位置** [verified-from-source]:**Memory dedup batch + LLM-SYNTHESIS + 人审 gate**(中间档),**缺 agent trace + feedback 自动入 memory pipeline**(R-COG-1 P1 直接打到生产)。

---

## 3. 6 个 RAG project 8 维位置图

| | Dify | LlamaIndex | RAGFlow | MS GraphRAG | Cognee | Pathway |
|---|---|---|---|---|---|---|
| **1. 文档 parsing** | 浅(extractor)| 中(159 reader)| 🟢 深(16 parser + 10 vision)| 浅 | 中 | 中(parsers.py)|
| **2. Chunking** | 2 splitter | 多种 strategy | 🟢 15 模板按类型 | 1 种 | task-based | splitters.py |
| **3. Index 类型** | Hybrid + KG | 18+ 类型 + Property Graph | Hybrid + 双流派 GraphRAG | 🟢 KG paradigm 起点 | 🟢 Graph + Vector unified | Vector store |
| **4. Retrieval pattern** | 4 retrieval method | 🟢 18+ Query Engine | Hybrid + RAPTOR + tree | 🟢 4 mode (drift 独家) | 🟢 22 retriever | RAG xpack 简单 |
| **5. Reranking** | model + weighted | 多种 | rerank_model | 不强 | (graph 内嵌) | rerankers.py |
| **6. Indexing 时态** | Batch | Batch | Incremental(2025-10)| Batch | Hybrid + memify | 🟢 Streaming 唯一 |
| **7. Synthesis** | LLM compact | 多种 | Citation-aware | Community report | (各 retriever 自定)| question_answering.py |
| **8. Feedback loop** | partial | ❌ | partial | ❌ | 🟢 memify 7 pipeline 闭环 | ❌ |

**🟢 标记的是每个项目最强维度**。6 个项目**没有一个 8 维全强**,SkillForge 借鉴时需**按维度 cherry-pick**,不要全套抄某一家。

---

## 4. SkillForge Memory v3 决策建议(基于 8 维 + ICLR'26 calibration)

### 4.1 已有(verified-from-source 不重做)

| 维度 | SkillForge 现状 | 业内对照 |
|---|---|---|
| 3. Index | Hybrid (FTS + Vector + RRF) | 业内主流 ✅ |
| 5. Reranking | RRF (RRF_K=60) | 业内 default ✅ |
| 6. Indexing 时态 | Incremental + Dream-Consolidation 定期 | RAGFlow / Cognee 同模式 ✅ |

### 4.2 真缺口(按 ROI 排序)

| 维度 | 缺口 | 借鉴源 | 优先级 | 工作量 |
|---|---|---|---|---|
| 8 | **Agent trace + user feedback 自动入 memory pipeline** | Cognee `persist_agent_trace_feedbacks_in_KG` | 🔴 P1 | 1-2 周 |
| (架构)| **Skill-internal memory decorator 抽象**(@with_memory)| Cognee `@cognify` decorator | 🔴 P1 | 1 周 |
| (架构)| **workflow_as_tool / skill_as_tool 抽象**(skill nested composition)| Dify `tools/workflow_as_tool/` | 🔴 P1 | 1-2 周 |
| (安全)| **gVisor sandbox**(skill engine 代码隔离)| RAGFlow `agent/sandbox/` | 🔴 P1 | 2-3 周 |
| 4 | Sub-question decomposition + Router(2-3 pattern,不抄 18+ 全套)| LlamaIndex SubQuestion + Router / Cognee | 🟡 P2 | 2 周 |
| 6 | Differential Dataflow 设计思想(memory dedup 高频时)| Pathway | 🟢 远期 | < 1 周 design 评估 |

### 4.3 ⚠️ 不要做(ICLR'26 calibration)

| 项 | 警示 |
|---|---|
| **KG memory v3 默认全量化** | ⚠️ ICLR'26 _"GraphRAG frequently underperforms vanilla RAG"_。先把 vanilla 做扎实,KG 只在 _multi-hop / 全局主题汇总_ 场景按需做 |
| **18+ Query Engine 全套抽象** | 过度工程。SkillForge 当前 2 pattern → 加到 4-5 pattern 即可 |
| **159 data reader / 104 LLM provider 全套** | LlamaIndex 商业 moat,SkillForge 垂直 SaaS 不需要 |
| **Streaming Differential Dataflow 代码移植** | Rust + BSL 1.1 license + JVM 栈不兼容。借设计思想,不借代码 |
| **Deep document understanding 16 parser** | SkillForge 不 ingest 客户 PDF。商业化时如果接 enterprise 文档场景再说 |

---

## 5. 总 R-RAG 借鉴清单(给 SkillForge 一个 decisive 清单)

合并 5 个 ingest 的 R-DIFY / R-LI / R-RF / R-GR / R-COG / R-PW 各页 frontmatter,出 SkillForge memory v3 **统一决策清单**:

### 🔴 P1 必做 4 条(共 5-8 周,直接打到生产能力短板)

| ID | 项 | 借鉴源 | SkillForge 影响 |
|---|---|---|---|
| **R-RAG-1**(R-COG-1) | **Agent trace + user feedback → memory 自动 pipeline** | Cognee | 撬动 SkillForge ATTRIBUTION-AGENT v3 + MEMORY v3 闭环 self-improving |
| **R-RAG-2**(R-COG-2) | **Decorator-based skill-internal memory API** | Cognee | Skill 内 LLM call 可声明 memory scope (session/user/cross-session) |
| **R-RAG-3**(R-DIFY-2) | **workflow_as_tool / skill_as_tool 抽象** | Dify | Skill nested composition,撬动复杂业务流程组合 |
| **R-RAG-4**(R-RF-2) | **gVisor sandbox 集成** | RAGFlow | Skill engine 代码安全隔离,商业化必备 |

### 🟡 P2 应做 3 条

| ID | 项 | 借鉴源 | 工作量 |
|---|---|---|---|
| **R-RAG-5** | Sub-question decomposition + Router(2-3 retrieval pattern)| LlamaIndex + Cognee | 2 周 |
| **R-RAG-6** | Composable memory blocks(功能维度叠加)| LlamaIndex | < 1 周 |
| **R-RAG-7** | Moderation 独立 first-class 模块 | Dify | < 1 周 |

### 🟢 P3 远期参考(不做不影响)

| ID | 项 | 备注 |
|---|---|---|
| R-RAG-8 | Deep document understanding 16 parser | 商业化 enterprise 客户上传文档时 |
| R-RAG-9 | KG memory v3 + 4 search mode | ⚠️ 不必默认,只在 multi-hop query 需求出现时 |
| R-RAG-10 | Differential Dataflow 设计思想 | Memory dedup 高频时 |
| R-RAG-11 | Temporal + statistical stdlib | Trace/metric 时序聚合复杂时 |
| R-RAG-12 | `skill.md` Anthropic Skill wrapper marketplace 模式 | < 半周 |

---

## 6. 跟现有 wiki 关系(完成度盘点)

| 之前 wiki 页 | 本次合并 | 后续动作 |
|---|---|---|
| [[concept/agent-memory]] 5 + 1 SkillForge 路线 | 加 Cognee 作为**第 6 路线 "Memory control plane + memify 闭环"** | 后续可单独深化 |
| [[eval/zep-graphiti]] / [[eval/mempalace]] / [[eval/letta]] / [[eval/mem0]] / [[eval/langmem]] | 跟 RAG 6 项目正交 | 维持现状 |
| [[eval/ragas]] | RAG eval framework,跟本页正交补充 | — |
| [[harness/openclaw]] | OpenClaw 跟 RAGFlow 有官方集成 skill(2026-03-24)| 印证已加 |
| [[concept/agent-harness]] R-HARNESS / [[concept/agent-state-separation]] R-AS / [[concept/runtime-context-injection]] R-INJECT | 跟本页 R-RAG 各自独立 | SkillForge 全栈 borrow roadmap 待合并 |

---

## 7. 写这页让我学到的 3 件事

1. **6 个 RAG project ingest 后,一个反直觉发现**:**SkillForge memory 不是普通 RAG over documents**,而是 _RAG over agent state_。这意味着 80% 的 _"deep document understanding"_ 工程对 SkillForge 不直接 applicable,但**抽象层(retrieval pattern / KG / temporal)+ 闭环工程(sandbox / incremental / feedback loop)是直接借鉴的**。
2. **业内"KG RAG"被高估**:ICLR'26 GraphRAG-Benchmark paper 直接 calibration _"frequently underperforms vanilla RAG"_。SkillForge 当前 hybrid FTS+Vector+RRF 是业内 default 优秀实现,不需要盲跟 KG 路线。
3. **6 个 project 没有一个 8 维全强**。SkillForge 借鉴**必须按维度 cherry-pick**:Cognee 拿 feedback loop 闭环 / Dify 拿 workflow_as_tool / RAGFlow 拿 gVisor sandbox / LlamaIndex 拿 sub-question pattern。**不要全套抄某一家**。

---

## 引用链路

- 5 个 RAG ingest:[[harness/dify]] / [[eval/llama_index]] / [[harness/ragflow]] / [[eval/ms-graphrag]] / [[eval/cognee]] / [[eval/pathway]]
- Memory 5 路线:[[eval/mem0]] / [[eval/letta]] / [[eval/mempalace]] / [[eval/langmem]] / [[eval/zep-graphiti]]
- 配套:[[eval/ragas]](RAG eval framework)/ [[concept/agent-memory]] / [[concept/agent-harness]]
- ICLR'26 paper:GraphRAG-Bench/GraphRAG-Benchmark _"When to use Graphs in RAG: A Comprehensive Analysis"_
- 跨页 borrow list 合并:[[concept/agent-harness]] R-HARNESS / [[concept/agent-state-separation]] R-AS / [[concept/runtime-context-injection]] R-INJECT / 本页 R-RAG 待合并出 SkillForge 全栈 roadmap
