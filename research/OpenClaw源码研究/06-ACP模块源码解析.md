---
modified: '2026-03-30T17:06:12.543136'
---

Fetching document 2752923313......
Done
# OpenClaw 源码精读：ACP 模块（Agent Communication Protocol）


> 源码路径：`/app/src/acp/`
>
>


---

## 一、ACP 是什么？

ACP（Agent Communication Protocol）是 OpenClaw 与外部 Coding Agent 之间的**结构化通信协议层**。它不是一个 IM 协议，而是一套让 OpenClaw 能够"驱动"外部 coding agent（如 acpx / Claude Code）的机制。


---

## 二、ACP vs tmux：两种完全不同的哲学

OpenClaw 在接入 coding agent 时有两种方式，理解它们的区别是理解 ACP 的关键：

```Plain
// 代码块
tmux 方式（传统）            ACP 方式（现代）
──────────────              ──────────────
面向"人类终端"               面向"程序"
输出 = 字符流（含ANSI）      输出 = 结构化 JSON 事件
取消 = 发 SIGINT 信号        取消 = 发 cancel 事件
无法区分"工具调用"和"文字"    可精确知道每个工具调用的名称/参数/结果
难以解析进度                 实时感知 agent 的每一步状态

```

一句话：**tmux 是给人看的，ACP 是给程序解析的。**


---

## 三、核心文件结构

```Plain
// 代码块
/app/src/acp/
├── types.ts              # 协议类型定义
├── server.ts             # ACP HTTP Server（接受外部 coding agent 连接）
├── session.ts            # 管理"入站"ACP 客户端连接（谁连进来了）
├── session-mapper.ts     # ACP session ↔ OpenClaw session 映射
├── translator.ts         # pi-agent-core 事件 → ACP 标准事件转换
├── policy.ts             # 安全策略（哪些 session 可以被连接）
├── client.ts             # ACP 客户端（OpenClaw 主动连接外部 agent）
├── commands.ts           # ACP 命令处理
├── event-mapper.ts       # 事件格式映射
├── control-plane/        # 管理"出站"派发给 coding agent 的 session
│   ├── manager.ts        # 核心：AcpControlPlaneManager
│   ├── manager.core.ts   # 生命周期管理
│   ├── manager.runtime-controls.ts  # 取消/暂停/恢复
│   ├── spawn.ts          # 启动新 ACP session
│   └── runtime-cache.ts  # 运行时实例缓存
└── runtime/              # ACP 运行时接口
    ├── registry.ts       # 注册后端实现（registerAcpRuntimeBackend）
    ├── types.ts          # 运行时接口定义（AcpRuntimeBackend）
    └── session-identifiers.ts

```


---

## 四、两个方向：入站 vs 出站

ACP 有两个方向，很容易混淆：

**入站（Inbound）**：外部 coding agent 主动连接进 OpenClaw

```Plain
// 代码块
外部 Claude Code / acpx → HTTP POST → ACP Server（server.ts）
                                         ↓
                                     session.ts 管理连接
                                         ↓
                                     session-mapper 映射到 OpenClaw session

```

**出站（Outbound / Control Plane）**：OpenClaw 主动驱动外部 coding agent

```Plain
// 代码块
用户发消息 → sessions_spawn(runtime="acp")
                  ↓
          control-plane/manager.ts
                  ↓
          spawn.ts 启动 pi-coding-agent（rpc 模式）
                  ↓
          acpx 通过 stdin 发命令，stdout 接收事件流

```


---

## 五、ACP Session 的生命周期（出站方向）

```TypeScript
// 代码块
type AcpSession = {
  sessionId: SessionId;      // 来自 @agentclientprotocol/sdk
  sessionKey: string;        // OpenClaw session key
  cwd: string;               // coding agent 工作目录
  createdAt: number;
  lastTouchedAt: number;
  abortController: AbortController | null;  // 优雅取消用
  activeRunId: string | null;               // 当前正在执行的 run ID
};

```

关键：`abortController` 是优雅取消的核心，发 `cancel` 事件后通过它通知 coding agent 停止当前 run，而不是粗暴地 kill 进程。


---

## 六、ACP vs acpx：接口与实现的分离

```Plain
// 代码块
ACP（接口/协议层）              acpx（具体实现层）
/app/src/acp/              /app/extensions/acpx/src/
─────────────              ──────────────────────────
定义通信格式                  实现实际的 coding agent 行为
定义 AcpRuntimeBackend 接口   通过 registerAcpRuntimeBackend("acpx", ...) 注册
相当于"插座规格"              相当于"插头"

```

`runtime/registry.ts` 维护后端注册表，支持替换不同的 coding agent 实现（以后可以接入 Gemini CLI、Codex CLI 等任何支持 ACP 的 agent）。


---

## 七、ACP vs Subagent：两种 agent 派发机制


| ACP Session | Subagent |
| --- | --- |
| 执行方式 | **独立进程**（外部 coding agent） | 进程内函数调用（同一 Node.js 进程） |
| 会话持久性 | **持久**（可恢复，JSONL 文件） | 一次性 |
| 取消能力 | **优雅取消**（cancel 事件） | 无（或强制中断） |
| 事件流 | **结构化事件流**（tool_call、message_update…） | 无 |
| 工具访问 | coding agent 自带工具集（read/bash/edit/write） | OpenClaw 工具集 |
| 定位 | **一等公民 coding agent**，适合复杂代码任务 | **轻量任务派发**，适合简单 AI 问答 |

何时用哪个：


- **代码开发任务** → ACP（sessions_spawn runtime="acp"）
- **调研/问答/简单自动化** → Subagent（sessions_spawn runtime="subagent"）


---

## 八、AcpServerOptions：服务端配置要点

```TypeScript
// 代码块
type AcpServerOptions = {
  gatewayUrl?: string;
  gatewayToken?: string;
  defaultSessionKey?: string;         // 默认落到哪个 session
  defaultSessionLabel?: string;       // session 标签
  requireExistingSession?: boolean;   // 是否拒绝创建新 session（安全控制）
  resetSession?: boolean;             // 是否在连接时重置 session
  prefixCwd?: boolean;                // 是否在 cwd 前加前缀
  sessionCreateRateLimit?: {          // 防止滥用创建 session
    maxRequests?: number;
    windowMs?: number;
  };
  verbose?: boolean;
};

```

`requireExistingSession` 是关键安全参数：设为 true 时，外部 coding agent 只能连入已存在的 session，不能随意创建新 session（防止资源耗尽攻击）。


---

## 九、translator.ts：事件翻译层

这是 ACP 中最核心的转换逻辑：

```Plain
// 代码块
pi-agent-core 原生事件          ACP 标准事件
──────────────────              ──────────────
AgentEvent (agent_loop)    →    ACPEvent (start/output/tool_use/end)

```

为什么需要翻译？pi-agent-core 是内部实现，事件格式是内部约定；ACP 是对外接口，格式由 `@agentclientprotocol/sdk` 定义。translator 做这两套格式之间的适配，保持内部实现可以自由变化而不影响外部 coding agent。


---

## 十、完整调用链

```Plain
// 代码块
 "帮我实现这个功能"
      ↓
 路由到 ACP handler
      ↓

      ↓
 AcpControlPlaneManager
      ↓
 启动 pi-coding-agent 进程（rpc 模式）
      ↓
 通过 stdin 发 RpcCommand:
  {"type":"prompt","message":"帮我实现这个功能"}
      ↓
 执行 Agent Loop：read → bash → edit → write
      ↓
 返回 AgentSessionEvent 流
      ↓
 AgentSessionEvent → ACP 标准事件
      ↓
 映射到 OpenClaw session
      ↓
 推送进度到大象

```


---

## 补充：ACP Control-Plane 生命周期深度解析（2026-03-30）

### 一、ACP 整体定位修正

ACP 分两层：

```
Control-Plane（OpenClaw 侧）
    AcpSessionManager（全局单例）
    负责：session 生命周期、cancel/close、runtime 缓存、并发控制
        ↓
AcpRuntime（外部 Coding Agent 侧）
    acpx / Claude Code / Codex 进程（本机进程，非远程服务）
    执行实际代码任务，返回结构化事件流
```

**关键认知**：
- acpx 是**本机进程**，不是远程服务
- ACP 连接是**流式同步**（for-await 消费事件流），不是 MQ 式异步
- OpenClaw sessionKey → acpx identity（backendSessionId + agentSessionId）双层映射

---

### 二、Session 状态与三种 Mode

**Session 状态**：
```
none   → 不是 ACP session，走普通 embedded agent
stale  → 曾是 ACP session 但 meta 丢失，属于异常
ready  → 正常 ACP session（meta.state: idle | running | error）
```

**三种 Session Mode**：

| mode | 含义 | 特点 |
|---|---|---|
| `persistent` | 长期存在，跨 turn 保留历史 | 有 resumeSessionId，可从断点恢复 |
| `oneshot` | 单次执行，完成即关闭 | turn 结束后自动 `runtime.close()` |
| `transient` | 临时会话 | 不保存 resumeSessionId |

---

### 三、SessionActorQueue：串行化保证

每个 sessionKey 的所有操作通过 `withSessionActor` 串行排队，保证不出现竞态：

```
runTurn 和 cancelSession 不会同时执行
initializeSession 和 closeSession 不会竞争
```

**唯一例外**：`cancelSession` 的快速路径——有活跃 turn 时直接 abort AbortController，不排队，立即生效。

---

### 四、Cancel / Close / Stop 的区别

| 操作 | 含义 | 影响 |
|---|---|---|
| `cancelSession` | **中断当前 turn**，session 继续存在 | state → idle，可继续发新消息 |
| `closeSession` | **彻底关闭 session**，清理外部进程 | 可选 clearMeta |
| `closeSession(clearMeta=true)` | 彻底销毁包括持久化 meta | 就像从未存在过 |

**Cancel 两条路径**：
```
有活跃 turn（快速路径）：
    activeTurn.abortController.abort()
    runtime.cancel({ handle, reason })
    → 立即生效，不排队

无活跃 turn：
    withSessionActor() 排队
    → ensureRuntimeHandle → runtime.cancel() → setSessionState(idle)
```

---

### 五、coding agent 的返回信息流向

**coding agent 的输出直接投递给用户，完全绕过 main agent**：

```
coding agent 返回内容
    ↓
AcpReplyProjector.onEvent(event)
    ├── text_delta  → sendBlockReply()（流式文字）
    ├── tool_call   → sendToolResult()（工具调用摘要）
    └── status      → 系统状态提示
    ↓
直接投递到绑定的 Discord thread / Telegram topic
← main agent 完全不知道
```

`dispatch-from-config.ts` 里的路由判断是互斥的：

```typescript
const acpResolution = acpManager.resolveSession({ cfg, sessionKey });
if (acpResolution.kind === "none") {
    return null;  // 不是 ACP session → 走 main agent
}
// 是 ACP session → 直接走 ACP dispatch，main agent 完全绕过
await acpManager.runTurn({ ... });
```

**ACP session 建立并绑定 thread 后，main agent 退出这个对话。** 只能通过：
- `/acp steer <instruction>` — 从外部插话（mode: "steer"）
- `/acp close` — 收回控制权

---

### 六、三种启动 ACP 的方式

**方式1：用户手动命令**
```
/acp spawn codex --mode persistent --thread auto
```
用户主动建立长期 ACP session，后续在绑定 thread 里直接和 coding agent 对话。

**方式2：main agent 自主调用工具（最常用）**

main agent 通过内置工具 `sessions_spawn` 自动触发：
```json
{
  "tool": "sessions_spawn",
  "params": {
    "task": "重构 src/utils.ts 里的 parseDate 函数",
    "runtime": "acp",
    "agentId": "codex",
    "thread": true,
    "mode": "run"
  }
}
```
用户说"帮我重构这个函数" → main agent 决策 → 调用 `sessions_spawn` → ACP session 建立 → coding agent 工作 → 结果直接给用户。

**方式3：Cron Job / 程序化触发**

通过 `spawnAcpDirect()` 函数自动触发，适合自动化场景。

**`sessions_spawn` 的两个 runtime 选项**：

| runtime | 含义 |
|---|---|
| `"acp"` | 启动外部 coding agent（acpx/Codex），走 ACP 协议 |
| `"subagent"` | 启动 OpenClaw 自己的子 Agent（embedded runner） |

---

### 七、Identity Reconcile：session 身份同步

外部 coding agent 有自己的 session ID 体系，OpenClaw 通过双层映射维持对应关系：

```
OpenClaw sessionKey（如 agent:codex:acp:uuid）
    ↓ 映射
外部 agent identity：
    backendSessionId  ← acpx 进程自己的 session 标识
    agentSessionId    ← coding agent（Codex）的 session 标识
```

`persistent` mode 下，`agentSessionId` 作为 `resumeSessionId` 存入 meta，下次重建连接时带上，外部 coding agent 恢复上次对话上下文。

**Gateway 重启后的恢复流程**：
```
reconcilePendingSessionIdentities()
    └── 扫描所有 state=pending 的 ACP session
        └── 向 acpx 查询真实 session ID
            └── 更新 meta.identity
```

---

### 八、完整对话场景示例

```
用户: /acp spawn codex --thread auto
    └── main agent 处理命令
        └── 创建 sessionKey: "agent:codex:acp:xxx"，绑定到当前 thread
        └── 回复: "✅ Spawned ACP session..."

用户: "帮我重构这个函数"
    └── dispatch-from-config → resolveSession → kind="ready"（ACP session）
    └── runTurn(text="帮我重构...") → coding agent
    └── coding agent 输出流式文字 → 直接到用户（main agent 不知道）

coding agent: "需要确认：是否删除旧实现文件？"
    └── projector → 直接投递给用户（main agent 不知道）

用户: "确认，删除"
    └── dispatch-from-config → resolveSession → kind="ready"（同一 ACP session）
    └── runTurn(text="确认，删除") → coding agent 继续执行（main agent 不知道）

用户: /acp close
    └── main agent 处理命令 → closeSession → 收回控制权
```