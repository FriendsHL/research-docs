---
created: '2026-04-30'
tags:
- hermes-agent
- gateway
- messaging
status: draft
---

# Gateway 多平台消息系统源码解析

## 1. 先说结论

Gateway 是 Hermes 的“消息平台入口”。

它把 Telegram、Discord、Slack、WhatsApp、飞书、企业微信、邮件、Webhook、API Server 等平台消息统一成一条 Hermes agent turn。

可以先用一句话理解：

```text
Gateway = 多平台 Adapter + Session 管理 + Slash Command + AIAgent 调度 + 平台回复发送
```

它和 classic CLI 的最大区别是：

```text
CLI 是一个人在一个终端里同步聊天。
Gateway 是很多平台、很多用户、很多 chat 同时异步发消息。
```

所以 Gateway 重点不是“怎么调用模型”，而是：

- 这条消息来自哪里？
- 谁有权限用 bot？
- 这条消息属于哪个 session？
- 当前 session 是否已经有 agent 在跑？
- `/stop`、`/approve` 这种命令能不能插队？
- agent 的流式输出怎么发回平台？
- agent 跑到一半网关重启怎么办？

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `gateway/run.py` | `GatewayRunner`，网关总控 |
| `gateway/platforms/base.py` | `BasePlatformAdapter`，所有平台 adapter 的基类 |
| `gateway/session.py` | `SessionSource`、`SessionStore`、session key 生成 |
| `gateway/config.py` | Gateway 配置、平台枚举、reset policy |
| `gateway/stream_consumer.py` | 把 agent token stream 编辑/发送到平台 |
| `gateway/delivery.py` | cron / delivery target 路由 |
| `gateway/platform_registry.py` | 插件平台 adapter 注册表 |
| `gateway/platforms/*.py` | Telegram、Slack、Discord 等具体平台实现 |

最核心的两个类：

```text
BasePlatformAdapter
  负责平台连接、接消息、发消息、第一层 busy guard。

GatewayRunner
  负责鉴权、命令、session、agent、流式输出、运行状态。
```

## 3. 一条消息的主链路

假设 Telegram 用户发：

```text
帮我查一下今天的部署失败原因
```

大概流程是：

```text
TelegramAdapter 收到消息
  -> 转成 MessageEvent
  -> BasePlatformAdapter.handle_message(event)
  -> 计算 session_key
  -> 如果这个 session 没在运行，启动后台 task
  -> _process_message_background(event, session_key)
  -> GatewayRunner._handle_message(event)
  -> 鉴权
  -> slash command 判断
  -> GatewayRunner._handle_message_with_agent(...)
  -> SessionStore.get_or_create_session(source)
  -> build_session_context_prompt(...)
  -> 加载历史 transcript
  -> 创建或复用 AIAgent
  -> agent.run_conversation(...)
  -> GatewayStreamConsumer / adapter.send 发回平台
```

也可以画成更短的版本：

```text
Platform message
  -> MessageEvent
  -> Adapter guard
  -> GatewayRunner
  -> SessionStore
  -> AIAgent
  -> StreamConsumer / Adapter send
```

## 4. MessageEvent 和 SessionSource

平台 adapter 不直接把平台原始对象丢给 agent。

它会整理成通用结构。

其中最关键的是 `SessionSource`：

```python
SessionSource(
    platform=Platform.TELEGRAM,
    chat_id="123",
    chat_name="某个群",
    chat_type="group",
    user_id="456",
    user_name="Alice",
    thread_id=None,
)
```

这个对象回答几个问题：

```text
消息来自哪个平台？
来自哪个 chat？
是 DM、group、channel 还是 thread？
是谁发的？
有没有 thread/topic？
```

这些信息会被用于三件事：

```text
1. 生成 session_key。
2. 给模型注入当前平台上下文。
3. 回复时知道发回哪里。
```

## 5. session_key：怎么判断“同一个会话”

Gateway 不能简单用 `session_id`。

因为平台消息天然有很多维度：

- Telegram DM
- Telegram 群
- Telegram forum topic
- Discord channel
- Discord thread
- Slack thread
- 同一个群里的不同用户

所以 Hermes 先生成一个稳定的 `session_key`。

入口在：

```python
build_session_key(source, group_sessions_per_user=True, thread_sessions_per_user=False)
```

例子：

### 5.1 DM

```text
platform=telegram
chat_type=dm
chat_id=123
```

可能得到：

```text
agent:main:telegram:dm:123
```

### 5.2 群聊，默认按用户隔离

```text
platform=telegram
chat_type=group
chat_id=group1
user_id=userA
group_sessions_per_user=True
```

可能得到：

```text
agent:main:telegram:group:group1:userA
```

也就是说同一个群里，不同用户默认可以有不同 session。

### 5.3 thread，默认共享

如果是 thread：

```text
chat_id=channel1
thread_id=thread9
thread_sessions_per_user=False
```

可能得到：

```text
agent:main:slack:channel:channel1:thread9
```

默认不拼 user_id。这样一个 thread 里的多人会共享同一条对话线。

## 6. SessionStore：session_key 到 session_id

`SessionStore` 做的是：

```text
session_key -> SessionEntry -> session_id
```

`session_key` 是平台维度的稳定 key。

`session_id` 是真正写入 `state.db` 的会话 id，比如：

```text
20260501_103011_a1b2c3d4
```

`SessionStore` 会保存一个 `sessions.json` 索引，同时用 `SessionDB` 写 SQLite。

它负责：

- 创建新 session。
- 根据 idle/daily reset policy 自动重置 session。
- `/new` 或 `/reset` 强制创建新 session。
- gateway 重启时标记 `suspended` 或 `resume_pending`。
- 保存 token 统计和最后 prompt token 数。

简单理解：

```text
SessionDB 负责完整 transcript。
SessionStore 负责“平台来源 -> 当前活跃 session_id”的映射。
```

## 7. session context prompt：告诉模型它在哪里

Gateway 会给模型加一段动态上下文：

```python
build_session_context_prompt(context, redact_pii=False)
```

里面会告诉模型：

```text
你现在运行在 Slack / Telegram / Discord。
当前是 DM / group / channel / thread。
当前用户是谁。
这个 session 是否多人共享。
连接了哪些平台。
定时任务可以投递到哪里。
平台有什么限制。
```

例如 Slack 会注入类似意思：

```text
你在 Slack 里。
你没有 Slack API 权限，不能搜索频道历史、pin 消息、列用户。
不要承诺你能做这些。
```

Discord 也会注入平台说明。如果 Discord 工具没有启用，就会提醒模型不要假装自己能调用 Discord API。

这个 prompt 是 Gateway 和 CLI 的一个重要差异：

```text
CLI 默认只知道本地终端。
Gateway 必须让模型知道“我在哪个平台里回复谁”。
```

## 8. 启动流程

`GatewayRunner.start()` 做的事情比较多。

主线是：

```text
1. 写 gateway runtime status = starting
2. 检查 allowlist / allow_all 配置
3. discover_plugins()
4. 注册 shell hooks
5. 加载 gateway hooks
6. 恢复 background process checkpoint
7. 处理上次异常退出留下的 active sessions
8. 遍历 config.platforms
9. 为每个平台创建 adapter
10. 给 adapter 设置 message handler / fatal error handler / session store / busy handler
11. adapter.connect()
12. 成功的 adapter 放入 self.adapters
13. 启动 expiry watcher / reconnect loop 等后台任务
```

平台 adapter 的创建有两条路：

```text
plugin platform_registry
  -> plugin 注册的平台 adapter

legacy built-in if/elif
  -> Telegram / Slack / Discord 等内置平台
```

`platform_registry.py` 是给插件平台用的。插件可以注册：

```python
PlatformEntry(
    name="irc",
    label="IRC",
    adapter_factory=lambda cfg: IRCAdapter(cfg),
    check_fn=check_requirements,
)
```

这样 Gateway 不需要在 core 里硬编码所有第三方平台。

## 9. 两层 busy guard：为什么要两层

这是 Gateway 里最关键、也最容易绕晕的一块。

有两层“这个 session 正在运行”的判断：

```text
第一层：BasePlatformAdapter._active_sessions
第二层：GatewayRunner._running_agents
```

### 9.1 第一层：adapter guard

在 `BasePlatformAdapter.handle_message()`。

它的作用是：

```text
同一个 session_key 同一时间只启动一个后台 message-processing task。
```

如果 session 正在跑，新消息进来：

- 如果是可插队的 slash command，就直接交给 GatewayRunner。
- 如果是普通消息，就进入 `_pending_messages`。
- 默认会触发 interrupt event。
- 如果是 photo burst，则合并排队，不立即打断。

它主要防止：

```text
同一个 Telegram chat 同时启动两个 agent。
```

### 9.2 第二层：runner guard

在 `GatewayRunner._handle_message()`。

它看的是：

```text
self._running_agents[session_key]
```

它的作用是：

```text
agent 已经创建/正在启动/正在跑时，决定具体怎么处理新消息和 slash command。
```

它能做更细的判断：

- `/stop` 直接停止当前 agent。
- `/new` 中断并重置 session。
- `/approve` / `/deny` 解锁危险命令审批。
- `/queue` 加入 FIFO 队列。
- `/steer` 注入当前 agent loop。
- `/model` 运行中拒绝。
- 普通消息按 busy_input_mode 处理：interrupt / queue / steer。

### 9.3 为什么不能只有一层

如果只有 adapter guard：

```text
/approve 可能被排队
agent 正在等 approval
但 approval 永远到不了
```

这就是死锁。

如果只有 runner guard：

```text
两个平台消息可能同时进入 _handle_message_with_agent
同一个 session 启动两个 agent
```

会导致重复回复、重复工具调用、历史错乱。

所以两层各管一件事：

```text
adapter guard 管并发 task
runner guard 管 agent 状态和命令语义
```

## 10. 运行中消息的三种模式

Gateway 支持 busy input mode。

大概有三种：

```text
interrupt
queue
steer
```

### 10.1 interrupt

默认语义：

```text
用户又发了一条普通消息。
当前 agent 被 interrupt。
新消息作为下一轮处理。
```

例子：

```text
用户：帮我跑完整测试
agent：开始跑
用户：算了，只跑 gateway 测试
```

Gateway 会调用：

```python
running_agent.interrupt(event.text)
```

然后把新消息存在 pending 里，当前 agent 结束后处理新消息。

### 10.2 queue

语义：

```text
当前 agent 不被打断。
新消息排队，等当前任务完成后再处理。
```

`/queue` 强制使用这种语义。

例子：

```text
/queue 修完后总结一下改了哪些文件
```

Gateway 用 `_queued_events` 保证 `/queue` 是 FIFO，不会被合并掉。

### 10.3 steer

语义：

```text
把用户补充的话注入当前 agent loop，尽量当前轮生效。
```

例子：

```text
/steer 不要改数据库迁移，先只修 gateway
```

Gateway 会尝试：

```python
running_agent.steer(steer_text)
```

这个文本会在 agent loop 中被追加到某个工具结果附近，模型下一次 API call 能看到。

如果 agent 还没准备好，或者 steer 失败，就 fallback 到 queue。

## 11. 可插队命令：/stop、/approve、/deny

有些命令不能排队。

最典型的是：

```text
/stop
/approve
/deny
/new
/status
```

### /approve 和 /deny

危险命令审批时，agent 线程可能阻塞在一个 `threading.Event` 上。

如果 `/approve` 被 adapter guard 排队，就会变成：

```text
agent 等 approve
approve 等 agent 结束后才处理
```

这就是死锁。

所以 `/approve` 和 `/deny` 必须绕过 busy queue，直接到：

```python
_handle_approve_command()
_handle_deny_command()
```

然后唤醒等待中的工具线程。

### /stop

`/stop` 也必须插队。

它不只是普通 interrupt。因为有些情况下 agent 可能真的卡住，不会检查 `_interrupt_requested`。

Gateway 会：

```text
1. interrupt running agent
2. 清理 _running_agents
3. 清理 pending 状态
4. 释放 session lock
5. 回复用户：Stopped
```

### /new

`/new` / reset 类命令也要插队。

否则它可能被当成普通用户文本喂给模型，模型会看到：

```text
/new
```

然后继续在坏掉的旧历史上运行。

## 12. Gateway slash command 分发

Gateway 和 CLI 共用 `hermes_cli/commands.py` 的命令元数据，但执行逻辑在 `gateway/run.py`。

流程：

```text
event.get_command()
  -> resolve_command(command)
  -> canonical name
  -> emit command:<canonical> hook
  -> 一系列 if canonical == ...
```

例子：

```text
/m gpt-5.5
```

会解析成：

```text
canonical = model
```

然后走：

```python
_handle_model_command(event)
```

Gateway 也支持 plugin hook 拦截 command：

```text
decision = deny     阻止命令
decision = handled  插件已经处理
decision = rewrite  改写成另一条命令
decision = allow    放行
```

这比 CLI 更复杂，因为 Gateway 需要让平台插件、策略插件、安全插件参与消息流。

## 13. Gateway hooks

Gateway 有 hook 系统。

在消息入口处：

```text
pre_gateway_dispatch
```

插件可以返回：

```json
{"action": "skip"}
{"action": "rewrite", "text": "..."}
{"action": "allow"}
```

在 slash command 处：

```text
command:<name>
```

插件可以：

- deny
- handled
- rewrite
- allow

另外启动时还会：

```text
self.hooks.discover_and_load()
```

这些 hook 让 Gateway 不需要把所有业务逻辑写进 `gateway/run.py`。

## 14. Agent cache：为什么 Gateway 要缓存 AIAgent

CLI 通常一个进程一个交互 session。

Gateway 不一样：

```text
用户发一句
隔 5 分钟再发一句
又隔 10 分钟再发一句
```

如果每条消息都新建 `AIAgent`，system prompt 和 tool schemas 都会重新组装，prompt caching 很容易失效。

所以 Gateway 有：

```python
self._agent_cache: OrderedDict[str, tuple]
```

key 是：

```text
session_key
```

value 大概是：

```text
(AIAgent, config_signature)
```

如果本轮的配置签名没变，就复用旧 agent。

如果模型、provider、toolsets、ephemeral config 等影响 prompt/tool 的配置变了，就创建新 agent。

这就是 Gateway 保持多轮 prompt cache 的关键。

## 15. 运行 agent 的核心流程

`_handle_message_with_agent()` 是准备阶段。

它会做：

```text
1. get_or_create_session()
2. build_session_context()
3. build_session_context_prompt()
4. 加载 history transcript
5. 必要时做 gateway 级别的预压缩
6. 准备 model/provider/toolsets/runtime config
7. 创建或复用 AIAgent
8. 设置 callbacks
9. 注册 approval callback
10. agent.run_conversation(...)
11. 处理 final_response / media / tokens / transcript
12. 处理 pending follow-up
```

真正调用模型主循环还是：

```python
agent.run_conversation(
    _run_message,
    conversation_history=agent_history,
    task_id=session_id,
)
```

所以 Gateway 没有重写 agent loop。

它只是把 messaging 场景需要的所有东西准备好，再调用同一个 `AIAgent`。

## 16. 流式输出：GatewayStreamConsumer

agent 在 worker thread 里跑。

平台发送是 async。

所以需要一个桥：

```text
agent.stream_delta_callback(text)
  -> GatewayStreamConsumer.on_delta(text)
  -> queue.Queue
  -> async GatewayStreamConsumer.run()
  -> adapter.send / adapter.edit
```

`GatewayStreamConsumer` 做几件事：

- 收集 token delta。
- 按 `edit_interval` 和 `buffer_threshold` 限流。
- 先发一条消息，再持续 edit。
- final 时去掉 cursor。
- 工具边界时开新 message segment。
- 过滤 `<think>` / reasoning block，避免用户看到模型思考标签。
- 处理平台消息长度限制。
- 如果平台 edit 失败或 flood control，就 fallback。

例子：

```text
模型流式输出：
“我先检查日志...”

Telegram:
发送 “我先”
编辑成 “我先检查”
编辑成 “我先检查日志...”
最后编辑成完整答案
```

如果中间开始工具调用，stream consumer 会 segment break，让后续文本出现在工具进度消息下面，而不是一直编辑旧消息。

## 17. 平台 adapter 负责发送细节

`BasePlatformAdapter._process_message_background()` 在 handler 返回 response 后，会处理平台发送。

它会做：

- 提取 `MEDIA:<path>`。
- 提取图片 URL。
- 自动识别本地文件路径。
- voice 输入时可能自动 TTS。
- 发送 text。
- 发送 images。
- 发送 audio/video/document。
- 调用 post delivery callbacks。
- 检查 pending message，继续处理下一条。

也就是说，agent 返回的 final response 不一定只是一段文字。

它可能包含：

```text
MEDIA:/tmp/foo.mp3
```

adapter 会把它变成平台原生语音/文件发送。

## 18. pending message 怎么继续处理

当 agent 正在跑时，用户又发了一条消息。

这条消息通常会进入：

```text
adapter._pending_messages[session_key]
```

当前 agent 结束后，Gateway 会：

```text
1. _dequeue_pending_event(adapter, session_key)
2. 如果有 /queue overflow，_promote_queued_event()
3. 判断是不是 slash command 泄漏，如果是就丢弃
4. 如果当前响应还没发给用户，先发当前响应
5. 用 result["messages"] 作为 updated_history
6. 调用 _run_agent(...) 处理 pending 消息
```

这里有个重要安全网：

```text
pending 里如果是已知 slash command，不允许当作用户消息喂给模型。
```

因为 `/stop`、`/new` 这种命令如果漏进 agent，会造成很奇怪的行为。

## 19. background process completion

Hermes 工具里可以启动后台进程。

比如 terminal 工具：

```text
background=true
notify_on_complete=true
```

Gateway 会有 watcher 检测后台进程完成。

完成后，它会生成一个 internal `MessageEvent`，触发新的 agent turn。

这种 internal event 会：

```text
跳过普通用户鉴权
保留 routing 信息
把“后台进程完成了，输出是...”作为重要消息交给 agent
```

所以用户可以在消息平台里启动一个长任务，任务完成后 bot 自动回来继续总结。

## 20. 重启和恢复：suspended / resume_pending

Gateway 是长驻进程，可能被重启、更新、崩溃。

它有两个不同的恢复状态：

### 20.1 suspended

意思是：

```text
上次可能有 session 卡住了。
下一次用户发消息时，直接开新 session。
```

常见来源：

- 非干净退出后，启动时 `suspend_recently_active()`。
- `/stop` 后为了避免 stuck resume loop。

### 20.2 resume_pending

意思是：

```text
Gateway 重启/停止时 agent 没跑完，
但 transcript 还在，下一次保留原 session_id，尝试续上。
```

下一轮 Gateway 会给模型加 system note：

```text
上一轮被 gateway restart/shutdown 打断。
历史还在。
如果有未处理的 tool result，先处理它们，再回答用户新消息。
```

这和普通 reset 不同。`resume_pending` 是“继续原会话”，`suspended` 是“开新会话”。

## 21. 鉴权和 pairing

Gateway 不能默认让任何人用 bot。

`_handle_message()` 会先检查：

```text
source.user_id
_is_user_authorized(source)
```

如果是未授权 DM，可能走 pairing：

```text
生成 pairing code
提示 bot owner 执行 hermes pairing approve ...
```

如果是群消息里的未授权用户，通常静默忽略。

另外 busy path 也会再次鉴权。

这是一个很重要的细节：

```text
如果只在冷路径鉴权，未授权用户可能趁 session 正在跑时往 active session 注入消息。
```

所以 `_handle_active_session_busy_message()` 也会做 authorization gate。

## 22. 一个完整例子：agent 跑着时用户发 /approve

场景：

```text
用户：帮我清理日志
agent：准备执行 rm ...
gateway：发现危险命令，需要审批
用户：/approve
```

流程：

```text
1. adapter.handle_message("/approve")
2. 发现 session_key 在 _active_sessions
3. should_bypass_active_session("approve") = True
4. 直接调用 GatewayRunner._handle_message(event)
5. GatewayRunner 发现 _running_agents 里有 agent
6. 命令 canonical = approve
7. _handle_approve_command()
8. resolve_gateway_approval(...)
9. 被阻塞的工具线程恢复
10. agent 继续执行
```

如果第 3 步没有 bypass，就会死锁。

## 23. 一个完整例子：agent 跑着时用户发普通消息

场景：

```text
用户：帮我检查整个项目
agent：开始读文件和跑测试
用户：先别管前端，只看 gateway
```

如果 busy mode 是 `interrupt`：

```text
1. adapter 发现 session active
2. 调用 busy handler
3. GatewayRunner 把消息放进 pending
4. running_agent.interrupt("先别管前端，只看 gateway")
5. 当前 run 在检查点退出
6. Gateway 取出 pending
7. 用当前 result["messages"] 作为历史
8. 重新进入 agent.run_conversation()
```

如果 busy mode 是 `queue`：

```text
不打断当前 agent。
消息等当前任务完成后处理。
```

如果 busy mode 是 `steer`：

```text
尝试 agent.steer(...)
让模型在当前轮下一次 API call 看到这句话。
```

## 24. 一个完整例子：Slack thread 里的多人会话

假设 Slack thread 里：

```text
Alice：帮我们分析部署失败
Bob：我补充一下，失败的是 worker 服务
```

如果 `thread_sessions_per_user=False`，这两条消息会共享 session_key：

```text
agent:main:slack:channel:<channel_id>:<thread_id>
```

模型会被告知：

```text
这是 multi-user thread。
消息会带 [sender name] 前缀。
多个用户可能参与。
```

这样模型不会误以为所有话都来自同一个人。

## 25. 设计观察

### 25.1 Gateway 的复杂度主要来自并发和平台差异

核心 agent loop 还是同一个 `AIAgent`。

Gateway 的难点是：

- 多平台输入不一样。
- 同一 session 可能同时来多条消息。
- 有些命令必须插队。
- 有些平台能 edit，有些不能。
- 有些平台有 thread，有些没有。
- 长任务和重启恢复要保住状态。

### 25.2 两层 guard 是必要复杂度

adapter guard 防止同一 session 同时启动多个处理 task。

runner guard 处理 agent 已经存在时的命令语义。

这两层任何一层缺失，都会出问题。

### 25.3 Slash command 在 Gateway 更像控制面

CLI 的 slash command 很多是本地 UI 操作。

Gateway 的 slash command 还承担控制 running agent 的职责：

- `/stop`
- `/approve`
- `/deny`
- `/queue`
- `/steer`
- `/status`

所以不能简单套用 CLI 的处理方式。

### 25.4 Prompt caching 仍然是核心约束

Gateway 缓存 `AIAgent`，避免每条消息都重建 system prompt。

session context、模型切换 note、reload skills note 都尽量用“下一轮一次性注入”的方式，而不是随便改历史。

### 25.5 平台 adapter 要尽量薄，但现实里会变厚

理想上 adapter 只负责收发消息。

现实中 adapter 还要处理：

- typing indicator
- 图片/文件/语音
- edit message
- approval buttons
- thread metadata
- 平台限流

所以 `BasePlatformAdapter` 会比较重。

## 26. 后续可以继续深挖的问题

后续如果继续 Gateway 模块，可以看：

- Telegram adapter 如何处理 polling / topic / photo album。
- Slack adapter 如何处理 Assistant API 和 typing status。
- Discord adapter 如何处理 slash command / button approval。
- `/background` 和 process watcher 的完整链路。
- gateway hooks 的配置格式和执行顺序。
- `GatewayStreamConsumer` 的 flood control 和 fallback 细节。
- gateway update / restart drain 的完整状态机。

