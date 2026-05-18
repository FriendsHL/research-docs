---
type: source
name: opencode
origin: "/Users/youren/myspace/opencode/ (Anomaly Corp / anomalyco/opencode, MIT)"
ingested_at: 2026-05-16
ingested_commit: 2d6bedecd
ingested_commit_date: 2026-05-15
ingested_version_label: "1.15.0(`packages/opencode/package.json`)"
contributed_to:
  - harness/opencode.md
  - concept/lsp-integration.md
  - concept/agent-runtime-stacks.md
  - concept/multi-agent-isolation.md
  - concept/context-compaction.md
  - concept/code-mode-orchestration.md
---

# Source — opencode

> Anomaly Corp(SST 创始人 Jay 团队)出品的**开源 Claude Code 替代品**,MIT,Bun + TypeScript + **Effect-TS 重度 functional 风格** + Drizzle ORM + SQLite。22 packages monorepo + 22 国语言 README(业内最广)+ npm 包 `opencode-ai`。**Client/Server 架构,TUI 只是众多 client 之一**。

## 仓库结构概览

```
opencode/
├── packages/                            # 22 个 packages(pnpm + Bun monorepo)
│   ├── opencode/                        # 主 agent 包(packages/opencode/src/ 45+ 子模块)
│   ├── core/                            # 核心库(共享 utility / event / schema)
│   ├── app/                             # 主 app
│   ├── tui/                             # TUI(neovim 风格,terminal.shop 团队优化)
│   ├── desktop/                         # 桌面 App(Mac/Win/Linux 全平台)
│   ├── web/                             # Web UI
│   ├── llm/                             # LLM 抽象层(provider/protocol)
│   ├── plugin/                          # 插件 SDK(shell/tool/tui 3 类)
│   ├── sdk/                             # 用户 SDK
│   ├── enterprise/                      # 企业版
│   ├── console/                         # 管理控制台
│   ├── containers/                      # 容器集成
│   ├── docs/                            # 文档站
│   ├── function/                        # Function(lambda 集成?)
│   ├── http-recorder/                   # HTTP 录制(eval / debug 用)
│   ├── identity/                        # 身份认证
│   ├── extensions/                      # 扩展
│   ├── script/                          # 脚本
│   ├── slack/                           # Slack 集成
│   ├── storybook/                       # Storybook
│   ├── ui/                              # UI 组件
│   └── web/                             # Web
├── .opencode/                           # 仓库自家 dogfood 目录(类 Langfuse/Opik .agents/)
│   ├── agent/{duplicate-pr,triage}.md   # 2 个自定义 agent
│   ├── command/                         # 8 个 slash command
│   ├── skills/{effect,improve-codebase-architecture}/
│   ├── plugins/{smoke-theme.json,tui-smoke.tsx}/
│   ├── tool/{github-pr-search,github-triage}.ts
│   ├── themes/
│   ├── opencode.jsonc                   # 项目配置($schema from opencode.ai/config.json)
│   └── tui.json                         # TUI 配置
├── infra/{app,console,enterprise,monitoring,secret,stage}.ts  # SST infrastructure
├── nix/                                 # Nix 包定义
├── patches/                             # 打补丁
├── github/                              # GitHub Actions
├── AGENTS.md (4KB)                      # 极简 functional style guide
├── README.{en,zh,zht,ko,de,es,fr,it,da,ja,pl,ru,bs,ar,no,br,th,tr,uk,bn,gr,vi}.md  # 22 国语言
├── flake.nix / flake.lock               # Nix 集成
├── bun.lock (874KB)                     # Bun 锁
└── install (13KB)                       # 安装脚本(`curl -fsSL https://opencode.ai/install | bash`)
```

## 关键 file:line 锚点

### Agent 系统(Effect-TS schema)
- `packages/opencode/src/agent/agent.ts:29-46` — **`Agent.Info` Effect-TS Schema 定义**(用 `Schema.Struct`,非 zod):
  - **`mode: Schema.Literals(["subagent", "primary", "all"])`** — 3 个 agent 模式
  - **`permission: Permission.Ruleset`** — permission 跟 agent 绑定(类 Codex `role.toml` ConfigLayerStack)
  - `model: {modelID, providerID}` 跟 agent 绑定
  - `topP` / `temperature` / `steps` 跟 agent 绑定
  - `prompt` 跟 agent 绑定
- `agent.ts:50-54` — **`GeneratedAgent`** schema:`{identifier, whenToUse, systemPrompt}`,**`Service.generate()` 用 `ai` SDK `streamObject` 自动生成 agent**(从 description)
- `agent.ts:1-26` — 依赖:Effect-TS (`Effect, Context, Layer, Schema, Option`)+ `remeda`(functional lodash)+ `@effect/opentelemetry/Tracer`(OTel)
- `agent.ts:10-15` — **6 个 prompt template**:`prompt/compaction.txt` + `prompt/explore.txt` + `prompt/scout.txt` + `prompt/summary.txt` + `prompt/title.txt` + `generate.txt`
- `packages/opencode/src/agent/subagent-permissions.ts` — Subagent 权限隔离

### Builtin Agent(2 个 + 1 个 subagent)
- README `### Agents` section — 完整列出:
  - **build** — Default,full-access,**Tab 切换**
  - **plan** — Read-only,denies file edits,asks permission before bash,**Tab 切换**
  - **`@general` subagent** — 复杂搜索 + 多步任务,`@general` mention 调用

### Plan / Build Tab 切换(工具级实现)
- `packages/opencode/src/tool/plan.ts:14-50` — **`PlanExitTool`**(`Tool.define("plan_exit", ...)`)— plan agent 完成后,用 `Question.ask` 问用户:
  > "Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?"
  Yes → 切到 build agent + 用 plan 文件内容启动新对话 / No → `Question.RejectedError`
- `packages/opencode/src/tool/plan-enter.txt` + `plan-exit.txt` — plan mode 进入 / 退出的 prompt
- **跨 agent 切换是 tool 触发的**(`plan_exit` 工具),不是 framework 内置 — **业内独家工程实现**

### Compaction 系统
- `packages/opencode/src/session/compaction.ts:42-48` — **关键常量**:
  - `PRUNE_MINIMUM = 20_000`(tokens,触发 prune 的下限)
  - `PRUNE_PROTECT = 40_000`(tokens,保护区间)
  - `TOOL_OUTPUT_MAX_CHARS = 2_000`(单 tool output 字符上限)
  - **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — **skill tool 输出不被 prune**(独家:把 skill 调用结果当 first-class memory)
  - `DEFAULT_TAIL_TURNS = 2`(默认保留尾部 turn 数)
  - `MIN_PRESERVE_RECENT_TOKENS = 2_000` / `MAX_PRESERVE_RECENT_TOKENS = 8_000`
- `compaction.ts:50-87` — **`SUMMARY_TEMPLATE`** 强制 Markdown 7 段结构:Goal / Constraints & Preferences / Progress(Done/In Progress/Blocked)/ Key Decisions / Next Steps / Critical Context / Relevant Files
- `compaction.ts:88-92` — 4 条 SUMMARY 规则:
  - "Keep every section, even when empty."
  - "Use terse bullets, not prose paragraphs."
  - "Preserve exact file paths, commands, error strings, and identifiers when known."
  - "Do not mention the summary process or that context was compacted."
- `compaction.ts:31-37` — `BusEvent.define("session.compacted", {sessionID})` — **compaction 是事件驱动**
- `packages/opencode/src/session/overflow.ts` — `isOverflow` + `usable` 判定 context 满
- `packages/opencode/src/session/projectors.ts` + `projectors-next.ts` — **状态投影器(CQRS / event sourcing 风格)**— wide events 哲学的工程实现

### LSP 集成(4 家独家 first-class)
- `packages/opencode/src/lsp/` 完整 6 文件:`client.ts` + `server.ts` + `diagnostic.ts` + `language.ts` + `launch.ts` + `lsp.ts`
- `lsp/lsp.ts:1-15` — 依赖:Effect-TS Schema + Bus(`lsp.updated` event)+ `lspspawn` from `./launch`
- `lsp/lsp.ts:22-49` — **Effect-TS Schema 定义 LSP 协议类型**:`Position` / `Range` / `Symbol` / `DocumentSymbol` — **LSP 协议类型化**
- `packages/opencode/src/tool/lsp.ts` + `lsp.txt` — **Agent 可调 LSP tool 拿 diagnostic / hover / definition / symbol**

### Session 模块(CQRS / event sourcing)
- `packages/opencode/src/session/` 20+ 文件:
  - `session.ts` + `session.sql.ts`(Drizzle SQL schema,snake_case per AGENTS.md style guide)
  - `message.ts` + `message-v2.ts`(message v1/v2 双版本)
  - `compaction.ts` + `overflow.ts` — Compaction
  - **`projectors.ts` + `projectors-next.ts`** — CQRS 状态投影器
  - `processor.ts` — 处理器
  - `retry.ts` + `revert.ts` — 重试 + 撤销
  - `summary.ts` — Summary
  - `run-state.ts` — 运行状态机
  - `instruction.ts` — System instruction
  - `prompt.ts` + `prompt/` — Prompt 管理
  - `schema.ts` — schema 定义(`SessionID`, `MessageID`, `PartID`)
  - `status.ts` — Status

### Tool 集合(16 个内置)
- `packages/opencode/src/tool/`:
  - `apply_patch.ts` — **跟 Codex 同款 patch 协议**
  - `edit.ts` / `read.ts` / `glob.ts` / `grep.ts` — 文件操作
  - **`lsp.ts`** — LSP tool(独家)
  - `mcp-websearch.ts` — MCP websearch
  - **`plan.ts` + `plan-enter.txt` + `plan-exit.txt`** — Plan mode 工具
  - `question.ts` + `question.txt` — Human-in-the-loop
  - **`repo_clone.ts` + `repo_overview.ts`** — Repo 探索独家
  - `shell/` + `shell.ts` — Shell
  - **`skill.ts`** — Skill 调用工具
  - `external-directory.ts` — 外部目录
  - `invalid.ts` — Invalid tool 处理
  - `json-schema.ts` + `schema.ts` — Schema tooling
  - `registry.ts` — Tool dynamic registry

### Client/Server 架构(4 家独家)
- `packages/opencode/src/server/` — HTTP server(opencode core 跑这里)
- `packages/opencode/src/control-plane/` — Control plane
- `packages/opencode/src/bus/` — 事件总线
- `packages/opencode/src/event-v2-bridge.ts` — Event v2 桥
- **TUI / Desktop / Web 都是 client,通过 HTTP / WebSocket 跟 server 通信**
- `packages/tui/` + `packages/desktop/` + `packages/web/` — 多 client 实现

### LLM 抽象
- `packages/llm/src/`:
  - `cache-policy.ts` — Cache policy(类 Anthropic prompt cache)
  - `llm.ts` + `provider.ts` + `providers/` — Provider 抽象
  - `protocols/` — OpenAI + Anthropic 协议
  - `route/` — Routing
  - `tool.ts` + `tool-runtime.ts` — Tool runtime
  - `schema/` — Schema

### Plugin SDK(精简 4 类)
- `packages/plugin/src/index.ts` — 入口
- `packages/plugin/src/shell.ts` — Shell plugin
- `packages/plugin/src/tool.ts` — Tool plugin
- `packages/plugin/src/tui.ts` — TUI plugin
- `packages/plugin/src/example.ts` + `example-workspace.ts` — 示例

### Multi-platform support
- **Desktop App**(README 全平台):
  - macOS Apple Silicon `opencode-desktop-mac-arm64.dmg`
  - macOS Intel `opencode-desktop-mac-x64.dmg`
  - Windows `opencode-desktop-windows-x64.exe`
  - Linux `.deb` / `.rpm` / `.AppImage`
  - Homebrew Cask:`brew install --cask opencode-desktop`
  - Scoop:`scoop install extras/opencode-desktop`
- 安装路径优先级(README):
  1. `$OPENCODE_INSTALL_DIR`
  2. `$XDG_BIN_DIR`
  3. `$HOME/bin`(若存在或可创建)
  4. `$HOME/.opencode/bin`(默认 fallback)

### SST infrastructure
- `infra/{app,console,enterprise,monitoring,secret,stage}.ts` — **SST(Serverless Stack)部署**,SST 团队自家做的工具

### Effect-TS 重度集成
- `packages/opencode/src/effect/` 子目录 — Effect-TS runtime helpers
- `AGENTS.md` style guide:"Prefer Effect schema helpers such as `Schema.UnknownFromJsonString` and `Schema.decodeUnknownOption` over manual `JSON.parse` wrapped in `Effect.try`"
- 整个 `agent.ts` / `compaction.ts` / `lsp.ts` / `tool/plan.ts` 都用 Effect.gen + Service + Layer + Context

### AGENTS.md 极简 functional Style Guide(4KB)
- "Avoid try/catch where possible"
- "Avoid `any` type"
- "Avoid else"(早返回)
- "Reduce total variable count by inlining"
- "Avoid unnecessary destructuring"
- "functional array methods (flatMap, filter, map) over for loops"
- "Use type inference"
- "Bun APIs(`Bun.file()`)"
- "Drizzle snake_case schema(不要 `text("column_name")` 字符串重复)"
- "Testing: Avoid mocks as much as possible"
- "ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE"

## 与 wiki 现有记录的关系

### 印证
- **2 个 builtin agent(build / plan)** 印证业内"精简 builtin agent"主流做法(Codex 2 个 builtin `awaiter/explorer`,Opik 6 个,opencode 2 个)
- **`apply_patch` 工具协议** 跟 [[../harness/codex]] `apply_patch.rs` 同款,印证 patch 协议是行业事实标准
- **ACP 协议集成** 跟 [[../harness/openclaw]] `extensions/acpx/` 一致,印证 ACP 是跨 agent 通信尝试
- **`@general` subagent + `subagent-permissions.ts`** 印证 [[../concept/multi-agent-isolation]] "隔离派"主流
- **22 国语言 README** 印证 [[../eval/opik]] 11 国 + [[../eval/langfuse]] 4 国趋势,**开源 LLM 工具走全球本地化**

### 矛盾
- **LSP first-class**(`lsp/` 6 模块 + `tool/lsp.ts`)— 4 家独家,**wiki 缺 LSP 维度**,需新建 [[../concept/lsp-integration]]
- **Client/Server 架构 + 多 client**— wiki 当前 channels 维度对"TUI 只是 client"路线展开不够
- **Effect-TS 极端 functional + Java/Spring OOP** 哲学冲突 — 应建 [[../concept/agent-runtime-stacks]] 对照 5 家
- **Plan / Build Tab 切换 + `plan_exit` 工具触发** — **业内独家的 multi-agent 模式**,[[../concept/multi-agent-isolation]] 当前 3 路线(隔离 / 继承 / actor)不够,需加第 4 种"用户主动 Tab 切换 + 工具级跨 agent"
- **`projectors.ts` CQRS 风格** — [[../concept/observability]] 当前没有 CQRS / event sourcing 维度
- **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — skill tool 输出不被 prune,**业内独家把 skill 调用结果当 first-class memory**,[[../concept/context-compaction]] 需加这条

### 新增
- **harness/opencode.md**(新建,跟 [[../harness/claude-code]] 直接对照)
- **concept/lsp-integration.md**(新建)
- **concept/agent-runtime-stacks.md**(新建)
- **[[../concept/multi-agent-isolation]] 加第 4 种**:Tab 切换 builtin agent + tool 触发跨 agent 切换
- **[[../concept/context-compaction]] 加 opencode 路线**:7 段强制 Markdown SUMMARY_TEMPLATE + projectors CQRS
- **[[../concept/code-mode-orchestration]] 加 opencode**:工具级 mode 切换(plan-enter/exit)作为 Code Mode 的对立面(Code Mode 是 LLM 写代码编排,opencode 是 framework 提供工具让 LLM 切 mode)

## 事实漂移 / 注意点

- **opencode 版本 = 1.15.0**(`packages/opencode/package.json`)+ commit `2d6bedecd` @ 2026-05-15
- **`anomalyco/opencode` 是仓库**(Anomaly Corp = SST 团队 Jay 的公司),不是 `opencodeai/opencode` 等其他同名项目
- **跟 Opik / Langfuse 等 eval framework 不同**,opencode 是**纯 coding agent**(没有内建 eval / observability platform),可走 OTel 出到 Langfuse / Opik 集中查询
- **Bun + Effect-TS + Drizzle + SQLite** 全栈选型,跟 Hermes(Python)/ Claude Code(TS + Bun + Ink TUI)/ Codex(Rust)/ Opik / Langfuse(TS / Java + Node)都不同
- **SST(Serverless Stack)** 部署 — SST 团队自家工具,反过来也是 SST 用户案例
- **OpenCode Zen** 是商业 model proxy 服务(README 提及),不是开源功能 — 类 OpenRouter
- AGENTS.md 提**默认 branch 是 `dev`**(不是 `main`)— git workflow 不同
- AGENTS.md 提"Tests cannot run from repo root (guard: `do-not-run-tests-from-root`); run from package dirs like `packages/opencode`" — monorepo testing 约束
