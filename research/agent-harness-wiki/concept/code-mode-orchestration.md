---
type: concept
name: code-mode-orchestration
status: draft
sources:
  - "[[../source/codex]]"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/lib.rs"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/description.rs"
  - "/Users/youren/myspace/codex/codex-rs/code-mode/src/runtime/mod.rs"
related_harnesses: [codex]
---

# Code Mode Orchestration — LLM 写代码调工具

> **新范式提示**:本概念目前**只在 Codex 一家**实现。其他 3 家 harness(Hermes/Claude Code/OpenClaw)都是传统 JSON tool_use,无等价物。Codex 是 4 家里唯一首创者。

## 定义

**Code Mode** = LLM 不再用 `tool_use` JSON 块调单个工具,而是**写一段 JavaScript 源码**,沙箱(V8 isolate)执行后回报结果。所有工具暴露成 `tools.<normalized_name>(...)` 全局对象,LLM 用 `await Promise.all(...)` 并行、用 `store()/load()` 跨调用持久化、用 `yield_control()` 流式吐结果。

**传统 tool_use**(其他 3 家 + 业界主流):

```
LLM → {"tool_name": "read_file", "args": {"path": "x.py"}}
框架 → 执行 → {"tool_result": "..."}
LLM → 看到结果 → 下一个 tool_use
```

**Code Mode**(Codex 独家):

```
LLM → "await tools.read_file({path: 'x.py'}); await tools.exec_command(...);"
V8 isolate → 跑 JS → 完成后死亡 → 返回所有 text()/image()/notify() 输出
LLM → 看到完整结果
```

## 设计空间(8 维)

### 1. 编排粒度
- **细粒度**(JSON tool_use,3 家):一次 tool 调用 = 一次 LLM 决策回合
- **粗粒度**(Code Mode):一段 JS 可以包含多个 tool 调用,串行/并行/条件分支由 LLM 写代码控制

### 2. 沙箱语言
- **JavaScript**(Codex):V8 isolate,`rusty_v8` + `deno_core_icudata`(`code-mode/Cargo.toml:22,28`)
- 理论替代:Python(嵌 Pyodide)/ Lua(嵌 mlua)/ Wasm(嵌 wasmtime)/ Java(嵌 GraalVM)

### 3. 工具暴露形式
- **JSON Schema → Function call**(传统):tool_use 块里 args 字段
- **JSON Schema → TypeScript 接口**(Codex `render_json_schema_to_typescript`,`code-mode/src/description.rs`):工具变成有强类型签名的 `tools.<name>` 方法
- **JSON Schema → Python 函数**(理论替代,如 OpenAI o-series 的 code interpreter)

### 4. 跨调用状态
- **传统 tool_use**:每次 tool 结果回到 LLM,LLM 自己记
- **Code Mode**:`store(key, value)` + `load(key)` 是 V8 host 提供的 KV(`code-mode/src/description.rs:28-29`),跨 exec 调用同 session 内持久化

### 5. 流式输出
- **传统 tool_use**:tool 结果整块返回
- **Code Mode**:`yield_control()`(`code-mode/src/description.rs:34`)让 JS 把 accumulated 输出推给模型后**脚本继续跑**,模型可以决定继续等还是 `wait` tool 继续拿

### 6. 取消 / 超时
- Codex 的 pragma `// @exec: {"yield_time_ms": 10000, "max_output_tokens": 1000}`(`description.rs:19-21`)— 单次 exec 内自带超时和输出预算
- 默认值:`DEFAULT_EXEC_YIELD_TIME_MS=10_000` / `DEFAULT_MAX_OUTPUT_TOKENS_PER_EXEC_CALL=10_000`(`runtime/mod.rs:23-25`)
- 长任务用 `cell_id` + `wait` tool 续拿(`runtime/mod.rs:44-48` `WaitRequest { cell_id, yield_time_ms, terminate }`)

### 7. 防失控 / 安全
- V8 自带沙箱(`sandbox = ["v8/v8_enable_sandbox"]`,`code-mode/Cargo.toml:13`)
- 明示"no Node, no fs, no network, no console"(`description.rs:17`)
- isolate 完成后死亡,**未 await 的 promise 静默丢弃**(`description.rs:22`)
- `setTimeout` 不延长 isolate 生命周期(`description.rs:31`)
- 跟外层 OS 级 sandbox(bwrap/seatbelt/landlock)是**两层防御**

### 8. 工具发现
- `ALL_TOOLS` 全局变量暴露 enabled tool 的 metadata(`description.rs:33`)
- LLM 可以 filter `ALL_TOOLS` 找未明示文档化的 nested MCP tool(`description.rs:10-11` "Some nested MCP/app tools may be omitted from this description")

## 各 harness 怎么实现

| 维度 | **Codex Code Mode** | **smolagents CodeAgent** ⭐(2026-05-18 新加 verified) | Hermes / Claude Code / OpenClaw | **opencode tool-level mode**(2026-05-16) |
|---|---|---|---|---|
| 范式 | LLM 写 **JavaScript** | LLM 写 **Python** | LLM 输 JSON tool_use | LLM 调 `plan_exit` 切 mode |
| Sandbox | **V8 isolate**(进程内,工程门槛极高) | **AST interpreter**(`local_python_executor.py:1583` evaluate_python_code)**进程内,业内罕见路线** + 5 远程 executor 选项 | 框架 sandbox(Codex bwrap/Hermes 7 backends/OpenClaw Docker) | 进程级 shell+pty |
| Sandbox 选项数 | 1(V8 only) | **6**(`executor_type: Literal["local","blaxel","e2b","modal","docker","wasm"]`,业内最多) | 单一 | 单一 |
| 编排粒度 | 一段 JS = 多 tool | **一段 Python = 多 tool** | 一回合 = 一 tool | 一回合一 tool + mode 切换 |
| 跨调用状态 | `store/load` KV(JS primitive) | **`state` dict**(`evaluate_python_code:1607` 跨 step 持久化) + `AgentMemory.return_full_code()` 拼接 replay | LLM 自己记 | plan 文件载体 |
| 并行 | `await Promise.all` JS primitive | Python `asyncio` / `concurrent.futures.ThreadPoolExecutor`(verified `agents.py:26`) | 框架支持 | 框架支持 |
| 流式 | `yield_control()` JS primitive | `stream_outputs: bool`(`agents.py:1538`)+ Rich Live | tool_result 整块 | Bus + Effect-TS Stream |
| 中断 | `exit()` JS primitive | **`FinalAnswerException(BaseException)`**(`local_python_executor.py:1572`)**+ `RemotePythonExecutor.FINAL_ANSWER_EXCEPTION` 常量跨边界一致**(elegant Pythonic) | 框架内 cancel | `Question.RejectedError` |
| 安全防御 | V8 sandbox feature flag + 删 4 危险全局(console/Atomics/SharedArrayBuffer/WebAssembly)| **白名单+黑名单+反射防御四道**:`BASE_PYTHON_TOOLS` 40+ allow + `DANGEROUS_MODULES` 9 deny + `DANGEROUS_FUNCTIONS` deny + `nodunder_getattr` 反射防御 + 4 硬限额(MAX_OPERATIONS=10M / WHILE=1M / TIME=30s / OUTPUT=50K) | OS/容器层 | shell 沙箱 |
| 工具发现 | `ALL_TOOLS` 全局 | tool 注入到 Python state 全局命名空间(`evaluate_python_code` static_tools) | 框架 prompt 列 | prompt 列 + mode 切换 |
| 定位 | 工程实现旗手 | **理念旗手**(2024-12 launch blog post _"agents that think in code"_) | 传统 JSON 路线 | 第 3 种工具触发切换 |

**opencode 是 Code Mode 的对立面**:Codex Code Mode 把"工具编排"搬到 LLM 写代码,**opencode 把"agent mode 切换"搬到 LLM 调工具**。
- Code Mode:framework 提供 sandbox,LLM 在 sandbox 写编排代码
- opencode tool-level mode:framework 提供切换工具(`plan_exit`),LLM 调工具触发 framework 切 agent

两条思路都减少 framework hard-code,**把决策上移给 LLM**(Code Mode 上移到代码层,opencode 上移到工具层)。

## Trade-off

### Code Mode 优势
1. **表达力**:JS 完整语言,LLM 能写 `await Promise.all`、`try/catch`、loop、条件
2. **token 节省**:一段 JS 几行 vs 多回合 JSON tool_use 几十个块
3. **延迟降低**:并行 + 跨调用 store,减少 LLM 往返
4. **熟悉度**:JS 是 LLM 训练数据里最多的语言之一

### Code Mode 代价
1. **沙箱成本**:嵌 V8 引入 ~30MB native binary 依赖(`rusty_v8`)
2. **调试难度**:LLM 写的代码出错,栈跟踪要回到 LLM 让它修
3. **可观测性**:trace 现在是 "LLM 写的 JS" 而不是"框架执行的 tool sequence",理解成本高
4. **安全表面**:V8 sandbox 漏洞历来频繁(虽然 isolate 限制了大部分),要紧跟 V8 安全公告
5. **工具协议异质性**:既要 tool_use 兼容(用于不擅长写代码的模型)又要 Code Mode(用于擅长的)— Codex 实际是双轨制

## SkillForge 当前选择 + 演进方向

**当前**:SkillForge 用 Spring/Java DSL 编排 skill,LLM 出 JSON tool_use,Java 端解析并执行 — 跟其他 3 家 harness 一致

**演进选项**:
1. **保持现状**(推荐起步)— Spring DSL + JSON tool_use,稳态
2. **GraalVM JS 嵌入**作为第二编排选项 — JVM 生态嵌 V8 不现实,但 GraalVM JS 是原生 JVM 实现,可以做 POC。优势:无原生依赖,Java 安全模型保留
3. **Pyodide / Python wasm**:LLM 写 Python 编排 — 适合 OpenAI o-series 风格 user
4. **不引入**:Code Mode 调试成本和 trace 可观测性是真问题,SkillForge 当前 trace 工程化未完整,贸然引入会扩大盲区

判定:**先做技术 POC 看 GraalVM 嵌入成本,实际项目落地优先级低于 goal token_budget + 8 hooks 精简 + multi-agent actor 等已验证的借鉴项**。

## 学习视角:为什么 Codex 敢做 Code Mode

OpenAI 的"模型 + 工具"哲学跟 Anthropic 不同:
- **Anthropic**:tool_use 协议是 first-class,模型训练去对齐细粒度 tool_use
- **OpenAI**:模型可以写代码(GPT-4 起 code interpreter 已经在 ChatGPT 跑),Code Mode 是把这个能力扩展到 agent harness 里

这是模型能力差异塑造了 harness 设计 — 学习时要把 "Code Mode 是 OpenAI 独家" 当成 **模型生态的影子**,而不是单纯工程选择。
