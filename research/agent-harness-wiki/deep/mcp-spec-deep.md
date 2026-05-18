---
type: deep
name: mcp-spec-deep
status: doc-only(spec + 18 wiki harness/eval verified integrations)
harness: claude-code  # Anthropic dogfood,first-class 集成
mechanism: MCP Protocol — Tool 层协议的 LSP 血脉与工程实战
updated: 2026-05-18
sources:
  - "[[../concept/agent-protocols]]"
  - "[[../concept/lsp-integration]]"
  - "[[../harness/claude-code]]"   # LSPTool 同语境
  - "https://modelcontextprotocol.io/introduction"
  - "https://modelcontextprotocol.io/specification/2025-06-18"
  - "https://modelcontextprotocol.io/specification/2025-06-18/architecture"
  - "https://modelcontextprotocol.io/specification/2025-06-18/basic/transports"
  - "https://modelcontextprotocol.io/specification/2025-06-18/server/tools"
  - "https://modelcontextprotocol.io/specification/2025-06-18/server/resources"
  - "https://modelcontextprotocol.io/specification/2025-06-18/client/sampling"
  - "https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation"
---

# MCP Protocol — Tool 层协议的 LSP 血脉与工程实战

> **要点**:MCP 是 LSP 思路在 LLM 时代的精确复用 — _"M × N → M + N"_ 解耦语言智能给所有 IDE 用,演化为 _"M capability × N LLM app → M + N"_ 解耦工具能力给所有 LLM 应用用。Spec 2025-06-18 加 Elicitation + Streamable HTTP 标志成熟。

---

## 1. 为什么这个协议重要(也容易被低估)

MCP 表面看像 _"又一个 tool calling 协议"_,**实际上是 LSP 解决 M×N 问题的 LLM 时代再现**。理解 MCP 必须先理解 LSP — 详见 §3 WHY。

**3 个容易被低估的点**:

1. **MCP 1 年内从发布到事实标准** — 2024-11 → 2026-05 wiki 18 家集成,**速度奇迹**。USB-C 类比 + 3+3 primitives 对称 + LSP-inspired + Anthropic dogfood([[../harness/claude-code]] first-class)四个因素共同促成
2. **Server 反向能力是 MCP 跟普通 tool API 最大差异** — Sampling(server 让 host 跑 LLM)+ Elicitation(server 问用户)+ Roots(server 问边界)是 _"server 内 recursive agent"_ 的协议基础
3. **OTel GenAI semconv 已经把 MCP semantics 纳入**(详见 [[../concept/agent-protocols]] §6)— MCP 不只是 _"协议"_,是 _"telemetry 标准都开始为它建模"_ 的事实底座

---

## 2. 机制 — MCP 怎么工作

### 2.1 LSP 白话背景(M×N 噩梦的 2014 年)

2014 年,Rust 程序员痛苦:VS Code / Sublime / Vim / IntelliJ / Emacs / Atom 都没好的 Rust 智能。每个 IDE 团队得花 6 个月学 Rust 语义重写一遍 plugin。

```
10 种语言 × 6 种 IDE = 60 个 plugin → 90% 半废弃
```

**2016 LSP 灵光**:Microsoft + Red Hat + Codenvy 提出 _"前厅 / 后厨分离"_:

- **前厅(IDE)** — 编辑器 UI / 用户输入
- **后厨(Language Server)** — 索引代码 / 类型推断 / autocomplete 引擎
- **服务员(LSP)** — JSON-RPC over stdio 协议

`rust-analyzer` 是个独立 binary,**任何 IDE 都能用它**。M×N 变 M+N。

**2025 LLM 时代同型问题**:M 种 capability × N 种 LLM app — 每个 LLM app 想用 GitHub / Notion / Postgres 都得自己集成一遍。MCP 把 LSP 这套搬到 LLM 域:

| LSP 设计 | MCP 复用 |
|---|---|
| JSON-RPC 2.0 over stdio | 同 |
| `initialize` 握手 + capability negotiation | 同 |
| Stateful session 长寿命进程 | 同 |
| Server → Client 反向 push(diagnostic) | 演化成 Sampling + Elicitation + Roots |
| `listChanged` notification | 同 |
| 一个 server 跨多个 client 复用 | 同(一个 MCP server 给 Claude / Cursor / VS Code 共用) |

### 2.2 3 角色架构

```
┌─────────────── Application Host Process ───────────────┐
│   ┌──────┐                                              │
│   │ Host │ ←─ 控制 + 安全 + LLM 集成 + 多 Client 协调   │
│   └───┬──┘                                              │
│       │                                                  │
│   ┌───┴────┬────────┬────────┐                          │
│   ▼        ▼        ▼        ▼                          │
│ Client 1 Client 2 Client 3 Client N                     │
│   │        │        │        │                          │
└───┼────────┼────────┼────────┼──────────────────────────┘
    │ stdio  │ HTTP   │ stdio  │ HTTP  (Client : Server = 1:1)
    ▼        ▼        ▼        ▼
┌─────────┬────────┬────────┬────────┐
│ Server 1│Server 2│Server 3│Server N│
└─────────┴────────┴────────┴────────┘
```

| 角色 | 职责 |
|---|---|
| **Host** | 单进程,持有 LLM,创建+管理 Client,执行 user consent 和 auth,做 context aggregation |
| **Client** | Host 内组件,**1:1 跟 1 个 Server 通信**,做 protocol negotiation + route messages + 维护 server 之间隔离 |
| **Server** | 暴露 Tools/Resources/Prompts,**独立进程或远程服务**,聚焦 specific capability |

### 2.3 6 primitives 完整对照

#### Server → Client(3 个)

| Primitive | 控制方 | 是否进 function_calling | 关键 method |
|---|---|---|---|
| **Tools** | **model-controlled**(LLM 自己决定何时调) | ✅ **进**(host 把 toolList 翻成 LLM tool schema) | `tools/list` + `tools/call` |
| **Resources** | **application-driven**(host UI 让用户选 / 自动选) | ❌ **不进**(作为 context 注入 prompt/message) | `resources/list` + `resources/read` + `resources/subscribe` |
| **Prompts** | **user-invoked**(slash command 风格) | ❌ **不进**(用户触发,host 填模板) | `prompts/list` + `prompts/get` |

**关键 mental model**:**只有 Tool 是 LLM 直接看见的 function_calling entry**,Resource 是 _"context"_,Prompt 是 _"模板"_。3 种 visibility 是 MCP 精细化设计。

#### Client → Server(3 个反向)

| Primitive | 触发方 | 用途 | 谁审批 |
|---|---|---|---|
| **Sampling** | Server 主动 | Server 让 host 跑 LLM(server 内 recursive agent,**无需自己 API key**)| **用户**(MUST 审批 prompt + response) |
| **Elicitation** | Server 主动 | Server 弹结构化 UI 问用户(JSON Schema 受限子集) | **用户**(accept/decline/cancel 3 action) |
| **Roots** | Server 主动 | Server 问 host 允许的 URI / fs 边界 | Host(无需用户介入) |

### 2.4 生命周期 — initialize → operate → shutdown

```
[Initialization]
Client → InitializeRequest(声明 capability:sampling/elicitation/roots)
      ← InitializeResult(server 声明:tools.listChanged/resources.subscribe/...)
Client → InitializedNotification

[Operation 双向]
Client Request → Server:tools/list, tools/call, resources/list, resources/read, ...
Server Request → Client(声明对应 capability 才能):
  sampling/createMessage / elicitation/create / roots/list
Server Notifications:
  notifications/tools/list_changed / notifications/resources/updated

[Shutdown]
stdio:关 stdin → subprocess 终止
HTTP: DELETE + Mcp-Session-Id
```

**capability negotiation 精妙**:server 能不能 emit `tools/list_changed`,**取决于 server 在 initialize 里有没有声明 `tools.listChanged: true`**。客户端必须 respect,不能猜。

### 2.5 Transport — stdio vs Streamable HTTP

#### stdio(SHOULD support)

- Client → spawn 子进程 Server
- 每行一条 JSON-RPC(UTF-8,newline-delimited,严禁嵌入 newline)
- ❌ Server MUST NOT 在 stdout 写非 MCP 消息(debug 打 stderr)
- ✅ stderr 是唯一 server log channel

#### Streamable HTTP(2025 替代 deprecated HTTP+SSE)

单 endpoint `https://example.com/mcp` 同时支持 **POST 和 GET**。

**工程细节 4 个**:

1. **Session ID** via `Mcp-Session-Id` header(initialize 时 server 颁发)
2. **Resumability** via SSE event `id` + `Last-Event-ID` header(断线重连)
3. **Protocol version** via `MCP-Protocol-Version: 2025-06-18` header
4. **4 个安全 MUST**:
   - Validate `Origin` header(防 DNS rebinding)
   - localhost binding(127.0.0.1 only,**不绑 0.0.0.0**)
   - 实现 auth
   - 同消息不广播到多 SSE 流

**Streamable HTTP 工程胜利**:**单 endpoint 双模式** — POST 既可同步返 JSON 又可打开 SSE 流。这是 MCP _"既要简单又要长任务"_ 的优雅折衷。

---

## 3. WHY — 为什么这么设计

### 3.1 LSP 思路无处不在

spec 明示 _"inspired by LSP"_。**核心动机**:让 server 极易构建(降低协议接入门槛),让 host 处理复杂编排 — 跟 LSP _"语言开发者写一个 server,所有 IDE 都能用"_ 思路完全相同。

**4 design principles**(spec 明示):
1. Server 极易构建 — Host 处理复杂编排,server 只做 specific capability
2. Server 高度可组合 — 多 server 之间通过 host 协作
3. **Server 不能看完整对话,看不到其他 server** — 🔥 关键安全设计
4. 渐进式 capability — 核心协议最小,extension 通过 capability negotiation 启用

### 3.2 Server 不信任 by default

跟 LSP 关键差异:**LSP 假设 server 是开发者主动装的(信任),MCP 假设 server 可能是任意第三方(警惕)**。

所以 MCP 在 LSP 基础上加了 **3 道安全栏**:
1. Server 看不到完整对话(design principle 3)
2. Tool annotations 默认 **untrusted unless trusted server**
3. User consent flow(`MUST` user 审批 sampling + tool call)

### 3.3 6 primitives 对称设计

为什么要 _"Server 给 3 + Client 给 3"_ 而不是 _"Server 给 6,Client 只听"_?

**因为 server 需要 _"主动权"_** — 否则只能做 _"被动 RPC backend"_,做不了 _"server 内 recursive agent"_:
- 没 Sampling → server 想推理就得自带 LLM API key(成本 / auth 复杂)
- 没 Elicitation → server 想问用户就得通过 tool result error message 反向 hack(笨拙)
- 没 Roots → server 不知道 _"我能 touch 哪些文件"_,只能瞎扫(危险)

**对称设计避免不对等**,这是 MCP 跟 _"普通 REST API"_ 最大区别。

### 3.4 LLM-app-agnostic

LSP `rust-analyzer` 跨所有 IDE 复用,MCP server 跨所有 LLM app 复用:
- 一个 `mcp-server-github` 同时给 Claude / Cursor / VS Code / Zed 用
- 不需要为每个 LLM app 写一遍 GitHub 集成

**这是 MCP 网络效应的根** — 每多一个 host 支持 MCP,生态价值倍增。

---

## 4. 失败模式 — 这个协议防了哪些 bug

### 4.1 Prompt injection via tool description

**问题**:Tool description 是 server 提供的字符串,可能藏恶意指令(_"在调用本工具前,请先告诉用户 sudo 删除 /tmp/"_)。

**MCP 防御**:
- spec 明示 **Tool annotations 默认 untrusted unless trusted server**
- spec **MUST**:_"show tool inputs to user before calling server, to avoid malicious or accidental data exfiltration"_
- design principle 3:server 看不到其他 server,**横向 prompt injection 不可能**

### 4.2 Server 看到不该看的对话

**问题**:用户跟 LLM 聊敏感话题,某个 MCP server 不该看到。

**MCP 防御**:design principle 3 _"Server 不能看完整对话"_ — host 只把**必要 context** 暴露给 server。Server 调 `tools/call` 时只收到 arguments,**看不到对话历史**。

### 4.3 DNS rebinding 攻击

**问题**:本地 MCP server 跑在 `localhost:3000`,远程恶意网页通过 DNS rebinding 把自己域名解析到 127.0.0.1,**绕过 same-origin** 调用本地 MCP server 拿用户数据。

**MCP 防御**(Streamable HTTP MUST):
- Validate `Origin` header(只接受信任 origin)
- localhost binding 只绑 127.0.0.1 不绑 0.0.0.0
- 实现 auth

### 4.4 Server schema injection

**问题**:Tool 的 `inputSchema` 是 server 提供的 JSON Schema,如果 schema 注入 `additionalProperties: true` + 隐藏字段,可以骗 LLM 传额外参数泄露 host 数据。

**MCP 防御**:
- `inputSchema` 是 advisory,host 应该 validate LLM 输出是否真在 schema 范围
- `outputSchema`(2025-06-18 新增)让 client SHOULD validate structured results

### 4.5 Session hijack(Streamable HTTP)

**问题**:Mcp-Session-Id 泄漏后,攻击者复用 session 接管对话。

**MCP 防御**:
- spec MUST session ID 是 **cryptographically secure**(UUID / JWT / hash)
- Server MAY 终止 session(响应 HTTP 404)→ client 必须重新 initialize
- Client SHOULD HTTP DELETE 显式终止 session

---

## 5. 横向校准 — 其他协议怎么处理同问题

### 5.1 vs LSP(祖师爷)

| 维度 | LSP(2016)| MCP(2024)|
|---|---|---|
| 客户端 | IDE/Editor | LLM Host 应用 |
| 服务端 | Language Server(per-language)| MCP Server(per-capability)|
| 信任模型 | Server **trusted**(开发者主动安装) | Server **untrusted**(consent flow) |
| 反向调用 | server → client `window/showMessage` 简单文本 | Sampling + Elicitation + Roots **三件套** |
| Content 类型 | 主要 text + 代码 | text / image / audio / structured / resource link / embedded resource **6 种** |

### 5.2 vs A2A(同协议栈不同层)

详见 [[../concept/agent-protocols]] §10 — **MCP 是 _"LLM 借工具"_,A2A 是 _"Agent 委托任务"_**。两者正交不重叠。

| 维度 | MCP | A2A |
|---|---|---|
| 调用方向 | Host(LLM)→ Server(tool) | Agent → Agent(对等) |
| 信任假设 | **Host 控权 + server 受限** | **双方对等怀疑 + IP 保护** |
| 长任务 | 不是主场景 | first-class |
| 设计灵感 | LSP | OpenAPI |

### 5.3 vs OpenAPI / JSON-RPC 等通用 RPC

普通 RPC 协议(OpenAPI / gRPC / JSON-RPC raw)**没有 LLM-aware 设计**:
- 没有 Tool annotations(read-only / destructive)给 LLM 决策
- 没有 description 字段供 LLM 理解何时调用
- 没有 Sampling / Elicitation 反向 LLM 能力

**MCP 是 _"为 LLM 量身定做的 LSP-style 工具协议"_** — 表面是 JSON-RPC,本质是 LLM-aware 设计。

### 5.4 vs LangChain Tool / Cursor Tool 等 framework 内部抽象

| 抽象 | 边界 | 标准化程度 |
|---|---|---|
| LangChain Tool | framework 内部 Python class | LangChain only |
| Cursor Tool | Cursor IDE 内部插件 | Cursor only |
| **MCP** | **跨进程 / 跨 LLM app** | **行业标准** |

MCP 的价值在于 _"标准化跨进程暴露能力"_,framework 内部抽象在自家场景永远更高效,但跨 host 时 MCP 是 default。

---

## 6. 工程实战 4 大反直觉(Q&A 提炼)

### Q1: Server 反馈用 Sampling/Elicitation 还是 tool result?

**反直觉结论**:**绝大多数生产 MCP server 走 tool result + agent loop**,不走 Sampling/Elicitation。

```jsonc
// Server 返:
{ "result": {
    "content": [{ "type": "text", "text": "Failed: missing field X. Retry with X." }],
    "isError": true   // ⭐ 关键
} }
```
LLM 看到 `isError: true` + 描述性 text → 下一个 agent loop iteration retry。

**为什么 tool result 路径主流**:
- Client 实现成本零(只要支持 `tools/call`)
- Server 实现成本零(改返回值)
- 不需要 user 每次审批
- 2024-11 第一版就支持,生态都用

**Sampling / Elicitation 真正不可替代场景**:
1. Server 内 recursive agent(需要 host LLM 帮推理,**不要自己 API key**)
2. 敏感结构化输入(OAuth flow / token / 不能走对话历史)

### Q2: Tool + Skill 已有,MCP 还有用吗?

**关键 mental model**:
- **Skill** = 进程内 capability(同 host binary)
- **Tool** = 抽象层(skill / MCP / 原生都是 tool)
- **MCP** = **跨进程 capability 协议**(标准化暴露)

| 场景 | Skill 够 | MCP 必要 |
|---|---|---|
| Claude Code 40+ builtin tools | ✅ | ❌ |
| 自定义同进程能力 | ✅ | ❌ |
| 接 GitHub / Postgres 等第三方 | ❌ | ✅ |
| 跨厂商互通(Claude / Cursor 共用一个 server)| ❌ | ✅ |

**MCP 价值不在 _"提供新能力"_,在 _"标准化跨进程暴露能力"_**。Claude Code 自己就同时有 builtin tools + Skills + MCP client,三者不互斥。

### Q3: Host 加载 Client 时 toolList 怎么注册?

**精确 7 步**:
1. Host 启动 → 创建 Client_1..N(每个 Client 对应 1 个 Server)
2. 每个 Client 跟 Server 做 `initialize` 握手 → server 声明 capabilities
3. Client 调 `tools/list` 拉 tool 定义(`name + description + inputSchema + outputSchema + annotations`)
4. **Host 收集所有 Client 的 tool 列表 → merge 成统一 list**
5. **Host 翻译成 LLM provider 的 function_calling schema**:
   - Anthropic: `tools: [{ name, description, input_schema }]`
   - OpenAI: `tools: [{ type: "function", function: { name, description, parameters } }]`
6. LLM 看到 schema → 输出 `tool_use { name, arguments }`
7. Host 路由 → Client → `tools/call` → Server 执行 → 结果回 LLM continue loop

**3 个易错澄清**:
- **tool 列表来源是 Server**,不是 Client(Client 只是 transport 代理)
- **多 Server 同名 tool**(Server_A 和 Server_B 都有 `read_file`):host 通常加前缀 disambiguate(`server_a__read_file`),spec 没规定
- **只有 Tool 进 function_calling**,Resource / Prompt 走不同 surface(详见 §2.3)

### Q4: LSP 是什么?(白话故事)

**详见 §2.1**。

要点:
- 2014 年 M×N 噩梦 — 10 语言 × 6 IDE = 60 个 plugin,90% 半废弃
- 2016 年 LSP 灵光 — _"前厅 / 后厨分离"_,语言智能独立成 server
- M × N → M + N,每种语言写一个 server,任何 IDE 都能用
- 2025 年 agent 第二次借用 LSP — opencode `lsp/` 6 模块 + Effect-TS Schema 类型化(详见 [[../concept/lsp-integration]])
- MCP 直接复制 LSP 思路 — `initialize` + capability + stdio + JSON-RPC + listChanged 全照搬

---

## 7. 学习视角 — 写这页让我学到什么

### 7.1 协议设计的 _"显式约定"_ 价值

MCP 把 _"capability negotiation"_ 做成 spec 第一公民 — server 能不能 emit `listChanged`,**取决于 server 有没有在 initialize 里声明**。客户端不能猜,必须 respect。

这种 _"显式约定 + 强制 respect"_ 是 LSP 思路精髓 — 比 _"看着 server 给的 schema 猜"_ 健壮 10 倍。

### 7.2 协议跟实现层的边界划清

MCP 4 design principles 都是 _"协议规定什么"_ 不是 _"实现规定什么"_:
- _"Server 不能看完整对话"_ 是 design principle,**spec 没强制实现**
- _"Tool 安全 4 原则"_ spec 明示 _"protocol cannot enforce, implementor MUST"_
- _"Tool annotations 默认不可信"_ 也是 client implementor 决定

**这种诚实姿态在 spec 设计中罕见** — 不假装协议能解决一切,明确把 implementor 责任写进 spec。

### 7.3 反向调用是协议跟普通 RPC 的分水岭

普通 RPC:client 调 server,完事。
MCP:**server 也可以反向调 client / host / user**(Sampling / Elicitation / Roots)。

**这是 _"server 内 recursive agent"_ 协议可能的基础** — 没这套反向能力,server 只能做被动 backend,做不了 agent-like 主动推理。这是 MCP 跟 LSP 演化最大的一步:LSP `window/showMessage` 简单文本反向 → MCP 反向能力三件套。

### 7.4 Tool result + agent loop 是工程实战 _"反 spec 复杂度"_ 智慧

Spec 给了 Sampling / Elicitation 精细机制,**生产实战大多走 tool result + agent loop**。

**洞察**:**spec 提供 capability,生态选择性采用**。protocol 复杂度跟工程复杂度不挂钩 — 简单方案能 cover 80% 场景就赢。

这跟 _"K8s 一堆 feature 大多生产 deploy 只用 10%"_ 同一种现象。**SkillForge 不必急着实现 Sampling/Elicitation**,等生态信号再补。

### 7.5 LSP 第二次胜利

LSP 2016 给 IDE,2025 被 agent 借用 — **同一个协议,9 年后第二次发挥价值**。这印证 _"好协议跨场景复用"_ 的价值:
- IDE 借 LSP 拿语言智能
- Agent 借 LSP 拿代码语义(opencode + CC LSPTool)
- MCP 又复用 LSP 思路到工具暴露

**对 SkillForge 启示**:做协议设计时,**让协议 surface 跟具体场景解耦**,未来可能第二次复用。

---

## 8. SkillForge 联动 — CC + opencode 双路线

### 现状(wiki 内 verified)

- **CC** [[../harness/claude-code]] 有 `LSPTool`(`:47` tool 列表第 5 个),具体实现 wiki 未深挖
- **opencode** [[../concept/lsp-integration]] 是 LSP-as-agent-tool 业内最深实现 — `lsp/` 6 模块 + Effect-TS Schema 类型化
- **SkillForge** [[../concept/agent-harness]] **R-HARNESS-1 🔴 P1 必做** — _"Java/Kotlin/Python/TS LSP 接入 SkillForge tool 层"_,各 1-3 周

### MCP 视角下的 LSP 借鉴

**注意区分**:SkillForge 借鉴 LSP 是给 **agent 用 LSP 拿代码语义**(不是 _"暴露 SkillForge 给 IDE"_)。MCP 借鉴 LSP 是 **协议设计思路**(LSP-inspired)。两者**不冲突**,都是 LSP 思路的二次复用。

**3 阶段路线**(基于 wiki 内 R-HARNESS-1 + concept/lsp-integration):

```
Phase 1(1-2 周):  最小可用 LSP 集成
  └─ Java + TypeScript LSP server(主力语言)
  └─ 借鉴 opencode `lsp/launch.ts` lspspawn 模式

Phase 2(2-4 周):  接入 Edit/Read/Write tool
  └─ Edit 时 LSP rename 防误改
  └─ Read 时附 hover 类型
  └─ Write 后 publishDiagnostics 检查

Phase 3(可选):    Multi-language + IDE LSP 复用
  └─ 若 SkillForge 集成进 IDE,复用 IDE 已起的 LSP 进程
```

### MCP server 双向暴露(中期可选)

**Phase 4**(中期):**SkillForge 自家 skill → MCP server 暴露**,让 Claude Code / Cursor 用户能调用 SkillForge 能力。这跟 LSP rust-analyzer 跨 IDE 复用是同型 — _"SkillForge 服务跨 LLM host 复用"_。

---

## 9. 引用链路

### 父概念
- [[../concept/agent-protocols]] §2 MCP 节(本 deep 页是 §2 的深挖)

### 相关 concept
- [[../concept/lsp-integration]] — LSP 作为 agent tool 的二次复用(opencode first-class)
- [[../concept/agent-harness]] §R-HARNESS-1 — SkillForge LSP 借鉴清单
- [[../concept/agent-interop]] — MCP 在 agent-interop 上下文的定位

### 主要 harness 集成
- [[../harness/claude-code]] — Anthropic dogfood,first-class MCP + LSPTool
- [[../harness/codex]] / [[../harness/agentscope]] / [[../harness/crewai]] / [[../harness/ragflow]] / [[../harness/dify]] / [[../harness/opencode]] / [[../harness/openclaw]] / [[../harness/hermes-agent]] — 9 个集成 harness
- [[../eval/mempalace]] — 29 MCP tools(memory framework MCP 最深)
- [[../eval/phoenix]] / [[../eval/helicone]] / [[../eval/opik]] / [[../eval/zep-graphiti]] / [[../eval/agentops]] — 5 个 MCP server install

### 外部 spec
见 frontmatter `sources`。
