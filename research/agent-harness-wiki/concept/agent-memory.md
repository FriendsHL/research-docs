---
type: concept
name: agent-memory
status: draft
sources:
  - "[[../source/mem0]]"
  - "[[../source/letta]]"
  - "[[../source/mempalace]]"
  - "[[../source/langmem]]"
  - "[[../source/zep-graphiti]]"
related_harnesses: [hermes-agent, claude-code, openclaw, codex, opencode]
related_evals: [mem0, letta, mempalace, langmem, zep-graphiti]
---

# Agent Memory — 5 framework 设计空间 + SkillForge 校准

> 本概念汇总 5 个业内主流 memory framework 的设计选择,形成 agent memory 的完整设计空间,作为 SkillForge memory 系统(MEMORY-v2 + DREAM-CONSOLIDATION + LLM-SYNTHESIS 三件套)演进的指导。

## 定义

**Agent Memory** = 让 LLM agent 在跨 session / 长期对话中**保留 / 检索 / 演化**相关信息(facts / entities / events / preferences)的能力。

跟 [[context-compaction]] 关系:
- **Context compaction** = session 内 context 满时压缩历史(短期)
- **Agent memory** = 跨 session 持久化信息(长期)
- 两者协作:compaction 触发时 memory 应保留关键事实

跟 [[self-evolution]] 关系:
- **Memory** = 信息层面(facts / entities)
- **Self-evolution** = 行为层面(prompt / rule / skill)
- Memory 可能是 self-evolution 的输入(从 memory 提炼 rule)

## 设计空间(10 维)

### 1. 存储模式

| 模式 | 代表 | trade-off |
|---|---|---|
| **LLM-extracted facts**(累加)| mem0 v2 | 紧凑 + 失原文 |
| **Verbatim 原文存储** | MemPalace | 完整 + 存储成本高 |
| **Memory blocks**(可读可写 block)| Letta(MemGPT)| Agent 主动操作 |
| **Temporal KG triplets**(entity → relation → entity)| Zep+Graphiti | 时序丰富 + 复杂 |
| **LangGraph BaseStore**(generic store)| LangMem | 灵活 + 依赖 LangGraph |
| **LLM-SYNTHESIS proposal + 人审 gate** | **SkillForge** | 受控 + 慢 |

### 2. 写入策略

| 策略 | 代表 | trade-off |
|---|---|---|
| **Single-pass ADD-only**(累加,no UPDATE/DELETE)| mem0 v2 | 1 LLM call,token 低 |
| **CRUD**(传统增删改查)| Letta + 多数早期 framework | 完整但 LLM call 多 |
| **Verbatim 不修改**(append only origin)| MemPalace | 0 LLM call(raw mode)|
| **Background async reflection**(异步后台 extract)| LangMem + OpenClaw Memory Dreaming | 不阻塞 hot path |
| **Temporal validity windows**(自动 invalidate)| Zep+Graphiti | 时序完整 |
| **Embedding dedup + archived_reason** | **SkillForge MEMORY-DREAM-CONSOLIDATION** | 节省存储 |

### 3. 检索策略

| 策略 | 代表 | trade-off |
|---|---|---|
| **Pure semantic**(vector 余弦)| 早期 framework | 简单但 entity 弱 |
| **Multi-signal**(semantic + BM25 + entity)| mem0 v2 | 准确率高 |
| **Hybrid v4**(keyword + temporal-proximity + preference)| MemPalace | LongMemEval 业内最高 |
| **Graph traversal + hybrid**(semantic + keyword + 跨节点遍历)| Zep+Graphiti | 关系丰富但 graph DB 成本 |
| **Hierarchical scope**(wings × rooms × drawers)| MemPalace | scope 缩小,精度高 |
| **Cross encoder reranking** | Graphiti 独家 | 比 LLM rerank 便宜 |
| **LLM reader rerank**(top-20 → 1 final)| MemPalace hybrid+rerank | ≥99% R@5 但 1 LLM call |

### 4. Scope / Namespace

| 模式 | 代表 |
|---|---|
| **3 层固定**(user / session / agent)| mem0 v2 |
| **Tuple-based 任意深度** | LangMem `namespace=("memories",)` |
| **Hierarchical wings/rooms/drawers** | MemPalace |
| **Multi-tenant namespaces** | Graphiti `graphiti_core/namespaces/` |
| **Memory blocks(per-block scope)** | Letta(MemGPT)|
| **未明示** | SkillForge MEMORY-v2(待 verify) |

### 5. 时序建模

| 模式 | 代表 |
|---|---|
| **无 temporal**(只时间戳)| 早期 framework |
| **Temporal Reasoning**(查询时间感知)| mem0 v2 |
| **Validity windows**(fact 有 from / to)| **Graphiti + MemPalace** |
| **Episode provenance**(每 fact 追溯到 source)| Graphiti |
| **Updated summaries**(entity summary 随时间更新)| Graphiti |
| **Archived with reason** | **SkillForge MEMORY-DREAM-CONSOLIDATION** |

### 6. Agent-facing 还是 framework-managed

| 模式 | 代表 |
|---|---|
| **Framework auto**(用户不调 tool,framework 自动管)| mem0 + MemPalace + LangMem `ReflectionExecutor` |
| **Agent function-call memory ops**(LLM 调 tool 主动操作)| Letta(MemGPT)`core_memory_append/replace + archival_memory_insert/search` |
| **Hybrid**(framework 自动 + agent 可主动)| LangMem `create_manage_memory_tool + create_search_memory_tool` |

### 7. 人审 gate

| 模式 | 代表 |
|---|---|
| **完全自动**(无人审)| mem0 + MemPalace + Letta + LangMem + Graphiti(都 LLM 自动)|
| **LLM 提议 + 人审 gate** | **SkillForge MEMORY-LLM-SYNTHESIS**(业内独家 production-ready) |

### 8. Knowledge Graph 维度

| 模式 | 代表 |
|---|---|
| **无 KG**(只 vector + facts)| 早期 framework |
| **Entity linking**(独立 entity store)| mem0 v2 |
| **Local KG**(SQLite + validity windows)| MemPalace |
| **Multi-DB Graph framework**(Neo4j / FalkorDB / Kuzu / Neptune)| Graphiti |
| **GraphRAG single file** | LangMem `graph_rag.py` |

### 9. Production 部署

| 模式 | 代表 |
|---|---|
| **Local-first**(SQLite + 0 cloud)| MemPalace |
| **Self-host + Cloud SaaS** | mem0 / Letta / Graphiti |
| **Cloud SaaS-only**(主 source 闭源)| Zep |
| **依赖 LangGraph Platform** | LangMem |
| **Multi-tenant first-class** | Graphiti `namespaces/` |

### 10. 与 OpenClaw / Claude Code 等 harness 联动

| 模式 | 代表 |
|---|---|
| **`mem0/openclaw/dream-gate.ts` 给 OpenClaw plugin SDK** | **mem0 + OpenClaw 共建** |
| **Claude Code auto-save hooks**(periodic + pre-compaction)| **MemPalace** |
| **`@letta-ai/letta-code` 独立 CLI**(类 Claude Code / opencode)| Letta |
| **MCP server**(给 IDE 用)| MemPalace 29 / Graphiti 1(Python)+ Zep 1(Go)/ mem0 + Letta?|

## 5 framework + SkillForge 横向对比矩阵

| 维度 | **mem0 v2** | **MemPalace** | **Letta(MemGPT)** | **LangMem** | **Zep+Graphiti** | **SkillForge** |
|---|---|---|---|---|---|---|
| 版本 | 2.0.2(2026-05-16) | bb31396(2026-05-15) | 0.16.8(2026-05-14) | 0.0.30(2026-05-11) | Graphiti 0.29.0 / Zep faf2ace | MEMORY-v2 + DREAM + LLM-SYNTHESIS |
| License | Apache 2.0 | 待 verify(local-first) | Apache 2.0 | 待 verify(LangChain ecosystem) | Graphiti OSS / Zep Cloud | 商业 |
| 存储模式 | LLM-extracted facts | Verbatim | Memory blocks | LangGraph BaseStore | Temporal KG triplets | LLM-SYNTHESIS proposal |
| 写入策略 | **Single-pass ADD-only** | Verbatim 不修改 | CRUD | Background reflection | Validity windows | **LLM 提案 + 人审 + dedup** |
| 检索 | Multi-signal | Hybrid v4 + LLM rerank | Memory hierarchy swap | LangGraph store query | **Graph traversal + Hybrid + Cross encoder** | semantic + ? |
| LongMemEval R@5 | **94.8** | **96.6 raw / 98.4 / ≥99** | (论文 baseline) | 不公布 | 不公布(Cloud) | 待 benchmark |
| Scope | user / session / agent | Wings/Rooms/Drawers | Memory blocks | tuple-based 任意深度 | Multi-tenant namespaces | 待 verify |
| Temporal | Temporal Reasoning | Validity windows | 无 | 无 | **Validity windows + Episodes** | archived_reason |
| Agent-facing | framework auto | framework auto | function-call ops | Hybrid | framework auto | framework auto |
| 人审 gate | 无 | 无 | 无 | 无 | 无 | **SkillForge 独家** |
| KG | Entity store | Local SQLite KG | 无 | GraphRAG single file | **Neo4j/FalkorDB/Kuzu/Neptune** | 无 |
| MCP server | 待 verify | 29 tools | 待 verify | 无 | 2 个(Python + Go)| ? |

## 4 路线汇总(memory framework 5 条对立路线)

```
┌─────────────────────────────────────────────────────────┐
│  mem0 路线:LLM-extracted facts + ADD-only 累加          │
│  → 1 LLM call,token 低,Entity linking 独家             │
│  → 跟 OpenClaw plugin SDK 联动(dream-gate.ts)          │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  MemPalace 路线:Verbatim + Hierarchical scope           │
│  → 0 API call(raw mode),0 cloud,LongMemEval 业内最高   │
│  → Wings(人/项目)→ Rooms(主题)→ Drawers(原文)       │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  Letta 路线:MemGPT Agent OS + function-call memory ops │
│  → Virtual context + Memory hierarchy + Inner Thoughts  │
│  → Git 版本控制 memory block(业内独家)                  │
│  → READ_ONLY_BLOCK 防误改                                │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  LangMem 路线:LangChain ecosystem memory utilities      │
│  → 跟 LangGraph BaseStore native 集成                    │
│  → Background ReflectionExecutor(Local + Remote)        │
│  → Memory + Prompt optimization 一体(4 种 kind)        │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  Zep+Graphiti 路线:Temporal KG + Episodes               │
│  → Facts 有 validity windows(from / to / superseded)   │
│  → Multi-DB Graph(Neo4j/FalkorDB/Kuzu/Neptune)         │
│  → Cross encoder reranking(比 LLM rerank 便宜)         │
│  → arxiv 2501.13956 学术 backbone                       │
└─────────────────────────────────────────────────────────┘

──────────────────────────────────────────────────────────
SkillForge 路线(第 6):LLM-SYNTHESIS proposal + 人审 gate
                    + DREAM-CONSOLIDATION dedup + archive
──────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────┐
│  [[../harness/openhuman]] 路线(第 7,2026-05-18 新加):│
│  Memory Tree + Obsidian vault 双写(Karpathy llm-wiki)   │
│  → 17 JSON-RPC controller(`memory/tree/schemas.rs:27-48`)│
│  → SQLite 本地(`rusqlite "bundled"`)+ `.md` 双写         │
│  → ≤3k token chunks + 评分 + hierarchical 折叠           │
│  → 118+ OAuth(经 Composio)+ 20min auto-fetch loop      │
│  → 用户可直接编辑 .md 降低 memory 黑盒信任成本           │
│  → 详见 [[llm-wiki-pattern]] §OpenHuman Memory Tree     │
└─────────────────────────────────────────────────────────┘
```

**7 路线小结对照**:mem0 _"LLM 提取累加"_ / MemPalace _"verbatim hierarchical"_ / Letta _"function-call ops"_ / LangMem _"LangChain ecosystem reflection"_ / Zep+Graphiti _"temporal KG episodes"_ / SkillForge _"人审 gate proposal"_ / OpenHuman _"Karpathy llm-wiki SQLite+Obsidian 双写"_ — **OpenHuman 是 wiki 内首家把 Karpathy llm-wiki 模式直接当 agent memory 后端的 verified production**(其他 6 路线都没有这种 "memory = browsable wiki" 思路)。

## SkillForge 校准(B 模式 verified,2026-05-16)

> **2026-05-16 update**:R-MEM 清单第一版基于 SkillForge `docs/delivery-index.md`(只 doc-only)。本次 grep SkillForge 源码 + Flyway migrations 校准为 `[verified-from-source]` 版本,**修订了 R-MEM-5 优先级**(已实现 Hybrid + RRF)。

SkillForge 当前 memory 系统(都已交付):
- **N1 Memory 向量检索**(2026-04-16,`V7__memory_vector_search.sql`)— pgvector HNSW + tsvector FTS GIN 双索引
- **MEMORY-v2 写入/召回/淘汰**(2026-04-27,`V29__memory_v2_status_importance.sql`)— `status(ACTIVE/STALE/ARCHIVED)` + `importance(high/medium/low)` + `last_score` 调度
- **MEMORY-DREAM-CONSOLIDATION**(2026-05-08,`V66__memory_archived_reason.sql`)— `archived_reason(expired_ttl/capacity_demote/dedup_merge_with_<id>)`
- **MEMORY-LLM-SYNTHESIS**(2026-05-11,`V68__memory_llm_synthesis.sql`)— `t_memory_proposal` + 4 类 proposal(dedup/reflection/optimize/contradiction)+ 5 状态(proposed/approved/rejected/auto_archived/stale)+ `auto_archive_after = created_at + 7 days` generated column + 人审 gate

**SkillForge 走的是第 6 条路线**(其他 5 framework 都没有的"人审 gate" + LLM 提议)。

### 🎯 重大发现(verified):SkillForge 已完成业内 hybrid retrieval

**[verified-from-source]** `MemoryService.java:853-908`:

```java
// MemoryService.java:853-879 hybridRecallL1
private List<MemorySearchResult> hybridRecallL1(Long userId, String taskContext) {
    fts = searchByFts(userId, taskContext, FTS_LIMIT);        // FTS_LIMIT=20
    vec = embeddingService.embed(taskContext)
        .map(v -> searchByVector(userId, v, VEC_LIMIT))       // VEC_LIMIT=20
        .orElse(List.of());
    // Filter to L1 types only
    fts = fts.stream().filter(r -> l1Set.contains(r.type())).toList();
    vec = vec.stream().filter(r -> l1Set.contains(r.type())).toList();
    return rrfMerge(fts, vec, L1_TOP_K);                      // L1_TOP_K=8
}

// MemoryService.java:886-908 Reciprocal Rank Fusion(K=60)
private static List<MemorySearchResult> rrfMerge(...) {
    for (int i = 0; i < fts.size(); i++)
        scores.merge(fts.get(i).memoryId(), 1.0 / (RRF_K + i + 1), Double::sum);  // RRF_K=60
    // ...
}
```

**关键常量**(`MemoryService.java:711-730`):
- `L0_BUDGET_CHARS=2048` / `L0_PER_ENTRY_CHARS=200` — L0 总 budget
- `L1_BUDGET_CHARS=4096` / `L1_PER_ENTRY_CHARS=500` / `L1_TOP_K=8` — L1 总 budget
- `FTS_LIMIT=20` / `VEC_LIMIT=20` / `RRF_K=60`
- **`L0_TYPES = ["preference", "feedback"]`**(recency-ordered 头部注入)
- **`L1_TYPES = ["knowledge", "project", "reference"]`**(Hybrid recall + RRF)

**SkillForge 业内独家 L0/L1 分级**(wiki 之前未记录):L0 是固定注入(用户偏好 + 反馈),L1 是动态 hybrid 检索。

### 🎯 重大发现:memory_kind 三态分类(`MemoryEntity.java:82-104` + V68:73-84)

**[verified-from-source]**:
- `observation`(默认)— 从 user session 提取
- `reflection` — `LlmMemorySynthesizer` 合成(对照 [[../eval/letta]] reflection 概念)
- `optimized` — LLM 重写,原文保留在 `originalContent`

**业内独家**:跟 `t_memory.type`(业务分类 preference/feedback/knowledge/project/reference)正交。

### 🎯 重大发现:V80 t_optimization_event 完整因果链(ATTRIBUTION-AGENT)

**[verified-from-source]** `V80__create_optimization_event.sql`:
- **State machine**:`proposal_pending → proposal_approved/rejected → candidate_created → ab_running → ab_passed/failed → canary_started → promoted/rolled_back → verified`
- **`surface_type`**:skill / prompt / behavior_rule(V3 + V4 multi-surface)
- **`confidence DECIMAL(3,2) + CHECK 0..1`** — agent 自我 reject at < 0.5
- **`cooldown_expires_at`** 24h cooldown
- **`attribution_session_id`** 链接 curator agent session

### 🎯 重大发现:V85 EVAL-DYNAMIC-USER-SIM Phase 1 已部分交付

**[verified-from-source]** `V85__create_simulator_trial_seed_user_simulator.sql` + `V86__add_user_sim_origin.sql`:**对照 [[../eval/tau-bench]] `LLMUserSimulationEnv`**,SkillForge 已开始做。

## 真差距清单(B 模式 verified,10 条 actionable)

> **2026-05-16 update**:第一版基于 SkillForge `docs/delivery-index.md` doc-only 推断。本次 grep 源码 + Flyway migrations 校准。**R-MEM-5 从"必做"修订为"R-MEM-4 配套"**(SkillForge 已实现 Hybrid + RRF,只缺 entity signal)。

### 🔴 优先级 1 — 必做(短期 1-2 周)

| # | 借鉴 | 来源 | Verified 现状 | 工作量 |
|---|---|---|---|---|
| **R-MEM-1** | **跑 LongMemEval benchmark** + 对比 mem0 94.8 / MemPalace 96.6 raw | mem0 / MemPalace `benchmarks/` | docs/todo.md 空闲,无 benchmark mention | 1 周 |
| **R-MEM-2** | **Validity windows**(`valid_from + valid_to + superseded_by`)— 加 fact 时间窗 vs 当前 archive 视角 | Zep+Graphiti / MemPalace | **`[verified-from-source]` `MemoryEntity.java:46-65`** 只有 `status + archivedAt + archivedReason`,**无 validity windows** | 1 周 |
| **R-MEM-3** | **Episode provenance**(`source_session_id + source_message_id + source_kind`)— session/message 级 source | Graphiti Episodes + tau-bench typed enum | **`[verified-from-source]` `MemoryEntity.java:39-40, :103`** 只有 `extractionBatchId`(粗粒度)+ `synthesisRunId`(run 级),**无 session/message 级** | 半周 |

### 🟡 优先级 2 — 重要(中期 1 个月)

| # | 借鉴 | 来源 | Verified 现状 | 工作量 |
|---|---|---|---|---|
| **R-MEM-4** | **Entity linking**(独立 entity store + entity-level scoring)| mem0 v2 `memory/main.py:388-419 _entity_store` | **`[verified-from-source]`** 无 `t_memory_entity` 表 / EntityEntity 类 / graph node — **完全缺** | 1-2 周 |
| **R-MEM-5** | ✅ **Hybrid FTS + Vector + RRF 已实现**,**只缺 entity matching signal**(R-MEM-4 配套) | mem0 v2 multi-signal | **`[verified-from-source]` `MemoryService.java:853-908`** 已实现 RRF(K=60)+ L0/L1 分级 + FTS_LIMIT=20 + VEC_LIMIT=20 | **<半周**(只加 entity signal 注入 rrfMerge) |
| **R-MEM-6** | **Hierarchical scope**(Wings/Rooms/Drawers 3 层)| MemPalace | **`[verified-from-source]` `MemoryService.java:730`** 只 `userId × type`(L0_TYPES + L1_TYPES)2 层,**无 wing/room hierarchical** | 2 周 |
| **R-MEM-7** | **git 版本控制 memory**(fact-level blame + history) | Letta `block_manager_git.py` | **`[verified-from-source]` `V25__sprint1_memory_snapshot_visibility_attribution.sql`** 有 `t_memory_snapshot`(**batch rollback 粒度,不是 fact-level**) | 2-3 周 |

### 🔵 优先级 3 — 演进(长期)

| # | 借鉴 | 来源 | Verified 现状 | 工作量 |
|---|---|---|---|---|
| **R-MEM-8** | **Cross encoder reranking**(比 LLM rerank 便宜)| Graphiti `cross_encoder/` | RRF 融合但**不是 cross encoder** | 1 周(Python sidecar)|
| **R-MEM-9** | **Read-only block**(防 agent 误改 hardcoded 配置)| Letta `READ_ONLY_BLOCK_EDIT_ERROR` | **`[verified-from-source]`** `MemoryEntity` 无 `is_readonly` 字段 | 1 周 |
| **R-MEM-10** | **Ontology**(prescribed + learned)| Graphiti `ontology_utils/` | 有 `type` enum(L0/L1 5 种)+ `importance` enum,**无 learned ontology** | 3-4 周 |

## ❌ SkillForge 不该走的路(5 条避坑)

1. **完整集成 Graphiti / mem0 / Letta**(infrastructure + 语言栈成本高,**只学概念**)
2. **切 Verbatim 存储**(SkillForge MEMORY-DREAM-CONSOLIDATION dedup 已优化路线,verbatim 会破坏当前架构)
3. **切 ADD-only 累加**(SkillForge 人审 gate 路线更适合 production 安全)
4. **引入 Neo4j 等 graph DB**(SkillForge 当前 Postgres + pgvector 已够,加 graph DB 增运维)
5. **依赖 LangChain ecosystem**(SkillForge 自家 SPI 抽象 + Java 栈)

## ✅ SkillForge 已领先 6 条(memory 维度,verified)

> 2026-05-16 update:从原 4 条扩展为 6 条(B 模式校准新发现 2 个能力)。

| # | 已领先 | Verified file:line |
|---|---|---|
| 1 | **MEMORY-LLM-SYNTHESIS 人审 gate**(业内独家)| `V68__memory_llm_synthesis.sql:11-35` t_memory_proposal + 4 类 + 5 状态 + auto_archive_after 7 days generated column |
| 2 | **MEMORY-DREAM-CONSOLIDATION embedding dedup + archived_reason** | `V66__memory_archived_reason.sql` + `MemoryConsolidator.java` |
| 3 | **ATTRIBUTION-AGENT 因果链完整 state machine** | `V80__create_optimization_event.sql` + V83 multi-surface + V81 attribution_curator_agent |
| 4 | **`memory_kind` 三态分类**(observation/reflection/optimized)— 业内独家 | `MemoryEntity.java:82-104` + `V68:73-84` |
| 5 | **Hybrid FTS + Vector + RRF retrieval**(L0/L1 分级 budget)— 业内独家 L0/L1 分级 | `MemoryService.java:853-908` + `:711-730` 常量 |
| 6 | **EVAL-DYNAMIC-USER-SIM Phase 1 部分**(对照 tau-bench) | `V85__create_simulator_trial_seed_user_simulator.sql` + `V86__add_user_sim_origin.sql` |

## 重大发现(影响 wiki 其他章节)

### 1. **mem0 + OpenClaw 是协作 ecosystem**

`mem0/openclaw/dream-gate.ts` + `openclaw-plugin-sdk.d.ts` + `openclaw.plugin.json` — **mem0 仓库内置 OpenClaw plugin SDK**。

含义:
- [[../harness/openclaw]] Memory Dreaming 实际可能是 **mem0 + OpenClaw 共建**(待 verify [[../harness/openclaw]] `extensions/memory-core/src/dreaming.ts` 是否 import mem0)
- 修订 [[../harness/openclaw]] 页面的 self-evolution / Memory Dreaming 章节

### 2. `mempalace.tech` 是 imposter site

MemPalace 官方源唯一:GitHub `MemPalace/mempalace` + PyPI `mempalace` + `mempalaceofficial.com`。

**wiki 之前 WebSearch 看到的 "Milla Jovovich + Ben Sigman 创建" 信息来自 imposter,作废**。真实创建者待 verify(`MISSION.md` / `CONTRIBUTING.md`)。

### 3. Zep 主仓库已转 Cloud-only,Graphiti 接续 OSS

Zep 主仓库现在是 examples + integrations + legacy + benchmarks,**主 source 转 Zep Cloud SaaS**(`getzep.com`)。

Graphiti(`getzep/graphiti` 0.29.0)是 OSS 接续的 Temporal KG framework — **跟 mem0 / MemPalace / Letta 同档活跃**。

## 反模式

- ❌ **同时引入多 framework**(mem0 + Graphiti + Letta + LangMem)— 集成成本极高,选 1 个借鉴概念即可
- ❌ **完整复制 verbatim 存储 + ADD-only + Validity windows 三策略**(互相冲突)— 选定一条路线
- ❌ **不跑 benchmark 直接切路线**(SkillForge 应先跑 LongMemEval 看自家 R@5)
- ❌ **依赖 LangChain ecosystem**(SkillForge 应保持栈无关)
- ❌ **memory + prompt optimization 强耦合**(对照 LangMem 路线)— SkillForge 走自家 PromptImproverService 路线
- ❌ **Cloud SaaS-only**(对照 Zep)— SkillForge 应保持 self-host capable

## 学习视角:memory framework 5 条路线为什么共存

**没有最好的 memory 路线,只有最适合场景的路线**:

| 场景 | 推荐路线 |
|---|---|
| 个人 AI assistant(隐私优先)| **MemPalace**(0 API + local-first + Wings/Rooms/Drawers)|
| 商业 SaaS chatbot(latency 优先)| **mem0**(LongMemEval 94.8 + token 低)|
| 长任务 agent OS(主动 memory ops)| **Letta(MemGPT)**(Agent function-call memory)|
| LangChain 已用 + 加 memory | **LangMem**(LangGraph BaseStore 集成)|
| 时序事实关键场景(法律 / 医疗 / 金融)| **Zep+Graphiti**(Validity windows + Episodes) |
| 多租户 production platform(SkillForge)| **SkillForge 自家**(LLM-SYNTHESIS + 人审 gate + DREAM-CONSOLIDATION) |
