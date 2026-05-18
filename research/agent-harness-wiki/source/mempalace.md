---
type: source
name: mempalace
origin: "/Users/youren/myspace/mempalace/ (MemPalace, license 待 verify)"
ingested_at: 2026-05-16
ingested_commit: bb31396
ingested_commit_date: 2026-05-15
ingested_version_label: "mempalace(PyPI: mempalace,本仓库为官方)"
contributed_to:
  - eval/mempalace.md
  - concept/agent-memory.md
---

# Source — MemPalace

> ⚠️ **官方源唯一**:GitHub `MemPalace/mempalace` + PyPI `mempalace` + 文档 `mempalaceofficial.com`。其他域名(包括 `mempalace.tech`)是 **impostor + may distribute malware**(README 顶部 CAUTION)。
>
> **Local-first AI memory**:**Verbatim 存储 + 语义检索 + 零 API call 零 cloud + LongMemEval R@5 96.6% raw / Hybrid v4 98.4% / Hybrid+rerank ≥99%**。结构化 index(wings/rooms/drawers)+ **29 MCP tools** + **Temporal Knowledge Graph(SQLite)** + Claude Code auto-save hooks。

## 仓库结构概览

```
mempalace/
├── mempalace/                       # ⭐ Python 主包
│   ├── __init__.py
│   ├── __main__.py
│   ├── _stdio.py
│   ├── backends/                    # ⭐ Pluggable backend(default ChromaDB)
│   │   ├── base.py                  # ⭐ 接口定义
│   │   ├── chroma.py                # ChromaDB 实现
│   │   └── registry.py
│   ├── cli.py                       # CLI 主入口
│   ├── closet_llm.py
│   ├── config.py
│   ├── convo_miner.py + convo_scanner.py  # ⭐ Mine conversation(Claude Code sessions 等)
│   ├── corpus_origin.py
│   ├── dedup.py                     # ⭐ Dedup
│   ├── dialect.py
│   ├── diary_ingest.py              # ⭐ Diary(agent-specific diary)
│   ├── embedding.py
│   ├── entity_detector.py + entity_registry.py  # ⭐ Entity 系统
│   ├── exporter.py
│   ├── fact_checker.py              # ⭐ Fact check
│   ├── general_extractor.py
│   ├── hooks_cli.py                 # ⭐ Claude Code hooks
│   ├── i18n/                        # 国际化
│   ├── instructions/ + instructions_cli.py
│   ├── knowledge_graph.py           # ⭐ Temporal Knowledge Graph(SQLite + validity windows)
│   ├── layers.py                    # 多层抽象
│   ├── llm_client.py + llm_refine.py
│   ├── mcp_server.py                # ⭐ MCP server(29 tools)
│   ├── migrate.py
│   └── miner.py                     # ⭐ 主 miner
├── benchmarks/                       # ⭐ 完整 benchmark suite
│   ├── BENCHMARKS.md
│   ├── HYBRID_MODE.md
│   ├── lme_split_50_450.json        # LongMemEval 50/450 train/test split
│   ├── longmemeval_bench.py
│   ├── locomo_bench.py
│   ├── convomem_bench.py
│   ├── membench_bench.py
│   ├── mine_bench.py
│   └── model_eval/
├── docs/                            # 文档(`docs/HISTORY.md` 含 imposter timeline)
├── examples/
├── hooks/                            # Claude Code hooks 示例
├── integrations/                     # 集成
├── landing/                          # 落地页
├── tests/
├── tools/
├── website/                          # 官方 website 源码
├── AGENTS.md + CLAUDE.md
├── MISSION.md
├── ROADMAP.md
├── CHANGELOG.md
├── openarena-claim.txt              # OpenArena 声明
├── pyproject.toml
└── uv.lock                          # ⭐ uv 包管理(Astral)
```

## 关键 file:line 锚点

### Benchmark 结果(README "Benchmarks")

**LongMemEval(500 questions,R@5)**:
| Mode | R@5 | LLM required |
|---|---|---|
| **Raw**(semantic search, no heuristics, no LLM) | **96.6%** | None |
| **Hybrid v4**(held-out 450q,tuned on 50 dev)| **98.4%** | None |
| **Hybrid v4 + LLM rerank**(full 500)| **≥99%** | Any capable model |

**Hybrid v4 加** keyword boosting + temporal-proximity boosting + preference-pattern extraction(no LLM)。
**LLM rerank** 用 reader LLM 选 top-20 中最佳(已 reproduce Claude Haiku / Claude Sonnet / minimax-m2.7 via Ollama Cloud)。

**其他 benchmark**:
| Benchmark | Metric | Score | Notes |
|---|---|---|---|
| LoCoMo(session, top-10, no rerank) | R@10 | 60.3% | 1,986 questions |
| **LoCoMo(hybrid v5, top-10, no rerank)** | R@10 | **88.9%** | 同 set |
| ConvoMem(250 items, 50 per category) | Avg recall | **92.9%** | |
| MemBench(ACL 2025, 8,500 items) | R@5 | **80.3%** | All categories |

**Reproducibility 一等公民**(README):
```bash
git clone https://github.com/MemPalace/mempalace.git
cd mempalace
uv sync --extra dev
uv run python benchmarks/longmemeval_bench.py /path/to/longmemeval_s_cleaned.json
```

### 不做 cross-project benchmark 比较(README 哲学)

> "We deliberately do not include a side-by-side comparison against Mem0, Mastra, Hindsight, Supermemory, or Zep. Those projects publish different metrics on different splits, and placing retrieval recall next to end-to-end QA accuracy is not an honest comparison."

**工程文化漂亮**:不堆 cherry-pick 对比,让用户自己看每个项目的 research page。

### 核心抽象:Verbatim + Wings/Rooms/Drawers(README "What it is")

**Verbatim 存储**:不 summarize / extract / paraphrase(对照 [[mem0]] LLM-extracted facts / [[../harness/openclaw]] Memory Dreaming)

**结构化 index**:
- **Wings** — 人 / 项目级(top-level)
- **Rooms** — 主题级(mid-level)
- **Drawers** — 原始内容(leaf-level)

**查询 scope** — 在 wing × room 范围内检索,而非 flat 全 corpus(检索精度高的关键)。

### Pluggable backend(`mempalace/backends/`)

```
backends/
├── __init__.py
├── base.py        # ⭐ 接口定义
├── chroma.py      # default(ChromaDB)
└── registry.py
```

用户可加自家 backend(Qdrant / pgvector / FAISS / Weaviate)— 不改其他代码。

### 29 MCP tools(`mempalace/mcp_server.py`)

**业内 memory framework MCP 集成最深**。Tools 覆盖:
- Palace reads/writes
- Knowledge-graph operations
- Cross-wing navigation
- Drawer management
- Agent diaries

完整列表:`mempalaceofficial.com/reference/mcp-tools`(本 ingest 未深读 mcp_server.py source)

### Temporal Knowledge Graph(`mempalace/knowledge_graph.py`)

**SQLite-backed + validity windows**(README):
- 实体 + 关系 + 时间窗口
- 操作:add / query / **invalidate**(关系失效)/ timeline
- 跟 [[zep+graphiti]] Temporal KG 同方向但**本地 SQLite**(对照 Graphiti 通常配 Neo4j)

### Agents + Diaries(README "Agents")

每个 specialist agent 有自己的 **wing + diary**(运行时通过 `mempalace_list_agents` 发现)— **不污染 system prompt**(`no bloat in your system prompt`)。

业内独家:agent 自家 diary 作为长期 memory。

### Auto-save Claude Code Hooks(README "Auto-save hooks")

2 个 Claude Code hooks:
1. **Periodic save**(定期)
2. **Pre-compaction save**(context compression 前)

**重要警告**(README 顶部):
> "Claude Code sessions expire in 30 days w/out auto-save hooks wired!"

跟 [[../harness/claude-code]] hook 体系深度集成。

### CLI workflow(README "Quickstart")

```bash
# 安装
uv tool install mempalace

# 初始化 palace
mempalace init ~/projects/myapp

# Mine 内容
mempalace mine ~/projects/myapp                       # 项目文件
mempalace mine ~/.claude/projects/ --mode convos      # Claude Code sessions
                                  --wing per-project  # 每项目一 wing

# 搜索
mempalace search "why did we switch to GraphQL"

# Load context for new session
mempalace wake-up
```

`wake-up` 是 MemPalace 独家命令 — **新 session 启动时 load 相关 wings/rooms 进 context**(对照 SkillForge MEMORY-v2 retrieval)。

### Imposter Site 警告

README 顶部明确:
- 官方 GitHub:`github.com/MemPalace/mempalace`
- 官方 PyPI:`pypi.org/project/mempalace/`
- 官方 docs:`mempalaceofficial.com`
- **Imposter:`mempalace.tech` may distribute malware**

`docs/HISTORY.md` 含 imposter timeline(待 verify)。

⚠️ **wiki 之前 WebSearch 看到的 "Milla Jovovich + Ben Sigman" 信息来自 imposter site(mempalace.tech),应忽略**。真实 MemPalace 创建者待 verify(`MISSION.md` / `CONTRIBUTING.md`)。

### Benchmark 框架完整(`benchmarks/`)

```
benchmarks/
├── BENCHMARKS.md          # 完整 reproduce 指南
├── HYBRID_MODE.md         # Hybrid 模式文档
├── lme_split_50_450.json  # LongMemEval 50/450 split
├── longmemeval_bench.py   # ⭐ LongMemEval(96.6%)
├── locomo_bench.py        # LoCoMo
├── convomem_bench.py      # ConvoMem
├── membench_bench.py      # MemBench
├── mine_bench.py          # Mine 性能 benchmark
└── model_eval/            # Model eval
```

### `convo_miner.py + convo_scanner.py + diary_ingest.py` 3 ingestion 路径

- **`convo_miner.py`** — Mine Claude Code sessions(`~/.claude/projects/`)
- **`convo_scanner.py`** — Scan conversation
- **`diary_ingest.py`** — Agent diary 摄入

### Entity 系统(`entity_detector.py + entity_registry.py`)

业内独家:**专门的 entity 系统**(对照 [[mem0]] v2 `_entity_store` 同思路,但 MemPalace 显式分 detector + registry)

### `fact_checker.py`

业内独家:**Fact check 模块**(跟 [[../eval/ragas]] `_faithfulness` 同方向但内置在 memory framework)

### `llm_refine.py`

Hybrid 模式用 LLM refine retrieval 结果。

## 与 wiki 现有

### 印证
- **MCP 集成深**(29 tools)跟 [[../eval/phoenix]] / [[helicone]] MCP server 一致业内主流
- **Temporal KG** 跟 [[zep+graphiti]] Graphiti 同方向
- **Knowledge graph backed by SQLite** 跟 [[../harness/openclaw]] memory-wiki 本地 SQLite 一致
- **Local-first + 零 API call** 跟 [[../harness/opencode]] / [[../harness/codex]] CLI-first 一致

### 矛盾
- **MemPalace verbatim 存储 vs mem0 LLM-extracted facts** — 两条对立路线
- **MemPalace 不 cross-project benchmark vs mem0 强调对比** — 工程文化对立
- **`mempalace.tech` 是 imposter site**(README 警告)— 我之前 WebSearch 看到的 "Milla Jovovich 创建" 信息**应作废**

### 新增
- **`eval/mempalace.md`**(新建)
- **`concept/agent-memory.md`**(新建)— 加 MemPalace verbatim + wings/rooms/drawers 路线 vs mem0 ADD-only 路线 vs Letta MemGPT Agent OS 路线 vs SkillForge DREAM-CONSOLIDATION 路线

## 事实漂移 / 注意点

- **官方仓库 `MemPalace/mempalace`**(注意大写 P),不是 `mempalace/mempalace`(我之前 clone 用的 URL 实际 redirect 到正确)
- **`mempalace.tech` is impostor**,警告 README 顶部
- **真实 MemPalace 创建者未 verify**(`MISSION.md` / `CONTRIBUTING.md` 未深读)— 不是 Milla Jovovich(那是 imposter site claim)
- **LongMemEval 96.6% raw + Hybrid 98.4% + Hybrid+rerank ≥99%** 是业内最高(高于 mem0 v2 94.8%)
- **Verbatim 路线 trade-off**:存储成本高(对照 mem0 LLM 提取 facts 压缩)+ 隐私好(数据不离开本地)
- **`uv tool install` 部署模式** — Astral uv 生态成员
- **`README.md` 顶部 2 个 CAUTION/IMPORTANT** — Imposter 警告 + Claude Code hook wiring 警告
- **`docs/HISTORY.md` 含 imposter timeline** — 工程文化(透明历史)
- **`MISSION.md + ROADMAP.md`** 单独 file — 项目方向公开
- **不 cross-project benchmark 哲学** — README 明示"不 cherry-pick 对比"
