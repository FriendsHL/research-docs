---
type: eval-framework
name: zep-graphiti
status: active
language: python + go

# === 版本时效 ===
last_ingested_version: "zep @ commit faf2ace (2026-04-08, work in progress) + graphiti 0.29.0 @ commit 9a2d6d0 (2026-05-14)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - Zep Cloud SaaS 闭源细节(getzep.com),本仓库已转 examples + legacy + integrations

sources:
  - "[[../source/zep-graphiti]]"

# === 9 维度 ===
target: |
  **Zep AI 出品 Temporal Context Graph framework**(getzep.com,Cloud SaaS + OSS Graphiti)。
  - **Zep**:End-to-End Context Engineering Platform(**Cloud SaaS-only**,**主仓库已转 examples + integrations**)
  - **Graphiti**(`getzep/graphiti` 0.29.0):OSS Temporal KG library(arxiv 2501.13956)

  跟其他 memory framework 对比(关键差异化:**Temporal validity windows**):
  - [[mem0]] = Single-pass ADD-only(facts 累加,时间戳但无 validity window)
  - [[mempalace]] = Verbatim + hierarchical scope + 本地 SQLite KG(有 validity windows)
  - [[letta]] = MemGPT Agent OS + virtual context(memory blocks 无 temporal)
  - [[langmem]] = LangChain ecosystem memory utilities
  - **Zep+Graphiti = Temporal KG + validity windows + graph DB(Neo4j 等)**

  跟 [[../eval/langchain]] 关系:**Zep / Graphiti 是 LangChain 5 大产品生态的"memory 补充"**,LangChain 用户可直接集成 Graphiti(对照 LangChain partners/ 16 PyPI 同思路)
  [verified-from-source]

dataset: |
  ### Zep benchmark suite(`zep/benchmarks/`)
  - **`longmemeval/zep_longmem_eval.py`** — Zep on LongMemEval
  - **`longmemeval/zep_memgpt_eval.ipynb`** — Zep vs MemGPT(Letta)对比
  - **`locomo/`** — LoCoMo benchmark
  - **`session_analysis_summary.txt`** — Session 分析

  ### Episode 系统(Graphiti 独家)
  - Raw data(对话 / 文档 / event)= Episode
  - 每个 fact / entity 都可追溯到产生它的 Episode(provenance)
  - 对照 [[mempalace]] verbatim 存储所有原文(类思路但 Graphiti 更结构化)
  [verified-from-source]

scorer: |
  ### Zep / Graphiti 不公布 benchmark 分数
  README 跟 [[mempalace]] 哲学一致 — **不 cross-project benchmark cherry-pick 比较**

  ### Graphiti Hybrid Search(`graphiti_core/search/`)
  README 明示:**hybrid retrieval = semantic + keyword + graph traversal**

  跟其他对照:
  - [[mem0]] v2:multi-signal(semantic + BM25 + entity)
  - [[mempalace]] hybrid v4:keyword + temporal-proximity + preference
  - **Graphiti**:semantic + keyword + **graph traversal**(独家:跨节点遍历检索)

  ### Cross Encoder reranking(`graphiti_core/cross_encoder/`)
  **业内独家**:Cross encoder(query × candidate pair 直接 encode)reranking,比 LLM rerank 更精细 + 更便宜

  ### Ontology 评分(prescribed + learned)
  - Prescribed ontology(用户预定义 entity / relationship type)
  - Learned ontology(自动从数据学习新 type)
  - `zep/ontology/default_ontology.py` Zep 自带 default
  [verified-from-source]

reproducibility: |
  ### Multi-DB Driver(`graphiti_core/driver/`)
  4 个 graph DB 适配:
  - **Neo4j**(default,业内主流)
  - **FalkorDB**(Redis 兼容)
  - **Kuzu**(embedded,本地)
  - **AWS Neptune**

  ### Schema Migrations(`graphiti_core/migrations/`)
  Schema 演进一等公民

  ### Multi-tenant Namespaces(`graphiti_core/namespaces/`)
  业内 memory framework 罕见的 multi-tenant first-class

  ### Docker + Helm
  - `Dockerfile + docker-compose.yml + docker-compose.test.yml`
  - Zep 自家有 `legacy/Dockerfile.ce + docker-compose.ce.yaml`(legacy CE)
  - Graphiti 提供完整 self-host stack

  ### OpenTelemetry(`graphiti/OTEL_TRACING.md`)
  独立文档解释 OTel 集成(对照 [[../eval/phoenix]] OpenInference / [[../eval/langfuse]] OTel)
  [verified-from-source]

coverage: |
  ### Zep 5 大支柱(README)
  1. **Add context**(chat / business data / events)
  2. **Graph RAG**(temporal KG)
  3. **Retrieve & assemble**(pre-formatted context blocks)
  4. **Sub-200ms latency**(README 标榜)
  5. **SOC2 Type 2 / HIPAA**(Enterprise)

  ### Graphiti 核心(README + arxiv 2501.13956)
  - **Entities**(nodes)+ updated summaries
  - **Facts / Relationships**(edges)+ temporal validity windows
  - **Episodes**(raw data provenance)
  - **Ontology**(prescribed + learned)
  - **Hybrid Search**(semantic + keyword + graph traversal)
  - **Cross Encoder reranking**

  ### MCP server 2 个
  - `graphiti/mcp_server/`(Python)
  - `zep/mcp/zep-mcp-server/`(Go)

  ### 3 语言 Zep SDK
  - Python:`pip install zep-cloud`
  - TypeScript:`npm install @getzep/zep-cloud`
  - Go:`go get github.com/getzep/zep-go/v2`

  ### 不覆盖
  - **Zep 不再开源主 source**(legacy CE,主走 Cloud SaaS)
  - **Graphiti 不含 memory store full stack**(只是 Temporal KG framework)
  - **没有 prompt optimizer**(对照 [[../eval/dspy]] / [[../eval/opik]])
  - **没有 background reflection**(对照 [[langmem]] ReflectionExecutor)
  - **没有 LongMemEval 公开分数**(对照 [[mem0]] 94.8 / [[mempalace]] 96.6 raw)
  [verified-from-source]

production_loop: |
  ### Zep Cloud 5 步 loop(README)
  ```
  1. Add context(chat / business / events)→ Zep
       ↓
  2. Graph RAG:Zep 自动 extract relationships
       ↓
  3. Build Temporal Knowledge Graph(facts + validity windows + Episodes)
       ↓
  4. Retrieve(hybrid search:semantic + keyword + graph traversal)
       ↓
  5. Assemble:pre-formatted context blocks → LLM
  ```

  **Sub-200ms latency**(README 标榜)

  ### Graphiti context graph 演化
  ```
  New Episode(raw conversation / document / event)
       ↓
  LLM extract:entities + relationships + 时间戳
       ↓
  Update graph:
    - 新 entity → add node
    - 新 fact → add edge with validity window(from / to)
    - 旧 fact 被 superseded → 更新 edge `to` 时间戳(invalidate)
    - Entity summary 随时间更新
       ↓
  Episode 作为 provenance 保留(每个 fact 可追溯)
  ```

  跟 SkillForge:**SkillForge MEMORY-LLM-SYNTHESIS 4 类 proposal + 人审 gate** 跟 Graphiti **自动 LLM extract + validity window** 不同 — Graphiti 全自动,SkillForge 有人审
  [verified-from-source]

automation: |
  - **`graphiti_core/migrations/`** schema 演进
  - **2 MCP server**(Python + Go)
  - **3 语言 SDK**(Python + TS + Go)
  - **Multi-DB driver**(Neo4j / FalkorDB / Kuzu / Neptune)— 用户按需选
  - **Multi-tenant namespaces**
  - **OpenTelemetry**(`OTEL_TRACING.md`)
  - **`graphiti/server/`** FastAPI server
  - **`graphiti/conftest.py + pytest.ini`** 测试
  - **`graphiti/depot.json + ellipsis.yaml`** CI/CD
  [verified-from-source]

cost: |
  - **Cloud SaaS 商业**(`getzep.com` 收费)
  - **Sub-200ms latency**(README 标榜)
  - **SOC2 Type 2 / HIPAA compliance**(Enterprise 一等公民)
  - **Graphiti OSS 自托管**:Neo4j 等 graph DB 是主要 infrastructure cost
  - **Cross encoder reranking 比 LLM rerank 便宜**(对照 [[mem0]] reranker + [[mempalace]] LLM rerank)
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Graphiti 是 **SkillForge memory 加 Temporal KG 维度的开源参考**

  SkillForge 当前 memory:
  - **MEMORY-v2** + **MEMORY-DREAM-CONSOLIDATION** + **MEMORY-LLM-SYNTHESIS**
  - 主要走 LLM-driven + 人审 + embedding dedup + archive 路线
  - **没有 Temporal KG 维度**(facts validity windows + entity 时间演化)

  Graphiti 提供:**Temporal KG framework**(给 SkillForge 加时序事实建模能力)

  ## ⚠️ 关键决策点:**SkillForge 是否要加 Temporal KG 维度?**

  ### Pro
  - **时序事实建模**:用户偏好/项目状态随时间变("我项目现在用 GraphQL"→ 3 月后"现在改 REST")— SkillForge MEMORY 当前可能记两条事实但不知哪条最新
  - **Validity windows**:fact 有 "from / to" 时间窗,**自动 invalidate 旧事实**(对照 SkillForge MEMORY-DREAM-CONSOLIDATION archived_reason 手动 archive)
  - **Episode provenance**:每个 fact 可追溯到产生它的 episode(对话 / 事件) — 跟 SkillForge ATTRIBUTION-AGENT 因果链思路对齐
  - **Hybrid search 加 graph traversal**:跨节点遍历检索(对照 SkillForge 当前可能纯 semantic)
  - **Multi-tenant namespaces** first-class(对照 SkillForge 多 tenant 场景)

  ### Con
  - **新 infrastructure**:Graphiti 需 graph DB(Neo4j / FalkorDB / Kuzu / Neptune)— SkillForge 当前 Postgres + pgvector
  - **Java 栈 Graphiti 不能直接用**(Python)— 需要桥
  - **Graphiti 不含完整 memory store**(只是 KG framework)— SkillForge 仍需自家 memory store

  ### 推荐
  - **不要立即引入 Graphiti**(infrastructure + 语言栈成本高)
  - **可借鉴 4 个 actionable**(见下"可借鉴")
  - **优先借鉴 validity windows + Episode provenance 概念**给 SkillForge MEMORY 加时序维度

  ## 可借鉴(给 SkillForge memory 加时序维度)

  ### 高优(1-2 周)
  1. **Validity windows**(from / to 时间窗)— SkillForge Memory Entity 加 `valid_from` + `valid_to` 字段,**让 fact 有 superseded 概念**(不只是 archive)
  2. **Episode provenance** — SkillForge Memory 加 `source_event_id` 字段,**每个 fact 可追溯到产生它的 session / message**(对照 ATTRIBUTION-AGENT 因果链)
  3. **Entity-level memory**(对照 [[mem0]] entity store 同思路)— SkillForge 把"实体"(人 / 项目 / 工具)作为独立 first-class

  ### 中优(1 个月)
  4. **Hybrid search 加 graph traversal** — SkillForge Memory retrieval 当前可能纯 semantic,加 graph traversal(跨 entity 邻居遍历)
  5. **Ontology 系统(prescribed + learned)**(`graphiti_core/utils/ontology_utils/`)— SkillForge 可加 entity type / relationship type 一等公民,**用户定义 ontology + LLM 学习新 type**
  6. **Cross encoder reranking**(`graphiti_core/cross_encoder/`)— 比 LLM rerank 便宜 + 精细,SkillForge retrieval 加 cross encoder reranking

  ### 低优 / 不必跟
  - **完整 Graphiti 集成**(infrastructure 成本高,Java + Python 桥不直接)
  - **Neo4j graph DB**(SkillForge Postgres 已够,加 Neo4j 增运维)
  - **Zep Cloud SaaS**(SkillForge 不是 SaaS,跟 Zep 商业路线冲突)
  - **3 语言 SDK**(SkillForge 自家 Channel SPI 已足够)

  ## SkillForge 已领先
  - **人审 gate**(MEMORY-LLM-SYNTHESIS)— Graphiti 全自动 LLM extract 没有人审
  - **Production trace + 飞轮 + ATTRIBUTION-AGENT**(Graphiti 是 memory framework,不是 production loop platform)
  - **Skill marketplace + SKILL-IMPORT**(Graphiti 不做 skill 系统)
  - **Multi-channel + ChannelAdapter SPI**(Graphiti 是 library 不是 platform)

  ## 关键决策建议

  **真正可借鉴 3 条**:
  1. **Validity windows**(`valid_from` + `valid_to` 字段)— SkillForge Memory Entity schema 加
  2. **Episode provenance**(`source_event_id`)— SkillForge Memory 加事实来源追溯
  3. **Entity-level memory + Ontology**(prescribed + learned)— SkillForge 把"实体"first-class

  **不建议完整集成 Graphiti**(语言栈 + infrastructure 成本高)— **只学概念**

learning_takeaway: |
  ## 跟其他 memory framework 不同的是 (differentiator)
  1. **Temporal Knowledge Graph 一等公民**(arxiv 2501.13956)— **Facts 有 validity windows**(when true / when superseded)
  2. **Episode provenance** — 每个 fact / entity 可追溯到产生它的 raw data(对话 / 文档 / event)
  3. **Ontology(prescribed + learned)** — 用户预定义 + 自动学习 entity / relationship type
  4. **Multi-DB driver**(Neo4j / FalkorDB / Kuzu / Neptune)— 业内罕见 graph DB 多 backend
  5. **Hybrid Search 加 graph traversal** — semantic + keyword + **跨节点遍历**(跟 [[mem0]] / [[mempalace]] hybrid 不同维度)
  6. **Cross Encoder reranking** — 比 LLM rerank 便宜 + 精细(业内独家)
  7. **Multi-tenant namespaces** first-class(`graphiti_core/namespaces/`)
  8. **Zep Cloud SaaS + Graphiti OSS** 双层(对照 LangChain framework + LangSmith SaaS 同模式)
  9. **Sub-200ms latency** + SOC2 Type 2 / HIPAA(Enterprise 一等公民)
  10. **2 MCP server**(Python + Go)— 业内独家(对照 Phoenix / Helicone / MemPalace 各 1 个 MCP)
  11. **3 语言 SDK**(Python + TypeScript + Go)— 比 mem0 2 / MemPalace 1 多
  12. **arxiv 2501.13956 学术 backbone**(2025-01)

  ## 优势 (strengths)
  1. **Temporal validity windows** 业内独家工程
  2. **Multi-DB driver 4 个**(Neo4j default + FalkorDB Redis + Kuzu embedded + Neptune cloud)— 用户按 deployment 场景选
  3. **`graphiti_core/cross_encoder/`** 独立模块 — reranking 比 LLM 便宜
  4. **`graphiti_core/utils/ontology_utils/`** — Ontology 独立 utility
  5. **`graphiti_core/namespaces/`** Multi-tenant first-class
  6. **`OTEL_TRACING.md`** 独立 OTel 文档
  7. **Cloud SaaS Sub-200ms latency** + SOC2 / HIPAA compliance
  8. **`zep/benchmarks/zep_memgpt_eval.ipynb`** — Zep vs MemGPT(Letta)直接对比 benchmark
  9. **`zep/ontology/default_ontology.py`** — Zep 自带 default ontology(用户可扩)
  10. **`legacy/`** 保留旧 Zep CE Go source — 历史透明

  ## 哪些点做得好 (what_done_well)
  1. **Validity windows + Episodes** 组合 — 不只时间戳,**fact 有 lifecycle**(from / to / superseded by)
  2. **4 graph DB driver 抽象**(Neo4j / FalkorDB / Kuzu / Neptune)— 对照 [[mem0]] vector store 多 backend 同模式
  3. **Cross encoder 独立模块** — 比 LLM rerank 更工程化(便宜 + 精细)
  4. **`zep/legacy/`** 命名而非 `deprecated/` — **历史保留体面**(对照 LangChain `langchain_classic/`)
  5. **Zep + Graphiti 双仓库分工**:Zep 主仓库 = 用户接入示例,Graphiti = OSS core engine — concern 分离
  6. **`zep/mcp/zep-mcp-server/`** Go MCP server + **`graphiti/mcp_server/`** Python MCP server — **2 语言 MCP**
  7. **`graphiti_core/__init__.py + graphiti.py + graphiti_types.py`** 三 file 标准模块组织
  8. **`Zep-CLA.md`** Contributor License Agreement 公开 — 商业 + OSS 协调
  9. **`depot.json + ellipsis.yaml`** CI/CD tooling — 现代 ML eng stack
  10. **arxiv 2501.13956 + paper-backed framework** — 学术 + 工程双重 credibility
---

# Zep + Graphiti(Zep AI,Temporal Knowledge Graph)

## 一句话定位

**Zep AI 出品 Temporal Context Graph framework**:Zep Cloud SaaS(主 source 闭源,getzep.com)+ **Graphiti OSS Python library**(`getzep/graphiti` 0.29.0,arxiv 2501.13956)。**Facts 有 validity windows**(when true / when superseded)+ **Episode provenance** + **Multi-DB driver**(Neo4j / FalkorDB / Kuzu / Neptune)+ **Hybrid search(semantic + keyword + graph traversal)** + **Cross encoder reranking**。

## 5 memory framework 横向对比

| Framework | 存储模式 | LongMemEval R@5 | Temporal 维度 |
|---|---|---|---|
| **mem0 v2** | LLM-extracted facts + ADD-only 累加 | **94.8** | 时间戳 + Temporal Reasoning |
| **MemPalace** | Verbatim + Wings/Rooms/Drawers | **96.6 raw / 98.4 hybrid** | Temporal KG(SQLite + validity windows) |
| **Letta(MemGPT)** | Memory blocks + virtual context | (MemGPT paper baseline) | 无显式 temporal |
| **LangMem** | LangGraph BaseStore + background reflection | 不公布 | 无显式 temporal |
| **Zep+Graphiti** | **Temporal KG(Neo4j等)+ Episodes** | 不公布(Zep Cloud)| **Validity windows + Episodes provenance**(业内最强 temporal) |

## SkillForge 借鉴 3 大概念(不集成完整 Graphiti)

1. **Validity windows**(`valid_from` + `valid_to` 字段)— SkillForge Memory Entity schema 加 fact 时间窗,自动 invalidate 旧事实
2. **Episode provenance**(`source_event_id`)— 每个 fact 可追溯到产生它的 session / message(对照 ATTRIBUTION-AGENT 因果链)
3. **Entity-level memory + Ontology**(prescribed + learned)— 把"实体"first-class,用户定义 entity/relationship type + LLM 学习新 type

## ⚠️ 不建议完整集成 Graphiti(成本高)

- **新 infrastructure**(graph DB Neo4j 等)
- **Java + Python 跨语言桥**
- **Cloud SaaS 商业模式跟 SkillForge 不符**

**只学概念,不引入 framework**。

## 待回答
- Zep Cloud SaaS 跟 Graphiti OSS 功能差异具体多少(SaaS 闭源)
- `zep/benchmarks/zep_memgpt_eval.ipynb` Zep vs MemGPT 对比结果
- Graphiti 4 graph DB driver(Neo4j / FalkorDB / Kuzu / Neptune)实际性能对比
- Cross encoder reranking 用哪个 model(默认 BGE / Cohere?)
- Ontology learned 部分怎么 LLM 学习(prompts/ 模板)
