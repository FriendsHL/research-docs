---
created: '2026-04-30'
tags:
- hermes-agent
- cli
- slash-commands
status: draft
---

# CLI 与 Slash Commands 源码解析

## 1. 先说结论

这一层可以理解成 Hermes 的“本地终端外壳”。

用户在终端里输入一行内容，CLI 先判断它是不是 `/` 开头：

- 不是 `/` 开头：当作普通用户消息，交给 `AIAgent.run_conversation()`。
- 是 `/` 开头：当作本地命令，先走 `HermesCLI.process_command()`。

所以 CLI 层本身不负责模型推理。它主要负责：

- 显示 banner、欢迎语、spinner、tool activity。
- 用 `prompt_toolkit` 读取输入。
- 处理 `/model`、`/help`、`/tools`、`/queue`、`/compress` 这类本地命令。
- 把普通聊天消息交给 agent loop。
- 把 skill slash command 转成一条“特殊用户消息”，再交给 agent。

一句话概括：

```text
CLI = 终端 UI + slash command 分发器 + agent 调用入口
```

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `cli.py` | classic CLI 主体，`HermesCLI` 在这里，文件很大 |
| `hermes_cli/main.py` | 命令行入口、profile override、argparse |
| `hermes_cli/commands.py` | slash command 元数据中心，`COMMAND_REGISTRY` |
| `hermes_cli/completion.py` | slash command 补全、路径补全、自动建议 |
| `agent/skill_commands.py` | 把已安装 skill 暴露成 `/skill-name` |
| `hermes_cli/skin_engine.py` | CLI 皮肤配置 |
| `agent/display.py` | spinner、tool prefix、工具输出展示 |
| `hermes_cli/banner.py` | 启动 banner |

这里最重要的是三个点：

```text
COMMAND_REGISTRY 负责“有哪些命令”
process_command() 负责“命令怎么执行”
run() 负责“终端输入循环”
```

## 3. 一条最普通的聊天链路

假设用户输入：

```text
帮我看一下这个项目的结构
```

大概流程是：

```text
HermesCLI.run()
  -> prompt_toolkit 读到用户输入
  -> 发现不是 / 开头
  -> 放进 pending_input
  -> CLI 后台线程取出 message
  -> self.agent.run_conversation(user_message=message, ...)
  -> agent loop 调模型、调工具
  -> CLI 流式显示模型输出和工具进度
```

也就是说，普通聊天最终还是回到前面研究过的 `AIAgent.run_conversation()`。

CLI 层在这里主要做这些事：

- 保存当前会话历史。
- 注册 sudo、approval、secret capture 回调。
- 把模型输出用 Rich / prompt_toolkit 画到终端。
- 如果之前执行过 `/model` 或 `/reload-skills`，会把一段一次性 note 加到本轮用户消息前面。

代码里这个“一次性 note”很重要，例如模型切换后：

```text
[Note: model was just switched from old-model to new-model ...]

用户真正输入的消息
```

它不是改 system prompt，而是加到下一轮 user message 前面。这样可以少破坏 prompt caching。

## 4. 一条 slash command 链路

假设用户输入：

```text
/model gpt-5.5 --provider openai
```

大概流程是：

```text
HermesCLI.run()
  -> prompt_toolkit 读到 /model ...
  -> process_command("/model gpt-5.5 --provider openai")
  -> resolve_command("model")
  -> 找到 canonical command = "model"
  -> _handle_model_switch(...)
  -> 修改 CLI 当前 provider/model/base_url/api_mode
  -> 如果 agent 已经存在，也尝试 agent.switch_model(...)
  -> 打印切换结果
```

这一轮不会调用 LLM，因为 `/model` 是本地命令。

如果加了 `--global`：

```text
/model gpt-5.5 --provider openai --global
```

它还会调用 `save_config_value()`，把结果写回 `config.yaml`。不加 `--global` 就只影响当前 session。

## 5. COMMAND_REGISTRY：命令元数据中心

`hermes_cli/commands.py` 里有一个 `CommandDef`：

```python
@dataclass(frozen=True)
class CommandDef:
    name: str
    description: str
    category: str
    aliases: tuple[str, ...] = ()
    args_hint: str = ""
    subcommands: tuple[str, ...] = ()
    cli_only: bool = False
    gateway_only: bool = False
    gateway_config_gate: str | None = None
```

它描述一条命令“叫什么、怎么展示、有哪些别名、CLI/Gateway 是否可用”。

例如：

```text
/new      新会话
/model    切换模型
/tools    查看工具
/queue    agent 忙的时候排队追加消息
/steer    agent 正在跑的时候临时插入引导
/compress 手动压缩上下文
```

`COMMAND_REGISTRY` 是一个中心列表。下游很多东西都从这里派生：

- CLI `/help`
- CLI 自动补全
- Gateway 可识别命令集合
- Telegram command menu
- Slack subcommand mapping
- Gateway help 输出

所以新增一个命令时，第一步通常是往 `COMMAND_REGISTRY` 加一条 `CommandDef`。

但是注意：

```text
COMMAND_REGISTRY 只登记“有这个命令”
真正怎么执行，要去 cli.py 的 process_command() 加 handler
```

## 6. process_command()：真正的命令分发

`cli.py` 里的 `process_command()` 是 classic CLI 的命令主入口。

它做的第一件事是解析命令名：

```text
/m gpt-5.5
  -> base_cmd = "m"
  -> resolve_command("m")
  -> canonical = "model"
```

然后进入一大串分支：

```text
if canonical == "help":
    self.show_help()
elif canonical == "model":
    self._handle_model_switch(...)
elif canonical == "compress":
    self._handle_compress_command(...)
elif canonical == "queue":
    ...
elif canonical == "steer":
    ...
```

可以把它理解成：

```text
commands.py 负责“菜单”
cli.py::process_command() 负责“厨房”
```

菜单上写了 `/model`，但真正切模型的逻辑在 `cli.py`。

## 7. slash command 的 fallback 顺序

如果 `process_command()` 没有匹配到内置命令，它不会马上报错。它还有几层 fallback。

顺序大概是：

```text
1. 内置 COMMAND_REGISTRY 命令
2. quick_commands
   - exec: 执行一个本地 shell 命令
   - alias: 映射成另一条 slash command
3. plugin 注册的 CLI command
4. skill slash command，例如 /code-review
5. 前缀匹配提示，例如 /mod 可以提示 /model
6. 仍然找不到，提示 unknown command
```

### 例子：quick command alias

如果配置里有：

```yaml
quick_commands:
  gs:
    type: alias
    command: "/status"
```

用户输入：

```text
/gs
```

CLI 会把它递归转成：

```text
/status
```

再走一次 `process_command()`。

### 例子：quick command exec

如果配置里有：

```yaml
quick_commands:
  pwd:
    type: exec
    command: "pwd"
```

用户输入：

```text
/pwd
```

CLI 会在本地执行 `pwd`，把结果打印到终端。这个也不会进入 LLM。

## 8. Skill slash command：为什么不是改 system prompt

Hermes 允许安装很多 skills。每个 skill 有一个 `SKILL.md`。

`agent/skill_commands.py` 会扫描：

```text
~/.hermes/skills/
external skill dirs
```

然后把 skill 名变成 slash command。

例如一个 skill 叫：

```text
code review
```

可能会暴露成：

```text
/code-review
```

用户输入：

```text
/code-review 帮我检查最近改动
```

流程大概是：

```text
process_command()
  -> 没匹配到内置命令
  -> 在 _skill_commands 里找到 /code-review
  -> build_skill_invocation_message(...)
  -> 读取 SKILL.md 内容
  -> 拼上 skill directory、supporting files、用户补充指令
  -> 放进 _pending_input
  -> 下一轮作为 user message 交给 AIAgent
```

这里最关键的设计是：

```text
skill command 注入的是 user message，不是 system prompt
```

原因很实际：如果每次 `/skill` 都去改 system prompt，prompt caching 很容易失效。Hermes 更倾向于保持历史前缀稳定，只在当前轮 user message 里告诉模型：

```text
你现在要使用这个 skill。
这是 skill 内容。
这是 skill 所在目录。
用户额外要求是：帮我检查最近改动。
```

这样模型能看到 skill，缓存也不容易被打碎。

## 9. /reload-skills 的处理方式

`/reload-skills` 也很能说明 Hermes 对 prompt caching 的态度。

它做的事情不是重建 system prompt，而是：

```text
1. 重新扫描 skills 目录
2. 计算 added / removed / unchanged
3. 打印变化
4. 把一段一次性 note 存到 _pending_skills_reload_note
5. 下一轮用户消息前面带上这段 note
```

例子：

```text
/reload-skills
```

如果新增了一个 skill，下一轮用户输入：

```text
用新 skill 帮我处理一下
```

真正交给 agent 的可能类似：

```text
[USER INITIATED SKILLS RELOAD:

Added Skills:
    - my-new-skill: ...

Use skills_list to see the updated catalog.]

用新 skill 帮我处理一下
```

它没有在命令执行那一刻往 conversation history 里硬塞一条消息，而是等到下一轮正常 user turn 再附加。这样 role 顺序不会乱，缓存也更稳定。

## 10. /queue 和 /steer：agent 忙的时候怎么办

CLI 里有一个状态：

```text
self._agent_running
```

当 agent 正在跑工具循环时，用户可能还会继续输入。

### /queue

`/queue` 的语义是：

```text
当前 agent 还在跑，我先把一句话排队，等它跑完后再作为下一轮用户输入。
```

例子：

```text
/queue 跑完以后顺便总结一下改了哪些文件
```

如果 agent 正在运行，这条消息会进入 `_pending_input`，等当前任务结束再处理。

### /steer

`/steer` 更像“临时打方向盘”。

例子：

```text
/steer 不要继续改 UI，优先修测试
```

如果 agent 正在运行，CLI 会尽量直接调用：

```text
agent.steer(...)
```

这样这句话会被注入到下一次工具结果附近，让模型在同一轮 agent loop 里尽早看到。

区别可以这么理解：

```text
/queue = 下一轮再说
/steer = 当前轮尽快插一句
```

## 11. 输入、补全和 prompt_toolkit

classic CLI 使用 `prompt_toolkit`。

`HermesCLI.run()` 会创建一个输入区域：

```text
TextArea(
  multiline=True,
  history=FileHistory(...),
  completer=SlashCommandCompleter(...),
  complete_while_typing=True,
)
```

`SlashCommandCompleter` 会拿到：

- 内置 slash command
- command subcommands
- skill slash commands
- 路径补全
- 一些上下文补全

所以用户输入：

```text
/mo
```

可能会补全到：

```text
/model
```

输入：

```text
/skills in
```

可能会提示：

```text
install
```

技能命令也会动态出现，因为 completer 会通过：

```python
skill_commands_provider=lambda: _skill_commands
```

拿当前扫描到的 skill command。

## 12. Skin 和显示系统

Hermes 的 classic CLI 支持皮肤。

皮肤不是写死在代码里的，而是一组数据：

```text
hermes_cli/skin_engine.py
  -> built-in skins
~/.hermes/skins/*.yaml
  -> 用户自定义 skins
```

启动时：

```text
load_cli_config()
  -> 读 display.skin
  -> init_skin_from_config(CLI_CONFIG)
  -> set_active_skin(...)
```

皮肤能控制：

- banner 颜色
- welcome 文案
- prompt symbol
- spinner faces
- spinner verbs
- tool prefix
- response box label
- per-tool emoji

例子：

```yaml
display:
  skin: mono
```

或者在 CLI 里：

```text
/skin mono
```

`agent/display.py` 会懒加载 active skin，用它来决定工具输出前缀、spinner 样式等。

## 13. CLI 和 Gateway 为什么共用命令元数据

Hermes 不只有本地 CLI，还有 Telegram、Slack、Discord、Webhook 等 gateway。

这些入口都需要知道：

```text
有哪些 slash command？
命令描述是什么？
别名是什么？
哪些命令只允许 CLI？
哪些命令 gateway 也能用？
```

所以命令元数据统一放在 `COMMAND_REGISTRY`。

但执行逻辑是分开的：

```text
classic CLI:
  cli.py::process_command()

gateway:
  gateway/run.py 里的 command dispatch
```

这样做的好处是：

- 帮助文档和菜单不容易不一致。
- Telegram / Slack 这种平台菜单能自动生成。
- 新增别名时不用每个地方都改。

但是也有一个代价：

```text
COMMAND_REGISTRY 加了命令，只代表“大家看得到它”
CLI/Gateway 的执行 handler 仍然要分别实现
```

## 14. cli_only、gateway_only、gateway_config_gate

`CommandDef` 里有几个开关。

### cli_only

表示默认只在 CLI 可用。

例如一些明显依赖本地终端状态的命令，就不适合直接暴露给 gateway。

### gateway_only

表示只给 gateway 用，CLI 不展示。

### gateway_config_gate

这个比较特殊。

有些命令默认是 CLI-only，但如果用户配置打开了某个开关，也可以在 gateway 里用。

逻辑在 `commands.py`：

```text
_resolve_config_gates()
  -> 读 config.yaml
  -> 看 gateway_config_gate 指向的配置是否为 true
  -> 决定 gateway help/menu 里要不要展示
```

它的核心思想是：

```text
命令可以被 gateway 识别
但是否展示给用户，要看配置
```

## 15. 一个完整例子：用户调用 skill

假设用户安装了一个 skill：

```text
~/.hermes/skills/git-review/SKILL.md
```

frontmatter 里写：

```yaml
name: git review
description: Review git changes
```

启动 CLI 时，模块级代码会执行：

```text
_skill_commands = scan_skill_commands()
```

于是得到：

```text
/git-review
```

用户输入：

```text
/git-review 看一下最近有没有明显 bug
```

执行过程：

```text
1. prompt_toolkit 读到整行输入
2. process_command() 发现不是内置命令
3. 在 _skill_commands 找到 /git-review
4. build_skill_invocation_message()
5. 读 SKILL.md
6. 拼出一条很长的 user message
7. 放进 _pending_input
8. 后台 agent loop 把它当成用户消息处理
```

模型看到的不是简单一句“运行 git-review”，而是一整段：

```text
[Skill activation note]

SKILL.md 的正文内容

[Skill directory: /Users/.../.hermes/skills/git-review]
Resolve any relative paths ...

The user has provided the following instruction alongside the skill invocation:
看一下最近有没有明显 bug
```

这样模型既知道 skill 的规则，也知道用户这次想干什么。

## 16. 一个完整例子：运行中追加要求

假设用户输入：

```text
帮我修复测试
```

agent 开始跑，可能正在读文件、改代码、跑测试。

这时用户又输入：

```text
/steer 先不要处理 lint，优先处理 failing tests
```

CLI 判断：

```text
_agent_running = True
command = steer
```

然后直接把 steer 发给运行中的 agent。

如果用户输入的是：

```text
/queue 修完后顺便总结根因
```

那就是排到下一轮。当前 agent 不会马上改变方向。

这个设计解决的是本地 CLI 的真实交互问题：

```text
用户不一定会等 agent 完全停下来才补充信息
```

所以 CLI 需要区分：

- 当前轮插话
- 下一轮排队
- 本地命令立即执行
- 忙的时候禁止执行的命令

## 17. 设计观察

### 17.1 命令元数据集中，但执行分散

`COMMAND_REGISTRY` 统一了命令展示和平台菜单，这是好的。

但真正执行逻辑仍然分散在：

- `cli.py`
- `gateway/run.py`
- plugin command handler
- skill command builder

所以读代码时不要以为 `commands.py` 里有完整逻辑。它更像“路由表”和“帮助文档源”。

### 17.2 CLI 很重

`cli.py` 非常大，里面同时有：

- 配置加载
- UI 渲染
- 命令分发
- agent 生命周期
- model switch
- session 管理
- approval/sudo/secret callback
- skill reload
- clipboard image

这让它很强，但也让阅读成本比较高。

实际调研时建议只沿着一条链路读，比如：

```text
/model 怎么走？
/skills 怎么走？
普通消息怎么进 run_conversation？
```

不要从头到尾硬读。

### 17.3 Prompt caching 是很多命令设计的隐含约束

Hermes 很在意不要随便改过去的上下文，也不要中途重建 system prompt。

所以你会看到很多“下一轮 user message 前面加一次性 note”的设计：

- `/model` 切换后，下一轮告诉模型“刚切了模型”。
- `/reload-skills` 后，下一轮告诉模型“skill 列表变了”。
- skill slash command 作为 user message 注入，而不是改 system prompt。

这类设计看起来绕了一点，但目的是保持上下文前缀稳定。

### 17.4 CLI 和 Gateway 的命令需要同时考虑

新增命令时不能只看本地 CLI。

至少要问：

```text
这个命令 gateway 应该能用吗？
如果能用，gateway/run.py 是否有 handler？
Telegram/Slack 菜单是否应该展示？
agent 正在运行时，这个命令应该排队、打断、还是直接拒绝？
```

这也是为什么 `commands.py` 里会有 `cli_only`、`gateway_only`、`gateway_config_gate`。

## 18. 后续可以继续深挖的问题

下一步如果继续 CLI 模块，可以看这些点：

- `/skills install --now` 具体如何处理 cache invalidation。
- `/compress` 手动压缩命令如何接到 agent compression。
- `/retry`、`/undo`、`/rollback` 和 session history 的关系。
- prompt_toolkit 的 key binding、paste image、multiline 输入。
- TUI 入口和 classic CLI 的关系：哪些命令复用，哪些重新实现。

