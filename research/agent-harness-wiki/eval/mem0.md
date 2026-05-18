---
type: eval-framework
name: mem0
status: active
language: python + typescript

# === 版本时效 ===
last_ingested_version: "mem0 2.0.2 @ commit 219b1a6 (2026-05-16, 当天发布)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/mem0]]"

# === 9 维度(mem0 不是传统 eval,是 memory framework,但 wiki schema 适配)===
target: |
  **业内最广泛使用的 LLM memory framework**(Apache 2.0,Y Combinator S24,52k+ stars)。**The Memory Layer for Personalized AI**。
  跟其他 eval / 平台对照:
  - [[../eval/langfuse]] / [[../eval/opik]] / [[phoenix]] / [[helicone]] = LLM observability platform
  - [[../eval/ragas]] / [[../eval/promptfoo]] = eval / metric library
  - [[../eval/dspy]] / [[../eval/langchain]] = framework
  - **mem0 = memory framework**(新一类 — 给 agent 加 memory layer)

  跟 wiki 已有 memory 相关:
  - [[../harness/openclaw]] Memory Dreaming(发现 **mem0 仓库内置 OpenClaw plugin**!`mem0/openclaw/dream-gate.ts`)
  - [[../harness/hermes-agent]] `commit_memory_session`
  - [[../harness/claude-code]] `autoDream`
  - [[../harness/codex]] memories 3 crate(read/write/mcp)
  [verified-from-source]

dataset: |
  - **`evaluation/`** 自家评测框架
  - 跑业内 3 大 memory benchmark:
    - **LoCoMo** — Long Conversation Memory(2024 paper)
    - **LongMemEval** — Long-term Memory Evaluation
    - **BEAM** — 1M / 10M token 规模评测
  - **github.com/mem0ai/memory-benchmarks** 开源 benchmark 框架
  [verified-from-source]

scorer: |
  ### 2026-04 新算法 5 大变化(README "New Memory Algorithm")
  1. **Single-pass ADD-only extraction** — 1 LLM call,**no UPDATE/DELETE**(memories 累加不覆盖)
  2. **Agent-generated facts first-class** — agent 确认的 action 同等存储
  3. **Entity linking** — 实体提取 + 嵌入 + 跨 memory 链接(检索 boost)
  4. **Multi-signal retrieval** — semantic + BM25 + entity matching 并行打分融合
  5. **Temporal Reasoning** — 时间感知检索(current state / past events / upcoming plans)

  ### Benchmark 结果(新算法 vs 旧)
  | Benchmark | Old | New | Tokens | Latency p50 |
  |---|---|---|---|---|
  | LoCoMo | 71.4 | **91.6**(+20) | 7.0K | 0.88s |
  | LongMemEval | 67.8 | **94.8**(+27) | 6.8K | 1.09s |
  | BEAM (1M) | — | **64.1** | 6.7K | 1.00s |
  | BEAM (10M) | — | **48.6** | 6.9K | 1.05s |

  ### Memory API(`mem0/memory/main.py:331+`)
  - `Memory.add(...)`(`:573`)— 主入口
  - `Memory.search(...)`(`:1126`)— Multi-signal retrieval
  - `Memory.get(memory_id)`(`:973`)/ `get_all(...)`(`:1016`)
  - `Memory.update / delete / delete_all`(`:1501-1540`)— v1 兼容(v2 算法不依赖 update/delete)
  - **`AsyncMemory`**(`:1795`)完整 async 镜像
  [verified-from-source]

reproducibility: |
  - **`pyproject.toml:version = "2.0.2"`** + `MIGRATION_GUIDE_v1.0.md`
  - **`mem0/openclaw/openclaw.plugin.json`** plugin manifest
  - **`alembic/`** DB migrations(`server/alembic/`)
  - **Telemetry config 自动 sanitize sensitive fields**(`memory/main.py:171` `_is_sensitive_field`)
  [verified-from-source]

coverage: |
  ### 6 大组件
  1. **Memory(Python `mem0` package)** — 核心 framework
  2. **mem0-ts**(npm `mem0ai`)— TypeScript SDK
  3. **mem0-plugin** — Plugin SDK
  4. **server/** — Self-host FastAPI server + dashboard + alembic
  5. **openmemory/** — Self-host memory dashboard 独立(api + ui + compose)
  6. **embedchain/** — RAG library(mem0 整合)

  ### Multi-Level Memory(README)
  - **User**(`user_id`)— 跨 session 用户级
  - **Session**(`run_id`)— 单次对话
  - **Agent**(`agent_id`)— Agent 特定

  ### 多 vector store
  Qdrant / FAISS / Chroma / pgvector / Weaviate / Milvus / 等(`mem0/vector_stores/` 多 provider)

  ### 不覆盖
  - 没有 LLM observability(SDK + dashboard 主要是 memory)
  - 没有 eval framework 内置 metric(对照 [[../eval/ragas]] 27)
  - 没有 Optimizer(对照 [[../eval/opik]] 7 / [[../eval/dspy]] 15)
  - 没有 prompt optimization
  [verified-from-source]

production_loop: |
  **Memory-centric production loop**:
  ```
  Agent conversation → mem0.add(messages, user_id, agent_id, run_id)
       ↓
  Single-pass LLM extraction(facts + entities)
       ↓
  Vector store + entity store 双写
       ↓
  Next conversation → mem0.search(query, user_id) → multi-signal retrieval
       ↓
  Top-K memories 注入 next prompt
  ```

  **跟 OpenClaw Memory Dreaming 集成**(`mem0/openclaw/dream-gate.ts`):
  - dream-gate 控制 OpenClaw Dreaming cron 是否触发 mem0 add
  - **mem0 + OpenClaw 是协作 ecosystem**
  [verified-from-source]

automation: |
  - **`Memory.add()`** 1 行调用 — 1 LLM call(single-pass)
  - **Plugin SDK**(`mem0-plugin/`)+ **OpenClaw plugin**(`mem0/openclaw/`)
  - **`server/main.py` FastAPI** + REST API
  - **`server/dashboard/`** 内置 web dashboard
  - **`server/alembic/`** schema 演进
  - **`server/rate_limit.py`** rate limiting
  - **Telemetry**(`memory/main.py` `MEM0_TELEMETRY` env)— 匿名,可关
  [verified-from-source]

cost: |
  - **Single-pass ADD-only**(v2 算法):**1 LLM call 完成 facts 提取**(对照旧版可能多 call update/delete)
  - **Token efficient**:LongMemEval 94.8 仅 6.8K tokens(对照 RAG 全文 retrieval 几十 K)
  - **Latency p50 < 1.1s** 4 个 benchmark 一致
  - 没有内置 cost dashboard(对照 [[helicone]] / Langfuse)— 用户自接外部 observability
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:mem0 是 **SkillForge MEMORY-v2 + MEMORY-LLM-SYNTHESIS 的开源对照**

  SkillForge 当前 memory:
  - **N1 Memory 向量检索**(2026-04-16 已交付)
  - **MEMORY-v2 写入/召回/淘汰一体化重构**(2026-04-27 已交付)
  - **MEMORY-DREAM-CONSOLIDATION**(2026-05-08 已交付,**embedding dedup + archived_reason**)
  - **MEMORY-LLM-SYNTHESIS**(2026-05-11 已交付,**LLM 驱动 4 类 proposal + 人审 gate**)

  SkillForge 跟 mem0 不同路线:
  - **SkillForge MEMORY-DREAM-CONSOLIDATION = 写入时 dedup + archive**
  - **mem0 v2 = ADD-only 累加,retrieval 时 multi-signal scoring 选最相关**

  ## ⚠️ 关键决策点:**SkillForge 是否要切 mem0 v2 ADD-only 路线?**

  ### Pro(mem0 v2 路线)
  - **LongMemEval 94.8 vs Old 67.8**(+27 分)— 业内最高分跟 [[mempalace]] LongMemEval 96.6 / 100% hybrid 同档
  - **Token-efficient**:6.8K tokens vs RAG 全文几十 K
  - **Latency < 1.1s** 4 benchmark 一致
  - **Entity linking + Temporal Reasoning** 是 SkillForge 当前缺的两个维度
  - **Single-pass 1 LLM call** vs SkillForge dedup 多步

  ### Con(SkillForge 当前 dedup 路线 trade-off)
  - **ADD-only 累加 → 存储成本高**(每条对话都加,vs SkillForge dedup 节省 storage)
  - **Retrieval 慢**(每次查都跑 multi-signal scoring)
  - **依赖 retrieval 算法**(没法手动审计 memory 库,vs SkillForge MEMORY-LLM-SYNTHESIS 4 类 proposal + 人审)

  ### 推荐
  - **不要立即切**:SkillForge MEMORY-LLM-SYNTHESIS 人审 gate 路线 + Skill marketplace 跟"自动累加"不冲突
  - **可借鉴 4 个具体细节**(见下"可借鉴")
  - **跑一次 LongMemEval benchmark 对比**:用 mem0 跟 SkillForge 当前 memory 各跑一次,看哪个高

  ## 可借鉴(给 SkillForge memory 加强)

  ### 高优(1-2 周)
  1. **Entity linking**(mem0 v2 新算法独立 entity store collection `{collection_name}_entities`,`memory/main.py:398`)— **SkillForge 当前 memory 是 fact-level,加 entity-level linking**:用户提到的 "我妹妹叫 X" / "我项目叫 Y" 这些 entity 应单独 store + 跨 memory 链接(检索 boost)
  2. **Multi-signal retrieval**(semantic + BM25 + entity)— **SkillForge 当前可能纯 semantic(pgvector cosine)**,加 BM25 keyword + entity match 并行打分融合
  3. **Temporal Reasoning** — 时间感知检索(查 "current state" 时拿最新 dated instance,查 "past events" 拿对应日期 instance,查 "upcoming plans" 拿未来 dated)— **SkillForge MEMORY-v2 当前可能纯按时间排序**

  ### 中优(1 个月)
  4. **`Memory.search(query, user_id, agent_id, run_id)` 三层过滤** — SkillForge memory scope 应有 User/Session/Agent 三层
  5. **Reranker 可选**(`memory/main.py:354`)+ `RerankerFactory` — SkillForge retrieval 加 optional reranker(Cohere / Voyage 等)
  6. **`AsyncMemory` 双类**(`memory/main.py:1795`)— SkillForge Java backend async 调用 memory(Spring `@Async`)

  ### 不必跟
  - **ADD-only 累加路线**(SkillForge MEMORY-DREAM-CONSOLIDATION dedup 已交付,选自家路线更适合 production)
  - **`openmemory/` 独立 dashboard**(SkillForge 已有 dashboard)
  - **`embedchain/` 整合**(SkillForge 不需 RAG framework)
  - **mem0-plugin SDK**(SkillForge plugin 走自家 N3 Lifecycle Hooks)

  ## 关键决策建议

  **跑一次 LongMemEval benchmark**:
  - 把 SkillForge MEMORY-v2 + DREAM-CONSOLIDATION + LLM-SYNTHESIS 当一个 memory 系统跑 LongMemEval
  - 对比 mem0 v2 的 94.8
  - 如果 SkillForge < 80,**借鉴 mem0 v2 entity linking + multi-signal retrieval + temporal reasoning 3 大新算法变化**
  - 如果 SkillForge > 90,继续自家路线

  **mem0 + OpenClaw 互通发现的影响**:
  - `mem0/openclaw/dream-gate.ts` 显示 **OpenClaw Memory Dreaming 跟 mem0 联动**
  - 需要 verify [[../harness/openclaw]] `extensions/memory-core/src/dreaming.ts` 是否 import mem0
  - 这影响 [[../harness/openclaw]] 页面的 self-evolution 章节

learning_takeaway: |
  ## 跟其他 memory framework 不同的是 (differentiator)
  1. **业内最广泛使用**(52k+ stars,Y Combinator S24)— GitHub trending memory 第 1 名
  2. **2026-04 新算法 LongMemEval 94.8 / LoCoMo 91.6 / BEAM 64.1**(+27 / +20 / first)— 业内 benchmark 跃进
  3. **Single-pass ADD-only**(no UPDATE/DELETE)— 跟传统 memory store CRUD 完全不同
  4. **Agent-generated facts first-class** — agent 自己确认的 action 跟 user 输入同等权重
  5. **Entity linking**(独立 entity store collection `{collection_name}_entities`,`memory/main.py:398`)— 业内独家
  6. **Multi-signal retrieval**(semantic + BM25 + entity)— 跟 RAG 单 vector 检索不同
  7. **Temporal Reasoning** — 时间感知检索,**跟 [[zep+graphiti]] Temporal Knowledge Graph 同方向**
  8. **Multi-Level Memory**(User / Session / Agent)— 业内事实标准 3 层划分
  9. **仓库内置 OpenClaw plugin**(`mem0/openclaw/dream-gate.ts`)— mem0 + OpenClaw 是协作 ecosystem
  10. **`openmemory/` self-host dashboard 独立**(api + ui + compose)— 对照 Langfuse / Opik 商业模式
  11. **`embedchain/` 整合** — 之前的 RAG library 收并入 mem0
  12. **`LLM.md` + `AGENTS.md` + `CLAUDE.md` 3 个给 LLM 看的文档** — LLM-readable docs 最深 dogfood

  ## 优势 (strengths)
  1. **Python + TypeScript 双 SDK**(`mem0/` + `mem0-ts/`)+ Vercel AI SDK 集成
  2. **多 vector store + 多 embedder + 多 LLM + Reranker 4 Factory**(`vector_stores/` + `embeddings/` + `llms/` + `reranker/`)— 工程化深
  3. **`Memory.add()` 1 LLM call** — token + latency 双低
  4. **`evaluation/` 自家 benchmark 框架 + `mem0ai/memory-benchmarks` 开源** — 业内 benchmark 主推者
  5. **`server/alembic/` DB migrations** + `dashboard/` 内置 — self-host 完整
  6. **`MIGRATION_GUIDE_v1.0.md`** v1.0 → v2.0 (新算法)迁移 + backward compatible API
  7. **`mem0/openclaw/` plugin SDK** — 跟 OpenClaw memory runtime 互通
  8. **Telemetry 自动 sanitize sensitive fields**(`memory/main.py:171`)— 安全 default
  9. **`MEM0_TELEMETRY` env 控制** — 可关
  10. **Y Combinator S24 backed** — production-ready 商业模式

  ## 哪些点做得好 (what_done_well)
  1. **`memory/main.py:331` Memory + `:1795` AsyncMemory 双类** — sync/async 镜像,API 一致
  2. **Entity store lazy init**(`memory/main.py:388-419`)— **第一次用才创建**,节省启动成本
  3. **Multi-signal scoring 融合**(README 描述)— 单 vector cos 不够,加 BM25 + entity match
  4. **Single-pass ADD-only**(`memory/main.py:573`)— 1 LLM call,反传统 CRUD memory
  5. **Multi-Level Memory 三参数(user_id / agent_id / run_id)**— scope 划分清晰
  6. **`Memory.add(messages, ..., infer=True/False)`** — `infer=False` 跳过 LLM 提取,直接存(用户已知 facts)
  7. **`alembic` migrations + `Dockerfile + docker-compose.yaml`** — self-host 工程化
  8. **`dream-gate.ts`** 给 OpenClaw Memory Dreaming 加 gate — 工程级集成
  9. **`embedchain/` 收编** — 反 NIH 工程文化,旧库整合而非 deprecate
  10. **`LongMemEval +27`** 单算法迭代提升 — 工程跃进
---

# Mem0(Y Combinator S24, Apache 2.0, 2.0.2)

## 一句话定位

**业内最广泛使用的 LLM memory framework**(52k+ stars,YC S24)。**2026-04 新算法 LongMemEval 94.8 / LoCoMo 91.6 / BEAM(1M) 64.1**。Single-pass ADD-only + Entity linking + Multi-signal retrieval + Temporal Reasoning + Multi-Level Memory(User/Session/Agent)+ **仓库内置 OpenClaw plugin**(`mem0/openclaw/dream-gate.ts`,跟 OpenClaw Memory Dreaming 联动)。

## 5 大算法变化(2026-04,vs v1)

| # | 变化 | 意义 |
|---|---|---|
| 1 | **Single-pass ADD-only**(1 LLM call,no UPDATE/DELETE) | Token + latency 双低,memories 累加不覆盖 |
| 2 | **Agent-generated facts first-class** | Agent 确认的 action 跟 user 输入同等权重 |
| 3 | **Entity linking** | 实体单独 store + 跨 memory 链接,检索 boost |
| 4 | **Multi-signal retrieval**(semantic + BM25 + entity) | 跨信号融合打分,RAG 单 vector 检索升级 |
| 5 | **Temporal Reasoning** | 时间感知检索,跟 Graphiti Temporal KG 同方向 |

## 跟 SkillForge memory 系统对照

| 维度 | mem0 v2 | SkillForge |
|---|---|---|
| 写入策略 | **ADD-only 累加** | **DREAM-CONSOLIDATION dedup + archive** |
| 提取 | Single-pass LLM 1 call | MEMORY-LLM-SYNTHESIS 多步 + 人审 gate |
| Entity store | 独立 collection(`{name}_entities`) | ❓ 待 verify |
| Retrieval | Multi-signal(semantic + BM25 + entity) | ❓ 当前可能纯 semantic |
| Temporal | First-class temporal reasoning | ❓ 当前可能按时间排序 |
| Multi-Level | User / Session / Agent 三层 | MEMORY-v2 scope 划分?待 verify |
| Plugin | 仓库内置 OpenClaw plugin | N3 Lifecycle Hooks |
| LongMemEval | **94.8**(最高分之一) | 待 benchmark 跑过 |

## 重大发现:**mem0 + OpenClaw 是协作 ecosystem**

`mem0/openclaw/dream-gate.ts` 文件名直接对应 OpenClaw "Memory Dreaming" 系统。需 verify:
- [[../harness/openclaw]] `extensions/memory-core/src/dreaming.ts` 是否 import mem0
- mem0 SDK 跟 OpenClaw runtime 互通详细

这个发现影响 [[../harness/openclaw]] Memory Dreaming 章节,**OpenClaw Memory Dreaming 实际是 mem0 + OpenClaw 共建产物**(待 verify)。

## SkillForge 关键决策

1. **跑一次 LongMemEval benchmark**(SkillForge memory vs mem0 v2 对照)
2. **借鉴 entity linking + multi-signal retrieval + temporal reasoning** 3 大算法
3. **不切 ADD-only 路线**(SkillForge MEMORY-DREAM-CONSOLIDATION 已优化 dedup 路线)
4. **`Memory.add(infer=False)`** 模式 — 用户已知 facts 跳 LLM 提取(SkillForge 可加)

## 待回答
- `dream-gate.ts` 具体怎么集成 OpenClaw Memory Dreaming
- `mem0/openclaw/isolation.ts` 跟 OpenClaw 多 agent 隔离关系
- `embedchain/` 整合后保留多少 RAG 能力
- v2 新算法 paper 完整(`mem0.ai/research`)
- LongMemEval 94.8 vs MemPalace 96.6 / 100% hybrid 差异原因
