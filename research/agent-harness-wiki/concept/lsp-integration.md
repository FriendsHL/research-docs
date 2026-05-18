---
type: concept
name: lsp-integration
status: draft
sources:
  - "[[../source/opencode]]"
  - "/Users/youren/myspace/opencode/packages/opencode/src/lsp/"
  - "/Users/youren/myspace/opencode/packages/opencode/src/tool/lsp.ts"
related_harnesses: [opencode]
---

# LSP Integration — Agent 跟 Language Server 对话

> **4 家独家提示**:本概念目前**只在 opencode 一家**作为 first-class 实现(`packages/opencode/src/lsp/` 6 模块 + `tool/lsp.ts`)。其他 4 家(Hermes/Claude Code/OpenClaw/Codex)虽然多少跟 IDE 集成(Codex `ide/` crate / Claude Code IDE plugins),但**没有把 LSP 协议本身集成到 agent 工具集**。

## 定义

**LSP Integration** = Agent 通过 Language Server Protocol 跟语言服务对话,获得 IDE 级的语义理解能力(diagnostic / hover / definition / references / symbol),而不是只靠 grep / regex 推断代码意图。

**对照传统做法**:
- **Grep / regex 派**(其他 4 家):agent 用 `grep` / `find` / `tree-sitter` 查代码,**靠文本匹配 + 启发式**
- **LSP 派**(opencode):agent 用 LSP 协议查代码,**拿编译器 / 类型系统级语义信息**

## 设计空间(7 维)

### 1. LSP client 实现层级
- **进程外**(opencode `lsp/launch.ts` `lspspawn`):agent 启动 LSP 进程,通过 stdio JSON-RPC 通信
- **进程内**(理论):嵌入 LSP server 库,无 IPC overhead(LSP 多语言写,实际很难)
- **复用 IDE LSP**(Cursor / VS Code 集成时):跟 IDE 共享同一个 LSP 进程

### 2. 协议覆盖
- **完整 LSP**(理论):textDocument/{hover, definition, references, completion, codeAction, ...} 全协议
- **核心几个**(opencode `lsp/lsp.ts:22-49` Schema 定义):**Position / Range / Symbol / DocumentSymbol** 4 个,**diagnostic 单独走** `lsp/diagnostic.ts`
- **只 diagnostic**(理论最小):Agent 只关心错误和警告

### 3. Agent 调 LSP 的入口
- **专门 LSP tool**(opencode `tool/lsp.ts` + `tool/lsp.txt`):一个工具暴露所有 LSP 操作
- **每个 LSP 操作一个 tool**(理论):`lsp_hover` / `lsp_definition` / `lsp_diagnostic` 多工具
- **隐式注入**(理论):agent 跑工具(如 edit)时,framework 自动 LSP 校验,**agent 不直接调**

### 4. Multi-language 支持
- **`lsp/language.ts` 注册多语言**(opencode):TypeScript / Python / Rust / Go 等各自 LSP server
- **单语言**(简化):只支持一种语言

### 5. LSP 启动时机
- **Lazy**:Agent 第一次需要时启动
- **Eager**(opencode `lsp/launch.ts` 推断):Agent 启动时预热常用语言 LSP
- **按文件类型自动**:打开 `.ts` 文件触发 TypeScript LSP

### 6. 事件传播
- **Event-driven**(opencode `lsp/lsp.ts:18-20` `BusEvent.define("lsp.updated", ...)`):LSP 状态变化通过事件总线通知
- **Polling**:agent 每次调 LSP 都拉
- **Push subscription**:agent 订阅特定 LSP 事件

### 7. 安全 / 沙箱
- **进程隔离**(opencode):LSP server 是独立进程,沙箱 OS 级保护
- **共享进程**(理论):LSP server 跟 agent 同进程,互相影响

## opencode LSP 实现详解

### 完整 6 模块(`packages/opencode/src/lsp/`)

| 文件 | 作用 |
|---|---|
| `client.ts` | LSP 客户端(JSON-RPC over stdio) |
| `server.ts` | LSP 服务端(可能是 wrapper 起 LSP server 子进程) |
| `diagnostic.ts` | 实时编译错误 / 警告 |
| `language.ts` | 语言注册(多 LSP 协调) |
| `launch.ts` | LSP 进程启动(`lspspawn`) |
| `lsp.ts` | 主入口 + Schema 定义(`Position` / `Range` / `Symbol` / `DocumentSymbol`) |

### Effect-TS Schema 类型化 LSP 协议(`lsp/lsp.ts:22-49`)

```ts
const Position = Schema.Struct({
  line: NonNegativeInt,
  character: NonNegativeInt,
})
export const Range = Schema.Struct({
  start: Position,
  end: Position,
}).annotate({ identifier: "Range" })
export const Symbol = Schema.Struct({
  name: Schema.String,
  kind: NonNegativeInt,
  location: Schema.Struct({
    uri: Schema.String,
    range: Range,
  }),
}).annotate({ identifier: "Symbol" })
```

业内独家:**LSP 协议用 Effect-TS Schema 强类型化**(其他实现多用 zod 或 ad-hoc 类型)。

### Agent 调 LSP tool(`tool/lsp.ts` + `lsp.txt`)

Agent 通过统一 `lsp` tool 操作:
- 拿 diagnostic(实时编译错误)
- 拿 hover(类型 / 文档)
- 拿 definition(跳转定义)
- 拿 symbol(符号树)

**Bus 事件驱动**:`BusEvent.define("lsp.updated", ...)` — LSP 状态变化通过事件总线通知,agent 可订阅。

## 与其他 harness 对比

| Harness | LSP 集成 | 备注 |
|---|---|---|
| **opencode** | ✅ first-class(`lsp/` 6 模块 + `tool/lsp.ts`) | **业内独家** |
| Claude Code | 部分(IDE plugin 集成,VS Code/Cursor/Windsurf 走 IDE 自家 LSP) | 不是 agent 工具集成 |
| Codex | 部分(`codex-rs/ide/` 走 IDE 集成) | LSP 不是 agent 工具 |
| Hermes | ❌ | grep / 文件读为主 |
| OpenClaw | ❌ | 同 Hermes |

**为什么其他家没做**:
1. **多语言适配成本高** — 每个语言一个 LSP server,启动 + 通信 + 类型映射工程量大
2. **业务定位不同** — Hermes 是 RL training research,OpenClaw 是 personal assistant,Claude Code / Codex 主要靠 IDE plugin 把 IDE LSP 给用户(而非给 agent)
3. **LSP 协议复杂** — 完整 LSP 几百个 method,只挑核心几个仍需工程
4. **Effect-TS / 强类型生态适配**:opencode 选择 Effect-TS 后,Schema 类型化 LSP 协议比较自然

## Trade-off

### LSP 集成的优势

1. **语义级代码理解** — agent 不靠文本匹配,直接拿编译器 / 类型系统知识
2. **diagnostic 实时反馈** — agent 改代码后立即知道是否编译错误,无需跑 build
3. **跳转定义** — agent 改一个 function,用 LSP `textDocument/references` 拿调用点,**改全调用点跨文件**
4. **类型推导辅助** — agent 知道类型,生成的代码类型正确率高
5. **Symbol tree 探索** — agent 用 `documentSymbol` 拿文件结构,不需 grep 全文件

### LSP 集成的代价

1. **多语言适配工程量** — 每个语言一个 LSP server,opencode `lsp/language.ts` 抽象层做这事
2. **LSP server 启动慢** — TypeScript LSP 启动可能 5-10s,Lazy 启动会有第一次调用延迟
3. **进程开销** — N 个 LSP server 进程,对长 session 内存占用大
4. **不是所有语言都有好 LSP** — Bash / SQL / 配置文件 LSP 弱
5. **LSP 协议本身复杂** — 几百个 method,边缘情况多

## SkillForge 当前选择 + 演进方向

**当前**:SkillForge coding agent 主要靠 grep / file read / shell run(无 LSP)

**演进选项**:
1. **不引入**(默认):工程量大,SkillForge 多语言场景压力大
2. **引入 1 个语言 POC**(推荐):**TypeScript LSP**(用户最广)用 opencode `lsp/` 6 模块做参考实现
3. **引入完整 LSP 抽象**:opencode `lsp/language.ts` 抽象 + 多语言 server 注册
4. **复用 IDE LSP**:让 SkillForge 用户的 IDE(Cursor / VS Code)把 LSP 共享给 SkillForge agent

**判定**:**LSP 集成是 SkillForge coding agent 真正缺口**,业内只有 opencode 做。借鉴优先级:
- 短期:不做(工程量大)
- 中期:**TS LSP POC**(单语言,opencode `lsp/` 模块直接抄改)
- 长期:多语言抽象 + Diagnostic 实时反馈集成 SkillForge eval

**关键决策**:
- **JVM 嵌 LSP client 没问题**(LSP 是 JSON-RPC over stdio,任何语言都能写 client)
- **复用现有 LSP server**(不需重写 server,直接 spawn 系统装的 LSP server 进程,如 `typescript-language-server` / `pyright`)
- **Effect-TS Schema 替代品**:Java 用 Spring `@Validated` + Jackson schema,功能等价

## 学习视角:为什么 LSP 集成现在才在 agent 里出现

### 历史脉络

- **2016**:Microsoft 推出 LSP,VS Code 用,IDE 标准
- **2018-2022**:LSP 在 IDE / editor 普及(Vim / Emacs / JetBrains 都支持)
- **2023**:**LLM coding agent 出现**(GitHub Copilot Chat / Claude Code 等),仍主要靠 grep / file read
- **2024**:开始有 agent 集成 IDE 但仅 UI 层(VS Code 插件,LSP 在 IDE 内,agent 不直接看)
- **2026 opencode**:**首次把 LSP 协议作为 agent first-class 工具**(`tool/lsp.ts`)

### 为什么 agent 终于需要 LSP

1. **agent 修复 bug 需要 diagnostic** — 没有 LSP,agent 改完代码不知道有没有编译错误,要 build / run 才知道
2. **agent 重构需要 references** — 改 function 名要全 codebase 找调用点,grep 找不全(动态分发 / 间接调用)
3. **agent 生成代码需要类型推导** — 没有 LSP,agent 写 `.method()` 不知道类型有没有这个方法
4. **大 codebase 探索成本高** — `repo_overview` tool + LSP `documentSymbol` 比 grep 全文件高效

### 跟 IDE plugin 的关系

| | LSP-in-agent(opencode 路线) | IDE plugin(Claude Code / Codex 路线) |
|---|---|---|
| LSP 在哪 | agent 进程内 | IDE 进程内 |
| Agent 看 LSP | ✅ 直接调 LSP tool | ❌ 不直接看,需 IDE 中转 |
| 适合场景 | TUI / headless / mobile / server-side agent | IDE 内交互式编辑 |
| SkillForge 适合 | ✅ server-side 自然 | ❌ SkillForge 不是 IDE plugin |

**SkillForge server-side 定位 → opencode LSP-in-agent 路线匹配度高**。

## 反模式

- ❌ **agent 改代码不验证** — 没有 LSP diagnostic 等价物,改完不知道编译失败
- ❌ **靠 grep 找 references 重构** — 漏掉动态调用,引入 silent regression
- ❌ **每个 LSP 操作一个 tool** — 扩散 tool 数,agent 选择困难,opencode 单 `lsp` tool 加 sub-action 更清晰
- ❌ **LSP 启动失败 silent ignore** — 应有 fallback 到 grep,但要明示 agent "LSP unavailable, using grep fallback"
- ❌ **LSP server 共享进程** — 一个 LSP 崩了影响 agent,opencode `lspspawn` 进程隔离
- ❌ **完整 LSP 协议照单全收** — 几百个 method,agent 实际只需 hover / definition / references / diagnostic / documentSymbol 5 个
