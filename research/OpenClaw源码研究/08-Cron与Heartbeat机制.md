---
modified: '2026-03-30T11:33:49.091084'
---

Fetching document 2752633573......
# OpenClaw 源码精读：Cron 定时任务 & Heartbeat 机制


> 源码路径：`/app/src/cron/` 和 `/app/src/infra/heartbeat-*.ts`
>
>


---


> ⚠️ **注意**：Cron 和 Heartbeat 是两个独立但深度耦合的系统，需要放在一起理解。
>
>


---

## 一、Cron 系统

### 1.1 CronJob 完整数据结构

```TypeScript
// 代码块
type CronJob = {
  id: string;
  agentId?: string;
  sessionKey?: string;         // 任务归属的 session（用于消息推送路由）
  name: string;
  description?: string;
  enabled: boolean;
  deleteAfterRun?: boolean;    // 一次性任务：跑完自动删除
  createdAtMs: number;
  updatedAtMs: number;
  schedule: CronSchedule;
  sessionTarget: CronSessionTarget;  // "main" | "isolated"
  wakeMode: CronWakeMode;            // "next-heartbeat" | "now"
  payload: CronPayload;
  delivery?: CronDelivery;
  state: CronJobState;
};

```

### 1.2 三种调度模式（schedule）

```TypeScript
// 代码块
type CronSchedule =
  | { kind: "at"; at: string }                  // 一次性：ISO 时间字符串
  | { kind: "every"; everyMs: number; anchorMs?: number }  // 固定间隔
  | { kind: "cron"; expr: string; tz?: string; staggerMs?: number };  // cron 表达式

```

`**staggerMs（打散模式）**`：这是一个常被忽略的细节。当多个任务配置在整点执行时，OpenClaw 会在 `staggerMs` 窗口内自动打散执行时间，避免同时爆发（流量削峰）。

`**every 模式的锚点机制**`：

```Plain
// 代码块
anchorMs = 某个绝对时间戳（默认 = 首次注册时间）
elapsed = nowMs - anchorMs
steps = ceil(elapsed / everyMs)
nextRun = anchorMs + steps * everyMs

```

这意味着即使 Gateway 重启，下次执行时间也能精确对齐到锚点，不会"漂移"。

### 1.3 两种 SessionTarget（最关键区别）

```TypeScript
// 代码块
type CronSessionTarget = "main" | "isolated";
type CronWakeMode = "next-heartbeat" | "now";

```


| `main` + `next-heartbeat` | `isolated` + `now` |
| --- | --- |
| 执行方式 | 写入主 session 系统事件队列，**等下次 heartbeat 唤醒** | 创建独立 session，**立即执行** |
| 上下文 | **共享主 session 完整历史** | 完全隔离（空白或指定 session） |
| 等待时间 | 取决于 heartbeat 间隔（默认30分钟） | 近实时 |
| 适合场景 | "帮我总结今天所有对话"（需要上下文） | 定时报告、健康检查（纯自动化） |

**为什么 main 模式需要等 heartbeat？**

因为 main session 不能被"强行插队"——用户可能正在对话，贸然注入系统事件会打断当前流程。设计上让 heartbeat 作为"安全窗口"：heartbeat 触发时如果没有用户活动，才执行队列里的 cron 事件。

### 1.4 两种 Payload

```TypeScript
// 代码块
type CronPayload =
  | { kind: "systemEvent"; text: string }
  | {
      kind: "agentTurn";
      message: string;
      model?: string;         // 可以用不同 model 执行定时任务
      thinking?: string;
      timeoutSeconds?: number;
      deliver?: boolean;      // 执行完是否推送结果
      channel?: CronMessageChannel;
      to?: string;
      bestEffortDeliver?: boolean;
    };

```


- `systemEvent`：系统事件注入，agent 感知到有事发生（用于 main 模式）
- `agentTurn`：完整 agent 执行，独立跑一次完整的 prompt（用于 isolated 模式）

### 1.5 Delivery 配置

```TypeScript
// 代码块
type CronDelivery = {
  mode: "none" | "announce" | "webhook";
  channel?: ChannelId | "last";   // "last" = 发到最近活跃的渠道
  to?: string;                    // 具体接收人
  accountId?: string;
  bestEffort?: boolean;           // 发送失败不影响任务状态
};

```

### 1.6 CronJobState：状态追踪

```TypeScript
// 代码块
type CronJobState = {
  nextRunAtMs?: number;
  runningAtMs?: number;
  lastRunAtMs?: number;
  lastRunStatus?: "ok" | "error" | "skipped";
  lastError?: string;
  lastDurationMs?: number;
  consecutiveErrors?: number;    // 连续错误次数，用于退避计算
  scheduleErrorCount?: number;   // cron 表达式解析失败次数（超阈值自动禁用）
  lastDeliveryStatus?: "delivered" | "not-delivered" | "unknown" | "not-requested";
  lastDelivered?: boolean;
};

```

### 1.7 可靠性设计

**错误退避**（`consecutiveErrors` 驱动）：

```Plain
// 代码块
consecutiveErrors=1  → 30s 后重试
consecutiveErrors=2  → 1min
consecutiveErrors=3  → 5min
consecutiveErrors=4  → 15min
consecutiveErrors=5+ → 60min

```

**重启补偿**（`service.restart-catchup.test.ts` 覆盖此逻辑）：


- Gateway 重启时，检查所有任务的 `nextRunAtMs`
- 如果 `nextRunAtMs < now`（即任务在宕机期间"错过"了），立即补跑
- 但补跑有一个最大追溯窗口，不会无限往回追


---

## 二、Heartbeat 系统

### 2.1 (http://HEARTBEAT.md "HEARTBEAT.md") 的真实作用（常见误解）

**❌ 误解**：(http://HEARTBEAT.md) 是在 agent 启动时加载到上下文里的背景文件，类似 (http://SOUL.md)。

**✅ 正确理解**：(http://HEARTBEAT.md) **完全不是**上下文文件。

它是每次 heartbeat 触发时的**任务清单**——heartbeat 触发 → 读取 (http://HEARTBEAT.md "HEARTBEAT.md") 内容 → 把内容**作为消息体**发给 agent → agent 自行判断执行。

```Plain
// 代码块
heartbeat 触发（interval / cron 事件 / exec 完成 / ...）
    ↓
heartbeat-runner.ts 读取 HEARTBEAT.md
    ↓
appendCronStyleCurrentTimeLine() 追加当前时间（给 agent 定位时间用）
    ↓
把内容发给 agent（Provider 标记为 "heartbeat"）
    ↓
Agent 处理清单：
    - 有任务 → 执行 → 返回结果 → 推送到大象
    - 无事可做 → 回复 HEARTBEAT_OK → OpenClaw 静默处理（不推送）

```

### 2.2 七种触发原因（源码实际定义）

```TypeScript
// 代码块
type HeartbeatReasonKind =
  | "interval"     // 定时器到期（默认30分钟，可配置）
  | "cron"         // cron 任务完成，需要 agent 处理（reason="cron:<jobId>"）
  | "exec-event"   // exec/bash 脚本退出（reason="exec-event"）
  | "wake"         // 显式唤醒请求（reason="wake"）
  | "hook"         // Plugin/Internal Hook 主动唤醒（reason="hook:<jobId>"）
  | "manual"       // 用户手动触发（/heartbeat 命令）
  | "retry"        // 上次 heartbeat 失败后重试
  | "other";       // 其他原因

```

**事件驱动的判断**（`isHeartbeatEventDrivenReason`）：

```TypeScript
// 代码块
// exec-event / cron / wake / hook → 属于"事件驱动"唤醒
// interval / manual / retry → 属于"定时/人工"唤醒
// 两者在处理优先级和 prompt 构建方式上有所不同

```

### 2.3 Active Hours：避免半夜被打扰

Heartbeat 支持 Active Hours 配置：


- 配置文件中设置 `heartbeat.activeHours`（例如 "9:00-22:00"）
- `heartbeat-active-hours.ts` 的 `isWithinActiveHours()` 在每次触发时检查
- 非活跃时间的 interval 触发会被跳过（但事件驱动的仍然执行）

### 2.4 HEARTBEAT_OK 的处理

当 agent 回复 `HEARTBEAT_OK` 时：

```TypeScript
// 代码块
// heartbeat.ts
export function stripHeartbeatToken(text: string): string {
  // 把 HEARTBEAT_OK 从回复中剥离
}
export function isHeartbeatContentEffectivelyEmpty(text: string): boolean {
  // 判断除了 HEARTBEAT_OK 还有没有实质内容
}

```

如果回复中只有 `HEARTBEAT_OK`（无其他内容），OpenClaw 静默处理，不推送到大象。  
如果回复中有 `HEARTBEAT_OK` 但还有其他内容（少见），剥离 token 后正常推送。

### 2.5 HeartbeatSummary：状态查询

```TypeScript
// 代码块
type HeartbeatSummary = {
  enabled: boolean;
  every: string;           // 人类可读的间隔（如 "30m"）
  everyMs: number | null;
  prompt: string;          // 当前 HEARTBEAT.md 内容
  target: string;          // 推送到哪里
  model?: string;          // 指定 model（可选）
  ackMaxChars: number;     // HEARTBEAT_OK 最大长度（防止 agent 回复太长）
};

```

`ackMaxChars`：这个限制是防止 agent 在 HEARTBEAT_OK 后附加大量内容（比如有些 agent 会写完整的日志），超过限制视为无效 ack，按正常消息处理。


---

## 三、Cron 与 Heartbeat 的协作关系

```Plain
// 代码块
Cron 任务调度
    ↓
到期 → 根据 wakeMode 选择：
    ├── "now"（isolated）→ 直接创建独立 session 执行，执行完推送结果
    └── "next-heartbeat"（main）→ 写入 systemEvent 队列
              ↓
         下次 heartbeat 触发
              ↓
         heartbeat-runner 检查队列
              ↓
         有 systemEvent → 构建 prompt（含 cron 事件描述）→ 发给 agent
         agent 处理 → 回复 → 根据 delivery 配置推送

```

**为什么这样设计**：Cron 和 Heartbeat 解耦，各司其职：


- Cron 只管"什么时候该做什么"（调度 + payload）
- Heartbeat 管"在不打扰用户的情况下把任务交给 agent"（安全窗口 + 推送）


---

## 四、实用配置示例

**每天 9:00 检查 Raptor 异常（isolated 模式，立即执行，发结果到大象）**：

```JSON
// 代码块
{
  "id": "raptor-check",
  "schedule": { "kind": "cron", "expr": "0 9 * * *", "tz": "Asia/Shanghai" },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "检查 Raptor 异常指标，生成今日简报"
  },
  "delivery": {
    "mode": "announce",
    "channel": "daxiang",
    "to": "1777656213"
  }
}

```

**每30分钟检查邮件，有事通知我（main 模式，共享上下文）**：

```Plain
// 代码块
# HEARTBEAT.md
检查是否有未处理的紧急邮件或消息，有则汇报，无则 HEARTBEAT_OK

```

Done


---

## 补充：上下文管理、两种机制对比与选择策略（2026-03-30）

### 一、sessionTarget 详解：上下文如何管理

`sessionTarget` 是控制定时任务"是否有连续上下文"的核心字段：

| sessionTarget | 上下文行为 | 典型用途 |
|---|---|---|
| `main` | 注入到主 Agent 的对话流，**共享主会话上下文** | 系统事件通知 |
| `isolated` | **每次执行强制创建新 session**，无历史 | 独立定时任务 |
| `current` | 绑定到触发时的当前会话（规范化后 = `session:xxx`） | 用户触发任务 |
| `session:<key>` | 绑定到指定 sessionKey，**跨次复用同一上下文** | 长期持续对话 |

**关键代码**（`src/cron/isolated-agent/run.ts`）：
```typescript
const cronSession = resolveCronSession({
  cfg: params.cfg,
  sessionKey: agentSessionKey,
  agentId,
  nowMs: now,
  // isolated 模式：强制每次创建新 session
  forceNew: params.job.sessionTarget === "isolated",
});
```

**如果想让定时任务有连续上下文**，用 `session:<key>`：
```json
{
  "schedule": { "kind": "cron", "expr": "0 9 * * *" },
  "sessionTarget": "session:my-daily-report",
  "payload": { "kind": "agentTurn", "message": "继续追踪昨天的待办事项" }
}
```

跨天 session 重置由 `session.reset` 配置控制，默认每天凌晨 4 点重置（`mode: "daily"`, `atHour: 4`）。

---

### 二、Cron Job 与 Heartbeat 是两套独立机制

**常见误区**：以为 HEARTBEAT.md 里可以写精确调度的任务，由 Heartbeat 检测后执行。

**真相**：

```
CronService（定时器引擎）
├── 每分钟的 Cron Job（isolated）
│   CronService 定时器 → 到期执行 → 启动独立 Agent → 完成
│   ↑ 完全不经过 Heartbeat
│
└── 每 30 分钟的 Heartbeat（内置 Cron Job）
    CronService 定时器 → 到期执行 → 注入主 Agent → 读 HEARTBEAT.md
    ↑ 这是另一个独立的 job，sessionTarget=main
```

两个 job 互相不干扰，各自按自己的时间表执行。

**Heartbeat 的本质**：它不是一个独立 Agent，而是利用 CronService 的 `sessionTarget=main` 路径，把一条"探测指令"注入到主 Agent 里，让主 Agent 自己去读 HEARTBEAT.md 判断有没有事情要处理。

**Heartbeat 执行流程**：
1. 每 30 分钟触发（可配置 `agents.defaults.heartbeat.every`）
2. 主 Agent 收到 systemEvent，读取 `HEARTBEAT.md`
3. 文件空/只有注释 → 直接跳过（不调用 LLM API，优化性能）
4. 有内容 → Agent 判断是否有待执行的任务
5. 回复 `HEARTBEAT_OK`（+ ≤300字符说明）→ 静默不投递
6. 有实际任务 → 执行并投递结果

---

### 三、选择策略：什么时候用 Cron Job，什么时候用 Heartbeat

**口诀**：
> **时间精确 + 动作固定 → Cron Job**
> **持续监控 + 条件判断 → Heartbeat.md**

#### 选 Cron Job 的场景
- 时间要求精确（每天9点、每小时、每5分钟）
- 执行动作明确且固定，不需要 Agent 自己判断"要不要执行"
- 需要投递结果到消息渠道
- 需要独立上下文（`isolated`）或持续上下文（`session:xxx`）

#### 选 Heartbeat（HEARTBEAT.md）的场景
- 任务是"持续监控"性质（关注某个状态，有变化就告警）
- 任务逻辑需要 Agent 自己判断（条件复杂，不是"到时间就执行"）
- 任务会动态变化，不想每次都修改 Cron Job 配置
- 可以接受 30 分钟的延迟精度

#### 对比表

| 需求 | 选择 | 原因 |
|---|---|---|
| 每天9点发日报 | Cron Job | 时间固定，动作固定 |
| 监控 GitHub PR，有新的就通知 | Heartbeat | 持续监控，Agent 自己判断 |
| 每小时备份数据库 | Cron Job | 时间固定，动作固定 |
| 检测内存告警（超80%通知） | Heartbeat | 条件判断，不是固定时间触发 |
| 每周一发周报 | Cron Job | `0 9 * * 1`，时间固定 |
| 跟踪待办清单，有完成的就总结 | Heartbeat | 任务动态，条件判断 |
| 每分钟轮询消息队列 | Cron Job | 精度要求高，不能用30分钟粒度 |

#### 两者结合使用

```markdown
# HEARTBEAT.md（灵活任务清单）
- 如果今天是周一，生成上周工作总结
- 检查待处理的 GitHub PR，有过期的提醒我
- 监控系统状态，异常时告警
```

配合一个精确触发的 Cron Job：
```json
{
  "schedule": { "kind": "cron", "expr": "0 8 * * *" },
  "sessionTarget": "main",
  "wakeMode": "now",
  "payload": {
    "kind": "systemEvent",
    "text": "早上8点检查：读取 HEARTBEAT.md 执行今日任务"
  }
}
```

这样既有精确的触发时间，又有灵活可编辑的任务内容。