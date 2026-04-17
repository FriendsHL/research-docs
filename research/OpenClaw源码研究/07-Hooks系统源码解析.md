Fetching document 2753002958......
# OpenClaw 源码精读：Hooks 系统（Internal Hooks + Plugin Hooks）

# OpenClaw 源码精读：Hooks 系统


> 源码路径：`/app/src/hooks/`
>
>


---

## 一、两套 Hook 系统（关键区分）

OpenClaw 实际上有**两套完全不同**的 Hook 系统，初看容易混淆：


| Internal Hooks | Plugin Hooks |
| --- | --- |
| 定义方式 | `HOOK.md`（frontmatter）+ `handler.ts`（文件系统） | `api.on("hook_name", handler)`（代码） |
| 作者 | 用户/工作区开发者 | Plugin 开发者（npm 包） |
| 事件粒度 | **粗**（message / session / command / agent / gateway） | **细**（before_tool_call / llm_input / 共25种） |
| 是否可拦截 | 只能监听，**不能修改/阻断** | **可以修改参数、阻断执行**（如拒绝 tool call） |
| 注册时机 | Gateway 启动时扫描文件系统 | Plugin 加载时调用 `api.on()` |
| 存放位置 | `hooks/bundled/`、工作区、npm | Plugin npm 包内部 |


---

## 二、Internal Hooks：文件即 Hook

### 2.1 结构

每个 Internal Hook 是一个目录，包含两个文件：

```Plain
// 代码块
hooks/my-hook/
├── HOOK.md       # 声明 hook 元数据（events、条件、描述）
└── handler.ts    # 具体处理逻辑

```

`HOOK.md` frontmatter 示例：

```YAML
// 代码块
---
events: 
always: false
emoji: "🔔"
requires:
  bins: 
  env: 
---
这个 Hook 在收到新消息时转发到 Slack

```

### 2.2 事件类型（源码实际定义）

```TypeScript
// 代码块
type InternalHookEventType = "command" | "session" | "agent" | "gateway" | "message";

```

具体 action 组合：


| type | action | 触发位置 | 触发时机 |
| --- | --- | --- | --- |
| `command` | `new` | `commands-core.ts` | 用户发 `/new` |
| `command` | `reset` | `commands-core.ts` | 用户发 `/reset` |
| `agent` | `bootstrap` | `agents/bootstrap-hooks.ts` | agent 启动，加载工作区文件时 |
| `gateway` | `startup` | `gateway/server-startup.ts` | Gateway 进程启动 |
| `message` | `received` | `auto-reply/reply/dispatch-from-config.ts` | 收到 IM 消息 |
| `message` | `sent` | `infra/outbound/deliver.ts` | 发出 IM 消息 |

注意：`command` 事件指的是**斜杠命令**（`/new`、`/reset`），不是编程意义上的命令。

### 2.3 Hook 匹配逻辑（源码）

```TypeScript
// 代码块
// internal-hooks.ts
export async function triggerInternalHook(event: InternalHookEvent) {
  const typeHandlers = handlers.get(event.type) ?? [];           // 监听 "message"（不含action）
  const specificHandlers = handlers.get(`${event.type}:${event.action}`) ?? [];  // 监听 "message:received"
  
  const allHandlers = ;
  for (const handler of allHandlers) {
    await handler(event);  // 错误被 catch，不阻断其他 handler
  }
}

```

在 `HOOK.md` 里写 `events: ` 可以同时匹配 message:received 和 message:sent；写 `events: ` 只匹配收消息。

### 2.4 内置 Hook（bundled）

```Plain
// 代码块
hooks/bundled/
├── session-memory/    # /new 时将对话摘要写入 MEMORY.md
├── command-logger/    # 记录所有斜杠命令到日志
├── boot-md/           # agent 启动时注入 BOOT.md 到上下文
└── bootstrap-extra-files/  # agent:bootstrap 时注入额外工作区文件
                            # （AGENTS.md、SOUL.md、TOOLS.md 等）
                            # 适合 monorepo 场景，让 agent 开箱即用

```

### 2.5 agent:bootstrap 事件详解

这个事件最特殊，值得单独说明：

```TypeScript
// 代码块
type AgentBootstrapHookContext = {
  workspaceDir: string;
  bootstrapFiles: WorkspaceBootstrapFile[];  // 已准备好注入的文件列表
  cfg?: OpenClawConfig;
  sessionKey?: string;
  sessionId?: string;
  agentId?: string;
};

```

`bootstrap-extra-files` Hook 监听这个事件，在 agent 启动时把 `SOUL.md`、`AGENTS.md` 等文件追加到 `bootstrapFiles` 里，agent 就能在第一条消息中读到这些上下文。

### 2.6 Hook 的四种来源（source 字段）

```TypeScript
// 代码块
type HookSource =
  | "openclaw-bundled"    // 内置在 OpenClaw 里
  | "openclaw-managed"    // 通过 openclaw hooks install 安装
  | "openclaw-workspace"  // 用户工作区自定义
  | "openclaw-plugin";    // 由 Plugin 注册

```


---

## 三、Gmail Watcher：Hooks 里的独立子系统

`/app/src/hooks/gmail*.ts` 是一个独立的子系统，和上面的 Hook 机制关系不大，而是：


1. 启动 `gog gmail watch serve` 子进程
2. 接收 Gmail Pub/Sub 推送（新邮件通知）
3. 将新邮件事件转换成 OpenClaw `message:received` Hook 事件
4. 触发 agent 处理（相当于把 Gmail 变成一个 IM 渠道）


---

## 四、Plugin Hooks：深度拦截 LLM 流程

### 4.1 注册方式

Plugin 通过 `api.on()` 注册，支持优先级：

```TypeScript
// 代码块
api.on("before_tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.params.command.includes("rm -rf")) {
    return { block: true, blockReason: "危险命令被拦截" };
  }
}, { priority: 10 });  // 数字越小优先级越高

```

### 4.2 完整事件列表（源码）

```TypeScript
// 代码块
type PluginHookName =
  // ── LLM 流程控制 ──────────────────────────────────────
  | "before_model_resolve"    // 可覆盖：modelOverride, providerOverride
  | "before_prompt_build"     // 可注入：systemPrompt, prependContext
  | "before_agent_start"      // legacy 兼容，同时包含以上两者功能
  | "llm_input"               // 只读：完整 messages 列表（prompt + history）
  | "llm_output"              // 只读：assistantTexts, usage token 统计
  | "agent_end"               // agent turn 结束
  
  // ── 上下文压缩 ────────────────────────────────────────
  | "before_compaction"       // 知道：messageCount, tokenCount, sessionFile 路径
  | "after_compaction"        // 知道：compactedCount, sessionFile 路径
  | "before_reset"            // /new 或 /reset 前触发
  
  // ── 工具调用 ─────────────────────────────────────────
  | "before_tool_call"        // 可修改 params，可 block（含 blockReason）
  | "after_tool_call"         // 只读：result, error, durationMs
  | "tool_result_persist"     // 工具结果写入 transcript 前
  | "before_message_write"    // 消息写入 transcript 前
  
  // ── 消息收发 ─────────────────────────────────────────
  | "message_received"        // 收到 IM 消息
  | "message_sending"         // 发出前，可修改 content，可 cancel
  | "message_sent"            // 发出后
  
  // ── Session 生命周期 ──────────────────────────────────
  | "session_start"
  | "session_end"
  
  // ── Subagent 生命周期 ─────────────────────────────────
  | "subagent_spawning"
  | "subagent_delivery_target"
  | "subagent_spawned"
  | "subagent_ended"
  
  // ── Gateway 生命周期 ──────────────────────────────────
  | "gateway_start"
  | "gateway_stop";

```

### 4.3 能拦截/修改的 Hook 详解

`**before_model_resolve**`：修改 model 选择

```TypeScript
// 代码块
api.on("before_model_resolve", async (event, ctx) => {
  // event.prompt 可用，但 messages 还没准备好
  if (event.prompt.includes("复杂分析")) {
    return { modelOverride: "claude-opus-4-5", providerOverride: "anthropic" };
  }
});

```

`**before_prompt_build**`：注入 system prompt

```TypeScript
// 代码块
api.on("before_prompt_build", async (event, ctx) => {
  return {
    systemPrompt: "你是一个专注于代码安全的助手，优先考虑安全性。",
    prependContext: `当前项目：${ctx.workspaceDir}`,
  };
});

```

`**before_tool_call**`：拦截危险工具调用

```TypeScript
// 代码块
api.on("before_tool_call", async (event, ctx) => {
  // ctx.toolName, event.params
  if (event.toolName === "exec" && isDangerousCommand(event.params)) {
    return { block: true, blockReason: "安全策略：禁止该命令" };
  }
  // 也可以修改参数
  return { params: { ...event.params, timeout: 30 } };
});

```

`**message_sending**`：修改或取消发出的消息

```TypeScript
// 代码块
api.on("message_sending", async (event, ctx) => {
  if (containsSensitiveInfo(event.content)) {
    return { cancel: true };  // 阻止发送
  }
  return { content: redact(event.content) };  // 脱敏后发送
});

```


---

## 五、两套 Hook 系统的选择原则

```Plain
// 代码块
需要监听 IM 消息/Gateway 启动/agent 启动？
    → 用 Internal Hook（HOOK.md + handler.ts）
    → 简单，任何人都能写，不需要打包

需要拦截/修改 LLM 的 prompt、tool call、输出？
    → 用 Plugin Hook（api.on）
    → 必须在 Plugin 里注册，但功能更强大

```

Done

