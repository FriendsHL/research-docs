---
type: harness
name: openhuman
status: active(Early Beta + project-in-flux)
language: rust+typescript

last_ingested_version: 0.53.49 (core) / 0.53.45 (app)
last_ingested_at: 2026-05-18
community_known_newer: false
known_post_ingest_features:
  - "QuickJS skills runtime 已 removed,execution surfaces 在 rebuild(AGENTS.md:14-22)"

sources:
  - "/Users/youren/myspace/openhuman/README.md"
  - "/Users/youren/myspace/openhuman/AGENTS.md"
  - "/Users/youren/myspace/openhuman/Cargo.toml"
  - "/Users/youren/myspace/openhuman/src/openhuman/agent/README.md"
  - "/Users/youren/myspace/openhuman/src/openhuman/memory/tree/schemas.rs"
  - "/Users/youren/myspace/openhuman/src/openhuman/tokenjuice/mod.rs"
  - "[[../source/openhuman]]"

agent_loop: |
  [verified-from-source]
  `src/openhuman/agent/` 完整模块,**`Agent` struct + `AgentBuilder` fluent + `run_single` / `run_interactive` turn lifecycle**(`harness/session/{builder,runtime,turn}.rs:23-27`)
  **`ToolDispatcher` trait** (`dispatcher.rs:14-50`) pluggable tool-call format(**XML / JSON / P-Format 3 种**)
  事件总线 `agent/bus.rs` + `DomainEvent::Agent(*)` + `Trigger*` events 跨 channel/cron/webhooks/composio 触发
  **`agent/multimodal.rs` first-class** + `agent/profiles.rs` + `agent/progress.rs`
  RPC:`agent.chat` / `agent.chat_simple` / `agent.server_status` / `agent.list_definitions` / `agent.get_definition` / `agent.reload_definitions` / `agent.triage_evaluate`(`schemas.rs:17-158` 7 个 controller)

tool_protocol: |
  [verified-from-source]
  `src/openhuman/tools/` 工具系统 + `tools/impl/memory/tree/` 3 memory tree tools verified
  **MCP 双向**:`src/openhuman/mcp_client/` + `mcp_server/` 都 first-class submodules
  **Composio 桥接 100+ external tool**(`src/openhuman/composio/` 独立模块)— **_"借平台拿生态"_ 路线**,不自写每个 OAuth
  **3 种 tool-call format**:XML / JSON / **P-Format** 自家格式(`agent/pformat.rs` + `dispatcher.rs:14-50`)— 业内多 format 选项罕见

context_mgmt: |
  [verified-from-source]
  **TokenJuice context compression first-class library**(`src/openhuman/tokenjuice/`)— `vincentkoc/tokenjuice` Rust port + **3-layer rule overlay**(Builtin `include_str!` + User `~/.config/tokenjuice/rules/` + Project `.tokenjuice/rules/` 优先级递增)
  5 子模块:`classify / reduce / rules / text / tool_integration / types`
  HTML→Markdown + URL 缩短 + dedup + 多字节 grapheme preserve(CJK/emoji 不被剥离)
  **降本 80%**(README claim)
  **v1 library only,无 JSON-RPC/CLI**(`tokenjuice/mod.rs:29-33`)— 工程诚实

memory: |
  [verified-from-source]
  **Memory Tree 业内最完整 _"Karpathy llm-wiki 模式"_ production 实现**(wiki 内第 3 家 verified,对照 [[../harness/openclaw]] Memory Wiki + [[../eval/mempalace]] Verbatim hierarchical)
  **17 JSON-RPC controllers**(`memory/tree/schemas.rs:27-48`):
  - 写侧:`ingest` / `trigger_digest` / `delete_chunk` / `flush_now` / `wipe_all` / `reset_tree`
  - 读侧:`list_chunks` / `get_chunk` / `list_sources` / `search` / `recall` / `entity_index_for` / `chunks_for_entity` / `top_entities` / `chunk_score` / `graph_export`
  - 配置:`get_llm` / `set_llm`(backend-selector UI)
  **`memory/` 18+ 子模块**:chunker / conversations / ingestion / ops / safety(独立 first-class)/ schemas / store / sync_status / tool_memory / tree/(13+ files)/ traits / global
  **数据流**:118+ OAuth → chunker ≤3k-token Markdown chunks → scored → folded hierarchical summary trees → **SQLite 本地**(`rusqlite "bundled"` `Cargo.toml:73`)+ **Obsidian-compatible vault `.md` 双写**
  **20-min auto-fetch loop**(README)— _"every twenty minutes the core walks each active connection and pulls fresh data into the memory tree"_

multi_agent: |
  [verified-from-source]
  **14 built-in agent archetypes**(`agent/agents/`):**orchestrator / planner / researcher / code_executor / critic / summarizer / archivist / skill_creator / tool_maker / integrations_agent / morning_briefing / help / welcome / trigger_triage / trigger_reactor**
  `harness/subagent_runner/` — `run_subagent` + `SubagentRunOptions` + `SubagentRunError`
  `harness/definition.rs` — `AgentDefinition` + `AgentDefinitionRegistry` + **`SandboxMode` enum + `ToolScope` enum**(built-in + workspace TOML loading,**跟 [[../harness/claude-code]] subagent loading 思路一致**)
  **`harness/fork_context.rs` KV-cache reuse**(task-local parent context)— **wiki 内第 2 家 verified Fork-style multi-agent**(对照 [[../harness/claude-code]] Fork mode)
  **`triage/` trigger 分类管线**:`run_triage` / `apply_decision` / `TriggerEnvelope` / `TriageDecision` / `TriageAction`(`triage/mod.rs:34-45`)— 外部 trigger → triage → escalate to sub-agent

eval_loop: |
  [verified-from-source]
  **无 LLM-based eval framework**(对照 [[../harness/openclaw]] QA Character Eval / [[../eval/swe-bench]])
  工程 test:Vitest(`app/`)+ WDIO E2E(`e2e/`)+ Rust integration(`tests/agent_*_public.rs`)
  Schema regression test(`schemas.rs:393-410` `controller_schema_inventory_is_stable`)防 RPC 表面破坏
  **Docker-based E2E harness**(`e2e/Dockerfile` + `docker-compose.yml` + `docker-entrypoint.sh`)

self_evolution: |
  [verified-from-source]
  **业内独特 _"User Profile Reflection 路线"_** —— 不是传统 prompt-evolution,而是 _"agent 通过 reflection 更新 user profile + tool tracker"_
  `src/openhuman/learning/` 18 文件:**`reflection.rs` ⭐ + `user_profile.rs` + `stability_detector.rs`(变化稳定性检测)+ `tool_tracker.rs`(tool 使用跟踪)+ `transcript_ingest/`(transcript → profile)+ `linkedin_enrichment.rs`(外部 enrich)+ `cache.rs` + `scheduler.rs` + `candidate.rs` + `extract/` + `profile_md_renderer.rs` + `prompt_sections.rs`**
  **跟 [[../deep/openclaw-memory-dreaming-6-dim]] _"6 维评分晋升 memory"_ 路线相邻但不同**:OpenClaw 是 _"memory 自我演化"_,OpenHuman 是 _"对 user 认知演化"_

observability: |
  [verified-from-source]
  `src/core/observability.rs` + `core/rpc_log.rs`(RPC log)+ `src/openhuman/agent/cost.rs`(cost tracking)
  **CEF 嵌入 Chromium**(`vendored Tauri/CEF sources` git submodule)— web 工具调试可能借力 Chrome DevTools
  无明确 OTel 集成(verified)— Tauri stack 自家 observability,不接 OTel ecosystem

hooks: |
  [verified-from-source]
  **`agent/hooks.rs` first-class 一等公民模块** + `agent/stop_hooks.rs` 独立 stop hooks
  跟 [[../harness/claude-code]] 27 hooks / [[../harness/hermes-agent]] 16 hooks 对照,数量未深查(file 存在 verified)

safety:
  sandbox: "[verified-from-source] **`SandboxMode` enum**(`agent/harness/definition.rs`)— sub-agent 级 sandbox 模式声明 + workspace TOML 配置;**桌面 app 整体 single-tenant local trust 模型**,没有 cross-tenant sandbox 需求。详见 [[../concept/sandbox-isolation]] §1 阶梯 0(本地 trust)"
  tool_acl: "[verified-from-source] **`ToolScope` enum**(`harness/definition.rs`)+ `src/openhuman/approval/` first-class submodule(approval flow)+ `src/openhuman/tools/ops.rs` operation-level ACL"
  bash_blocklist: "? 未深查(coder toolset 含 grep/lint/test 但 bash blocklist 未 verify)"
  path_traversal: "? 未深查"
  anti_runaway: "[verified-from-source] `harness/interrupt.rs` `InterruptFence` + `check_interrupt` + `InterruptedError` graceful cancellation primitives + `stop_hooks.rs` + agent loop step 控制"
  prompt_injection: "? 未深查"
  secret_scan: "[verified-from-source] **本地加密 3 道防御**:`aes-gcm` + `chacha20poly1305` + `argon2`(`Cargo.toml:46-47, 68`)+ `src/openhuman/encryption/` + `src/openhuman/credentials/` first-class submodules;**per-launch hex bearer auth `OPENHUMAN_CORE_TOKEN`**(in-process core RPC)"
  auth: "[verified-from-source] 桌面 app 单租户;Tauri host frontend RPC via HTTP 127.0.0.1:<port>/rpc + `OPENHUMAN_CORE_TOKEN` hex bearer per-launch(AGENTS.md:30);`OPENHUMAN_CORE_REUSE_EXISTING=1` debug 模式 attach 外部 process"

provider: |
  [verified-from-source]
  `src/openhuman/inference/` + **Model routing**(README:62)— reasoning / fast / vision 3 路由 one subscription
  **Ollama local AI**(README:62)+ **whisper-rs / llama.cpp**(local STT + local LLM,`Cargo.toml` GGML_NATIVE workaround 暗示)
  **`src/openhuman/local_ai/`** submodule 一等公民

channels: |
  [verified-from-source]
  **桌面 Tauri app**(Windows / macOS / Linux,`app/src-tauri/` desktop-only,**no Android/iOS branches** AGENTS.md:28)
  **桌面 mascot first-class**(`remotion` 视频渲染 mascot lip-sync)
  **Google Meet agent**(`src/openhuman/meet/` 一等 submodule)— **作为真实参与者加入会议**(README)
  **Messaging channels**(`src/openhuman/channels/` + `channels/runtime/dispatch.rs`)inbound/outbound
  **CEF 嵌入 Chromium**(vendored Tauri/CEF sources)— web 工具集成
  **CLI binary `openhuman-core`**(`src/main.rs`)

persistence: |
  [verified-from-source]
  **本地 SQLite**(`rusqlite "bundled"` `Cargo.toml:73`)— Memory Tree + 所有 state + 加密
  **Obsidian vault `.md` 双写**(memory tree chunks 同时落 markdown 到 Obsidian-compatible vault)
  **本地加密 3 道防御**:aes-gcm + chacha20poly1305 + argon2
  **In-process core via Tauri tokio task**(PR #1061 sidecar removed,AGENTS.md:30)

cancellation: |
  [verified-from-source]
  **`harness/interrupt.rs`** — `check_interrupt` + `InterruptFence` + `InterruptedError` graceful cancellation primitives
  `agent/stop_hooks.rs` 独立 stop hooks 模块
  **跟 [[../harness/agentscope]] _"interrupt 协议最完整"_**(fake ToolResultBlock + chunk-level is_interrupted)**思路相邻但实现不同** — OpenHuman 走 Rust 风 RAII `InterruptFence` guard

skillforge_takeaway: |
  ## 可借鉴

  ### 🟡 R-OH-1 P2 — Memory Tree _"≤3k 块 + 评分 + 折叠 + SQLite + Obsidian 双写"_ 模式(2-4 周技术评估)
  OpenHuman 是 Karpathy llm-wiki 模式 wiki 内**第 3 家** production 实现(对照 [[../harness/openclaw]] Memory Wiki + [[../eval/mempalace]] Verbatim hierarchical)。**17 JSON-RPC controllers** 表面结构化清晰,SkillForge memory 远期选型应基于 3 家对照决策。**关键 differentiator**:**Obsidian vault 双写** 让用户能直接 `.md` 编辑,**降低 _"memory 是黑盒"_ 信任成本** — SkillForge 多租户场景如果允许用户审 memory,可借鉴。

  ### 🟡 R-OH-2 P2 — TokenJuice _"context compression as library + 3-layer rule overlay"_(1-2 周)
  跟 [[../harness/claude-code]] 5 级 in-framework compaction / [[../harness/codex]] 3 套 compact 不同路线 — **抽出独立 library 让 framework 调用**,3-layer rule overlay(Builtin + User + Project)比 hardcoded 灵活。**SkillForge 可考虑把 compaction 抽成独立 library**(`skillforge-tokenjuice`?)给跨产品 SDK 复用。

  ### 🟢 R-OH-3 P3 — Composio 桥接 _"借平台拿 100+ OAuth 生态"_ 路线
  SkillForge 远期接外部集成时,**借 Composio 比自写每个 OAuth 工程量小 10x**,但 vendor lock-in 风险。**当前不必跟**,但保留作为 _"如果 SkillForge 想做 personal AI 方向 quick traction 加速器"_ 选项。

  ### 🟡 R-OH-4 P2 — `SandboxMode` + `ToolScope` enum-driven sub-agent 模型(2-3 周)
  `harness/definition.rs` `AgentDefinition` + `SandboxMode` enum + `ToolScope` enum + workspace TOML loading —— **跟 [[../harness/claude-code]] subagent loading 思路一致**,但用 Rust enum 强类型化。SkillForge subagent 配置可借鉴这种 _"enum-driven + TOML 配置"_ 模式。

  ### 🟢 R-OH-5 P3 — `fork_context.rs` KV-cache reuse(business-case decided 后再做)
  Wiki 内**第 2 家 verified Fork-style multi-agent**(对照 [[../harness/claude-code]] Fork mode)。SkillForge 远期 multi-agent 路线如果选 Fork(共享 parent context + cache),**OpenHuman + Claude Code 两家对照足够 derive 实施细节**。

  ### 🟢 R-OH-6 P3 — User Profile Reflection 路线(only personal AI 场景需要)
  `src/openhuman/learning/` 18 文件 _"agent reflect 更新 user profile + tool tracker + stability detector"_ —— SkillForge server-side B2B 定位**用不上 personal AI reflection**。但 _"`stability_detector.rs` 检测 user profile 变化稳定性"_ 思路可借鉴到 _"SkillForge memory 变化检测"_。

  ## SkillForge 现状对照

  - **Compaction library 抽出**:? 未 verify SkillForge 是否做了 compaction-as-library 抽象
  - **Subagent loading 路线**:? 未 verify SkillForge subagent 是 TOML / YAML / hardcoded
  - **Memory 表面**:? 未 verify SkillForge memory 是否 17-controller-level 细粒度 RPC

  ## 风险 / 不抄

  - **桌面 mascot + Google Meet first-class channel** — channel 路线跟 SkillForge server-side B2B 定位完全不匹配,**不抄**
  - **Tauri stack** — SkillForge JVM/Spring stack,**不抄整体技术栈**
  - **QuickJS skills runtime 已 removed**(AGENTS.md:14)— OpenHuman skills 系统 in-flux,**6 个月后 re-ingest 看 rebuild 后形态**
  - **Early Beta status** — _"current release has not been independently stress-tested at scale"_,**production-grade 借鉴谨慎**

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)
  - **业内首家 _"Personal AI superintelligence"_ niche** — wiki 内第一家这种定位,跟 coding agent / multi-agent framework / RAG framework 三大类目都不重叠
  - **Tauri 2.x + Rust core in-process(tokio task)+ React frontend + CEF 嵌入 Chromium** 桌面 app 完整 stack — **wiki 内首家 Tauri verified harness**
  - **Composio 桥接拿 118+ OAuth 集成** — _"借平台拿生态"_ 路线,跟 [[../harness/dify]] 自建 5 datasource type / [[../harness/ragflow]] 自建 connector 矩阵对比
  - **TokenJuice context compression 抽出独立 library**(`vincentkoc/tokenjuice` Rust port,3-layer rule overlay)— 业内独家(其他 harness 都 in-framework)
  - **桌面 mascot + Google Meet 作为真实参与者**(`src/openhuman/meet/` + `remotion` lip-sync)— 业内独家 channel 路线
  - **User Profile Reflection 路线**(`learning/` 18 文件)— 跟 [[../deep/openclaw-memory-dreaming-6-dim]] _"对 memory 评分晋升"_ 不同,**OpenHuman _"对 user 认知不断 reflect + stability detect"_**

  ## 优势(strengths,横向比谁做得对)
  - **17 个 Memory Tree JSON-RPC controllers 完整 RPC 表面**(`memory/tree/schemas.rs:27-48`)— 比 [[../harness/openclaw]] Memory Wiki 直接 file/wiki 操作更结构化
  - **Local-first encryption 3 道防御**(aes-gcm + chacha20poly1305 + argon2 + per-launch hex token)— 桌面 app 隐私设计认真,**对照 [[../eval/mempalace]] _"零 API call / 零 cloud / SQLite + ChromaDB"_ 哲学一致**
  - **TokenJuice 3-layer rule overlay**(Builtin via `include_str!` + User `~/.config/` + Project `.tokenjuice/`)— **承认 OSS prior art**(`vincentkoc/tokenjuice` Rust port),工程诚实
  - **In-process core via Tauri tokio task + per-launch hex token auth**(PR #1061 sidecar removed)— 工程精细化
  - **14 built-in agent archetypes + harness/definition.rs SandboxMode/ToolScope enum** — 跟 [[../harness/claude-code]] subagent 思路一致,Rust 强类型化
  - **`agent/harness/` 子模块**真叫这个名(跟 wiki concept 同名)— **工程命名共识 anchor**
  - **3 家 LLM dogfood**(CLAUDE.md + AGENTS.md + CODEX_WORKPAD.md)— 业内罕见 _"3 家 coding agent 都 dogfood"_

  ## 哪些点做得好(what_done_well,elegant 设计)
  - **README 嵌入 `html2md` 移除注释**(`Cargo.toml:28-37`)— 记录 _"为什么移除某依赖"_(894 MB peak heap on real Gmail HTML)+ 替代方案(`fast_html_to_text` linear-time tag-and-entity stripper)— **production engineering 透明工程诚实**
  - **`GGML_NATIVE=OFF` workaround**(AGENTS.md:71-72)— _"whisper-rs / llama.cpp on macOS Tahoe Apple Silicon failed with `-mcpu=native`"_ 显式记录解决方案
  - **QuickJS skills runtime removal 公开承认**(AGENTS.md:14, 22)— _"skill execution surfaces being rebuilt; check current domain modules before assuming a skill can run end-to-end"_ 工程诚实
  - **`memory/safety/` 独立 first-class submodule** — 把 memory safety 当独立关注点(不是放 _"misc"_ 文件夹)
  - **`agent/harness/interrupt.rs`** `InterruptFence + check_interrupt + InterruptedError` Rust RAII 风 graceful cancellation — 跟 [[../harness/agentscope]] _"interrupt 协议最完整"_ 思路相邻但 Rust 风更 elegant
  - **3 种 tool-call format**(XML / JSON / P-Format,`dispatcher.rs:14-50`)— 业内多 format 选项罕见
  - **20-min auto-fetch loop** — 不要求用户写 polling loop,framework 自动周期拉新数据,**降低用户认知负担** elegant
  - **Schema regression test**(`schemas.rs:393-410` `controller_schema_inventory_is_stable`)— 防 RPC 表面破坏,**API stability first-class**
---

# openhuman

## 一句话定位

**tinyhumansai 2026-05-13 launch 的 _"Personal AI superintelligence"_ Tauri 2.x 桌面 app**(v0.53.49 Early Beta),**Rust core in-process tokio task + React/Vite frontend + CEF 嵌入 Chromium**;核心 **Memory Tree + Obsidian Wiki 17 JSON-RPC + SQLite 本地 + 118+ OAuth(经 Composio)+ TokenJuice 三层 rule overlay token 压缩 + 桌面 mascot 加入 Google Meet**;wiki 内**首家 personal AI / local-first / OAuth-heavy / voice-native niche** harness。

## 起源与定位

**tinyhumansai** 2026-05-13 v0.53.43 launch,5 天爬上 GitHub Trending(2026-05-17 ~7.8k stars,629 forks)。Creator [@senamakel](https://x.com/intent/follow?screen_name=senamakel)。

**定位差异化**(跟 wiki 现有 11 harness 对照):
- 不是 coding agent(对照 [[claude-code]] / [[codex]] / [[opencode]] / [[smolagents]])
- 不是 multi-agent framework(对照 [[crewai]] / [[ag2]] / [[metagpt]] / [[agentscope]])
- 不是 RAG framework(对照 [[ragflow]] / [[dify]])
- 不是 self-evolution platform(对照 [[openclaw]] / [[hermes-agent]])
- **是 _"Personal AI superintelligence"_** —— 服务个人,长期 memory,本地优先,OAuth-heavy 桌面 app

## 核心架构(verified `AGENTS.md` + 源码)

```
┌──────────── Tauri Desktop App (Windows / macOS / Linux) ────────────┐
│                                                                       │
│  ┌─── app/ (pnpm workspace openhuman-app v0.53.45) ─────────────┐    │
│  │  app/src/         React + Vite frontend                       │    │
│  │  app/src-tauri/   Tauri desktop host (desktop-only)           │    │
│  │  app/test/        Vitest unit + OAuth login section tests    │    │
│  └────────────────────────────┬───────────────────────────────────┘   │
│                               │ HTTP RPC                              │
│                               │ http://127.0.0.1:<port>/rpc           │
│                               │ + OPENHUMAN_CORE_TOKEN per-launch hex │
│                               ▼                                       │
│  ┌─── Rust Core (in-process tokio task, PR #1061 sidecar removed) ─┐  │
│  │                                                                  │  │
│  │  src/core/        transport: HTTP/JSON-RPC + CLI + dispatch +    │  │
│  │                   auth + event_bus + socketio + observability    │  │
│  │                                                                  │  │
│  │  src/openhuman/   60+ 域子模块:                                  │  │
│  │     ├─ agent/     14 archetypes + harness/(fork_context +        │  │
│  │     │             interrupt) + subagent_runner + triage         │  │
│  │     ├─ memory/    18 sub: chunker / safety / tree/(17 RPC) ⭐   │  │
│  │     ├─ tokenjuice/ vincentkoc port + 3-layer rule overlay ⭐    │  │
│  │     ├─ composio/  Composio 桥接 100+ OAuth ⭐                    │  │
│  │     ├─ integrations/ 7 直接 (apify/twilio/...) + 100+ via composio│  │
│  │     ├─ mcp_client/ + mcp_server/ MCP 双向                       │  │
│  │     ├─ learning/  18 文件 user profile reflection 路线          │  │
│  │     ├─ meet/      Google Meet agent ⭐                          │  │
│  │     ├─ encryption/ + credentials/ + approval/  独立 first-class │  │
│  │     └─ inference/ + local_ai/  Model routing + Ollama + llama   │  │
│  │                                                                  │  │
│  │  e2e/ Docker-based WDIO E2E harness                              │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             ▼                                          │
│  ┌─── 持久化 ──────────────────────────────────────────────────┐      │
│  │  SQLite (rusqlite "bundled") + Obsidian vault .md 双写       │      │
│  │  本地加密 3 道:aes-gcm + chacha20poly1305 + argon2          │      │
│  └──────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────┘
```

## 13 维度详解

frontmatter 已 verified 详细。**3 大 differentiator**:

### Memory Tree(17 JSON-RPC controllers)— Karpathy llm-wiki 模式第 3 家 production

```
118+ OAuth (经 Composio)
   │ 20min auto-fetch loop
   ▼
chunker.rs (≤3k-token Markdown chunks)
   │
   ▼
scored + folded → hierarchical summary trees
   │
   ├── SQLite local (rusqlite bundled)
   └── Obsidian-compatible vault .md 双写  ← 让用户可以直接编辑!
   
RPC 表面:
- 写:ingest/trigger_digest/delete_chunk/flush_now/wipe_all/reset_tree
- 读:list_chunks/get_chunk/list_sources/search/recall/
       entity_index_for/chunks_for_entity/top_entities/chunk_score/graph_export
- 配置:get_llm/set_llm (backend-selector UI)
```

### TokenJuice — context compression as library 3-layer rule overlay

```
Builtin (include_str! 编译时嵌入)
  ↓ override
User    (~/.config/tokenjuice/rules/)
  ↓ override
Project (.tokenjuice/rules/ relative to cwd)
```

5 子模块:`classify / reduce / rules / text / tool_integration / types`
**v1 library only** — 无 JSON-RPC/CLI 表面(工程诚实)
Example:`bash + git status → "M: src/lib.rs"`(extreme compaction)
**降本 80%**

### Composio 桥接 — 借平台拿 100+ OAuth 生态

`src/openhuman/composio/` 独立模块桥接 [Composio](https://composio.dev)(Zapier for AI agents)
**不自写每个 OAuth** —— 跟 [[dify]] 自建 5 datasource / [[ragflow]] 自建 connector 矩阵对比,**工程量小 10x**

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway` 6 条 R-OH-N。最高 ROI 是 **R-OH-2 TokenJuice 抽出 compaction library**(1-2 周,跟 SkillForge 跨产品 SDK 复用直接相关)。

## 待回答的问题

- [ ] `bash_blocklist` / `path_traversal` / `prompt_injection` 3 维 — 桌面 app coder toolset 安全细节
- [ ] hooks 数量 — `agent/hooks.rs` + `agent/stop_hooks.rs` 具体注册了几个 hook
- [ ] **QuickJS 移除后 skills execution surfaces rebuild 进度**(6 个月后 re-ingest)
- [ ] `openhuman-skills` 独立 repo 是否值得单独 ingest
- [ ] **Memory Tree 评分算法细节** — `chunker.rs` 怎么评分 + `tree/tree_source/` `bucket_seal.rs` 是什么?
- [ ] **Learning 模块 `reflection.rs` + `stability_detector.rs` 算法细节** — 跟 [[../deep/openclaw-memory-dreaming-6-dim]] 6 维评分对比

## 引用链路

### 父概念
- [[../concept/llm-wiki-pattern]] — OpenHuman 是 wiki 内**第 3 家** Karpathy llm-wiki 模式 verified 实现(对照 [[openclaw]] / [[../eval/mempalace]])
- [[../concept/agent-memory]] — OpenHuman 是 wiki 内**第 7 条** memory 路线(_"SQLite+Obsidian dual-write + 20min auto-fetch + ≤3k chunks + hierarchical scoring"_)

### 相关 concept
- [[../concept/multi-agent-isolation]] — OpenHuman `harness/fork_context.rs` KV-cache reuse 是 wiki 内第 2 家 Fork-style verified
- [[../concept/context-compaction]] — TokenJuice 是 _"compaction as library"_ 独家路线
- [[../concept/agent-protocols]] §2 MCP — `mcp_client/` + `mcp_server/` 双向集成
- [[../concept/sandbox-isolation]] — 桌面 app 本地 trust 阶梯 0 + `SandboxMode` enum sub-agent 级声明

### 对照 harness
- [[claude-code]] — `harness/fork_context.rs` KV-cache 跟 CC Fork mode 思路一致;`SandboxMode/ToolScope` enum 跟 CC subagent loading 一致
- [[openclaw]] — Memory Wiki 同样 Karpathy llm-wiki 模式 production / OpenClaw 走 self-evolution / OpenHuman 走 user profile reflection
- [[agentscope]] — interrupt 协议;`InterruptFence` Rust RAII vs AgentScope fake ToolResultBlock + chunk-level is_interrupted

### 已知近期演进(post v0.53.49,unverified)
- QuickJS skills runtime removal — execution surfaces 在 rebuild
- 6 个月后(2026-11)re-ingest 看演化
