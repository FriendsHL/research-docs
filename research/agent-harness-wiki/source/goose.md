---
name: goose
type: source
language: rust
last_ingested_version: 1.34.0
last_ingested_at: 2026-05-18
last_commit: 3cf0bcdec708da8e6ebbb45e470ed0dd48c63a94
mode: B
---

# source/goose — verified file:line 锚点

> 给 [[harness/goose]] 提供 verify-able 锚点。所有事实 grep-able。

## Cargo.toml workspace

- `Cargo.toml:13-17` — workspace.package version 1.34.0, edition 2021, authors = ["AAIF <ai-oss-tools@block.xyz>"], license = "Apache-2.0", repository = "github.com/aaif-goose/goose"
- `Cargo.toml:26` — `rmcp = { version = "1.7.0", features = ["schemars", "auth"] }` — **MCP first-class**
- `Cargo.toml:27-28` — `agent-client-protocol-schema = "0.12"` + `agent-client-protocol = "0.11"` — **ACP secondary**

## 8 crate workspace

- `crates/goose/` — core logic
- `crates/goose-acp-macros/` — ACP proc-macros (295 行)
- `crates/goose-cli/` — CLI entry
- `crates/goose-server/` — backend `goosed` daemon
- `crates/goose-mcp/` — 5 内置 MCP server
- `crates/goose-sdk/` — 程序化 SDK
- `crates/goose-test/` + `goose-test-support/` — 测试基础

## 11 hook events (Open Plugins 标准)

`crates/goose/src/hooks/mod.rs:49-77`:

```rust
pub enum HookEvent {
    PreToolUse,
    PostToolUse,
    PostToolUseFailure,
    SessionStart,
    SessionEnd,
    UserPromptSubmit,
    BeforeReadFile,
    AfterFileEdit,
    BeforeShellExecution,
    AfterShellExecution,
    Stop,
}
```

- `mod.rs:1-2` — _"Lifecycle hooks support, modelled after the Open Plugins hooks specification"_, open-plugins.com/agent-builders/components/hooks
- `mod.rs:5` — `<plugin-root>/hooks/hooks.json` 申明位置
- `mod.rs:21-23` — 仅支持 `type: "command"` action,未知 event / action 类型 _ignored per spec_
- `mod.rs:42` — `const DEFAULT_HOOK_TIMEOUT_SECS: u64 = 30`

## Context management

`crates/goose/src/context_mgmt/mod.rs`:

- `:19` — `pub const DEFAULT_COMPACTION_THRESHOLD: f64 = 0.8` (80% 上下文窗口阈值)
- `:65` — `pub async fn compact_messages(...)` 主入口
- `:200, :432` — `unwrap_or(DEFAULT_COMPACTION_THRESHOLD)` 配置兜底

## Security 6 文件 (业内罕见 prompt-injection + egress 双向防御)

`crates/goose/src/security/mod.rs:1-7`:

```rust
pub mod adversary_inspector;
pub mod classification_client;
pub mod egress_inspector;
pub mod patterns;
pub mod scanner;
pub mod security_inspector;
```

`security/adversary_inspector.rs`:

- `:15` — `const DEFAULT_TOOLS: &[&str] = &["shell", "computercontroller__automation_script"]`
- `:17-26` — `DEFAULT_RULES` BLOCK 5 类:exfiltrate / destructive / malware / privilege escalation / untrusted remote scripts
- `:29` — `const MAX_RECENT_USER_MESSAGES: usize = 4` (LLM 评估上下文窗口)

`security/egress_inspector.rs`:

- `:11` — `pub struct EgressInspector;`
- `:25-30` — `struct EgressDestination { kind: String, destination: String, domain: String }` (regex domain 提取)

`security/mod.rs:39-49`:

```rust
pub struct SecurityManager {
    scanner: OnceLock<PromptInjectionScanner>,
}
pub struct SecurityResult {
    pub is_malicious: bool,
    pub confidence: f32,
    pub explanation: String,
    pub should_ask_user: bool,
    pub finding_id: String,
    pub tool_request_id: String,
}
```

`security/mod.rs:27-37` — `set_security_defaults` 含 `SECURITY_PROMPT_ENABLED` + `SECURITY_COMMAND_CLASSIFIER_ENABLED`

## Permission 5 文件

`crates/goose/src/permission/`:

- `mod.rs` (8 行 re-export)
- `permission_confirmation.rs`
- `permission_inspector.rs`
- `permission_judge.rs` — **LLM 风评估**(judge tool 是否安全执行)
- `permission_store.rs` — 持久化

## Providers 40+ 文件 (业内最广)

`crates/goose/src/providers/`:

- **5 ACP impl 业内独家**:`amp_acp.rs` / `claude_acp.rs` / `codex_acp.rs` / `copilot_acp.rs` / `pi_acp.rs` + `acp_tooling.rs`
- **5 跨家 coding agent provider 业内独家**:`claude_code.rs` / `chatgpt_codex.rs` / `cursor_agent.rs` / `gemini_cli.rs` / `kimicode.rs`
- **15+ 标准 LLM**:`anthropic.rs` / `openai.rs` / `google.rs` / `azure.rs` / `bedrock.rs` / `databricks.rs` / `gemini_cli.rs` / `xai.rs` / `ollama.rs` / `openrouter.rs` / `litellm.rs` / `snowflake.rs` / `sagemaker_tgi.rs` / `tetrate.rs` / `nanogpt.rs` / `gcpvertexai.rs` / `gemini_oauth.rs` / `githubcopilot.rs`
- 5 子模块:`canonical/` + `declarative/` + `inventory/` + `formats/` + `local_inference/`
- `providers/base.rs` — Provider trait(AGENTS.md 内 _"Implement Provider trait see providers/base.rs"_)
- 5 OAuth flow:`oauth_device_flow.rs` + `oauth.rs` + `azureauth.rs` + `gcpauth.rs` + `gemini_oauth.rs`

## ACP proc-macro (业内最完整)

`crates/goose-acp-macros/src/lib.rs:295`:

- `:42-46` — `#[proc_macro_attribute] pub fn custom_methods(...)` — impl 块自动 dispatch
- Doc 注释 (`:7-40`) — 详细说明:
  - 自动生成 `handle_custom_request` dispatcher + `custom_method_schemas`
  - method 名从 `#[request(method = "...")]` 编译期提取,消除重复
  - 支持 0 或 1 参数 handler,return 必须 `Result<T, agent_client_protocol::Error>` 且 `T: Serialize`

## 5 内置 MCP server (MCP-as-extension)

`crates/goose-mcp/src/`:

- `autovisualiser/`
- `computercontroller/`
- `memory/`
- `peekaboo/`
- `tutorial/`
- `lib.rs` + `mcp_server_runner.rs` + `subprocess.rs`

## Agent core

`crates/goose/src/agents/agent.rs:2917`:

- `:17` — `use super::tool_confirmation_router::ToolConfirmationRouter;`
- `:57` — `use crate::utils::is_token_cancelled;`
- `:65` — `use tokio_util::sync::CancellationToken;`
- `:202` — `pub tool_confirmation_router: ToolConfirmationRouter`
- `:313` — `tool_confirmation_router: ToolConfirmationRouter::new()`
- `:669, :883, :985` — `CancellationToken` 贯穿 agent reply + tool_execution

`crates/goose/src/agents/` 21 文件含:
- `subagent_execution_tool/` + `subagent_handler.rs` + `subagent_task_config.rs` — SubAgent 3 文件
- `tool_confirmation_router.rs` / `tool_execution.rs` / `platform_tools.rs` / `platform_extensions/`
- `extension.rs` / `extension_manager.rs` / `extension_malware_check.rs` / `validate_extensions.rs`
- `large_response_handler.rs` / `final_output_tool.rs` / `prompt_manager.rs` / `reply_parts.rs` / `retry.rs`
- `container.rs` / `execute_commands.rs` / `mcp_client.rs` / `moim.rs` / `schedule_tool.rs` / `snapshots/` / `types.rs`

## AGENTS.md 工程纪律

`AGENTS.md:78-86` — Rules 段:
- Test: Prefer tests/ folder
- Test: When adding features, update goose-self-test.yaml, rebuild, then run `goose run --recipe goose-self-test.yaml`
- Error: Use `anyhow::Result`
- Provider: Implement Provider trait see providers/base.rs
- MCP: Extensions in crates/goose-mcp/
- Server: Changes need `just generate-openapi`

`AGENTS.md:99-110` — Ink TUI 7 条 best practice(业内 TUI 工程经验最丰富文档之一)

`AGENTS.md:118-122` — Entry Points:
- CLI: `crates/goose-cli/src/main.rs`
- Server: `crates/goose-server/src/main.rs`
- UI: `ui/desktop/src/main.ts`
- Agent: `crates/goose/src/agents/agent.rs`

## Observability

`crates/goose/src/otel/` + `crates/goose/src/posthog.rs` + `crates/goose/src/logging.rs`
— **PostHog 加进 trace 路线**(产品级 event analytics + OTel infra trace 双层)

## Channels 4 个

1. CLI:`crates/goose-cli/src/main.rs`
2. Electron desktop:`ui/desktop/` + `goose-server` (`goosed` daemon)
3. Ink TUI:`ui/text/` (React-rendered terminal)
4. SDK:`crates/goose-sdk/` (custom_requests.rs + lib.rs)

## Recipe / Scheduler / Skills

- `crates/goose/src/recipe/` + `recipe_deeplink.rs` — declarative workflow spec
- `crates/goose/src/scheduler.rs` + `scheduler_trait.rs` — Cron-style 定时
- `crates/goose/src/skills/` — (待 verify)
- `crates/goose/src/slash_commands.rs` — slash command
- `crates/goose/src/dictation/` — 语音输入

## ACP crate 子结构

`crates/goose/src/acp/`:
- `adapters.rs` / `common.rs` / `fs.rs` / `mcp_app_proxy.rs` / `mod.rs` / `provider.rs`
- `server/` + `server.rs` + `server_factory.rs` / `tools.rs` / `transport/` / `templates/`
