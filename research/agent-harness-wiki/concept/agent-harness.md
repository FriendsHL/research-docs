---
name: agent-harness
type: concept
status: B-mode source-grounded
updated: 2026-05-16
sources:
  - "[[harness/hermes-agent]]"
  - "[[harness/claude-code]]"
  - "[[harness/openclaw]]"
  - "[[harness/codex]]"
  - "[[harness/opencode]]"
  - "[[../skillforge/skillforge-core]] (read-only)"
  - "[[../skillforge/skillforge-server]] (read-only)"
---

# Agent Harness — Cross-Harness Design Space & R-HARNESS Clipboard

> 5 harness 横向 + B 模式 verify SkillForge `skillforge-core/` + `skillforge-server/` 源码,得 R-HARNESS-1~R-HARNESS-10 actionable 借鉴清单。  
> ⚠️ R-HARNESS 列表与 [[concept/agent-memory]] R-MEM 系列、[[concept/observability]] R-OBS 系列同体系,优先级独立打分。

---

## 1. 5 Harness 一句话定位

| 项目 | 哲学 | 独家 / 强项 | 短板 |
|---|---|---|---|
| **Hermes** (Nous Research) | 自进化 Python Agent + `commit_memory_session` simple trigger | MAX_DEPTH=1 / 16 hooks / 7 sandbox backends | 单一 LLM provider / 无 multi-agent isolation |
| **Claude Code** (Anthropic) | 官方 v2.1.88 TS+Bun+Ink | 27 hooks zod / 5 级 compaction / autoDream 3-gate / SubAgent+Fork+Coordinator 三模式 / 5 permission + YOLO ML | 闭源 |
| **OpenClaw** (Hermes 前身) | TS pnpm monorepo + pi-sdk 嵌入式 | 双层 agent loop 显式拆分 / Memory Dreaming 6 维评分 / Memory Wiki(Karpathy 模式 production)/ QA Character Eval 最完整 framework | 单一 provider |
| **Codex** (OpenAI 官方) | Rust monorepo · 100+ crates | Code Mode V8(LLM 写 JS)/ Goals 协议级一等 + per-goal token_budget / Multi-agent actor + mailbox / External agent migration(4 家独家)/ 5 OTEL metric | 极重(Rust)/ Code Mode 是高风险路径 |
| **opencode** (Anomaly Corp / MIT) | TS Bun + Effect-TS functional + Drizzle/SQLite | LSP first-class(`lsp/` 6 模块 + `tool/lsp.ts`,4 家独家)/ Client/Server 架构 / `PRUNE_PROTECTED_TOOLS=["skill"]` / `PlanExitTool` 跨 agent 切换 | Effect-TS 学习曲线陡 / 不适合 Java 移植 |

5 路线 + SkillForge 自家 Java/Spring 第 6 路线,详见 [[concept/agent-runtime-stacks]]。

---

## 2. 10 维设计空间(跨 5 harness)

| 维度 | Hermes | Claude Code | OpenClaw | Codex | opencode | SkillForge 现状(verified) |
|---|---|---|---|---|---|---|
| **1. 工具协议** | JSON tool_use | JSON tool_use | JSON tool_use | JSON tool_use + Code Mode V8 isolate | JSON tool_use | JSON tool_use(`AgentLoopEngine.java`) |
| **2. Multi-Agent 模型** | depth=1 单层 | SubAgent + Fork + Coordinator | depth=1 单层 | Actor + mailbox + control + `resume_agent` | 2 builtin agent + `@general` subagent + `PlanExitTool` | SubAgent + crash recovery(`SubAgentRunEntity` + `SubAgentStartupRecovery.java`) |
| **3. Context Compaction** | 切新 session | in-place + 5 级 pipeline + reactive 413 recovery | 双层 loop 显式拆分 | 3 套 + InitialContextInjection | 4 规则 + `PRUNE_PROTECTED_TOOLS=["skill"]` | Light/Full + Compactable whitelist(`CompactableToolRegistry.java`)+ 4-axis exit(budget/duration/max_tokens/max_loops) |
| **4. Hooks 数量** | 16 | 27 zod-validated | (沿用 Hermes) | 8 | ~27 (估) | **5**(SESSION_START / USER_PROMPT_SUBMIT / POST_TOOL_USE / STOP / SESSION_END,`HookEvent.java:13-18`) |
| **5. Memory 路线** | 3 层 | 4 层 + autoDream + Team + secret guard | Memory Dreaming + Memory Wiki(Karpathy 模式 production) | 无独立 memory framework | 不显式 memory framework | LLM-SYNTHESIS proposal + 人审 gate + DREAM-CONSOLIDATION dedup(详见 [[concept/agent-memory]]) |
| **6. LSP 集成** | 无 | 无 | 无 | 无 | **6 模块 + `tool/lsp.ts` 4 家独家** | **无**(`grep lsp\|languageserver\|pylsp\|tsserver\|jdt.ls\|textdocument`命中 0 业务代码) |
| **7. Goals + Token Budget** | 隐式 | 隐式 | 隐式 | Goals 协议级 + per-goal token_budget + 5 OTEL metric | 隐式 | session-level budget exit(`AgentLoopEngine.java:1363-1384`)+ scenario `businessGoal` field(`EvalScenarioEntity.java:76-244`,V84+) |
| **8. External Agent Migration** | 无 | 无 | 无 | `external-agent-sessions/` 4 家独家(SHA256 ledger + 50 条上限 + 30 天 + NOTE_MAX=2000 + TOOL_RESULT_MAX=4000) | 无 | **仅 CLAUDE.md import**(`SystemPromptBuilder.java:34` + `ContextBreakdownService.java:142` claude_md segment),无 AGENTS.md / cursorrules / .cursor/ |
| **9. Skill 装载源** | 无 | plugin / built-in / `.claude/` ≈ 3-4 | builtin + ext + memory-wiki | builtin + plugin | builtin + plugin | **8 SkillSource enum**(UPLOAD / SKILL_CREATOR / DRAFT_APPROVE / EVOLUTION_FORK / SKILLHUB / CLAWHUB / GITHUB / FILESYSTEM,`SkillSource.java:20-28`)— **业内最多** |
| **10. Permission Model** | binary | 5 modes + YOLO ML | builtin allow/deny | sandbox profile 3 平台 | per-tool ask | session per-tool confirm + agent allow/deny |

---

## 3. R-HARNESS Clipboard (B-mode verified 2026-05-16)

> Verify 方法:`grep -rEni "keyword" skillforge-core skillforge-server --include="*.java"`(`grep -v test`)+ 精读关键 enum / entity / engine 文件。

### 🔴 P1 — 真缺口,值得做

#### R-HARNESS-1 · LSP first-class(借鉴 opencode)

**借鉴源**:opencode `packages/opencode/src/lsp/` 6 模块 + `tool/lsp.ts`,4 家独家。Effect-TS Schema 类型化 LSP 协议(Position / Range / Symbol / DocumentSymbol)。Agent 直接看 diagnostic / hover / definition / references。

**SkillForge 现状** [verified-from-source]:
- `grep -rEni "lsp\|languageserver\|pylsp\|tsserver\|jdt\.ls\|textdocument" skillforge-core skillforge-server --include="*.java"` → **业务代码 0 命中**(仅 `toolsPrompt` / `writeToolSpan` 名字 hit,与 LSP 无关)
- Tool 层目前只通过 prompt 串描述 + JSON 输出消化,无源码语义层 fact

**真缺口**:Java/Kotlin/Python/TS LSP 接入 SkillForge tool 层,让 Edit/Read/Write tool 拿到 diagnostic + symbol + reference 后再决策。

**优先级**:🔴 高(SkillForge coding agent 真正能力短板)— 详见 [[concept/lsp-integration]] 7 维设计空间。

---

#### R-HARNESS-6 · PreToolUse / PreCompact hook event(借鉴 Claude Code)

**借鉴源**:Claude Code 27 hook events,opencode ~27。关键 gate event:**PreToolUse**(危险工具调用前阻断)+ **PreCompact**(压缩前快照 / 用户介入)+ **Notification**(异步通知 channel)。

**SkillForge 现状** [verified-from-source]:
- `HookEvent.java:13-18` 仅 **5 events**:`SESSION_START / USER_PROMPT_SUBMIT / POST_TOOL_USE / STOP / SESSION_END`
- **缺 PreToolUse / PreCompact / Notification**
- 体系完整(`LifecycleHooksConfig.java` 解析 + `AgentAuthoredHookService` agent 自带 hook + `EffectiveHook.java` user/system/agent 三层 + `ScriptHandlerRunner.java` 环境变量 `SF_HOOK_EVENT` 传递)

**真缺口**:加 **PreToolUse**(最高价值,gate dangerous tool before exec,跟 SkillForge V?? permission 系统配套)+ **PreCompact**(让用户在自动 compaction 前 inspect / cancel)。

**优先级**:🔴 高(PreToolUse 是 safety / permission 系统的核心 chokepoint)。

---

### 🟡 P2 — 配套增强,部分已实现

#### R-HARNESS-2 · Codex 风格 per-goal token_budget(校准)

**借鉴源**:Codex `core/src/goals.rs` Goals 协议级一等公民 + per-goal token_budget + 5 OTEL metric。

**SkillForge 现状** [verified-from-source]:
- **session-level token budget 已实现**:`AgentLoopEngine.java:1363-1384` 4-axis exit reason `token_budget_exceeded / duration_exceeded / max_tokens_exhausted / max_loops_reached`,`ChatService.java:810` `isSilentExit = "token_budget_exceeded"` 处理
- **business-level goal 已实现**:`EvalScenarioEntity.java:76-244` `businessGoal` field(V84+ eval scenario 业务目标)+ `SessionScenarioExtractorService.java:131` LLM-extracted from session + `SimulatorTrialOrchestrator.java:580` 注入 user simulator

**真缺口**:**协议级 per-goal token_budget**(把 Codex Goals 协议作为 SkillForge agent loop 的一等抽象,而非 eval scenario 的属性)。当前 SkillForge `businessGoal` 是 eval-time 属性,不是 runtime agent loop 内的 first-class goal stack。

**优先级**:🟡 中(已有 budget 退出 + 已有 businessGoal 属性,只缺协议级抽象统一)— **校准:R-HARNESS-2 工作量从 1-2 周降到 < 半周**。

---

#### R-HARNESS-3 · 校准为"已实现"(从原 P1 → 已有)

**借鉴源**:opencode `PRUNE_PROTECTED_TOOLS = ["skill"]`(`session/compaction.ts`)。

**SkillForge 现状** [verified-from-source]:
- `CompactableToolRegistry.java:9-15` 显式文档:_"Tools NOT in the whitelist (e.g. **Memory, SubAgent**) are left untouched to preserve semantically dense content."_
- DEFAULT_COMPACTABLE_TOOLS(`:27-31`):Bash / Read / Write / Edit / Grep / Glob / WebFetch / WebSearch / Browser / CodeSandbox / CodeReview
- **Per-agent override**(`:64-81` `fromAgentConfig` 读 `compactable_tools` key,opencode 没有 per-agent 维度)
- inverse pattern:opencode = blocklist("这些 protect"),SkillForge = whitelist("这些 compact"),**功能等价 + per-agent override 更强**

**真缺口**:无。**SkillForge 已领先**(详见 §4)。

**优先级**:✅ 已实现,无需借鉴 — 原 R-HARNESS-3 推断错误,校准为 SkillForge **已领先**。

---

#### R-HARNESS-4 · Multi-Agent 模式扩展(校准)

**借鉴源**:Claude Code 三模式(SubAgent / Fork / Coordinator)+ Codex actor + mailbox + `resume_agent`。

**SkillForge 现状** [verified-from-source]:
- **完整 SubAgent 系统已存在**(原推断严重低估):
  - `skillforge-server/.../subagent/`:`AgentRoster.java` / `CollabRunService.java` / `SubAgentRegistry.java` / `SubAgentRunSweeper.java`
  - `SubAgentRunEntity.java`:`runId / parentSessionId / childSessionId / childAgentId / childAgentName / task / status(RUNNING/COMPLETED/FAILED/CANCELLED) / finalMessage / spawnedAt / completedAt`
  - **Crash recovery**(业内罕见):`SubAgentStartupRecovery.java:35-95` 服务重启时遍历 RUNNING 行,孤儿 run 标 cancelled 并通知 parent
  - **Async result delivery**:`SubAgentPendingResultEntity` + `SubAgentPendingResultRepository.findByParentSessionIdAndStatusIsNullOrderByIdAsc`
  - **Cross-thread context**:`LlmCallContext.java:13` 不用 ThreadLocal,explicit pass — _"跨线程传递明确,便于在 SubAgent / 异步上下文中保持一致"_
  - **N+1 fallback resolver**:`SessionRepository.java:48-61` OBS-1 §7.4 R3-WN2 — SubAgent TOOL_CALL spans 失配时 collapse 批量回退
  - **Top-level filter**:`SessionRepository.java:23` "只取顶层 session(过滤掉 SubAgent 派发出来的子 session)"

**真缺口**:**Fork 模式**(继承 parent context + cache 而非完全隔离,Claude Code 独家)和 **Coordinator 模式**(多 SubAgent 协作总线,Claude Code 独家)。Codex `resume_agent`(暂停一个 agent 切到另一个再回来)也是 SkillForge 没有的。

**优先级**:🟡 中 — 当前 SubAgent 是"隔离 spawn"模式(类 Claude Code SubAgent + Codex actor),已能 cover 80% 场景。Fork / Coordinator 是 nice-to-have。**校准:R-HARNESS-4 工作量大幅下调,只需补 Fork 模式即可**(< 1 周)。

---

#### R-HARNESS-5 · External agent migration(借鉴 Codex)

**借鉴源**:Codex `external-agent-sessions/` 4 家独家。从 Claude Code / Cursor / opencode / Aider 4 家 harness 导入 session。SHA256 ledger 防重复 + 50 条上限 + 30 天窗口 + `NOTE_MAX_LEN=2000` + `TOOL_RESULT_MAX_LEN=4000`。

**SkillForge 现状** [verified-from-source]:
- **仅 CLAUDE.md import**:`SystemPromptBuilder.java:34` 拼接顺序 _"CLAUDE.md → AGENT.md → SOUL.md → TOOLS.md/默认Guidelines → Context"_(:86 "1. CLAUDE.md — 全局规则",:462 "构建 system prompt(注入全局 CLAUDE.md)")
- `ContextBreakdownService.java:142` `Segment.leaf("claude_md", "CLAUDE.md", ...)` — context breakdown 一等公民
- `grep "agents\.md\|AGENTS\.md\|cursorrules\|\.cursor/"` 命中 0 业务代码 — 无 AGENTS.md / cursorrules / .cursor 支持

**真缺口**:
1. **AGENTS.md 支持**(已是事实标准,opencode/promptfoo 等都用)
2. **`.cursor/rules` 支持**
3. **从他家 harness 迁 session**(Codex 风格,4 家独家,SkillForge 抓 Cursor / Claude Code 用户的 wedge)

**优先级**:🟡 中 — AGENTS.md 是低成本高价值(改 `SystemPromptBuilder` 加一段)。Session migration 是 product wedge,优先级取决于用户迁移路径策略。

详见 [[concept/agent-interop]]。

---

#### R-HARNESS-7 · 校准为"SkillForge 已领先"

**借鉴源**:opencode 6 LoadedFrom / Claude Code 3-4 plugin source。

**SkillForge 现状** [verified-from-source]:
- `SkillSource.java:20-28` **8 enum values**:`UPLOAD / SKILL_CREATOR / DRAFT_APPROVE / EVOLUTION_FORK / SKILLHUB / CLAWHUB / GITHUB / FILESYSTEM`
- 完整 storage layout 映射(`SkillSource.java` 顶部 JavaDoc):
  - `UPLOAD → upload/{ownerId}/{uuid}`
  - `SKILL_CREATOR → skill-creator/{ownerId}/{uuid}`
  - `DRAFT_APPROVE → draft-approve/{ownerId}/{uuid}`(人审 gate 出口)
  - `EVOLUTION_FORK → evolution-fork/{ownerId}/{parentSkillId}/{uuid}`(EvolvingAgent 自进化 fork)
  - `SKILLHUB / CLAWHUB → marketplace`
  - `GITHUB → github/{repoSlug}/{ref}`
  - `FILESYSTEM → filesystem/{ownerId}/{uuid}`
- 完整治理流程(`SkillImportService.java:107` `allowMediumRisk` + `buildAllocationContext` 路径分配 + `SkillBatchImporter.java:70-111` 批量导入)
- **`DRAFT_APPROVE` + `EVOLUTION_FORK` 治理流程业内独家**(其他 harness 只有 marketplace + local + plugin,无 draft approval 工作流 + evolution lineage)

**真缺口**:无。**SkillForge 已领先**(详见 §4)。

**优先级**:✅ 已实现,无需借鉴 — 原 R-HARNESS-7 校准为 SkillForge **已领先**。

---

### 🔵 P3 — 不该走的路 / 重复造轮子

#### R-HARNESS-8 · Code Mode V8 isolate(❌ 不该跟)

**借鉴源**:Codex `code-mode/` `rusty_v8` + `deno_core_icudata`,LLM 写 JS 在 V8 isolate 调工具,而非 JSON tool_use。

**为什么不该跟**:
1. **JVM ≠ V8**:SkillForge Java/Spring 栈,引入 V8 isolate 需要 GraalJS 或 Polyglot,运维成本极高
2. **Risk profile**:Code Mode 把 LLM 提升到 "写代码 → 执行" 级别,安全边界从 JSON schema gate 升级到 sandbox,需要的 mitigation 是 2-3x
3. **业务价值未证**:Codex 自家也是 V8(`0.0.0-dev`)2026-05-15 dev branch,尚未稳定产品化
4. **SkillForge 已有更好路径**:Skill marketplace + DRAFT_APPROVE 人审 gate(governance > raw V8 capability)

**优先级**:🔵 P3 — **不做**(详见 [[concept/code-mode-orchestration]])。

---

#### R-HARNESS-9 · Memory Wiki Karpathy 模式(❌ 重复)

**借鉴源**:OpenClaw `extensions/memory-wiki/` Karpathy llm-wiki 模式 production 实现。

**为什么不该跟**:
1. **SkillForge 已有 LLM-SYNTHESIS proposal + 人审 gate 路线**([[concept/agent-memory]] R-MEM 已 verify)
2. **Karpathy wiki 模式是 collaborative editing**,SkillForge 路线是 **LLM-proposed + Human-approved → optimized memory**(更适合 SaaS scope)
3. **同源不同形**:wiki 是 verbatim 累积,SkillForge 是 LLM-synthesis dedup
4. 我们手做的 [[agent-harness-wiki]] 已经吸收 Karpathy 模式的"分目录 + frontmatter + wikilink + lint"骨架,**实现已落地**

**优先级**:🔵 P3 — **不做**(详见 [[concept/llm-wiki-pattern]])。

---

#### R-HARNESS-10 · 27 个细粒度 hooks(❌ 过度)

**借鉴源**:Claude Code 27 hooks(`HOOK_EVENTS` zod-validated)+ opencode ~27。

**为什么不该跟**(细粒度部分):
1. SkillForge 当前 5 events 已覆盖 80% 场景
2. 27 events 是 Claude Code 多 channel(IDE / TUI / Desktop / Web)的产物,SkillForge 通过 channel 抽象层处理,不需要 hook event 爆炸
3. **PreToolUse / PreCompact / Notification 是核心增量**(已列入 R-HARNESS-6,优先级 P1)— 其余 22 events 大部分是 UI lifecycle,SkillForge 不需要

**注意**:R-HARNESS-6 ≠ R-HARNESS-10。R-HARNESS-6 = 加 3 个关键 event(P1),R-HARNESS-10 = 不照搬 27 个全套。

**优先级**:🔵 P3 — **不做**(只做 R-HARNESS-6 关键 3 个 event)。

---

## 4. SkillForge 已领先(verify 后扩展)

| # | 项目 | SkillForge 实现 [verified-from-source] | 业内对照 |
|---|---|---|---|
| 1 | **Skill 装载源 8 enum** | `SkillSource.java:20-28` UPLOAD / SKILL_CREATOR / DRAFT_APPROVE / EVOLUTION_FORK / SKILLHUB / CLAWHUB / GITHUB / FILESYSTEM | Claude Code 3-4 / opencode ~6 |
| 2 | **Skill 治理流程**(DRAFT_APPROVE + EVOLUTION_FORK) | `SkillDraftService.java:683/705` allocate + setSource / `SkillEvolutionService.java:369` evolution fork | 业内独家 — 其他 harness 无 draft approval + evolution lineage |
| 3 | **SubAgent crash recovery** | `SubAgentStartupRecovery.java:35-95` 服务重启遍历 RUNNING 行 + 孤儿标 cancelled + 通知 parent | Claude Code 无(in-memory)/ Codex actor 部分支持 |
| 4 | **Async SubAgent result delivery** | `SubAgentPendingResultEntity` + `findByParentSessionIdAndStatusIsNullOrderByIdAsc` | 业内罕见 |
| 5 | **Compactable whitelist per-agent override** | `CompactableToolRegistry.java:64-81` `fromAgentConfig` 读 agent `compactable_tools` | opencode `PRUNE_PROTECTED_TOOLS` 全局常量(无 per-agent) |
| 6 | **Cross-thread agent loop context** | `LlmCallContext.java:13` explicit pass 不用 ThreadLocal — _"便于在 SubAgent / 异步上下文中保持一致"_ | Claude Code TS 单线程 / Codex actor 内嵌 mailbox |
| 7 | **4-axis loop exit reason** | `AgentLoopEngine.java:1363-1384` `token_budget_exceeded / duration_exceeded / max_tokens_exhausted / max_loops_reached` | 业内普遍只有 token + max_loops 2 轴 |
| 8 | **Business-level goal in eval scenario** | `EvalScenarioEntity.java:76-244` `businessGoal` field + `SessionScenarioExtractorService.java:131` LLM-extracted + `SimulatorTrialOrchestrator.java:580` 注入 simulator | 业内 eval 普遍只测 task accuracy,无业务目标抽象 |
| 9 | **CLAUDE.md 一等公民 segment** | `SystemPromptBuilder.java:34/86/462` 拼接顺序 + `ContextBreakdownService.java:142` claude_md segment in context breakdown | Claude Code 自家 / opencode 仿照 / 其他无 |
| 10 | **Agent-authored hooks** | `AgentAuthoredHookService.java` + `EffectiveHook.java:30` `agent(HookEvent, HookEntry, ...)` user/system/agent 三层 | Claude Code 只有 user-defined / agent built-in 二层 |

**结论**:SkillForge 已领先 **10 条**(从 R-MEM verify 后的 6 条扩到 R-HARNESS verify 后的 10 条,系统性强于 5 harness 多数维度)。

---

## 5. 优先级总览

| 优先级 | 数量 | 项目 | 工作量 |
|---|---|---|---|
| 🔴 P1 必做 | 2 | R-HARNESS-1 LSP / R-HARNESS-6 PreToolUse+PreCompact hooks | 各 1-3 周 |
| 🟡 P2 配套 | 3 | R-HARNESS-2 Goals 协议级(< 半周)/ R-HARNESS-4 Fork 模式(< 1 周)/ R-HARNESS-5 AGENTS.md+session migration(1-2 周) | 共 2-4 周 |
| ✅ 已实现 | 2 | R-HARNESS-3 pruning whitelist(已 ≥ opencode)/ R-HARNESS-7 SkillSource 8 enum(业内最多) | — |
| 🔵 P3 不做 | 3 | R-HARNESS-8 Code Mode V8 / R-HARNESS-9 Memory Wiki / R-HARNESS-10 27 hooks 全套 | — |

---

## 6. 不该走的路 5 条(沿用 R-MEM + 扩展)

1. ❌ **Code Mode V8 isolate**(R-HARNESS-8)— JVM ≠ V8,governance > raw capability
2. ❌ **Effect-TS Java 移植**(opencode 路线)— Effect-TS 是 TS-only,Java 套用类似抽象成本高于收益
3. ❌ **100+ crate 细拆分**(Codex 路线)— SkillForge 多模块 Maven 已是合理粒度,不需要 100+ Maven module
4. ❌ **27 hooks 全套**(R-HARNESS-10)— 只做 PreToolUse / PreCompact / Notification 3 个增量(R-HARNESS-6)
5. ❌ **per-provider crate**(Codex 路线)— SkillForge `OpenAiProvider` + `ClaudeProvider` 2 个 class 已 cover,不需要 per-provider 独立模块

---

## 7. 后续动作

- [ ] **R-HARNESS-1 LSP**:开 design 文档 + Java/Kotlin/Python LSP client wrapper POC(优先支持 Java Eclipse JDT.LS,因 SkillForge 自家是 Java)
- [ ] **R-HARNESS-6 PreToolUse / PreCompact**:加 3 个 `HookEvent` 值 + dispatch 逻辑(`LifecycleHookDispatcher.dispatch` 已支持 generic event,只需注入新 enum)
- [ ] **R-HARNESS-2 Goals 协议级**:把 `EvalScenarioEntity.businessGoal` 升级为 `AgentLoopContext.goalStack`,runtime per-goal token budget
- [ ] **R-HARNESS-5 AGENTS.md**:`SystemPromptBuilder.java:34` 拼接顺序加一段 `AGENTS.md → CLAUDE.md` parsing
- [ ] 与 [[concept/agent-memory]] R-MEM list + [[concept/observability]] R-OBS list 合并,出 SkillForge 全栈 borrow roadmap

---

## 引用链路

- 5 harness 来源:[[harness/hermes-agent]] / [[harness/claude-code]] / [[harness/openclaw]] / [[harness/codex]] / [[harness/opencode]]
- 同体系 borrow lists:[[concept/agent-memory]] R-MEM / [[concept/observability]] R-OBS / [[concept/trace-based-eval]] R-OBS-Trace
- 维度专题:[[concept/lsp-integration]] / [[concept/hooks-design]] / [[concept/code-mode-orchestration]] / [[concept/multi-agent-isolation]] / [[concept/agent-interop]] / [[concept/llm-wiki-pattern]]
