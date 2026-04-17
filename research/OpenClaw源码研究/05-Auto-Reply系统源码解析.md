---
modified: '2026-03-30T11:47:29.766389'
---

Fetching document 2752851211......
# Auto-Reply 系统源码解析


> 研究日期：2026-03-23 | 源码路径：`/app/src/auto-reply/`
>
>


---

## 一、定位

Auto-Reply 系统是消息从 Channel 进来到 AI 回复出去的**完整链路**，是 OpenClaw 最核心的业务流程层。


---

## 二、整体数据流

```Plain
// 代码块
用户发消息
  ↓
Channel Plugin（Telegram/大象/Discord...）
  收到原始消息，包装成 MsgContext
  ↓
inbound-debounce（防抖聚合）
  同一对话窗口内的连续消息合并
  ↓
inbound-dedupe（去重）
  同一 MessageSid 不处理两次
  ↓
dispatchReplyFromConfig()  ← 核心调度入口（dispatch.ts）
  ├── 触发 plugin hook: message_received
  ├── 触发 internal hook: message/received（HOOK.md 系统）
  ├── 检查 fast abort（用户发了 stop/取消）
  ├── 检查 send policy（是否允许发送）
  └── 检查是否走 ACP dispatch（子 Agent 流程）
  ↓
getReplyFromConfig()（reply.ts）
  ├── 解析 Agent 配置（workspace、model、timeout...）
  ├── 解析 Directive（/think、/verbose、/model 等指令）
  ├── 检测 Slash Commands（/new、/status、/compact...）
  └── 处理 Inline Actions（媒体理解、链接预览等）
  ↓
runPreparedReply()（reply.ts）
  ├── 检查是否有活跃的 Agent Run（queue policy）
  └── 调用 runReplyAgent() ← 启动 Agent Loop
        ├── 构建 system prompt（SOUL.md + 工具列表 + Inbound Meta）
        ├── 构建用户消息（含媒体附件、群聊上下文）
        └── 调用 pi-embedded（嵌入式 Agent Loop）→ 流式产出 AgentEvent
  ↓
ReplyDispatcher（发送控制器，reply-dispatcher.ts）
  ├── sendToolResult()   工具调用结果（实时发）
  ├── sendBlockReply()   流式回复块（边生成边发）
  └── sendFinalReply()   最终完整回复
        ├── normalize（过滤 HEARTBEAT_OK、NO_REPLY 等特殊 token）
        ├── 可选 human delay（800ms-2500ms 随机延迟）
        ├── 可选 TTS 转换（语音回复）
        └── deliver() → Channel Plugin outbound → 发出去

```


---

## 三、MsgContext：消息的全量描述（源码：`templating.ts`）

每条进来的消息被包装成 MsgContext，核心字段：

```TypeScript
// 代码块
type MsgContext = {
  // 来源
  From: string;         // 发送方 ID
  Provider: string;     // Channel 类型（"telegram"/"daxiang"...）
  ChatType: "direct" | "group" | "thread";

  // 内容
  Body: string;         // 文本内容
  MediaType?: string;   // 媒体类型（"image/jpeg"...）
  MediaUrl?: string;    // 媒体 URL

  // 路由
  SessionKey: string;   // 会话唯一键
  MessageSid: string;   // 去重 ID

  // 跨 Channel 路由
  OriginatingChannel?: string;
  OriginatingTo?: string;

  // 授权
  CommandSource: "channel" | "native";
};

```


---

## 四、三种消息前置过滤

### 防抖（`inbound-debounce.ts`）

用户快速连续发消息时，等一个窗口期再统一处理。同一 key 的消息在窗口期内不断重置计时器（标准防抖）。

```TypeScript
// 代码块
const enqueue = async (item: T) => {
  const existing = buffers.get(key);
  if (existing) {
    existing.items.push(item);     // 追加到缓冲区
    scheduleFlush(key, existing);  // 重置计时器
    return;
  }
  // 新建缓冲区，启动计时
};

```


- `debounceMs` 可全局配置，也可按 Channel 单独配置（`messages.inbound.byChannel`）
- `debounceMs=0` 时消息直接透传，不缓冲

### 去重（`inbound-dedupe.ts`）

Key：`provider|accountId|sessionKey|peerId|threadId|messageId` 组合


- TTL：20 分钟
- 最大缓存：5000 条
- 场景：网络重试导致同一条消息被投递两次时直接丢弃

### Fast Abort（`chat-abort.ts`）

消息内容匹配 "stop/cancel/停止" 等关键词时，立即中止正在运行的 Agent，不走 LLM，直接返回中止确认。


---

## 五、Directive 系统（源码：`reply/` 目录）

Directive 是内嵌在消息里的行内指令，分两类：

**Fast Lane（无需 LLM 直接处理）：**


| 指令 | 作用 |
| --- | --- |
| `/status` | 显示当前 session 状态 |
| `/help` | 显示帮助信息 |
| `/new` | 重置 session |
| `/compact` | 手动触发上下文压缩 |

**Inline Directive（影响 LLM 调用参数，per-session 持久）：**


| 指令 | 作用 | 持久性 |
| --- | --- | --- |
| `/think low/high/max` | 控制 Reasoning 强度 | per-session 持久 |
| `/verbose on/off` | 工具调用是否展示细节 | per-session 持久 |
| `/model gpt-4o` | 临时切换模型 | per-session 持久 |
| `/reasoning on/off` | 是否开启 Chain of Thought | per-session 持久 |
| `/elevated` | 申请提权（执行高风险命令） | per-session 持久 |

**持久性实现：** per-session Directive 存储在 SessionEntry 里（`modelOverride`、`thinkingLevel`、`verboseLevel`），写入 Session Store JSON 文件，下次会话继续生效。


---

## 六、Queue Policy（并发控制，源码：`reply.ts`）

当 Agent 正在运行时，新消息进来的处理策略：

```Plain
// 代码块
resolveActiveRunQueueAction():

当前无活跃 Run → run-now（直接执行）
当前有活跃 Run：
  - 是 Heartbeat → drop（丢弃，不打断 Agent）
  - 配置了 steer 模式 → enqueue-followup（加入队列，追加指令）
  - 默认 → run-now（中断当前 Run，开新的）

```

**steer 模式：** 新消息不打断正在运行的 Agent，而是在 Agent 运行完后追加进去，适合"长任务 + 用户追加说明"的场景。


---

## 七、ReplyDispatcher：三种消息类型（源码：`reply/reply-dispatcher.ts`）


| 类型 | 方法 | 触发时机 | 用户感知 |
| --- | --- | --- | --- |
| 工具结果 | `sendToolResult()` | 工具执行期间 | "正在搜索..." 之类的实时状态 |
| 流式块 | `sendBlockReply()` | LLM 流式输出 | 打字机效果，边生成边发 |
| 最终回复 | `sendFinalReply()` | Agent 完成后 | 完整最终答案 |

**normalize 处理：**


- 过滤 `NO_REPLY`（静默回复标记）
- 过滤 `HEARTBEAT_OK`（心跳确认，不发给用户）
- 过滤空消息
- 应用 response prefix 模板

**Human Delay（模拟真人打字节奏）：**

```TypeScript
// 代码块
// 第一条 block 立即发，从第二条开始加随机延迟
const DEFAULT_HUMAN_DELAY_MIN_MS = 800;
const DEFAULT_HUMAN_DELAY_MAX_MS = 2500;
const shouldDelay = kind === "block" && sentFirstBlock;

```

**串行队列保证顺序：**

```TypeScript
// 代码块
// 所有发送操作排入同一个 Promise 链，严格有序（tool → block → final）
sendChain = sendChain.then(async () => {
  if (shouldDelay) await sleep(delayMs);
  await options.deliver(normalized, { kind });
});

```


---

## 八、TTS 处理管线

```Plain
// 代码块
maybeApplyTtsToPayload()
  - 用户是语音消息发来的? → 自动 TTS 回复（ttsAuto 模式）
  - 配置了 tts.mode = "final"? → 对最终回复生成语音
  - block streaming 完成后，累积文本统一生成一条语音

```

语音生成在 `sendFinalReply()` 环节，不在 streaming 阶段——TTS 需要完整文本才能合成。


---

## 九、跨 Channel 路由

当 `OriginatingChannel != 当前 Surface` 时触发。

**场景：** Telegram 用户触发任务，session 挂在 Slack 上处理，回复发回 Telegram 而不是 Slack。这让一个 Agent 可以接受多个 Channel 的输入，统一处理，分别回复。


---

## 十、Block Streaming（打字机效果，各 Channel 实现不同）

`sendBlockReply()` 每次被调用就发一段，Channel Plugin 把这些 block 编辑/追加到同一条消息里：


| Channel | 实现方式 |
| --- | --- |
| Telegram | `edit_message_text` API，不断更新同一条消息 |
| 大象（Daxiang） | 类似 Telegram，追加更新 |
| WebUI | WebSocket event 推流 |


---

## 十一、与 Memory 系统的交互点

Auto-Reply 在构建 system prompt 时调用 Memory 搜索（`memory_search` 工具），是两个模块的关键连接点。Memory 搜索的结果以 `Source: MEMORY.md#12` 格式注入到 Agent 上下文。


---

## 十二、设计亮点总结


| 亮点 | 说明 |
| --- | --- |
| 防抖 + 去重双重保护 | 防止消息重复处理，可按 Channel 细粒度配置 |
| Fast Abort | stop 关键词立即中断，不等 LLM |
| Directive 行内指令 | 无需单独界面，消息里就能控制 Agent 行为 |
| steer 队列模式 | 长任务不被打断，新消息追加为指令 |
| 串行 Promise 链 | 严格保证 tool → block → final 顺序 |
| Human Delay | 从第二条 block 开始随机延迟，模拟真人节奏 |
| 跨 Channel 路由 | 一个 Agent 多 Channel 输入，分别回复 |

Done


---

## 补充：并发控制与 Queue Policy 深度解析（2026-03-30）

### 一、整体架构：三道关卡

消息从进来到 Agent 开始处理，经过三道关卡：

```
用户消息到达
    │
    ▼
① inbound-debounce（防抖聚合）
    同一对话窗口内连续消息合并，等静默期结束才批量触发
    │
    ▼
② dispatch-from-config（调度入口）
    去重 → 快速中止检测 → send policy → ACP分流
    │
    ▼
③ Queue Policy（并发控制核心）
    Agent 正在跑？→ 根据 mode 决定：run-now / enqueue-followup / drop
```

---

### 二、防抖层（inbound-debounce.ts）

按 SessionKey 分组的 `DebounceBuffer`：

```typescript
type DebounceBuffer<T> = {
  items: T[];          // 积累的消息
  timeout: ...|null;   // 定时器，每来一条消息都重置
  debounceMs: number;
};
```

你连续发 3 条消息，每条到来都会重置定时器，等静默 `debounceMs` 毫秒后，3 条消息作为一批一起触发。

每个 SessionKey 维护独立的等待链（`keyChains`），保证同一对话消息不乱序、不"超车"。

```yaml
messages:
  inbound:
    debounceMs: 500       # 全局防抖
    byChannel:
      telegram: 300        # 按渠道覆盖
```

---

### 三、快速中止检测（abort.ts::tryFastAbortFromMessage）

消息进来后**第一件事**是判断是否是中止指令（stop/取消等关键词），如果是：

1. `abortEmbeddedPiRun(sessionId)` — 中止正在跑的 Agent Loop
2. `clearSessionQueues(keys)` — 清空等待队列里所有排队消息
3. 级联递归停止子 Agent（`stopSubagentsForRequester`）
4. ACP session 一并 cancel

这是**最高优先级**，比 Queue Policy 更早，直接绕过所有排队逻辑，用户立即收到 `"⚙️ Agent was aborted."`。

---

### 四、Queue Policy：并发控制核心（queue-policy.ts）

决策函数：

```typescript
resolveActiveRunQueueAction({
  isActive: boolean;      // Agent 是否正在跑
  isHeartbeat: boolean;   // 是否是心跳消息
  shouldFollowup: boolean;
  queueMode: QueueMode;
}): "run-now" | "enqueue-followup" | "drop"
```

决策树：

```
Agent 没在跑？            → run-now
是心跳消息？              → drop（静默丢弃）
shouldFollowup=true
  OR queueMode="steer"？  → enqueue-followup（进队列）
其他情况？                → run-now（并发执行）
```

---

### 五、六种 Queue Mode

| mode | 行为 | 适用场景 |
|---|---|---|
| `collect`（默认） | Agent 跑完后，把所有排队消息**合并成一条**发给 Agent | 用户连续追问，让 Agent 一次性看到所有问题 |
| `followup` | Agent 跑完后，**逐条依次**执行排队消息 | 每条消息都要单独回复 |
| `steer` | 新消息**插入**当前 Agent 上下文（不等完成），实时引导 | 用户想修正 Agent 的方向 |
| `steer-backlog` | steer + 处理积压消息 | steer 的增强版 |
| `interrupt` | 新消息到来，**中断**当前 Agent，重新开始 | 用户完全改变了意图 |
| `queue` | 排队顺序执行，不合并 | 需要严格顺序处理 |

```yaml
messages:
  queue:
    mode: collect          # 全局默认
    byChannel:
      telegram: followup   # 按渠道覆盖
    debounceMs: 1000       # 队列防抖（不同于 inbound 防抖）
    cap: 20                # 队列最大容量
    drop: summarize        # 超出 cap 后的处理
```

---

### 六、三种 Drop Policy（队列满了怎么办）

| dropPolicy | 行为 |
|---|---|
| `old` | 丢弃最旧的消息，保留最新的 |
| `new` | 丢弃新来的消息，保留已有的 |
| `summarize`（默认） | 把被丢弃消息的 `summaryLine` 积累起来，下次执行时告知 Agent "有 N 条消息被丢弃，摘要如下" |

`summarize` 最智能，不丢信息只压缩。

---

### 七、队列排干（drain.ts）流程

Agent 跑完后调用 `scheduleFollowupDrain`：

```
while (queue.items.length > 0 || queue.droppedCount > 0)
    │
    ├── waitForQueueDebounce()    // 等 debounceMs，让消息积累
    │
    ├── mode = "collect"?
    │   ├── 检查是否跨 Channel
    │   │   是 → 逐条处理（保证路由正确）
    │   │   否 → 合并成一条大消息：
    │   │         "[Queued messages while agent was busy]
    │   │          ---
    │   │          Queued #1: 消息内容
    │   │          ---
    │   │          Queued #2: 消息内容"
    │   └── 一次性发给 Agent
    │
    └── 其他 mode → 逐条消费（drainNextQueueItem）
```

---

### 八、去重保障（inbound-dedupe）

同一条消息可能被 Channel 重试投递两次，通过 **messageId + 5 分钟 TTL LRU 缓存**去重：

```typescript
const RECENT_QUEUE_MESSAGE_IDS = resolveGlobalDedupeCache(key, {
  ttlMs: 5 * 60 * 1000,  // 5 分钟内同一 messageId 只处理一次
  maxSize: 10_000,
});
```

---

### 九、完整场景示例

**场景1：Agent 跑到一半，你又连发了 3 条消息（mode=collect）**

```
消息1 → Agent 开始跑（isActive=true）

消息2、3、4 → resolveActiveRunQueueAction
               isActive=true, shouldFollowup=true
               → "enqueue-followup"，依次入队

Agent 跑完
    ↓
scheduleFollowupDrain → waitForQueueDebounce(1s)
    ↓
mode=collect → 合并消息2+3+4为一条发给 Agent
```

**场景2：Agent 跑到一半，你发了 "stop"**

```
"stop" → tryFastAbortFromMessage（最高优先级）
    ↓
abortEmbeddedPiRun()   // 中止 Agent Loop
clearSessionQueues()   // 消息2、3、4全部丢弃
stopSubagentsForRequester()  // 子 Agent 递归停止
    ↓
回复 "⚙️ Agent was aborted."
```

**场景3：心跳消息在 Agent 跑的时候来了**

```
heartbeat systemEvent → resolveActiveRunQueueAction
    isActive=true, isHeartbeat=true
    → "drop"（静默丢弃，不排队，不影响当前执行）
```