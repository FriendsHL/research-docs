---
type: harness
name: codex
status: active
language: rust

# === 版本时效 ===
last_ingested_version: "0.0.0-dev @ 302149d979 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/codex]]"
  - "/Users/youren/myspace/codex/AGENTS.md"
  - "/Users/youren/myspace/codex/codex-rs/core/src/"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/"
  - "/Users/youren/myspace/codex/codex-rs/hooks/src/"
  - "/Users/youren/myspace/codex/codex-rs/otel/src/"
  - "/Users/youren/myspace/codex/codex-rs/external-agent-sessions/src/"
  - "/Users/youren/myspace/codex/codex-rs/sandboxing/src/"
  - "/Users/youren/myspace/codex/codex-rs/rollout/src/"

# === 13 个能力维度 ===

agent_loop: |
  双模式 (verified):
  (a) 传统 `while tool_use` — `core/src/codex_thread.rs` + `core/src/thread_manager.rs` 编排,goals 可 token_budget 中止
  (b) **Code Mode** — LLM 写 JavaScript,在 V8 isolate 跑(`code-mode/src/` crate),工具暴露成 `tools.<name>(...)` 全局对象
  Goal 一等公民: `core/src/goals.rs` 把 ThreadGoal 作为协议级实体, token_budget per goal 内嵌资源约束 (5 OTEL metric)
  [verified-from-source]

tool_protocol: |
  - 强类型 `ToolName` (`protocol/src/tool_name.rs`)
  - `dynamic_tools.rs` (`protocol/src/dynamic_tools.rs`) 支持运行时注册
  - **Code Mode 把 JSON Schema 渲染成 TypeScript** (`code-mode/src/description.rs`, `render_json_schema_to_typescript`) 暴露给 V8
  - MCP 一等: `mcp.rs` + `mcp_skill_dependencies.rs` + `mcp_tool_approval_templates.rs` + `mcp_tool_call.rs`
  - 完整 MCP TypeScript preamble (`code-mode/src/description.rs:44-95`): TextContent/ImageContent/AudioContent/BlobResourceContents/ResourceLink 都进 JS world
  [verified-from-source]

context_mgmt: |
  **3 套并存的 compact 实现**:
  - `core/src/compact.rs` (585 行, local inline)
  - `core/src/compact_remote.rs` (387 行)
  - `core/src/compact_remote_v2.rs` (456 行)
  路由: `should_use_remote_compact_task(provider)` (`compact.rs:65-67`) 按 provider 是否 `supports_remote_compaction()` 决定 local/remote
  注入位置: `InitialContextInjection::{BeforeLastUserMessage, DoNotInject}` (`compact.rs:59-63`),注释明确"模型训练时见到的位置"决定 injection point — mid-turn 用 BeforeLastUserMessage,pre-turn/manual 用 DoNotInject
  常量: `COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000` (`compact.rs:48`)
  Telemetry: 7 个枚举 `CodexCompactionEvent` / `CompactionImplementation` / `Phase` / `Reason` / `Status` / `Strategy` / `Trigger` (`compact.rs:17-23`)
  模板: `templates/compact/prompt.md` + `templates/compact/summary_prefix.md`
  Hook 协作: `core/src/hook_runtime.rs` 暴露 `run_pre_compact_hooks` + `run_post_compact_hooks`,与 hook 8 事件中的 PreCompact/PostCompact 对齐
  [verified-from-source]

memory: |
  **拆 3 个独立 crate**: `memories/read/` + `memories/write/` + `memories/mcp/`
  - `memories/read/src/{citations,prompts,usage,metrics,lib}.rs` — 读侧带 citations + usage tracking + 独立 metrics
  - `protocol/src/memory_citation.rs` — memory 引用是**协议级**(citation 跨 client/server 边界)
  具体 store 后端没深查,但拆分粒度 (read/write 分离 + mcp host) 是其他 3 家没有的
  [verified-from-source]

multi_agent: |
  **Actor 模型** (`core/src/agent/`):
  - `agent/mod.rs` 导出 `agent_resolver` / `control` (`AgentControl` 状态机) / `mailbox` (`Mailbox`/`MailboxReceiver`) / `registry` / `role` / `status`
  - 50 个预定义代号 (`agent/agent_names.txt`,科学家名 Euclid → Chandrasekhar)只是 spawn 时分配的人格代号,**不是 50 个 role**
  - 只有 2 个 builtin role TOML: `awaiter.toml` (单一职责轮询)+ `explorer.toml`
  - **Role 用 `ConfigLayerStack` 层叠** (`agent/role.rs:40-80`),而非 system prompt 注入。preserve caller 当前 profile/provider 不静默 fallback。
  - **Depth 跟踪通过 SessionSource 链** (`agent/registry.rs:65-76`): `SessionSource::SubAgent(ThreadSpawn{depth})`,`next_thread_spawn_depth` + `exceeds_thread_spawn_depth_limit`
  - `DEFAULT_AGENT_MAX_THREADS = Some(6)` (`core/src/config/mod.rs:180`)
  - **multi_agents v1 + v2 双 API** (`tools/handlers/multi_agents/` + `multi_agents_v2/`),`Feature::MultiAgentV2` 控制启用
  - **`resume_agent`** (`multi_agents/resume_agent.rs`) 支持中断恢复 — actor 而非 fire-and-forget
  - `agent_jobs.rs` 提供 job-style agent 调度
  对照其他 3 家: Hermes/OpenClaw 完全隔离(无 resume),Claude Code SubAgent 隔离 + Fork 继承 — Codex 是**第三条路: actor mailbox + ConfigLayer 层叠 + resume**
  [verified-from-source]

eval_loop: |
  不是 Codex 显式强项。`rollout` JSONL + SQLite 索引可作为 replay 基础(`rollout/src/{recorder,list,session_index,sqlite_metrics,state_db}.rs`),snapshot 测试用 `cargo insta`(AGENTS.md:111-128)。
  未发现 Prompt A/B / 自动晋升 / 业内 SWE-bench 等 eval pipeline 一等公民支持。
  [verified-from-source]

self_evolution: |
  **Goal-embedded budget 自约束** (新一类,wiki 当前未覆盖):
  - `core/src/goals.rs` ThreadGoal 内嵌 `token_budget`,运行时 budget_limited 自动注入 `templates/goals/budget_limit.md` 提示模型收敛
  - 没有 Hermes commit_memory_session 等价物,没有 Claude Code autoDream 等价物
  - 没有显式 "/evolve" meta self-evolution(对照 [[concept/self-evolution]] 的两类分类,Codex 是第三类: runtime 内嵌资源约束而非事后提炼)
  - `core/src/personality_migration.rs` 存在但未细查,可能跟 agent 人格延续相关
  [verified-from-source]

observability: |
  **3 家里工程化最深** (`otel/` 独立 crate):
  - `otel/src/{config,events,metrics,otlp,provider,targets,trace_context}.rs` + `metrics/{client,config,error,mod,names,process,runtime_metrics,tags,timer,validation}.rs`
  - 完整 OTLP 导出
  - 5 个 GOAL metric(`codex.goal.created` / `codex.goal.completed` / `codex.goal.duration_seconds` / `codex.goal.token_count` / `codex.goal.budget_limited`,见 `otel/src/metrics/names.rs:31-32`)
  - 7 个 compaction telemetry 枚举
  - `rollout` JSONL + SQLite 双轨持久化,自动关联 `GitInfo`,支持 replay / inspect / list / cursor 分页 / 归档(`SESSIONS_SUBDIR` + `ARCHIVED_SESSIONS_SUBDIR`)
  - `response-debug-context` 独立 crate
  - `analytics` 独立 crate(`codex_analytics::*`)
  [verified-from-source]

hooks: |
  **8 lifecycle 精简版**(`hooks/src/lib.rs:18-27`):
  PreToolUse / PermissionRequest / PostToolUse / PreCompact / PostCompact / SessionStart / UserPromptSubmit / Stop
  其中 6 个事件真正 dispatch matcher (`HOOK_EVENT_NAMES_WITH_MATCHERS`,`hooks/src/lib.rs:34-41`,Stop / UserPromptSubmit 不用)
  独立 crate,有:
  - `engine/` — dispatch engine
  - `registry.rs` + `HooksConfig` — 配置加载
  - `output_spill.rs` — **防 hook 输出爆量**(类 Claude Code MAX_HOOK_OUTPUT_TOKENS)
  - `legacy_notify.rs` — 向后兼容(`legacy_notify_json` / `notify_hook`)
  - `plugin_hook_declarations` — **plugin 显式声明监听哪些 event**(白名单式注册,vs Claude Code 隐式 SDK 注册)
  对照其他: Hermes `VALID_HOOKS=16` / Claude Code `HOOK_EVENTS=27` / Codex `HOOK_EVENT_NAMES=8` — 三家精简程度成阶梯,展开见 [[concept/hooks-design]]
  [verified-from-source]

safety:
  sandbox: "**3 平台 + Windows 独立**(`sandboxing/` + `linux-sandbox/` + `bwrap/` + `windows-sandbox-rs/`):Linux bubblewrap + Linux landlock + macOS Seatbelt + Windows,有 `policy_transforms.rs` 抽象层 [verified-from-source]。详见 [[../concept/sandbox-isolation]](阶梯 1+2+5',业内唯一显式 OS abstraction layer)"
  tool_acl: "权限协议级 (`protocol/src/permissions.rs` + `request_permissions.rs` + `approvals.rs` + `mcp_approval_meta.rs` + `request_user_input.rs`),Hooks 8 个事件中 `PermissionRequest` 是独立事件 [verified-from-source]"
  bash_blocklist: "execpolicy 独立 crate (`execpolicy/` + `execpolicy-legacy/`) + `core/src/exec_policy.rs` + `parse_command.rs` shell 解析 [verified-from-source]"
  path_traversal: "Seatbelt `restricted_read_only_platform_defaults.sbpl` policy + `windows_sandbox_read_grants.rs` 显式 read grant + bwrap mount 隔离 [verified-from-source]"
  anti_runaway: "Goal `token_budget` per goal + `GOAL_BUDGET_LIMITED_METRIC` 信号 + `DEFAULT_AGENT_MAX_THREADS=6`(`config/mod.rs:180`)+ depth 跟踪 (`agent/registry.rs:65-76`) [verified-from-source]"
  prompt_injection: "`code-mode/src/description.rs` Code Mode 沙箱明示 'no Node/no fs/no network/no console',V8 `sandbox = ['v8/v8_enable_sandbox']` (`code-mode/Cargo.toml:13`) [verified-from-source]"
  secret_scan: "`secrets/` 独立 crate + `keyring-store/` 凭据 + `network-proxy/` 网络代理隔离 + `process-hardening/` 进程加固 [verified-from-source]"
  auth: "`agent-identity/` 独立 crate + `protocol/src/auth.rs` + `protocol/src/account.rs` + `login/` 独立 crate(完整身份链) [verified-from-source]"

provider: |
  **每个 provider 都是独立 crate**(`model-provider/` + `model-provider-info/` + `responses-api-proxy/` + `chatgpt/` + `lmstudio/` + `ollama/` + `keyring-store/` + `login/`)。
  对照 SkillForge 的"统一 provider SPI"抽象,Codex 把每家 provider 单独 crate 处理 — 工程化更深但抽象更弱(每加一个 provider 是新 crate 而不是新 impl)
  Compact 路由按 provider 区分 (`should_use_remote_compact_task`,见 context_mgmt)
  [verified-from-source]

channels: |
  **形态多样**(各为独立 crate):
  - TUI: `tui/` (ratatui + Stylize 约定见 AGENTS.md:82-101)
  - CLI: `cli/` + `exec/` + `exec-server/`(headless exec)
  - Daemon / IDE: `app-server/` + `app-server-client/` + `app-server-daemon/` + `app-server-protocol/`(v1 + v2 双版本 RPC,Rust derive TS,见 AGENTS.md:180-220)
  - Cloud 形态: `cloud-tasks/` + `cloud-tasks-client/` + `cloud-tasks-mock-client/` + `chatgpt/`
  - Realtime: `realtime-webrtc/`(语音 / 实时)
  - IDE 扩展:VS Code / Cursor / Windsurf(README 提及)
  [verified-from-source]

persistence: |
  - `rollout/` JSONL session 文件(`SESSIONS_SUBDIR` + `ARCHIVED_SESSIONS_SUBDIR`)+ SQLite 索引(`session_index.rs` + `sqlite_metrics.rs` + `state_db.rs`)
  - `thread-store/` 独立 crate
  - `agent-graph-store/` 独立 crate(agent 关系图持久化)
  - `message-history/` 独立 crate
  - `external-agent-sessions/` ledger:`external_agent_session_imports.json` 防重复(SHA256 content hash,见 `ledger.rs:12`)
  [verified-from-source]

cancellation: |
  - `AgentControl` 状态机 (`core/src/agent/control.rs`)
  - `agent_jobs.rs` job 级控制
  - `arc_monitor.rs` Arc 监视器(可能配合 ref 计数检测 leak/cancel,需深查)
  - **`resume_agent`** (`tools/handlers/multi_agents/resume_agent.rs`) 支持中断后恢复 — Codex 是 4 家里唯一 actor 风格 cancel/resume
  [verified-from-source]

# === 服务 SkillForge 决策的关键栏 ===
skillforge_takeaway: |
  ## 可借鉴 (8 条)
  1. **Goals + token_budget per goal + 5 OTEL metric** — SkillForge eval_loop 可在每个目标内嵌 token 预算,trace 直接关联 goal lifecycle (`codex.goal.created` / `.completed` / `.duration_seconds` / `.token_count` / `.budget_limited`),budget_limited 信号可作为自动晋升/降级决策依据
  2. **Hooks 精简 8 lifecycle**(`PreToolUse` / `PermissionRequest` / `PostToolUse` / `PreCompact` / `PostCompact` / `SessionStart` / `UserPromptSubmit` / `Stop`)— 比 Claude Code 27 个 hook 易理解,SkillForge 应优先精简而非细粒度
  3. **`output_spill` 防 hook 输出爆量**(`hooks/src/output_spill.rs`)— SkillForge hook 实现必加
  4. **`plugin_hook_declarations`** 让 plugin 显式声明监听哪些事件(白名单式注册)— SkillForge plugin 体系应学
  5. **Compact provider 路由**(`should_use_remote_compact_task(provider)`)— SkillForge multi-provider 场景下应该把 server-side compaction 支持作为 provider capability flag
  6. **InitialContextInjection 注入位置区分**(`BeforeLastUserMessage` mid-turn / `DoNotInject` pre-turn-manual)— SkillForge 当前 Light/Full 两档不区分注入位置,可能影响模型对 summary 的解读
  7. **ConfigLayerStack 层叠 role**(`agent/role.rs:40-80`)— SkillForge 当前 sub-agent 用 system prompt 注入,可以改成 config 层叠(preserve caller profile/provider 不静默 fallback)
  8. **Multi-agent actor 模型 + resume_agent** — SkillForge SubAgent (树) + Team (网状) 都是 fire-and-forget,Codex 给出第三条路 (actor mailbox + control 状态机 + resume),适合长任务 agent

  ## 业内验证 (5 条)
  - SubAgent 完全隔离派(Hermes/OpenClaw/Codex multi_agents)是主流,Claude Code Fork 继承 cache 是少数派 — SkillForge 选 SubAgent 隔离方向被 3 家验证
  - Depth-tracking via spawn chain (Codex `SessionSource` / Hermes `parent_session_id` / Claude Code SDK options) — 跟踪 spawn 链是行业共识
  - OTLP 标准命名(`codex.goal.created` 等)— SkillForge trace 命名应对齐 OTLP
  - 3 平台沙箱(bwrap/seatbelt/windows)是 Codex 给出的完整跨平台模板,印证 SkillForge 后续需做平台抽象
  - External agent migration(从其他 harness 导入 session)是 Codex 独家,印证 SkillForge "channel adapter" 思路可延伸到 "agent adapter"

  ## SkillForge 领先 (3 条)
  - **`/evolve` meta self-evolution**(从 session 提取 skill+memory + 迭代建议)Codex 没有等价物 — 这条路 SkillForge 跟 Claude Code insight(已知未 verify)同方向,但 Codex 走 goal budget 自约束方向
  - **Provider 统一 SPI 抽象** — Codex 每个 provider 一个 crate,SkillForge 统一 SPI 抽象更轻量
  - **Channel SPI** — Codex 每个 channel 独立 crate,SkillForge 统一 ChannelAdapter SPI 更容易扩展

  ## 待决策 (5 条)
  1. **是否引入 V8 Code Mode 范式作为 skill 编排第二选项?** — LLM 写 JS 在 sandbox 编排 skill,跟 SkillForge 当前 Spring/Java DSL 互补。判定:工程成本高(嵌 V8),但对长链复杂编排表达力强,建议先做技术 POC
  2. **Goals 协议级 + token_budget** — SkillForge 当前是 task 级,goal 是显式实体可以加(eval pipeline 受益)
  3. **Hooks 数量上限**:学 Codex 8 还是 Claude Code 27 — 建议 8 起步,后续按需扩(对应 SkillForge 当前 6 个 lifecycle hook,基本一致)
  4. **External agent migration** — SkillForge 当前没有,作为 differentiator 候选(从 Cursor/Claude Code/Codex 导入 session)
  5. **App-server protocol v1+v2 双版本 + Rust derive TS** — SkillForge gateway API 版本管理可对齐(`*Params/*Response/*Notification` 命名 + camelCase + experimental 字段标记 + cursor 分页)

  ## 风险 (3 条)
  - **每个 provider 独立 crate** 这条路 SkillForge **不要走** — 抽象成本反而高,统一 SPI 更适合 Spring 生态
  - **app-server v1+v2 并存** 给迁移留 4-6 个月,但长期是技术债 — SkillForge gateway 设计初期应预留版本字段,避免后期改造
  - **V8 嵌入** 会引入 native 依赖(rusty_v8 ~30MB binary),JVM 嵌入 V8 不现实,可考虑用 GraalVM JS 替代

# === 服务个人学习 / 理解 harness 设计空间的关键栏 ===
learning_takeaway: |
  ## 跟其他 harness 不同的是 (differentiator)
  1. **Code Mode V8 isolate** — LLM 写 JavaScript 在 V8 沙箱里调工具(`tools.<name>(...)` 全局对象),不是传统 JSON tool_use。这是一个**真正的新范式**:JSON tool_use 是"模型说 → 框架执行",Code Mode 是"模型写代码 → 沙箱跑代码"。能用 `store()/load()` 跨 exec 调用持久化,能 `await Promise.all(...)` 并行,能 `yield_control()` 流式吐结果。其他 3 家完全没有等价。
  2. **Rust 单体跨平台 + 100+ crates 极细拆分** — Hermes Python / Claude Code TS Bun / OpenClaw TS pnpm 都是单语言,Codex 是**唯一 Rust 实现 + 工程化最深**(read/write memory 都分 crate,每个 provider 一个 crate)
  3. **Goals 作为协议级一等公民 + token_budget per goal** — 4 家里唯一把"目标"作为协议实体并内嵌资源约束的。其他 3 家把 token budget 放在 session/turn 级别
  4. **Multi-agent actor 模型 + resume_agent** — Hermes/OpenClaw 完全隔离 fire-and-forget,Claude Code SubAgent 派发 + Fork 继承,Codex 是 actor mailbox + control 状态机 + 支持中断恢复 — **第三条路**
  5. **External agent migration**(`SESSION_IMPORT_MAX_COUNT=50` / 30 天)— 4 家里独家,从其他 agent harness 导入 session
  6. **Daemon / app-server v1+v2 双版本 RPC + Rust 自动 derive TS** — Codex 真正是 daemon 形态(`app-server-daemon` crate),其他 3 家都是 CLI 嵌入式
  7. **OpenTelemetry 完整集成 + 5 GOAL metric + 7 compaction enum** — observability 工程化深度业内标杆
  8. **每个 provider 一个 crate**(`chatgpt/`/`lmstudio/`/`ollama/`/`responses-api-proxy/`)— 拆分粒度业内最细,跟 SkillForge 单 SPI 抽象路线截然不同

  ## 优势 (strengths)
  1. **Compact 3 套 + provider 路由 + 训练对齐**:`should_use_remote_compact_task(provider)`(`compact.rs:65-67`)按 provider 是否支持 server-side compaction 路由 local/remote。`InitialContextInjection::{BeforeLastUserMessage, DoNotInject}`(`compact.rs:59-63`)把"模型训练时见到的位置"作为注入位置依据 — 业内最精细
  2. **跨平台沙箱抽象**:`policy_transforms.rs` + 3 个 `.sbpl` Seatbelt policy + Windows 独立 crate + bwrap/landlock(`sandboxing/src/`)— 工程化最完整
  3. **强类型协议**:`ToolName`/`ThreadId`/`SessionId` 都是独立类型,Rust 类型系统在 `app-server-protocol` 里 `#[ts(...)]` 自动 derive TypeScript,前后端类型对齐自动化 — 业内独家
  4. **Hooks 8 精简版 + output_spill 防爆量 + plugin 显式声明**(`hooks/src/lib.rs:18-41`)— 比 Claude Code 27 个 hook 易懂,同时保留了关键工程细节
  5. **Goals OTEL 5 metric**(`codex.goal.created` / `.completed` / `.duration_seconds` / `.token_count` / `.budget_limited`,`otel/src/metrics/names.rs:31-32`)— 把 goal lifecycle 完整暴露给监控系统
  6. **External agent migration ledger**(`SESSION_IMPORT_LEDGER_FILE = "external_agent_session_imports.json"`,SHA256 防重复)— 工程细节漂亮
  7. **AGENTS.md 17KB 给 LLM 看**(`AGENTS.md:1-220`)— Codex 把自己的 contributor guide 写给 codex CLI 读,把"模型按规则写代码"做到极致

  ## 哪些点做得好 (what_done_well)
  1. **`should_use_remote_compact_task(provider)` 单函数解决 provider 异构** — 一个 capability flag 把 local/remote compaction 选择上推到 provider,而不是 if-else 散落在 compact 路径上
  2. **`InitialContextInjection` 枚举 + 注释说明"模型训练位置"决定 injection point** — 把"前训练对齐"做成代码级硬约束,而非埋在 prompt template 里
  3. **`SessionSource::SubAgent(ThreadSpawn{depth})` 把 depth 跟踪嵌入 session 来源类型** — depth 不是单独字段,而是 spawn 链的天然属性
  4. **`HOOK_EVENT_NAMES_WITH_MATCHERS: [&str; 6]`**(`hooks/src/lib.rs:34-41`)— 显式区分"哪些事件 dispatch matcher 有意义",Stop/UserPromptSubmit 不用 matcher 因为没有 tool/compaction trigger/session source 可匹配
  5. **`ConfigLayerStack` 层叠 role** + preserve current profile/provider — `role.rs:36-39` 注释明确"rebuilding without those overrides would make a spawned agent silently fall back to the default provider, which is the bug this preservation logic avoids" — 把曾经踩过的坑作为代码注释保留
  6. **`SYSTEM_SKILLS_MARKER_FILENAME + fingerprint`**(`skills/src/lib.rs:14`)缓存避免每次启动重装 system skills — 工程细节
  7. **`output_spill` 单独模块防止 hook 爆量** — 把"防 hook 占用过多 context" 作为独立责任,而非散在 hook engine 里
  8. **`AgentLimitReached { max_threads }`** 错误类型(`agent/registry.rs:80-95`)— 把"agent 上限"信号作为结构化错误,可被 caller 精确捕获
  9. **AGENTS.md 里"resist adding code to codex-core"**(自家 contributor guide 反对 god crate)— 工程文化反 god module 漂亮
  10. **Code Mode 全局函数**(`exit()/text()/image()/store()/load()/notify()/setTimeout()/clearTimeout()/yield_control()/ALL_TOOLS`,`code-mode/src/description.rs:24-34`)— 一个最小完备的 JS host API,精心挑选过的 primitives
---

# Codex (OpenAI Codex CLI)

## 一句话定位
OpenAI 官方 coding agent,**Rust 单体跨平台 monorepo**(100+ crates,Bazel+Cargo+pnpm 三栈),Cloud + Local + Daemon 三形态,首创 **V8 Code Mode** 范式让 LLM 写 JavaScript 调工具。

## 起源与定位
OpenAI 推出的 coding agent CLI,npm/brew 装的本质都是 Rust 二进制。设计哲学跟 ChatGPT Codex(Cloud 形态)+ IDE 集成(VS Code/Cursor/Windsurf)+ 桌面 App 是一体的(README 提及),所以会看到 `cloud-tasks*` + `chatgpt/` + `app-server-daemon` 等 crate。

跟其他 3 家 harness 最大的不同是**工程化深度** —— 100+ crates 极细拆分,每个 provider/channel/sandbox 都是独立 crate,memory 拆 read/write/mcp 3 crate,hooks/skills/otel/rollout/execpolicy 都是独立 crate。

## 核心架构

```
┌─────────────────────────────────────────────────────────────────────┐
│              Channels (各为独立 crate)                              │
│  tui (ratatui)  cli  exec  exec-server  app-server (v1+v2 RPC)      │
│  cloud-tasks  realtime-webrtc  chatgpt  VS Code/Cursor/Windsurf IDE │
└──────────────┬──────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│              app-server-protocol (v1 实验 + v2 正式, RPC)            │
│              Rust derive TS via #[ts(export_to="v2/")]              │
└──────────────┬──────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   codex-core (主 crate)                              │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────────┐    │
│  │ agent/       │  │ goals.rs        │  │ thread_manager.rs    │    │
│  │ (actor 模型) │  │ (token_budget   │  │ codex_thread.rs      │    │
│  │ mailbox      │  │  per goal       │  │ session/             │    │
│  │ control      │  │  + 5 OTEL metric)│  │ context_manager/     │    │
│  │ registry     │  └─────────────────┘  └──────────────────────┘    │
│  │ role (Config │                                                    │
│  │  LayerStack) │  ┌───────────────────────────────────────────┐    │
│  │ 50 names +   │  │ compact.rs / compact_remote / _v2          │    │
│  │ 2 builtins   │  │ should_use_remote_compact_task(provider)   │    │
│  └──────────────┘  │ InitialContextInjection(BeforeLast/Do not)│    │
│                    │ 7 telemetry 枚举                            │    │
│                    └───────────────────────────────────────────┘    │
└─┬───────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│       横切独立 crate                                                  │
│ code-mode (V8 isolate, JS host)  hooks (8 lifecycle)                │
│ skills  memories/{read,write,mcp}  otel (OTLP + 5 GOAL metric)      │
│ rollout (JSONL + SQLite + git)  sandboxing (bwrap+landlock+seatbelt)│
│ external-agent-{migration,sessions} (从他家迁 session)               │
│ execpolicy  secrets  process-hardening  protocol (强类型)            │
│ model-provider*  responses-api-proxy  chatgpt  lmstudio  ollama     │
└─────────────────────────────────────────────────────────────────────┘
```

## 13 维度详解

详见 frontmatter 各字段。下面挑 5 个最有价值的展开。

### 1. Agent Loop — 双模式

(a) **传统 while tool_use**:`core/src/codex_thread.rs` + `thread_manager.rs` 编排,goal `token_budget` 超限时注入 `budget_limit.md` 模板让模型收敛
(b) **Code Mode V8 isolate**(`code-mode/` crate):LLM 写 JavaScript,V8 fresh isolate 跑 async module,工具变成 `tools.<normalized_name>(...)` 调用,完成后 isolate 死亡丢弃未 await 的 promise

Code Mode 的 prompt 描述(`code-mode/src/description.rs:12-34`)明示:
- "Runs raw JavaScript -- no Node, no file system, no network access, no console"
- 全局函数:`exit()` / `text()` / `image()` / `store(key, value)` / `load(key)` / `notify()` / `setTimeout()` / `clearTimeout()` / `yield_control()` / `ALL_TOOLS`
- Pragma `// @exec: {"yield_time_ms": 10000, "max_output_tokens": 1000}` 控制超时和 token 上限

这是 4 家 harness 里最大胆的范式选择 —— 把"工具编排"从 framework 责任搬到 LLM 责任,framework 只提供 sandbox + primitives。详见 [[concept/code-mode-orchestration]]。

### 2. Context Compaction — 3 套 + provider 路由 + 训练对齐

`compact.rs` / `compact_remote.rs` / `compact_remote_v2.rs` 三套并存,**路由靠 `should_use_remote_compact_task(provider)`**(`compact.rs:65-67`)。这意味着 OpenAI provider 走 remote(`/responses` 的 server-side compact)、其他 provider 走 local。

**`InitialContextInjection` 枚举**(`compact.rs:59-63`)是亮点:
- `BeforeLastUserMessage` — mid-turn compact 用,因为"模型训练时见到的"compaction summary 在 user message 之前
- `DoNotInject` — pre-turn / manual compact 用,clear `reference_context_item` 让下一 turn 重新 inject

把"前训练对齐"做成代码级硬约束,而不是埋在 prompt template 里 —— 这是工程细节漂亮的体现。

`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000` 跟 7 个 telemetry 枚举(`Implementation/Phase/Reason/Status/Strategy/Trigger` + `CodexCompactionEvent`)让 compaction 决策完全可观测。详见 [[concept/context-compaction]]。

### 3. Multi-Agent — Actor 模型 + ConfigLayer 层叠

`core/src/agent/` 是完整的 actor 模型:
- `mailbox.rs` — 消息队列(`Mailbox` + `MailboxReceiver`)
- `control.rs` — `AgentControl` 状态机
- `registry.rs` — `AgentRegistry` 跟 `reserve_spawn_slot(max_threads)`,超限抛 `CodexErr::AgentLimitReached`
- `role.rs` — `apply_role_to_config` 用 `ConfigLayerStack` 层叠(不是 system prompt 注入)
- `agent_resolver.rs` — role name → role config 解析

**`DEFAULT_AGENT_MAX_THREADS = Some(6)`**(`config/mod.rs:180`),`Feature::MultiAgentV2` 控制启用 v2 API。

**Depth 跟踪**(`registry.rs:65-76`)通过 `SessionSource::SubAgent(ThreadSpawn{depth})` 链,`next_thread_spawn_depth` + `exceeds_thread_spawn_depth_limit` 在 3 个 spawn 入口(`multi_agents/spawn.rs:62-64` + `multi_agents/resume_agent.rs:47-49` + `multi_agents_v2/spawn.rs:67`)都过闸门。

**`resume_agent`**(`multi_agents/resume_agent.rs`)是 Codex 独家 — 4 家里唯一支持中断恢复的 actor。

50 个预定义代号(`agent_names.txt`:Euclid → Chandrasekhar)只是 spawn 时分配的人格代号 **不是 50 个 role**。Builtin role 只有 2 个:`awaiter`(单一职责轮询)+ `explorer`。

`awaiter.toml` 显示了 OpenAI 对"单一职责子 agent"的偏好:
> "Your role is to await the completion of a specific command or task and report its status only when it is finished. You must NOT: Modify the task. Interpret or optimize the task. Perform unrelated actions. Stop awaiting unless explicitly instructed."

详见 [[concept/multi-agent-isolation]]。

### 4. Hooks — 8 lifecycle 精简版

`hooks/src/lib.rs:18-27` 完整列表:`PreToolUse` / `PermissionRequest` / `PostToolUse` / `PreCompact` / `PostCompact` / `SessionStart` / `UserPromptSubmit` / `Stop`。

**只有 6 个 dispatch matcher**(`hooks/src/lib.rs:34-41`)— `Stop` 跟 `UserPromptSubmit` 不需要 matcher,因为没有 tool name / compaction trigger / session source 可匹配。这种**显式区分**比 Claude Code "所有 hook 都有 matcher" 的写法更精确。

独立 crate,`output_spill.rs` 防 hook 输出爆量,`plugin_hook_declarations` 让 plugin 显式声明监听哪些事件(白名单式注册)。详见 [[concept/hooks-design]]。

### 5. External Agent Migration — 4 家独家

`external-agent-sessions/` crate 提供从其他 harness(Claude Code 等)迁移 session 到 Codex 的能力。

- `detect.rs:10-11` — `SESSION_IMPORT_MAX_COUNT: usize = 50;` + `SESSION_IMPORT_MAX_AGE: Duration = Duration::from_secs(30 * 24 * 60 * 60);`(30 天)
- `ledger.rs:12` — `external_agent_session_imports.json` 防重复(SHA256 content hash + `imported_thread_id` 关联)
- `records.rs:14-16` — `NOTE_MAX_LEN=2_000` + `TOOL_RESULT_MAX_LEN=4_000`(导入截断阈值)+ `EXTERNAL_AGENT_TOOL_CALL_TAG` / `EXTERNAL_AGENT_TOOL_RESULT_TAG` 标签

这是 SkillForge 可以模仿的差异化方向 —— "agent adapter" 思路。详见 [[concept/agent-interop]]。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`。最关键的 3 个决策点:

1. **是否引入 V8 Code Mode 作为 skill 编排第二选项?** — JVM 嵌入 V8 不现实,但可考虑 GraalVM JS 替代
2. **Goals 协议级 + token_budget per goal + 5 OTEL metric** — eval pipeline 受益,建议加
3. **Hooks 数量 8 vs 27** — 建议 8 起步,SkillForge 当前 6 个 lifecycle 跟 Codex 更接近

## 待回答的问题

- `core/src/personality_migration.rs` 具体是 agent 人格迁移还是 model 升级时的 prompt 迁移?需深查
- `arc_monitor.rs` 跟 cancel/leak 检测的具体机制
- Memory store 的具体后端(看了 `memories/read/src/` 但未看 `write/` 跟 `mcp/`)— 是 SQLite?vector?文件?
- `agent-graph-store/` 把 agent 关系图持久化是什么形态?
- `code-mode/src/runtime/{callbacks,globals,module_loader,timers,value}.rs` 5 个 V8 host 模块的具体接入方式
- `app-server-protocol/v2` 跟 v1 的具体差异(目前只看到 v2 是新 API)
- `responses-api-proxy/` 是什么 — OpenAI Responses API 的本地代理?
- `agent-identity/` 独立 crate 的身份模型(怎么做 multi-tenant?)
- `external-agent-migration/` 跟 `external-agent-sessions/` 的分工
- `cloud-tasks*` Cloud 形态的部署形态(是 chatgpt.com/codex Web 的后端?)
