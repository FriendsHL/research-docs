Fetching document 2753190472......
# Gateway 模块源码解析


> 研究日期：2026-03-23 | 源码路径：`/app/src/gateway/`
>
>


---

## 一、定位

Gateway 是整个 OpenClaw 的**核心神经中枢**，是 `openclaw gateway start` 启动的进程本体。所有 Channel（Telegram/Discord/大象）、所有 Agent 运行、所有 Cron、所有 WebUI——全都挂在 Gateway 进程里。


---

## 二、物理结构

Gateway 是一个 HTTP + WebSocket **双协议服务器**，默认端口 18789。**没有用任何 Web 框架**（不是 Express/Fastify），完全原生 `node:http`。


> **为什么不用 Express？**<br>
> Gateway 是常驻进程，HTTP 端点极少，绝大多数通信走 WebSocket。原生 `node:http` 启动极快（<10ms）、内存占用小、完全可控，没必要为几个 HTTP 路由引入整个 web 框架。
>
>


---

## 三、HTTP 端点路由表

源码：`server-http.ts`、`openai-http.ts`、`openresponses-http.ts`


| 路径 | 处理器 | 功能 |
| --- | --- | --- |
| `POST /v1/chat/completions` | handleOpenAiHttpRequest | OpenAI 兼容 Chat API |
| `POST /v1/responses` | OpenResponses handler | OpenResponses API |
| `/hook/*` | handleHooksRequest | Webhook 入口 |
| `/slack/*` | handleSlackHttpRequest | Slack Events API |
| `/canvas/*` | handleCanvasHostRequest | Canvas UI 资源 |
| `/a2ui/*` | handleA2uiHttpRequest | A2UI 组件渲染 |
| `/control/*` | handleControlUiHttpRequest | 控制台 Web UI |


---

## 四、WebSocket 通信协议（三种帧）

源码：`protocol/` 目录

自定义 JSON 帧协议，三种帧类型：

```JSON
// 代码块
// 请求帧（客户端 → Gateway）
{ "type": "req", "id": "abc123", "method": "chat.send", "params": {...} }

// 响应帧（Gateway → 客户端）
{ "type": "res", "id": "abc123", "ok": true, "payload": {...} }

// 事件帧（Gateway → 客户端，主动推送）
{ "type": "event", "event": "agent", "payload": {...}, "seq": 42, "stateVersion": {...} }

```

**连接握手流程：**


1. 客户端发送 ConnectParams（含 minProtocol/maxProtocol、client info、auth token）
2. Gateway 返回 hello-ok（含 protocol 版本、server info、features、snapshot、policy）
3. 后续双向通信：req/res + 服务器主动 event 推送

**seq + stateVersion 设计（解决断线重连的经典问题）：**


- `seq`：递增序号，客户端检测丢帧后请求重新同步
- `stateVersion`：服务端状态版本号，断线重连时对比版本决定是否重新拉全量 snapshot
- 这解决了"断线期间服务端状态变了，客户端不知道错过了什么"的问题


---

## 五、Gateway Methods（RPC 方法，共 100+）

源码：`server-methods/`、`server-methods.ts`、`server-methods-list.ts`


| 分类 | 方法示例 | 说明 |
| --- | --- | --- |
| 系统 | `health`, `status`, `logs.tail` | 健康检查、状态、日志 |
| 配置 | `config.get/set/apply/patch/schema` | 读写配置 |
| Agent | `agent`, `agent.identity.get` | 触发 AI 对话 |
| 会话 | `sessions.list/preview/patch/reset/delete` | session 管理 |
| 消息 | `send`, `wake`, `chat.send/history/abort` | 发消息给 Agent |
| Agents | `agents.list/create/update/delete` | 多 Agent 管理 |
| Skills | `skills.status/install/update` | Skill 管理 |
| Cron | `cron.list/add/update/remove/run` | 定时任务 |
| Node | `node.list/describe/invoke/pair.*` | 远程 Node 管理 |
| Exec 审批 | `exec.approval.request/waitDecision/resolve` | 命令执行审批流 |


---

## 六、Gateway Events（主动推送，共 20+）

源码：`events.ts`


| 事件 | 触发时机 |
| --- | --- |
| `agent` | AI 输出流（每个 token/工具调用） |
| `chat` | 聊天界面流式消息 |
| `tick` | 定时心跳（每 5 秒） |
| `health` | 系统健康状态变更 |
| `heartbeat` | Agent Heartbeat 结果 |
| `cron` | Cron 任务执行结果 |
| `exec.approval.requested/resolved` | 命令审批请求/完成 |
| `node.pair.requested/resolved` | Node 配对 |
| `update.available` | 有新版本可用 |


---

## 七、启动顺序（共17步）

源码：`server-startup.ts`、`boot.ts`

```Plain
// 代码块
startGatewayServer()
 1. 读 config + 迁移旧配置格式
 2. 加载 secrets（KMS/env/file）
 3. 解析 Gateway Auth（token/password 模式）
 4. loadGatewayPlugins()            ← 加载 Channel Plugins
 5. loadGatewayModelCatalog()       ← 加载模型目录
 6. 绑定端口，创建 httpServer + wss
 7. startGatewaySidecars()          ← 启动辅助服务（见下）
 8. channelManager.startChannels()  ← 启动所有已配置 Channel
 9. cron.start()                    ← 启动定时任务调度器
10. startHeartbeatRunner()          ← 启动 Agent Heartbeat 轮询
11. startChannelHealthMonitor()     ← Channel 健康检查（每5分钟）
12. onAgentEvent(...)               ← 订阅 Agent 事件
13. recoverPendingDeliveries()      ← 恢复上次崩溃未发出的消息
14. attachGatewayWsHandlers()       ← 注册 WS 连接处理器
15. startGatewayDiscovery()         ← 局域网发现广播（mDNS/UDP）
16. startGatewayTailscaleExposure() ← 可选 Tailscale 暴露
17. httpServer.listen()             ← 开始监听端口

```

**第 7 步 startGatewaySidecars() 内部细节（源码 server-startup.ts）：**


- 清理过期的 session write lock（stale 阈值 30 分钟）
- 启动 Browser Control Server（可配置禁用）
- 启动 Gmail Watcher（hooks.gmail 配置项）
- 加载 Internal Hooks（从配置和目录自动发现）
- 启动所有 Channel（可通过 `OPENCLAW_SKIP_CHANNELS=1` 跳过，测试用）
- 触发 `gateway:startup` internal hook 事件（延迟 250ms）


---

## 八、Channel 健康监控（源码：`channel-health-monitor.ts`）

不只是检查"活没活"，而是一套精细的**自动恢复机制**：

**核心参数：**


- `checkIntervalMs`：检查间隔，默认 5 分钟
- `startupGraceMs`：启动后宽限期，默认 60 秒（避免刚启动就误判）
- `cooldownCycles`：两次重启之间至少间隔几个检查周期，默认 2 个（即 10 分钟冷却）
- `maxRestartsPerHour`：每小时最多重启次数，默认 3 次

**健康判断逻辑：**

```TypeScript
// 代码块
function isChannelHealthy(snapshot) {
  if (!isManagedAccount(snapshot)) return true;   // 未配置的账号不管
  if (!snapshot.running) return false;             // 进程挂了
  if (snapshot.connected === false) return false;  // 网络断开
  return true;
}

```

超过 `maxRestartsPerHour` 后停止自动重启，等待人工介入。


---

## 九、Exec Approval 审批系统（源码：`exec-approval-manager.ts`）

核心类 `ExecApprovalManager`（单例）：

```TypeScript
// 代码块
class ExecApprovalManager {
  create(request, timeoutMs, id?)  // 创建审批记录（同步）
  register(record, timeoutMs)      // 注册并返回 Promise（等待决策）
  resolve(recordId, decision, resolvedBy?)  // 解决审批
}

```

**为什么 register() 和 create() 分开？**<br>
`create()` 是同步的（只创建记录），`register()` 才开始等待。好处是可以先把记录广播给所有客户端，确认"注册成功"后再进入等待，不会因为网络时序丢失审批请求。

**对 Agent 透明：** Agent 只是"等待 exec 工具的结果"，完全不感知审批逻辑。任何连接 Gateway WS 的客户端（手机 App、WebUI、CLI）都能参与审批。

**超时与宽限期：** 已解决的记录保留 15 秒宽限期（`RESOLVED_ENTRY_GRACE_MS`），防止网络延迟导致的重复 `awaitDecision`。


---

## 十、ReplyDispatcher 串行队列（源码：`auto-reply/reply/reply-dispatcher.ts`）

```TypeScript
// 代码块
// 用 Promise 链实现严格有序投递
let sendChain: Promise<void> = Promise.resolve();
let pending = 1;  // 从1开始，代表一个"预占位"

```

**为什么 pending 从 1 开始？** 这是防止"消息还没入队就触发 idle"的竞态条件。Agent 开始运行时预占 1，运行结束调 `markComplete()` 才释放，保证所有消息发完后才触发 idle 信号。

**Human Delay：** 第一条 block 立即发（感知响应快），从第二条开始加 800-2500ms 随机延迟，模拟真人打字节奏。


---

## 十一、设计亮点总结


| 亮点 | 说明 |
| --- | --- |
| 无框架原生 node:http | 常驻进程 + 极少 HTTP 端点，原生性能更好 |
| WS seq + stateVersion | 解决断线重连状态同步问题 |
| Channel 独立崩溃恢复 | AbortController + 指数退避，Channel 互不影响 |
| Delivery Recovery | 崩溃前落盘，重启后重投递，"至少一次"语义 |
| Exec Approval 透明代理 | Agent 不感知审批逻辑，任意客户端可参与 |
| Channel 健康监控 | 精细恢复：宽限期 + 冷却 + 每小时限制次数 |
| Dispatcher pending 计数 | 保证消息顺序 + 精确 idle 信号，无竞态 |

Done

