---
type: harness
name: opencode
status: active
language: typescript

# === 版本时效 ===
last_ingested_version: "1.15.0 @ commit 2d6bedecd (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/opencode]]"
  - "/Users/youren/myspace/opencode/README.md"
  - "/Users/youren/myspace/opencode/AGENTS.md"
  - "/Users/youren/myspace/opencode/packages/opencode/src/agent/agent.ts"
  - "/Users/youren/myspace/opencode/packages/opencode/src/session/compaction.ts"
  - "/Users/youren/myspace/opencode/packages/opencode/src/lsp/"
  - "/Users/youren/myspace/opencode/packages/opencode/src/tool/plan.ts"
  - "/Users/youren/myspace/opencode/.opencode/"

# === 13 个能力维度 ===

agent_loop: |
  Effect-TS 重度 functional 风格(`packages/opencode/src/agent/agent.ts:1-26`):
  - `Effect.gen` + `Service` + `Layer` + `Context` 是 first-class
  - `Agent.Info` Effect-TS Schema(`agent.ts:29-46`)定义 agent 实体:`name` / `description` / `mode: ["subagent", "primary", "all"]` / `permission: Permission.Ruleset` / `model: {modelID, providerID}` / `topP` / `temperature` / `prompt` / `steps`
  - **6 个 prompt template variant**(`prompt/{compaction,explore,scout,summary,title}.txt` + `generate.txt`)— agent 可以是不同 variant
  - **`Service.generate()` 用 `ai` SDK `streamObject` 自动生成 agent**(从 description → `{identifier, whenToUse, systemPrompt}`)
  - **OpenTelemetry 集成**(`@effect/opentelemetry/Tracer`)— observability 一等公民
  [verified-from-source]

tool_protocol: |
  - `tool/registry.ts` 动态注册
  - `tool/json-schema.ts` + `tool/schema.ts` — Schema tooling
  - **MCP 集成**:`packages/opencode/src/mcp/` + `tool/mcp-websearch.ts`
  - **`apply_patch.ts`** — 跟 [[codex]] 同款 patch 协议(行业事实标准)
  - **LSP tool**(`tool/lsp.ts` + `lsp.txt`)— 4 家独家:Agent 可调 LSP 拿 diagnostic / hover / definition / symbol
  - **Skill tool**(`tool/skill.ts`)— Skill 是工具调用形式(类 Claude Code skill)
  - **Plan / Build mode 切换工具**(`tool/plan.ts`)— `plan_exit` 工具触发跨 agent 切换(详见 multi_agent)
  - **Repo 探索独家**:`tool/repo_clone.ts` + `tool/repo_overview.ts`
  - 16 个内置 tool(完整列表见 [[../source/opencode]])
  [verified-from-source]

context_mgmt: |
  **完整 compaction + overflow + projectors 三件套**(`packages/opencode/src/session/`):

  ### compaction.ts 关键常量(`compaction.ts:42-48`)
  - `PRUNE_MINIMUM = 20_000` tokens(触发 prune 下限)
  - `PRUNE_PROTECT = 40_000` tokens(保护区间)
  - `TOOL_OUTPUT_MAX_CHARS = 2_000`(单 tool output 字符上限)
  - **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — **skill tool 输出不被 prune**(独家:把 skill 调用结果当 first-class memory,详见 [[../concept/context-compaction]] 更新)
  - `DEFAULT_TAIL_TURNS = 2`
  - `MIN_PRESERVE_RECENT_TOKENS = 2_000` / `MAX_PRESERVE_RECENT_TOKENS = 8_000`

  ### `SUMMARY_TEMPLATE`(`compaction.ts:50-87`)
  **强制 LLM 输出 7 段 Markdown 结构**:Goal / Constraints & Preferences / Progress(Done/In Progress/Blocked)/ Key Decisions / Next Steps / Critical Context / Relevant Files
  4 条规则(`compaction.ts:88-92`):
  - "Keep every section, even when empty."
  - "Use terse bullets, not prose paragraphs."
  - "Preserve exact file paths, commands, error strings, and identifiers when known."
  - "Do not mention the summary process or that context was compacted."

  ### overflow.ts
  - `isOverflow` + `usable` 判定 context 满

  ### projectors.ts(CQRS / event sourcing)
  - `projectors.ts` + `projectors-next.ts` — **状态投影器**,事件流 → 投影 → 当前状态
  - 类 [[../eval/langfuse]] Wide Events 哲学的工程实现

  ### Bus 事件驱动
  - `compaction.ts:31-37` `BusEvent.define("session.compacted", {sessionID})`
  - 整个 session lifecycle 走 `bus/` 事件总线
  [verified-from-source]

memory: |
  - Drizzle ORM + SQLite(`session.sql.ts` + `drizzle.config.ts` + `data-migration.sql.ts`)
  - **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — Skill 调用历史是 first-class memory(`compaction.ts:46`)
  - `session/projectors.ts` CQRS 风格状态投影
  - 没有 Hermes 风格的 `commit_memory_session` / Claude Code `autoDream` / OpenClaw `Memory Dreaming` 等价物 — **opencode 是纯 coding agent,不做 self-evolution memory**
  [verified-from-source]

multi_agent: |
  **业内独家"用户主动 Tab 切换 + 工具触发跨 agent"模式**(README "Agents" + `tool/plan.ts`):

  ### 2 builtin agent + 1 subagent
  - **build** — Default,full-access dev work
  - **plan** — Read-only,denies file edits + asks permission before bash(对照 [[claude-code]] plan permission mode)
  - **`@general` subagent** — 复杂搜索 + 多步任务,`@general` mention 调用

  ### Tab 键切换 build / plan
  - 用户用 Tab 键在 build 跟 plan 之间切换 — **业内最简单 multi-agent 模式**

  ### 工具触发跨 agent 切换(独家)
  - `tool/plan.ts:14-50` **`PlanExitTool`**(`Tool.define("plan_exit", ...)`):plan agent 完成后,用 `Question.ask` 问用户:
    > "Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?"
    Yes → 切到 build agent + 用 plan 文件内容启动新对话
    No → `Question.RejectedError`
  - **跨 agent 切换是 tool 触发,不是 framework 内置** — 业内独家工程实现

  ### Permission 跟 agent 绑定
  - `agent.ts:42` `permission: Permission.Ruleset` — 每个 agent 自己的 permission ruleset
  - `subagent-permissions.ts` — Subagent 权限隔离

  ### Agent 自动生成
  - `agent.ts:50-54` `GeneratedAgent { identifier, whenToUse, systemPrompt }`
  - `Service.generate()` 用 `ai` SDK `streamObject` 从 description 自动生成 agent — **对照 Codex 50 个预定义代号 + 2 个 builtin role,opencode 走"按需生成"路线**

  对照其他 4 家:
  - Hermes/OpenClaw:完全隔离 fire-and-forget
  - Claude Code:SubAgent 隔离 + Fork 继承(2 选)
  - Codex:actor mailbox + control + resume(`DEFAULT_AGENT_MAX_THREADS=6`)
  - **opencode**:Tab 切换 builtin + 工具触发跨切 + agent 自动生成 — **第 4 种路线**(详见 [[../concept/multi-agent-isolation]] 更新)
  [verified-from-source]

eval_loop: |
  **未发现内建 eval framework**。
  opencode 是纯 coding agent,**eval 走外部**(可通过 OTel 出到 [[../eval/langfuse]] / [[../eval/opik]]):
  - `packages/opencode/src/bus/` 事件总线
  - `packages/http-recorder/` HTTP 录制(可能 eval / debug 用)
  - 外部 LLM observability platform 集中查询
  [verified-from-source]

self_evolution: |
  **未发现明显 self-evolution 机制**。
  - 无 Hermes `commit_memory_session` 等价物
  - 无 Claude Code `autoDream` 等价物
  - 无 Codex Goals + token_budget 等价物
  - 无 Opik Agent Optimizer 等价物
  - **`agent/agent.ts:Service.generate()` 是"按需生成 agent"**,但不是事后 self-evolution

  opencode 是纯 coding agent 定位,self-evolution 路线靠外部(用户在 [[../eval/opik]] 跑 Optimizer 后回写 prompt)
  [verified-from-source]

observability: |
  - **OpenTelemetry first-class**(`@effect/opentelemetry/Tracer` 在 `agent.ts` 直接 import)
  - **Bus 事件总线**(`packages/opencode/src/bus/` + `event-v2-bridge.ts`)— 事件驱动架构
  - **Drizzle ORM SQLite 持久化**(`session.sql.ts` + `data-migration.sql.ts`)
  - **`session/projectors.ts` CQRS / event sourcing**
  - HTTP recorder(`packages/http-recorder/`)
  - **可走 OTel 出到 [[../eval/langfuse]] / [[../eval/opik]]**
  [verified-from-source]

hooks: |
  - **Plugin SDK 4 类**(`packages/plugin/src/`):shell / tool / tui / example
  - 没有 Claude Code 27 / Codex 8 / Hermes 16 那种**显式 hook 数量列表**(grep "HOOK" 未找到 enum)
  - Plugin 通过 `@/plugin` 加载(`agent.ts:21` import)
  - **未深查具体 lifecycle hook 数**,但风格上比 Claude Code / Hermes 简化(opencode 偏 plugin SDK + 事件驱动 bus,而非 hook framework)
  [doc-only] [verified-from-source 部分]

safety:
  sandbox: "进程级,通过 shell 沙箱(`packages/opencode/src/shell/`)+ pty(`packages/opencode/src/pty/`)[verified-from-source]。详见 [[../concept/sandbox-isolation]](阶梯 0,信任本地用户的合理偷懒)"
  tool_acl: "**Permission ruleset 跟 agent 绑定**(`agent.ts:42` `permission: Permission.Ruleset`)+ `subagent-permissions.ts` Subagent 权限隔离 + plan agent 默认 deny edits + asks permission before bash(README) [verified-from-source]"
  bash_blocklist: "通过 plan agent permission ruleset(`subagent-permissions.ts`)+ Question 工具 ask user 路径 [verified-from-source]"
  path_traversal: "?(未深查 file system 边界)"
  anti_runaway: "`agent.ts:Schema.optional(Schema.Finite) steps` 字段 — agent 可配 max steps(具体默认值未 grep) [verified-from-source]"
  prompt_injection: "**Effect-TS Schema 强制 IO boundary**(AGENTS.md style guide:'Prefer Effect schema helpers such as `Schema.UnknownFromJsonString`')+ tool input/output 全 Effect Schema 校验 [verified-from-source]"
  secret_scan: "`.gitleaksignore` + `packages/identity/` + 自家 secret 管理 [verified-from-source]"
  auth: "`packages/identity/` + `packages/opencode/src/auth/` + `account/` [verified-from-source]"

provider: |
  **Provider-agnostic 是核心卖点**(README FAQ vs Claude Code 第 2 条):
  - `packages/llm/src/`:`llm.ts` + `provider.ts` + `providers/` + `protocols/`(OpenAI + Anthropic 协议)
  - `cache-policy.ts` — Anthropic prompt cache 一等公民
  - `route/` — Provider routing
  - **OpenCode Zen** 商业 model proxy 服务(README 提及,跟 OpenRouter 同类)
  - 支持:Claude / OpenAI / Google / **本地模型**(Ollama 等)
  [verified-from-source]

channels: |
  **Client/Server 架构 + 多 client(4 家独家)**:
  - `packages/opencode/src/server/` — HTTP server(opencode core 跑这里)
  - `packages/opencode/src/control-plane/` — Control plane
  - **TUI**(`packages/tui/`)— neovim 风格(terminal.shop 团队优化)
  - **Desktop App**(`packages/desktop/`)— Mac arm64/x64 + Windows + Linux(`.deb`/`.rpm`/`.AppImage`)
  - **Web**(`packages/web/`)— Web UI
  - **Mobile**(README 提及"mobile app remote drive",但未在 packages/ 找到)
  - **Slack 集成**(`packages/slack/`)
  - **HTTP recorder**(`packages/http-recorder/`)
  - 详见 [[../concept/agent-runtime-stacks]] 更新

  **Server 跟 client 通信**:HTTP / WebSocket(类 LSP client/server 模型,opencode 自己也实现 LSP)
  [verified-from-source]

persistence: |
  - **Drizzle ORM + SQLite**(`session.sql.ts` + `drizzle.config.ts`)
  - **snake_case schema**(AGENTS.md style guide 强约束:不要 `text("column_name")` 字符串重复,直接 `text()`)
  - `data-migration.sql.ts` + `data-migration.ts` — schema migration
  - `packages/opencode/src/storage/` — Storage abstraction(被 `compaction.ts` import `NotFoundError`)
  - **Local-first** — SQLite 在本地($HOME/.opencode/),不需要远程数据库
  [verified-from-source]

cancellation: |
  - `session/run-state.ts` 运行状态机
  - `Question.RejectedError`(`tool/plan.ts:42`)— 用户拒绝时抛错回退
  - Effect-TS `Effect.fail` / `Effect.die` — 错误处理一等公民
  - 具体 cancel 实现未深查
  [verified-from-source]

# === SkillForge 启示栏 ===
skillforge_takeaway: |
  ## 可借鉴 (10 条)
  1. **LSP first-class 集成**(`packages/opencode/src/lsp/` 6 文件 + `tool/lsp.ts`)— 4 家独家,**SkillForge coding agent 缺这个**,加 LSP 让 agent 看到 diagnostic / hover / definition,**借鉴优先级最高**(详见 [[../concept/lsp-integration]])
  2. **Client/Server 架构 + 多 client**(TUI / Desktop / Web / Mobile)— SkillForge 已经是 server-side,opencode 的 `packages/{tui,web,desktop}/` 分离做法值得参考(channel SPI 实现具体形态)
  3. **Permission ruleset 跟 agent 绑定**(`agent.ts:42` `permission: Permission.Ruleset`)+ `subagent-permissions.ts` — SkillForge SubAgent 权限隔离应跟 agent 绑定
  4. **`PlanExitTool` 工具触发跨 agent 切换**(`tool/plan.ts:14-50`)— SkillForge multi-agent 切换可学这种"工具触发 + Question.ask 用户确认"模式
  5. **`PRUNE_PROTECTED_TOOLS = ["skill"]`**(`compaction.ts:46`)— Skill 调用历史是 first-class memory,SkillForge compaction 应保留 skill 调用结果不 prune
  6. **`SUMMARY_TEMPLATE` 7 段 Markdown 强制结构**(`compaction.ts:50-87`)— SkillForge compact summary 应有等价模板(对照 Hermes Phase B 结构化 summary 更细)
  7. **`apply_patch` 跟 Codex 同款** — SkillForge 编辑工具应支持(行业事实标准)
  8. **`agent.ts:Service.generate()` 自动生成 agent** — `ai` SDK `streamObject` 从 description → `{identifier, whenToUse, systemPrompt}`,SkillForge 可借鉴用 LLM 自动生成 agent
  9. **`packages/llm/src/cache-policy.ts`** — Anthropic prompt cache 一等公民,SkillForge 多 provider 抽象时学
  10. **OpenTelemetry first-class**(`agent.ts:25` `@effect/opentelemetry/Tracer`)— SkillForge trace 走 OTel 跟 Langfuse / Opik 互通

  ## 业内验证 (5 条)
  - **Provider-agnostic + 本地模型支持** opencode + Codex(Apache 2.0,Rust)+ Hermes(Python) 都做,**业内主流做法**
  - **MIT / Apache 2.0 license** opencode + Codex + Opik 都选 OSI 友好 license
  - **Drizzle ORM + SQLite local-first** Bun + Node 生态主流选择
  - **`apply_patch` 协议** opencode + Codex 同款,业内事实标准
  - **`@general` subagent + permission 隔离** 跟其他 4 家一致(隔离派)

  ## SkillForge 领先 (3 条)
  - **Eval Pipeline + Prompt A/B + 自动晋升**:opencode 完全没有,SkillForge first-class
  - **Self-Evolution `/evolve`**:opencode 完全没有,SkillForge meta dev-facing 路线领先
  - **Multi-Provider 统一 SPI 抽象 + ChannelAdapter SPI**:SkillForge Java/Spring SPI 比 opencode `packages/llm/protocols/` callback 模式更统一

  ## 待决策 (4 条)
  1. **是否引入 LSP 集成**:**强烈建议是**,SkillForge coding agent 真正缺口,opencode `lsp/` 6 模块是开源参考实现
  2. **是否做 Tab 切换 builtin agent UI**:SkillForge 当前 SubAgent + Team 复杂,opencode 给"用户主动切 1 个 agent"的极简模式作为对照
  3. **`PRUNE_PROTECTED_TOOLS` 等价**:SkillForge compaction 是否保留 skill 调用结果不 prune?(建议保留,对照 opencode)
  4. **Effect-TS 风格借鉴度**:SkillForge 是 Java/Spring,无法用 Effect-TS,**但 functional 哲学(避 try/catch / 早返回 / inline 变量 / Schema-first IO)对 Java 也适用**,style guide 可学

  ## 风险 (3 条)
  - **Effect-TS 学习曲线陡** — SkillForge Java/Spring 团队学不动,只能学 style guide 哲学
  - **22 packages monorepo 拆分粒度过细** — SkillForge 早期不需要,opencode 是 7+ 年 SST 团队工程经验积累
  - **opencode 没 eval / self-evolution / Optimizer** — SkillForge 不能完全照搬定位,opencode 是**纯 coding agent**,SkillForge 是 coding agent + manager + skill 编排平台,定位更广

# === 学习视角栏 ===
learning_takeaway: |
  ## 跟其他 harness 不同的是 (differentiator)
  1. **Open source Claude Code 替代品**(MIT)— 业内唯一明确"对标 Claude Code 开源替代"的项目,README FAQ 直接列 5 大差异
  2. **LSP first-class 集成**(`lsp/` 6 模块 + `tool/lsp.ts` + Effect-TS 类型化 LSP 协议)— 4 家独家,Agent 可看 diagnostic / hover / definition,跟 IDE 体验对齐
  3. **Client/Server 架构 + 多 client**(TUI / Desktop / Web / Mobile)— 4 家独家(其他 4 家都是 CLI 嵌入式),**TUI 只是众多 client 之一**
  4. **Effect-TS 重度 functional 风格**(`Effect.gen` + `Service` + `Layer` + `Context`)— 4 家独家,业内 LLM agent 项目少见用 Effect-TS
  5. **Permission ruleset 跟 agent 绑定**(`agent.ts:42`)— 跟 [[codex]] role TOML / [[hermes-agent]] delegation max_iterations 不同的 permission 设计
  6. **Plan / Build Tab 切换 + 工具触发跨 agent**(`PlanExitTool`)— 业内独家"工具级 multi-agent 切换"
  7. **Agent 自动生成**(`Service.generate()` 用 `ai` SDK `streamObject`)— LLM 从 description 自动生成 `{identifier, whenToUse, systemPrompt}` agent
  8. **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — Skill 调用历史是 first-class memory,业内独家
  9. **`SUMMARY_TEMPLATE` 7 段强制 Markdown** — 比 [[hermes-agent]] Phase B 5 段更细,比 [[codex]] `templates/compact/prompt.md` 自由 summary 更结构化
  10. **22 国语言 README**(业内最广,Opik 11 / Langfuse 4 / Hermes 1)
  11. **Anomaly Corp(SST 团队)出品** + SST 部署 + OpenCode Zen 商业 proxy — 商业模式 SST + opencode 共生
  12. **`apply_patch` 跟 Codex 同款** — 跨家 patch 协议互通

  ## 优势 (strengths)
  1. **`agent.ts:Service.generate()` 用 `ai` SDK `streamObject` 自动生成 agent** — `{identifier, whenToUse, systemPrompt}` LLM 自动产出,业内独家
  2. **Effect-TS Schema 强制 IO boundary**(AGENTS.md style guide)— 所有 untrusted JSON 走 `Schema.UnknownFromJsonString`,业内最严类型边界
  3. **`SUMMARY_TEMPLATE` "Do not mention the summary process or that context was compacted"**(`compaction.ts:92`)— 防 LLM 在 summary 里说"我刚被压缩了",业内独家工程细节
  4. **AGENTS.md "Reduce total variable count by inlining"**(reducer 哲学)— 业内最细 functional style guide
  5. **6 个 prompt template variant**(`prompt/{compaction,explore,scout,summary,title}.txt`)— 单 agent 多 variant,prompts 跟 agent 解耦
  6. **`packages/llm/src/cache-policy.ts`** + Anthropic cache 抽象 — 跨 provider 缓存抽象
  7. **`projectors.ts` + `projectors-next.ts` CQRS / event sourcing 双 projector** — Wide events 哲学的工程实现
  8. **22 packages monorepo 极细拆分**(opencode + core + tui + desktop + web + llm + plugin + sdk + ...)— 服务化粒度跟 [[../eval/opik]] 6 app 一档
  9. **SST infrastructure 自家用**(`infra/{app,console,enterprise,monitoring,secret,stage}.ts`)— SST 团队 dogfood SST,反过来也是 SST 用户案例
  10. **README 直接对比 Claude Code**(README FAQ "How is this different from Claude Code?")— 业内罕见的 transparent positioning

  ## 哪些点做得好 (what_done_well)
  1. **`PRUNE_PROTECTED_TOOLS = ["skill"]`**(`compaction.ts:46`)— 一行常量定义"哪些工具输出绝对不 prune",**业内独家:把 skill 调用结果当 first-class memory**,工程细节漂亮
  2. **`PlanExitTool` 用 `Question.ask` Yes/No 切换**(`tool/plan.ts:34-44`)— **跨 agent 切换是 tool 触发的**,不是 framework 内置,Yes 切到 build agent + 用 plan 文件启动新对话,No 抛 `Question.RejectedError` — 工程优雅
  3. **`agent.ts:Agent.Info` Effect-TS Schema** 比 zod 更严:`Schema.Literals(["subagent", "primary", "all"])` 编译期保证 mode 只能 3 个值
  4. **`SUMMARY_TEMPLATE` 7 段 Markdown 强制结构 + 4 条规则**(`compaction.ts:50-92`)— 比 [[hermes-agent]] `_generate_summary` 更细致,把"summary 该长啥样"写进代码而非仅 prompt
  5. **`agent.ts:25` `@effect/opentelemetry/Tracer`** 在 agent loop 内直接 import — observability 是 agent loop 的一部分,不是事后接
  6. **AGENTS.md "Avoid try/catch where possible" + Effect-TS** — 用 Effect 类型系统替代异常,**编译期可见所有失败路径**
  7. **`tool/plan-enter.txt` + `plan-exit.txt`** — Plan mode 进入 / 退出有专门 prompt,跟 build mode 切换是有仪式感的转换
  8. **Drizzle snake_case 强约束**(AGENTS.md)— `text()` 不写字符串字段名,字段名跟 column 名同名,**减少错配**
  9. **`bus/` 事件总线 + `BusEvent.define("session.compacted", ...)`** — 完整事件驱动,所有 lifecycle 都是 BusEvent
  10. **22 国语言 README** + `.github/instructions/` Copilot 专属隔离 — 多 LLM 工具兼容性 + 全球本地化双重做到极致
---

# opencode

## 一句话定位

Anomaly Corp(SST 团队)出品的**开源 Claude Code 替代品**(MIT,Bun + TypeScript + **Effect-TS 重度 functional**),22 packages monorepo + **Client/Server 架构(TUI 只是 client)** + **LSP first-class**(4 家独家)+ Provider-agnostic(Claude/OpenAI/Google/local)+ 2 builtin agent(build/plan)Tab 切换 + **工具触发跨 agent 切换**(`PlanExitTool`)。

## 起源与定位

- **Anomaly Corp 出品**(`anomalyco/opencode`,SST 创始人 Jay 团队)
- **MIT license**(2025 起)+ npm `opencode-ai` + 全平台 desktop App + 22 国语言 README
- README FAQ 明确 5 大差异化(对 Claude Code):
  1. 100% open source
  2. Provider-agnostic
  3. Built-in opt-in LSP support
  4. TUI-focused(neovim + terminal.shop 团队)
  5. Client/server 架构

## 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│  Multiple Clients(opencode 4 家独家)                         │
│  ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  TUI   │  │ Desktop  │  │  Web UI  │  │  Mobile (TBD)│  │
│  │(neovim │  │ (全平台) │  │          │  │              │  │
│  │ 风格)  │  │ Mac/Win/ │  │          │  │              │  │
│  │        │  │ Linux)  │  │          │  │              │  │
│  └────┬───┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
└───────┼───────────┼─────────────┼──────────────┬┘            
        │           │             │              │
        └─────────HTTP / WebSocket──────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  packages/opencode (主 server)                                │
│                                                                │
│   ┌─────────────────┐  ┌──────────────────┐                   │
│   │ agent/          │  │ session/         │                   │
│   │ (Effect-TS:     │  │ - compaction.ts  │                   │
│   │  Service/Layer) │  │ - overflow.ts    │                   │
│   │ - mode:         │  │ - projectors.ts  │                   │
│   │   subagent /    │  │   (CQRS)         │                   │
│   │   primary / all │  │ - run-state.ts   │                   │
│   │ - permission    │  │ - retry / revert │                   │
│   │   ruleset 绑定  │  └──────────────────┘                   │
│   │ - generate()    │                                          │
│   │   自动生成      │  ┌──────────────────┐                   │
│   └─────────────────┘  │ tool/(16 个)     │                   │
│                        │ - apply_patch    │                   │
│   ┌─────────────────┐  │ - lsp(独家!)    │                   │
│   │ lsp/(独家)       │  │ - plan(切 agent)│                   │
│   │ - client/server │  │ - skill          │                   │
│   │ - diagnostic    │  │ - repo_clone     │                   │
│   │ - language      │  │ - repo_overview  │                   │
│   │ - launch        │  └──────────────────┘                   │
│   └─────────────────┘                                          │
│                        ┌──────────────────┐                   │
│   ┌─────────────────┐  │ permission/      │                   │
│   │ bus/            │  │ subagent-perms   │                   │
│   │ (event-driven)  │  └──────────────────┘                   │
│   └─────────────────┘                                          │
│   server / control-plane / ide / mcp / acp / shell / pty /    │
│   format / git / image / patch / question / share / etc.      │
└─┬───────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ packages/llm (LLM 抽象)                                       │
│  cache-policy / providers(OpenAI/Anthropic/Google/local)     │
│  protocols / route / tool-runtime                            │
└─────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                  Drizzle ORM + SQLite(local-first)
```

## 13 维度详解

详见 frontmatter 各字段。挑 3 个最有学习价值的展开。

### 1. LSP first-class 集成(4 家独家)

`packages/opencode/src/lsp/` 完整 6 模块 + `packages/opencode/src/tool/lsp.ts` Agent 工具入口。

**Effect-TS Schema 类型化 LSP 协议**(`lsp/lsp.ts:22-49`):
```ts
const Position = Schema.Struct({
  line: NonNegativeInt,
  character: NonNegativeInt,
})
const Range = Schema.Struct({
  start: Position,
  end: Position,
})
const Symbol = Schema.Struct({
  name: Schema.String,
  kind: NonNegativeInt,
  location: Schema.Struct({ uri: Schema.String, range: Range }),
})
```

**Agent 可调 LSP tool 拿**:
- diagnostic(实时编译错误)
- hover(类型 / 文档)
- definition(跳转定义)
- symbol(符号树)

这是 SkillForge coding agent **真正缺口**,详见 [[../concept/lsp-integration]]。

### 2. Plan / Build Tab 切换 + `PlanExitTool` 工具触发跨 agent

**`tool/plan.ts:14-50` `PlanExitTool`** 实现:

```ts
export const PlanExitTool = Tool.define(
  "plan_exit",
  Effect.gen(function* () {
    const session = yield* Session.Service
    const question = yield* Question.Service
    const provider = yield* Provider.Service

    return {
      description: EXIT_DESCRIPTION,  // 来自 plan-exit.txt
      parameters: Parameters,
      execute: (_params, ctx) =>
        Effect.gen(function* () {
          // 拿 plan 文件路径
          // ...
          // 用 Question.ask 问用户
          const answers = yield* question.ask({
            sessionID: ctx.sessionID,
            questions: [{
              question: `Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?`,
              header: "Build Agent",
              options: [
                { label: "Yes", description: "Switch to build agent and start implementing the plan" },
                { label: "No", description: "Stay with plan agent to continue refining the plan" },
              ],
            }],
          })

          if (answers[0]?.[0] === "No") yield* new Question.RejectedError()
          // Yes → 切到 build agent + 用 plan 文件启动新对话
          // ...
        })
    }
  })
)
```

**业内独家"跨 agent 切换是 tool 触发的"工程模式**:
- 不是 framework 内置 hard-coded 转换
- 不是用户手动重新 prompt
- 是 LLM 自己判断"plan 完了"调 `plan_exit` 工具,工具用 `Question.ask` 问用户,用户同意后 framework 切到 build agent + 拿 plan 文件作为 build agent 起点

详见 [[../concept/multi-agent-isolation]] 第 4 种路线。

### 3. SUMMARY_TEMPLATE 7 段强制 Markdown(`compaction.ts:50-92`)

7 段强制结构:
- **Goal**(single-sentence task summary)
- **Constraints & Preferences**
- **Progress**(Done / In Progress / Blocked 三 sub-section)
- **Key Decisions**(decision and why)
- **Next Steps**(ordered next actions)
- **Critical Context**(important technical facts, errors, open questions)
- **Relevant Files**(file or directory path: why it matters)

4 条规则:
- "Keep every section, even when empty."(防 LLM 跳 section)
- "Use terse bullets, not prose paragraphs."(防 LLM 写散文)
- "Preserve exact file paths, commands, error strings, and identifiers when known."(防 LLM 改 identifier)
- **"Do not mention the summary process or that context was compacted."**(防 LLM 在 summary 里说"我刚被压缩了" — 业内独家工程细节)

对照 [[../harness/hermes-agent]] Phase B 5 段(Active Task / Completed Actions / Relevant Files / Critical Context / Remaining Work),opencode **多 Goal + Constraints & Preferences + Key Decisions + Next Steps,更细致**。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`。最关键的 4 个决策点:

1. **是否引入 LSP 集成** — **强烈建议是**,opencode `lsp/` 6 模块是开源参考实现
2. **`PRUNE_PROTECTED_TOOLS` 等价** — SkillForge compaction 应保留 skill 调用结果不 prune
3. **Permission ruleset 跟 agent 绑定** — 学 opencode `agent.ts:42` 设计
4. **Effect-TS 哲学(无法用代码,但 style guide 可学)** — Java/Spring 可借鉴 functional 哲学(避 try/catch、早返回、inline、Schema-first IO)

## 待回答的问题

- `agent.ts:Service.generate()` 自动生成 agent 的具体 prompt 在哪(`generate.txt` 完整内容)
- `subagent-permissions.ts` 具体 permission 检查规则
- `packages/plugin/` 跟 hooks 的关系(opencode 用 plugin 替代 hook?)
- `session/projectors.ts` CQRS 实现细节
- `packages/llm/src/cache-policy.ts` 跨 provider cache 抽象
- `packages/http-recorder/` 是 eval 用还是 debug 用
- `packages/function/` 是什么(lambda?)
- ACP 协议在 opencode 怎么实现(对比 OpenClaw)
- Mobile client 是否实际存在(README 提及但 packages/ 没找到)
- `packages/storybook/` 用途(UI 组件库?)
- OpenCode Zen 模型 proxy 的实际架构
- `packages/enterprise/` 跟 OSS edition 边界
- 自动 generate 的 agent 是 runtime 还是 dev artifact
