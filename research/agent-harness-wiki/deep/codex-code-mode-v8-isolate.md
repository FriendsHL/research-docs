---
type: deep
name: codex-code-mode-v8-isolate
status: B-mode source-grounded
harness: codex
mechanism: V8 isolate + deno_core_icudata + 8 JS primitives (tools/text/image/store/load/notify/yield_control/exit) + delete dangerous globals + nested tool call via async pause/resume
updated: 2026-05-17
sources:
  - "[[harness/codex]]"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/Cargo.toml:1-30"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/lib.rs:1-38"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/runtime/mod.rs:1-578"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/runtime/globals.rs:1-157"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/runtime/callbacks.rs:1-271"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/service.rs:1-1507"
---

# Codex Code Mode — LLM 不写 JSON tool_use,LLM 写 JavaScript 在 V8 isolate 跑

> 要点:Codex `code-mode` crate(9 文件 4251 行 Rust)实现了**业内最激进的 tool 协议范式**:LLM 不输出 `{"tool": "read_file", "args": {...}}`,而是输出**真 JavaScript 代码**,通过 V8 isolate(`v8` + `deno_core_icudata` crate)执行。代码内调用 `tools.read_file(path)` / `await tools.bash(cmd)` / `text(...)` / `store(...)` 等 8 个 primitive。
>
> 这页深挖 3 个反直觉工程决策:**(1) V8 isolate 启动时显式 `delete console / Atomics / SharedArrayBuffer / WebAssembly` 4 个危险全局**;**(2) JS 代码内 `await tools.X(...)` 是 pause cell + 异步调 real tool + resume cell 的非平凡机制**;**(3) `store/load` primitive 让 cell 间传值,LLM 不再需要每次 JSON marshalling**。
>
> 业内对照:Claude Code / Aider / Cursor 都是 JSON tool_use,**只有 Codex 把 V8 isolate 直接接进 agent runtime**。这是 OpenAI 试图**改 agent 协议层的根本性 bet**。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通 agent runtime 跑 tool 用法:

```json
// LLM 输出:
{"tool_use": [{"name": "read_file", "args": {"path": "X"}}]}

// 框架解析 → 调用 → 返回:
{"tool_result": "<file contents>"}

// LLM 再输出:
{"tool_use": [{"name": "read_file", "args": {"path": "Y"}}]}
```

**问题**:LLM 要读 10 个文件做对比,需要 **10 个 turn**。每个 turn 都重新 sample(token cost 10x),且 LLM 看到全部 10 个文件内容后才能 reason。

Codex Code Mode 用法:

```javascript
// LLM 输出 (单个 turn):
const files = ["a.py", "b.py", "c.py", "d.py", "e.py", "f.py", "g.py", "h.py", "i.py", "j.py"]
const contents = await Promise.all(files.map(f => tools.read_file(f)))
for (let i = 0; i < contents.length; i++) {
  if (contents[i].includes("TODO")) {
    text(`Found TODO in ${files[i]}`)
    store(`todo_${files[i]}`, contents[i])
  }
}
exit()
```

**1 个 turn 跑完 10 个 read + 过滤 + store**。**10x token 节省**。

为什么这个机制容易被忽略:
1. README 不会展开讲(只说 _"OpenAI 官方 agent CLI"_)
2. 跟 Anthropic / OpenAI agent demo 完全不同范式,大部分 reader 默认 _"JSON tool_use 是 universal"_
3. 实现复杂(需要 V8 sandbox + async pause/resume 机制 + dangerous globals delete + nested tool call routing)
4. 当前还是 dev 状态(Codex 0.0.0-dev),没大规模产品化

**对 agent 协议层设计的启示**:**JSON tool_use 不是 universal,只是 _Anthropic claude-3 训练假设_**。如果 model 训练时见过 _"写 JavaScript 调 tool"_ 的样本,这个范式可能比 JSON 更高效。Codex 这是在 _"训练 + runtime 共同投入"_ 押宝下一代协议。

---

## 2. 机制 — V8 isolate + 8 JS primitive

### 2.1 公共 API:`exec` + `wait` 两个 tool

`lib.rs:36-37` 整个 crate 对外只暴露 2 个 tool:

```rust
pub const PUBLIC_TOOL_NAME: &str = "exec";
pub const WAIT_TOOL_NAME: &str = "wait";
```

LLM 调用 `exec({source: "<JS code>"})` 启动一个 cell。如果 cell 内 `await tools.X(...)` 触发 nested tool call,host 处理后 cell 自动 resume。如果 cell `yield_control()` 主动让出,host 通过 `wait({cell_id})` 恢复。

### 2.2 V8 环境删除 4 个危险全局

```rust
[code: runtime/globals.rs:13-19]
pub(super) fn install_globals(scope: &mut v8::PinScope<'_, '_>) -> Result<(), String> {
    let global = scope.get_current_context().global(scope);
    delete_global(scope, global, "console")?;           // ⭐ stdout 通道(防 leak)
    delete_global(scope, global, "Atomics")?;            // ⭐ 并发原语(防 race)
    delete_global(scope, global, "SharedArrayBuffer")?;  // ⭐ 共享内存(防 timing attack)
    delete_global(scope, global, "WebAssembly")?;        // ⭐ 原生执行(防 sandbox escape)
    ...
}
```

**4 个 delete 是 production security 必备**:

| 删除项 | 防什么 |
|---|---|
| `console` | 让 LLM 不能用 `console.log` 私下输出(强制走 `text(...)` / `image(...)` 走 host 通道)|
| `Atomics` | 多线程并发原语,sandbox 内不需要 + 历史安全漏洞 source |
| `SharedArrayBuffer` | Spectre/Meltdown timing attack 入口 |
| `WebAssembly` | 允许加载原生编译代码 → sandbox escape 风险 |

加上 `Cargo.toml:14-15`:
```toml
[features]
sandbox = ["v8/v8_enable_sandbox"]
```

V8 自家 sandbox feature flag(隔离 heap)— 双层安全。

### 2.3 8 JS primitive(`globals.rs:22-42`)

```javascript
// LLM 在 cell 内可用的 8 个全局函数/对象:
tools           // Object — 每个 enabled tool 是 tools.X(input) → Promise<result>
ALL_TOOLS       // Array — [{name, description}, ...] 让 LLM 枚举可用 tool
text(content)   // → 输出 text content item 给 host
image(...)      // → 输出 image content item
store(key, val) // → 持久化变量到 cell stored_values
load(key)       // → 从 stored_values 读
notify(msg)     // → 推消息给 host(不终止 cell)
yield_control() // → 主动让出 + 标 Yielded outcome
exit()          // → 终止 cell + 标 Result outcome
setTimeout/clearTimeout  // 标准 timer
```

**精心设计的最小 API surface**:LLM 不需要熟悉 Deno / Node API,**8 个 primitive + 标准 JS**(数组 / 对象 / Promise / async/await / 控制流)足够表达大部分 agent 行为。

`globals.rs:32` 还有 `tools` Object 和 `ALL_TOOLS` Array 双暴露 — 前者 callable,后者 introspectable(`for (const t of ALL_TOOLS) {...}`)。

### 2.4 Tool call routing — Nested async pause/resume

```rust
[code: runtime/mod.rs:124-146]
pub struct CodeModeNestedToolCall {
    pub cell_id: String,
    pub runtime_tool_call_id: String,
    pub tool_name: ToolName,
    pub tool_kind: CodeModeToolKind,
    pub input: Option<JsonValue>,
}

pub(crate) enum TurnMessage {
    ToolCall(CodeModeNestedToolCall),
    Notify { cell_id, call_id, text },
}

pub(crate) enum RuntimeCommand {
    ToolResponse { id: String, result: JsonValue },
    ...
}
```

**完整 flow**:
```
1. LLM 输出 JS: const x = await tools.read_file("a.py")
2. exec(source) 启 V8 isolate, 跑 JS
3. JS 跑到 await tools.read_file(...) 时:
   - tool_callback (callbacks.rs) 被调
   - 创建 CodeModeNestedToolCall { cell_id, runtime_tool_call_id, tool_name: "read_file", input }
   - 通过 mpsc 发给 host (TurnMessage::ToolCall)
   - JS 端 await Promise pending → cell stalls
4. Host (CodeModeTurnHost) 收 ToolCall,
   - 实际调 read_file (file system / mcp / etc)
   - 把结果发回 (RuntimeCommand::ToolResponse { id, result })
5. Runtime resolve await 的 Promise with result
6. JS 继续跑 (x 拿到值)
```

**异步 pause/resume 是核心 magic**:LLM 写 _"看起来同步"_ 的 `await tools.X(...)`,实际跑在 V8 + Rust async runtime 协作的真异步。

### 2.5 store/load — Cell 间持久化

```javascript
// Cell A:
const data = await tools.read_file("a.py")
const parsed = parseAST(data)
store("a_ast", parsed)
exit()

// Cell B (后一个 turn):
const a_ast = load("a_ast")  // 跨 cell 拿到
// 用 a_ast 做事
```

`store(key, val)` 把 JSON-serializable value 存进 cell 的 `stored_values: HashMap<String, JsonValue>`,`load(key)` 跨 cell 取。**LLM 不需要把中间状态序列化进 prompt context** — 这是 Code Mode 比 JSON tool_use 节省 token 的另一关键。

### 2.6 yield_control 与 wait 配合

```javascript
// Cell A:
const url = "https://long-running.com/job"
const jobId = await tools.start_job(url)
yield_control()  // ⭐ 让 host 处理别的事,不阻塞
```

```
Host 看到 Yielded → 把 cell_id 加 watch list
Host 之后 wait(cell_id) → cell 恢复 → 检查 job 状态 → 拿结果
```

**yield_control + wait 是 cooperative multitasking**。Cell 不阻塞 main loop,主 agent 可以在等待间穿插其他 tool call(类似 Erlang scheduler)。

---

## 3. WHY — 为什么这么设计

### 3.1 为什么用 V8 而非 Python / WASM?

3 个考虑:
- **V8 是已知最强 sandbox 之一**(Chrome 每天对 V8 做 fuzzing,security 投入巨大)
- **JavaScript 是 LLM 训练样本最多的语言之一**(GitHub 上 JS 代码量 Top 1,LLM 写 JS 流畅)
- **JS async/await 是天然 nested tool call model**(Python 也有但 LLM 不熟,WASM 没有标准 JS-like async)

Codex 选 V8 不是因为 _"喜欢 JS"_,是 _"安全 + LLM 熟悉度 + async 模型"_ 三者交集。

### 3.2 为什么删 console / Atomics / SharedArrayBuffer / WebAssembly?

如前 §2.2 — production security。但更深层:**强制所有 output 走 host 通道**(`text` / `image` / `notify`)→ host 可以 audit / log / rate-limit。如果留 console,LLM 可以 `console.log` 偷渡数据,host 看不到。

`Atomics` / `SharedArrayBuffer` / `WebAssembly` 是 _"虽然没明显恶意但增加 attack surface"_。Production sandbox 原则:**默认 deny,需要时才 enable**。

### 3.3 为什么 `store/load` 用 string key 而非自动作用域?

```javascript
store("user_pref", {...})  // 显式 key
load("user_pref")
```

vs `var x = ...` 自动跨 cell 持久化?

**显式 store/load**:
- LLM 必须明确意图("我想跨 cell 保留这个")
- Host 可以 audit / inspect / clear `stored_values`
- 跨 cell 的依赖图明确(grep store/load 找出)
- 避免 LLM 误用全局变量导致难调试 state pollution

**自动持久化**:
- LLM 友好(更像普通编程)
- 但 host 难管理生命周期
- 难追踪跨 cell 数据流

Codex 选显式 — Pythonic _"explicit > implicit"_。

### 3.4 为什么 yield_control + wait 而非 generator/coroutine?

JS 有 generator(`function*` + `yield`)和 async generator,理论上可以直接用。Codex 用更原始的 `yield_control` + `wait` 因为:
- Host 跨语言(Rust)看不到 JS generator 内部状态
- `yield_control` 显式标记 _"我要让出"_,host 加 cell 进 watch list
- `wait(cell_id)` host 主动唤醒,resume 流程清楚

JS 端 `yield_control` 是普通 await `Promise`,内部 callback resolve 触发 resume。复杂度藏在 Rust 端,JS 端简单。

---

## 4. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | Code Mode 怎么防 | 代码位置 |
|---|---|---|
| **LLM 输出恶意 JS 跑 process.exit(0) 杀进程** | V8 isolate,无 Node API + sandbox feature flag | `Cargo.toml:14-15` |
| **LLM 用 console.log 私下输出 sensitive data 不走 audit** | `delete console` 全局 | `globals.rs:15` |
| **LLM 用 Atomics 触发 race condition 或 deadlock** | `delete Atomics` | `globals.rs:16` |
| **LLM 用 SharedArrayBuffer 做 Spectre timing attack** | `delete SharedArrayBuffer` | `globals.rs:17` |
| **LLM 加载 WebAssembly 跑原生代码 escape sandbox** | `delete WebAssembly` | `globals.rs:18` |
| **LLM JS 死循环卡住 main loop** | `DEFAULT_EXEC_YIELD_TIME_MS = 10_000` 强制 10s yield + `DEFAULT_MAX_OUTPUT_TOKENS_PER_EXEC_CALL = 10_000` 输出 cap | `runtime/mod.rs:23-25` |
| **LLM 输出无限 text(...) 撑爆 context** | `max_output_tokens` 显式限制,超了 truncate | `runtime/mod.rs:25,40` |
| **LLM 写错忘了 exit() cell 永远 live** | `EXIT_SENTINEL = "__codex_code_mode_exit__"` + 超时 yield 触发 terminate | `runtime/mod.rs:26` |
| **JS 异常没 catch 卡住 V8 isolate** | `RuntimeResponse::Result { error_text: Some(...) }` 显式 error propagate | `runtime/mod.rs:116-121` |
| **Nested tool call 的 id 跟 host 的全局 id 冲突** | `CodeModeNestedToolCall { runtime_tool_call_id }` host 仍可 assign 自家 globally unique id(`mod.rs:124-136` 注释 _"Code mode owns the per-cell runtime id. Hosts should preserve it for provenance/debugging, but should still assign their own runtime tool call id if their tool-call graph requires globally unique ids."_)| `runtime/mod.rs:124-136` |
| **Cell 异步状态丢失导致 wait 永远 hang** | `WaitOutcome::{LiveCell, MissingCell}` 显式区分,MissingCell 立刻返回 | `runtime/mod.rs:62-68` |

---

## 5. 横向校准 — 4 家 agent runtime tool 协议对照

| | Codex Code Mode | Claude Code / Anthropic | OpenAI Function Calling | LangChain ReAct |
|---|---|---|---|---|
| **协议** | ⭐ **LLM 写 JS,V8 isolate 跑** | JSON tool_use (Anthropic schema) | JSON function_call (OpenAI schema) | ReAct text format (Thought/Action/Observation) |
| **多 tool call** | 1 turn 内任意多个(JS 循环 / Promise.all) | 1 turn 多个 tool_use array | 1 turn 多个 function_call | 1 turn 1 个 (ReAct loop) |
| **控制流(if/loop)** | ⭐ JS 原生 | LLM 在多 turn 间 reason | 同 | 同 |
| **中间状态持久化** | ⭐ `store/load` primitive | tool_result in context | 同 | 同(intermediate steps in scratchpad) |
| **Async pause/resume** | ⭐ `yield_control` + `wait` | 无 (每 turn 同步完成) | 无 | 无 |
| **安全** | V8 sandbox + 4 globals delete + sandbox feature flag | tool layer permission check | OpenAI 服务端 (Codex 自家 sandbox) | tool 自家 implementation |
| **业内成熟度** | dev (0.0.0-dev 状态) | production (Claude 3+) | production | production |
| **训练数据需求** | ⭐ **需要 LLM 训练时见过 "写 JS 调 tool" 样本** | 训练已 cover JSON tool_use | 同 | 同(自然语言 ReAct format)|

**核心洞察**:Code Mode **是从 protocol 层重新定义 agent 行为**。其他 4 家是 _"LLM 输出结构化数据,framework 解析执行"_,Code Mode 是 _"LLM 写完整 program,runtime 执行"_。

**Trade-off**:
- ✅ Token 效率(N 个 tool call 折成 1 个 JS file 1 个 turn)
- ✅ 控制流强(LLM 不需要靠多 turn 表达 if/loop)
- ✅ 中间状态 first-class(store/load)
- ❌ 安全复杂度高(必须 sandbox 严格)
- ❌ 训练数据要求(model 没见过这种用法可能差)
- ❌ Debug 难(JS error 需要解栈,host 看 cell 状态需要新工具)

**业内对 Code Mode 路线的接受度**:目前低。Codex 自家也是 dev 状态。**这是 _"未来 5 年才能验证"_ 的 bet**,不是当下 default。

---

## 6. 学习视角 — 写这页让我学到的 3 件事

1. **协议层的革命比 framework 优化重要**:Code Mode 不是 "更好的 framework",是 _"不同的 LLM-runtime contract"_。如果 OpenAI 训练时大量加 _"写 JS 调 tool"_ 样本,这个范式可能是下一代 default。**协议层 leverage > framework 层优化**。
2. **`delete console / Atomics / SharedArrayBuffer / WebAssembly` 是 production sandbox checklist**:任何 _"让 LLM 跑代码"_ 的系统都该 audit 这 4 个删除 + sandbox feature flag。简单但**容易忘**。
3. **`store/load` 显式 > 隐式作用域**:LLM 编程时,**强制显式 _"我要保留这个"_ 比自动持久化更好**(audit / lifecycle / debug 都受益)。跟 [[deep/codex-actor-resume-agent]] §5.3 RAII 强制 cleanup 同思路:**编程语言 / API 设计 _强制好习惯_ 比 _鼓励好习惯_ 强一个数量级**。

---

## 引用链路

- 源码:`/Users/youren/myspace/codex/codex-rs/code-mode/`
  - `Cargo.toml:1-30` 完整(v8 + deno_core_icudata + tokio + sandbox feature)
  - `src/lib.rs:1-38` 完整(public API:exec + wait tool name)
  - `src/runtime/mod.rs:1-578` 关键(ExecuteRequest / WaitRequest / RuntimeResponse / CodeModeNestedToolCall / TurnMessage / RuntimeCommand)
  - `src/runtime/globals.rs:1-157` 完整(4 globals delete + 8 JS primitive install + tools Object + ALL_TOOLS Array build)
  - `src/runtime/callbacks.rs:1-271`(本次未深读,8 primitive 的 V8 callback implementation)
  - `src/service.rs:1-1507`(本次未深读,CodeModeService / CodeModeTurnHost / CodeModeTurnWorker host 端协议)
- Harness 入口:[[harness/codex]] §tool_protocol
- 横向 concept:[[concept/code-mode-orchestration]] (8 维 paradigm 设计空间)
- 横向 deep:[[deep/codex-actor-resume-agent]] (Codex actor model 互补 — Code Mode 用 V8 跑 JS,Actor 跑多 agent)
