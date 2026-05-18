---
created: '2026-04-30'
tags:
- hermes-agent
- agent-loop
status: draft
---

# Agent 主循环源码解析

> 研究日期：2026-04-30  
> 主文件：`run_agent.py`  
> 核心类：`AIAgent`

## 一、先给结论

Hermes 的 Agent Loop 可以先用一句非常直白的话理解：

> 模型先想下一步要不要用工具；如果要用工具，Hermes 执行工具并把结果塞回消息历史；模型再看工具结果继续决定下一步。直到模型不再调用工具，而是直接输出最终回答，或者被中断、预算耗尽、出错。

最小版循环就是：

```text
用户说一句话
  -> Hermes 把 system prompt + 历史 + 当前用户消息 + tools 发给模型
  -> 模型返回：
       A. tool_calls：Hermes 执行工具，把 tool result 追加进历史，然后继续问模型
       B. 普通文本：Hermes 认为任务结束，返回给用户
       C. 异常/截断/空响应：Hermes 尝试 retry、fallback、压缩或返回错误
```

但源码里的真实形态更接近：

```text
turn setup
  -> stable system prompt / session / memory / plugin context
  -> preflight compression
  -> while budget remains:
       build provider-specific API payload
       call model with streaming health checks
       retry / fallback / compress / continue on abnormal responses
       normalize response to assistant_message
       if tool_calls:
          validate / repair / execute / append tool results
          maybe compress
          continue
       else:
          finalize text response
  -> if budget exhausted: one toolless summary request
  -> persist session / trajectory / plugin post hooks
```

也就是说，`run_conversation()` 同时承担了五类职责：

1. 组装上下文。
2. 驱动 LLM/tool loop。
3. 处理 provider 异常、重试、fallback、压缩。
4. 管理 session/memory/plugin hooks。
5. 通过 callbacks 向 CLI/TUI/Gateway 输出事件。

## 二、一个具体例子

假设用户输入：

> “帮我看一下 `gateway/run.py` 里 `/stop` 是怎么处理的，总结一下。”

Agent Loop 大概会这样跑：

```text
第 1 次 API call
  输入：用户问题 + system prompt + 可用工具 schema
  模型决定：需要先找代码
  输出 tool_calls:
    - search_files(pattern="/stop", path="gateway")

Hermes 执行 search_files
  得到若干匹配行
  追加一条 tool message:
    role=tool, tool_call_id=xxx, content=搜索结果

第 2 次 API call
  输入：原问题 + search_files 的结果
  模型决定：需要读具体文件片段
  输出 tool_calls:
    - read_file(path="gateway/run.py", offset=6100, limit=180)

Hermes 执行 read_file
  追加 tool result

第 3 次 API call
  输入：原问题 + 搜索结果 + 文件片段
  模型决定：信息够了，不再用工具
  输出普通文本：
    “/stop 先经过 base adapter guard ...”

Hermes 看到没有 tool_calls
  -> 把这段文本作为 final_response
  -> 保存 session
  -> 返回给 CLI/TUI/Gateway
```

这就是最典型的 ReAct 风格：模型不直接拥有文件系统，它只能“请求工具”；Hermes 才真正执行工具，并把执行结果作为下一轮上下文交回模型。

## 三、什么时候停止

Loop 停止不是只有一种情况，源码里主要有这些出口：

| 停止原因 | 代码层表现 | 用户看到什么 |
|---|---|---|
| 正常完成 | 模型返回 assistant 文本且没有 `tool_calls` | 最终回答 |
| 用户中断 | `_interrupt_requested=True` | 停止当前 turn，Gateway/CLI 可接新消息 |
| 达到最大轮数 | `api_call_count >= max_iterations` 或 `iteration_budget.remaining <= 0` | Hermes 额外发一次无工具请求，让模型总结已完成内容 |
| Codex incomplete 过多 | Codex Responses 连续 `incomplete` 超限 | partial error |
| 无效工具名自纠失败 | 模型连续生成不存在的工具名 | partial error |
| tool arguments 截断 | 工具 JSON 参数明显被截断 | partial error，拒绝执行半截工具 |
| API 重试耗尽 | provider 错误重试/fallback 后仍失败 | 错误消息 |
| 压缩失败 | context overflow 且压缩无法降到阈值以下 | compression exhausted error |

最重要的正常停止条件是：

```text
assistant_message.tool_calls 为空
并且 assistant_message.content 有可见文本
```

这时 Hermes 会把这个 assistant message 追加进 `messages`，设置 `final_response`，然后跳出 while。

## 四、关键源码位置

| 功能 | 文件 / 函数 | 说明 |
|---|---|---|
| Agent 类 | `run_agent.py::AIAgent` | 核心运行时 |
| 初始化 | `AIAgent.__init__()` | provider、client、tools、session、memory、compression |
| 主循环 | `AIAgent.run_conversation()` | 一整个用户 turn |
| 简化入口 | `AIAgent.chat()` | 调用 `run_conversation()` 后只返回 `final_response` |
| tool 执行入口 | `_execute_tool_calls()` | 判断并发或串行 |
| 并发 tool | `_execute_tool_calls_concurrent()` | 安全批次走线程池 |
| 串行 tool | `_execute_tool_calls_sequential()` | 交互/有副作用工具默认路径 |
| 单 tool 调用 | `_invoke_tool()` | agent-level tools + registry dispatch |
| registry dispatch | `model_tools.handle_function_call()` | 普通工具统一分发 |
| 最大轮数处理 | `_handle_max_iterations()` | 额外一次无工具总结请求 |
| 上下文压缩 | `_compress_context()` | 历史压缩并可能创建新 session |

## 五、`AIAgent.__init__()` 做了什么

`AIAgent.__init__()` 很长，本质上是在把“平台入口传入的运行时参数”落成一个可执行 agent。

### 1. 保存运行时上下文

初始化最前面保存这些基础状态：

- `model`
- `max_iterations`
- `iteration_budget`
- `tool_delay`
- `platform`
- gateway 用户/聊天/thread 元信息
- `ephemeral_system_prompt`
- `skip_context_files`
- `load_soul_identity`
- `pass_session_id`
- `credential_pool`

`platform` 是后续 system prompt、session source、memory provider scope、UI callback 行为的重要输入。

### 2. 判断 API mode

Hermes 不是只走 OpenAI chat completions。`__init__()` 会根据显式 `api_mode`、`provider`、`base_url` 自动选择：

- `chat_completions`
- `codex_responses`
- `anthropic_messages`
- `bedrock_converse`

典型规则：

- `provider == "openai-codex"` 或 ChatGPT Codex URL -> `codex_responses`
- `provider == "anthropic"` 或 Anthropic URL -> `anthropic_messages`
- Bedrock runtime URL -> `bedrock_converse`
- 否则默认 `chat_completions`

GPT-5.x 或直接 OpenAI URL 还可能从 chat completions 自动升级到 Codex/Responses 路径；Azure OpenAI 是例外。

### 3. 初始化 provider client

这里要特别说明：Hermes 的 provider 层是“自己做了一层抽象”，但并不是所有请求都自己手写 HTTP。

它的结构是：

```text
AIAgent
  -> agent/transports/*        # Hermes 自己的格式转换/响应归一化层
  -> provider SDK / HTTP client # 底层真实请求
```

也就是说：

- Hermes 自己维护 `api_mode`、message/tool 格式转换、response normalize、retry/fallback、prompt caching、credential resolution。
- 但底层调用会使用官方或事实标准 SDK：
  - OpenAI-compatible / OpenRouter / Nous / local OpenAI wire：`openai.OpenAI` SDK。
  - Anthropic native：`anthropic` SDK，外面包了 `agent/anthropic_adapter.py`。
  - Bedrock：通过 Anthropic Bedrock / boto3 相关路径。
  - MCP server：依赖 `mcp` 包。
  - 一些 provider 兼容层会用 `httpx` 或自定义 adapter，例如 Gemini native facade。

所以更准确的说法是：**Hermes 自己做 provider 路由和协议适配，但没有完全避开官方 SDK。**

provider client 初始化分几条大路：

```text
api_mode == anthropic_messages
  -> agent.anthropic_adapter.build_anthropic_client()
  -> 处理 native Anthropic / Bedrock Anthropic / 第三方 Anthropic-compatible

api_mode == bedrock_converse
  -> boto3 / Bedrock Converse

其他 OpenAI-wire provider
  -> 显式 api_key/base_url
  -> 或 agent.auxiliary_client.resolve_provider_client()
  -> _create_openai_client()
```

这一段还处理很多 provider-specific header：

- OpenRouter headers
- GitHub Copilot headers
- Kimi user-agent
- Qwen Portal headers
- ChatGPT Codex Cloudflare headers
- Claude on OpenRouter 的 fine-grained tool streaming beta header

`agent/transports/base.py` 定义了统一接口：

```text
convert_messages()
convert_tools()
build_kwargs()
normalize_response()
validate_response()
map_finish_reason()
```

所以 `run_conversation()` 不直接关心每个 provider 的原始响应长什么样，而是拿到统一的 `assistant_message`。

### 4. 初始化 fallback chain

`fallback_model` 现在支持单 dict 或 list：

```text
fallback_model:
  dict -> [dict]
  list[dict] -> fallback chain
```

主循环遇到 rate limit、overload、连接失败、空响应等场景时，会调用 `_try_activate_fallback()` 切换到 fallback provider/model。

### 5. 加载工具 surface

核心调用：

```python
self.tools = get_tool_definitions(
    enabled_toolsets=enabled_toolsets,
    disabled_toolsets=disabled_toolsets,
    quiet_mode=self.quiet_mode,
)
self.valid_tool_names = {tool["function"]["name"] for tool in self.tools}
```

这一步把 registry + toolsets + availability checks 的结果固定到 agent 实例里。

Gateway 复用 agent cache 的原因也在这里：tool schemas 一旦变化，prompt cache prefix 就可能失效。

### 6. 初始化 session 和持久状态

如果外部传入 `session_db`，`AIAgent` 会创建/登记 session：

```text
session_id
source = platform / HERMES_SESSION_SOURCE
model
model_config
parent_session_id
```

同时也维护 JSON session log：

```text
~/.hermes/sessions/session_<session_id>.json
```

注意：SQLite 创建 session 失败不会把 `_session_db` 置空。源码注释说明这是为了避免短暂 SQLite lock 导致 `session_search` 永久不可用。

### 7. 初始化 memory

Hermes 有两套相关机制：

1. 内置 memory store：`tools.memory_tool.MemoryStore`
2. 外部 memory provider plugin：`agent.memory_manager.MemoryManager`

如果 `skip_memory=False`：

- 从 config 读 `memory.memory_enabled`、`user_profile_enabled`、`nudge_interval`。
- 加载本地 memory/user profile。
- 如果配置了 `memory.provider`，加载对应 memory plugin。
- 初始化 provider 时注入 session、platform、user、chat、thread、gateway_session_key、profile 等 scope。
- provider 自己暴露的 tool schemas 会追加到 `self.tools`。

Cron 会显式 `skip_memory=True`，原因是定时任务 system prompt 不应该污染用户表征。

### 8. 初始化 compression

`AIAgent` 会读 `compression` 配置并创建 context compressor。

后续有三类压缩触发：

- turn 前 preflight compression。
- API 报 context overflow 后压缩重试。
- tool 执行后基于 token usage 判断是否压缩。

## 六、`run_conversation()` 的阶段拆解

`run_conversation()` 是一个完整用户 turn。可以拆成 9 个阶段。

### 阶段 1：turn 前清理和标识

入口参数：

```python
run_conversation(
    user_message,
    system_message=None,
    conversation_history=None,
    task_id=None,
    stream_callback=None,
    persist_user_message=None,
)
```

开头会做：

- 安装 safe stdio，防止 broken pipe 导致 agent 崩溃。
- 设置日志 session context。
- 如果上轮激活了 fallback，本轮先恢复 primary runtime。
- 清理 user message 里的非法 surrogate。
- 生成/保存 `effective_task_id`，用于 terminal/browser/env 隔离。
- 重置本 turn 的 retry counters。
- 清理 stale connections。
- 重置 `iteration_budget = IterationBudget(self.max_iterations)`。

这里有一个细节：`iteration_budget` 是每个 turn 重新建的，避免上一轮 subagent/tool 使用消耗下一轮预算。

### 阶段 2：准备 messages 和 user turn

```text
messages = list(conversation_history) if conversation_history else []
hydrate todo store from history
append {"role": "user", "content": user_message}
```

`prefill_messages` 不会写入 `messages`，只在 API-call-time 插入。这保证 few-shot/prefill 不会进入 session DB 或 trajectory。

`persist_user_message` 允许 API-facing user message 和持久化 transcript 不一致。例如语音模式可以给 API 临时加前缀，但不污染历史。

### 阶段 3：system prompt 缓存

这一段对 prompt caching 很关键。

如果 `self._cached_system_prompt is None`：

1. 如果是继续旧 session 且 DB 里有 `system_prompt`，直接复用旧 prompt。
2. 否则调用 `_build_system_prompt(system_message)` 构建。
3. 新 session 触发 `on_session_start` plugin hook。
4. 把 system prompt snapshot 写入 SQLite。

核心原则：

> 继续会话时不重新从磁盘加载 memory / rules 来重建 system prompt，因为那会改变 cache prefix，破坏 Anthropic prompt caching。

只有 context compression 这类明确改变历史边界的事件才会重建/更新 system prompt。

### 阶段 4：preflight compression

如果历史已经很大，进入主循环前会估算：

```text
messages + system prompt + tool schemas
```

如果超过 `context_compressor.threshold_tokens`，最多压缩 3 次。

压缩后会把 `conversation_history = None`，这是为了后续 flush 到 DB 时把压缩后的新 session 全量写入，而不是按旧 history 长度跳过前面的消息。

### 阶段 5：plugin / memory 上下文注入

主循环前会触发 `pre_llm_call` hook。

插件返回的 context、外部 memory provider prefetch 结果，都不会注入 system prompt，而是在每次 API call 构造 `api_messages` 时临时追加到当前 user message。

原因同样是 prompt cache：

```text
system prompt = Hermes 稳定前缀
plugin/memory recall = 当前 user message 的 ephemeral context
```

## 七、主 while 循环

主循环条件：

```python
while (
    api_call_count < self.max_iterations
    and self.iteration_budget.remaining > 0
) or self._budget_grace_call:
```

每次循环代表一次模型 API call。

### 1. interrupt 与预算

每轮开头：

- 如果 `_interrupt_requested`，直接跳出。
- `api_call_count += 1`
- `iteration_budget.consume()`
- 如果是 `_budget_grace_call`，允许额外一次，不再 consume。

`_budget_grace_call` 的语义是：预算已耗尽后，允许模型最后再说一次话，不鼓励中途提前放弃。

### 2. step callback

如果入口提供 `step_callback`，每轮会把上一轮工具和结果整理出来传给外层。

Gateway hooks 可以用这个生成 agent step event。

### 3. `/steer` drain

`/steer` 不直接插入新的 user message，因为那会破坏 role alternation。

它的策略是：

- 如果已经有 tool result，就把 steer 文本追加到最近的 tool message。
- 如果还没有 tool result，就先挂起，等后续 tool batch 完成后再注入。

### 4. 构造 API messages

每轮都会基于内部 `messages` 重新构造 `api_messages`：

```text
copy messages
  -> 当前 user message 注入 memory/plugin context
  -> assistant reasoning 转成 provider API 需要的字段
  -> 移除内部字段 reasoning / finish_reason / _thinking_prefill
  -> strict provider 下清理 tool_calls 特殊字段
  -> prepend system prompt
  -> insert prefill_messages
  -> apply Anthropic cache_control
  -> sanitize orphan tool results
  -> drop thinking-only assistant turns
  -> normalize whitespace / tool-call JSON
  -> sanitize surrogate chars
```

这一层很重要：内部 session history 和 API payload 不是同一个东西。Hermes 保留更多内部字段，但发给 provider 前会做兼容性清理。

### 5. API call 和重试内循环

每轮 API call 内部还有一个 retry loop：

```python
while retry_count < max_retries:
    build api_kwargs
    pre_api_request hook
    streaming or non-streaming call
    validate response
    map finish_reason
    break / retry / fallback / compress
```

默认优先 streaming，即使没有 UI/TTS consumer。注释里说明这是为了健康检查：

- stale stream detection
- read timeout
- provider 长时间 SSE ping 但无实际响应时能中断

只有 mock client、Copilot ACP、明确禁用 streaming 等场景会走 non-streaming。

异常处理里会分类：

- rate limit
- overload
- auth / billing
- context overflow
- payload too large
- long context tier
- thinking signature
- local validation error
- transient network / malformed response

处理动作包括：

- jittered backoff
- respect `Retry-After`
- fallback provider
- rebuild primary transport
- context compression 后重启本轮
- auth refresh retry
- image shrink retry
- 输出用户可理解的失败提示

这部分相当于 OpenClaw “外层容错循环”的一部分，但 Hermes 没有拆成独立外层函数，而是嵌在 `run_conversation()` 内。

## 八、响应归一化后的三大分支

API response 成功后，会通过 transport normalize 成统一的 `assistant_message`。随后进入三大分支：

### 分支 A：Codex incomplete

`api_mode == "codex_responses"` 且 `finish_reason == "incomplete"`：

- 构建 interim assistant message。
- 如果不是重复 interim，就 append 到 `messages`。
- 最多继续 3 次。
- 仍 incomplete 则返回 partial error。

### 分支 B：有 tool_calls

这是核心 tool-use 路径。

先做 guardrails：

1. 自动修复近似工具名。
2. 校验工具名是否在 `self.valid_tool_names`。
3. 无效工具名最多让模型自纠 3 次。
4. 校验 tool arguments 是否有效 JSON。
5. 如果 JSON 看起来是截断的，直接返回 partial，拒绝执行。
6. 非截断 JSON 错误会先 retry，耗尽后注入 tool error 让模型自纠。
7. 限制 `delegate_task` 调用数量。
8. 对重复 tool call 去重。

然后：

```text
assistant_msg = _build_assistant_message(...)
messages.append(assistant_msg)
emit interim assistant
flush streaming display
_execute_tool_calls(...)
maybe refund execute_code-only iteration
maybe compress
save session log
continue while loop
```

如果 assistant 同时有 content 和 tool_calls，Hermes 会把 content 记录为 fallback final response。这处理一种常见模式：模型已经回答了用户，同时调用 memory/todo/skill_manage 做“收尾 housekeeping”。如果后续 tool round 后模型空回复，可以回退到这段内容。

### 分支 C：没有 tool_calls

这就是 final response 路径：

- `final_response = assistant_message.content or ""`
- 检查是否只有 think block 没有可见内容。
- 处理 partial streamed response。
- reasoning-only 会尝试 thinking prefill continuation。
- empty response 最多 retry 3 次。
- empty 后可尝试 fallback provider。
- 成功则 strip think blocks，构建 final assistant message，append 到 messages，break。

## 九、压缩都在哪些环节做

当前研究文档里还没有单独的“压缩模块”文档，所以先把压缩放在主循环里讲完整。原因也很直接：压缩不是一个孤立功能，它会影响下一轮 LLM call、session 切分、memory provider、context engine、todo snapshot 和 prompt caching。

代码上可以按这几个入口看：

```text
run_agent.py
  AIAgent._compress_context()
    -> 真正由 agent loop 调用的统一入口

agent/context_compressor.py
  ContextCompressor.should_compress()
    -> 判断要不要压

  ContextCompressor.compress()
    -> 真正压缩 messages

  ContextCompressor._prune_old_tool_results()
    -> 先便宜地裁剪旧工具结果，不调用模型

  ContextCompressor._generate_summary()
    -> 调辅助模型生成结构化摘要

agent/manual_compression_feedback.py
  -> 手动 /compress 时给用户看的反馈文案
```

一句话说：**主循环决定什么时候压，`ContextCompressor` 决定怎么压，`AIAgent._compress_context()` 负责压完以后怎么接回整个系统。**

### 1. 触发点一：Turn 前压缩

发生在进入主 while loop 之前。

典型场景：

```text
用户 resume 一个很长的旧 session
或者刚切到一个上下文更小的模型
```

Hermes 会先粗略估算：

```text
system prompt + conversation messages + tool schemas
```

如果超过压缩阈值，就先压缩，再进行第 1 次 API call。这样可以避免第一下请求就被 provider 拒绝。

直白理解：

```text
还没问模型之前，先检查包袱是不是太大。
太大就先整理行李，再出发。
```

### 2. 触发点二：API 报错后的压缩

发生在 API call 抛出 context length / payload too large 一类错误时。

流程是：

```text
API call
  -> provider 返回 context overflow
  -> Hermes 调 _compress_context()
  -> 压缩成功后回到当前 API call 重新试
```

这是“被动压缩”：预估可能没拦住，但 provider 明确告诉 Hermes 上下文太长，于是压缩后重试同一轮。

### 3. 触发点三：Tool 执行后的压缩

发生在工具执行完、tool result 已经追加到 `messages` 之后。

原因是工具结果经常会让上下文突然变大，例如：

- `read_file` 读了大文件。
- `terminal` 输出了大量日志。
- 搜索或网页抽取返回了长文本。
- 多个 tool 并行返回，结果一起塞回 messages。

这时 Hermes 会根据上一轮 provider 返回的真实 prompt tokens，或者本地 rough estimate，判断是否超过阈值。如果超过，就先压缩，再进入下一轮 API call。

直白理解：

```text
模型让工具去查资料。
工具把资料搬回来。
Hermes 发现资料太多，会先整理成摘要，再让模型继续看。
```

### 4. 触发点四：手动 `/compress`

CLI/Gateway/TUI 里可以手动触发 `/compress`。它不属于自动 while loop 的普通分支，但最终仍然复用同一套压缩能力。

`/compress <focus>` 可以带一个 focus topic。比如：

```text
/compress agent loop
```

这会告诉压缩器：摘要时优先保留和 `agent loop` 相关的内容。代码里会把更多摘要预算分给 focus 相关信息，类似 Claude Code 的 `/compact <focus>`。

### 5. 触发点五：Gateway 的长会话 hygiene

Gateway 场景里还有一个外围清理逻辑：消息进入 agent 前，如果历史太长，gateway 会做 session hygiene 式的自动压缩。它本质上也是构造一个 agent，然后调用 `_compress_context()` 这条路径。

这个逻辑不改变主循环的本质，只是触发位置更靠外：

```text
消息平台收到新消息
  -> 读取历史 transcript
  -> 发现历史太长
  -> 先压缩历史
  -> 再启动本轮 agent
```

### 6. 压缩到底做了什么

真正压缩器在 `agent/context_compressor.py::ContextCompressor`。它不是简单删旧消息，而是分几步做。

#### 第一步：判断是否真的有东西可压

`has_content_to_compress()` 会先检查 messages 是否足够长。太短的对话不会压，因为压缩本身也有成本，而且摘要可能比原文还不划算。

#### 第二步：先裁剪旧 tool 结果

这是便宜预处理，不需要调用模型。

```text
旧 tool result 很长
  -> 换成一行摘要或 placeholder

重复 tool result
  -> 去重

很大的 tool call arguments
  -> 截短，但尽量保持 JSON 合法
```

这一步的目标是：能不调用模型就先省一点 token，尤其是 terminal 输出、文件内容、网页内容这类大块文本。

更具体地说，Hermes 对工具相关消息有几种处理方法。

**方法 A：旧 tool result 改成一行摘要**

如果一个旧 tool result 超过 200 字，并且不在受保护的最近 tail 区域里，它不会继续保留完整输出，而是变成一行说明。

例子：

```text
原始 tool result:
  role=tool
  content="pytest 输出的 800 行日志..."

压缩预处理后:
  role=tool
  content="[terminal] ran `pytest tests/` -> exit 1, 800 lines output"
```

不同工具会生成不同摘要：

```text
[terminal] ran `npm test` -> exit 0, 47 lines output
[read_file] read config.py from line 1 (3,400 chars)
[write_file] wrote to app.py (120 lines)
[search_files] content search for 'compress' in agent/ -> 12 matches
[patch] replace in run_agent.py (2,300 chars result)
[web_search] query='...' (12,000 chars result)
[web_extract] https://example.com (+2 more) (40,000 chars)
[delegate_task] '分析 provider 层...' (8,000 chars result)
```

也就是说：**tool result 会被调整**。老的、大的工具输出会从“完整原文”变成“这个工具做了什么、目标是什么、结果大概多大”。

**方法 B：重复 tool result 去重**

如果同一个内容被工具读了多次，比如重复 `read_file` 同一个文件，Hermes 会保留最新那份完整结果，旧的重复结果改成：

```text
[Duplicate tool output — same content as a more recent call]
```

这样可以避免同一份文件内容在上下文里重复出现 3 到 5 次。

**方法 C：长 tool_call arguments 安全截断**

有些工具的参数本身就很大，不只是结果大。典型例子：

```text
write_file(path="foo.md", content="五万字内容...")
patch(path="foo.py", old="很长...", new="很长...")
```

这些内容存放在 assistant 消息的 `tool_calls[].function.arguments` 里。Hermes 会解析 JSON，然后只截断里面过长的字符串值，最后重新序列化成合法 JSON。

它不会直接把 JSON 字符串硬切掉，因为那样会变成坏格式：

```text
{"path": "foo.md", "content": "很长很长...
```

坏 JSON 会导致后续 provider 直接 400，所以这里的实现是“结构化截断”，不是字符串暴力截断。

**方法 D：给摘要模型看的内容再截断一次**

即使 tool result 已经进入“要被摘要的 middle 区域”，Hermes 也不会把每条消息完整塞给摘要模型。`_serialize_for_summary()` 会限制单条消息体：

```text
每条 content 最多约 6000 chars
  -> 保留前 4000 chars
  -> 中间插入 ...[truncated]...
  -> 保留后 1500 chars

tool arguments 最多约 1500 chars
  -> 保留前 1200 chars
  -> 后面 ...
```

这样摘要模型仍然能看到路径、命令、错误开头、日志结尾，但不会被某一条巨大输出撑爆。

**方法 E：中间历史结构化摘要**

这是最主要的“压缩”。被选中的 middle turns 会被序列化成带角色标签的文本：

```text
[USER]: ...

[ASSISTANT]: ...
[Tool calls:
  read_file({"path":"run_agent.py","offset":1})
]

[TOOL RESULT call_xxx]: ...
```

然后辅助模型把它压成结构化 handoff summary。

**方法 F：压缩后修复 tool_call / tool_result 配对**

OpenAI 风格 tool calling 要求非常严格：

```text
assistant 有 tool_call id=abc
后面必须有 role=tool tool_call_id=abc
```

压缩切 head/middle/tail 时可能切到工具调用组边界。Hermes 会尽量移动边界，避免把一个工具组切开；压完后还会做 sanitizer：

```text
如果 tool result 还在，但原来的 assistant tool_call 被压掉
  -> 删除这个孤儿 tool result

如果 assistant tool_call 还在，但对应 tool result 被压掉
  -> 补一个 stub result:
     [Result from earlier conversation — see context summary above]
```

这个细节很重要，否则下一轮 API call 会因为 tool message 格式不合法直接失败。

#### 第三步：切三段

```text
head: 最开头几条，默认 protect_first_n = 3
middle: 中间大段历史，用来压缩
tail: 最近上下文，按 token 预算和消息数量保护
```

保留 head 是为了保住最早的任务背景。保留 tail 是为了让模型继续当前任务时不失忆。真正被摘要的是 middle。

注意 tail 不是死板只保留 20 条。代码会结合最近 token 预算来找边界，尽量保留最近约一段有效上下文。

#### 第四步：把 middle 变成结构化摘要

`_generate_summary()` 会调用辅助模型生成摘要。摘要不是自由发挥，而是要求保留这些内容：

- 当前任务是什么。
- 用户目标是什么。
- 约束和偏好。
- 已经完成了什么。
- 当前状态是什么。
- 哪些事情正在进行。
- 哪些问题被阻塞。
- 做过哪些关键决定。
- 哪些问题已经解决。
- 用户还有哪些待办要求。
- 相关文件和位置。
- 剩余工作。
- 不能丢的关键上下文。

摘要还会遵守几个规则：

- 尽量用原对话同一种语言。
- 不回答旧问题，只做交接记录。
- 移除或隐藏明显 secret。
- 如果已经压缩过，会基于旧 summary 做迭代更新。
- 如果有 focus topic，会优先保留 focus 相关内容。

#### 第五步：重新组装 messages

```text
head messages
  + compression summary
  + recent tail messages
```

然后还会清理不完整的 tool 调用配对，避免出现这种坏上下文：

```text
assistant 说要调用 tool
但对应 tool result 被压掉了
```

或者反过来：

```text
tool result 还在
但原来的 assistant tool_call 不见了
```

这类孤儿消息会破坏 OpenAI tool-call message 格式，所以压缩后会做修复。

#### 第六步：压完以后接回 agent 系统

`AIAgent._compress_context()` 不只是替换 messages。它还会做系统级收尾：

```text
memory_manager.on_pre_compress(messages)
  -> 给 memory provider 一个压缩前同步机会

ContextCompressor.compress(...)
  -> 生成压缩后的 messages

追加 todo snapshot
  -> 如果当前有 todo 状态，压缩后补回去

invalidate/rebuild system prompt
  -> 压缩是少数允许改变上下文边界的地方

end old session(reason="compression")
create new session(parent_session_id=old_session_id)
  -> 旧 transcript 仍可查，新 session 继续跑

context_engine.on_session_start(boundary_reason="compression")
memory_provider.on_session_switch(reason="compression")
  -> 通知外部上下文和记忆系统换边界

reset flush cursor / file read dedup cache
  -> 避免压缩前后的状态错位
```

所以压缩不是“把数组缩短”这么简单。它会切出一个新的 session 边界。

### 7. 如果一次 trace 调了 30 轮工具，会是什么样

先看没有压缩时的形态。假设模型连续做了 30 轮工具调用，每一轮通常至少有两类消息：

```text
assistant:
  tool_calls = [
    {"id":"call_01","function":{"name":"read_file","arguments":"{\"path\":\"run_agent.py\"}"}}
  ]

tool:
  tool_call_id = "call_01"
  content = "run_agent.py 的大段内容..."
```

30 轮以后，messages 可能长这样：

```text
system
user: 请调研 agent loop

assistant tool_call: search_files("run_conversation")
tool result: 100 行搜索结果

assistant tool_call: read_file("run_agent.py", offset=1)
tool result: 6000 行文件内容片段

assistant tool_call: read_file("run_agent.py", offset=600)
tool result: 6000 行文件内容片段

assistant tool_call: search_files("_compress_context")
tool result: 80 行搜索结果

assistant tool_call: read_file("agent/context_compressor.py")
tool result: 9000 行文件内容片段

...

第 30 轮 assistant tool_call: patch("01-Agent主循环.md")
第 30 轮 tool result: patch 成功

assistant: 准备继续回答
```

如果不压缩，下一次 LLM call 会把这 30 轮工具调用和结果几乎都带上。问题有三个：

- token 很快爆掉。
- 很多旧 tool result 对当前任务已经只是背景，不需要原文。
- provider 要求 tool_call/tool_result 配对完整，不能随便删一半。

触发压缩后，Hermes 大概会变成这样：

```text
system
  + 附加一段 note：部分早期对话已被 compact 成 handoff summary

user: 请调研 agent loop

summary:
  ## Active Task
  用户正在要求补充 Hermes agent loop 文档中关于压缩的细节。

  ## Completed Actions
  1. SEARCH run_conversation — 找到主循环入口 [tool: search_files]
  2. READ run_agent.py — 确认 _compress_context 调用位置 [tool: read_file]
  3. READ agent/context_compressor.py — 确认压缩算法 [tool: read_file]
  4. PATCH 01-Agent主循环.md — 已补充压缩触发点 [tool: patch]

  ## Relevant Files
  - run_agent.py: AIAgent.run_conversation(), _compress_context()
  - agent/context_compressor.py: ContextCompressor.compress()
  - research-docs/research/Hermes源码研究/01-Agent主循环.md: 正在编辑

  ## Critical Context
  旧 tool result 会被一行摘要替换；重复结果会去重；长 arguments 会 JSON 安全截断。

最近 tail 原文:
  assistant tool_call: read_file("agent/context_compressor.py", offset=620)
  tool result: 最近这次源码片段仍保留完整或近似完整
  user: 他们都有哪些压缩方法，压缩的细节也没有说清楚...
```

注意这里有两层变化：

```text
第一层：进入 summary 之前，旧 tool result 已经被轻量裁剪
第二层：middle 区域整体被 summary 替换
```

所以 30 轮工具 trace 压缩后，不是 30 轮都消失，也不是 30 轮都原样保留，而是：

```text
最早关键任务背景保留
中间大量工具轨迹变成结构化 summary
最近几轮工具调用和结果保留原文
不完整的 tool_call/tool_result 配对被修复
```

这个设计是为了让模型继续当前任务，而不是让它重新审计完整历史日志。

### 8. 一个更小的压缩例子

上面的 30 轮 trace 更接近真实情况，但第一次理解时可以先看一个小例子。

假设压缩前 messages 是这样：

```text
0 system:
  你是 Hermes...

1 user:
  帮我分析 agent loop。

2 assistant:
  我要先搜索入口。
  tool_calls = [
    call_1: search_files({"pattern":"run_conversation","path":"."})
  ]

3 tool(call_1):
  {"total_count": 12, "matches": "很多搜索结果..."}   # 1200 chars

4 assistant:
  找到 run_agent.py 了，继续读取。
  tool_calls = [
    call_2: read_file({"path":"run_agent.py","offset":10000})
  ]

5 tool(call_2):
  "run_agent.py 的大段源码......"                    # 30000 chars

6 assistant:
  我已经确认主循环位置。接下来查看压缩器。
  tool_calls = [
    call_3: read_file({"path":"agent/context_compressor.py"})
  ]

7 tool(call_3):
  "context_compressor.py 的大段源码......"           # 25000 chars

8 user:
  压缩方法讲细一点，举个例子。
```

如果触发压缩，第一步不是马上 summary，而是先处理旧 tool result。

假设最近 tail 要保护 `6,7,8`，那么旧的 `3,5` 会先变短：

```text
3 tool(call_1):
  "[search_files] content search for 'run_conversation' in . -> 12 matches"

5 tool(call_2):
  "[read_file] read run_agent.py from line 10000 (30,000 chars)"

7 tool(call_3):
  "context_compressor.py 的大段源码......"           # 最近 tail，保留
```

然后切三段：

```text
head:
  0 system
  1 user
  2 assistant/tool_call call_1

middle:
  3 tool(call_1) 的一行摘要
  4 assistant/tool_call call_2
  5 tool(call_2) 的一行摘要

tail:
  6 assistant/tool_call call_3
  7 tool(call_3) 原文
  8 user 最新请求
```

middle 会被喂给摘要模型。摘要模型看到的不是 30000 字源码，而是已经变短后的工具轨迹：

```text
[TOOL RESULT call_1]:
[search_files] content search for 'run_conversation' in . -> 12 matches

[ASSISTANT]:
找到 run_agent.py 了，继续读取。
[Tool calls:
  read_file({"path":"run_agent.py","offset":10000})
]

[TOOL RESULT call_2]:
[read_file] read run_agent.py from line 10000 (30,000 chars)
```

生成的 summary 可能长这样：

```text
[CONTEXT COMPACTION — REFERENCE ONLY]

## Active Task
用户要求把压缩方法讲细，并用例子说明。

## Completed Actions
1. SEARCH run_conversation — 找到 agent loop 入口在 run_agent.py [tool: search_files]
2. READ run_agent.py — 读取主循环相关区域，确认 run_conversation() 管理 API/tool loop [tool: read_file]

## Relevant Files
- run_agent.py: agent 主循环和 _compress_context() 调用位置

## Critical Context
- 旧 tool result 会先被一行摘要替换。
- 最近 tail 中的 tool result 会尽量保留原文。
```

最终发给下一轮模型的 messages 大概变成：

```text
0 system:
  你是 Hermes...
  [Note: Some earlier conversation turns have been compacted...]

1 user:
  帮我分析 agent loop。

2 assistant:
  我要先搜索入口。
  tool_calls = [call_1: search_files(...)]

3 assistant 或 user:
  [CONTEXT COMPACTION — REFERENCE ONLY]
  ## Active Task ...
  ## Completed Actions ...
  ## Critical Context ...

4 assistant:
  我已经确认主循环位置。接下来查看压缩器。
  tool_calls = [call_3: read_file(...)]

5 tool(call_3):
  "context_compressor.py 的大段源码......"           # 最近结果仍在

6 user:
  压缩方法讲细一点，举个例子。
```

这里最关键的是：

```text
call_2 那次 read_file 的 30000 chars 原文不再保留。
它的信息被折叠进 summary：
  “读过 run_agent.py，确认了主循环位置”

call_3 是最近 tail，仍然保留原文。
因为下一轮模型很可能马上要用它回答用户问题。
```

如果边界切得不巧，比如保留了 `assistant tool_call call_2`，但 `tool(call_2)` 被压掉了，Hermes 会补一个 stub：

```text
tool(call_2):
  "[Result from earlier conversation — see context summary above]"
```

如果反过来，只剩 `tool(call_2)`，但对应 assistant tool_call 被压掉，Hermes 会删除这个孤儿 tool result。

所以压缩后的上下文既要省 token，也要保持 tool-call 格式合法。

### 9. 一个具体对话例子

假设用户让 Hermes 调研一个项目：

```text
第 1 轮：用户问“分析 agent loop”
第 2 轮：模型 read_file(run_agent.py)
第 3 轮：模型 rg("_compress_context")
第 4 轮：模型 read_file(agent/context_compressor.py)
第 5 轮：terminal 输出了很多搜索结果
...
到第 40 轮：messages 已经很大
```

压缩前可能是：

```text
system
user: 分析项目
assistant/tool: 早期大量搜索和文件读取
assistant/tool: 中期大量 run_agent.py 分析
assistant/tool: 中期大量 context_compressor.py 分析
user: 继续 agent 主循环
assistant/tool: 最近几轮正在写文档
```

压缩后会变成：

```text
system
最开始几条关键消息
summary:
  - 用户正在研究 Hermes agent loop
  - 已确认主循环在 run_conversation()
  - 工具执行后可能触发压缩
  - 压缩器在 agent/context_compressor.py
  - 已写入 research-docs/research/Hermes源码研究/01-Agent主循环.md
  - 还需要补充压缩触发和方法
最近几轮原文:
  user: 我们有专门的压缩 文档模块吗？
  assistant/tool: 最近的源码定位和编辑上下文
```

模型下一轮看到的不是完整 40 轮历史，而是：

```text
早期关键背景 + 中间摘要 + 最近原文
```

这样它既不会因为上下文爆掉失败，也还能继续当前任务。

### 10. 压缩失败和反复压缩

压缩也可能失败，比如辅助模型调用失败。Hermes 不会直接丢掉所有上下文，而是插入一个 fallback marker，说明中间历史被压缩但摘要生成失败。这不是最佳结果，但比请求直接崩掉更可控。

还有一个反复压缩保护：

```text
如果最近两次压缩节省都小于 10%
  -> should_compress() 会倾向于不再继续自动压
  -> 提示用户考虑 /new 或 /compress <focus>
```

这是为了避免“压了也省不了多少 token，但每轮都在压”的循环。

如果同一会话已经压缩多次，Hermes 还会给出提醒。因为多次摘要叠摘要后，细节丢失风险会升高。

### 11. 压缩不是什么

它不是：

- 不是简单删掉最旧消息。
- 不是只截断 tool output。
- 不是把最新用户请求压掉。
- 不是每一轮都重建系统提示词。
- 不是直接销毁旧 session。

更准确地说，它是：

```text
在保留开头任务背景和最近原文的前提下，
把中间历史变成可交接的结构化摘要，
然后切出新的 session 边界继续运行。
```

### 12. 压缩阈值

默认配置大概是：

```text
threshold = context_length * 0.50
target_ratio = 0.20
protect_first_n = 3
protect_last_n = 20
```

意思是：当请求接近模型上下文窗口的 50% 时就开始压缩，不等到快爆掉。摘要目标长度按阈值的一定比例计算。

## 十、Tool 执行路径

### 1. 并发还是串行

入口：

```python
_execute_tool_calls(assistant_message, messages, effective_task_id, api_call_count)
```

它先调用 `_should_parallelize_tool_batch(tool_calls)` 判断是否安全并发。

并发判断在 `run_agent.py::_should_parallelize_tool_batch()`，规则非常保守。

它的判断顺序是：

```text
1. 只有 0/1 个 tool call
   -> 串行

2. 任何 tool 在 _NEVER_PARALLEL_TOOLS
   -> 串行

3. 每个 tool 的 arguments 必须能解析成 dict
   -> 解析失败就串行

4. 如果是路径相关工具 read_file/write_file/patch
   -> 计算目标 path
   -> 路径缺失或路径相互重叠就串行

5. 其他工具必须在 _PARALLEL_SAFE_TOOLS
   -> 不在白名单就串行

6. 全部通过
   -> 并行，最多 8 个 worker
```

并发条件总结：

- tool call 数量大于 1。
- 不包含 `_NEVER_PARALLEL_TOOLS`，目前 `clarify` 永远不并发。
- read-only 工具在 `_PARALLEL_SAFE_TOOLS` 内。
- 文件工具 `read_file/write_file/patch` 只有路径不重叠才可并发。
- 其他未知/有副作用工具默认串行。

最大并发 worker 数：`_MAX_TOOL_WORKERS = 8`。

### 2. 哪些工具可以并行

源码里的并行安全白名单包括：

```python
_PARALLEL_SAFE_TOOLS = {
    "ha_get_state",
    "ha_list_entities",
    "ha_list_services",
    "read_file",
    "search_files",
    "session_search",
    "skill_view",
    "skills_list",
    "vision_analyze",
    "web_extract",
    "web_search",
}
```

路径工具单独处理：

```python
_PATH_SCOPED_TOOLS = {"read_file", "write_file", "patch"}
```

这里容易误解：`write_file` / `patch` 不是绝对不能并行，而是 **目标路径不重叠时可以并行**。

例子：

```text
可以并行：
  read_file("README.md")
  read_file("pyproject.toml")
  search_files("AIAgent")

可以并行：
  write_file("docs/a.md")
  write_file("docs/b.md")

不能并行：
  write_file("docs/a.md")
  patch("docs/a.md")

不能并行：
  clarify(...)
  read_file(...)

不能并行：
  terminal("npm test")
  read_file("package.json")
```

为什么 `terminal` 默认不并行？因为 terminal 可能修改全局状态、占用 cwd、触发审批、产生长时间输出，Hermes 不容易证明它和其他工具互不影响，所以保守串行。

### 3. agent-level tools

有些工具不能走 `model_tools.handle_function_call()`，因为它们需要 agent 内部状态。

`model_tools._AGENT_LOOP_TOOLS`：

```python
{"todo", "memory", "session_search", "delegate_task"}
```

实际在 `_invoke_tool()` / sequential path 里还有：

- `todo`：需要当前 agent 的 `TodoStore`。
- `session_search`：需要 `self._session_db` 和当前 `session_id`。
- `memory`：需要内置 `MemoryStore`，并桥接 external memory provider 写事件。
- `clarify`：需要 `self.clarify_callback`。
- `delegate_task`：需要 `parent_agent=self`，用于子 agent 继承上下文/中断等。
- context engine tools：通过 `self.context_compressor.handle_tool_call()`。
- memory provider tools：通过 `self._memory_manager.handle_tool_call()`。

普通工具才走：

```python
handle_function_call(
    function_name,
    function_args,
    effective_task_id,
    tool_call_id=tool_call.id,
    session_id=self.session_id or "",
    enabled_tools=list(self.valid_tool_names),
    skip_pre_tool_call_hook=True,
)
```

这里 `skip_pre_tool_call_hook=True` 是因为 agent loop 已经在执行前触发过 plugin block check，避免 hook 重复触发。

### 4. tool 执行期间的附加行为

无论并发还是串行，tool 执行还会处理：

- interrupt 检查，用户中断后跳过剩余 tool calls。
- checkpoint：`write_file` / `patch` / destructive `terminal` 前 snapshot。
- activity heartbeat：避免 Gateway 以为 agent 卡死。
- tool progress / start / complete callbacks。
- approval/sudo callbacks 传播到 worker thread。
- tool result 持久化：`maybe_persist_tool_result()`。
- subdirectory context hints。
- per-turn tool result budget enforcement。
- `/steer` 注入到 tool result。

tool 执行后会 append：

```python
{
  "role": "tool",
  "content": function_result,
  "tool_call_id": tool_call.id,
}
```

这是下一轮 LLM 看到工具结果的唯一渠道。

## 十一、预算耗尽和收尾

如果 loop 结束但没有 `final_response`，且达到：

```text
api_call_count >= max_iterations
or iteration_budget.remaining <= 0
```

Hermes 不直接报错，而是调用 `_handle_max_iterations()`：

1. 追加一个 user message，要求模型总结目前完成情况。
2. 发起一次无工具请求。
3. 得到最终总结。

这保证用户至少得到“做了什么、卡在哪里”的结果。

完成后收尾：

- `_save_trajectory(...)`
- `_cleanup_task_resources(effective_task_id)`
- `_persist_session(messages, conversation_history)`
- 写 turn-exit diagnostic log
- 触发 `post_llm_call` plugin hook
- 抽取最后 assistant reasoning
- 返回 result dict

返回结构包含：

- `final_response`
- `last_reasoning`
- `messages`
- `api_calls`
- `completed`
- `partial`
- `interrupted`
- `model/provider/base_url`
- token/cost 统计
- `pending_steer`

`chat()` 只是：

```python
result = self.run_conversation(message, stream_callback=stream_callback)
return result["final_response"]
```

## 十二、状态机简图

```text
run_conversation()
  |
  v
turn setup
  |
  v
append user message
  |
  v
build/reuse cached system prompt
  |
  v
preflight compression?
  |
  v
pre_llm_call hook + memory prefetch
  |
  v
while budget:
  |
  +--> build api_messages
  |
  +--> provider API call
  |      |
  |      +--> invalid/exception -> retry / fallback / compress / fail
  |
  +--> normalize assistant_message
         |
         +--> incomplete -> continue up to limit
         |
         +--> tool_calls
         |      |
         |      +--> validate/repair
         |      +--> append assistant tool_call msg
         |      +--> execute tools
         |      +--> append tool results
         |      +--> maybe compress
         |      +--> continue
         |
         +--> no tool_calls
                |
                +--> empty/reasoning-only recovery?
                +--> append final assistant msg
                +--> break
  |
  v
if no final_response -> max-iteration summary
  |
  v
persist + cleanup + hooks + return result
```

## 十三、设计观察

### 1. 强缓存意识

很多设计都围绕“不破坏 prompt cache”：

- system prompt 缓存到 session。
- continuation 不重新从磁盘加载 memory。
- plugin/memory context 注入 user message，不改 system prompt。
- Gateway 复用 agent cache，保持 tool schemas 和 system prompt 稳定。

这是 Hermes 和普通 agent demo 的重要差异。

### 2. 主循环聚合了太多责任

`run_conversation()` 同时处理：

- provider call
- retry/fallback
- compression
- tool validation
- tool execution
- streaming
- persistence
- plugin hooks
- memory prefetch
- UI callbacks

这让行为集中、跨入口一致，但也让函数非常难读。后续如果要重构，最自然的边界是：

- TurnContext / MessageBuilder
- ProviderCallRunner
- ToolCallRunner
- RecoveryPolicy
- PersistenceSink

### 3. Hermes 的容错是“内嵌外层循环”

OpenClaw 参考资料里常见“外层容错循环 + 内层 tool loop”。Hermes 没有独立拆出外层 runner，而是在每次 API call 的 retry loop 中处理：

- context overflow
- auth/rate limit
- fallback
- compression restart
- malformed response

这让单函数能掌握所有状态，但也使异常路径很密集。

### 4. tool execution 很重视平台长驻场景

很多看似冗余的 heartbeat、interrupt、worker thread callback、activity touch，主要服务 Gateway/TUI 这类长驻或异步入口。

例如 terminal 长命令执行时，Gateway 需要知道 agent 仍活着；并发 worker thread 也要继承 approval/sudo callback，否则 CLI 会死锁。

## 十四、和 Claude Code / OpenClaw 的对比

### 类 Claude Code

- tool-use loop。
- prompt caching。
- context compression。
- slash/steer 类在线控制。
- tool result 必须与 assistant tool_calls 成对。
- thinking/reasoning-only 消息 sanitizer。

### 类 OpenClaw

- gateway 运行场景强影响 agent loop。
- memory/user profile 进入 agent 上下文。
- 失败恢复和 context overflow 压缩。
- delegate/subagent 工具。

### Hermes 差异

- 主循环是同步函数，但服务多入口。
- provider abstraction 更复杂，覆盖 chat completions / Anthropic / Bedrock / Codex Responses。
- tool 并发由 agent loop 判断，而不是全串行。
- 内置 memory + external memory provider 同时存在。

## 十五、后续深挖问题

- `_build_system_prompt()` 具体组合了哪些上下文文件、memory、skills、platform hints。
- `_build_api_kwargs()` 如何按不同 transport 转换 tools、reasoning、max_tokens。
- `_try_activate_fallback()` 如何替换 client/runtime，又如何在下一 turn 恢复 primary。
- `_compress_context()` 如何创建新 session，并处理 old/new session 的历史边界。
- `delegate_task` 子 agent 如何继承 toolsets、budget、interrupt、file state。

## 十六、Agent Loop 里的 Hook 模块

有。Hermes 的 hook 主要来自插件系统，核心文件是：

```text
hermes_cli/plugins.py
  VALID_HOOKS
  PluginContext.register_hook()
  invoke_hook()
  get_pre_tool_call_block_message()
```

插件目录里的 `register(ctx)` 可以调用：

```python
ctx.register_hook("pre_llm_call", callback)
ctx.register_hook("post_tool_call", callback)
```

然后 agent loop 或 tool dispatcher 在固定位置调用 `invoke_hook(...)`。

### 1. Hook 总表

`VALID_HOOKS` 里注册的 hook 包括：

```text
pre_tool_call
post_tool_call
transform_terminal_output
transform_tool_result
pre_llm_call
post_llm_call
pre_api_request
post_api_request
on_session_start
on_session_end
on_session_finalize
on_session_reset
subagent_stop
pre_gateway_dispatch
pre_approval_request
post_approval_response
```

不是每个 hook 都在 `run_conversation()` 主循环里直接触发。有些在 CLI/Gateway、terminal tool、approval tool、delegate tool 里触发。

### 2. Agent 主循环里直接触发的 hook

主循环直接相关的有这几个：

```text
on_session_start
  位置：run_conversation() 构建新 system prompt 后
  用途：新 session 初始化，比如插件预热 session 级缓存

pre_llm_call
  位置：进入 while tool loop 之前
  用途：给当前用户消息注入额外上下文
  特点：注入 user message，不改 system prompt，避免破坏 prompt cache

pre_api_request
  位置：每一次 API call 前
  用途：观测本次请求的模型、provider、message_count、tool_count、估算 token 等

post_api_request
  位置：每一次 API call 成功返回后
  用途：记录 duration、finish_reason、usage、assistant 文本长度、tool_call 数量

post_llm_call
  位置：整个 tool-calling loop 结束后，有 final_response 且未中断
  用途：持久化本轮对话、同步外部系统、收尾观测

on_session_end
  位置：run_conversation() 最末尾
  用途：每次 run_conversation 调用结束后的 cleanup/flush
```

这里要区分两个概念：

```text
post_api_request = 每次模型 API 请求结束都触发
post_llm_call    = 整个用户 turn 结束才触发一次
```

如果一轮用户请求里模型调用了 30 轮工具，那么可能是：

```text
pre_llm_call: 1 次

pre_api_request/post_api_request: 多次
  第 1 次 API：模型决定调用工具
  第 2 次 API：看到工具结果后继续调用工具
  ...
  第 N 次 API：模型最终回答

post_llm_call: 1 次
on_session_end: 1 次
```

### 3. Tool 执行相关 hook

tool hook 有两层入口。

第一层在 agent loop 里。`run_agent.py` 的顺序执行路径和并发执行路径都会先检查：

```text
get_pre_tool_call_block_message(...)
```

这个函数内部会触发：

```text
pre_tool_call
```

插件可以返回：

```python
{"action": "block", "message": "Reason"}
```

如果返回 block，工具不会执行，Hermes 会把错误结果作为 tool result 放回 messages。

第二层在 `model_tools.py::handle_function_call()`。普通 registry 工具执行后会触发：

```text
post_tool_call
transform_tool_result
```

含义是：

```text
post_tool_call
  -> 观察型 hook，拿到 tool_name、args、result、duration_ms
  -> 返回值不改变结果

transform_tool_result
  -> 变换型 hook
  -> 第一个返回 str 的插件可以替换最终 tool result
  -> 替换后的 result 会进入 conversation messages
```

所以 tool result 不只会被压缩器调整，也可能在进入 messages 之前先被插件 hook 改写。

### 4. Terminal 输出还有专门 hook

`terminal` 工具内部还有一个更早的输出变换点：

```text
transform_terminal_output
```

位置在 `tools/terminal_tool.py`：

```text
命令执行完成
  -> 拿到原始 output
  -> transform_terminal_output hook
  -> 默认截断、ANSI strip、redact 等处理
  -> 返回 tool result
```

它比 `transform_tool_result` 更窄，只针对 terminal 输出；也更早，发生在 terminal 工具内部默认清洗之前。

### 5. 压缩和 hook 的关系

压缩本身不是 plugin hook，但压缩前后会触发其他生命周期：

```text
_compress_context()
  -> memory_manager.on_pre_compress(messages)
  -> context_compressor.compress(...)
  -> context_engine.on_session_start(boundary_reason="compression")
  -> memory_provider.on_session_switch(reason="compression")
```

这些不是 `hermes_cli/plugins.py` 的 plugin hook，而是 memory/context engine 自己的生命周期回调。

也就是说 Hermes 有两套“hook-like”机制：

```text
插件 hook
  -> hermes_cli/plugins.py
  -> 给外部插件扩展 agent/tool/API/session 生命周期

内部生命周期回调
  -> memory_manager / context_engine / context_compressor
  -> 给内置上下文、记忆、压缩模块同步状态
```

### 6. Hook 在一次完整 trace 里的位置

用一个带工具调用的 turn 举例：

```text
用户输入
  -> on_session_start       只在新 session 首轮
  -> pre_llm_call           注入插件上下文到 user message

while loop 第 1 轮:
  -> pre_api_request
  -> 调 provider
  -> post_api_request
  -> assistant 返回 tool_calls

执行工具:
  -> pre_tool_call
     -> 可能 block
  -> 真正执行工具
  -> post_tool_call
  -> transform_tool_result
  -> tool result append 到 messages
  -> 可能触发 compression

while loop 第 2 轮:
  -> pre_api_request
  -> 调 provider，带上工具结果
  -> post_api_request
  -> assistant 可能继续调用工具或最终回答

最终回答:
  -> post_llm_call
  -> memory sync / background review
  -> on_session_end
```

如果中间调用了 `terminal`：

```text
terminal command
  -> transform_terminal_output
  -> terminal 默认截断/清洗
  -> post_tool_call
  -> transform_tool_result
```

### 7. Hook 的设计取舍

几个明显设计点：

- `pre_llm_call` 只允许注入 user message，不改 system prompt，主要是保护 prompt cache。
- `pre_tool_call` 可以 block，所以它是策略控制点。
- `post_tool_call` 是观察点，不改结果。
- `transform_tool_result` 是结果改写点，影响后续模型看到的上下文。
- hook 调用基本都是 fail-open：插件异常会记录 warning，但不会打断 agent loop。
- `pre_api_request/post_api_request` 是观测 API 请求的细粒度 hook，适合 Langfuse 这类 tracing 插件。
