---
type: source
name: mem0
origin: "/Users/youren/myspace/mem0/ (Mem0 AI Inc, Apache 2.0)"
ingested_at: 2026-05-16
ingested_commit: 219b1a6
ingested_commit_date: 2026-05-16
ingested_version_label: "2.0.2(`pyproject.toml`),Y Combinator S24"
contributed_to:
  - eval/mem0.md
  - concept/agent-memory.md
---

# Source — Mem0

> **业内最广泛使用的 LLM memory framework**(Apache 2.0,Y Combinator S24,**52k+ GitHub stars**)。**2026-04 新算法 LongMemEval 94.8(+27 分)/ LoCoMo 91.6(+20 分)/ BEAM(1M) 64.1**。**Multi-Level Memory**(User / Session / Agent)+ 自动 LLM 提取 facts + Python + TypeScript + 完整 self-host platform。**仓库内置 OpenClaw plugin SDK**!

## 仓库结构概览

```
mem0/
├── mem0/                            # ⭐ Python 主包(PyPI: `mem0ai`)
│   ├── __init__.py
│   ├── memory/                      # ⭐ Memory 核心(main.py 含 Memory + AsyncMemory 双类)
│   │   ├── base.py                  # MemoryBase 基类
│   │   ├── main.py                  # Memory(:331) + AsyncMemory(:1795)实现
│   │   ├── storage.py               # SQLite history
│   │   ├── telemetry.py             # 匿名 telemetry
│   │   ├── utils.py                 # 工具
│   │   └── setup.py
│   ├── client/                      # API client
│   ├── configs/                     # 配置 schema
│   ├── embeddings/                  # Embedding 适配
│   ├── llms/                        # LLM 适配
│   ├── proxy/                       # Proxy(OpenAI 兼容)
│   ├── reranker/                    # Reranker 集成
│   ├── utils/
│   ├── vector_stores/               # 多 vector store(Qdrant/FAISS/Chroma/pgvector/Weaviate/Milvus 等)
│   └── exceptions.py
├── mem0-ts/                         # ⭐ TypeScript SDK(npm: `mem0ai`)
├── mem0-plugin/                     # Plugin SDK
├── server/                          # ⭐ Self-host FastAPI server(`mem0/server/`)
│   ├── main.py
│   ├── alembic/                     # DB migrations
│   ├── auth.py
│   ├── dashboard/                   # 内置 dashboard
│   ├── db.py
│   ├── docker-compose.yaml + Dockerfile
│   ├── models.py
│   ├── rate_limit.py
│   └── ...
├── openmemory/                      # ⭐ Self-host memory dashboard(独立)
│   ├── api/                         # API
│   ├── ui/                          # React UI
│   ├── compose/                     # docker compose
│   ├── backup-scripts/
│   └── run.sh
├── openclaw/                        # ⭐⭐ mem0 给 OpenClaw 的 plugin!
│   ├── openclaw-plugin-sdk.d.ts     # TS SDK 类型
│   ├── openclaw.plugin.json         # plugin 配置
│   ├── backend/                     # plugin backend
│   ├── cli/                         # plugin CLI
│   ├── config.ts
│   ├── dream-gate.ts                # ⭐ 跟 OpenClaw "Memory Dreaming" 系统对齐!
│   ├── filtering.ts
│   ├── fs-safe.ts                   # 文件安全
│   ├── isolation.ts                 # 隔离
│   └── package.json + pnpm-lock.yaml
├── embedchain/                      # ⭐ Embedchain library(mem0 收购?)
├── evaluation/                      # ⭐ 评测框架(对照 LongMemEval / LoCoMo / BEAM benchmark)
├── examples/                        # 用户示例
├── cookbooks/                       # Cookbook
├── docs/                            # 文档站
├── scripts/                         # 脚本
├── skills/                          # Skill 集成?
├── tests/
├── vercel-ai-sdk/                   # Vercel AI SDK 集成
├── server/                          # 重复(已列上面)
├── AGENTS.md + CLAUDE.md            # LLM agent guide
├── LLM.md                           # ⭐ 给 LLM 看的项目说明
├── MIGRATION_GUIDE_v1.0.md          # v1.0 迁移指南
├── pyproject.toml(version = "2.0.2")
└── poetry.lock
```

## 关键 file:line 锚点

### 2026-04 新算法(README "New Memory Algorithm")

**Benchmark 大幅提升**(README 表格):
| Benchmark | Old | New(2026-04) | Tokens | Latency p50 |
|---|---|---|---|---|
| **LoCoMo** | 71.4 | **91.6**(+20.2) | 7.0K | 0.88s |
| **LongMemEval** | 67.8 | **94.8**(+27) | 6.8K | 1.09s |
| **BEAM (1M)** | — | **64.1** | 6.7K | 1.00s |
| **BEAM (10M)** | — | **48.6** | 6.9K | 1.05s |

**新算法 5 大变化**:
1. **Single-pass ADD-only extraction** — 1 LLM call,**no UPDATE/DELETE,memories 累加不覆盖**
2. **Agent-generated facts first-class** — agent 确认的 action 同等存储
3. **Entity linking** — 实体提取 + 嵌入 + 跨 memory 链接(检索 boost)
4. **Multi-signal retrieval** — semantic + BM25 + entity matching 并行打分融合
5. **Temporal Reasoning** — 时间感知检索(current state / past events / upcoming plans)

`mem0.ai/research` 完整 paper + `github.com/mem0ai/memory-benchmarks` 开源 benchmark 框架。

### Memory 主类(`mem0/memory/main.py:331`)

```python
class Memory(MemoryBase):
    def __init__(self, config: MemoryConfig = MemoryConfig()):
        # 5 大子系统
        self.embedding_model = EmbedderFactory.create(...)   # ⭐ Embedder
        self.vector_store = VectorStoreFactory.create(...)   # ⭐ Vector store
        self.llm = LlmFactory.create(...)                    # ⭐ LLM
        self.db = SQLiteManager(self.config.history_db_path) # ⭐ SQLite history
        self.reranker = RerankerFactory.create(...)          # ⭐ Reranker(optional)

        # Entity store lazy init
        self._entity_store = None  # 新算法:实体存储独立 collection(`{collection_name}_entities`)
```

**核心方法**(`memory/main.py`):
- `def add(...)` — `:573`(主入口)
- `def get(memory_id)` — `:973`
- `def get_all(...)` — `:1016`
- `def search(...)` — `:1126`
- `def update(memory_id, data, metadata=None)` — `:1501`
- `def delete(memory_id)` — `:1524`
- `def delete_all(user_id, agent_id, run_id)` — `:1540`

### `AsyncMemory` 双类(`memory/main.py:1795`)

完整 async 镜像,同 API 但 `async def`(对照 Ragas async / [[../eval/opik]] async)。

### Multi-Level Memory(README "Core Capabilities")

**User / Session / Agent 三层**(`memory/main.py` 参数 `user_id` / `agent_id` / `run_id`):
- `user_id` — 跨 session 的用户级 memory(用户偏好 / 长期事实)
- `agent_id` — Agent 级 memory(agent 特定知识)
- `run_id` — Session 级 memory(单次对话)
- **三层组合查询**:`get_all(user_id="x", agent_id="y")` 拿用户 × agent 的交集

### mem0 内置 OpenClaw plugin(`mem0/openclaw/`)

**Wiki 新发现**:**mem0 给 OpenClaw 做了 plugin SDK**!
- `openclaw-plugin-sdk.d.ts` — TypeScript 类型
- `openclaw.plugin.json` — plugin manifest
- `dream-gate.ts` — **直接对接 OpenClaw "Memory Dreaming" 系统**(详见 [[../harness/openclaw]] 整 OpenClaw Memory Dreaming 实现)
- `filtering.ts` + `fs-safe.ts` + `isolation.ts` — plugin 安全保护
- `backend/` + `cli/` — plugin 后端 + CLI

**关键 insight**:mem0 + OpenClaw 是同一 memory ecosystem 的两端,**mem0 给 SDK,OpenClaw 给 runtime**。

### `openmemory/`(self-host memory dashboard)

完整 self-host 体系(对照 [[../eval/langfuse]] / [[../eval/opik]] self-host):
- `api/` + `ui/`(React)+ `compose/` docker
- `backup-scripts/` 备份
- `run.sh` 一键启动

### `server/`(FastAPI memory server)

- `main.py` + `db.py` + `models.py` + `auth.py` + `rate_limit.py`
- **`alembic/`** DB migrations(对照 Langfuse Drizzle / Opik MySQL)
- **`dashboard/`** 内置 dashboard
- `docker-compose.yaml` + `Dockerfile`

### Vector Store 多 provider(`mem0/vector_stores/`)

支持 Qdrant / FAISS / Chroma / pgvector / Weaviate / Milvus 等(具体列表需 ls,模式跟 Langfuse / Opik 类似 multi-backend)。

### `evaluation/` 评测框架

mem0 自家 evaluation:对 LoCoMo / LongMemEval / BEAM 跑 benchmark(对照 [[../eval/opik]] 8 内置 benchmark / [[../eval/swe-bench]] 评测框架)。

### `vercel-ai-sdk/` 集成

跟 Vercel AI SDK 一等公民集成(对照 Langfuse / Phoenix / Helicone 都集成 Vercel AI SDK)。

### `LLM.md` 文档(`mem0/LLM.md`)

**业内独家**:给 LLM 看的项目说明(对照 [[../harness/codex]] AGENTS.md 17KB / [[../harness/opencode]] 4KB)。

### `MIGRATION_GUIDE_v1.0.md`

v1.0 → v2.0(2026-04 新算法)迁移指南 — **`add()` 调用方式 backward compatible**,但内部算法大改。

## 与 wiki 现有

### 印证
- **多 vector store 抽象** 跟 [[../eval/langfuse]] / [[../eval/opik]] 同思路
- **Self-host platform** 跟 Langfuse / Opik / Phoenix 同模式
- **Y Combinator backed**(S24)跟 Langfuse(W23)/ Helicone YC 一致

### 新发现(重大)
- **mem0 仓库内置 OpenClaw plugin**(`mem0/openclaw/`)!**dream-gate.ts** 直接对接 OpenClaw Memory Dreaming
- 这意味着 [[../harness/openclaw]] 的 Memory Dreaming 系统**实际是 mem0 + OpenClaw 共建产物**(待 verify dream-gate.ts 跟 OpenClaw `extensions/memory-core/src/dreaming.ts` 关系)

### 矛盾
- **mem0 新算法"no UPDATE/DELETE,memories 累加"** 跟 SkillForge MEMORY-DREAM-CONSOLIDATION(2026-05-08 已交付,**embedding dedup + archived_reason**)路线不同:
  - mem0 v2:累加 → 靠 retrieval 时 entity linking + multi-signal scoring 选最相关
  - SkillForge:dedup → 写入时去重 + archive
  - 两条路线 trade-off:**mem0 retrieval 慢但 ADD 快,SkillForge ADD 慢但 retrieval 快**

### 新增
- **`eval/mem0.md`**(新建)
- **新建 `concept/agent-memory.md`**:memory 设计空间(对照 [[../concept/self-evolution]] 4 类 + [[../concept/observability]] Wide Events)
- **扩 [[../concept/self-evolution]]**:加 mem0 的"自动 LLM 提取 facts"路线(对照 OpenClaw Memory Dreaming / Hermes commit_memory_session / Claude Code autoDream)

## 事实漂移 / 注意点

- **2.0.2 是 2026-05-16 当天发布**,这是 mem0 v2 算法最新版
- **52k+ GitHub stars** 跟 OpenClaw 60K+ → 300K+ 量级有 5-10x 差距,但 mem0 在 memory 垂直市场是业内最广用
- **`openmemory/` vs `server/`**:openmemory 是 mem0 团队推的 self-host dashboard 商业 SaaS 反向,server 是 base FastAPI
- **`embedchain/`** 在仓库内 — mem0 之前的产品(embedchain 是更早的 RAG library),整合到 mem0 v2
- **Multi-Level Memory(User/Session/Agent)** 跟 SkillForge 当前 memory 维度对照:SkillForge MEMORY-v2 的 scope 划分?
- **Single-pass ADD-only** v2 算法 + **Temporal Reasoning** — **跟 [[zep+graphiti]] Temporal Knowledge Graph 路线同方向**(待 verify)
- **`LLM.md` + `AGENTS.md` + `CLAUDE.md`** 3 个给 LLM 看的文档 — 业内最深 LLM-readable docs dogfood
- **mem0 跟 OpenClaw 互通** 是 wiki 重大新发现,需要更新 [[../source/openclaw]] + [[../harness/openclaw]] Memory Dreaming 章节
