---
type: harness
name: claude-code
status: active
language: typescript
verification: source-grounded
ingest_mode: B
updated: 2026-05-16

# 版本时效
last_ingested_version: v2.1.88
last_ingested_at: 2026-05-15
community_known_newer: true
known_post_ingest_features:
  - goal (目标系统)
  - agent-view (agent 可视化)
  - agent-manager (agent 管理增强)
  - **/insights**(2026-05-17 verified: `src/commands/insights.ts` 3200 行 — 6 维 55 typed enum + MULTIPLE TIMES algorithm + 8 INSIGHT_SECTIONS,详见 [[../deep/claude-code-insights]])
sources:
  - "[[../source/claude-code]]"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/"
  - "[[../../claude code 源码/00 模块总览]]"
  - "[[../../claude code 源码/01 Agent 主循环]]"
  - "[[../../claude code 源码/06 上下文压缩流水线]]"
  - "[[../../claude code 源码/07 Hooks 系统]]"

agent_loop: |
  [verified-from-source]
  Async generator `query()` (`query.ts:219`) + 主循环 `queryLoop()` (`:241`),`while(true)` (`:307`)。
  Mutable State 9 字段 (`query.ts:268-279`): messages / toolUseContext / autoCompactTracking /
  stopHookActive / hasAttemptedReactiveCompact / turnCount / pendingToolUseSummary /
  transition / maxOutputTokensOverride。
  `taskBudgetRemaining` 跨 compaction boundary 追踪(`:282-292`,server-side counting limitation
  长注释)。
  `using pendingMemoryPrefetch = startRelevantMemoryPrefetch(...)` (`:301-304`) ES Stage 3 自动 dispose。
  `feature('TOKEN_BUDGET')` Bun compile-time gate (`:280`)。
  笔记 01 说"5 个 query() 调用点,queryLoop 13 步"(本次未深读 verify)。

tool_protocol: |
  [verified-from-source]
  `Tool.ts` 792 行基类,`getAllBaseTools()` (`tools.ts:193`) 返回 40+ tools。
  Tool 字段 (`Tool.ts:124, 137, 394`): mode: PermissionMode / prePlanMode? / inputSchema。
  `tengu_tool_pear` feature gate (`Tool.ts:470`)。
  `toolSchemaCache.ts` 缓存 JSON schema(prompt 效率)。
  40+ tools 各自子目录 (`tools/` ls): AgentTool / BashTool / FileReadTool / FileEditTool /
  FileWriteTool / GlobTool / GrepTool / WebFetchTool / WebSearchTool / TaskCreateTool /
  TeamCreateTool / TodoWriteTool / SkillTool / REPLTool / LSPTool / NotebookEditTool /
  AskUserQuestionTool / EnterPlanModeTool / ExitPlanModeTool / EnterWorktreeTool /
  ExitWorktreeTool / ListMcpResourcesTool / ReadMcpResourceTool / SleepTool /
  SnipTool / ToolSearchTool / SendMessageTool / RemoteTriggerTool / ScheduleCronTool /
  PowerShellTool / BriefTool / SyntheticOutputTool ...

context_mgmt: |
  [verified-from-source]
  **5 级 pipeline** (`services/compact/` 11 files):

  1. **budget**: `query/tokenBudget.ts` + `feature('TOKEN_BUDGET')`
  2. **snip**: `tools/SnipTool/` — 历史 snippet 提取
  3. **microcompact**:
     - `services/compact/microCompact.ts` — 普通 microcompact
     - `services/compact/apiMicrocompact.ts` — API 层 microcompact
     - `services/compact/timeBasedMCConfig.ts` — GrowthBook 时间触发(主循环间隔超阈值)
  4. **collapse**:
     - `utils/collapseReadSearch.ts` — 折叠 read/search tool 结果摘要
     - `utils/collapseHookSummaries.ts` — 折叠 hook 摘要
  5. **autocompact**: `services/compact/autoCompact.ts`

  默认阈值常量 (`services/compact/autoCompact.ts:62-71`):
  - `AUTOCOMPACT_BUFFER_TOKENS = 13_000` (autocompact 在 ctx 还剩 13K 时触发)
  - `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000` (UI warning 在 ctx 还剩 20K 时显示)
  - `ERROR_THRESHOLD_BUFFER_TOKENS = 20_000`
  - `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`
  - `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` (注释:BQ 2026-03-10 — 1,279 sessions 50+ 连续失败 = 250K wasted API calls/day)
  - env override: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`
  - `getAutoCompactThreshold(model)` = `effectiveContextWindow - 13_000` (`:103-105`)

  主 compact 函数 (`services/compact/compact.ts`,1568 行):
  - `compactConversation` (`:387`) — 主入口
  - `partialCompactConversation` (`:772`)
  - `truncateHeadForPTLRetry` (`:243`) — reactive 413 recovery path
  - `mergeHookInstructions` (`:374`) — hook 注入摘要指令到 compaction prompt
  - `annotateBoundaryWithPreservedSegment` (`:349`) — suffix-preserving (reactive / session-memory) last summary
  - `buildPostCompactMessages` (`:330`)
  - `createCompactCanUseTool` (`:1125`) — fork agent 的 canUseTool
  - `createPostCompactFileAttachments` (`:1415`) + `createPlanAttachmentIfNeeded` / `createSkillAttachmentIfNeeded` / `createPlanModeAttachmentIfNeeded` / `createAsyncAgentAttachmentsIfNeeded`

  详见 [[../concept/context-compaction]]。

memory: |
  [verified-from-source]
  **4 系统**:

  ① **autoMemory** (`memdir/` 8 files):
     `memdir.ts` + `findRelevantMemories.ts` + `memoryAge.ts` + `memoryScan.ts` +
     `memoryTypes.ts` + `paths.ts` + `teamMemPaths.ts` + `teamMemPrompts.ts`
     MEMORY.md 上限:**≤ 200 lines AND ≤ 25KB** (`consolidationPrompt.ts:55`)
     `getAutoMemPath()` per-user storage,`isAutoMemoryEnabled()` (`autoDream.ts:98`)

  ② **SessionMemory** per-session 提取 (`services/SessionMemory/sessionMemory.ts`):
     `tengu_session_memory` feature gate (`:81`)
     `shouldExtractMemory(messages)` (`:134-181`) 触发逻辑:
       (hasMetTokenThreshold AND hasMetToolCallThreshold)
       OR (hasMetTokenThreshold AND !hasToolCallsInLastTurn) — natural break
       Token threshold **always required**(防过度提取)
     `extractSessionMemory()` (`:272`) sequential async — querySource='session_memory',forkLabel='session_memory'
     `manuallyExtractSessionMemory()` (`:387`) 手动触发
     Per-session memory dir mode 0o700

  ③ **Team memory** (`services/teamMemorySync/` 5 files):
     `index.ts` + `secretScanner.ts` + `teamMemSecretGuard.ts` + `types.ts` + `watcher.ts`
     **secret scanner + guard** 防 secrets 共享(Hermes 完全没有)
     watcher 监控 team memory 变化

  ④ **extractMemories** service (`services/extractMemories/prompts.ts`):
     2 个 prompt 变种:
       `buildExtractAutoOnlyPrompt(...)` (`:50`) — 仅 auto memory
       `buildExtractCombinedPrompt(...)` (`:101`) — auto + team 联合
     说明 MEMORY.md 200 lines truncated (`:78`):"keep the index concise"

  **三层 memory file 大小约定**(consolidationPrompt.ts:55):
     - Entrypoint 文件 (MEMORY.md):≤ 200 lines AND ≤ 25KB
     - 每条 index 行 < 150 chars (`- [Title](file.md) — hook`)
     - 超过 200 chars 的 entry 应该 demote 到 topic file

multi_agent: |
  [verified-from-source]
  **3 模式 + 1 系统**:

  ① **AgentTool 普通 SubAgent** (`tools/AgentTool/` 14 files):
     - `loadAgentsDir.ts` — user 自定义 agent
     - `built-in/` + `builtInAgents.ts` — 内置 agent type
     - `agentColorManager.ts` / `agentDisplay.ts` / `agentMemory.ts` / `agentMemorySnapshot.ts`
     - `AgentTool.tsx` — 主工具实现
     - `prompt.ts` — agent 创建 prompt
     - `runAgent.ts` + `resumeAgent.ts` — 创建 + 恢复
     - `UI.tsx` — Ink 渲染
     - Agent 创建后**可 resume**(`resumeAgent.ts`),Hermes 没有

  ② **FORK_SUBAGENT** (`tools/AgentTool/forkSubagent.ts:42`):
     `FORK_SUBAGENT_TYPE = 'fork'`
     `feature('FORK_SUBAGENT')` Bun compile-time gated
     **Fork 继承 parent system prompt + context + cache** (`AgentTool.tsx:485`):
       "Fork path: child inherits the PARENT's system prompt (not FORK_AGENT's)"
     Fork prompt 是 **directive**(不是 standalone briefing,`prompt.ts:95`):
       "Since the fork inherits your context, the prompt is a directive — what to do, not what the situation is"
     Resume 也可:`resumeAgent.ts:102-145`(reconstruct parent system prompt)
     **跟 Hermes `MAX_DEPTH=1 + skip_memory=True` 完全相反的设计**
     详见 [[../concept/multi-agent-isolation]]

  ③ **Coordinator Mode** (`coordinator/coordinatorMode.ts:36`):
     `isCoordinatorMode()` feature gate
     `getCoordinatorSystemPrompt()` (`:111-330+`) 系统 prompt 给 coordinator:
       "You are a coordinator. Your job is to:
        - Help the user achieve their goal
        - Direct workers to research, implement and verify code changes
        - Synthesize results and communicate with the user"
     3 工具 (`:127-136`):
       AGENT_TOOL_NAME — spawn worker
       SEND_MESSAGE_TOOL_NAME — continue worker(用 agent ID `to` field)
       TASK_STOP_TOOL_NAME — stop worker
     Workers **async + parallel** (`:213`):"Parallelism is your superpower"
     `<task-notification>` XML 格式 (`:145-156`):
       <task-id> <status> <summary> <result> <usage>

  ④ **Task 系统** (`Task.ts` 125 行 + `tasks/`):
     5 task types (`Task.ts:6-13`):
       LocalAgentTask / RemoteAgentTask / LocalShellTask /
       InProcessTeammateTask / DreamTask
     `TaskCreated` / `TaskCompleted` hook events (`coreTypes.ts:43-44`)

eval_loop: |
  [doc-only]
  内部 **transcript classifier**(AFK ML — 笔记 README 提及)
  **YOLO classifier**(permission ML — `hooks/toolPermission/handlers/interactiveHandler.ts:25, 483`)
  `services/analytics/` 大量 `tengu_*` GrowthBook events
  无对外暴露的 prompt A/B / dataset eval pipeline UI(跟 Hermes 一样,SkillForge Eval Pipeline 反而更前)

self_evolution: |
  [verified-from-source]
  **autoDream**(README 称 "Dream system" — Claude 字面意义"做梦",`services/autoDream/` 4 files):

  **3-gate trigger**(`autoDream.ts:5-8`,cheapest first):
  1. **Time gate**: hours since `lastConsolidatedAt` ≥ `minHours` (default 24)
  2. **Session gate**: transcript count with mtime > lastConsolidatedAt ≥ `minSessions` (default 5)
  3. **Lock gate**: no other process mid-consolidation(`consolidationLock.ts`)

  **配置** (`autoDream.ts:63-65`):
  - `minHours = 24` 默认
  - `minSessions = 5` 默认
  - `tengu_onyx_plover` GrowthBook 配置覆盖
  - `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` (10 min throttle)

  **Skip 条件** (`autoDream.ts:95-99`):
  - `getKairosActive()` — KAIROS 模式跑自己的 dream
  - `getIsRemoteMode()`
  - `!isAutoMemoryEnabled()`
  - `!isAutoDreamEnabled()` (`config.ts`)

  **4-phase prompt** (`consolidationPrompt.ts`):
  - **Phase 1 — Orient** (`:26-31`):`ls` memory dir, read entrypoint, skim existing
  - **Phase 2 — Gather recent signal** (`:33-43`):daily logs > drifted memories > narrow transcript grep `grep -rn "<term>" ${transcriptDir}/ --include="*.jsonl"`
  - **Phase 3 — Consolidate** (`:44-52`):write/update memory files, 绝对日期, 删 contradicted
  - **Phase 4 — Prune and index** (`:53-61`):keep entrypoint ≤ 200 lines AND ≤ 25KB,demote >200 chars entries

  **Read-only bash** for dream subagent(README:234 + `createAutoMemCanUseTool` `extractMemories.ts`)

  **Forked subagent** 跑(独立 context,但 fork 继承 parent system prompt — 这里指 dream agent fork)

  **SessionMemory + extractMemories** 联动(见 memory 维度)

  详见 [[../concept/self-evolution]]。

observability: |
  [doc-only]
  `services/analytics/` 目录(未深读)
  `tengu_*` GrowthBook events 大量(auto_mode_decision / iron_gate_closed / session_memory_*  / auto_dream / etc)
  `getFeatureValue_CACHED_MAY_BE_STALE()` 防 main loop 阻塞
  `services/diagnosticTracking.ts` + `services/internalLogging.ts` 存在
  ANT-only:`~/.config/claude/dump-prompts/` debug prompt dumping(README)
  **未见 Langfuse 风格 trace UI / session replay UI / model usage dashboard**(SkillForge 反而更前)

hooks: |
  [verified-from-source]
  **HOOK_EVENTS = 27 个**,zod-validated literal union(`entrypoints/sdk/coreTypes.ts:25-53` + `coreSchemas.ts:355-572+`):

  ```
  PreToolUse, PostToolUse, PostToolUseFailure,
  Notification, UserPromptSubmit,
  SessionStart, SessionEnd, Stop, StopFailure,
  SubagentStart, SubagentStop,
  PreCompact, PostCompact,
  PermissionRequest, PermissionDenied, Setup,
  TeammateIdle,
  TaskCreated, TaskCompleted,
  Elicitation, ElicitationResult,
  ConfigChange, WorktreeCreate, WorktreeRemove,
  InstructionsLoaded, CwdChanged, FileChanged
  ```

  **5 hook 类型**(笔记 07):command / prompt / http / agent / callback

  **ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']** (`utils/hooks/hookEvents.ts:18`)

  **Plugin hooks 实际位置**:`utils/hooks/`(不是 `src/hooks/` — 那是 React UI hooks):
  - `AsyncHookRegistry.ts` (line 220 `isSessionStart` check)
  - `execAgentHook.ts` — agent hook 执行
  - `execPromptHook.ts` — prompt hook 执行
  - `hooksConfigManager.ts` — 配置管理
  - `registerFrontmatterHooks.ts` (line 39-41:agent stop event 映射为 SubagentStop)
  - `sessionHooks.ts` + `hookEvents.ts` + `hookHelpers.ts`
  - `postSamplingHooks.ts` (from autoDream import)

  **跟 Hermes 16 hooks 对比** Claude Code 多 11 个:
  - PostToolUseFailure / StopFailure(失败专门 hook)
  - SubagentStart(成对 with SubagentStop)
  - PostCompact(compaction 完成观察,Hermes 仅内部)
  - PermissionRequest / PermissionDenied(细分 approval lifecycle)
  - TeammateIdle(网状协作)
  - TaskCreated / TaskCompleted(Task 系统)
  - Elicitation / ElicitationResult(用户澄清 lifecycle)
  - ConfigChange / WorktreeCreate / WorktreeRemove / InstructionsLoaded / CwdChanged / FileChanged(IDE-aware events)

safety:
  permission_modes: "[verified-from-source] **5 modes** (`utils/permissions/PermissionMode.ts:21-95`): default / plan / autoAccept / bypassPermissions / **auto**(ant-only,外部不可见,line 95). 还可能有 bubble(line 104 `!== 'bubble'`)"
  yolo_classifier: "[verified-from-source] ML-based permission auto-approval — `setYoloClassifierApproval` (`hooks/toolPermission/handlers/interactiveHandler.ts:25, 483`). Permission Explainer 是独立 LLM call(README,[doc-only])"
  iron_gate: "[verified-from-source] `tengu_iron_gate_closed` 高危 gate (`utils/permissions/permissions.ts:848`),配合 `tengu_auto_mode_decision` events (lines 626/666/733/1009) + `tengu_auto_mode_denial_limit_exceeded` (line 1009)"
  protected_files: "[doc-only] README:.gitconfig / .bashrc / .zshrc / .mcp.json / .claude.json + 等. 未深读源码 verify 完整 list"
  path_traversal: "[doc-only] URL-encoded / Unicode normalization / backslash injection / case-insensitive — 防 (README). 未深读源码"
  team_memory_secret_guard: "[verified-from-source] **secret scanner + guard** (`services/teamMemorySync/secretScanner.ts, teamMemSecretGuard.ts`) — 防 team-shared memory 包含 secrets,Hermes 完全没有"
  anti_runaway_autocompact: "[verified-from-source] `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` (`services/compact/autoCompact.ts:71`) + 注释 BQ 数据(1,279 sessions × 50+ failures = 250K wasted API calls/day)"
  permission_logging: "[doc-only] `hooks/toolPermission/permissionLogging.ts`"
  feature_dead_code_elimination: "[verified-from-source partial] Bun `feature()` compile-time + dead-code elimination — internal features(KAIROS/BUDDY/PROACTIVE/COORDINATOR_MODE 等)外部 build 不出现"
  ant_only_features: "[verified-from-source partial] `USER_TYPE === 'ant'` gate(README) — staging API / Undercover mode / ConfigTool / TungstenTool / `/security-review` command / dump-prompts"

provider: |
  [doc-only]
  单 `@anthropic-ai/sdk` v0.80 (`package.json:19`) + `@modelcontextprotocol/sdk` v1.29 (`:20`)
  `services/api/` 未深读
  **Beta headers**(README 提到):
    interleaved-thinking-2025-05-14, context-1m-2025-08-07, structured-outputs-2025-12-15,
    web-search-2025-03-05, advanced-tool-use-2025-11-20, effort-2025-11-24,
    task-budgets-2026-03-13, prompt-caching-scope-2026-01-05, fast-mode-2026-02-01,
    redact-thinking-2026-02-12, token-efficient-tools-2026-03-28, afk-mode-2026-01-31,
    cli-internal-2026-02-09, advisor-tool-2026-03-01, summarize-connector-text-2026-03-13
  **无 Hermes 那种 4 api_mode 多 provider 抽象**(单 Anthropic + MCP)

channels: |
  [verified-from-source partial]
  - **CLI** (`entrypoints/cli.tsx`)
  - **Ink TUI**(`ink/`,React 19)
  - **Bridge to claude.ai**(`bridge/`,JWT auth + 3 modes:single-session/worktree/same-dir)
  - **Remote**(`remote/`)+ DirectConnect(`server/directConnectManager`)
  - **WebSocket** session(`SessionsWebSocket`)
  - **MCP server**(`mcp_serve.ts` 等等)
  KAIROS 工具:**PushNotification / SendUserFile / SubscribePR**(主动通知 — Hermes 无)
  **无 messaging gateway**(对比 Hermes 18+ Telegram/Discord/Slack/Feishu/etc)
  定位差异:Claude Code 是 developer CLI,Hermes 是 multi-channel personal agent

persistence: |
  [doc-only]
  `~/.config/claude/` (per-user storage)
  `getClaudeConfigHomeDir()` (`loadSkillsDir.ts:83-85` 上下文)
  `services/SessionMemory/sessionMemoryUtils.ts` (未深读)
  `memdir/paths.ts` (未深读)
  Bridge JWT tokens + trusted device tokens
  Transcripts: `.jsonl` files

cancellation: |
  [doc-only]
  `Stop` / `StopFailure` hooks (`coreTypes.ts:33-34`)
  `hooks/useCancelRequest.ts` UI 层
  `TASK_STOP_TOOL_NAME` Coordinator 用于 stop worker
  未深读 `query.ts` 取消机制完整

skillforge_takeaway: |
  ## 直接可借鉴(高优先级,带源码锚点)

  1. **[HIGH] 27 hooks zod-validated** (`entrypoints/sdk/coreTypes.ts:25-53` + `coreSchemas.ts:355+`)
     - SkillForge 应加 hooks(对比 Hermes 16):
       * `PostToolUseFailure` / `StopFailure`(失败专门 hook,便于调试 + retry logic)
       * `PostCompact`(compaction 完成观察)
       * `SubagentStart`(成对 with SubagentStop)
       * `PermissionRequest` / `PermissionDenied`(细分 approval lifecycle)
       * `TeammateIdle`(网状协作)
       * `Elicitation` / `ElicitationResult`(用户澄清 lifecycle 而非单点)
     - zod schema validation 是 SkillForge 类型安全的范式

  2. **[HIGH] 5 级 compaction pipeline** (`services/compact/` 11 files)
     - SkillForge Light/Full 两档可拆为:budget → snip → microcompact → collapse → autocompact
     - 特别是 **microcompact**(时间触发的小裁剪)和 **collapse**(read/search 折叠摘要)
     - `MAX_CONSECUTIVE_FAILURES=3` + BQ 数据印证(250K wasted API calls/day)

  3. **[HIGH] autoDream 3-gate + 4-phase prompt** (`services/autoDream/`)
     - SkillForge 自进化模块的系统化蓝本(对比 Hermes simple `commit_memory_session`)
     - 3-gate(time/sessions/lock cheapest-first)+ 4-phase prompt(orient/gather/consolidate/prune)
     - `getKairosActive()` / `getIsRemoteMode()` 跳过条件值得借鉴(避免 mode 互冲)

  4. **[HIGH] Fork subagent 模式**(继承 parent context + cache)(`tools/AgentTool/forkSubagent.ts` + `AgentTool.tsx:485`)
     - SkillForge SubAgent 当前是隔离(对应 Hermes),应**同时支持 Fork 模式**
     - 两种 trade-off 不同:**隔离**(独立 context,无污染但 cold start)vs **继承**(共享 context + cache,但易污染)
     - 详见 [[../concept/multi-agent-isolation]]

  5. **[HIGH] SessionMemory 触发逻辑**(`services/SessionMemory/sessionMemory.ts:134-181`)
     - (token AND tool-call) OR (token AND natural break),token threshold **always required**
     - SkillForge memory extraction 触发可对照 — 比 Hermes 在 compaction rotation 前 trigger 更主动

  6. **[HIGH] Team memory secret scanner + guard** (`services/teamMemorySync/secretScanner.ts, teamMemSecretGuard.ts`)
     - SkillForge 多用户/Team 场景**必须有**(Hermes 完全没考虑)
     - Watcher 监控 team memory 变化也是 SkillForge 协作场景需要的

  7. **[HIGH] 5 Permission modes** (`utils/permissions/PermissionMode.ts:21-95`)
     - SkillForge 当前 approval 系统对比 Hermes 4 modes
     - Claude Code 多了 `plan` (plan mode 专用) + `autoAccept`(类似 Hermes `auto` 但更明确)
     - `bypassPermissions` 是 break-glass,跟 Hermes `off` 等价
     - **`auto` mode ant-only** — SkillForge 多租户应有等价 internal/external 区分

  8. **[HIGH] YOLO classifier ML** + **Permission Explainer LLM call**
     - SkillForge 可借鉴:ML 模型预判 + LLM 解释危险性
     - SkillForge `compact-reviewer` 等 reviewer agent 是同思路

  9. **[HIGH] `<task-notification>` XML format** (`coordinator/coordinatorMode.ts:145-156`)
     - SkillForge Multi-Agent Team 的 worker 结果传递格式标准化

  10. **[MED] `taskBudgetRemaining` 跨 compaction boundary 追踪** (`query.ts:282-292`)
      - SkillForge 长会话 budget 追踪可对照

  11. **[MED] `using` ES Stage 3 自动 dispose** (`query.ts:301-304`)
      - SkillForge TypeScript 部分可借鉴(Java 用 try-with-resources)

  12. **[MED] Bun feature() compile-time gate + dead-code elimination**
      - SkillForge OSS vs Enterprise feature 区分:Maven profile + Spring `@ConditionalOnProperty` 等价

  ## 业界印证(SkillForge 选择合理)

  - SkillForge `persistence-shape-invariant` ↔ Claude Code `annotateBoundaryWithPreservedSegment` + `mergeHookInstructions`
  - SkillForge `compact-reviewer` ↔ Claude Code `MAX_CONSECUTIVE_FAILURES=3` 保护
  - SkillForge Skill 两层(system + user)↔ Claude Code 6 源(bundled/skills/plugin/managed/mcp/commands_DEPRECATED)— 但 Claude Code 更细分
  - SkillForge Multi-Agent Team(网状)↔ Claude Code Coordinator Mode 一致 — **SkillForge 跟 Claude Code 同源**,Hermes 反而是单 Subagent 简化

  ## SkillForge 反而更前

  - [+] **Langfuse 风格 trace + replay** — Claude Code 仅日志/analytics,SkillForge UI 更前
  - [+] **Prompt A/B 自动晋升** — Claude Code 无,SkillForge eval pipeline 优势
  - [+] **多 provider 抽象** — Claude Code 单 Anthropic,SkillForge 多 provider(跟 Hermes 一致)
  - [+] **18+ channel adapter** — Claude Code 无 messaging gateway,SkillForge ChannelAdapter SPI 优势(跟 Hermes 一致)

  ## 待 SkillForge 决策(建议立项)

  - **[DECIDE]** SubAgent 是否同时支持 **Fork**(继承)+ **隔离** 两种模式?(详见 [[../concept/multi-agent-isolation]])
  - **[DECIDE]** Compaction 是否从 2 档(Light/Full)拆为 5 级 pipeline?
  - **[DECIDE]** Memory 是否引入 **autoDream 模式**(3-gate + 4-phase)的定期自动 consolidation?
  - **[DECIDE]** **Team memory** 是否引入?secret scanner + guard 是 Team 场景的硬需求
  - **[DECIDE]** Permission 是否细分 5 modes?**`plan` mode** 专用是 SkillForge 没有的概念
  - **[DECIDE]** Hooks 是否扩展到 27 个(对比 Hermes 16 + Claude Code 27)?哪些 IDE-aware 事件(WorktreeCreate / FileChanged)SkillForge 服务端要不要?

  ## 风险/避坑

  - **Fork 继承 cache 的"上下文污染"**:Fork 跟 parent 共享上下文,parent 的 bug/旧 state 会影响 child。SkillForge 加 Fork 模式必须明确何时用 Fork(只读 research)vs 隔离 SubAgent(独立任务)
  - **27 hooks 实际启用复杂度**:`ALWAYS_EMITTED = ['SessionStart', 'Setup']` 只有 2 个默认开,其他靠插件 opt-in — SkillForge 不应所有 hook 默认开
  - **Bun feature flags + dead-code elimination 的不可见性**:internal features 在外部 build 完全消失,**review 时容易漏看**。SkillForge 用 Maven profile 等价时要小心 review process
  - **Autocompact 失败 3 次的代价**(250K API calls/day wasted)— SkillForge 必须有等价 anti-thrashing 保护

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)

  - **"Agent OS"而非"agent runtime"** —— 40+ tools / 30+ services / 27 hooks 的体量,
    工程规模数量级超 Hermes,定位是 production agent platform(Anthropic 商业化产品)
  - **5 级 compaction pipeline**(budget→snip→microcompact→collapse→autocompact),
    而不是同类的"单一 compactor 多触发点"模式
  - **4 层 memory 系统**(autoMemory + SessionMemory + Team memory + extractMemories),
    超越 Hermes 三层,且**有 Team memory + secret scanner**(企业场景必备)
  - **Multi-agent 三模式并存**(SubAgent 隔离 + Fork 继承 + Coordinator 网状),
    给用户选择 —— 而不是 Hermes "只给一种隔离 SubAgent"或其他 agent"只给网状"
  - **autoDream"做梦"系统** —— 字面意义模拟"睡眠时大脑整理记忆",
    `time(24h) + sessions(5) + lock` 3 gate + 4 phase prompt,
    **把 self-evolution 工程化到 production 级别**
  - **27 HOOK_EVENTS zod-validated literal union** —— 运行时 schema 校验 +
    类型安全,大多 agent 用字符串 enum 没校验
  - **Bun compile-time `feature()` + dead-code elimination** —— internal feature(KAIROS/BUDDY/
    PROACTIVE/COORDINATOR_MODE)在 external build 编译期消除,不只是 runtime gate
  - **`USER_TYPE === 'ant'` 双轨**:同一份代码 Anthropic 内部 vs 外部用户看到不同 UI/工具,
    `ConfigTool / TungstenTool / /security-review / staging API` 全 gate

  ## 优势(strengths)

  - **autocompact `MAX_CONSECUTIVE_FAILURES = 3`** + 注释 **BQ 2026-03-10:1,279 sessions ×
    50+ failures = ~250K wasted API calls/day**(`services/compact/autoCompact.ts:71`)
    —— 这条注释本身就是优势:用真实生产数据驱动设计决策,而不是凭直觉拍脑袋
  - **Fork subagent 继承 parent system prompt + cache**(`tools/AgentTool/AgentTool.tsx:485` +
    `prompt.ts:86`) —— Hermes 完全没有,是 production agent 性能优化的关键
    (Anthropic prompt caching 5 分钟 TTL,Fork 共享 cache → 5-10× 便宜)
  - **`<task-notification>` XML 标准化 worker 结果格式**
    (`coordinator/coordinatorMode.ts:145-156`)—— XML 而非 JSON 的原因:可以混在 user message
    text 里被 LLM "看到",但又能被 parser 严格识别。**是 LLM-friendly format 的精彩设计**
  - **`startRelevantMemoryPrefetch` + `using` ES Stage 3 自动 dispose**
    (`query.ts:301-304`)—— TypeScript 现代语言特性应用,生命周期清晰
  - **5 级 compaction 中 `collapse` 折叠 read/search 结果**
    (`utils/collapseReadSearch.ts`)—— 不是简单截断,而是**结构化摘要**,比 Hermes 的
    "[read_file] read X.py from line N (M chars)"一行摘要更丰富
  - **`taskBudgetRemaining` 跨 compaction boundary 追踪**(`query.ts:282-292`)+ 长注释解释
    server-side counting limitation —— 处理"server 看不到完整历史"这个微妙边界,
    需要长时间运行才会发现的坑

  ## 哪些点做得好(what_done_well)

  - **HOOK_EVENTS zod schema literal union**(`entrypoints/sdk/coreSchemas.ts:355+`)
    —— hook event 名是字符串字面量 union,zod 用同一份 union 做 runtime 校验,
    **类型 + 运行时 schema 一份定义**。这种"don't repeat yourself"的模式优于多数 agent
    "枚举 in TS + 单独 JSON schema for validation"
  - **`auto` permission mode 是 ant-only**(`utils/permissions/PermissionMode.ts:95`)
    —— Anthropic 内部用户才能用 ML 自动批准,**对外用户保留显式 approval** ——
    Trust model 跟用户身份绑定的精细设计(对比"all users get same modes"的简单设计)
  - **autoDream 4 phase prompt 的写作**(`consolidationPrompt.ts`)—— Orient/Gather/Consolidate/
    Prune+Index,每 phase 一个明确动词 + 一个明确产物,LLM 可执行性极高。
    **agent self-evolution 的提示词工程典范**
  - **`MAX_CONSECUTIVE_FAILURES = 3` 注释里贴 BQ 数据** —— 工程文档实践 elegant:
    **"为什么这个数字"用真实数据回答**,而不是"经验值"或"看起来合理"
  - **`feature('FORK_SUBAGENT')` Bun gate**(`tools/AgentTool/forkSubagent.ts:33`)——
    新模式上线先 gate,**dead-code elimination 让外部 build 完全无 fork 代码**,
    既支持灰度发布又零运行时开销
  - **`commands_DEPRECATED` 在 LoadedFrom union 里保留** —— `loadSkillsDir.ts:68` 仍然
    支持 legacy 命名,但 union 字面量直接告诉读者"这是 deprecated 路径不要用",
    **不删除 + 显式贬抑**比"悄悄移除导致用户迁移痛苦"更友好
  - **`ALWAYS_EMITTED = ['SessionStart', 'Setup']`**(`hookEvents.ts:18`)
    —— 27 个 hook 中只这两个默认开,其他靠插件 opt-in。**默认最小集 + 显式 opt-in** 是
    plugin 系统的正确默认
  - **`SnipTool` 作为 5 级 compaction 的一环** —— compaction 不只是 background 任务,
    而是把"模型主动选择 snippet 提取"也作为 pipeline 一环,**Agent-driven compaction**
    是 elegant 的 inversion of control
---

# Claude Code v2.1.88

## 一句话定位

Anthropic 官方 Claude Code 逆向源码(v2.1.88,2026-03-31 通过 npm sourcemap 泄漏),**TypeScript + Bun + Ink React 实现的 production-grade agent platform**。远超 CLI 工具 — 40+ tools / 27 hooks / 5 级 compaction / autoDream / KAIROS / Coordinator 网状 / Fork subagent / Team memory + secret guard / Computer Use ("Chicago") / 大量 Bun feature gates。

## 起源与定位

- **背景**:Anthropic 官方,2026-03-31 v2.1.88 通过 npm sourcemap 意外泄漏(Bun 默认生成 sourcemap 未禁用)
- **目标用户**:Anthropic 员工(`USER_TYPE === 'ant'` gate 大量内部 feature)+ 外部开发者(基于订阅:Pro / Max)
- **核心差异化 vs Hermes**:
  - "Developer-facing IDE + CLI agent" vs Hermes "personal multi-channel agent"
  - Production scale + Anthropic 商业化(rate limit / billing header / attestation)
  - 多模式 multi-agent(SubAgent + Fork + Coordinator)
  - 4 层 memory 系统(autoMemory + SessionMemory + Team + extractMemories)
  - 5 级 compaction pipeline(更细分)
  - Bun compile-time + GrowthBook runtime 双层 feature gating
- **不解决**:多 channel messaging(无 Gateway 等价物);多 provider(单 Anthropic)

## 核心架构

```
                ┌───────────────────────────────────────────────┐
                │  入口: CLI / Ink TUI / Bridge / Remote / MCP   │
                └─────────────────────┬─────────────────────────┘
                                      ▼
                ┌───────────────────────────────────────────────┐
                │   query.ts:219 (async generator)              │
                │   queryLoop.ts:241 while(true)                │
                │   9 字段 mutable State                         │
                │   `using` memory prefetch (ES Stage 3)        │
                │   feature('TOKEN_BUDGET') Bun gate            │
                └──────────┬─────────────┬──────────────────────┘
                           ▼             ▼
        ┌────────────────────────┐  ┌──────────────────────────┐
        │ Tool.ts (792 LOC)      │  │ services/compact/         │
        │ 40+ tools 子目录        │  │ 5-level pipeline          │
        │ PermissionMode/        │  │ budget→snip→microcompact  │
        │ prePlanMode/yolo       │  │ →collapse→autocompact      │
        │ tengu_tool_pear gate   │  │ MAX_CONSEC_FAILURES=3      │
        └────────────────────────┘  └──────────────────────────┘
                           │             │
              ┌────────────┼─────────────┼──────────────────┐
              ▼            ▼             ▼                  ▼
       ┌──────────┐  ┌──────────┐  ┌─────────────┐  ┌─────────────┐
       │ Memory 4 │  │ Multi-Ag │  │ Hooks 27    │  │ Permission  │
       │ 系统     │  │ 3 modes  │  │ zod-validated│  │ 5 modes +   │
       │ memdir/  │  │ SubAg/   │  │ + 5 types   │  │ YOLO clsfr  │
       │ Session  │  │ Fork/    │  │ Always:     │  │ + iron_gate │
       │ Team+secG│  │ Coord    │  │ Session+Setup│  │ + Explainer │
       │ extractM │  │ +Task    │  │             │  │             │
       └──────────┘  └──────────┘  └─────────────┘  └─────────────┘

       ┌─────────────────────────────────────────────────────────┐
       │ autoDream (3-gate + 4-phase) — Memory consolidation     │
       │ services/autoDream/ — independent fork subagent         │
       └─────────────────────────────────────────────────────────┘

       ┌─────────────────────────────────────────────────────────┐
       │ Bun compile-time feature() + GrowthBook runtime         │
       │ Internal (KAIROS/BUDDY/PROACTIVE) vs External builds    │
       │ USER_TYPE === 'ant' gates                                │
       └─────────────────────────────────────────────────────────┘
```

## 13 维度详解

详见 frontmatter,每条带 `[code: file:line]` 锚点,可跳源码 verify。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`,分 4 类:
- **直接可借鉴 12 条**(高 + 中优先级)
- **业界印证 4 条**(SkillForge 选择合理)
- **SkillForge 反而更前 4 条**
- **待 SkillForge 决策 6 条**(建议立项)
- **风险/避坑 4 条**

## 已知近期演进 (post v2.1.88, unverified)

> 2026-05-16 update:本节列 v2.1.88 ingest 后 Anthropic 已发布的新功能,**当前无源码访问,所有事实标 `[community-knowledge]`**,verify 优先级见每条。下一次新版可访问时,这些事实应升级为 `[verified-from-source]` 并填具体 file:line 锚点。

### goal(目标系统) `[community-knowledge]`
- **HIGH verify 优先级**:对照 [[harness/codex]] `core/src/goals.rs` Goals 协议级 + `token_budget` per goal + 5 OTEL metric — Anthropic 跟进 OpenAI Goals 设计是高概率事件
- 推断:`ThreadGoal` / `goal-create` / `goal-update` 等 SDK API,可能跟 `query()` 主循环集成
- 待 verify:goal 是否带 budget、是否产生 OTEL metric、跟 27 hooks 中哪些事件交互

### agent-view(agent 可视化) `[community-knowledge]`
- **MEDIUM verify 优先级**:UI 可视化通常不影响 agent 设计哲学
- 推断:实时显示 agent 树(SubAgent + Fork 树形结构)+ 状态(running/cancelled/completed)+ tool 调用流
- 待 verify:跟 `coordinator/coordinatorMode.ts` 网状协作如何呈现、Fork SubAgent 跟 SubAgent 在 UI 上是否区分

### agent-manager(agent 管理增强) `[community-knowledge]`
- **MEDIUM verify 优先级**:对照 [[harness/codex]] `multi_agents/{spawn,resume_agent}.rs` actor 模型 + `DEFAULT_AGENT_MAX_THREADS=6`
- 推断:agent lifecycle 管理增强,可能含 pause/resume/kill/list/inspect/migrate 等
- 待 verify:是否引入 actor 模型(对照 Codex)、是否仍 fire-and-forget(对照 v2.1.88)、跟 Fork SubAgent 关系

### `/insights` slash command(meta self-evolution: session 分析 + 建议改 CLAUDE.md)`[verified-from-source 2026-05-17]`

**v2.1.88 已 ship,源码 `src/commands/insights.ts` 3200 行单文件**。详细深挖见 [[../deep/claude-code-insights]] + [[../source/claude-code-insights]]。

**触发方式**:用户主动 slash command `/insights`(不是 hook 自动触发,不是 `Stop` / `SessionEnd`)。设计哲学:**用户驱动 self-improvement,LLM 给建议但不自动改 CLAUDE.md**。

**实现概要**(verified):
- **6 维 55 typed enum**(`:352-415` `LABEL_MAP`):goal_categories 13 / friction 12 / satisfaction 8 / outcome 5 / helpfulness 5 / session_type 5 + success 子维度 — **业内 Layer D typed enum schema 最完整**(对照 tau-bench 7 / AgentOps 15)
- **Opus model 跑分析**(`:41-48` 2 个 getter 都用 `getDefaultOpusModel()`,不省成本)
- **3 层数据模型**(`:228-326`):SessionMeta(deterministic stats)+ SessionFacets(LLM-extracted qualitative)+ AggregatedData(30+ 字段)
- **8 INSIGHT_SECTIONS 并行 LLM call**(`:1336-1495`):project_areas / interaction_style / what_works / friction_analysis / **suggestions ⭐** / on_the_horizon / fun_ending(+ Ant-only cc_team / model_behavior_improvements)
- ⭐ **`MULTIPLE TIMES` algorithm**(`:1430` 核心 prompt 注释):_"PRIORITIZE instructions that appear MULTIPLE TIMES in the user data. If user told Claude the same thing in 2+ sessions, that's a PRIME candidate"_ — Layer E 自动归因核心 wisdom
- **Facets per-session 永久 cache**(`~/.claude/usage-data/facets/`)+ INSIGHT_SECTIONS 每次重跑 — 首次 ~$5/10min,后续 ~$1/30s
- **`suggestions` section 输出 3 类建议**:`claude_md_additions`(给你 CLAUDE.md 加什么)+ `features_to_try`(从 5 个 CC FEATURE REFERENCE 选)+ `usage_patterns`(工作模式)
- **`detectMultiClauding` 滑窗算法**(`:1062-1143`,30 min OVERLAP_WINDOW)— 业内独家检测多 Claude 并发会话
- **InsightsExport S3 upload**(`:2655-2737`)— Ant-only 后台 upload 给 Anthropic 产品团队分析
- **Remote host data collection**(`:60-220`)— Ant-only SCP 拉 Coder workspace `~/.claude/projects/`
- 输出 interactive HTML report,自动打开浏览器

**对照 [[../concept/attribution]] §Layer E**:Claude Code `/insights` 在 ① 标注 + ② 聚类 + ③ 归因决策 都做到 LLM-driven 自动化,但 ④ candidate 生成只 _建议_ 不 _自动改 CLAUDE.md_,⑤⑥ A/B + canary + verify **完全没**(用户决策替代)。

跟 SkillForge V3 + V80 _"自动 canary"_ 路线**两条互补 Layer E 路径**:
- Claude Code `/insights` = **用户驱动 review trace pattern + 决策 100% 给用户**(适合 CLI 工具改 user config)
- SkillForge V3 + V80 = **自动 candidate + canary + verify**(适合 platform 改系统内部 surface)

跟 Anthropic 自家 [[../deep/claude-code-autodream-3-gate]] **互补 2 条 self-improvement 路径**:
- autoDream = **自动后台 memory consolidation**(用户无感)
- `/insights` = **用户主动 review session pattern + 改 CLAUDE.md**(用户决策)

**社区生态**(围绕 `/insights` 闭环 ④⑤⑥):
- `yahav10/claude-insights` CLI — 把 HTML report 转 actionable artifact(skill/rule/hook configs)
- `rebelytics/one-skill-to-rule-them-all` — meta-skill 持续观察 sessions + 建议改进
- `aviadr1/claude-meta` — 把 CLAUDE.md 当 learning system 持续累积
- MindStudio `learnings.md` pattern — 项目 root 维护 `learnings.md`,session start 读 + session end 更新

## 跟 [[harness/hermes-agent]] 的对比表

详见 [[../source/claude-code]] "跟 Hermes 的关键对比"表格 13 行。

## 待回答的问题(lint 会扫这一节)

- 笔记 04 说"Skills 6 来源(bundled/user/project/plugin/managed/mcp)" → 源码实际是 `commands_DEPRECATED/skills/plugin/managed/bundled/mcp`,**user/project 实际合并到 `skills`**,通过 `SettingSource`(`userSettings`/`projectSettings`)区分(`loadSkillsDir.ts:67-90`)。**笔记表述需修**
- 笔记 01 说"5 个 query() 调用点 + queryLoop 13 步" → 仅 verified queryLoop 入口,**未深 grep verify 完整**
- KAIROS proactive mode 详细实现(`assistant/`)未读
- BUDDY tamagotchi(`buddy/`)未读 — 非核心 agent 能力,可跳
- ULTRAPLAN 30-minute remote planning 实际通信协议未读
- Bridge to claude.ai JWT 流程(`bridge/`)未读
- Computer Use "Chicago" 实现细节(`@ant/computer-use-mcp`)未读
- `tools/permissions/` README 提到的实际是 `hooks/toolPermission/`(确认了)+ `utils/permissions/`,WRITE_DENIED_PATHS 等具体常量未源码 verify
- 跟 [[harness/openclaw]] 的继承关系 — **已 verified**(2026-05-16 OpenClaw ingest):OpenClaw 是 Memory Dreaming 源头(`MANAGED_DREAMING_CRON_NAME = "Memory Dreaming Promotion"`),Claude Code v2.1.88 借鉴命名但简化实现
