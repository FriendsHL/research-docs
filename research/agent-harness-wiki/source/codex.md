---
type: source
name: codex
origin: "/Users/youren/myspace/codex/ (OpenAI Codex CLI, public source)"
ingested_at: 2026-05-16
ingested_commit: 302149d979
ingested_commit_date: 2026-05-15
ingested_version_label: 0.0.0-dev (workspace + npm 都标 0.0.0, monorepo 不打 semver)
contributed_to:
  - harness/codex.md
  - concept/code-mode-orchestration.md
  - concept/hooks-design.md
  - concept/agent-interop.md
  - concept/multi-agent-isolation.md
  - concept/context-compaction.md
  - concept/self-evolution.md
---

# Source — codex (OpenAI Codex CLI)

> Rust monorepo,100+ Cargo crates,Bazel + Cargo + pnpm 三构建栈。docs/ 全是占位指向 developers.openai.com,因此必须 **B 模式**从源码读。

## 仓库结构概览

```
codex/
├── codex-rs/          # Rust 主体, 100+ crates
│   ├── core/          # 主 crate (注释明确"resist adding code to codex-core"!)
│   ├── code-mode/     # V8 isolate JS-host 范式
│   ├── hooks/         # 8 lifecycle events 独立 crate
│   ├── skills/        # 系统 skill, include_dir 嵌入
│   ├── memories/{read,write,mcp}/  # 3 个独立 crate (读写拆开)
│   ├── otel/          # OpenTelemetry 完整 (metrics/events/otlp/trace_context)
│   ├── rollout/       # JSONL session + SQLite 索引
│   ├── sandboxing/    # bwrap + landlock + seatbelt + 3 个 .sbpl policy
│   ├── linux-sandbox/, bwrap/, windows-sandbox-rs/  # 平台专属
│   ├── protocol/      # 强类型 ToolName/ThreadId/SessionId + plan_tool + memory_citation
│   ├── app-server-protocol/  # v1 + v2 双版本 RPC, Rust derive TS
│   ├── app-server/, app-server-client/, app-server-daemon/  # daemon 模式
│   ├── cloud-tasks*/  # Cloud 模式 (本地/云端双形态)
│   ├── external-agent-{migration,sessions}/  # 从其他 harness 迁 session
│   ├── execpolicy/, execpolicy-legacy/  # exec 策略
│   ├── secrets/, process-hardening/  # 安全
│   ├── model-provider/, model-provider-info/, responses-api-proxy/, chatgpt/, lmstudio/, ollama/, keyring-store/, login/  # provider 全拆 crate
│   ├── tui/, cli/, exec/, exec-server/  # UI / 入口
│   ├── realtime-webrtc/  # 语音 / 实时
│   ├── agent-graph-store/, agent-identity/, thread-manager-sample/, thread-store/, message-history/  # 持久化
│   └── ... (还有 60+ crate)
├── codex-cli/         # npm wrapper (本质是 Rust 二进制)
├── docs/              # 占位符, 真 docs 在 developers.openai.com
├── AGENTS.md          # 17KB, Codex 给自己 LLM 的 contributor guide
└── BUILD.bazel, MODULE.bazel, Cargo.toml, package.json, pnpm-workspace.yaml
```

## 关键 file:line 锚点

### Agent Loop / Multi-Agent
- `core/src/agent/mod.rs:1-15` — `agent` 模块导出:`agent_resolver` / `control` / `mailbox` / `registry` / `role` / `status`(actor 模型)
- `core/src/agent/registry.rs:65-76` — `next_thread_spawn_depth` + `exceeds_thread_spawn_depth_limit`,depth 跟踪通过 `SessionSource::SubAgent(ThreadSpawn{depth})` 链
- `core/src/agent/registry.rs:80-95` — `reserve_spawn_slot(max_threads)`,超限抛 `CodexErr::AgentLimitReached`
- `core/src/config/mod.rs:180` — `pub(crate) const DEFAULT_AGENT_MAX_THREADS: Option<usize> = Some(6);`
- `core/src/config/mod.rs:2941-2955` — `if features.enabled(Feature::MultiAgentV2)` 才启用 v2 + fallback 到 `DEFAULT_AGENT_MAX_THREADS`
- `core/src/agent/agent_names.txt` — 50 个预定义科学家名字(Euclid → Chandrasekhar)
- `core/src/agent/builtins/awaiter.toml` — 单一职责轮询 agent("只 await,不修改/优化/解释")
- `core/src/agent/builtins/explorer.toml` — 另一个 builtin role
- `core/src/agent/role.rs:29` — `pub const DEFAULT_ROLE_NAME: &str = "default";`
- `core/src/agent/role.rs:40-80` — `apply_role_to_config`,**Role 用 `ConfigLayerStack` 层叠**(非 system prompt 注入),preserve caller 当前 profile/provider 不静默 fallback
- `core/src/tools/handlers/multi_agents/spawn.rs:62-64` + `multi_agents/resume_agent.rs:47-49` + `multi_agents_v2/spawn.rs:67` — 3 个 spawn/resume 入口都过 `next_thread_spawn_depth` + `exceeds_thread_spawn_depth_limit` 闸门
- `core/src/tools/handlers/agent_jobs.rs:115-117` — agent jobs 也走同一 depth 检查

### Code Mode (V8 isolate)
- `code-mode/src/lib.rs:18-37` — 公开 API:`render_json_schema_to_typescript` / `CodeModeService` / `CodeModeTurnHost`,常量 `PUBLIC_TOOL_NAME="exec"` + `WAIT_TOOL_NAME="wait"`
- `code-mode/src/runtime/mod.rs:23-26` — 默认值 `DEFAULT_EXEC_YIELD_TIME_MS=10_000` / `DEFAULT_WAIT_YIELD_TIME_MS=10_000` / `DEFAULT_MAX_OUTPUT_TOKENS_PER_EXEC_CALL=10_000`,`EXIT_SENTINEL="__codex_code_mode_exit__"`
- `code-mode/src/description.rs:10-42` — `EXEC_DESCRIPTION_TEMPLATE` + `WAIT_DESCRIPTION_TEMPLATE`(LLM 看到的 tool 描述,**明示是 V8 sandbox, no Node/no fs/no network/no console**)
- `code-mode/src/description.rs:44-95` — MCP TypeScript preamble(把 MCP CallToolResult/TextContent/ImageContent/AudioContent/BlobResourceContents/ResourceLink 都编译进 JS world)
- `code-mode/Cargo.toml:13,22,28` — `sandbox = ["v8/v8_enable_sandbox"]` + `deno_core_icudata` + `v8 = { workspace = true }`(用 rusty_v8)
- `code-mode/src/runtime/{callbacks,globals,module_loader,timers,value}.rs` — V8 host 实现 5 个模块
- `v8-poc/src/lib.rs:22` — `rusty_v8` 验证

### Context Compaction
- `core/src/compact.rs:46-47` — 模板 `templates/compact/prompt.md` + `templates/compact/summary_prefix.md`
- `core/src/compact.rs:48` — `const COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000;`
- `core/src/compact.rs:59-63` — `InitialContextInjection::{BeforeLastUserMessage, DoNotInject}`,注释明确"模型训练时见到的位置"决定 injection point
- `core/src/compact.rs:65-67` — `should_use_remote_compact_task(provider)` provider 决定走 remote/local
- `core/src/compact.rs:17-23` — analytics 集成:`CodexCompactionEvent` + 7 个枚举(`CompactionImplementation`/`Phase`/`Reason`/`Status`/`Strategy`/`Trigger` + `now_unix_seconds`)
- 文件大小:`compact.rs:585` / `compact_remote.rs:387` / `compact_remote_v2.rs:456`,**3 套并存**
- `core/src/hook_runtime.rs` — `run_pre_compact_hooks` + `run_post_compact_hooks`(compact 跟 hook 协作)

### Hooks (8 lifecycle)
- `hooks/src/lib.rs:18-27` — `HOOK_EVENT_NAMES: [&str; 8]` = `["PreToolUse", "PermissionRequest", "PostToolUse", "PreCompact", "PostCompact", "SessionStart", "UserPromptSubmit", "Stop"]`
- `hooks/src/lib.rs:34-41` — `HOOK_EVENT_NAMES_WITH_MATCHERS: [&str; 6]`(只有 6 个事件 dispatch matcher,Stop/UserPromptSubmit 不用)
- `hooks/src/lib.rs:43-74` — 公开 API:每个 event 对应 `*Request` + `*Outcome` 结构(`PreToolUse`/`PermissionRequest`/`PostToolUse`/`PreCompact`/`PostCompact`/`SessionStart`/`Stop`/`UserPromptSubmit`)+ `PluginHookDeclaration`
- `hooks/src/lib.rs:91-101` — `hook_key(key_source, event_name, group_index, handler_index)` 唯一化 hook handler
- `hooks/src/{engine,events,registry,output_spill,legacy_notify,schema,types,declarations,config_rules}.rs` — 完整 engine,`output_spill` 防输出爆量,`legacy_notify` 向后兼容

### Goals (一等公民, runtime self-constraint)
- `core/src/goals.rs:1-5` — 注释:"Core support for persisted thread goals... 桥接 core sessions 和 state-db goal table"
- `core/src/goals.rs:18-22` — 5 个 OTEL metric:`GOAL_BUDGET_LIMITED_METRIC` / `GOAL_COMPLETED_METRIC` / `GOAL_CREATED_METRIC` / `GOAL_DURATION_SECONDS_METRIC` / `GOAL_TOKEN_COUNT_METRIC`
- `core/src/goals.rs:46-55` — `SetGoalRequest { objective, status, token_budget }` + `CreateGoalRequest { objective, token_budget }`
- `core/src/goals.rs:57-79` — 3 个 prompt template:`templates/goals/continuation.md` / `templates/goals/budget_limit.md` / `templates/goals/objective_updated.md`
- `otel/src/metrics/names.rs:31-32` — `pub const GOAL_CREATED_METRIC: &str = "codex.goal.created";` + `GOAL_COMPLETED_METRIC = "codex.goal.completed"`(OTLP 标准命名)

### Sandboxing (3 平台)
- `sandboxing/src/{bwrap,landlock,seatbelt,manager,policy_transforms}.rs` — Linux bubblewrap / Linux landlock / macOS seatbelt + manager 调度 + policy_transforms 抽象层
- `sandboxing/src/{seatbelt_base_policy,seatbelt_network_policy,restricted_read_only_platform_defaults}.sbpl` — 3 个 macOS Seatbelt policy 文件
- `windows-sandbox-rs/` — Windows 独立 crate
- `core/src/landlock.rs` + `core/src/windows_sandbox.rs` + `core/src/sandbox_tags.rs` + `core/src/sandboxing/` — core 层的 sandbox wrapper
- AGENTS.md:6-10 — `CODEX_SANDBOX_NETWORK_DISABLED=1` 环境变量 + `CODEX_SANDBOX=seatbelt`(让测试感知沙箱状态而早退)

### External Agent Migration (4 家独家)
- `external-agent-sessions/src/detect.rs:10-11` — `SESSION_IMPORT_MAX_COUNT: usize = 50;` + `SESSION_IMPORT_MAX_AGE: Duration = Duration::from_secs(30 * 24 * 60 * 60);`(30 天)
- `external-agent-sessions/src/ledger.rs:12` — `SESSION_IMPORT_LEDGER_FILE: &str = "external_agent_session_imports.json";`(CODEX_HOME 下 ledger)
- `external-agent-sessions/src/ledger.rs:20-26` — `ImportedExternalAgentSessionRecord { source_path, content_sha256, imported_thread_id, imported_at }`(SHA256 防重复)
- `external-agent-sessions/src/records.rs:14-16` — `NOTE_MAX_LEN: 2_000` + `TOOL_RESULT_MAX_LEN: 4_000`(导入截断)+ `EXTERNAL_AGENT_TOOL_CALL_TAG` / `EXTERNAL_AGENT_TOOL_RESULT_TAG`(导入标签)
- `external-agent-sessions/src/{detect,export,ledger,records,lib}.rs` — 完整 detect/export/ledger 流水线
- `external-agent-migration/` — 配对 crate(更高层迁移逻辑)

### Rollout (Trace / Replay)
- `rollout/src/recorder.rs:1` — 注释:"Persist Codex session rollouts (.jsonl) so sessions can be replayed or inspected later."
- `rollout/src/recorder.rs:31-43` — 引用 `ARCHIVED_SESSIONS_SUBDIR` / `SESSIONS_SUBDIR` + `ThreadListConfig` 分页
- `rollout/src/{list,metadata,recorder,session_index,sqlite_metrics,state_db,policy,config}.rs` — JSONL + SQLite 双轨持久化
- `rollout/src/recorder.rs:50-58` — 自动关联 `GitInfo`(`collect_git_info` / `get_git_repo_root`)
- 路径:`SESSIONS_SUBDIR` 子目录 + `ARCHIVED_SESSIONS_SUBDIR` 子目录

### Memory (3 crate split)
- `memories/read/` + `memories/write/` + `memories/mcp/` — 3 个独立 crate,read 和 write 分开
- `memories/read/src/{citations,prompts,usage,metrics,lib}.rs` — read 侧带 citations + usage tracking + 独立 metrics
- `protocol/src/memory_citation.rs` — memory 引用是**协议级**(citation 跨越 client/server 边界)

### Skills
- `skills/src/lib.rs:1-15` — `SYSTEM_SKILLS_DIR = include_dir!("$CARGO_MANIFEST_DIR/src/assets/samples")`(build-time 嵌入),`SKILLS_DIR_NAME = "skills"`,`SYSTEM_SKILLS_DIR_NAME = ".system"`(CODEX_HOME/skills/.system)
- `skills/src/lib.rs:14` — `SYSTEM_SKILLS_MARKER_FILENAME = ".codex-system-skills.marker"` + `SYSTEM_SKILLS_MARKER_SALT = "v1"` — fingerprint 缓存避免每次启动重装

### Protocol (强类型, RPC)
- `protocol/src/` — `account.rs` / `auth.rs` / `approvals.rs` / `dynamic_tools.rs` / `mcp_approval_meta.rs` / `memory_citation.rs` / `network_policy.rs` / `parse_command.rs` / `permissions.rs` / `plan_tool.rs`(plan 是协议级) / `request_user_input.rs`(人在回路 RPC)/ `tool_name.rs` / `thread_id.rs` / `session_id.rs` / `shell_environment.rs`
- `app-server-protocol/src/` — `experimental_api.rs` / `export.rs` / `jsonrpc_lite.rs` / `protocol/` + `schema_fixtures.rs`
- AGENTS.md:180-220 — App-server v2 命名规范:`*Params/*Response/*Notification` + camelCase wire + `#[serde(rename_all="camelCase")]` + `#[ts(export_to="v2/")]` + `#[experimental("method/or/field")]` 实验性标记 + cursor 分页(`cursor: Option<String>` + `next_cursor: Option<String>`)

### Provider 全拆 crate
- `model-provider/`, `model-provider-info/`, `responses-api-proxy/`, `chatgpt/`, `lmstudio/`, `ollama/`, `keyring-store/`, `login/`,各为独立 crate(对照 SkillForge 的 multi-provider 单点抽象,**Codex 把每个 provider 都当 crate 处理**)

### Channels / 形态
- `tui/`(ratatui)+ `cli/` + `exec/` + `exec-server/`(headless)+ `app-server/` 系列(IDE RPC)+ `cloud-tasks*`(Cloud 形态)+ `realtime-webrtc/`(语音 / 实时)+ `chatgpt/`(ChatGPT 集成)

## 与 wiki 现有记录的关系

### 印证
- 单一职责子 agent(`awaiter` 只轮询不修改)与 [[harness/openclaw]] 梦境日记 subagent 单职责一致 — 印证"专用 subagent + 单职责"是好做法
- depth-tracking via SessionSource chain 与 [[harness/hermes-agent]] `MAX_DEPTH=1` 同思路(都按 spawn 链跟踪)— 但 Codex 默认上限是 **threads 数(6),而不是 depth 数(1)**

### 矛盾
- Hooks 数量:Codex **8 lifecycle** vs Claude Code 27 vs Hermes 16(`VALID_HOOKS`) — **三家的精简程度成阶梯**,需在 [[concept/hooks-design]] 展开
- Multi-agent 模型:Hermes/OpenClaw 完全隔离(无 fork)/ Claude Code SubAgent 隔离 + Fork 继承 / **Codex actor 模型(mailbox + control + resume_agent)**— 三家路线对比 [[concept/multi-agent-isolation]] 需扩

### 新增
- **Code Mode(V8 isolate LLM-writes-JS)** wiki 完全没有 → 新 [[concept/code-mode-orchestration]]
- **External agent migration**(从别的 harness 迁 session)4 家独家 → 新 [[concept/agent-interop]]
- **App-server v1+v2 daemon + Rust derive TS** — wiki 当前没有"daemon / client-server / RPC 工程化"主题
- **Goals + token_budget per goal + 5 OTEL metric** — 一种新的"runtime self-constraint"形态,扩 [[concept/self-evolution]]
- **3 套 compact + provider 路由 + 训练对齐 InitialContextInjection** — 扩 [[concept/context-compaction]]

## 事实漂移 / 注意点

- Codex 的 **`version = "0.0.0-dev"`** 跟 Hermes/Claude Code/OpenClaw 不同 — monorepo 不打 semver,只能用 git commit hash 锁版本(`302149d979` @ 2026-05-15)
- docs/ 都是占位指向 developers.openai.com,**所有信息必须从源码读** — 这反而让 B 模式合规更彻底
- 50 个科学家名只是 spawn 时分配的代号,**不是 50 个预设 role** — 只有 2 个 builtin role TOML(`awaiter` / `explorer`)
- `code-mode/sandbox` feature 启用 `v8_enable_sandbox`,**V8 自己的内置沙箱**,跟 Codex 的 OS 级 sandbox(bwrap/seatbelt)是两件事
