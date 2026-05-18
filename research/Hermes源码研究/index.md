---
created: '2026-04-30'
tags:
- hermes-agent
- 源码阅读
- agent-architecture
status: draft
verified_by:
  - "agent-harness-wiki/source/hermes-agent.md @ 2026-05-15 — terminal backends 由 6 → 7,新增 vercel_sandbox"
---

# Hermes Agent 源码研究笔记

源码路径：`/Users/youren/myspace/hermes-agent/`

参考风格：
- `../research-docs/research/OpenClaw源码研究/`
- `../research-docs/research/claude code 源码/`

## 模块拆分

### 一、项目架构总览
- `README.md` / `AGENTS.md` — 项目能力、开发约束、核心入口说明
- `run_agent.py` — `AIAgent`，主对话循环
- `cli.py` — `HermesCLI`，交互式 CLI 总控
- `gateway/run.py` — 多平台消息网关总控
- `model_tools.py` / `toolsets.py` / `tools/registry.py` — 工具注册、发现、分发

### 二、Agent 主循环
- `run_agent.py` — `AIAgent.__init__()` / `run_conversation()` / `chat()`
- `agent/prompt_builder.py` — system prompt 与上下文装配
- `agent/context_compressor.py` / `trajectory_compressor.py` — 上下文压缩
- `agent/error_classifier.py` / `agent/retry_utils.py` — 错误分类与重试

### 三、模型 Provider 与传输层
- `agent/transports/` — chat completions / Anthropic / Bedrock / Codex transport
- `agent/anthropic_adapter.py` — Anthropic 与 Claude Code OAuth 兼容
- `agent/codex_responses_adapter.py` — Codex Responses API 适配
- `hermes_cli/models.py` / `model_catalog.py` — 模型目录与选择
- `agent/credential_pool.py` / `credential_sources.py` — 凭证池与来源

### 四、工具系统
- `tools/registry.py` — 工具注册表与自动发现判定
- `model_tools.py` — `handle_function_call()`、工具定义生成、调用分发
- `toolsets.py` — toolset 分组与核心工具清单
- `tools/*.py` — 文件、终端、浏览器、MCP、delegate、memory、skills 等工具实现
- `tools/environments/` — local/docker/ssh/modal/daytona/singularity/**vercel_sandbox** 执行环境(共 7 种,verified 2026-05-15)

### 五、CLI 与 Slash Commands
- `cli.py` — 交互式 CLI、命令分发、会话 UI
- `hermes_cli/commands.py` — `COMMAND_REGISTRY`，命令元数据唯一来源
- `agent/skill_commands.py` — skill slash commands 注入
- `hermes_cli/skin_engine.py` / `agent/display.py` — 皮肤、banner、spinner、输出样式

### 六、Session、状态与记忆
- `hermes_state.py` — `SessionDB`，SQLite session store 与 FTS5 搜索
- `agent/memory_manager.py` / `agent/memory_provider.py` — 记忆抽象与生命周期
- `plugins/memory/` — honcho、mem0、supermemory 等记忆插件
- `tools/memory_tool.py` / `tools/session_search_tool.py` — 记忆与历史检索工具

### 七、Gateway 多平台消息系统
- `gateway/run.py` — `GatewayRunner`，消息入口、命令、agent 调度
- `gateway/platforms/base.py` — 平台适配器基类、消息 guard、队列
- `gateway/platforms/` — telegram/discord/slack/wechat/email/api_server 等适配器
- `gateway/session.py` / `session_context.py` — messaging session 语义
- `gateway/stream_consumer.py` / `delivery.py` — 流式输出消费与发送

### 八、TUI 与 Dashboard 嵌入
- `ui-tui/src/` — Ink/React 终端 UI
- `tui_gateway/server.py` — Python JSON-RPC 后端
- `hermes_cli/pty_bridge.py` / `hermes_cli/web_server.py` — dashboard 通过 PTY 嵌入 TUI
- `web/src/pages/ChatPage.tsx` — 浏览器侧 terminal 容器

### 九、ACP、MCP 与外部协议
- `acp_adapter/` — ACP server，IDE 集成
- `mcp_serve.py` — Hermes 作为 MCP server
- `tools/mcp_tool.py` / `mcp_oauth.py` / `mcp_oauth_manager.py` — MCP 工具接入与 OAuth
- `acp_registry/` — ACP registry 支撑

### 十、插件与扩展系统
- `hermes_cli/plugins.py` — `PluginManager`、`PluginContext`、hook/tool/CLI/platform 注册
- `plugins/` — repo-shipped 插件
- `plugins/context_engine/` / `plugins/image_gen/` / `plugins/platforms/` — 专用插件面
- `gateway/hooks.py` / `hermes_cli/hooks.py` — hook 加载与调度入口

### 十一、Skills 系统
- `skills/` — 默认内置 skills
- `optional-skills/` — 可选 heavy/niche skills
- `tools/skills_tool.py` / `tools/skill_manager_tool.py` — skill 查询、安装、管理
- `agent/skill_preprocessing.py` / `agent/skill_utils.py` — skill 加载与预处理
- `hermes_cli/skills_hub.py` / `skills_config.py` — CLI 侧 skill hub 与配置

### 十二、配置、Profile、安装与发布
- `hermes_cli/config.py` — `DEFAULT_CONFIG`、配置加载、环境变量元数据
- `hermes_constants.py` — `get_hermes_home()` / `display_hermes_home()`，profile-aware 路径
- `hermes_cli/main.py` — profile override、argparse、setup/子命令入口
- `hermes_logging.py` — profile-aware 日志
- `scripts/` / `packaging/` / `Dockerfile` / `flake.nix` — 测试、安装、发布

## 笔记索引

| # | 笔记 | 主要内容 | 状态 |
|---|---|---|---|
| 00 | [[00-项目架构总览]] | 顶层目录、入口文件、运行形态、模块关系 | 初稿 |
| 01 | [[01-Agent主循环]] | `AIAgent` 初始化、消息装配、LLM/tool loop、预算与中断 | 待深挖 |
| 02 | [[02-模型Provider与传输层]] | provider 选择、credential、transport、Claude Code OAuth 兼容 | 待深挖 |
| 03 | [[03-工具系统]] | registry、toolsets、schema、dispatch、环境后端 | 待深挖 |
| 04 | [[04-CLI与SlashCommands]] | `HermesCLI`、命令注册、autocomplete、skin/display | 待深挖 |
| 05 | [[05-Session状态与记忆]] | session DB、FTS5、memory manager、memory plugins | 待深挖 |
| 06 | [[06-Gateway多平台消息系统]] | GatewayRunner、平台适配器、消息 guards、流式发送 | 待深挖 |
| 07 | [[07-TUI与Dashboard嵌入]] | Ink TUI、JSON-RPC、PTY bridge、dashboard chat | 待深挖 |
| 08 | [[08-ACP-MCP与外部协议]] | ACP server、MCP server/client、OAuth 管理 | 待深挖 |
| 09 | [[09-插件与扩展系统]] | PluginManager、hook、plugin tool、平台插件 | 待深挖 |
| 10 | [[10-Skills系统]] | built-in/optional skills、skill tools、slash skill commands | 待深挖 |
| 11 | [[11-配置Profile安装发布]] | config.yaml、profile、logging、setup、release | 待深挖 |

## 建议学习顺序

1. [[00-项目架构总览]] — 先建立全局地图。
2. [[01-Agent主循环]] — 理解 Hermes 作为 agent 的核心执行模型。
3. [[03-工具系统]] — 主循环和外部世界交互的关键接口。
4. [[02-模型Provider与传输层]] — 理解模型路由、凭证、重试与 provider 差异。
5. [[05-Session状态与记忆]] — 理解长期状态、历史搜索、记忆插件。
6. [[04-CLI与SlashCommands]] — 理解本地交互入口。
7. [[06-Gateway多平台消息系统]] — 理解 messaging 场景和多平台异步调度。
8. [[07-TUI与Dashboard嵌入]] — 理解新终端 UI 和 dashboard 复用方式。
9. [[08-ACP-MCP与外部协议]] / [[09-插件与扩展系统]] / [[10-Skills系统]] — 扩展面。
10. [[11-配置Profile安装发布]] — 横切配置、profile、安装测试与发布约束。

## 调研方法

每个模块按同一结构推进：

1. 先画职责边界：入口、依赖、输出。
2. 再追一条主链路：用户输入如何流到该模块，模块如何返回结果。
3. 标记关键数据结构和状态对象。
4. 对比 OpenClaw / Claude Code 中相似设计。
5. 总结设计亮点、复杂点、可复用模式、潜在风险。

