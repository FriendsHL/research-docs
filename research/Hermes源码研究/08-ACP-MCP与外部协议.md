---
created: '2026-04-30'
tags:
- hermes-agent
- acp
- mcp
status: draft
---

# ACP、MCP 与外部协议源码解析

## 1. 先说结论

这个模块容易混，因为 ACP 和 MCP 都是“外部协议”。

但它们在 Hermes 里做的事情不一样。

可以先分成三条线：

```text
ACP
  让 VS Code / Zed / JetBrains 这类编辑器把 Hermes 当成一个 coding agent 用。

MCP Server
  让 Claude Code / Cursor / Codex 等 MCP client 反过来调用 Hermes 的消息桥能力。

MCP Client
  让 Hermes 自己去连接外部 MCP server，把对方的工具注册成 Hermes 工具。
```

一句话：

```text
ACP = 编辑器进来用 Hermes。
MCP Server = 外部 agent 进来用 Hermes 的消息/会话能力。
MCP Client = Hermes 出去用外部工具。
```

这三个方向不要混。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `acp_adapter/entry.py` | ACP stdio 入口，加载 env，启动 ACP agent |
| `acp_adapter/server.py` | `HermesACPAgent`，ACP 协议方法实现 |
| `acp_adapter/session.py` | ACP session 到 AIAgent 的映射和持久化 |
| `acp_adapter/events.py` | 把 AIAgent callback 转成 ACP session update |
| `acp_adapter/permissions.py` | 把 Hermes 审批请求转成 ACP permission request |
| `acp_adapter/tools.py` | 把 Hermes tool call 转成 ACP tool call 内容 |
| `mcp_serve.py` | Hermes 作为 MCP server 暴露消息桥工具 |
| `tools/mcp_tool.py` | Hermes 作为 MCP client 连接外部 MCP server |
| `tools/mcp_oauth.py` | MCP OAuth token 存储、浏览器授权回调 |
| `tools/mcp_oauth_manager.py` | MCP OAuth provider 缓存、401 去重、磁盘 token reload |
| `hermes_cli/mcp_config.py` | `hermes mcp add/remove/list/test/configure/login` |
| `toolsets.py` | `hermes-acp` 工具集定义 |
| `tools/registry.py` | MCP 动态工具注册和冲突保护 |

## 3. 总体结构图

```text
编辑器
  -> ACP stdio
  -> acp_adapter.server.HermesACPAgent
  -> SessionManager
  -> AIAgent
  -> Hermes tools

外部 MCP client
  -> hermes mcp serve
  -> mcp_serve.py FastMCP
  -> SessionDB / gateway sessions / send_message_tool

Hermes Agent
  -> tools/mcp_tool.py
  -> 外部 MCP server
  -> 注册成 mcp-xxx toolset
  -> 模型像调用普通工具一样调用 MCP 工具
```

## 4. ACP 是什么

ACP 这里可以理解为“编辑器 agent 协议”。

编辑器希望做到：

- 新建一个 agent session
- 发 prompt
- 流式收到 agent 回复
- 看到工具调用开始和结束
- 看到 thinking
- 处理权限审批
- 取消当前任务
- 列出历史 session
- 恢复历史 session
- 切换模型

Hermes 通过 `acp_adapter` 把这些能力接到自己的 `AIAgent`。

## 5. ACP 启动流程

入口有几种：

```bash
hermes acp
python -m acp_adapter.entry
hermes-acp
```

`acp_adapter/entry.py` 做几件事：

```text
1. 设置 logging 到 stderr。
2. 加载 HERMES_HOME/.env。
3. 把项目根目录放进 sys.path。
4. 启动前 discover_mcp_tools()。
5. 创建 HermesACPAgent。
6. 调用 acp.run_agent(agent, use_unstable_protocol=True)。
```

为什么日志要进 stderr？

因为 ACP 和 TUI 一样：

```text
stdout 是协议通道。
stderr 才能写日志。
```

如果 stdout 里混入普通日志，编辑器收到的 JSON-RPC 就会坏。

## 6. ACP initialize / authenticate

编辑器连接后会先 `initialize`。

`HermesACPAgent.initialize()` 会返回：

- protocol version
- agent 名称和版本
- 支持 load session
- 支持 list / resume / fork
- 当前 provider 的认证方式

大概意思是：

```text
我是 hermes-agent。
我支持 session 管理。
我可以用当前 Hermes 配置里的 provider 凭证。
```

`authenticate()` 比较简单。

它只接受自己在 initialize 里声明过的 provider。

比如当前 provider 是 `openrouter`：

```text
authenticate("openrouter") -> OK
authenticate("unknown") -> None
```

## 7. ACP session 怎么映射到 Hermes

核心类是 `SessionManager`。

它维护：

```text
ACP session_id
  -> SessionState
       -> AIAgent
       -> cwd
       -> model
       -> history
       -> cancel_event
```

`SessionState` 里最关键的是：

```python
SessionState(
    session_id="uuid",
    agent=AIAgent(...),
    cwd="/workspace/project",
    model="...",
    history=[],
    cancel_event=threading.Event(),
)
```

ACP session 不是临时内存。

它会持久化到 `SessionDB`，source 是：

```text
acp
```

这样编辑器断开、ACP 进程重启后，`load_session` / `resume_session` 可以从 DB 恢复历史。

## 8. 新建 ACP session 的例子

编辑器请求：

```text
new_session(cwd="/Users/youren/myspace/hermes-agent")
```

流程：

```text
HermesACPAgent.new_session
  -> SessionManager.create_session(cwd)
  -> 生成 uuid session_id
  -> _make_agent(session_id, cwd)
  -> _register_task_cwd(session_id, cwd)
  -> 持久化到 SessionDB
  -> 返回 NewSessionResponse(session_id, models)
```

这里 `_register_task_cwd` 很重要。

它会把 ACP session 的工作目录绑定给 terminal 工具。

否则编辑器里打开的是项目 A，但 agent 执行命令可能跑到别的目录。

## 9. ACP 里的 AIAgent 怎么创建

`SessionManager._make_agent()` 创建 `AIAgent`。

关键参数：

```text
platform="acp"
enabled_toolsets=["hermes-acp", ...mcp toolsets]
quiet_mode=True
session_id=<ACP session id>
model=<配置里的默认模型或用户选的模型>
provider/api_mode/base_url/api_key=<runtime provider 解析结果>
```

ACP 默认工具集是：

```text
hermes-acp
```

它在 `toolsets.py` 里定义。

和完整 CLI 工具相比，ACP 更偏 coding：

- 有 terminal / process
- 有 read_file / write_file / patch / search_files
- 有 browser
- 有 todo / memory
- 有 session_search
- 有 execute_code / delegate_task

但少了 messaging、audio、clarify UI 这类不适合编辑器场景的工具。

## 10. ACP prompt 完整例子

假设用户在编辑器 agent 面板里输入：

```text
帮我修一下这个测试失败
```

ACP 流程大概是：

```text
1. 编辑器调用 prompt(session_id, prompt blocks)
2. HermesACPAgent.prompt(...)
3. 从 ACP content blocks 提取纯文本
4. 如果是已知 slash command，本地处理
5. 准备 tool/thinking/message/approval callbacks
6. 把 callbacks 挂到 state.agent
7. 在线程池里调用 agent.run_conversation(...)
8. agent 中途调用工具时，callback 发 ACP session_update
9. agent 输出文本时，message callback 发 agent_message_chunk
10. 完成后更新 state.history
11. save_session 写回 SessionDB
12. 返回 PromptResponse(stop_reason, usage)
```

更短地看：

```text
ACP prompt
  -> AIAgent.run_conversation
  -> callbacks
  -> ACP session_update
  -> editor UI
```

## 11. 为什么 AIAgent 要在线程池里跑

`AIAgent.run_conversation()` 是同步函数。

但 ACP server 是 async。

所以 `server.py` 用：

```text
ThreadPoolExecutor(max_workers=4)
```

执行 agent。

这样：

```text
ACP event loop
  继续处理协议请求

worker thread
  跑同步 AIAgent 主循环
```

否则一个长任务会卡住整个 ACP 连接。

## 12. ACP callback 怎么变成编辑器 UI

`acp_adapter/events.py` 里有几个 callback factory。

| Hermes callback | ACP update |
|---|---|
| `message_callback` | `agent_message_chunk` |
| `thinking_callback` | agent thought text |
| `tool_progress_callback` | tool call start |
| `step_callback` | tool call complete |

### 12.1 tool start

模型要调用工具时，Hermes 触发：

```text
tool_progress_callback("tool.started", name, preview, args)
```

ACP adapter 做：

```text
1. 生成 tool_call_id。
2. 按工具名放进 FIFO queue。
3. build_tool_start(...)
4. conn.session_update(session_id, update)
```

为什么要 FIFO queue？

因为可能并行调用多个同名工具。

例如模型一次发出两个 `read_file`：

```text
read_file("a.py")
read_file("b.py")
```

完成时只知道工具名和结果，adapter 要按开始顺序把完成事件配回正确的 ACP tool_call_id。

### 12.2 tool complete

`step_callback(api_call_count, prev_tools)` 会收到上一轮完成的工具信息。

它会：

```text
1. 找 tool_name。
2. 从 tool_call_ids[tool_name] 队列里 pop 一个 id。
3. build_tool_complete(...)
4. 发 ACP session_update。
```

### 12.3 文件编辑工具的特殊处理

如果工具是：

```text
write_file
patch
skill_manage
```

ACP adapter 会尝试捕获本地编辑前后的快照。

这样编辑器可以显示更结构化的 diff，而不只是纯文本结果。

## 13. ACP 权限审批怎么做

Hermes 里危险操作会走 approval。

ACP 需要把这个 approval 变成编辑器里的权限弹窗。

入口在 `acp_adapter/permissions.py`。

它生成一个 Hermes 兼容的 callback：

```python
approval_callback(command, description) -> "once" | "always" | "deny"
```

内部会调用 ACP client 的：

```text
conn.request_permission(...)
```

给编辑器的选项是：

```text
Allow once
Allow always
Deny
```

映射到 Hermes：

```text
allow_once   -> once
allow_always -> always
deny         -> deny
```

如果超时或报错：

```text
默认 deny
```

这是安全默认值。

## 14. approval callback 为什么要在线程里设置

`server.py` 里有一段注释很关键。

approval callback 是 thread-local。

所以不能在 async event loop 线程里设置。

必须在真正跑 `agent.run_conversation()` 的 executor thread 里设置。

代码逻辑大概是：

```text
_run_agent() 在线程池里执行
  -> set_session_vars(session_key=session_id)
  -> terminal_tool.set_approval_callback(approval_cb)
  -> os.environ["HERMES_INTERACTIVE"] = "1"
  -> agent.run_conversation(...)
  -> finally 恢复 callback/env/session vars
```

这里 `HERMES_INTERACTIVE=1` 的作用是：

```text
告诉 approval.py：这是交互环境，有 callback，不要走非交互自动分支。
```

## 15. ACP slash command

ACP adapter 自己实现了一小组 headless slash command。

包括：

```text
/help
/model
/tools
/context
/reset
/compact
/version
```

这些不是 classic CLI 的完整 slash command 系统。

它们是 ACP 场景里的轻量命令。

例子：

```text
/context
```

会返回：

```text
Conversation: 12 messages
  user: 4, assistant: 4, tool: 4, system: 0
Model: ...
```

`/compact` 会调用 agent 的压缩能力，但有一个 ACP 特殊处理：

```text
ACP session_id 要稳定。
```

所以压缩时会临时把 agent 的 `_session_db` 设成 `None`，避免触发普通压缩里的 SQLite session 分裂副作用。

压完后直接替换：

```text
state.history = compressed
save_session(state.session_id)
```

## 16. ACP 恢复历史为什么要 replay

`load_session` / `resume_session` 不只是服务端恢复 history。

它还要把历史消息重新发给编辑器。

否则会出现：

```text
Hermes 后端记得上下文
编辑器 UI 看起来是空白线程
```

所以 `_replay_session_history()` 会遍历历史里的 user/assistant message，发：

```text
user_message_chunk
agent_message_chunk
```

工具消息通常不 replay。

这里主要是恢复聊天可读内容。

## 17. ACP 里动态注册 MCP server

ACP 的 `new_session/load_session/resume_session/fork_session` 都支持传：

```text
mcp_servers
```

这些是编辑器侧给当前 session 的 MCP server。

`_register_session_mcp_servers()` 会：

```text
1. 把 ACP 的 McpServerStdio / McpServerHttp / McpServerSse 转成 Hermes config map。
2. register_mcp_servers(config_map)。
3. 给当前 agent 增加 enabled_toolsets：mcp-<server_name>。
4. 重新 get_tool_definitions(...)。
5. 更新 agent.valid_tool_names。
6. invalidate system prompt。
```

这说明 ACP session 可以临时带外部 MCP 工具进来。

但这会改变工具 surface，所以要 invalidate system prompt。

## 18. MCP Server：Hermes 暴露给外部 agent

`mcp_serve.py` 是 Hermes 自己作为 MCP server。

启动：

```bash
hermes mcp serve
```

外部 MCP client 可以这样配置：

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

它用 FastMCP 暴露一组工具。

这个方向是：

```text
Claude Code / Cursor / Codex
  -> MCP
  -> Hermes
  -> Gateway conversations / SessionDB / send_message
```

它不是让 Hermes 调外部工具。

它是让外部 agent 调 Hermes。

## 19. MCP Server 暴露哪些工具

`mcp_serve.py` 里暴露的工具：

| 工具 | 作用 |
|---|---|
| `conversations_list` | 列出 Hermes 已知平台会话 |
| `conversation_get` | 获取某个 conversation 的详情 |
| `messages_read` | 读取某个会话最近消息 |
| `attachments_fetch` | 读取某条消息的附件信息 |
| `events_poll` | 拉取新事件 |
| `events_wait` | 长轮询等新事件 |
| `messages_send` | 给平台 conversation 发消息 |
| `channels_list` | 列出可发送的 channel / target |
| `permissions_list_open` | 列出当前未处理审批 |
| `permissions_respond` | 回复审批 |

代码注释里提到它匹配 OpenClaw 的 9-tool channel bridge surface，额外加了 Hermes 的 `channels_list`。

## 20. MCP Server 的数据从哪里来

它不直接启动 Gateway。

它读的是已有状态：

```text
HERMES_HOME/sessions/sessions.json
  Gateway session_key -> session_id / platform routing 信息

SessionDB
  具体消息历史

HERMES_HOME/channel_directory.json
  可发送 channel 缓存
```

发送消息时走：

```text
tools.send_message_tool.send_message_tool
```

所以 MCP server 更像一个“桥”：

```text
读取已有 Hermes 会话状态
调用已有 Hermes 发送消息工具
```

## 21. MCP Server 事件怎么做

`mcp_serve.py` 里有 `EventBridge`。

它启动一个后台线程，轮询 SessionDB 和 sessions index。

```text
EventBridge
  -> 定时 poll state.db / sessions.json
  -> 发现新消息
  -> 放进内存 queue
  -> events_poll / events_wait 读取 queue
```

为什么是轮询？

因为这个 MCP server 是 stdio 进程，不一定和 Gateway 在同一个进程里。

它不能直接拿 Gateway 内部内存事件。

所以采用简单稳妥的方式：

```text
看数据库和文件有没有变化。
```

## 22. MCP Server 使用例子

假设 Claude Code 连接了 Hermes MCP server。

用户问：

```text
帮我看看 Slack engineering 频道最近有什么消息，然后回复一句收到
```

Claude Code 可能调用：

```text
channels_list(platform="slack")
  -> 找到 slack:#engineering

conversations_list(platform="slack", search="engineering")
  -> 找到 session_key

messages_read(session_key, limit=20)
  -> 读最近消息

messages_send(target="slack:#engineering", message="收到")
  -> 通过 Hermes gateway 发出去
```

也就是说外部 agent 通过 MCP server 可以操作 Hermes 连接的消息平台。

## 23. MCP Client：Hermes 使用外部 MCP 工具

另一条线是 `tools/mcp_tool.py`。

这个方向是：

```text
Hermes Agent
  -> 连接外部 MCP server
  -> 发现工具
  -> 注册到 tools.registry
  -> 模型调用这些工具
```

配置在 `config.yaml`：

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    env: {}
    timeout: 120
    connect_timeout: 60

  remote_api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer ..."
```

支持两类 transport：

```text
stdio
  command + args

HTTP / Streamable HTTP
  url + headers / oauth
```

## 24. MCP Client 启动发现流程

发现入口：

```python
discover_mcp_tools()
```

大概流程：

```text
1. 读取 config.yaml 里的 mcp_servers。
2. 过滤 enabled:false。
3. 启动 MCP 后台 event loop。
4. 并行连接所有新 server。
5. 每个 server list_tools。
6. 把工具 schema 转成 OpenAI-style function schema。
7. 注册到 tools.registry。
8. 返回所有 MCP tool names。
```

连接是并行的。

如果配置了 5 个 MCP server，不是一个一个等。

代码里用：

```text
asyncio.gather(...)
```

并且每个 server 有自己的 `connect_timeout`。

## 25. MCP 工具怎么命名

外部 MCP server 里的工具名可能和 Hermes 内置工具重名。

所以 Hermes 会加前缀。

比如：

```text
server name = github
MCP tool name = search_repos
```

注册到 Hermes 后可能是：

```text
github_search_repos
```

工具集是：

```text
mcp-github
```

同时 registry 还会注册 alias：

```text
github -> mcp-github
```

这样用户启用 toolset 时可以更自然。

## 26. MCP 工具注册到 registry

`_register_server_tools()` 会遍历 server 的 `_tools`。

对每个 tool 做：

```text
1. include/exclude 过滤。
2. 扫描 tool description 是否有 prompt injection 味道。
3. 转 schema。
4. 检查是否撞到内置工具。
5. registry.register(...)
```

include/exclude 配置例子：

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "..."]
    tools:
      include: ["search_repos", "create_issue"]
```

如果 include 有值：

```text
只注册 include 里的工具。
```

如果 exclude 有值：

```text
注册除了 exclude 之外的工具。
```

include 优先级更高。

## 27. MCP 工具调用例子

假设模型看到工具：

```text
github_search_repos
```

它调用：

```json
{
  "name": "github_search_repos",
  "arguments": {
    "query": "hermes agent"
  }
}
```

Hermes 执行 handler：

```text
_make_tool_handler(server_name="github", tool_name="search_repos")
```

handler 做：

```text
1. 检查 circuit breaker。
2. 找到已连接的 MCPServerTask。
3. 在 MCP 后台 loop 上执行 server.session.call_tool(...)。
4. 把 MCP result.content 变成 JSON 字符串。
5. 如果有 structuredContent，也一起返回。
6. 如果错误，清洗敏感信息后返回 error。
```

返回给模型的格式仍然是普通工具结果：

```json
{
  "result": "...",
  "structuredContent": {...}
}
```

所以 agent 主循环不需要知道这是 MCP 工具。

它只看到：

```text
一个 registry 里的普通工具。
```

## 28. MCP subprocess 环境为什么要过滤

stdio MCP server 是 Hermes 启动的子进程。

如果直接继承当前进程 env，可能把大量 API key 泄露给第三方 MCP server。

所以 `_build_safe_env()` 只传：

```text
PATH
HOME
USER
LANG
LC_ALL
TERM
SHELL
TMPDIR
XDG_*
用户在 mcp_servers.<name>.env 显式配置的变量
```

这点很重要。

MCP server 如果需要 GitHub token，应该显式写：

```yaml
env:
  GITHUB_PERSONAL_ACCESS_TOKEN: "..."
```

而不是默认把所有环境变量都给它。

## 29. MCP 错误为什么要脱敏

`tools/mcp_tool.py` 里有 `_sanitize_error()`。

它会替换：

- GitHub PAT
- OpenAI-style key
- Bearer token
- `token=...`
- `key=...`
- `API_KEY=...`
- `password=...`
- `secret=...`

变成：

```text
[REDACTED]
```

原因是 MCP tool error 会回到模型上下文。

如果错误里带 token，就等于把密钥塞进模型上下文。

## 30. MCP description prompt injection 扫描

MCP tool description 是外部 server 给的。

它也会进入模型可见的 tool schema。

所以 Hermes 会扫一些危险模式：

```text
ignore previous instructions
you are now a ...
system:
<system>
do not reveal
curl http://...
exec(...)
eval(...)
import subprocess
```

注意：这里是 warning，不是直接 block。

原因是误报会破坏合法 MCP server。

但日志里会留下线索，方便排查。

## 31. MCP 工具动态刷新

MCP server 可能发：

```text
notifications/tools/list_changed
```

Hermes registry 支持动态变化：

```text
deregister old tools
register new tools
generation +1
```

`tools/registry.py` 里有 generation counter。

它的作用是：

```text
工具 surface 变了，缓存的 tool definitions 不能继续用。
```

同时 registry 允许：

```text
MCP 工具覆盖 MCP 工具
```

但不允许：

```text
MCP / plugin 覆盖内置工具
```

这可以防止外部 server 抢内置工具名。

## 32. MCP 连接失败和 circuit breaker

如果 MCP server 连不上，Hermes 不应该让模型无限重试。

所以 handler 里有 circuit breaker。

大概行为：

```text
连续失败达到阈值
  -> breaker 打开
  -> 一段 cooldown 内直接返回“server unreachable，不要重试”
  -> cooldown 后允许一次半开探测
  -> 成功则重置
  -> 失败则重新打开
```

这对 agent loop 很重要。

否则模型可能连续几十轮都在调用同一个坏掉的 MCP 工具。

## 33. MCP OAuth 的问题

有些远程 MCP server 不用静态 bearer token，而是 OAuth。

Hermes 支持：

```yaml
mcp_servers:
  betterstack:
    url: "https://mcp.example.com/mcp"
    auth: "oauth"
    oauth:
      timeout: 300
```

OAuth 的难点有三个：

```text
1. token 要持久化。
2. 其他进程可能刷新 token。
3. 很多并发工具调用可能同时遇到 401。
```

所以有 `tools/mcp_oauth_manager.py`。

## 34. MCP OAuth token 存哪里

token 存在：

```text
HERMES_HOME/mcp-tokens/
```

每个 server 会有类似：

```text
<server_name>.json
<server_name>.client.json
```

这保证 profile 隔离。

不同 profile 的 MCP OAuth token 不会混用。

## 35. OAuth manager 做了什么

`MCPOAuthManager` 是进程级单例。

它维护：

```text
server_name
  -> provider
  -> server_url
  -> oauth_config
  -> last_mtime_ns
  -> pending_401
```

它解决三个问题。

### 35.1 provider 缓存

同一个 server 重复调用：

```text
get_or_build_provider(server_name, server_url, oauth_config)
```

会复用同一个 OAuth provider。

如果 URL 变了，就丢掉旧 provider，重新建。

### 35.2 磁盘 token reload

每次 auth flow 前，会检查 token 文件 mtime。

如果磁盘上的 token 被别的进程刷新了：

```text
mtime 变了
  -> provider._initialized = False
  -> 下次请求重新从 disk 读 token
```

例子：

```text
Hermes TUI 正在跑
另一个终端执行 hermes mcp login betterstack
token 文件更新
TUI 里的 MCP tool 下次调用前发现 mtime 变化
自动 reload token
```

不用重启 TUI。

### 35.3 401 去重

如果 10 个工具调用同时拿旧 token 请求，可能同时收到 401。

不能让 10 个请求同时刷新 OAuth。

manager 用：

```text
pending_401[failed_access_token] = future
```

第一个请求负责处理。

后面 9 个等同一个 future。

处理完一起知道是否可以 retry。

这就是 401 dedup。

## 36. `hermes mcp` CLI 管理

`hermes_cli/mcp_config.py` 实现：

```text
hermes mcp add
hermes mcp remove
hermes mcp list
hermes mcp test
hermes mcp configure
hermes mcp login
hermes mcp serve
```

`add` 会把 server 写进：

```text
HERMES_HOME/config.yaml
```

大概：

```yaml
mcp_servers:
  myserver:
    url: "https://example.com/mcp"
```

或者：

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    env: {}
```

`test` 会临时连接 server，列出工具，然后断开。

## 37. `/reload-mcp` 和 prompt caching

Classic CLI 有 `/reload-mcp`。

它会：

```text
shutdown_mcp_servers()
discover_mcp_tools()
重新生成工具 surface
```

这会影响 prompt caching。

因为工具 schema 是 system prompt / model request 的稳定部分之一。

如果会话中途工具变了，缓存就可能失效。

所以 CLI 的 `/reload-mcp` 会提示：

```text
Prompt cache invalidation warning
```

这和前面 agent loop 文档里说的原则一致：

```text
不要随便在会话中途改变工具集。
```

## 38. 一个完整例子：编辑器用 ACP 修代码

用户在 Zed 里打开项目，点 Hermes agent。

流程：

```text
1. Zed 启动 hermes acp。
2. acp_adapter.entry 加载 env，启动 HermesACPAgent。
3. Zed initialize。
4. Zed new_session(cwd=当前项目)。
5. SessionManager 创建 AIAgent(platform="acp")。
6. 用户输入“修复这个测试失败”。
7. Zed prompt(...)。
8. Hermes 在线程池里跑 agent.run_conversation。
9. agent 调用 search_files/read_file/patch。
10. events.py 把工具开始/完成转成 ACP updates。
11. Zed UI 显示工具调用和 diff。
12. agent 返回最终回答。
13. SessionManager 保存 history 到 SessionDB。
```

这条线里 Hermes 是：

```text
被编辑器调用的 agent 后端。
```

## 39. 一个完整例子：Hermes 调外部 MCP filesystem

用户配置：

```yaml
mcp_servers:
  fs:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
```

启动 Hermes 后：

```text
1. discover_mcp_tools() 读 config。
2. 连接 fs server。
3. list_tools。
4. 注册 mcp-fs toolset。
5. 模型看到 fs_read_file / fs_list_directory 等工具。
6. 模型调用 fs_list_directory。
7. handler 通过 MCP session.call_tool 发送给外部 server。
8. 外部 server 返回结果。
9. Hermes 包装成 JSON tool result。
10. agent 继续下一轮。
```

这条线里 Hermes 是：

```text
MCP client。
```

## 40. 一个完整例子：Claude Code 调 Hermes 消息桥

用户在 Claude Code 里配置：

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

Claude Code 调：

```text
conversations_list(platform="telegram")
messages_read(session_key="agent:main:telegram:dm:123")
messages_send(target="telegram:123", message="我稍后回复")
```

这条线里 Hermes 是：

```text
MCP server。
```

## 41. 设计观察

### 41.1 ACP 是深集成，MCP server 是桥

ACP 直接创建 `AIAgent`，处理工具、审批、模型切换、session 历史。

MCP server 不创建 agent。

它主要读 Hermes 的会话状态，并调用消息发送工具。

### 41.2 MCP client 把外部工具变成普通工具

一旦注册到 `tools.registry`，agent loop 不需要关心这个工具来自哪里。

这保持了主循环简单。

### 41.3 stdout 协议通道要严格保护

ACP 和 MCP stdio 都要求 stdout 干净。

日志、debug、子进程 stderr 都要重定向。

否则协议会被污染。

### 41.4 外部工具是安全边界

MCP server 的 tool schema、description、错误信息、subprocess env 都来自外部。

所以 Hermes 做了：

- env 过滤
- 错误脱敏
- description 风险扫描
- built-in tool collision 保护
- circuit breaker
- OAuth token profile 隔离

### 41.5 ACP 的复杂度主要在并发和回调桥接

ACP adapter 要让同步 `AIAgent` 跑在线程池里，同时把事件安全发回 async ACP client。

这就是 `events.py` 里 `asyncio.run_coroutine_threadsafe()` 的原因。

## 42. 后续可以继续深挖的问题

后面如果继续这个模块，可以拆：

- `acp_adapter/tools.py` 如何把 patch 转成 ACP diff block
- ACP model selector 如何和 Hermes provider/runtime provider 联动
- MCP sampling：外部 MCP server 如何请求 Hermes 帮它做 LLM completion
- MCP dynamic `tools/list_changed` 的完整刷新链路
- `hermes mcp login` OAuth 首次授权流程
- MCP server 的 approval event 如何和 Gateway approval 更深度打通

