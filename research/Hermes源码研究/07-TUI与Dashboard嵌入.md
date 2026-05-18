---
created: '2026-04-30'
tags:
- hermes-agent
- tui
- dashboard
status: draft
---

# TUI 与 Dashboard 嵌入源码解析

## 1. 先说结论

Hermes 的新 TUI 不是“Python 终端里直接 print UI”。

它分成两半：

```text
Node / Ink / React
  负责画终端 UI、处理键盘鼠标、渲染消息和工具状态

Python tui_gateway
  负责 session、AIAgent、工具调用、slash command、审批、压缩
```

两边通过 JSON-RPC 通信。

可以先把它理解成：

```text
TUI = 一个终端里的 React 前端 + 一个 Python agent 后端
```

Dashboard 的 `/chat` 也不是重新写了一套 React 聊天页面。

它是把真实的 `hermes --tui` 放进浏览器里的 xterm.js：

```text
浏览器 xterm.js
  -> WebSocket /api/pty
  -> Python PTY bridge
  -> hermes --tui
  -> Node Ink UI
  -> Python tui_gateway
  -> AIAgent
```

所以一个功能如果加在 Ink TUI 里，Dashboard 里也会自动有。

这是这个模块最重要的设计点。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `ui-tui/src/entry.tsx` | Node TUI 入口，启动 GatewayClient，渲染 Ink App |
| `ui-tui/src/gatewayClient.ts` | 启动 Python `tui_gateway.entry`，维护 JSON-RPC 请求和事件 |
| `ui-tui/src/app.tsx` | TUI App 外壳，本身很薄 |
| `ui-tui/src/app/createGatewayEventHandler.ts` | 把 Python 事件转成前端状态变化 |
| `ui-tui/src/app/useSubmission.ts` | 用户输入发送入口 |
| `ui-tui/src/app/createSlashHandler.ts` | slash command 前端分发 |
| `ui-tui/src/components/prompts.tsx` | approval / clarify / confirm 等覆盖层 |
| `tui_gateway/entry.py` | Python gateway 进程入口，读写 JSON-RPC line |
| `tui_gateway/server.py` | JSON-RPC method 注册、session、agent 调度 |
| `tui_gateway/transport.py` | stdio / websocket / tee transport 抽象 |
| `tui_gateway/ws.py` | dashboard 侧边栏可用的 JSON-RPC WebSocket |
| `hermes_cli/pty_bridge.py` | Dashboard 把 TUI 子进程接到 PTY |
| `hermes_cli/web_server.py` | `/api/pty`、`/api/ws`、`/api/pub`、`/api/events` |
| `web/src/pages/ChatPage.tsx` | Dashboard Chat 页，xterm.js 终端容器 |

## 3. 三种运行形态

### 3.1 终端里运行 TUI

用户执行：

```bash
hermes --tui
```

真实结构大概是：

```text
shell
  -> node ui-tui/dist/entry.js
  -> Ink App
  -> spawn python -m tui_gateway.entry
  -> AIAgent / tools / SessionDB
```

Node 进程负责 UI。

Python 进程负责 agent。

### 3.2 TUI 内部的 JSON-RPC

Node 和 Python 之间通过 stdout/stdin 传 JSON。

大概像这样：

```json
{"jsonrpc":"2.0","id":1,"method":"prompt.submit","params":{"session_id":"abc","text":"帮我看测试失败"}}
```

Python 不一定马上返回最终答案。

它会先返回：

```json
{"jsonrpc":"2.0","id":1,"result":{"status":"streaming"}}
```

然后不断发 event：

```json
{"jsonrpc":"2.0","method":"event","params":{"type":"message.delta","payload":{"text":"我先看一下..."}}}
```

最后发：

```json
{"jsonrpc":"2.0","method":"event","params":{"type":"message.complete","payload":{"text":"最终回答..."}}}
```

所以 TUI 不是一次 RPC 等到最终结果，而是：

```text
请求提交一次
事件流返回很多次
```

### 3.3 Dashboard 里的 Chat

Dashboard 的 `/chat` 页面打开后，会创建一个 xterm.js 终端。

浏览器连：

```text
/api/pty?token=...&channel=...
```

服务端做两件事：

```text
1. 创建一个 POSIX PTY。
2. 在 PTY 里启动 hermes --tui。
```

之后浏览器和 TUI 的关系就像：

```text
用户按键
  -> xterm.js
  -> WebSocket
  -> PTY stdin
  -> hermes --tui

TUI 输出 ANSI
  -> PTY stdout
  -> WebSocket bytes
  -> xterm.js 渲染
```

这就是“嵌入真实 TUI”。

## 4. classic CLI 和 TUI 的区别

Classic CLI 更像：

```text
一个 Python 进程
  -> prompt_toolkit 读输入
  -> AIAgent
  -> rich / display 打印输出
```

TUI 更像：

```text
Node 前端
  -> 画完整终端 UI
  -> 发送 JSON-RPC

Python 后端
  -> 管 agent、工具、session、命令
```

所以 TUI 能更容易做这些事情：

- 多区域布局
- 工具活动列表
- approval / clarify 弹层
- session picker
- slash command popup
- 鼠标操作
- Dashboard 复用同一套 UI

代价是：

- 多了 Node 和 Python 两个进程
- 必须严格保护 stdout，不能乱 print
- 事件协议要设计清楚

## 5. Node TUI 如何启动

入口在 `ui-tui/src/entry.tsx`。

它做的事情很直接：

```text
1. 检查是不是 TTY。
2. 创建 GatewayClient。
3. GatewayClient.start() 启动 Python gateway。
4. 等 Python 发 gateway.ready。
5. 动态 import Ink 和 App。
6. render(<App gw={gw} />)。
```

如果不是 TTY，它会直接退出：

```text
hermes-tui: no TTY
```

这说明 TUI 默认就是为真实终端准备的。

Dashboard 能用它，是因为 Dashboard 给它造了一个 PTY。

## 6. GatewayClient 做什么

`ui-tui/src/gatewayClient.ts` 是 Node 侧的“Python 后端客户端”。

它会启动：

```text
python -m tui_gateway.entry
```

启动时会处理几个细节：

- 优先使用 `HERMES_PYTHON`
- 其次用 `PYTHON`
- 再找 `.venv` / `venv`
- 最后 fallback 到 `python3` / `python`
- 设置 `PYTHONPATH` 到项目根目录

然后它维护三类东西：

```text
1. request id -> Promise
2. Python stdout JSON line -> response 或 event
3. Python stderr -> gateway.stderr event
```

为什么 stderr 也要转 event？

因为 Python 进程如果报错，终端 UI 不能直接被 stderr 打乱。

它要变成一条可控的活动消息：

```text
gateway.stderr
  -> 前端 activity feed
```

## 7. Python tui_gateway 入口

入口在 `tui_gateway/entry.py`。

它启动后会先发：

```json
{
  "jsonrpc": "2.0",
  "method": "event",
  "params": {
    "type": "gateway.ready",
    "payload": {
      "skin": "..."
    }
  }
}
```

Node 侧收到 `gateway.ready` 后，才认为后端可用。

之后 Python 就一直读 stdin：

```text
stdin 一行 JSON
  -> dispatch(req)
  -> 如果有 response，写 stdout 一行 JSON
```

一个关键细节：

```text
stdout 只允许写 JSON-RPC。
普通 print 会污染协议。
```

所以 `tui_gateway/server.py` 会把普通 stdout 重定向到 stderr。

## 8. server.py 的核心结构

`tui_gateway/server.py` 是 Python TUI 后端核心。

它里面有几个全局结构：

```text
_methods
  JSON-RPC method 注册表

_sessions
  TUI session 内存表

_pending / _answers
  clarify / sudo / secret 这类阻塞问题的等待表

_LONG_HANDLERS
  慢方法列表，会丢到线程池执行
```

method 通过装饰器注册：

```python
@method("prompt.submit")
def prompt_submit(params, rid):
    ...
```

dispatch 的大概逻辑是：

```text
收到 method
  -> 找 handler
  -> 如果是慢 handler，放到 ThreadPoolExecutor
  -> 否则当前线程直接执行
  -> 返回 JSON-RPC response
```

为什么要区分快慢方法？

比如：

```text
prompt.submit
slash.exec
session.resume
session.compress
shell.exec
```

这些可能很慢。

如果它们占住主读取线程，那么 approval.respond、interrupt 这种控制消息就进不来了。

所以长任务放线程池，控制消息还能继续处理。

## 9. AIAgent 是什么时候创建的

TUI 不是一启动就立即创建 AIAgent。

它是延迟创建：

```text
session 创建
  -> 先放一个 session 结构
  -> 真正第一次 prompt / 需要 agent 的命令来了
  -> _start_agent_build(...)
  -> 后台线程创建 AIAgent
```

这样做的好处是：

- TUI 打开更快
- session picker / slash help 这类操作不一定需要马上建 agent
- 创建模型客户端、工具、记忆等开销可以延后

创建 agent 时会绑定一批 callback。

这些 callback 很重要，因为 agent loop 内部的工具、思考、审批都靠它们发回前端。

## 10. callback 如何变成 UI 事件

`server.py` 里 `_agent_cbs(sid)` 会返回一组 callback。

大概对应关系是：

| Agent callback | TUI event |
|---|---|
| tool start | `tool.start` |
| tool complete | `tool.complete` |
| tool progress | `tool.progress` |
| thinking delta | `thinking.delta` |
| reasoning delta | `reasoning.delta` |
| status update | `status.update` |
| clarify callback | `clarify.request` |
| tool generating | `tool.generating` |

前端在 `createGatewayEventHandler.ts` 里接这些 event。

例如：

```text
tool.start
  -> Thinking / activity 区域显示某个工具开始运行

message.delta
  -> transcript 里追加 assistant token

message.complete
  -> assistant 消息收尾，更新 usage/status

approval.request
  -> 打开审批弹层
```

所以 Python 不直接控制 UI。

它只发事件：

```text
后端说发生了什么
前端决定怎么显示
```

## 11. 一条用户消息完整例子

假设用户在 TUI 里输入：

```text
帮我看一下 tests/agent/test_x.py 为什么失败
```

流程大概是：

```text
1. 用户按 Enter
2. useSubmission.send(text)
3. 前端先把 user message 放到本地 transcript
4. 调用 gw.request("prompt.submit", {session_id, text})
5. Python prompt.submit 检查 session 是否 busy
6. 如果不 busy，标记 running=True
7. 返回 {status:"streaming"}
8. 后台线程执行 _run_prompt_submit
9. emit message.start
10. agent.run_conversation(...)
11. 模型决定调用工具
12. callback emit tool.start
13. 工具完成 callback emit tool.complete
14. 模型开始输出答案
15. stream_callback emit message.delta
16. 完成后 emit message.complete
17. running=False
```

更短地看：

```text
输入
  -> prompt.submit
  -> message.start
  -> tool.start / tool.complete
  -> message.delta ...
  -> message.complete
```

### 11.1 如果 session 正在运行怎么办

`prompt.submit` 会检查：

```text
session["running"]
```

如果已经在跑，会返回类似“session busy”的错误。

前端会把输入放回 composer，并提示 queued。

也就是说 TUI 默认一个 session 同时只跑一个 agent turn。

## 12. 流式输出怎么做

`_run_prompt_submit` 里会传一个 `_stream(delta)` 给 `agent.run_conversation`。

模型每流出一段文本，都会走：

```text
stream_callback
  -> make_stream_renderer(cols)
  -> emit message.delta
```

这里不只是传纯文本。

如果终端列宽已知，还会生成 rendered ANSI。

这样前端可以更贴近终端布局显示。

最后完成时发：

```text
message.complete
```

里面通常有：

- final text
- usage
- status
- reasoning
- warning
- rendered

## 13. 工具调用怎么显示

工具调用本身还是 AIAgent / model_tools 那套逻辑。

TUI 关心的是状态事件。

例子：

```text
模型要读文件
  -> agent 触发 tool_start_callback
  -> Python emit tool.start
  -> 前端显示工具开始

工具读完
  -> agent 触发 tool_complete_callback
  -> Python emit tool.complete
  -> 前端更新工具结果状态
```

如果工具中途有进度：

```text
tool.progress
```

如果是 subagent：

```text
subagent.spawn_requested
subagent.start
subagent.thinking
subagent.tool
subagent.progress
subagent.complete
```

这些事件都被前端统一转成 UI 状态。

## 14. approval / clarify / secret 怎么跨进程

有些操作不能直接继续，需要问用户。

例如：

```text
agent 想执行一个需要审批的命令
```

流程大概是：

```text
Python agent
  -> 触发 approval request
  -> emit approval.request
  -> 前端打开 ApprovalPrompt
  -> 用户选择 once/session/always/deny
  -> 前端调用 approval.respond
  -> Python resolve_gateway_approval(...)
  -> agent 继续或拒绝
```

`prompts.tsx` 里有 `ApprovalPrompt`，选项大概是：

```text
once
session
always
deny
```

clarify / sudo / secret 更像阻塞问答。

后端 `_block(...)` 会：

```text
1. 生成 request_id。
2. 放进 _pending。
3. emit clarify.request / sudo.request / secret.request。
4. 等前端 respond。
5. 拿到答案继续。
```

这就是跨进程“暂停等用户输入”的方式。

## 15. Slash command 流程

TUI 的 slash command 不是全部丢给 Python。

它分三层：

```text
第一层：前端本地命令
  /help、/quit、/clear、/resume、/copy、/paste 等

第二层：slash.exec
  发给 Python 的 persistent _SlashWorker
  尽量复用 classic CLI 的命令行为

第三层：command.dispatch
  结构化处理 pending-input、skill command、alias、plugin command 等
```

举例：

### 15.1 `/clear`

这类只影响 TUI 本地画面。

前端直接处理，不需要让 agent 参与。

### 15.2 `/skills xxx`

如果是 skill slash command，后端可能返回：

```json
{"type":"skill","name":"xxx","message":"..."}
```

前端收到后不是展示一段结果，而是把 `message` 当成用户消息继续发送给 agent。

### 15.3 `/steer`

这类命令会走 `command.dispatch`，最后可能调用：

```text
agent.steer(text)
```

它影响正在跑的 agent。

## 16. TUI session 和历史

TUI 后端有内存里的 `_sessions`。

每个 session 大概保存：

- session id
- session key
- history
- history_version
- agent
- running 状态
- slash worker
- transport
- terminal cols

当 prompt submit 开始时，后端会 snapshot：

```text
history
history_version
images
```

agent 完成后，如果 `history_version` 没被别人改过，才把新 messages 写回 session history。

这是为了避免异步过程中历史被覆盖。

## 17. TUI 里的压缩

TUI 也有 session compression。

入口是 `session.compress`。

它会调用内部压缩逻辑，压完后还要处理一个细节：

```text
_sync_session_key_after_compress
```

为什么？

因为 agent 压缩上下文时，可能会生成新的 session id / session key 状态。

TUI 后端需要把自己的 session 表和 agent 的新状态对齐。

否则会出现：

```text
前端还以为是旧 session
agent 实际已经切到压缩后的新上下文
```

所以 TUI 的压缩不是单独一套，它是调用 agent/session 层能力，然后同步 TUI session 状态。

## 18. Transport 抽象

`tui_gateway/transport.py` 里把输出抽象成 transport。

目前主要有：

```text
StdioTransport
  TUI 正常运行时，写 stdout JSON line

WSTransport
  Dashboard 结构化 WebSocket 使用

TeeTransport
  一份事件写给主 transport，再 best-effort 写给 sidecar
```

为什么要抽象？

因为同一套 `server.dispatch` 可以服务两个入口：

```text
1. 终端 TUI：stdio
2. Dashboard sidecar：websocket
```

而且事件可以被复制到 Dashboard 侧栏。

## 19. Dashboard 为什么用 PTY

Dashboard 的主聊天区要显示完整 TUI。

如果重写成 React，会有几个问题：

- classic TUI 有的功能，Dashboard 要再实现一遍
- slash popup、selection、approval、session picker 都要重复维护
- Ink 的终端布局和浏览器 React 布局很难完全一致
- 新功能可能只在一个界面可用

所以 Dashboard 选择：

```text
直接跑 hermes --tui
浏览器只当终端模拟器
```

核心在 `hermes_cli/pty_bridge.py`：

```text
PtyBridge.spawn(argv, cwd, env, cols, rows)
  -> ptyprocess.PtyProcess.spawn(...)
  -> read PTY bytes
  -> write PTY bytes
  -> resize via TIOCSWINSZ
```

PTY 是字节流。

所以它不用 Unicode wrapper，而是直接 `os.read` / `os.write`。

原因是 ANSI 输出和 UTF-8 都可能被切成半包，字节流更安全。

## 20. `/api/pty` 做了什么

Dashboard Chat 页连接：

```text
/api/pty?token=<session_token>&channel=<channel_id>
```

服务端流程：

```text
1. 检查 embedded chat 是否开启。
2. 校验 token。
3. 确认来源是 localhost / loopback。
4. accept WebSocket。
5. 解析 resume 和 channel。
6. 构造 sidecar_url。
7. 解析 hermes --tui 的 argv/cwd/env。
8. PtyBridge.spawn(...)。
9. 开 reader task：PTY -> WebSocket bytes。
10. 主循环：WebSocket -> PTY。
11. 如果收到 RESIZE escape，只 resize，不写给 PTY。
12. 断开时关闭 PTY 子进程。
```

resize 的协议很简单：

```text
\x1b[RESIZE:<cols>;<rows>]
```

浏览器窗口变化时，xterm.js 重新 fit，然后发：

```text
\x1b[RESIZE:120;40]
```

服务端收到后调用：

```text
bridge.resize(cols=120, rows=40)
```

## 21. Dashboard ChatPage 做什么

`web/src/pages/ChatPage.tsx` 是浏览器侧。

它创建：

```text
Terminal from @xterm/xterm
FitAddon
Unicode11Addon
WebLinksAddon
WebglAddon
```

然后：

```text
term.onData(data)
  -> WebSocket send

ws.onmessage(data)
  -> term.write(data)
```

这就是终端输入输出主链路。

它还处理一些浏览器端细节：

- 移动端字体大小
- WebGL 宽屏使用，窄屏 fallback
- resize observer
- Clipboard / OSC 52
- 鼠标事件去重，避免 hover 产生太多 mouse move
- `/copy` 快捷按钮
- mobile side panel

注意：这些不是重新实现 chat。

它们是在浏览器里把终端体验接好。

## 22. Dashboard 侧栏事件怎么来

主画面走 PTY，只能拿到 ANSI 字节。

但是 Dashboard 还想在旁边显示结构化信息，比如：

- 当前模型
- 工具调用
- session 信息
- tool call sidebar

ANSI 字节不适合做这些结构化 UI。

所以 Hermes 又加了一条 sidecar 链路：

```text
PTY 里的 tui_gateway.entry
  -> HERMES_TUI_SIDECAR_URL
  -> WsPublisherTransport
  -> /api/pub
  -> /api/events
  -> Dashboard sidebar
```

具体说：

1. `/api/pty` 启动 TUI 子进程时，如果有 `channel`，会设置：

```text
HERMES_TUI_SIDECAR_URL=ws://.../api/pub?token=...&channel=...
```

2. `tui_gateway/entry.py` 发现这个环境变量后，会创建 `TeeTransport`。

3. 正常事件继续写给 Ink stdio。

4. 同一份事件 best-effort 复制到 `/api/pub`。

5. `/api/pub` 再广播给同 channel 的 `/api/events` 订阅者。

所以 Dashboard 是两条线：

```text
主聊天 UI
  PTY / xterm / ANSI

结构化侧栏
  sidecar event WebSocket
```

这个设计既保留了真实 TUI，又能让 Dashboard 有额外 React 面板。

## 23. 一个 Dashboard 请求例子

假设用户在 Dashboard `/chat` 里输入：

```text
帮我总结一下这个项目的工具系统
```

实际链路是：

```text
1. 用户在浏览器 xterm 输入文字。
2. xterm onData 把按键发到 /api/pty。
3. /api/pty 写入 PTY。
4. PTY 里的 hermes --tui 收到按键。
5. Ink composer 显示输入。
6. 用户 Enter。
7. TUI 前端调用 prompt.submit。
8. Python tui_gateway 执行 agent.run_conversation。
9. agent 需要读文件，触发 tool.start。
10. tool.start 事件发给 Ink，TUI 主画面显示工具状态。
11. 同一个 tool.start 也通过 sidecar 发到 /api/events。
12. Dashboard 右侧 React 面板也能显示工具状态。
13. assistant 输出 token，Ink 渲染成 ANSI。
14. ANSI 经过 PTY 发回浏览器 xterm。
15. 用户看到答案。
```

简单说：

```text
文字输入和答案显示：走 PTY
工具结构化信息：走 sidecar events
```

## 24. 为什么 stdout 保护很重要

TUI 的 Python 后端 stdout 是协议通道。

如果某处代码随手：

```python
print("debug")
```

Node 侧会尝试把这一行当 JSON parse。

结果就是：

```text
JSON-RPC 协议被污染
TUI 收不到正常事件
```

所以设计上：

```text
stdout = JSON-RPC
stderr = 日志 / 错误
```

`GatewayClient` 会把 stderr 变成 `gateway.stderr` event。

这样错误可以被 UI 展示，但不会破坏协议。

## 25. 设计上的取舍

### 25.1 好处

```text
一套 TUI，多处复用。
```

终端和 Dashboard 不用维护两套聊天体验。

### 25.2 代价

```text
链路更长。
```

一次输入会经过：

```text
browser / terminal
  -> Node Ink
  -> Python gateway
  -> AIAgent
  -> callbacks
  -> events
  -> UI
```

Dashboard 甚至还多了一层：

```text
xterm
  -> websocket
  -> PTY
  -> Node Ink
```

所以这里的关键不是“少写代码”，而是“同一个产品体验只实现一次”。

## 26. 后续可以继续看的点

下一步如果继续深入，可以拆这些模块：

- TUI 前端 store 和 transcript 渲染
- slash command 在 TUI 和 classic CLI 里的兼容层
- approval / permission 系统
- Dashboard 的 ChatSidebar / ToolCall 结构化面板
- `tui_gateway` 的 session resume / branch / compress
- Ink 组件布局和键盘鼠标输入处理

