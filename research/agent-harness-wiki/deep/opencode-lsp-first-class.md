---
type: deep
name: opencode-lsp-first-class
status: B-mode source-grounded
harness: opencode
mechanism: 9 LSP operations as agent tool via Effect-TS Schema + 80+ language extension mapping + per-file LSP client + touch-before-query
updated: 2026-05-17
sources:
  - "[[harness/opencode]]"
  - "/Users/youren/myspace/opencode/packages/opencode/src/tool/lsp.ts:1-113"
  - "/Users/youren/myspace/opencode/packages/opencode/src/lsp/lsp.ts:1-150"
  - "/Users/youren/myspace/opencode/packages/opencode/src/lsp/language.ts:1-121"
---

# opencode LSP first-class — coding agent 看见的不是文本,而是 AST + symbol + reference

> 要点:opencode `lsp/` 6 模块 + `tool/lsp.ts` 113 行,把 **LSP 9 操作直接暴露为 agent 可调用的 tool**,业内独家(4 个 production coding agent 里只有 opencode 这么做)。读完发现 3 件反直觉的事:**(1) 把 LSP 包成 agent tool 是 13 行 Effect-TS Schema 的事,真复杂度在 server.ts 2065 行的 80+ 语言 LSP server config 上**;**(2) `touchFile` before query 是关键 — 不 touch 的话 server 看不到 unsaved buffer**;**(3) LSP "1-based line/character" 跟 LSP spec "0-based" 的差异由 tool layer 转换,agent 看到的是 "editor 风" 不是 "LSP spec 风"**。
> 
> 这页深挖 _"为什么 LSP-as-agent-tool 是 coding agent 的下一代默认范式"_ + opencode 的实现选择 + 跟其他 coding agent(Claude Code / Codex / Aider / Cursor)的对照。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通用 Claude Code / Cursor 改代码的人,有过这种体验:
- 让 AI 改一个 function,它**只看到那个文件**,不知道哪些地方 call 这个 function
- 让 AI rename 一个 variable,它**漏掉**3 个其他文件里的引用
- 让 AI 用某个 type,它**重新编出来一个**,因为不知道项目已有这个 type

为什么?**Coding agent 默认只看 _plain text file_**。它没有:
- ❌ Symbol table(class / function / variable 定义在哪)
- ❌ Reference graph(哪些地方 call 这个 function)
- ❌ Type info(这个 variable 是什么类型)
- ❌ Diagnostic(编译器/linter 报错信息)
- ❌ Hover(symbol 的 doc string)
- ❌ Definition(从 call site 跳到定义)

**而 IDE 知道这些,通过 LSP**(Language Server Protocol — Microsoft 2016 提的标准)。

opencode 做的事:**把 IDE 用的 LSP 直接给 agent 用**。Agent 调用 `lsp({operation: "findReferences", filePath, line, character})` 就拿到所有 call site。Agent 调用 `lsp({operation: "documentSymbol", ...})` 就拿到文件 outline。

**业内对照**:
| Coding agent | 怎么拿代码语义信息? |
|---|---|
| **Claude Code** | grep / find / read 拼接(无 LSP) |
| **Codex** | grep / find / read 拼接(无 LSP) |
| **Cursor** | IDE 自带 LSP,但 agent 通过 _读 IDE 当前 selection / open file_ 拿到部分语义 |
| **Aider** | grep + treesitter parsing(有 AST 但不通 LSP)|
| **opencode** | ⭐ **LSP first-class as agent tool**(9 操作完整暴露) |

**为什么容易被忽略**:
1. README 不会直白讲 "我们让 agent 用 LSP" — opencode README 说 _"LSP first-class"_ 一句话
2. 实现非常薄(`tool/lsp.ts` 只有 113 行)— 真复杂度在 `lsp/` 子目录 6 文件
3. 大部分用户体验上感受不到差别 — 直到他们让 agent 做大型 refactor 才发现差距

**对 coding agent 设计的启示**:**LSP-as-agent-tool 是 coding agent 的下一代默认范式**,因为:
- LSP server 是开源 ecosystem(80+ 语言现成)
- LSP 协议是 stable spec(2016 起 9 年迭代,接口稳定)
- Agent 工程量小(opencode 113 行 tool)
- ROI 大(大型 refactor / cross-file 操作准确率显著提升)

---

## 2. 机制 — `tool/lsp.ts` 113 行做对了什么

### 2.1 9 LSP operations 作为 agent tool

```typescript
[code: tool/lsp.ts:11-21]
const operations = [
  "goToDefinition",     // 跳到 symbol 定义
  "findReferences",     // 找所有 call site
  "hover",              // 鼠标悬停 doc
  "documentSymbol",     // 文件 outline (所有 class/function)
  "workspaceSymbol",    // 整 workspace 找 symbol
  "goToImplementation", // 跳到具体实现 (vs interface)
  "prepareCallHierarchy", // call tree 起点
  "incomingCalls",      // 谁 call 我
  "outgoingCalls",      // 我 call 谁
] as const
```

**业内独家**:Claude Code / Codex / Aider 都没有把这 9 个操作明确作为 agent tool 暴露。最关键的是 `findReferences`(cross-file rename 必备)+ `incomingCalls`/`outgoingCalls`(call tree 分析必备)。

### 2.2 Effect-TS Schema validation

```typescript
[code: tool/lsp.ts:23-35]
export const Parameters = Schema.Struct({
  operation: Schema.Literals(operations).annotate({ description: "..." }),
  filePath: Schema.String.annotate({ description: "..." }),
  line: Schema.Int.check(Schema.isGreaterThanOrEqualTo(1)).annotate({
    description: "The line number (1-based, as shown in editors)",
  }),
  character: Schema.Int.check(Schema.isGreaterThanOrEqualTo(1)).annotate({
    description: "The character offset (1-based, as shown in editors)",
  }),
  query: Schema.optional(Schema.String).annotate({
    description: "Search query for workspaceSymbol. Empty string requests all symbols.",
  }),
})
```

**关键设计**:`line` 和 `character` 用 **1-based(editor 风)**,不用 LSP spec 的 0-based。

```typescript
[code: tool/lsp.ts:64]
const position = { file, line: args.line - 1, character: args.character - 1 }
```

**为什么 1-based**:
- LLM 看 grep 输出 / IDE 截图 都是 1-based("error at line 42, column 5")
- LSP spec 是 0-based("array index 风")
- 让 LLM 用 1-based input + tool 内部转 0-based — **LLM-friendly 而非 LSP-spec-friendly**

**业内对照**:大部分 LSP client SDK 暴露 0-based 给用户,opencode **特意做转换**让 LLM 用着自然。

### 2.3 Effect-TS effect chain 实现

```typescript
[code: tool/lsp.ts:37-113]
export const LspTool = Tool.define(
  "lsp",
  Effect.gen(function* () {
    const lsp = yield* LSP.Service           // 拿 LSP service
    const fs = yield* AppFileSystem.Service  // 拿 fs service
    return {
      description: DESCRIPTION,
      parameters: Parameters,
      execute: (args, ctx) =>
        Effect.gen(function* () {
          // 1. 解析 file path
          const file = path.isAbsolute(args.filePath) 
            ? args.filePath 
            : path.join(instance.directory, args.filePath)
          
          // 2. ⭐ External directory guard
          yield* assertExternalDirectoryEffect(ctx, file)
          
          // 3. ⭐ Permission gate
          yield* ctx.ask({
            permission: "lsp",
            patterns: ["*"],
            always: ["*"],
            metadata: meta,
          })
          
          // 4. ⭐ Pre-check: file exists?
          const exists = yield* fs.existsSafe(file)
          if (!exists) throw new Error(`File not found: ${file}`)
          
          // 5. ⭐ Pre-check: LSP server available for this file type?
          const available = yield* lsp.hasClients(file)
          if (!available) throw new Error("No LSP server available...")
          
          // 6. ⭐ Touch file before query
          yield* lsp.touchFile(file, "document")
          
          // 7. 按 operation 调用对应 LSP method
          const result = yield* (() => {
            switch (args.operation) {
              case "goToDefinition": return lsp.definition(position)
              case "findReferences": return lsp.references(position)
              ...
            }
          })()
          
          // 8. 返回结构化 result
          return {
            title,
            metadata: { result },
            output: result.length === 0 
              ? `No results found for ${args.operation}` 
              : JSON.stringify(result, null, 2),
          }
        }).pipe(Effect.orDie),
    }
  }),
)
```

**8 步流水线**:每步都有明确职责。Effect-TS `function*` generator + `yield*` 让异步逻辑读起来像同步代码,**type-safe error propagation** 借编译器强制。

---

## 3. 5 个反直觉的关键工程细节

### 3.1 `touchFile` before query(`tool/lsp.ts:80`)

```typescript
yield* lsp.touchFile(file, "document")
```

**为什么必须 touch**:LSP server 维护 in-memory document state。如果 agent 刚 edit 了一个文件**但没 save**,LSP server 看不到 unsaved 内容。`touchFile` 把当前 buffer 同步给 server,**确保 query 用的是最新状态**。

**`"document"` vs `"full"`**:
- `"document"` — 只 sync 这一个文件(便宜)
- `"full"` — sync 整个 workspace(贵,初始 init 用)

opencode 默认 `"document"`,因为 LSP query 是 file-specific 的。

### 3.2 80+ 语言 extension mapping(`lsp/language.ts:1-121`)

```typescript
export const LANGUAGE_EXTENSIONS: Record<string, string> = {
  ".ts": "typescript",
  ".tsx": "typescriptreact",
  ".py": "python",
  ".rs": "rust",
  ".go": "go",
  ".java": "java",
  ".kt": "kotlin",
  // ... 80+ 条
  ".astro": "astro",
  ".typ": "typst",     // ⭐ Typst (Pollen 语言)
  ".nix": "nix",       // ⭐ NixOS 配置
  ".zig": "zig",
  ".gleam": "gleam",
  ".hcl": "hcl",
  ".tf": "terraform",
  ...
}
```

**121 行覆盖 80+ 语言**。

业内对照:Cursor 内置 ~30 语言,Aider ~15 语言。opencode 80+ 是因为 **延迟 spawn**(`lsp/lsp.ts` 状态机有 `spawning: Map` 跟踪),只有真正 query 那个文件时才 spawn 对应 LSP server。

### 3.3 Per-file LSP client 而非 per-language

```typescript
[code: lsp/lsp.ts:114-122]
type LocInput = { file: string; line: number; character: number }

interface State {
  clients: LSPClient.Info[]           // ⭐ 多个 client 实例
  servers: Record<string, LSPServer.Info>
  broken: Set<string>                  // ⭐ broken server 黑名单
  spawning: Map<string, Promise<LSPClient.Info | undefined>>
}
```

**3 个特殊 state**:
- `clients` — 一个项目可能 spawn 多个 LSP client(monorepo 不同 subpackage 用不同 tsserver instance)
- `broken` — 启动失败的 server 加黑名单,不重试
- `spawning` — 正在 spawn 的 server 标 promise,**避免重复 spawn 同一个 server**(race protection)

### 3.4 Symbol kind filter(只关心 8 种,`lsp/lsp.ts:90-99`)

```typescript
const kinds = [
  SymbolKind.Class,
  SymbolKind.Function,
  SymbolKind.Method,
  SymbolKind.Interface,
  SymbolKind.Variable,
  SymbolKind.Constant,
  SymbolKind.Struct,
  SymbolKind.Enum,
]
```

LSP spec 定义 26 种 SymbolKind(`lsp/lsp.ts:61-88`),从 File 到 TypeParameter。opencode **只暴露 8 种** _"semantic 重要"_ 的 symbol:类 / 函数 / 方法 / 接口 / 变量 / 常量 / 结构体 / 枚举。

**为什么过滤**:`workspaceSymbol` query 可能返回数千个 symbol(包括每个 `.md` 文件的 heading),agent 不需要这些。过滤减少 LLM context bloat。

### 3.5 Permission gate before LSP call(`tool/lsp.ts:56-61`)

```typescript
yield* ctx.ask({
  permission: "lsp",
  patterns: ["*"],
  always: ["*"],
  metadata: meta,
})
```

**LSP 调用要 permission check**。为什么:
- LSP query 可能触发 server-side execution(某些 server 跑 init script)
- LSP 可能读到 workspace 外的文件(symlink / include path)
- 用户可能想 disable 整个 LSP(privacy concerns)

**`patterns: ["*"], always: ["*"]`** = 默认 always allow,但用户可以在 settings 关掉。

---

## 4. WHY — 为什么这么设计

### 4.1 为什么把 LSP 当 agent tool 而非 background indexer?

两种 model:
- **Model A**(Cursor 风):LSP 在 background 跑,IDE 把 LSP 信息融入 prompt(symbol panel / outline 自动注入)
- **Model B**(opencode 风):LSP 作为 agent **显式 callable tool**,agent 自己决定何时 query

**opencode 选 B 的原因**:
1. **Agent 知道自己在做什么**:大型 refactor 时,agent 知道要 findReferences;简单改字符串时,知道不需要。Background indexer 不智能,injects 多余 context。
2. **Token cost 显式**:Tool call 算 cost,LSP query 结果是 tool result。Background injection 会 _silently_ blow up context window。
3. **Production safety**:Permission gate 在 tool level,model A 没法 gate(已经 injection 了)。
4. **跨 agent 类型可复用**:Plan agent 不需要 LSP(只看 README),Build agent 大量用 LSP。Tool-level access 让不同 agent 按需用。

### 4.2 为什么 1-based?

如前 §2.2 — LLM-friendly。但更深一层:**LLM 出错位置感**。

LLM 看 stack trace 是 1-based("ValueError at line 42"),看 grep 输出是 1-based("file.py:42:def foo()"),写代码注释引用是 1-based("see line 42")。如果 tool input 是 0-based,LLM 经常 _off-by-one_ — 改错位置或问错位置的 symbol。

**1-based input + tool 内部 -1 转换** = LLM 看到 _editor 风_,无需 cognitive overhead。

### 4.3 为什么 touch 而非 init-all?

LSP server 协议有 `initialize` + `didOpen` + `didChange` + `didSave`。
- _Naive 做法_:agent 启动时 didOpen 整个 workspace 所有文件 → 慢(monorepo 几千文件)
- _Lazy 做法_:第一次 query 某文件时 touch(didOpen) → 快,但只覆盖被 query 的文件

opencode 选 lazy。`touchFile(file, "document")` 在每次 query 前 sync。

**Trade-off**:Lazy 让首次 query 慢一点(server 要处理 didOpen),但**总成本低**,因为 agent 不会 query 大部分文件。Monorepo 友好。

### 4.4 为什么有 broken Set + spawning Map?

`State.broken: Set<string>` — 启动失败的 server(比如 pylsp 没装)加黑名单,**避免每次 query 都重试 spawn 失败**。

`State.spawning: Map<string, Promise<...>>` — 正在 spawn 的 server 用 Promise 占位,**多个并发 query 同一语言文件时,只 spawn 一次**(race protection)。

这两个是 production engineering 细节,对小项目无所谓,对 large coding session(几十个文件 + 多语言)关键。

---

## 5. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | opencode 怎么防 | 代码位置 |
|---|---|---|
| **LLM 把 0-based line 误为 1-based 改错位置** | Tool input 1-based + 内部 -1 转换 | `tool/lsp.ts:26-31, 64` |
| **Agent 没 save 就 query,LSP 看不到 unsaved 改动** | `touchFile(file, "document")` 每次 query 前 sync | `tool/lsp.ts:80` |
| **Agent query 不存在的文件** | `fs.existsSafe(file)` precheck | `tool/lsp.ts:74-75` |
| **Agent query 不支持的语言** | `lsp.hasClients(file)` precheck + 明确 error message | `tool/lsp.ts:77-78` |
| **Server spawn 失败反复重试** | `State.broken: Set` 黑名单 | `lsp/lsp.ts:119` |
| **并发 query 同语言文件多次 spawn server** | `State.spawning: Map<string, Promise>` race protection | `lsp/lsp.ts:120` |
| **`workspaceSymbol` 返回千计 symbol 撑爆 context** | `kinds` filter 只保留 8 类 semantic-important symbol | `lsp/lsp.ts:90-99` |
| **Agent 通过 symlink 访问 workspace 外** | `assertExternalDirectoryEffect(ctx, file)` 显式 check | `tool/lsp.ts:49` |
| **用户不想用 LSP(privacy)** | `permission: "lsp"` gate 可被用户 disable | `tool/lsp.ts:56-61` |
| **LSP query 永久 hang** | Effect-TS timeout(`server.ts` 内,未读全)+ `pipe(Effect.orDie)` 错误传播 | `tool/lsp.ts:110` |

---

## 6. 横向校准 — 5 家 coding agent 怎么处理 code semantic

| | Claude Code | Codex | Cursor | Aider | opencode |
|---|---|---|---|---|---|
| **基础工具** | Grep / Read / Glob | Grep / Read / Glob | Grep + IDE selection | Grep + ⭐ tree-sitter AST | ⭐ Grep + LSP 9 ops |
| **Symbol table** | ❌(grep 拼) | ❌ | partial(IDE)| ✅(tree-sitter)| ✅(LSP) |
| **References (call site)** | ❌(grep)| ❌ | ❌ | partial | ✅ `findReferences` |
| **Type info** | ❌ | ❌ | partial(IDE)| ❌ | ✅ `hover` |
| **Definition jump** | ❌ | ❌ | ✅(IDE)| ✅(tree-sitter)| ✅ `goToDefinition` |
| **Call hierarchy** | ❌ | ❌ | ❌ | ❌ | ✅ `incomingCalls`/`outgoingCalls` |
| **Diagnostics** | ❌(run tests)| ❌ | partial(IDE)| ❌ | ✅ `diagnostic.ts` |
| **80+ 语言** | ✅(通过 grep)| ✅ | 30+ | 15+ | ✅(80+) |
| **架构** | tools 内置 | tools 内置 | IDE-coupled | Python lib | ⭐ LSP-as-tool 显式 |

**3 个不同流派**:
1. **Grep-only 流派**(Claude Code / Codex)— 不依赖任何代码语义工具,纯文本搜索 + LLM 自己 reason 出语义
2. **AST 流派**(Aider tree-sitter)— 用 tree-sitter 解析出 AST 给 LLM 看,但**没有 cross-file reference**
3. **LSP 流派**(opencode)— 跟 IDE 用同一套基础设施,**cross-file semantic 完整**

**Trade-off**:
- Grep-only 最简单 + 兼容性最好(不需要 LSP server 装好)
- AST 中等 + 单文件准
- LSP 最强 + 设置成本最高(LSP server 要装,broken 时降级)

opencode 在 _"准确性高,setup 成本可接受"_ 的 niche,适合 production coding agent。

---

## 7. 学习视角 — 写这页让我学到的 3 件事

1. **协议成熟度 = 工程红利**:LSP spec 2016 起 9 年,80+ 语言 server 现成,opencode 113 行 tool 就能用上 — 这是 _"站在巨人肩膀上"_ 的典范。**任何 agent runtime 都该考虑:有没有什么 mature 协议(LSP / DAP / OTel)能直接 wrap 成 tool?**
2. **1-based 还是 0-based 是 LLM-ergonomics 问题**:LSP spec 是 0-based,但 LLM 看的所有 _外部世界_(grep / stack trace / IDE)都是 1-based。Tool input 用 1-based + 内部转 0-based 是 _"为 LLM 翻译外部世界 convention"_ — 跟 [[concept/runtime-context-injection]] §5 memoryAge `"47 days ago"` 比 ISO timestamp 强同理。
3. **State 字段名暴露设计意图**:opencode `State { clients, servers, broken, spawning }` 4 个字段读 5 秒就明白整个 architecture — _"clients 池 + servers 配置 + broken 黑名单 + spawning race protection"_。**好的 struct 设计让 reader 不用看 method**。

---

## 引用链路

- 源码:`/Users/youren/myspace/opencode/packages/opencode/src/`
  - `tool/lsp.ts:1-113` 完整(9 operations + Effect-TS Schema + permission gate + touch-before-query)
  - `lsp/lsp.ts:1-150 (full 506 lines, only first 150 read here)` orchestrator(Schema 定义 + Status + State)
  - `lsp/language.ts:1-121` 完整(80+ 文件扩展名 → 语言)
  - `lsp/server.ts:1-2065`(本次未深读,per-language LSP server config)
  - `lsp/client.ts:1-688`(本次未深读,JSON-RPC LSP client)
  - `lsp/diagnostic.ts:1-29` / `lsp/launch.ts:1-21`(辅助)
- Harness 入口:[[harness/opencode]] §tool_protocol
- 横向 concept:[[concept/lsp-integration]] (7 维设计空间)
- 横向 deep:[[deep/claude-code-autodream-3-gate]] (Claude Code 路线对照 — 无 LSP) / [[deep/codex-actor-resume-agent]] (Codex 路线对照 — 无 LSP)
