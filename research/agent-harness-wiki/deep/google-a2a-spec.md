---
type: deep
name: google-a2a-spec
status: doc-only(spec v1.0 + 4 wiki harness/eval verified integrations)
harness: agentscope  # 最深 A2A 集成 verified
mechanism: A2A Protocol — Agent 层协议的 OpenAPI 血脉与跨组织协作设计
updated: 2026-05-18
sources:
  - "[[../concept/agent-protocols]]"
  - "[[../harness/agentscope]]"   # a2a/ 5 文件 + Nacos resolver + formatter
  - "[[../harness/crewai]]"       # a2a/ 子目录
  - "[[../concept/multi-agent-isolation]]"
  - "[[../concept/agent-team-patterns]]"
  - "https://a2a-protocol.org/latest/"
  - "https://a2a-protocol.org/latest/specification/"
  - "https://a2a-protocol.org/latest/topics/agent-discovery/"
  - "https://a2a-protocol.org/latest/topics/streaming-and-async/"
---

# A2A Protocol — Agent 层协议的 OpenAPI 血脉与跨组织协作设计

> **要点**:A2A 解决 _"两个互不信任的 agent 跨组织协作"_ 问题 — Google 出品 → Linux Foundation v1.0 2025 发布。**Agent Card 类比 OpenAPI schema 做发现**,**Task 8 状态机假设长任务 + 人介入是常态**,**Opaque Collaboration 协议层强制 IP 保护**。**ACP 已 merge 进 A2A**,Linux Foundation 扮演 _"协议聚合 hub"_。

---

## 1. 为什么这个协议重要(也容易被低估)

A2A 表面看像 _"另一个 RPC 协议"_,**实际上是第一个协议级假设 _"agent 之间互不信任"_ 的标准**。理解 A2A 要先理解它跟 MCP 的根本错位。

**3 个容易被低估的点**:

1. **A2A 不是 MCP 的 _"agent 版"_,是完全不同问题域** — MCP 解决 _"LLM 借工具"_(host 控权 + server 受限),A2A 解决 _"Agent 委托 Agent"_(双方对等怀疑 + IP 保护)。两者**正交不重叠**,真实 system 经常同时跑两个。
2. **Task 状态机 + INPUT_REQUIRED 是协议层 _"假设人介入是常态"_** — 跟 MCP `tools/call` _"call → return"_ 一发一收完全不同。A2A 假设任务从分钟到天级,中途需要人输入是 normal,**不是异常**。
3. **ACP 合并进 A2A under Linux Foundation 是 standardization 罕见健康姿态** — 跟 W3C / OpenStack 早期 _"各推一份"_ 碎片化对比鲜明。LF 在这里扮演 _"协议聚合 hub"_,**避免协议碎片化** 是 A2A 生态起步阶段的关键资产。

---

## 2. 机制 — A2A 怎么工作

### 2.1 跟 MCP 的根本错位(mental model)

| 维度 | MCP | A2A |
|---|---|---|
| 调用方向 | Host(LLM)→ Server(tool)— 借用 | Agent A → Agent B — 委托 |
| 信任假设 | **Host 信任 Server,server 受限看不到完整对话** | **双方互不完全信任,IP 保护** |
| 核心抽象 | Tool / Resource / Prompt(对 LLM) | **Task / Message / Artifact**(对 agent) |
| 1 个连接关系 | Client : Server = 1 : 1(host 内多 client)| 远端可能多个 Agent Card |
| 长任务 | 不是主流场景(`tools/call` 同步) | **first-class**(Task 8 状态机) |
| 设计灵感 | **LSP**(Language Server Protocol)| **OpenAPI**(schema-first discovery) |
| 反向交互 | Sampling / Elicitation / Roots 三件套 | **INPUT_REQUIRED + AUTH_REQUIRED** Task state |

**核心 mental model**:
- **MCP = _"借用一个工具"_**(borrow-tool)
- **A2A = _"委托一个任务"_**(delegate-task)

### 2.2 4 件套抽象

```
Agent Card    ← Agent 的"名片",JSON 文档,公开 capability/skill/auth
   │
   ▼
Task          ← 1 个 Work Unit,有 8 状态机 + 历史 + artifacts
   │
   ├─ Message ← 1 次交互轮次(role + parts + metadata)
   │     │
   │     └─ Part  ← Message 内的内容块(text / file / data 三种)
   │
   └─ Artifact   ← Task 产出的结构化结果
```

### 2.3 Agent Card — 协议入口

通过 `.well-known/agent-card.json`(**RFC 8615 标准** well-known URI)暴露。

```jsonc
{
  "name": "InvoiceAnalyzerAgent",
  "description": "Analyzes invoices and extracts line items",
  "url": "https://api.example.com/a2a",
  "version": "1.2.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "extendedAgentCard": false       // ⭐ 鉴权后可获取详细 card
  },
  "skills": [
    {
      "name": "analyze_invoice",
      "description": "...",
      "examples": [...],
      "supportedContentTypes": ["application/pdf", "image/*"]
    }
  ],
  "securitySchemes": { "oauth2": {...} }
}
```

**3 种 discovery 路线**:

| 路线 | URL 形态 | 适用场景 |
|---|---|---|
| **Well-Known URI**(RFC 8615) | `https://{domain}/.well-known/agent-card.json` | 公开 agent,自动化发现 |
| **Curated Registry**(spec 没规定 API) | registry 中心查询 | 企业内部 / 公开市场 |
| **Direct Configuration** | 客户端硬编码 | 紧耦合系统 |

**两层 card 设计 elegant 点**:
- 公开 Agent Card 只露 skill name / description
- **敏感 capability** 走 `extendedAgentCard`,鉴权后才能获取详细 schema/auth
- 类比 OAuth 的 well-known endpoint vs token-required endpoint

**Auth during discovery**:mutual TLS / IP 限制 / OAuth 2.0 都支持,spec 推荐 _"authenticated extended agent cards"_ 而非在 public card 嵌 static secret。

### 2.4 Task — 核心工作单元(8 状态机)

```
        ┌──────────┐
START → │SUBMITTED │
        └────┬─────┘
             ▼
        ┌──────────┐
        │ WORKING  │←──────────┐
        └────┬─────┘           │
             │                 │
   ┌─────────┼─────────┐       │
   ▼         ▼         ▼       │
┌────────┐┌──────────────┐┌────┴──────────┐
│COMPLETED││INPUT_REQUIRED││AUTH_REQUIRED  │
│ (term) ││ (active)     ││ (active)      │
└────────┘└──────┬───────┘└───────┬───────┘
                 │                │
                 └──── 等用户响应 ─┘
                 ▼
            (回 WORKING)

终态 4 个:COMPLETED / FAILED / CANCELED / REJECTED
活态 4 个:SUBMITTED / WORKING / INPUT_REQUIRED / AUTH_REQUIRED
```

**Task 数据结构**:
```jsonc
{
  "id": "task_abc123",              // server-generated
  "contextId": "ctx_xyz",           // 多 task 共享上下文(类似 OpenAI thread)
  "status": {
    "state": "WORKING",
    "message": {...},                // 最近一条消息
    "timestamp": "..."
  },
  "artifacts": [...],                // task 产出物
  "history": [Message, Message, ...],// 完整交互历史
  "metadata": {...}
}
```

**关键设计**:`contextId` 让多个相关 task 共享对话上下文 — 类似 OpenAI thread 概念,但 **task-level 而非 message-level**。

### 2.5 Message + Part

```jsonc
{
  "messageId": "msg_001",
  "role": "user" | "agent",
  "contextId": "ctx_xyz",
  "taskId": "task_abc123",
  "parts": [
    { "type": "text", "text": "Analyze this invoice" },
    { "type": "file", "url": "https://.../invoice.pdf", "mediaType": "application/pdf" },
    { "type": "data", "data": { "company": "ACME" } }
  ],
  "referenceTaskIds": ["task_prev"],  // 上下文链接
  "extensions": ["https://x.com/ext1"]
}
```

**Part 4 种 type**(exactly one per Part):
- `text` — 纯文本
- `raw` — base64 编码 bytes
- `url` — 文件引用(惰性下载)
- `data` — 结构化 JSON(parametric)

### 2.6 JSON-RPC 方法(8 个核心)

| 方法 | 作用 | 用法 |
|---|---|---|
| `message/send` | 发消息(同步 / 启动 task)| 同步路线 |
| `message/stream` | 发消息 + SSE streaming | **streaming 路线** |
| `tasks/get` | 查 task 状态 | 轮询 fallback |
| `tasks/list` | 列 task(cursor 分页) | 列表 / 历史 |
| `tasks/cancel` | 取消 task(幂等) | 控制 |
| `tasks/resubscribe` | 重订阅断开的 stream | recovery |
| `tasks/pushNotificationConfig/set` 等 4 个 | 注册 webhook | 异步路线 |

### 2.7 3 种 invocation 模式

```
模式 A:同步 / 短期
  client → message/send → server
  server → 返回 Task 或 Message(终态)
  适合:简单查询、确定性短任务

模式 B:Streaming SSE
  client → message/stream → server
  server ← SSE 流 ← Task(初始)+ TaskStatusUpdateEvent* + TaskArtifactUpdateEvent*
  连接保持到 task 终态
  适合:长任务 + 实时进度 / 流式 token

模式 C:Push Webhook
  client → tasks/pushNotificationConfig/set(webhook URL)
  client → message/send(返回 task ID 后立即断开)
  server → POST 到 webhook(达到关键状态时)
  client webhook → tasks/get 拉完整状态
  适合:mobile / serverless / 跨小时-跨天任务
```

**模式 B SSE 流 3 种事件**:
- `Task` — 首事件,**spec MUST**:订阅时第一个事件必须是当前 task 状态(防订阅间隙信息丢失)
- `TaskStatusUpdateEvent` — 状态变化
- `TaskArtifactUpdateEvent` — artifact 增量(用 `append` + `lastChunk` 拼)

**模式 C webhook 流程**:client 提供 `PushNotificationConfig { url }` → server 关键状态变化 POST 到 webhook → client webhook 拿到通知后**主动调 `tasks/get`** 拉完整状态。

### 2.8 完整 invoice 调用 walkthrough

```
[发现]
client → GET https://acme.com/.well-known/agent-card.json
       ← Agent Card { skills: [analyze_invoice],
                      capabilities: { streaming: true } }

[启动 + streaming]
client → POST /a2a {
  jsonrpc: "2.0",
  method: "message/stream",
  params: {
    message: {
      role: "user",
      parts: [
        { type: "text", text: "Analyze this invoice and extract line items" },
        { type: "file", url: "https://.../invoice.pdf", mediaType: "application/pdf" }
      ]
    }
  }
}

[SSE 流回]
← event: Task { id: "task_001", status: { state: "SUBMITTED" } }
← event: TaskStatusUpdateEvent { state: "WORKING", message: "Reading PDF..." }
← event: TaskArtifactUpdateEvent { artifact: {...partial}, append: true }
← event: TaskArtifactUpdateEvent { artifact: {...more}, lastChunk: true }
← event: TaskStatusUpdateEvent { state: "INPUT_REQUIRED",
            message: "Which currency to convert to?" }

[client 提供输入]
client → POST /a2a {
  method: "message/send",
  params: {
    message: { taskId: "task_001", parts: [{ type: "text", text: "USD" }] }
  }
}

[流继续]
← event: TaskStatusUpdateEvent { state: "WORKING" }
← event: TaskStatusUpdateEvent { state: "COMPLETED",
            message: "Done. 24 line items extracted, total $4,283.50 USD." }
(stream closes)
```

---

## 3. WHY — 为什么这么设计

### 3.1 Opaque Collaboration(协议层 IP 保护)

spec 明示:_"Agents collaborate based on declared capabilities and exchanged information, without needing to share internal thoughts, plans, or tool implementations."_

**这是 A2A 跟 multi-agent framework 内部协作最大的区别**:

| 形态 | 是否共享 prompt/tool/memory? | 信任模型 |
|---|---|---|
| CrewAI / AG2 / MetaGPT 内部 multi-agent | **共享**(同信任域) | 同一公司 / 同一用户 |
| A2A 跨 agent | **不共享**(只交换 Task + Message + Artifact)| **跨信任域,跨组织** |

**为什么协议层强制 IP 保护**:A2A 假设 _"我接你公司的 agent SaaS"_ 是常态 — 对方不可能把 prompt 给我看(那是核心 IP),我也不希望我的 agent 内部决策被对方看。**协议层强制 = 标准化双方期望,降低集成谈判成本**。

跟 [[../concept/multi-agent-isolation]] 5 维 trade-off 直接对应:**A2A 协议层站完全隔离一边**。

### 3.2 Long-Running by Default

Task 状态机的 INPUT_REQUIRED + AUTH_REQUIRED + 重订阅 + webhook 推送 —— **协议假设 task 跑分钟到天级**,不假设秒级响应。

**对照**:
- REST API 思路:client 等 response,timeout 30 秒
- MCP `tools/call` 思路:同步阻塞,长任务靠 Streamable HTTP SSE 是 _"权宜之计"_
- A2A 思路:**task lifecycle 内置,client 可以断开几小时再 `tasks/get` 接着拿状态**

**为什么 long-running first-class**:跨组织 agent 协作的真实任务大多 long-running(_"分析这个法律案件 → 24 小时"_ / _"训练模型 → 3 天"_ / _"等待人审批 → 不可预测"_)。**协议层假设 long-running 是 _"承认现实"_**。

### 3.3 Human-in-the-Loop First-Class

`INPUT_REQUIRED` 是协议级状态,不是后置约束。这是 3 个独立项目同时选择的设计:
- **A2A `INPUT_REQUIRED`** Task state
- [[../harness/codex]] `request_user_input` 协议事件
- AG-UI `human-in-the-loop interrupts` event(UI 层)

**协议设计共识**:_"agent 经常需要人介入,把它做成 protocol-level 而非后置约束"_。

### 3.4 Agent Card 借 OpenAPI 思路

跟 OpenAPI schema 一脉相承:
- 都是 _"服务能力的机器可读描述"_(`name + description + parameters + auth`)
- 都通过 well-known URL 暴露(`.well-known/openapi.yaml` vs `.well-known/agent-card.json`)
- 都支持 capability 协商
- **借力 web 开发者已有 mental model** — 不需要学新概念

### 3.5 Linux Foundation 做 _"协议聚合 hub"_

A2A 起源 Google,**v1.0 后转 Linux Foundation 治理**,并把 IBM ACP 合并进来。

**这是协议政治学健康姿态**:
- 单厂商主导 → 标准锁定,生态不敢接(W3C Apple/Google 互掐 Web Push 案例)
- **中立基金会治理 + 合并对手协议** → 行业 buy-in 概率高
- ACP 合并 = 减少协议碎片化,客户不用 _"选 A2A 还是 ACP"_,只有 A2A 一份标准

---

## 4. 失败模式 — 这个协议防了哪些 bug

### 4.1 跨组织 IP 泄漏

**问题**:Agent A(我家)调 Agent B(第三方 SaaS),如果协议设计要求暴露 internal prompt / tool list,**我家 IP 就漏给对方**。

**A2A 防御**:**Opaque Collaboration** 协议级约束 — 双方只能交换 Task / Message / Artifact,**内部 prompt / tool / memory 不通过协议传输**。即使 implementor 想偷偷传,协议也没字段给它放。

### 4.2 长任务断线丢状态

**问题**:Task 跑 6 小时,client 网络断开 30 分钟,重连后无法续接。

**A2A 防御**:
- `tasks/resubscribe` 方法 — 客户端断线后调这个**重新订阅同一 task 的更新流**
- **spec MUST**:resubscribe 时第一个事件必须是当前 Task 状态,防订阅间隙信息丢失
- Webhook push 模式 — 完全不要求 client 保持连接,server 主动推
- `contextId` 跨 task 持续 — 即使 client 重启,**可以用同 contextId 启新 task** 接续上下文

### 4.3 恶意 Agent 注册攻击

**问题**:攻击者注册一个 `.well-known/agent-card.json`,声称自己是 _"GitHub 官方"_ agent,骗 client 把 token 给它。

**A2A 防御**:
- spec 推荐 **mutual TLS** / IP 限制 / **OAuth 2.0** 在 discovery 阶段就介入
- `extendedAgentCard` 两层 card 设计 — public card 只露 skill name,**详细 schema 需要鉴权后获取**
- spec 推荐:_"authenticated extended agent cards"_ 而非在 public card 嵌 secret

### 4.4 Replay / Task ID 伪造

**问题**:攻击者拿到一个 task ID,伪造 `tasks/cancel` 中断别人的任务。

**A2A 防御**:协议层未明示(留 implementor),但**OAuth 2.0 + mutual TLS** 提供 transport 层防御 — task ID 在加密 channel 内传输,且每个 client 只有自己的 task ID 列表。

### 4.5 同步思维错误用 A2A

**问题**:工程师以为 A2A 是 _"agent 版 REST"_,只用 `message/send` 同步模式,**完全浪费 Task 8 状态机 + INPUT_REQUIRED + streaming + webhook 4 个杀器**。

**A2A 防御 / 教育**:spec 把 3 invocation 模式并列文档(`topics/streaming-and-async/`),明示 long-running 是主流场景。**但工程师如果不读 spec,容易掉进这个坑** —— 这是 [[../concept/agent-protocols]] §10.5 _"A2A 落地两条路线"_ 里说的 _"路线 A agent-as-tool 降级"_。

---

## 5. 横向校准 — 其他协议怎么处理同问题

### 5.1 vs MCP(同协议栈不同层,详见 [[../concept/agent-protocols]])

详见 §2.1 mental model 错位。**MCP 借用工具 / A2A 委托任务,正交不重叠**。

### 5.2 vs ACP(已合并)

**ACP historical note**:
- **Origin**:IBM BeeAI 参考实现
- **传输**:REST-based HTTP(跟 A2A JSON-RPC 不同 — 合并前的历史差异)
- **特点**:MIME type / 同步 + 异步 + streaming / framework-agnostic / **offline discovery via 嵌入式 metadata**(适合 secure / disconnected 环境)
- **2026-05-17 合并状态**:_"part of A2A under the Linux Foundation"_

**A2A 在合并后保留了 ACP 的哪些 idea**:
- 同步 + 异步 + streaming 全模式支持(A2A 3 invocation 模式)
- Framework-agnostic 思路
- offline discovery 没显式纳入 A2A spec,但 `direct configuration` 路线类似

### 5.3 vs REST API / OpenAPI

**REST + OpenAPI** 是 web service 标准,**没有 agent-specific 语义**:
- 没有 Task 状态机(REST 是 _"call → return"_,长任务靠各种 polling hack)
- 没有 INPUT_REQUIRED 反向问用户的协议状态
- 没有 Opaque Collaboration(REST 服务可以共享任何信息)
- 没有 Streaming 标准(REST 靠 SSE / WebSocket hack)

**A2A = REST + agent-aware 设计** — 不是替代 REST,是在 REST 思想上加 agent 时代需要的语义。

### 5.4 vs OpenAI Assistants API

OpenAI Assistants API 也有 thread / run / step 概念,**思路相近**:
- thread ≈ A2A `contextId`
- run ≈ A2A Task
- run 状态机 ≈ A2A Task 8 状态机

**关键差异**:
- Assistants API 是 **OpenAI 私有 API**,只能调 OpenAI 的 assistant
- **A2A 是开放协议**,任何人写的 agent 都能用同一协议互通
- Assistants API 内部假设单一信任域(你跟 OpenAI),**A2A 假设跨信任域**

Assistants API 思路启发了 A2A,但 A2A 是 standardization 之路,不是 vendor lock-in 之路。

### 5.5 vs ChatGPT Plugins(historical)

ChatGPT Plugins(2023-2024)是 OpenAI 的工具暴露协议,**已被 MCP + A2A 双标准实质替代**:
- Plugins 用 `.well-known/ai-plugin.json` 描述 plugin(影响了 A2A `.well-known/agent-card.json` 思路)
- 但 Plugins 限于 OpenAI 平台,不跨厂商
- 现在 OpenAI 也支持 MCP(`developers.openai.com/api/docs/mcp/`)

**Plugins 是 _"协议先驱但产品死掉"_ 的案例**,A2A + MCP 是同方向的开放标准化继承者。

---

## 6. 工程实战 — wiki 内 2 家 A2A 落地路线对照

A2A 协议没规定 _"集成方一定怎么暴露 agent 给本地 LLM"_,wiki 内 2 家 verified 走了完全不同路线:

### 6.1 路线 A — agent-as-tool([[../harness/crewai]])

```
LLM 看到的是一个 fine-grained tool:
  { "name": "call_invoice_analyzer",
    "description": "Invoice analyzer agent (remote)",
    "input_schema": { ... } }
LLM 调用 → CrewAI 内部走 A2A client → 返结果
```

**特点**:
- 简单 — 跟 MCP tool 用法一致,塞 LLM function_calling
- ⚠️ **降级** — 只能用 A2A `message/send` 同步模式,**丢 `message/stream` SSE + INPUT_REQUIRED + Webhook push 三大杀器**

### 6.2 路线 B — agent-as-actor([[../harness/agentscope]])

```
agentscope `a2a/` 5 文件:
  - 跨进程 actor 模式
  - `_nacos_resolver.py` 用 Alibaba Nacos 做服务发现
  - `formatter/_a2a_formatter.py` message-level 协议适配

LLM 看到一个 _"另一个 agent"_,通过显式 delegate / channel 触发
```

**特点**:
- ✅ **完整保留** A2A 语义 — Task 8 状态机 / streaming / human-in-the-loop / IP 保护都能落地
- ⚠️ LLM 需要 multi-agent prompt scaffold — 不再把 A2A agent 当 _"tool"_,而是 _"协作伙伴"_

### 6.3 两条路线的 trade-off

| 维度 | 路线 A | 路线 B |
|---|---|---|
| 工程简单度 | ⭐⭐⭐ 高 | ⭐ 复杂 |
| LLM 易用 | ⭐⭐⭐ 高(就是 tool) | ⭐⭐ 需要 multi-agent 训练 |
| A2A 语义保留 | ⚠️ 降级 | ✅ 完整 |
| 适合场景 | 简单 RPC 风格调用 | 长任务 + 跨组织协作 |

**给 SkillForge 暗示**:如果只走路线 A,A2A 跟 MCP 差异不大,P1 优先级正当;**如果想要路线 B 完整语义**,跟 [[../concept/agent-team-patterns]] R-TEAM-* multi-agent 内部抽象**深度耦合**,需要把 SubAgent / Team / A2A 三个抽象统一。

---

## 7. 学习视角 — 写这页让我学到什么

### 7.1 协议层 _"信任边界"_ 是设计 root

MCP 假设 host 信任 server(但限制 server 信息可见);A2A 假设双方都不完全信任(双向 IP 保护)。**信任假设决定协议所有设计** — Opaque Collaboration / Agent Card 两层 / mutual TLS first-class 都从 _"跨信任域"_ 这个 root 长出来。

**对照 SkillForge**:决定一个内部抽象时,**先想清楚 _"两边什么信任关系"_,再选协议复杂度**。

### 7.2 长任务 first-class 是协议跟 REST 的分水岭

REST 思维 + 30s timeout 是 web 时代假设。Agent 时代任务 _"分钟到天级"_ 是新常态。

**A2A 把这点协议化** — Task 8 状态机 + Resubscribe + Webhook push + contextId 跨 task 持续,**4 个机制叠加** = _"client 可以断开 6 小时再回来接着干"_。这是 A2A 比 REST + SSE hack 优雅的地方。

**对照 SkillForge**:agent loop 设计应该假设 long-running,**INPUT_REQUIRED 状态不应该是异常 retry**,应该是正常 lifecycle 一部分。

### 7.3 协议合并比协议竞争健康(ACP → A2A 案例)

W3C 标准之争经常出现 _"两家各推一份,几年后客户被迫选边"_(Web SQL vs IndexedDB / Push API 各种方言)。

**A2A + ACP 合并** = Linux Foundation 做 _"协议聚合 hub"_,**减少客户决策成本**(_"接 A2A 就够"_)+ 提高生态 buy-in(_"标准统一,投资 A2A 安全"_)。

**对照 SkillForge**:如果有同型对手协议出现,**主动合并 / 互通 比互掐健康** — 但需要中立 hub(LF / CNCF)做调解,单厂商主导的合并通常失败。

### 7.4 Opaque Collaboration 改变多 agent 集成谈判

跨组织接 agent SaaS,90% 谈判时间花在 _"你 prompt 给不给我看 / 我 tool 列表怎么暴露给你"_。**A2A 协议层直接说 _"双方都不用给"_** —— 谈判降到 0,直接谈 _"你接收什么输入 + 返什么输出"_(Task / Message / Artifact)。

**对照 SkillForge**:**未来接第三方 agent SaaS**(Devin / Replit Agent / Jules)时,如果对方支持 A2A,集成成本下降 10x — 不用谈 IP 暴露,只谈 API 接口。

### 7.5 OpenAPI / OAuth 经验直接复用

A2A 不发明新概念,而是 _"借力 web 开发者已有 mental model"_:
- Agent Card ≈ OpenAPI schema
- `.well-known/agent-card.json` ≈ OpenAPI `.well-known/openapi.yaml` + OAuth `.well-known/oauth-authorization-server`
- Extended Agent Card ≈ OAuth public discovery + token-required endpoint

**协议设计经验**:_"复用既有 mental model"_ 比 _"发明优雅新概念"_ 落地快 10x。MCP 借 LSP,A2A 借 OpenAPI/OAuth,都是这种思路。

---

## 8. SkillForge 联动

### 8.1 现状(wiki 内 verified)

- **[[../harness/agentscope]]** A2A first-class 集成最深 — `a2a/` 5 文件 + Nacos resolver + formatter,**跨进程 actor 模式**
- **[[../harness/crewai]]** A2A `a2a/` 子目录 — agent-as-tool 路线
- **SkillForge** A2A 集成现状 **未 verify**(大概率未集成,A2A 是 2025 新协议)

### 8.2 跟 [[../concept/agent-protocols]] §10 联动

详见 [[../concept/agent-protocols]] §10.4-10.8:
- §10.4 **3 边界 mental model** — 同进程 / 同信任域跨进程 / 跨信任域,A2A 在第 3 边界 native
- §10.5 **A2A 两条落地路线** — agent-as-tool vs agent-as-actor
- §10.6 **校准** — CC TeamCreate ≠ A2A(in-process XML vs 跨网络 JSON-RPC)
- §10.7 **skill 封装 vs A2A 决策表** — 7 场景判定
- §10.8 **时间线** — 2026 不必急 / 2027-2028 跨厂商 SaaS 集成必选

### 8.3 SkillForge 3 阶段路线(待 SkillForge 现状 verify 后细化)

```
Phase 1(2026,准备期 — 不实现 A2A,做架构准备):
  └─ Skill 抽象保留 _"远程 vs 本地"_ 切换点
  └─ Skill 元数据对齐 Agent Card schema(name / capability / examples / supportedContentTypes)
  └─ Skill SDK 支持 INPUT_REQUIRED 语义(即使不走 A2A,内部 multi-agent 也能用)

Phase 2(2027,选择性实现 A2A client):
  └─ 集成第三方 agent SaaS 时接 A2A client(Devin / Replit Agent / Jules)
  └─ 借鉴 agentscope `a2a/` 5 文件 + Nacos resolver 模式

Phase 3(2027-2028,A2A server):
  └─ SkillForge 暴露 agent 给外部 SaaS 客户时启用 A2A server
  └─ `.well-known/agent-card.json` 暴露 skill 列表
  └─ 配合 OAuth 2.0 + Extended Agent Card 两层鉴权
```

### 8.4 跟 [[../concept/agent-team-patterns]] 联动

R-TEAM-* 内部 multi-agent 模式(CrewAI Task-driven / AG2 Conversation-driven / MetaGPT Role-play)在**同信任域**内有效。**跨信任域加 A2A 是天然扩展** — SkillForge SubAgent / Team / A2A 应该作为 _"3 个 isolation 层级"_ 统一设计,不要做成 3 套独立抽象。

---

## 9. 引用链路

### 父概念
- [[../concept/agent-protocols]] §3 A2A 节(本 deep 页是 §3 的深挖)+ §10 SkillForge 选型(本页是 §10 的展开)

### 姊妹 deep
- [[../deep/mcp-spec-deep]] — MCP 协议深挖(协议栈第 1 层 vs 本页第 2 层)

### 相关 concept
- [[../concept/multi-agent-isolation]] — A2A _"不共享 internal logic"_ 跟"完全隔离"路线对应
- [[../concept/agent-team-patterns]] — A2A 是 multi-agent team 跨信任域的协议层基础
- [[../concept/agent-interop]] — A2A 在 agent-interop 上下文的定位

### 主要 harness 集成
- [[../harness/agentscope]] — A2A 最深集成(`a2a/` 5 文件 + Nacos resolver,跨进程 actor)
- [[../harness/crewai]] — A2A `a2a/` 子目录(agent-as-tool 路线)
- [[../eval/ag2]] — A2A protocol 对照引用

### 外部 spec
见 frontmatter `sources`。
