---
modified: '2026-03-30T17:38:28.273221'
---

Fetching document 2752711602......
# Session & Routing 系统源码解析


> 研究日期：2026-03-23 | 源码路径：`/app/src/routing/`、`/app/src/sessions/`
>
>


---

## 一、定位

Session / Routing 系统负责回答两个问题：


1. **这条消息应该交给哪个 Agent？**（Routing / Binding 系统）
2. **这个 Agent 应该用哪个 session，是新建还是继续上次的？**（Session Freshness）


---

## 二、SessionKey 格式（源码：`routing/session-key.ts`）

SessionKey 是一个冒号分隔的字符串，格式取决于会话类型：

```Plain
// 代码块
# 默认（DM，dmScope=main，所有 Channel 共享同一个 session）
agent:main:main

# 按 Channel + Peer 区分（dmScope=per-channel-peer）
agent:main:daxiang:direct:1777656213

# 按 Channel + Account + Peer 区分（dmScope=per-account-channel-peer）
agent:main:daxiang:default:direct:1777656213

# 群聊（永远包含 Channel，不受 dmScope 影响）
agent:main:discord:channel:1234567890

# Thread（Discord 帖子 / Telegram 话题）
agent:main:discord:thread:987654321

# Cron 任务
agent:main:cron:daily-report:run:abc123

# Sub-Agent
agent:main:subagent:abc123

```

**核心规律：** `agent:<agentId>:<rest>`，rest 编码了会话的所有维度。  
**全部小写：** `buildAgentPeerSessionKey()` 最后调 `toLowerCase()`，保证不同来源的 key 稳定匹配。


---

## 三、SessionKey 生成流程（五步，源码：`routing/session-key.ts`）

```Plain
// 代码块
Channel Plugin 收到消息
  ↓
1. 提取基础信息
   channel = "daxiang"
   peer    = { kind: "direct", id: "1777656213" }
  ↓
2. resolveAgentRoute()  ← 路由决策（bindings 系统）
   根据 bindings 配置决定用哪个 agentId
   （没有匹配的 binding → 用默认 agentId）
  ↓
3. 读取 dmScope 配置（默认 "main"）
  ↓
4. buildAgentPeerSessionKey()  ← 拼装 key
   dmScope = "main"                    → agent:main:main
   dmScope = "per-peer"                → agent:main:direct:1777656213
   dmScope = "per-channel-peer"        → agent:main:daxiang:direct:1777656213
   dmScope = "per-account-channel-peer"→ agent:main:daxiang:default:direct:1777656213
  ↓
5. toLowerCase()  ← 全小写，保证稳定性

```


---

## 四、Binding 系统（多 Agent 路由，源码：`routing/bindings.ts`、`resolve-route.ts`）

配置了多个 Agent 时，通过 bindings 规则决定消息走哪个 Agent：

```YAML
// 代码块
bindings:
  - agentId: "support-bot"
    match:
      channel: "telegram"
      peer: { kind: "group", id: "12345" }   # 特定群组

  - agentId: "vip-agent"
    match:
      channel: "*"
      accountId: "premium"                   # 特定账号

  - agentId: "main"                          # 默认兜底（无 match）

```

**匹配优先级（从高到低）：**


| 优先级 | 匹配维度 | 说明 |
| --- | --- | --- |
| 1 | peer 精确匹配 | 特定用户/特定群 |
| 2 | thread parent peer 匹配 | Thread 对应的父群 |
| 3 | guild + roles 匹配 | Discord 角色 |
| 4 | guild 匹配 | Discord 服务器 |
| 5 | team 匹配 | Slack workspace |
| 6 | accountId 精确匹配 | 特定账号 |
| 7 | accountId = "*" | Channel 级通配 |
| 8 | default | 无任何匹配，兜底 Agent |

**缓存优化：** Binding 匹配结果用 Map 缓存（key = `channelId:accountId`，最大 2000 条），避免每条消息都重新遍历 bindings 配置。


---

## 五、新建还是继续：Session Freshness 判断（源码：`sessions/send-policy.ts`）

SessionKey 生成后，查 Session Store 里有没有对应记录：

**没有记录** → 新建 session（`isNewSession = true`）

**有记录** → 检查 freshness（保鲜度）：

```Plain
// 代码块
evaluateSessionFreshness(updatedAt, now, policy)
  ↓
policy.mode = "daily"（默认）
  dailyResetAt = 今天凌晨 4:00（atHour=4）
  updatedAt < 凌晨4点 → stale（过期）→ 重置 session
  updatedAt >= 凌晨4点 → fresh（新鲜）→ 继续用

policy.mode = "idle"
  idleExpiresAt = updatedAt + idleMinutes × 60000
  now > idleExpiresAt → stale（空闲超时）→ 重置 session
  now <= idleExpiresAt → fresh → 继续用

```

**为什么默认凌晨 4 点而不是 0 点？**<br>
0 点正好是很多人还在用的时间，凌晨 4 点是绝大多数用户的"深夜"，不容易在对话中途触发重置。

**可按 session 类型分别配置：**

```YAML
// 代码块
session:
  reset:
    mode: "daily"
    atHour: 4
  resetByType:
    direct:
      mode: "idle"
      idleMinutes: 60    # DM 空闲1小时重置
    group:
      mode: "daily"
      atHour: 0          # 群聊每天 0 点重置
    thread:
      mode: "idle"
      idleMinutes: 10080 # Thread 空闲7天重置
  resetByChannel:
    telegram:
      mode: "idle"
      idleMinutes: 30    # Telegram 单独配置

```


---

## 六、Session Store 结构（源码：`sessions/` 目录）

Session Store 是一个 JSON 文件（路径：`~/.openclaw/agents/<agentId>/sessions.json`），以 SessionKey 为键：

```JSON
// 代码块
{
  "agent:main:daxiang:direct:1777656213": {
    "sessionId": "94f561a8-2134-4213-9a45-edacab208cbb",
    "updatedAt": 1742732878000,
    "sessionFile": "/root/.openclaw/agents/taizi/sessions/94f561a8-...jsonl",
    "chatType": "direct",
    "thinkingLevel": "high",
    "modelOverride": "gpt-4o",
    "verboseLevel": "on",
    "sendPolicy": "allow",
    "inputTokens": 12345,
    "outputTokens": 6789
  }
}

```

**SessionEntry 核心字段：**


| 字段 | 说明 |
| --- | --- |
| `sessionId` | 唯一 UUID，定位 transcript（.jsonl）文件 |
| `updatedAt` | 最后活跃时间（freshness 判断依据） |
| `sessionFile` | 对话历史 JSONL 文件完整路径 |
| `modelOverride` | 用户通过 /model 切换的模型，null = 用默认 |
| `thinkingLevel` | Reasoning 强度（low/high/max） |
| `verboseLevel` | 工具调用详细程度（on/off） |
| `inputTokens` | 累计输入 tokens |
| `outputTokens` | 累计输出 tokens |


---

## 七、"继续用"和"新建"的本质区别

SessionKey 本身只是一个字符串 ID，真正的上下文在 `.jsonl` 文件里：


- **继续用**：加载已有 transcript（.jsonl 文件），接续上次对话
- **新建（isNewSession=true）**：不加载任何历史，全新开始
- **重置（resetTriggered=true）**：有历史但已过期，清空（新建新 .jsonl，旧的不删除）

**Transcript 文件格式（每行一个 JSON 事件）：**

```Plain
// 代码块
{"type":"message","message":{"role":"user","content":"你好"},"timestamp":1742732878000}
{"type":"message","message":{"role":"assistant","content":}}
{"type":"tool_use","id":"toolu_xxx","name":"exec","input":{"command":"ls -la"}}
{"type":"tool_result","tool_use_id":"toolu_xxx","content":}

```


---

## 八、identityLinks：跨 Channel 统一身份（源码：`routing/session-key.ts`）

同一个用户在不同 Channel 上有不同 ID，通过 identityLinks 映射到同一个 session：

```YAML
// 代码块
session:
  dmScope: "per-peer"
  identityLinks:
    "canonical-user-id":
      - "telegram:123456"    # Telegram 上的 ID
      - "1777656213"         # 大象上的 ID
      - "discord:987654"     # Discord 上的 ID

```

**实现细节（resolveLinkedPeerId() 函数）：**


1. 遍历所有 identityLinks 条目，查找当前 `channel:peerId` 是否在某个链接组里
2. 如果找到，返回该组的"规范 ID"（第一个条目）
3. 后续所有 Channel 的消息都用同一个规范 ID 构建 SessionKey

配置后，无论从 Telegram 还是大象发消息，都得到同一个 SessionKey，共享对话历史和 Directive 配置。


---

## 九、AgentId 规范化（源码：`routing/session-key.ts normalizeAgentId()`）

```TypeScript
// 代码块
function normalizeAgentId(value: string): string {
  if (VALID_ID_RE.test(trimmed)) return trimmed.toLowerCase();
  // 容错：把非法字符替换为"-"，确保 path-safe + shell-friendly
  return trimmed.toLowerCase()
    .replace(INVALID_CHARS_RE, "-")
    .replace(/^-+/, "").replace(/-+$/, "")
    .slice(0, 64) || DEFAULT_AGENT_ID;
}

```

agentId 必须 path-safe（可用在文件路径里）且 shell-friendly，不能有空格和特殊符号。


---

## 十、设计亮点总结


| 亮点 | 说明 |
| --- | --- |
| dmScope 四档 | 一个配置控制 session 共享粒度，灵活且向后兼容 |
| identityLinks | 跨 Channel 统一身份，不需要重建上下文 |
| Binding 优先级缓存 | 避免每条消息都重新遍历所有 binding 规则 |
| 凌晨 4 点默认重置 | 不打扰活跃用户，选在真正的"深夜"重置 |
| 按类型配置 Freshness | DM/群聊/Thread 各有最合适的生命周期策略 |
| agentId 容错规范化 | 保证文件系统安全，不会因命名导致路径问题 |

Done


---

## 补充：Routing 决策引擎深度解析（2026-03-30）

### 一、Binding 匹配：7层 Tier 瀑布

源码 `resolve-route.ts` 的 `tiers` 数组，按顺序依次尝试，**第一个命中即返回**：

```
Tier 1: binding.peer         → peer 精确匹配（当前消息的 peer）
Tier 2: binding.peer.parent  → Thread 的父 peer（群聊 ID）
Tier 3: binding.guild+roles  → Discord guild + 角色（同时满足）
Tier 4: binding.guild        → Discord guild（不带角色约束）
Tier 5: binding.team         → Slack workspace
Tier 6: binding.account      → accountId 精确匹配
Tier 7: binding.channel      → accountId="*"（Channel 级通配）
```

7层全没命中 → `resolveDefaultAgentId(cfg)`，标记 `matchedBy: "default"`。

**匹配结果缓存**：整个路由结果（含 sessionKey、agentId、matchedBy）缓存进 `resolvedRouteCacheByCfg`（WeakMap，上限 4000 条）。以下两种情况跳过缓存：
- `shouldLogVerbose()` 为 true（debug 模式，需打 log）
- 配置了 `identityLinks`（peer ID 动态映射，不可缓存）

---

### 二、场景举例：不同参数的完整流程

#### 场景一：最简单（dmScope=main，默认）

什么都不配，用默认值：

```
用户 A 在 Telegram 发消息  → agent:main:main
用户 A 在大象发消息        → agent:main:main
用户 B 在 Discord 发消息   → agent:main:main
```

所有人所有 Channel 共享同一个 session。一个 OpenClaw 服务一个主人的典型配置。

---

#### 场景二：多用户隔离（dmScope=per-channel-peer）

```yaml
session:
  dmScope: "per-channel-peer"
```

```
用户 A 在 Telegram 发消息  → agent:main:telegram:direct:111
用户 A 在大象发消息        → agent:main:daxiang:direct:111  ← 不同 session！
用户 B 在 Telegram 发消息  → agent:main:telegram:direct:222
```

三条 session 互相隔离，各自维护独立上下文。

---

#### 场景三：多 Agent 路由（Binding）

```yaml
bindings:
  - agentId: "support-bot"
    match:
      channel: "telegram"
      peer: { kind: "group", id: "12345" }   # Telegram 群 12345

  - agentId: "vip-agent"
    match:
      channel: "*"
      accountId: "premium"                   # 账号 premium 的所有消息
```

```
Telegram 群 12345 来消息  → Tier1 命中 binding.peer → support-bot
                            → sessionKey: agent:support-bot:telegram:group:12345

账号 premium 的 Discord DM → Tier1-5 没命中，Tier6 binding.account 命中 → vip-agent
                             → sessionKey: agent:vip-agent:discord:direct:888

普通 Discord 用户           → 7层全没命中 → matchedBy: "default" → main
                             → sessionKey: agent:main:main
```

---

#### 场景四：Thread 消息的父 peer 继承（Tier2 的作用）

Discord 群 `group:999` 绑定了 `support-bot`，用户在群内的 Thread `thread:777` 里发消息：

```
Thread 消息来了
    ↓
Tier1: peer=thread:777 → 没有 binding 匹配 → 跳过
Tier2: parentPeer=group:999 → 命中 support-bot 的 binding → ✅
```

**结论**：Thread 里发消息，自动继承所在群的 Agent 绑定，无需单独配 binding。

---

#### 场景五：SendPolicy 控制消息入口

**背景**：连了多个 Channel，但只想让某个 Channel 触发 Agent 回复，其他 Channel 只作输出通道。

```yaml
session:
  sendPolicy:
    default: "deny"          # 默认全部拒绝输入
    rules:
      - match:
          channel: "telegram"
        action: "allow"      # 只有 Telegram 的消息能触发回复
```

Discord/大象 的消息在 `resolveSendPolicy()` 这一步被 deny，不进入 Agent 处理流程。
**注意**：SendPolicy 只控制"入"（消息触发 Agent），不控制"出"（Agent 主动发消息）。

**规则匹配逻辑（short-circuit）**：
1. Session entry 上有 `sendPolicy` override → 直接用
2. 遍历 `policy.rules`，遇到 `action: "deny"` 直接返回 deny（优先于 allow）
3. 有 allow 规则命中 → allow
4. 都没命中 → 用 `policy.default`（默认 allow）

---

### 三、lastRoutePolicy：解决"回复发到哪"的问题

当 OpenClaw 需要主动发消息（如 Cron 触发回复），要知道"发到哪个 Channel/账号"，靠查"上次路由记录"实现：

- **`"main"`**：`sessionKey === mainSessionKey`（dmScope=main）时，路由记录存在 `mainSessionKey` 上，统一查 main。
- **`"session"`**：`sessionKey !== mainSessionKey`（per-channel-peer 等细粒度）时，路由记录存在各自的 `sessionKey` 上。

```typescript
// 源码：resolve-route.ts
export function deriveLastRoutePolicy(params: {
  sessionKey: string;
  mainSessionKey: string;
}): "main" | "session" {
  return params.sessionKey === params.mainSessionKey ? "main" : "session";
}
```

---

### 四、InputProvenance：消息来源标记

每条进入 Agent 的 user message 可携带 `provenance` 字段（`sessions/input-provenance.ts`）：

| kind | 含义 | 典型场景 |
|---|---|---|
| `external_user` | 真实用户从 IM 发来的 | Telegram/Discord DM |
| `inter_session` | 跨 session 转发 | Sub-Agent 完成后把结果推给主 agent |
| `internal_system` | 系统内部注入 | Cron / Heartbeat 触发的 agentTurn |

`hasInterSessionUserProvenance()` 判断消息是否来自跨 session 调用，避免对这类消息重复触发 `message:received` Hook。

---

### 五、Session 生命周期事件总线

`sessions/session-lifecycle-events.ts` 提供全局事件总线，通知 session 的生命周期变更：

```typescript
// 订阅
onSessionLifecycleEvent((event) => {
  // event.sessionKey, event.reason, event.label
});

// 触发（新建 session、reset、resume 时）
emitSessionLifecycleEvent({
  sessionKey: "agent:main:daxiang:direct:123",
  reason: "new_session",
});
```

常见 reason：`new_session`、`session_reset`、`session_resumed`。
消费方：WebUI 实时刷新 session 列表、Hooks 系统触发 `session` 类型事件。

---

### 六、完整消息路由全流程

```
Channel Plugin 收到 IM 消息
    ↓
resolveAgentRoute()               ← 核心路由决策
    ├── 7层 Binding Tier 匹配 → agentId
    ├── buildAgentSessionKey()    → sessionKey（含 dmScope + identityLinks）
    └── deriveLastRoutePolicy()   → lastRoutePolicy("main" | "session")
    ↓
resolveSendPolicy()               ← 是否允许触发 Agent？
    └── deny → 直接丢弃
    ↓
Session Store 查询 (sessions.json)
    ├── 没有记录 → isNewSession=true
    └── 有记录 → evaluateSessionFreshness()
                 ├── fresh → 继续 (sessionId 不变，加载历史 .jsonl)
                 └── stale → resetTriggered=true (新建 .jsonl，旧的保留)
    ↓
applyInputProvenanceToUserMessage() ← 标注消息来源
    ↓
dispatchReplyFromConfig()          ← 进入 Auto-Reply 流程
```