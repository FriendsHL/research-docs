---
type: eval-framework
name: mempalace
status: active
language: python

# === 版本时效 ===
last_ingested_version: "MemPalace @ commit bb31396 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/mempalace]]"

# === 9 维度 ===
target: |
  **Local-first AI memory framework**(官方 GitHub `MemPalace/mempalace` + PyPI `mempalace` + 文档 `mempalaceofficial.com`)。**Verbatim 存储 + 语义检索 + 零 API call 零 cloud + LongMemEval R@5 96.6% raw / Hybrid v4 98.4% / Hybrid+rerank ≥99%**(业内最高分之一)。

  ⚠️ **`mempalace.tech` 是 impostor site,may distribute malware**(README 顶部 CAUTION)— 之前 WebSearch 看到的 "Milla Jovovich" 信息**作废**。

  跟其他 memory framework 对照:
  - [[mem0]] = **Single-pass ADD-only + LLM 提取 facts**(LongMemEval 94.8)
  - [[letta]] = MemGPT Agent OS + virtual context + function-call memory ops
  - **MemPalace = Verbatim 存储 + Wings/Rooms/Drawers hierarchical + 0 API call + 29 MCP tools**

  跟 SkillForge memory 对照:
  - SkillForge MEMORY-DREAM-CONSOLIDATION = embedding dedup + archived_reason
  - **MemPalace = verbatim 不做 dedup,靠 hierarchical scope 缩小检索范围**
  - **两条路完全相反**(dedup vs verbatim)
  [verified-from-source]

dataset: |
  - **Verbatim conversation 存储**(不 summarize / extract / paraphrase)
  - **3 ingestion 路径**(`mempalace/`):
    - `convo_miner.py` — Mine Claude Code sessions(`~/.claude/projects/`)
    - `convo_scanner.py` — Scan conversation
    - `diary_ingest.py` — Agent diary
  - **结构化 index**:wings(人/项目)/ rooms(主题)/ drawers(原始内容)
  - **本地 SQLite + ChromaDB**(default backend,可换)
  [verified-from-source]

scorer: |
  ### LongMemEval R@5(README "Benchmarks",500 questions)
  - **Raw**:**96.6%**(semantic search,no heuristics,no LLM)
  - **Hybrid v4**(held-out 450q,tuned on 50 dev):**98.4%**
  - **Hybrid v4 + LLM rerank**(full 500):**≥99%**

  ### 其他 benchmark(本仓库 reproducible)
  - **LoCoMo R@10**(hybrid v5,top-10,no rerank):**88.9%**(1,986 questions)
  - **ConvoMem Avg recall**(250 items):**92.9%**
  - **MemBench R@5**(ACL 2025,8,500 items):**80.3%**

  ### Hybrid 模式加什么(README + `benchmarks/HYBRID_MODE.md`)
  - Keyword boosting(BM25-like)
  - Temporal-proximity boosting(对照 [[mem0]] Temporal Reasoning)
  - Preference-pattern extraction
  - LLM rerank(top-20 candidates,用 reader LLM 选最佳)

  ### 不做 cross-project comparison(README 哲学)
  > "We deliberately do not include a side-by-side comparison against Mem0, Mastra, Hindsight, Supermemory, or Zep. Those projects publish different metrics on different splits, and placing retrieval recall next to end-to-end QA accuracy is not an honest comparison."

  **工程文化**:不 cherry-pick 对比
  [verified-from-source]

reproducibility: |
  ### 完整 benchmark reproduce(README)
  ```bash
  git clone https://github.com/MemPalace/mempalace.git
  cd mempalace
  uv sync --extra dev
  uv run python benchmarks/longmemeval_bench.py /path/to/longmemeval_s_cleaned.json
  ```

  - **`benchmarks/lme_split_50_450.json`** — LongMemEval 50/450 train/test split 锁定
  - **`benchmarks/BENCHMARKS.md`** — 完整 reproduce 指南
  - **`benchmarks/HYBRID_MODE.md`** — Hybrid 模式文档
  - **per-question result files** 提交在 `benchmarks/results_*`
  - **`uv` 包管理**(Astral)— 跟 DSPy / Promptfoo 一致
  - **零 API call raw mode**:`uv run` 离线跑(不需 API key)
  [verified-from-source]

coverage: |
  ### 5 大组件(README + 源码)
  1. **Palace storage**(`mempalace/backends/`)— Pluggable backend(ChromaDB default + 接口 `base.py`)
  2. **Hierarchical index**(wings/rooms/drawers)
  3. **Temporal Knowledge Graph**(`knowledge_graph.py`,SQLite + validity windows + invalidate + timeline)
  4. **29 MCP tools**(`mcp_server.py`)— palace reads/writes + KG operations + cross-wing nav + drawer mgmt + agent diaries
  5. **Auto-save Claude Code hooks**(`hooks_cli.py` + `hooks/` 目录)— periodic + pre-compaction

  ### Entity + Fact 系统(业内独家)
  - **`entity_detector.py`** — 实体检测
  - **`entity_registry.py`** — 实体注册
  - **`fact_checker.py`** — Fact check 模块(对照 [[../eval/ragas]] `_faithfulness`)
  - **`general_extractor.py`** — 通用 extractor
  - **`dedup.py`** — Dedup(verbatim 存储 + dedup 防完全相同内容重复)
  - **`llm_refine.py`** — Hybrid 模式 LLM refine 检索结果

  ### Agents + Diaries
  - 每个 specialist agent 自己 **wing + diary**
  - **`mempalace_list_agents`** 运行时发现 — **不污染 system prompt**

  ### CLI workflow(`mempalace/cli.py` + `__main__.py`)
  - `mempalace init <path>` — 初始化 palace
  - `mempalace mine <path>` — Mine 项目文件
  - `mempalace mine <claude/projects> --mode convos --wing per-project` — Mine Claude Code sessions
  - `mempalace search "query"` — 搜索
  - `mempalace wake-up` — 新 session 启动时 load context(业内独家命令)
  [verified-from-source]

production_loop: |
  ### Verbatim 存储 + Hierarchical 检索 loop
  ```
  Conversation source(Claude Code session / project files / diary)
       ↓ Mine
  Verbatim content + entity detection + dedup
       ↓ Index
  Wings(人/项目)/ Rooms(主题)/ Drawers(原文)+ SQLite KG
       ↓ Query
  Scoped search(wing × room 范围)+ Hybrid v4(keyword + temporal + preference)
       ↓ Optional
  LLM rerank(top-20)
       ↓
  wake-up: 注入新 session context
  ```

  ### Auto-save Claude Code hooks
  - **Periodic hook**:定期 save 当前 session
  - **Pre-compaction hook**:在 Claude Code context compression 前 save
  - **重要警告**(README):**"Claude Code sessions expire in 30 days w/out auto-save hooks wired!"**
  [verified-from-source]

automation: |
  - **MCP server 29 tools** — 业内 memory framework MCP 集成最深
  - **Claude Code auto-save hooks** — 跟 Claude Code 深度集成
  - **`uv tool install mempalace`** — 单命令安装
  - **CLI 5 command**(init / mine / search / wake-up / 等)
  - **`hooks_cli.py` + `hooks/`** 目录提供 hook 示例
  - **`integrations/`** 多 integration
  - **`tools/`** 工具集
  [verified-from-source]

cost: |
  - **零 API call(raw mode)**:96.6% LongMemEval 不需任何 LLM
  - **Hybrid v4 不需 LLM**(98.4%)
  - **Hybrid v4 + LLM rerank**(≥99%)需 LLM 但任意 capable model 都行(Claude Haiku / Sonnet / minimax-m2.7 via Ollama Cloud)
  - **零 cloud cost**:本地 SQLite + ChromaDB
  - **Token-efficient**:verbatim 存储但检索时 scope 缩小,只取 top-5/10
  - **vs SkillForge MEMORY-LLM-SYNTHESIS**:SkillForge 走 LLM-driven 4 类 proposal(每次 mine 调 LLM),MemPalace raw mode **0 LLM call**
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:MemPalace 是 **SkillForge memory 系统 benchmark 标杆**

  SkillForge 当前 memory:
  - **MEMORY-v2**(2026-04-27 已交付)— 写入/召回/淘汰
  - **MEMORY-DREAM-CONSOLIDATION**(2026-05-08 已交付)— embedding dedup + archived_reason
  - **MEMORY-LLM-SYNTHESIS**(2026-05-11 已交付)— LLM 驱动 4 类 proposal + 人审 gate

  ## ⚠️ 关键决策点:**MemPalace verbatim 路线 vs SkillForge DREAM-CONSOLIDATION dedup 路线**

  ### MemPalace 路线优势
  - **LongMemEval R@5 raw 96.6%**(业内最高 raw 分,**高于 mem0 v2 LongMemEval 94.8%**)
  - **Hybrid v4 98.4%** + **Hybrid+rerank ≥99%**
  - **零 LLM call(raw mode)** — 节省成本
  - **本地 SQLite + ChromaDB** — 隐私一等公民
  - **Verbatim 存储** — agent 看到原始 context(no information loss)

  ### MemPalace 路线代价
  - **存储成本高**(verbatim 不压缩)
  - **依赖 hierarchical index 准确性**(wings/rooms/drawers 错分会污染检索)
  - **没有 LLM-extracted facts**(对照 mem0 / SkillForge LLM-SYNTHESIS)

  ### SkillForge 决策建议
  - **不要立即切 verbatim 路线**(SkillForge MEMORY-LLM-SYNTHESIS 人审 gate 路线 + Skill marketplace 价值不重叠)
  - **可借鉴 4 个具体 actionable**(见下"可借鉴")
  - **跑 LongMemEval benchmark**:用 SkillForge memory 跑一次,对比 MemPalace 96.6% / mem0 94.8% / Letta(MemGPT 论文 baseline 较低)

  ## 可借鉴(给 SkillForge memory 加强)

  ### 高优(1-2 周)
  1. **Wings / Rooms / Drawers 三层 hierarchical index** — SkillForge memory 当前 scope 划分(对照 mem0 user/session/agent 三层),可加 MemPalace 的 "**人/项目 → 主题 → 内容**" 三层结构,**让检索 scope 而非 flat**
  2. **Hybrid retrieval(keyword + temporal + preference 3 信号)**(`benchmarks/HYBRID_MODE.md`)— SkillForge 当前可能纯 semantic,加 BM25 + temporal-proximity + preference 信号融合
  3. **`mempalace wake-up` 命令**(load context for new session)— SkillForge agent 新 session 启动可借鉴这种 explicit context load(对照当前可能自动加载)
  4. **Temporal Knowledge Graph + validity windows**(`knowledge_graph.py`)— SkillForge memory 可加时序事实建模(对照 [[zep+graphiti]] Graphiti 同方向)

  ### 中优(1 个月)
  5. **Entity detector + Entity registry 分开**(`entity_detector.py + entity_registry.py`)— SkillForge 可加 entity 一等公民(对照 mem0 entity store)
  6. **`fact_checker.py` Fact check 模块** — SkillForge memory 写入时可加 fact check(对照 [[../eval/ragas]] `_faithfulness`)
  7. **Pluggable backend interface**(`backends/base.py`)— SkillForge 当前 pgvector,但 backend interface 抽象后可换 Qdrant / Chroma / Weaviate

  ### 低优(可选)
  8. **MCP server 29 tools** — SkillForge dashboard 可暴露 MCP server 让 IDE / Claude Code / Cursor 一键查 memory
  9. **Claude Code auto-save hooks** — SkillForge 如有 Claude Code 用户场景可借鉴(对照当前 SkillForge 主要 server-side platform)

  ### 不必跟
  - **Verbatim 存储路线**(SkillForge MEMORY-DREAM-CONSOLIDATION dedup 已优化路线,verbatim 会破坏当前架构)
  - **零 LLM call**(SkillForge MEMORY-LLM-SYNTHESIS 走 LLM-driven proposal 路线)
  - **零 cloud(local-first)**(SkillForge 是 server-side platform)

  ## 关键决策

  **跑一次 LongMemEval benchmark**:
  - 用 SkillForge memory 跑 LongMemEval 500 questions(R@5)
  - 对比:
    - MemPalace raw 96.6% / hybrid 98.4%
    - mem0 v2 94.8%
    - SkillForge MEMORY-v2 + LLM-SYNTHESIS 当前实测?
  - 如果 SkillForge < 90,**借鉴 wings/rooms/drawers 三层 hierarchical scope + hybrid retrieval 3 信号**(高优 1-2 周工作量)
  - 如果 SkillForge > 95,继续自家路线

learning_takeaway: |
  ## 跟其他 memory framework 不同的是 (differentiator)
  1. **LongMemEval R@5 raw 96.6% + Hybrid 98.4% + Hybrid+rerank ≥99%**(业内最高 raw 分,**高于 mem0 v2 94.8%**)
  2. **Verbatim 存储**(不 summarize / extract / paraphrase)— 跟 [[mem0]] LLM-extracted facts / [[../harness/openclaw]] Memory Dreaming 完全对立
  3. **Hierarchical index(wings/rooms/drawers)** — 业内独家 scope-based 检索(对照 mem0 user/session/agent 3 层是 metadata-based)
  4. **零 LLM call(raw mode)+ 零 cloud + 零 API call** — 隐私 + 成本一等公民
  5. **29 MCP tools** — 业内 memory framework MCP 集成最深
  6. **Temporal Knowledge Graph(SQLite + validity windows + invalidate + timeline)** — 跟 [[zep+graphiti]] Graphiti 同方向但**本地 SQLite**
  7. **Specialist agent wing + diary** — 业内独家 agent-specific memory wing(不污染 system prompt)
  8. **Auto-save Claude Code hooks**(periodic + pre-compaction)— 跟 Claude Code 深度集成
  9. **Reproducibility 一等公民** — 完整 benchmark suite + 50/450 split + per-question result files
  10. **不 cross-project benchmark 工程文化**(README 哲学)— 业内独家"不 cherry-pick 对比"
  11. **Imposter site warning**(`mempalace.tech` 是假)— 文档透明历史

  ## 优势 (strengths)
  1. **Reproducible benchmark**(`uv run python benchmarks/longmemeval_bench.py`)
  2. **`uv tool install mempalace`** 单命令安装
  3. **Pluggable backend interface**(`backends/base.py`)— Chroma / Qdrant / 等可换
  4. **MCP 29 tools** 业内最深
  5. **Hierarchical scope-based 检索 + LLM rerank** 准确率业内最高
  6. **本地 SQLite KG + validity windows** — Temporal KG 不需 Neo4j 等额外依赖
  7. **`wake-up` 命令** explicit context load — UX 漂亮
  8. **Entity detector + registry + fact_checker 3 独立模块** — concern 分离
  9. **`benchmarks/lme_split_50_450.json`** train/test split 锁定 — 防 train/test contamination
  10. **`hooks/`** + `hooks_cli.py` Claude Code 集成深度

  ## 哪些点做得好 (what_done_well)
  1. **Verbatim + hierarchical scope** 组合 — verbatim 存储但 scope-based 检索高效(精度业内最高)
  2. **README CAUTION + IMPORTANT 顶部 2 警告** — 用户 onboarding 第一秒看到 imposter 警告 + Claude Code hook wiring 警告
  3. **`docs/HISTORY.md` 含 imposter timeline** — 工程文化透明
  4. **3 模式 R@5**(raw 96.6 / hybrid 98.4 / hybrid+rerank ≥99)— 让用户按需求选(0 LLM / 0 API / 任意 model)
  5. **`benchmarks/lme_split_50_450.json` 50/450 split 锁定** — Hybrid v4 tuned on 50 dev,reported on held-out 450 — 防过拟合
  6. **不 cherry-pick cross-project benchmark**(README 哲学)— 工程伦理
  7. **`mempalace wake-up`** 命名俏皮(对照 [[mem0]] `Memory.add` 中性 / Letta `core_memory_append` 技术 / MemPalace `wake-up` 自然)
  8. **Pluggable backend `base.py` interface + registry pattern** — 标准 plugin 模式
  9. **`benchmarks/HYBRID_MODE.md`** 单独文档解释 Hybrid 模式 — 工程文档化
  10. **`MISSION.md + ROADMAP.md`** 单独 file — 不堆 README,**项目方向公开**
---

# MemPalace(`MemPalace/mempalace`,记忆宫殿)

## 一句话定位

**Local-first AI memory framework**,**LongMemEval R@5 raw 96.6% / Hybrid 98.4% / Hybrid+rerank ≥99%(业内最高)**。**Verbatim 存储 + Wings/Rooms/Drawers hierarchical scope + 29 MCP tools + Temporal KG(SQLite)+ Claude Code auto-save hooks**。零 API call,零 cloud,零 LLM(raw mode)。

⚠️ **官方源唯一**:GitHub `MemPalace/mempalace` + PyPI `mempalace` + `mempalaceofficial.com`。`mempalace.tech` 是 **impostor**。

## 跟其他 memory framework 对照

| 项目 | 存储 | LLM 使用 | 检索精度(LongMemEval R@5)|
|---|---|---|---|
| **MemPalace raw** | Verbatim | **0 call** | **96.6%** |
| **MemPalace hybrid+rerank** | Verbatim | rerank only | **≥99%** |
| **mem0 v2** | LLM-extracted facts | ADD 1 call | 94.8% |
| **Letta (MemGPT)** | Memory blocks + virtual context | Agent function-call | (论文 baseline 较低)|
| **SkillForge** | LLM-SYNTHESIS proposal | LLM-driven(write)| 待 benchmark |

## SkillForge 关键决策(详见 frontmatter `skillforge_takeaway`)

1. **跑 LongMemEval benchmark**(SkillForge vs MemPalace 96.6 / mem0 94.8)
2. **借鉴 Wings/Rooms/Drawers hierarchical scope**(高优 1-2 周)
3. **借鉴 Hybrid retrieval 3 信号融合**(keyword + temporal + preference)
4. **借鉴 Temporal KG + validity windows**(对照 Graphiti 同方向)

## 待回答
- 真实 MemPalace 创建者(`MISSION.md` / `CONTRIBUTING.md` 未深读)— **不是 Milla Jovovich**(那是 imposter claim)
- License 具体(README 未明示,需查 `LICENSE` 文件)
- `mcp_server.py` 29 tools 完整列表
- `knowledge_graph.py` Temporal KG schema 跟 Graphiti 对比
- Hybrid v4 跟 v5 的差异(LoCoMo 用 v5,LongMemEval 用 v4)
