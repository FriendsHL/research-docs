---
created: '2026-04-30'
tags:
- hermes-agent
- provider
- transport
status: draft
---

# 模型 Provider 与传输层源码解析

## 一、先给结论

Hermes 的模型调用分成三层：

```text
Provider 选择层
  -> 你要用 OpenRouter、Anthropic、Bedrock、Codex、xAI、Custom 还是本地模型

Transport 转换层
  -> 把 Hermes 内部统一的 messages/tools 转成 provider 要的请求格式

Client/SDK 执行层
  -> 真正发 HTTP 请求，拿回 response，再转回统一格式
```

最重要的一点：

```text
Hermes 自己做 provider 抽象和请求转换。
但底层不是全部手写 HTTP。
```

它会按 provider 使用不同底层：

```text
OpenAI-compatible / OpenRouter / Nous / local OpenAI wire
  -> openai.OpenAI SDK

Anthropic native
  -> anthropic.Anthropic SDK

Bedrock Claude
  -> anthropic.AnthropicBedrock SDK

Bedrock non-Claude
  -> boto3 Bedrock Converse API

Codex Responses / xAI Responses
  -> OpenAI SDK 的 responses 风格接口，外加 Hermes 自己的 adapter

Custom provider
  -> 通常走 openai.OpenAI SDK，只是 base_url/api_key 换成用户配置
```

所以可以这样理解：

```text
Hermes 不是“直接把所有 provider 都交给一个官方 SDK”。
它是：
  自己决定 provider 和 api_mode
  自己转换 messages/tools/reasoning/usage
  再把最终 kwargs 交给对应 SDK 或 boto3
```

## 二、一个最简单的例子

用户配置：

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4.6
```

运行时大概发生：

```text
1. runtime_provider 解析 provider=openrouter
2. 找到 OpenRouter API key 和 base_url
3. api_mode = chat_completions
4. AIAgent 创建 openai.OpenAI(api_key=..., base_url=https://openrouter.ai/api/v1)
5. 主循环把 messages/tools 交给 ChatCompletionsTransport
6. transport 生成 chat.completions.create(...) kwargs
7. OpenAI SDK 发请求到 OpenRouter
8. 回来的 response 被统一成 assistant content/tool_calls/reasoning/usage
```

直白说：

```text
OpenRouter 看起来不是 OpenAI。
但它兼容 OpenAI chat completions 协议。
所以 Hermes 用 OpenAI SDK，换一个 base_url。
```

## 三、Provider 和 api_mode 是两件事

Hermes 里容易混淆两个概念：

```text
provider
  -> 你用哪家服务 / 哪个凭证来源

api_mode
  -> 请求走哪种协议格式
```

例子：

```text
provider=openrouter
api_mode=chat_completions

provider=anthropic
api_mode=anthropic_messages

provider=openai-codex
api_mode=codex_responses

provider=bedrock
api_mode=anthropic_messages       # Bedrock 上的 Claude

provider=bedrock
api_mode=bedrock_converse         # Bedrock 上的 Nova/Llama/DeepSeek 等

provider=custom
api_mode=chat_completions         # 普通 OpenAI-compatible 自定义 endpoint

provider=custom
api_mode=anthropic_messages       # URL 以 /anthropic 结尾的自定义 endpoint
```

一个 provider 可能对应不同 api_mode。比如 Bedrock：

```text
Bedrock + Claude
  -> 走 AnthropicBedrock SDK
  -> api_mode = anthropic_messages

Bedrock + Nova/Llama/DeepSeek
  -> 走 boto3 Converse
  -> api_mode = bedrock_converse
```

一个 api_mode 也可能服务很多 provider。比如 `chat_completions`：

```text
OpenRouter
Nous
NVIDIA NIM
Qwen
Kimi
DeepSeek
xAI 某些兼容接口
Ollama / LM Studio
Custom OpenAI-compatible endpoint
```

## 四、关键源码位置

### 1. Runtime provider 解析

```text
hermes_cli/runtime_provider.py
  resolve_runtime_provider()
  resolve_requested_provider()
  _resolve_runtime_from_pool_entry()
  _detect_api_mode_for_url()
```

这层负责回答：

```text
用户到底想用哪个 provider？
这个 provider 的 base_url 是什么？
API key 从哪里来？
应该用哪种 api_mode？
有没有 credential pool？
```

### 2. Agent 初始化和 client 创建

```text
run_agent.py
  AIAgent.__init__()
  _create_openai_client()
  _get_transport()
  _build_api_kwargs()
```

这层负责：

```text
根据 provider/api_mode 创建 client
缓存 transport
每次 API call 前构造 kwargs
按 api_mode 调不同 SDK/adapter
```

### 3. Transport 抽象

```text
agent/transports/base.py
agent/transports/__init__.py
agent/transports/chat_completions.py
agent/transports/anthropic.py
agent/transports/codex.py
agent/transports/bedrock.py
agent/transports/types.py
```

`ProviderTransport` 的核心接口是：

```text
convert_messages()
convert_tools()
build_kwargs()
normalize_response()
validate_response()
extract_cache_stats()
map_finish_reason()
```

一句话：

```text
transport 把 Hermes 内部格式和 provider 外部格式互相翻译。
```

### 4. Provider-specific adapter

```text
agent/anthropic_adapter.py
agent/codex_responses_adapter.py
agent/bedrock_adapter.py
agent/gemini_native_adapter.py
agent/gemini_cloudcode_adapter.py
```

这些文件更贴近具体 provider 协议，比如：

```text
Anthropic content block 怎么组织
Responses API input item 怎么组织
Bedrock Converse toolConfig 怎么组织
Gemini thought_signature 怎么回放
```

## 五、Provider 解析链路

入口一般在 CLI/Gateway 创建 `AIAgent` 之前。运行时会调用：

```text
resolve_runtime_provider(...)
```

它大概按这个顺序找：

```text
1. 用户显式指定 provider/base_url/api_key
2. config.yaml 的 model.provider / model.default / model.base_url / model.api_mode
3. 命名 custom provider
4. credential pool
5. provider 自己的 OAuth / API key 解析
6. auto fallback 到 OpenRouter/custom
```

返回结果类似：

```python
{
    "provider": "anthropic",
    "api_mode": "anthropic_messages",
    "base_url": "https://api.anthropic.com",
    "api_key": "...",
    "source": "env",
    "requested_provider": "anthropic",
}
```

或者：

```python
{
    "provider": "openai-codex",
    "api_mode": "codex_responses",
    "base_url": "https://chatgpt.com/backend-api/codex",
    "api_key": "...",
    "source": "hermes-auth-store",
}
```

这一步的关键是：

```text
provider 不只决定 key 从哪里来。
provider 还会影响 api_mode、base_url、headers、client 类型。
```

## 六、api_mode 怎么决定

Hermes 支持的核心 api_mode：

```text
chat_completions
anthropic_messages
codex_responses
bedrock_converse
```

### 1. chat_completions

这是默认模式。

适用：

```text
OpenAI-compatible API
OpenRouter
Nous
本地 LM Studio / Ollama
大多数 custom endpoint
```

内部 messages 本来就是 OpenAI 格式，所以转换最少：

```text
messages 基本原样
tools 基本原样
kwargs = {
  model,
  messages,
  tools,
  temperature,
  max_tokens 或 max_completion_tokens,
  extra_body,
}
```

但这个 transport 仍然很复杂，因为不同“兼容 OpenAI”的 provider 细节不一样：

```text
OpenRouter 有 provider routing extra_body
GitHub Models 有自己的 reasoning 参数
Gemini OpenAI-compat 有 thinking_config
Kimi/Moonshot tool schema 更严格
LM Studio reasoning effort 要按本地模型能力裁剪
Qwen Portal 要 metadata 和 cache_control
GPT-5/Codex 模型 system role 可能要换成 developer role
```

直白说：

```text
大家都说自己兼容 OpenAI。
但每家兼容得不完全一样。
ChatCompletionsTransport 就是在擦这些差异。
```

### 2. anthropic_messages

适用：

```text
Anthropic 官方
Bedrock 上的 Claude
部分 Anthropic-compatible 网关
URL 以 /anthropic 结尾的 custom provider
Kimi /coding 这类 Anthropic 协议 endpoint
```

Anthropic 的格式和 OpenAI 不一样。

OpenAI 风格：

```python
{"role": "assistant", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}
```

Anthropic 风格：

```text
assistant content block:
  {"type": "tool_use", "id": "...", "name": "...", "input": {...}}

user content block:
  {"type": "tool_result", "tool_use_id": "...", "content": "..."}
```

所以 `AnthropicTransport` 会调用：

```text
agent/anthropic_adapter.py
  convert_messages_to_anthropic()
  convert_tools_to_anthropic()
  build_anthropic_kwargs()
```

它还处理：

```text
thinking/reasoning block
prompt caching
OAuth token 判断
1M context beta header
tool_use stop_reason 映射
第三方 Anthropic-compatible endpoint 的模型名差异
```

### 3. codex_responses

适用：

```text
OpenAI Codex / ChatGPT backend
Direct OpenAI GPT-5.x Responses path
xAI direct Responses path
部分 Copilot/GitHub responses-style backend
```

Responses API 和 chat completions 最大差异：

```text
chat_completions:
  messages = [{"role": "...", "content": "..."}]

responses:
  instructions = system prompt
  input = 多种 input item
  tools = function definitions
  reasoning = {"effort": "...", "summary": "auto"}
  include = ["reasoning.encrypted_content"]
```

`ResponsesApiTransport` 会把 chat messages 转成 Responses input items：

```text
system message
  -> instructions

user/assistant/tool history
  -> input items

OpenAI tool schema
  -> responses function tools
```

还会保存 Codex 特有的 provider_data：

```text
codex_reasoning_items
codex_message_items
call_id
response_item_id
encrypted reasoning content
```

这些字段不能直接漏给普通 chat_completions provider，否则对方可能 400。所以切回 chat_completions 时，`ChatCompletionsTransport.convert_messages()` 会清理 Codex 字段。

### 4. bedrock_converse

适用：

```text
Bedrock 上的非 Claude 模型
比如 Amazon Nova、Llama、DeepSeek 等
```

Bedrock Converse 不走 OpenAI SDK，也不走 Anthropic SDK，而是 boto3：

```text
boto3.client("bedrock-runtime").converse(...)
boto3.client("bedrock-runtime").converse_stream(...)
```

`BedrockTransport` 负责：

```text
OpenAI messages -> Bedrock Converse messages
OpenAI tools -> Bedrock toolConfig
Bedrock response -> NormalizedResponse
Bedrock stopReason -> OpenAI finish_reason
```

如果 Bedrock 模型是 Claude，Hermes 反而优先走：

```text
AnthropicBedrock SDK + anthropic_messages
```

原因是 Claude 在 Anthropic SDK 路径上支持更完整的 Claude 特性，比如 prompt caching、thinking budgets、1M context beta。

## 七、Transport 在主循环里怎么用

主循环里所有 API 请求最终都要构造 `api_kwargs`：

```text
run_agent.py::_build_api_kwargs(api_messages)
```

按 `self.api_mode` 分支：

```text
anthropic_messages
  -> AnthropicTransport.build_kwargs()

bedrock_converse
  -> BedrockTransport.build_kwargs()

codex_responses
  -> ResponsesApiTransport.build_kwargs()

chat_completions
  -> ChatCompletionsTransport.build_kwargs()
```

可以把它理解为：

```text
主循环只知道“我要问模型”。
transport 知道“这家 provider 要我怎么问”。
```

例子：同一条内部消息：

```python
{"role": "user", "content": "读取 run_agent.py 看看主循环"}
```

OpenAI-compatible 请求里还是：

```python
messages=[
  {"role": "system", "content": "..."},
  {"role": "user", "content": "读取 run_agent.py 看看主循环"}
]
```

Anthropic 请求里变成：

```python
system="..."
messages=[
  {"role": "user", "content": [{"type": "text", "text": "读取 run_agent.py 看看主循环"}]}
]
```

Responses API 请求里变成：

```python
instructions="..."
input=[
  {"role": "user", "content": "..."}
]
```

Bedrock 请求里变成：

```python
messages=[
  {"role": "user", "content": [{"text": "读取 run_agent.py 看看主循环"}]}
]
```

模型返回后也要反向统一。

最终主循环希望看到的都是：

```python
NormalizedResponse(
    content="...",
    tool_calls=[...],
    finish_reason="tool_calls" | "stop" | "length",
    reasoning="...",
    usage=...
)
```

这样后面的 agent loop 就不用到处判断 provider。

## 八、Tool Call 的转换例子

Hermes 内部工具 schema 是 OpenAI 风格：

```python
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "Read a file",
    "parameters": {
      "type": "object",
      "properties": {
        "path": {"type": "string"}
      },
      "required": ["path"]
    }
  }
}
```

### OpenAI-compatible

基本原样：

```python
tools=[{"type": "function", "function": {...}}]
```

模型返回：

```python
tool_calls=[
  {
    "id": "call_abc",
    "function": {
      "name": "read_file",
      "arguments": "{\"path\":\"run_agent.py\"}"
    }
  }
]
```

### Anthropic

工具 schema 要变成 Anthropic input_schema：

```python
{
  "name": "read_file",
  "description": "Read a file",
  "input_schema": {
    "type": "object",
    "properties": {"path": {"type": "string"}},
    "required": ["path"]
  }
}
```

模型返回的 tool_use block：

```python
{
  "type": "tool_use",
  "id": "toolu_abc",
  "name": "read_file",
  "input": {"path": "run_agent.py"}
}
```

Transport 会统一回：

```python
ToolCall(
  id="toolu_abc",
  name="read_file",
  arguments="{\"path\":\"run_agent.py\"}"
)
```

### Responses API

Responses API 的 tool call 有额外 provider 状态：

```python
ToolCall(
  id="call_xxx",
  name="read_file",
  arguments="{...}",
  provider_data={
    "call_id": "call_xxx",
    "response_item_id": "fc_xxx"
  }
)
```

这些额外字段要保存，因为下一轮 Responses API 可能需要回放对应 item。

但如果后来 fallback 到普通 OpenAI-compatible provider，这些字段就要剥掉，否则普通 provider 不认识。

## 九、Provider client 是怎么创建的

`AIAgent.__init__()` 里按 api_mode 创建不同 client。

### 1. anthropic_messages

如果是 native Anthropic：

```text
build_anthropic_client(api_key, base_url)
  -> anthropic.Anthropic(...)
```

如果是 Bedrock Claude：

```text
build_anthropic_bedrock_client(region)
  -> anthropic.AnthropicBedrock(...)
```

这时：

```text
self._anthropic_client = ...
self.client = None
```

因为不用 OpenAI SDK client。

### 2. bedrock_converse

```text
self.client = None
```

真正请求时由 `bedrock_adapter.py` 用 boto3 创建/缓存 Bedrock runtime client。

### 3. chat_completions / codex_responses

一般都会创建 OpenAI SDK client：

```python
OpenAI(
  api_key=api_key,
  base_url=base_url,
  default_headers=...
)
```

这里的 `base_url` 可以是：

```text
https://openrouter.ai/api/v1
https://inference-api.nousresearch.com/v1
http://localhost:1234/v1
https://chatgpt.com/backend-api/codex
https://api.x.ai/v1
```

所以 OpenAI SDK 在 Hermes 里更像一个“通用 HTTP client for OpenAI wire protocol”。

## 十、凭证从哪里来

Provider 凭证来源很多，不只是 `.env`：

```text
env:<VAR>
  -> ~/.hermes/.env 或系统环境变量

claude_code
  -> ~/.claude/.credentials.json

hermes_pkce
  -> ~/.hermes/.anthropic_oauth.json

device_code
  -> ~/.hermes/auth.json providers.<provider>

qwen-cli
  -> ~/.qwen/oauth_creds.json

gh_cli
  -> GitHub CLI token

config:<name>
  -> config.yaml 里的 custom provider

model_config
  -> model.api_key

manual
  -> hermes auth add 手动添加
```

`agent/credential_pool.py` 会把同一 provider 的多个凭证放进 pool。

例子：

```text
provider=anthropic

pool 里可能有：
  1. ANTHROPIC_API_KEY
  2. Claude Code OAuth token
  3. Hermes 自己登录的 PKCE token
```

运行时选择一个可用 entry：

```text
pool.select()
  -> 返回一个 PooledCredential
  -> runtime_provider 根据 entry 生成 api_key/base_url/api_mode
```

## 十一、Credential Pool 怎么参与恢复

主循环遇到 401/429/限流/凭证失效时，不一定马上 fallback 到另一个 provider。

有时会先尝试同 provider 换一个 credential：

```text
当前 Anthropic key 被 429
  -> credential pool 里还有另一个 Anthropic token
  -> 切到另一个 token
  -> 重建 client
  -> retry 当前请求
```

这和 fallback provider 不一样。

```text
credential pool
  -> 同一个 provider 内换凭证

fallback provider
  -> 换 provider 或换模型
```

直白例子：

```text
你有 3 张同一家店的会员卡。
第一张刷爆了，先试第二张。

如果这家店整个关门了，
再换另一家店。
```

## 十二、Fallback Provider 怎么工作

`run_agent.py::_try_activate_fallback()` 负责 fallback。

大概场景：

```text
当前 provider 一直 429
当前 provider 返回无效响应
当前 provider 网络失败
当前模型不支持工具
当前 Responses API 出错
```

流程：

```text
1. 从 fallback chain 取下一个候选
2. 解析它的 provider/base_url/api_key/api_mode
3. 更新 self.model / self.provider / self.api_mode / self.base_url
4. 清 transport cache
5. 重建对应 client
6. 继续当前 turn
```

注意 fallback 后，历史消息还在。所以 provider 切换时要做清理：

```text
Codex provider_data 不能传给普通 chat_completions
Anthropic reasoning signatures 不能乱传给 DeepSeek/Kimi
严格 provider 不接受未知字段，要 sanitize
```

这也是为什么 transport 层和主循环里有大量 provider-specific sanitizer。

## 十三、Reasoning / Thinking 怎么处理

Hermes 内部会把推理内容分几类保存：

```text
assistant_msg["reasoning"]
reasoning_content
reasoning_details
codex_reasoning_items
encrypted_content
thought_signature
```

不同 provider 的规则不一样：

```text
Anthropic
  -> thinking block 可能带 signature
  -> 后续 replay 时要保留合法 reasoning_details

Codex Responses
  -> reasoning item / encrypted_content 要通过 provider_data 保存

Gemini
  -> tool call 可能带 thought_signature
  -> 下一轮要回放，否则 API 可能 400

DeepSeek/Kimi/Moonshot
  -> reasoning_content 字段要求和普通 content 不同
```

主循环做了一件事：

```text
对外给 provider 的 api_messages 里，放 provider 需要的 reasoning 字段；
对内持久化 messages 时，保留 Hermes 自己能理解的字段。
```

这就是为什么 provider 切换时要特别小心。一个 provider 的 reasoning 状态，可能对另一个 provider 是毒数据。

## 十四、Prompt Caching 和 Provider

Prompt caching 不是所有 provider 都一样。

Hermes 对 Anthropic-family 做了专门处理：

```text
native Anthropic
OpenRouter 上的 Claude
第三方 Anthropic-compatible gateway
```

主循环会在 API-facing messages 上加 cache_control。

关键点：

```text
system prompt 尽量保持稳定
plugin/context/memory 的临时内容注入 user message
压缩时才允许重建 system prompt 边界
```

这样可以让 Anthropic prefix cache 命中，降低长会话成本。

## 十五、一个完整请求例子：Anthropic

配置：

```yaml
model:
  provider: anthropic
  default: claude-sonnet-4-6
```

运行：

```text
resolve_runtime_provider()
  -> provider=anthropic
  -> api_mode=anthropic_messages
  -> base_url=https://api.anthropic.com
  -> api_key=ANTHROPIC_API_KEY 或 Claude Code OAuth token

AIAgent.__init__()
  -> build_anthropic_client()
  -> self._anthropic_client = anthropic.Anthropic(...)

run_conversation()
  -> 内部 messages 是 OpenAI 风格

_build_api_kwargs()
  -> AnthropicTransport.build_kwargs()
  -> convert_messages_to_anthropic()
  -> convert_tools_to_anthropic()

API call
  -> self._anthropic_client.messages.create(...)

response
  -> AnthropicTransport.normalize_response()
  -> content/tool_calls/reasoning/finish_reason 统一回 Hermes 格式
```

如果 Claude 返回 tool_use：

```text
Anthropic stop_reason = tool_use
```

Hermes 会映射成：

```text
finish_reason = tool_calls
```

然后进入 Agent 主循环的 tool 执行分支。

## 十六、一个完整请求例子：OpenAI-compatible Custom

配置：

```yaml
model:
  provider: custom
  default: local-model
  base_url: http://localhost:1234/v1
```

运行：

```text
resolve_runtime_provider()
  -> provider=custom
  -> api_mode=chat_completions
  -> base_url=http://localhost:1234/v1
  -> api_key=no-key-required 或 OPENAI_API_KEY

AIAgent.__init__()
  -> OpenAI(api_key="no-key-required", base_url="http://localhost:1234/v1")

_build_api_kwargs()
  -> ChatCompletionsTransport.build_kwargs()

API call
  -> client.chat.completions.create(...)
```

所以本地模型只要兼容 OpenAI `/v1/chat/completions`，Hermes 就能把它当成 chat_completions provider 使用。

## 十七、一个完整请求例子：Bedrock

配置：

```yaml
model:
  provider: bedrock
  default: us.amazon.nova-pro-v1:0

bedrock:
  region: us-east-1
```

运行：

```text
resolve_runtime_provider()
  -> provider=bedrock
  -> 判断不是 Claude 模型
  -> api_mode=bedrock_converse
  -> base_url=https://bedrock-runtime.us-east-1.amazonaws.com
  -> api_key=aws-sdk

AIAgent.__init__()
  -> self.client = None
  -> 记录 region / guardrail config

_build_api_kwargs()
  -> BedrockTransport.build_kwargs()
  -> convert_messages_to_converse()
  -> convert_tools_to_converse()

API call
  -> boto3 bedrock-runtime converse 或 converse_stream

response
  -> normalize_converse_response()
  -> NormalizedResponse
```

如果配置的是 Bedrock Claude：

```yaml
model:
  provider: bedrock
  default: us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

则变成：

```text
api_mode=anthropic_messages
client=AnthropicBedrock
```

原因是 Claude 走 AnthropicBedrock SDK 特性更完整。

## 十八、设计观察

### 1. 内部统一 OpenAI-style messages

Hermes 内部大部分逻辑都围绕 OpenAI-style messages：

```text
role = system / user / assistant / tool
assistant.tool_calls
tool.tool_call_id
```

这样主循环、压缩、session 存储、tool 执行都能用一套结构。

Provider 差异尽量推到 transport/adapter。

### 2. Provider 兼容成本很高

源码里大量判断不是“架构复杂化”，而是现实里 provider 差异很多：

```text
max_tokens 参数名不同
temperature 有的模型不能传
reasoning 参数各家不同
tool schema 严格程度不同
system role 有些要换 developer
图片格式和大小限制不同
prompt caching header 不同
OAuth token 刷新方式不同
```

所以 provider 层看起来很厚。

### 3. Transport 只管格式，不管重试

`ProviderTransport` 明确不负责：

```text
client construction
streaming
credential refresh
prompt caching
interrupt
retry
fallback
```

这些仍在 `AIAgent`。

这让 transport 职责比较清楚：

```text
输入格式 -> provider kwargs
provider response -> NormalizedResponse
```

但代价是 `run_agent.py` 仍然很重。

### 4. Codex / Anthropic / Gemini 的 provider_data 很关键

普通文本模型只要 content 就够。

但现代 agent 模型经常有隐藏状态：

```text
reasoning signature
encrypted thinking
response item id
thought signature
cache marker
```

这些状态不一定给用户看，但下一轮请求可能必须带回去。

Hermes 的 `provider_data` 就是为了放这些“只有某个 provider 懂”的东西。

## 十九、和 Agent 主循环的关系

Provider 模块给主循环提供两样东西：

```text
1. api_kwargs
   -> 这一轮怎么请求 provider

2. normalized response
   -> 主循环怎么判断 final answer / tool_calls / length / error
```

主循环不希望到处写：

```text
if Anthropic:
if Bedrock:
if Codex:
```

所以 transport 层尽量把差异提前处理掉。

但现实中仍然有一些 provider 分支留在主循环，例如：

```text
选择具体 SDK client
流式请求路径
错误恢复
credential pool
fallback
prompt caching
reasoning replay
context overflow 压缩
```

所以准确说：

```text
Transport 减少了主循环里的 provider 分支，
但没有完全消灭 provider 分支。
```

## 二十、后续深挖问题

- `agent/anthropic_adapter.py` 如何把 OpenAI messages 转成 Anthropic content blocks。
- `agent/codex_responses_adapter.py` 如何维护 Responses API item history。
- `agent/bedrock_adapter.py` 如何处理 ConverseStream、guardrails、model discovery。
- `agent/credential_pool.py` 的选择策略、刷新策略、失败标记。
- `_try_activate_fallback()` 的 fallback chain 和 provider 切换细节。
- Gemini / Qwen / Kimi 这些“兼容但又不完全兼容”的 provider 分支。
