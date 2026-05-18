---
type: source
name: claude-code
verification: source-grounded
origin:
  - /Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88 (Claude Code v2.1.88 逆向源码归档)
  - /Users/youren/myspace/research-docs/research/claude code 源码/ (7 篇研究笔记)
ingested_at: 2026-05-15
ingest_mode: B (source-grounded)
contributed_to:
  - "[[harness/claude-code]]"
  - "[[concept/context-compaction]]"
  - "[[concept/self-evolution]]"
  - "[[concept/multi-agent-isolation]]"

sources_read:
  docs:
    - README.md (全文,30K+ tokens — KAIROS/BUDDY/Dream/ULTRAPLAN 概览)
    - package.json
    - 笔记 00 模块总览.md
  typescript:
    - "query.ts:215-340 (query async generator + queryLoop State)"
    - "Tool.ts: grep (PermissionMode, prePlanMode, tengu_tool_pear)"
    - "entrypoints/sdk/coreTypes.ts:25-53 (HOOK_EVENTS 27 个完整列表)"
    - "services/compact/autoCompact.ts:60-100 (阈值常量 + MAX_CONSECUTIVE_FAILURES=3)"
    - "services/compact/compact.ts: grep (5-level pipeline 函数)"
    - "services/autoDream/consolidationPrompt.ts (全文 — 4-phase prompt)"
    - "services/autoDream/autoDream.ts:1-100 (3-gate trigger + defaults)"
    - "services/SessionMemory/sessionMemory.ts:130-180 (shouldExtractMemory 触发逻辑)"
    - "services/extractMemories/prompts.ts: grep (auto / combined 2 prompts)"
    - "coordinator/coordinatorMode.ts:110-180 (Coordinator system prompt 网状)"
    - "tools/AgentTool/forkSubagent.ts: grep (FORK_AGENT, feature flag)"
    - "tools/AgentTool/AgentTool.tsx: grep (Fork inherits PARENT system prompt)"
    - "tools/AgentTool/prompt.ts: grep (forkEnabled 文档化指令)"
    - "utils/permissions/PermissionMode.ts: grep (5 modes + auto ant-only + bubble)"
    - "utils/permissions/permissions.ts: grep (tengu_iron_gate_closed, auto_mode events)"
    - "hooks/toolPermission/handlers/interactiveHandler.ts: grep (YOLO classifier)"
    - "skills/loadSkillsDir.ts:65-74 (LoadedFrom 6 sources union)"
    - "skills/bundledSkills.ts: grep (source/loadedFrom = 'bundled')"
  directories_listed:
    - "src/ (39 顶层文件/目录)"
    - "src/tools/ (40+ tools)"
    - "src/services/ (30+ services 含 autoDream/compact/SessionMemory/extractMemories/teamMemorySync/etc)"
    - "src/services/compact/ (11 files for 5-level pipeline)"
    - "src/services/autoDream/ (4 files)"
    - "src/services/teamMemorySync/ (5 files with secretScanner + secretGuard)"
    - "src/memdir/ (8 files including teamMemPaths/teamMemPrompts)"
    - "src/tools/AgentTool/ (14 files including forkSubagent + resumeAgent + loadAgentsDir)"
    - "src/coordinator/ (coordinatorMode.ts)"
    - "src/hooks/toolPermission/ (handlers/ + PermissionContext.ts + permissionLogging.ts)"
    - "src/utils/hooks/ (plugin hooks 实际位置,不是 src/hooks/ 那是 React UI hooks)"
  notes_cross_referenced:
    - "[[../../claude code 源码/00 模块总览]]"
---

# Source — Claude Code v2.1.88

## 核心论点

Claude Code v2.1.88 是 Anthropic 官方 CLI 工具的逆向归档,**远超"CLI"的 production-grade agent platform**:

1. **40+ tools / 30+ services / 27 hooks** — 工程深度数量级超 Hermes
2. **5 级 compaction pipeline**(budget → snip → microcompact → collapse → autocompact)— vs Hermes 单 ContextCompressor
3. **autoDream + SessionMemory + Team memory + extractMemories** 四层 memory 系统 — vs Hermes 三层
4. **Fork subagent **继承** parent context + cache** — 跟 Hermes `skip_memory=True` **完全相反**的设计选择
5. **Coordinator Mode 网状多 agent** + Task 系统 5 type — vs Hermes 仅 SubAgent depth=1
6. **5 permission modes + YOLO classifier ML + iron_gate** — vs Hermes 4 modes
7. **Bun feature flags + dead-code elimination** — internal vs external builds 编译期区分
8. **27 HOOK_EVENTS zod-validated** — vs Hermes 16 plugin hooks(无 schema validation)

定位差异 vs Hermes:Hermes 是"个人 multi-channel agent"(18+ messaging gateway),Claude Code 是"developer-facing IDE + CLI agent"(无 messaging gateway,但有 Bridge to claude.ai)。

## B 模式发现的事实(全部 verified)

| # | 事实 | 行号 | 备注 |
|---|---|---|---|
| 1 | **HOOK_EVENTS = 27 个** (zod-validated union literal) | `entrypoints/sdk/coreTypes.ts:25-53` | 笔记 07 数对 |
| 2 | **AUTOCOMPACT_BUFFER_TOKENS = 13_000** | `services/compact/autoCompact.ts:62` | 跟 effective_context_window 算 threshold |
| 3 | **MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3** | `services/compact/autoCompact.ts:71` | 注释:1279 sessions 50+ failures = 250K wasted API calls/day |
| 4 | **autoDream 3-gate**(time:24h, sessions:5, lock) | `services/autoDream/autoDream.ts:5-8, 63-65` | `tengu_onyx_plover` GB config |
| 5 | **autoDream 4-phase prompt**(Orient/Gather/Consolidate/Prune+Index) | `services/autoDream/consolidationPrompt.ts:26-65` | MEMORY.md ≤ 200 lines AND ≤ 25KB |
| 6 | **5 Permission modes**(default/plan/autoAccept/bypassPermissions/auto) `auto` 是 **ant-only**) | `utils/permissions/PermissionMode.ts:21-95` | README 说的 "yolo" 实际不是模式,是 classifier |
| 7 | **YOLO classifier ML** for auto-approval | `hooks/toolPermission/handlers/interactiveHandler.ts:25, 483` | `setYoloClassifierApproval` |
| 8 | **Fork subagent 继承 parent system prompt** | `tools/AgentTool/AgentTool.tsx:485` | "Fork path: child inherits the PARENT's system prompt" |
| 9 | **Fork 默认 directive prompt**(不是 standalone)| `tools/AgentTool/prompt.ts:95` | "Since the fork inherits your context, the prompt is a directive" |
| 10 | **FORK_SUBAGENT_TYPE = 'fork'** + `feature('FORK_SUBAGENT')` gate | `tools/AgentTool/forkSubagent.ts:42` | Bun compile-time feature flag |
| 11 | **SessionMemory 触发**:token AND tool-call 阈值 OR (token + natural break) | `services/SessionMemory/sessionMemory.ts:134-181` | Token threshold **always required** |
| 12 | **Skills 6 LoadedFrom sources** | `skills/loadSkillsDir.ts:67-74` | `commands_DEPRECATED \| skills \| plugin \| managed \| bundled \| mcp` |
| 13 | **Coordinator system prompt**(网状,Workers async,parallel)| `coordinator/coordinatorMode.ts:111-330+` | `AGENT_TOOL_NAME spawn` / `SEND_MESSAGE_TOOL_NAME continue` / `TASK_STOP_TOOL_NAME stop` |
| 14 | **`<task-notification>` XML** worker results format | `coordinator/coordinatorMode.ts:145-156` | `<task-id> <status> <summary> <result> <usage>` |
| 15 | **Team memory secret scanner + guard** | `services/teamMemorySync/secretScanner.ts, teamMemSecretGuard.ts` | Hermes 没有 |
| 16 | **27 HOOK_EVENTS zod-validated** | `entrypoints/sdk/coreSchemas.ts:355-572+` | 每 hook event 单独 zod schema |
| 17 | **`tengu_iron_gate_closed`** 高危 gate | `utils/permissions/permissions.ts:848` | + `tengu_auto_mode_decision` events |
| 18 | **`taskBudgetRemaining`** 跨 compaction boundary | `query.ts:282-292` | 长注释解释 server-side counting limitation |
| 19 | **`startRelevantMemoryPrefetch` + `using` ES Stage 3** | `query.ts:301-304` | 比 Hermes prefetch_all 更现代 |

## 关键事实(精确数字,全部 verified)

| 类别 | 事实 | 来源 |
|---|---|---|
| 语言 / 体量 | TypeScript + Bun + Ink React 19,`main.tsx` 4683 行,`query.ts` 1729 行,`Tool.ts` 792 行,`services/compact/compact.ts` 1568 行 | wc -l |
| 工具数 | **40+** tools (`src/tools/` ls) | README + ls |
| 服务数 | **30+** services | `src/services/` ls |
| 入口数 | CLI / Ink TUI / Bridge(to claude.ai)/ Remote / MCP | `src/entrypoints/` |
| Hook 事件数 | **27**(zod-validated)| `coreTypes.ts:25-53` |
| Hook 类型 | 5(command / prompt / http / agent / callback) | 笔记 07 |
| Compaction 级数 | **5**(budget/snip/microcompact/collapse/autocompact) | `services/compact/` |
| Permission modes | **5**(default/plan/autoAccept/bypassPermissions/auto)| `PermissionMode.ts` |
| Task types | 5(LocalAgent/RemoteAgent/LocalShell/InProcessTeammate/Dream)| `Task.ts:6-13` |
| Skills 来源 | **6**(commands_DEPRECATED/skills/plugin/managed/bundled/mcp) | `loadSkillsDir.ts:67-74` |
| Multi-agent 模式 | 3(AgentTool 普通 SubAgent / Fork / Coordinator 网状) | `tools/AgentTool/` + `coordinator/` |
| autoCompact 阈值 | `ctx - 13K` buffer | `autoCompact.ts:62, 103-105` |
| autoCompact warning | `ctx - 20K` | `autoCompact.ts:63` |
| autoCompact 失败上限 | 3 次连续 | `autoCompact.ts:71` |
| autoDream gate | 24h + 5 sessions + lock | `autoDream.ts:63-65` |
| MEMORY.md 上限 | ≤ 200 lines AND ≤ 25KB | `consolidationPrompt.ts:55` |
| Provider | 单 Anthropic SDK + MCP SDK | `package.json:19-20` |
| Messaging gateway | **无**(对比 Hermes 18+) | `gateway/` 无该目录 |
| Bridge target | claude.ai(JWT auth) | `bridge/` |
| Feature gates | Bun compile-time `feature()` + GrowthBook runtime | README + `getFeatureValue_CACHED_MAY_BE_STALE` 大量 |

## 跟 Hermes 的关键对比

| 维度 | Hermes | Claude Code | 差异本质 |
|---|---|---|---|
| Agent loop | sync `while` in `run_conversation()` | async generator `query()` + `queryLoop` `while(true)` | Python sync vs TS async-gen |
| Hooks | 16 plugin hooks | 27 zod-validated HOOK_EVENTS | Claude Code 多 PostCompact / SubagentStart / TaskCreated / WorktreeCreate / FileChanged 等 IDE-aware events |
| Compaction | 1 个 ContextCompressor(7 步内部) | 5 级 pipeline | Claude Code 分层更细 |
| Memory | 3 层(SessionDB / MEMORY.md / 8 external) | 4 系统(memdir autoMemory / SessionMemory / Team memory / extractMemories)| Claude Code 加 Team memory + 自动 SessionMemory |
| Subagent | 仅深度 1,child `skip_memory=True` | Fork **继承** parent context + cache **OR** 普通 SubAgent 隔离 OR Coordinator 网状 | **完全相反 + 多模式** |
| Permission | 4 modes(on/auto/off/smart) | 5 modes + YOLO classifier ML + iron_gate | Claude Code 更细分 + ML |
| Self-evolution | `commit_memory_session` simple trigger | autoDream 3-gate + 4-phase + SessionMemory + extractMemories | Claude Code 系统化 |
| Provider | 4 api_mode 自抽象(200+ 模型) | 单 Anthropic SDK | Hermes 多 provider 优势 |
| Channels | 18+ messaging gateway | 仅 CLI/TUI/Bridge to claude.ai | Hermes multi-channel 优势 |
| Trust model | single-tenant operator(明确) | 复杂(USER_TYPE=ant 内部 vs external,Team memory 共享) | Claude Code 多租户 partial |
| Eval/observability | 仅日志 + RL trajectories | analytics + GrowthBook + tengu_* events | Claude Code 商业级 |
| Codebase | Python ~12K LOC main agent | TypeScript ~4.7K LOC main + 大量 services | Claude Code 更分散更模块化 |

## 与现有 wiki 的关系

- **印证**:[[harness/hermes-agent]] 的所有概念在 Claude Code 都有对应,但**更细分、更 production**
- **矛盾**(关键!):**Fork subagent 继承 vs Hermes `MAX_DEPTH=1 + skip_memory=True`** — 两种相反设计选择。本 ingest 新建 [[concept/multi-agent-isolation]] 专门讨论 trade-off
- **新增**:autoDream 4-phase / SessionMemory 触发逻辑 / Team memory + secret scan / Coordinator 网状 / 27 hooks 列表 / 5 permission modes / 5 级 compaction
