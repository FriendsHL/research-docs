---
type: source
name: openhuman
origin: /Users/youren/myspace/openhuman (tinyhumansai/openhuman shallow clone, 2026-05-18)
version: 0.53.49 (core Cargo.toml) / 0.53.45 (app)
ingested_at: 2026-05-18
contributed_to:
  - "[[../harness/openhuman]]"
  - "[[../concept/llm-wiki-pattern]]"      # 第 3 家 Karpathy llm-wiki 模式 anchor
  - "[[../concept/agent-memory]]"          # 第 7 条 memory 路线
  - "[[../concept/multi-agent-isolation]]" # 第 2 家 Fork-style verified
read_files:
  - "README.md (1-80)"
  - "AGENTS.md (1-100)"
  - "Cargo.toml (1-80)"
  - "src/openhuman/agent/README.md (1-60)"
  - "src/openhuman/memory/tree/schemas.rs (1-100)"
  - "src/openhuman/tokenjuice/mod.rs (1-60)"
  - "ls src/openhuman/ (60+ submodules)"
  - "ls src/openhuman/memory/ + tokenjuice/ + integrations/ + agent/ + agent/agents/ + learning/"
  - "ls e2e/ + app/test/"
---

# openhuman 源码摘要

## 核心论点

**OpenHuman 是 tinyhumansai 2026-05-13 newly trending _"Personal AI superintelligence"_ Tauri 2.x 桌面 app**(Rust core in-process + React frontend monorepo),wiki 内**首家 personal AI / local-first / OAuth-heavy / voice-native niche** harness。**Memory Tree 17 JSON-RPC controllers** 是 Karpathy llm-wiki 模式 wiki 内第 3 家 production 实现,**TokenJuice 是业内独家 _"context compression as library 三层 rule overlay"_**,**Composio 桥接拿 118+ OAuth 是 _"借平台拿生态"_ 路线**。

## 关键事实

### 项目元数据(已 verified)
- **License**:待 verify(`LICENSE` 文件存在,未读)
- **Version**:**0.53.49** (`Cargo.toml`) / app v0.53.45
- **Status**:**Early Beta** + project-in-flux(QuickJS skills runtime removed,execution surfaces rebuilding)
- **代码规模**:Rust core `src/openhuman/` 60+ 子模块 + `src/core/` transport 层 + `app/` React/Vite + Tauri host
- **依赖**:Rust 1.93.0 / Node.js 24+ / pnpm 10.10.0 / CMake / Ninja / ripgrep / Tauri 2.10.1
- **3 家 LLM dogfood**:`CLAUDE.md` + `AGENTS.md` + `CODEX_WORKPAD.md`

### 6 大核心特色(README + verified)
1. **UI-first & Human + 桌面 mascot** + Google Meet agent + 跨周记忆 + 后台思考
2. **118+ OAuth + 20min auto-fetch**(via Composio 桥接)
3. **Memory Tree + Obsidian Wiki**:≤3k chunks + 评分 + 折叠 + SQLite + .md 双写(明示借鉴 Karpathy obsidian-wiki workflow)
4. **Batteries included**:web search/scraper/coder toolset/voice(STT+ElevenLabs TTS+lip-sync+Meet)/Model routing/Ollama
5. **TokenJuice 3-layer rule overlay 降本 80%**
6. **Privacy/security**:本地加密(aes-gcm + chacha20poly1305 + argon2)+ workflow data 留在 device

### Memory Tree 17 JSON-RPC controllers(verified `memory/tree/schemas.rs:27-48`)
- 写:`ingest / trigger_digest / delete_chunk / flush_now / wipe_all / reset_tree`
- 读:`list_chunks / get_chunk / list_sources / search / recall / entity_index_for / chunks_for_entity / top_entities / chunk_score / graph_export`
- 配置:`get_llm / set_llm`(backend-selector UI)

### TokenJuice 设计(verified `tokenjuice/mod.rs:37-42`)
- **`vincentkoc/tokenjuice` 的 Rust port**(承认 OSS prior art)
- **3-layer rule overlay**:Builtin `include_str!` → User `~/.config/tokenjuice/rules/` → Project `.tokenjuice/rules/` 优先级递增
- 5 子模块:classify / reduce / rules / text / tool_integration / types
- **v1 library only**,无 JSON-RPC/CLI

### Agent 模块(verified `agent/README.md`)
- **14 built-in archetypes**:orchestrator / planner / researcher / code_executor / critic / summarizer / archivist / skill_creator / tool_maker / integrations_agent / morning_briefing / help / welcome / trigger_triage / trigger_reactor
- `harness/session/{builder,runtime,turn}.rs` — turn lifecycle
- `harness/subagent_runner/` — `run_subagent` + `SubagentRunOptions`
- **`harness/definition.rs`** — `AgentDefinition + AgentDefinitionRegistry + SandboxMode + ToolScope` enum + workspace TOML loading
- **`harness/fork_context.rs`** — task-local parent context for KV-cache reuse
- **`harness/interrupt.rs`** — `check_interrupt + InterruptFence + InterruptedError` graceful cancellation
- 3 种 tool-call format:XML / JSON / **P-Format**(自家,`pformat.rs`)
- 7 RPC controllers + `triage/` external trigger pipeline

### 安全 / 加密(verified `Cargo.toml`)
- **aes-gcm + chacha20poly1305 + argon2** 本地加密 3 道防御
- **per-launch hex bearer auth** `OPENHUMAN_CORE_TOKEN`(AGENTS.md:30)
- `src/openhuman/encryption/` + `credentials/` + `approval/` 3 个 first-class submodules
- `memory/safety/` 独立 first-class submodule

### Composio 桥接架构
- `src/openhuman/composio/` 独立 module 桥接 Composio platform
- **不自写 118 个 OAuth**,只 7 个直接集成(apify/google_places/seltz/stock_prices/twilio/parallel/client)

### In-process core 架构(AGENTS.md:30)
- **PR #1061 sidecar removed** — core 现在 in-process tokio task 跑在 Tauri host
- Frontend RPC via HTTP `http://127.0.0.1:<port>/rpc` + `OPENHUMAN_CORE_TOKEN`
- `OPENHUMAN_CORE_REUSE_EXISTING=1` debug 模式

## 与现有 wiki 的关系

### 印证
- **[[../concept/llm-wiki-pattern]]** Karpathy llm-wiki 模式 — OpenHuman 是**第 3 家** verified production(对照 openclaw / mempalace)
- **[[../concept/agent-memory]]** — **第 7 条 memory 路线**(SQLite + Obsidian 双写 + 20min auto-fetch + ≤3k chunks + hierarchical scoring)
- **[[../concept/multi-agent-isolation]]** — `fork_context.rs` 是**第 2 家** Fork-style KV-cache reuse verified(对照 CC)

### 新增
- **首家 _"Personal AI superintelligence"_ niche** harness
- **首家 Tauri 2.x stack verified**(Rust core in-process + React frontend + CEF Chromium)
- **业内独家 _"context compression as library"_ 抽象**(TokenJuice 3-layer rule overlay)
- **业内独家 _"借 Composio 平台拿 118+ OAuth 生态"_ 路线**
- **桌面 mascot + Google Meet 作为真实参与者** channel 路线
- **`agent/harness/` 子模块真叫这个名** — 跟 wiki concept 同名工程命名共识 anchor

### 校准 / 矛盾
- **QuickJS skills runtime 已移除**(AGENTS.md:14)— skills 执行表面在 rebuild,**标 [community-knowledge / project-in-flux]**
- **`openhuman-skills` 独立 repo** 未 ingest
- **Early Beta status** — production-grade 借鉴谨慎

## 待回答(下次 re-ingest 优先)

- `bash_blocklist / path_traversal / prompt_injection` 3 维 — 安全细节
- hooks 数量统计(`agent/hooks.rs` + `agent/stop_hooks.rs`)
- **QuickJS removal 后 skills 执行表面 rebuild 进度**(6 个月后 2026-11 re-ingest)
- `openhuman-skills` 独立 repo 是否值得单独 ingest
- **Memory Tree 评分算法细节** — `chunker.rs` + `tree_source/bucket_seal.rs` 是什么?
- **Learning 模块 `reflection.rs + stability_detector.rs` 算法** — 跟 [[../deep/openclaw-memory-dreaming-6-dim]] 6 维对比
