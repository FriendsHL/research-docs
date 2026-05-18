---
name: goose
type: harness
status: active
language: rust
last_ingested_version: 1.34.0
last_ingested_at: 2026-05-18
last_commit: 3cf0bcdec708da8e6ebbb45e470ed0dd48c63a94
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/goose]]"
  - "/Users/youren/myspace/goose/"
  - "github.com/aaif-goose/goose"

# === 13 维 (B 模式 source-grounded) ===

agent_loop: |
  `crates/goose/src/agents/agent.rs:2917` 单文件主 loop. ReplyParts 流式生成 + ToolConfirmationRouter
  (`agents/tool_confirmation_router.rs`) 拦截 tool 调用走 permission/security 双 inspector.
  `CancellationToken` (tokio_util) 贯穿 agent reply + tool_execution (`utils::is_token_cancelled`).
  goose-cli/main.rs 是 CLI entry, goose-server (`goosed` binary) 是 daemon, ui/desktop Electron + ui/text Ink TUI.

tool_protocol: |
  ⭐ **MCP first-class** — workspace dep `rmcp = "1.7.0"` (`Cargo.toml:26`).
  goose-mcp crate 内置 5 个 MCP server: autovisualiser / computercontroller / memory / peekaboo / tutorial
  (`crates/goose-mcp/src/`). **MCP-as-extension 模式** — 自家 framework 把内置 capability 包成 MCP server,业内罕见.
  
  ⭐ **ACP secondary** — workspace dep `agent-client-protocol = "0.11"` + `agent-client-protocol-schema = "0.12"`.
  goose-acp-macros crate (`src/lib.rs:295` 行) 是 proc-macro 自动生成 `handle_custom_request` + `custom_method_schemas`
  — **业内 ACP 实现最完整 + 5 ACP provider 实例** (amp_acp / claude_acp / codex_acp / copilot_acp / pi_acp).

context_mgmt: |
  `crates/goose/src/context_mgmt/mod.rs:810` 行 + `DEFAULT_COMPACTION_THRESHOLD: f64 = 0.8` (`:19`) —
  80% 上下文窗口阈值触发 `check_if_compaction_needed` → `compact_messages`. 
  跟 [[harness/claude-code]] `AUTOCOMPACT_BUFFER_TOKENS=13000` + `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3` 路线不同 —
  goose 走百分比阈值, Claude Code 走绝对 token 数 + retry budget.

memory: |
  `goose-mcp/memory/` 子 crate 作为内置 MCP server — memory 是可挂可拆的 extension, 不是 framework 内置必备.
  跟 [[harness/openhuman]] Memory Tree (17 JSON-RPC + Obsidian 双写) / [[harness/letta]] MemGPT 路线对比:
  goose memory 是 _extension-level capability_, 不是 _framework-level primitive_.

multi_agent: |
  SubAgent via 3 文件 (`agents/subagent_execution_tool/` + `subagent_handler.rs` + `subagent_task_config.rs`).
  没看到 actor model (`tokio mpsc` 看 agent loop 里) 或 ThreadSpawn 链 (vs [[harness/codex]] BFS),
  也没看到 Fork 继承 (vs [[harness/claude-code]] / [[harness/openhuman]]).
  路线归类: 偏 **隔离 fire-and-forget**, 但工具 confirmation 跑 router 模式而非完全独立.

eval_loop: |
  `evals/open-model-gym/` 独立 benchmark 子目录 + `goose-self-test.yaml` recipe-based regression
  (`goose run --recipe goose-self-test.yaml` 跑全 feature regression). 
  AGENTS.md 显式约定 _"When adding features, update goose-self-test.yaml, rebuild, then run goose run --recipe ..."_ —
  **recipe-as-test 跟 [[concept/agent-eval]] 一脉**.

self_evolution: |
  `recipe/` 子模块 + `recipe_deeplink.rs` — recipe 是 declarative workflow spec, 接近 Codex `recipe`.
  `skills/` 子模块存在, 但 verify 程度待深入 (本次 token 紧未读). 
  无 [[deep/claude-code-autodream-3-gate]] 风 background consolidation 证据.

observability: |
  `otel/` crate (跟 [[harness/codex]] `otel/` crate 路线一致) + `posthog.rs` 用户行为分析 + `logging.rs`.
  **PostHog 加进 trace 路线是 wiki 之前没见过的维度** — 跟 [[concept/observability]] Wide Events 2.0 互补:
  PostHog 是 product-level event analytics, OTel 是 infra-level trace.

hooks: |
  ⭐ **11 lifecycle event** (`crates/goose/src/hooks/mod.rs:49-77`):
  - PreToolUse / PostToolUse / **PostToolUseFailure**
  - SessionStart / SessionEnd / UserPromptSubmit / Stop
  - **BeforeReadFile / AfterFileEdit / BeforeShellExecution / AfterShellExecution** (4 个细粒度 file/shell hook)
  
  **跟 [[concept/hooks-design]] 阶梯校准**: [[harness/codex]] 8 < **goose 11** < [[harness/hermes-agent]] 16 < [[harness/claude-code]] 27
  
  ⭐ **Open Plugins 标准对齐** (`mod.rs:1-2` 顶注: _"modelled after the Open Plugins hooks specification"_, open-plugins.com/agent-builders/components/hooks)
  — **业内多家共建标准**, 跟 Claude Code/Codex/Hermes 各自独立设计不同. plugin 在 `<root>/hooks/hooks.json` 申明监听.
  
  - DEFAULT_HOOK_TIMEOUT_SECS = 30 (`mod.rs:42`)
  - 仅支持 `type: "command"` action, 未知 event / action 类型 _ignored per spec_

safety: |
  ⭐ **业内罕见 prompt-injection + egress 双向防御** — `crates/goose/src/security/` 6 文件:
  - `adversary_inspector.rs` — LLM 风 adversary 检测, DEFAULT_TOOLS=["shell", "computercontroller__automation_script"],
    DEFAULT_RULES BLOCK 5 类 (exfiltrate / destructive / malware / privilege escalation / untrusted remote scripts)
  - `egress_inspector.rs` — 数据出向防御 (curl/wget POST 检测, domain 提取 via regex)
  - `classification_client.rs` — command classifier (可由 LLM 后端打分)
  - `patterns.rs` + `scanner.rs` (PromptInjectionScanner) — 模式扫描
  - `security_inspector.rs` — 顶层 SecurityManager + SecurityResult (is_malicious / confidence / explanation / should_ask_user / finding_id)
  
  ⭐ **5 permission 文件** (`permission/`): permission_confirmation / permission_inspector / permission_judge / permission_store
  — judge 跑 LLM 风评估, store 持久化, inspector 跑 ToolInspector trait. 
  跟 [[concept/sandbox-isolation]] 6 阶梯隔离 + [[concept/agent-harness]] permission 维度互补 — **goose safety 是 _inspector-pipeline_ 风, 跟 OS sandbox 互补不重复**.
  
  sandbox: Rust workspace 不直接做 OS sandbox (依赖底层 OS / docker compose), Dockerfile + .devcontainer 给容器化.
  secret_scan / bash_blocklist / path_traversal: ? 未深 verify.

provider: |
  ⭐ **40+ providers 业内最广** (`crates/goose/src/providers/` 40+ 文件):
  
  - 标准 LLM (15+): anthropic / openai / google / azure / bedrock / databricks / gemini / xai / ollama / 
    openrouter / litellm / snowflake / sagemaker_tgi / tetrate / nanogpt / kimicode / vertexai / ...
  - **5 ACP impl 业内独家**: amp_acp / claude_acp / codex_acp / copilot_acp / pi_acp
  - **5 跨家 coding agent provider 业内独家**: claude_code / chatgpt_codex / cursor_agent / gemini_cli / kimicode
    — _"用 goose 调你 Claude Code 当 backend"_ 模式, [[concept/agent-runtime-stacks]] 完全没见过
  - canonical/ + declarative/ + inventory/ + formats/ + local_inference/ 5 子模块 — provider 抽象层
  
  Provider trait `providers/base.rs` (AGENTS.md 内 _"Implement Provider trait see providers/base.rs"_).
  OAuth: `oauth_device_flow.rs` + `oauth.rs` + `azureauth.rs` + `gcpauth.rs` + `gemini_oauth.rs` 5 个 OAuth flow.

channels: |
  4 channel: 
  1. **CLI** (`crates/goose-cli/src/main.rs`)
  2. **Electron desktop** (`ui/desktop/` + `goose-server` `goosed` daemon backend)
  3. **Ink TUI** (`ui/text/`, React-rendered terminal UI, AGENTS.md 内多段 Ink-* rule)
  4. **goose-sdk** (`crates/goose-sdk/`, `custom_requests.rs` + `lib.rs`) — 程序化集成

persistence: |
  `session/` + `session_context.rs` + `scheduler.rs` + `scheduler_trait.rs` (Cron-style 定时 agent run) + 
  `goose_apps/` (推测 app 持久化) + `instance_id.rs`. 
  跟 [[harness/langgraph]] BaseCheckpointSaver SPI 路线不同 — goose 走 session-level 持久化, 没看到 SPI 化的 conformance test.

cancellation: |
  `tokio_util::sync::CancellationToken` 贯穿 agent.rs (`:65, 669, 681, 883, 985`) + 
  `utils::is_token_cancelled` helper. 跟 [[harness/codex]] `AgentControl` 状态机 / [[harness/opencode]] `Question.RejectedError` 路线不同 —
  goose 走 Rust 生态标准 tokio_util CancellationToken (Erlang BEAM cancel 不一样).

# === Takeaway ===

skillforge_takeaway: |
  # 🔴 P1 - 业内罕见
  - **R-GOOSE-2 Security 双向防御** (adversary_inspector + egress_inspector) — SkillForge prompt injection 防御现状待 verify;
    goose 6 文件 security/ + LLM-judged + 配置化规则是参考模板. 2-3 周可起步 (`security_inspector.rs` + `adversary_inspector.rs` 两文件优先).
  
  # 🟡 P2 - 架构启示
  - **R-GOOSE-1 MCP-as-extension 模式** — 自家 framework 把内置 capability 包成 MCP server (goose-mcp 5 个).
    比传统 plugin SDK 更协议化, 自动跨 framework 兼容. SkillForge `Skill` / `Tool` 抽象可考虑包 MCP 暴露.
  - **R-GOOSE-3 11 hook 中间位置** — 4 个细粒度 file/shell hook (BeforeReadFile / AfterFileEdit / BeforeShellExecution / AfterShellExecution)
    介于 Codex 8 通用 / Claude Code 27 细 之间. SkillForge 当前 ~6 hook → 加 4 个 file/shell hook + Open Plugins 标准对齐 (R-GOOSE-6) 是清晰 next step.
  - **R-GOOSE-6 Open Plugins 标准对齐** (hooks 跟 open-plugins.com 共建) — 比 Claude Code / Codex / Hermes 自建强,
    互操作收益. SkillForge hook 系统可对齐, 立刻获得跨 framework plugin 复用.
  
  # 🟢 P3 - 视协议方向
  - **R-GOOSE-4 ACP proc-macro 自动生成 + 5 ACP impl** — 如果 SkillForge 走 ACP 协议路线 (但 [[concept/agent-protocols]] §10 判断 P2),
    goose-acp-macros 是参考实现. 当前 SkillForge 推 MCP/OTel 优先, ACP 不急.
  - **R-GOOSE-5 跨家 coding agent provider** (claude_code / chatgpt_codex / cursor_agent / gemini_cli 作为 LLM provider) —
    Provider 抽象的极限拓展, _"用 SkillForge 调你 Claude Code 当 backend"_ 模式. 商业模式可考虑但工程上 P3.
  
  # ✅ SkillForge 当前覆盖 (验证后填)
  - Hook: 6 lifecycle hook 已有, 跟 goose 11 还差 4 个 file/shell + 1 个 PostToolUseFailure
  - Security: prompt injection 防御现状 ? 未 verify (需查 skillforge-server safety 相关模块)
  - Permission: ? 未 verify
  
  # 待决策
  - SkillForge 要不要加 4 个 file/shell 细粒度 hook + PostToolUseFailure? 取决于 plugin/skill 生态需求.
  - Security 模块 6 文件结构 vs SkillForge SafetyHook 单 hook 路线, 哪个更适合 Java/Spring?

learning_takeaway: |
  # Differentiator (差异)
  - **40+ provider + 5 ACP + 5 跨家 coding agent provider 业内最广** — Block.xyz 把 goose 定位成 _"agent platform 而非 coding agent"_,
    跨 backend 兼容性是核心护城河.
  - **ACP proc-macro** (`goose-acp-macros` 295 行) — proc-macro 自动 dispatch + JSON schema, 业内 ACP 实现 1st 等级.
  - **MCP-as-extension** (goose-mcp 5 内置 server) — 自家 framework dogfood MCP 协议, 这是 _协议优先_ 设计哲学的体现.
  - **Security 双向 (adversary + egress)** — prompt injection + 数据出向防御并重, 比业内只防 prompt injection 多一层.
  - **PostHog 加进 observability** — 跟 OTel trace / Wide Events 2.0 互补的 product-level event analytics.

  # Strengths (强项)
  - **Rust workspace 8 crate 模块边界清晰** — 跟 [[harness/codex]] 11+ crate 同源, 但 goose 更分平台 (CLI / server / mcp / sdk / acp-macros 各独立).
  - **Recipe-as-test** (`goose-self-test.yaml`) — AGENTS.md 显式要求 _"adding features, update goose-self-test.yaml, then run goose run --recipe ..."_,
    eval 即 test 即 documentation 三位一体.
  - **Open Plugins 标准对齐** (hooks) — 跟业界共建标准, 而非自建框架, 工程克制 + 互操作友好.
  - **Ink TUI 详细 best practices** (AGENTS.md 7 条 Ink-* rule) — 业内 TUI 工程经验最丰富的 README 之一, 适合借鉴.

  # What done well (做得好)
  - **AGENTS.md 5KB 工程纪律明确** — Test / Error / Provider / MCP / Server / Code Quality / Ink / Never 各段都有可执行规则.
  - **Provider trait + 40+ impl** — 抽象层经得起 40+ 实现验证, base.rs 是参考样本.
  - **CancellationToken (tokio_util) 贯穿** — 业内 Rust agent 标准 cancel 模式, 不重造轮子.
  - **`.codex` / `.cursor` / `.claude` 都有目录** — 多 LLM 工具 dogfood (跟 [[concept/agent-runtime-stacks]] §`.agents/` 判断 7 印证).
---

# Goose — Block.xyz Rust Agent Platform

> Block (Square 母公司) 出品的 Rust agent framework. 1.34.0 @ 3cf0bcd (2026-05-15).
> 8 crate workspace + Electron desktop + Ink TUI + 5 内置 MCP server + 40+ provider (含 5 ACP + 5 跨家 coding agent provider).
> **业内最广 provider 兼容 + 业内最完整 ACP 实现 + 业内罕见 security 双向防御**.

## 一句话定位

**协议优先的 Rust agent platform** — MCP-as-extension(自家 capability 包成 MCP server)+ ACP proc-macro 自动生成 + Open Plugins hook 标准对齐 + 40+ provider(含 claude_code / chatgpt_codex / cursor_agent / gemini_cli 作 backend)。

## 起源与定位

- **Block / Square 出品**(`aaif-goose/goose`,AAIF = Block AI Open Source Tools team)
- Apache-2.0,Rust workspace 8 crate + Electron desktop + Ink TUI
- 跟 [[harness/codex]] 是业内 2 个 _Rust agent runtime_ — 但定位不同:Codex 偏 coding agent + 跑 V8 isolate,goose 偏 _跨 backend 兼容_ + 协议优先
- 跟 [[harness/claude-code]] 是业内 2 个 _desktop + CLI 双形态_ — 但 goose 开源,Claude Code 闭源

## 核心架构(8 crate workspace)

```
crates/
├── goose              # core logic (agents/ + context_mgmt/ + hooks/ + security/ + permission/ + providers/ + ...)
├── goose-acp-macros   # ACP proc-macro (295 行,业内 ACP 实现最完整)
├── goose-cli          # CLI entry (main.rs)
├── goose-server       # backend daemon `goosed` binary (Electron 用)
├── goose-mcp          # 5 内置 MCP server (autovisualiser / computercontroller / memory / peekaboo / tutorial)
├── goose-sdk          # 程序化 SDK
├── goose-test
└── goose-test-support

evals/open-model-gym/  # benchmark
ui/desktop/            # Electron app
ui/text/               # Ink TUI (React-rendered terminal)
```

## 13 维度详解

详见 frontmatter,以下为关键差异化 highlight:

### Hook 11 = 中间位置(刷新 [[concept/hooks-design]] 阶梯)

```
Codex 8 (lifecycle)   <  goose 11 (Open Plugins)  <  Hermes 16 (Python)  <  Claude Code 27 (zod)
   精简                       中等 + 4 细粒度 file/shell             细粒度 framework
```

goose 11 hook = 7 通用 + 4 个细粒度 **file/shell** hook(BeforeReadFile / AfterFileEdit / BeforeShellExecution / AfterShellExecution)— 业内独家在 _读文件 / 改文件 / 执行 shell_ 这三个 _高风险动作_ 上前后包夹,适合 security inspector 介入。

### Security 双向防御(业内罕见)

`crates/goose/src/security/` 6 文件:
- `adversary_inspector.rs` — LLM 风 adversary 检测(BLOCK 5 类:exfiltrate / destructive / malware / privilege / untrusted remote)
- `egress_inspector.rs` — 数据出向防御(regex domain 提取 + EgressDestination 结构)
- `classification_client.rs` — command classifier
- `patterns.rs` + `scanner.rs` (`PromptInjectionScanner`)
- `security_inspector.rs` — 顶层 `SecurityManager` + `SecurityResult` (is_malicious / confidence / explanation / should_ask_user)

**业内只防 prompt injection,goose 加 egress 防御是新维度**。

### ACP 实现示范(业内最完整)

- `Cargo.toml:27-28` workspace dep:`agent-client-protocol = "0.11"` + `agent-client-protocol-schema = "0.12"`
- `goose-acp-macros/src/lib.rs:295` 行 proc-macro 自动生成:
  - `handle_custom_request` dispatcher(自动 method match + JSON params 反序列化)
  - `custom_method_schemas` JSON Schema 自动生成(implements `schemars::JsonSchema`)
- 5 ACP provider 实例:`amp_acp / claude_acp / codex_acp / copilot_acp / pi_acp`
- ACP 通过 [[concept/agent-protocols]] §A2A 段已 merge 进 A2A under Linux Foundation,但 goose 是 _ACP 协议原本目标的最佳示范_

### MCP-as-extension(协议优先 dogfood)

`goose-mcp` crate 5 个内置 MCP server:
- `autovisualiser` — 自动可视化
- `computercontroller` — computer control(类 Anthropic Computer Use)
- `memory` — memory MCP server
- `peekaboo` — (推测 secret 检测?待 verify)
- `tutorial` — onboarding 教程

**自家 capability 包成 MCP server,不是用 plugin SDK** — _协议优先_ 的设计哲学体现,业内罕见。

## 与 SkillForge 的对照

### 🔴 P1 必做
- **R-GOOSE-2** Security 双向防御(2-3 周)

### 🟡 P2 架构借鉴
- **R-GOOSE-1** MCP-as-extension 模式
- **R-GOOSE-3** 加 4 个细粒度 file/shell hook + PostToolUseFailure
- **R-GOOSE-6** Open Plugins 标准对齐

### 🟢 P3 视协议方向
- **R-GOOSE-4** ACP proc-macro(SkillForge 协议优先级 P2 见 [[concept/agent-protocols]])
- **R-GOOSE-5** 跨家 coding agent provider(商业策略而非工程问题)

详见 frontmatter `skillforge_takeaway` 段。

## 待回答的问题

- `skills/` 子模块具体结构(本次未深 verify)
- `peekaboo` MCP server 实际功能(secret 检测?)
- `recipe/` 跟 [[harness/codex]] `recipe` 异同(declarative workflow spec 路线)
- `permission_judge` LLM 风评估具体 prompt + 跟 `adversary_inspector` 关系
- agent.rs 2917 行单文件主 loop 内部分段(本次 token 紧未全读)
