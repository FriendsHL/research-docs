---
modified: '2026-03-27T17:28:28.136415'
---

# OpenClaw Agent Loop 学习笔记

> 研究日期：2026-03-27 | 基于源码精读 + 问答推导

---

## 一、pi-sdk 是什么

OpenClaw 的底层 LLM 能力来自 Mario Zechner（badlogic）开发的开源工具包，分三层：

```
pi-ai            ← 底层：统一封装各家 LLM API（Anthropic/OpenAI/Google 等）
pi-agent-core    ← 中层：Agent Loop 运行时（ReAct 循环抽象）
pi-coding-agent  ← 上层：完整产品层（Session 持久化 + 工具 + Retry + Compaction）
```

OpenClaw 直接 import SDK、嵌入式集成，不是子进程调用。

---

## 二、双层循环架构

### 设计思路

拆成两层的本质是**职责分离**：

- 内层：只管"做什么"——跑完一次正常的 agent 对话
- 外层：只管"出错怎么办"——容错决策

如果混在一起，tool-use 跑到第 5 轮时 auth 过期，处理逻辑会极其复杂。

---

### 外层循环：runEmbeddedPiAgent

**文件**：`src/agents/pi-embedded-runner/run.ts`

**职责**：运行级别的容错编排，不参与 LLM 对话本身。

```
while(true)  最多重试 32~160 次（由 auth profile 数量动态决定）
  │
  ├── 初始化：workspace / provider / model / auth profile 解析
  ├── 运行前 hooks：before_model_resolve、before_agent_start（插件可介入）
  ├── 调用 runEmbeddedAttempt()  ← 内层
  │
  └── attempt 返回后的决策树：
        ├── context overflow？
        │     → 触发 compaction 压缩历史 → continue
        ├── auth error？
        │     → 刷新 OAuth token → continue
        ├── rate limit / overload？
        │     → 标记当前 profile 进入 cooldown
        │     → advanceAuthProfile() 轮换到下一个
        │     → 有可用 profile → continue
        │     → 无 profile + 有 fallback model → throw FailoverError → 换模型重跑
        │     → 无 profile + 无 fallback → fall-through → 返回错误给用户
        ├── thinking level 不支持？
        │     → 降级 high → low → off → continue
        ├── 超出重试上限？
        │     → 直接返回错误
        └── 正常结束 → markAuthProfileGood → 返回 payloads

```

---

### 内层循环：runEmbeddedAttempt

**文件**：`src/agents/pi-embedded-runner/run/attempt.ts`

**职责**：驱动一次完整的 tool-use 对话，失败直接抛出，不处理任何容错。

```
runEmbeddedAttempt()
  │
  ├── 准备阶段
  │     ├── 解析 sandbox context
  │     ├── 加载 skills prompt（把 SKILL.md 注入 system prompt）
  │     ├── 加载 bootstrap files（workspace 上下文文件）
  │     └── 创建 tools：createOpenClawCodingTools() + MCP tools + LSP tools
  │
  ├── 构建 system prompt（buildEmbeddedSystemPrompt）
  │
  ├── 初始化 SessionManager
  │     ├── 加载 session file（历史消息）
  │     └── sanitize history（修复 tool use/result 配对）
  │
  ├── SessionManager.run()  ← pi-coding-agent 驱动的真正 tool-use loop
  │     用户消息 → LLM → [有 tool call] → 执行工具 → 结果回传 → LLM → ... → end_turn
  │     通过 subscribeEmbeddedPiSession() 订阅流式事件：
  │       onPartialReply / onBlockReply / onToolResult / onReasoningStream
  │
  └── 结束处理
        ├── 触发 after_agent_end hooks
        ├── context engine 维护（向量库）
        └── 返回 EmbeddedRunAttemptResult

```

---

## 三、stopReason 的来源

`stopReason` 不是模型的原始字段，是 `pi-ai` 框架**标准化后的信号**，屏蔽了各 provider 差异。

### 正常流结束：经 mapStopReason() 映射

| Anthropic 原始值 | OpenAI 原始值 | pi-ai stopReason |
|---|---|---|
| end_turn | stop | "stop" |
| tool_use | tool_calls | "toolUse" |
| max_tokens | length | "length" |
| refusal | — | "error" |

### HTTP 层异常（如 429 rate limit）

模型根本没机会返回 stop_reason，pi-ai 在 catch 块自己赋值：

```javascript
output.stopReason = options?.signal?.aborted ? "aborted" : "error";
output.errorMessage = error.message; // "429 Too Many Requests..."
```

### 用户中断（AbortController）

框架直接赋值 `"aborted"`，和模型无关。

### 外层循环如何使用 stopReason

`stopReason === "error"` 只是**入场券**，真正的错误分类靠 `errorMessage` 文本：

```
classifyFailoverReason(errorMessage) → 返回以下之一：
  "rate_limit" / "overloaded" / "auth" / "auth_permanent"
  "billing" / "timeout" / "model_not_found" / "session_expired" / null
```

---

## 四、典型场景完整链路

### 场景1：LLM 调用 tool 缺少字段，向用户询问

**关键结论：这个错误不会触发外层循环，完全在内层自闭环处理。**

```
内层 tool-use loop：
  LLM 调用 tool（缺少必填字段）
    → 底层执行 tool → 返回 error: "missing required field: xxx"
    → error 作为 tool_result 写回消息历史
    → LLM 下一轮看到 error，自行决策：
        ├── 能补全 → 补全字段，重试 tool（继续 loop）
        └── 不能补全 → 输出文本请用户提供信息，stopReason = "stop"

外层循环：
  attempt 正常结束，不触发任何重试

payload 构建：
  isRecoverableToolError("missing required field") = true
  → showWarning = false（不显示 ⚠️ 警告）
  → 只输出 LLM 的文本回复
```

`RECOVERABLE_TOOL_ERROR_KEYWORDS`：`required / missing / invalid / must be / must have / needs / requires`

---

### 场景2：内层第二轮 LLM 调用触发 rate limit

```
内层第二轮 LLM 调用：
  HTTP 429 → pi-ai catch 块
    → stopReason = "error"
    → errorMessage = "429 Too Many Requests / rate limit..."
    → attempt 返回（内层不处理）

外层决策：
  1. contextOverflowError? → NO
  2. promptError? → NO（prompt 已提交，是第二轮报的错）
  3. 检查 lastAssistant：
       classifyFailoverReason("429 rate limit") → "rate_limit"
       isFailoverAssistantError = true → shouldRotate = true

  4. 进入 shouldRotate 分支：
       maybeMarkAuthProfileFailure(profileId, "rate_limit")  ← 标记 cooldown
       advanceAuthProfile()
         ├── 有下一个 profile → continue（重跑整个 attempt）
         ├── 无 profile + fallbackConfigured → throw FailoverError → 换模型
         └── 无 profile + 无 fallback → fall-through → 用户收到错误提示
```

**重要细节**：重跑时是重新跑整个 attempt，但 session file 保留了之前的 tool 调用历史，LLM 看到完整上下文，**用户无感知切换**。

---

### 场景3：无可替换 profile 也无 fallback model

```
shouldRotate 分支：
  advanceAuthProfile() → false（无可用 profile）
  fallbackConfigured → false

  → logAssistantFailoverDecision("surface_error")  ← 只记日志
  → fall-through 到正常返回路径
  → buildEmbeddedRunPayloads()
      formatAssistantErrorText() → "⚠️ API rate limit reached. Please try again later."
  → 返回给用户，isError: true
```

**降级返回**，不抛异常，不崩溃，用户收到一条带 isError 标记的友好错误提示。

---

## 五、pi-sdk 在项目中的完整使用范围

Agent loop 是最核心的使用场，但 pi-sdk 是整个项目的底层 LLM 基础设施：

| 使用场景 | 核心文件 | 说明 |
|---|---|---|
| 主对话 tool-use 循环 | pi-embedded-runner/run/attempt.ts | agent loop 内 |
| 对话历史压缩 | agents/compaction.ts | generateSummary + estimateTokens，loop 结束后独立运行 |
| TTS 文本处理 | tts/tts-core.ts | completeSimple 单次调用，完全独立 |
| 轻量单次 LLM 调用 | agents/simple-completion-transport.ts | 不走完整 loop，用于 auto-topic-label / memory summary 等 |
| 模型发现探测 | agents/pi-model-discovery.ts | 探测 provider 支持哪些模型 |
| 上下文引擎类型 | context-engine/legacy.ts | 仅类型依赖 |