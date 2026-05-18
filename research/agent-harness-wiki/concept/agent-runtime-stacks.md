---
type: concept
name: agent-runtime-stacks
status: draft
sources:
  - "[[../source/hermes-agent]]"
  - "[[../source/claude-code]]"
  - "[[../source/openclaw]]"
  - "[[../source/codex]]"
  - "[[../source/opencode]]"
related_harnesses: [hermes-agent, claude-code, openclaw, codex, opencode]
---

# Agent Runtime Stacks — 5 harness 技术栈对照与编程哲学

> 本概念把 5 个 harness 的技术栈选型 + 编程范式 + 工程文化做横向对比,揭示**技术栈选择背后的设计哲学**对 agent 设计本身的影响。

## 5 harness 技术栈速览

| 维度 | [[hermes-agent]] | [[claude-code]] | [[openclaw]] | [[codex]] | [[opencode]] |
|---|---|---|---|---|---|
| **语言** | Python 3.10+ | TypeScript / Bun + Ink | TypeScript / pnpm + pi-sdk | **Rust 17+** | TypeScript / Bun + **Effect-TS** |
| **构建栈** | uv / poetry | Bun(`package.json` + Ink TUI) | pnpm + Turborepo | **Cargo + Bazel + pnpm 三栈** | pnpm + Bun + flake.nix |
| **License** | MIT | 闭源(逆向 v2.1.88) | 闭源(自研) | **Apache 2.0** | **MIT** |
| **包管理** | pip / uv | npm | pnpm | Cargo + npm | npm + Bun |
| **类型系统** | Python type hints | TypeScript + zod | TypeScript + Effect Schema(部分) | **Rust 强类型 + serde** | TypeScript + **Effect-TS Schema(全栈)** |
| **DB / 持久化** | SQLite / 文件 | 文件 + IPC | SQLite | **SQLite + JSONL** | **Drizzle + SQLite(local-first)** |
| **TUI 库** | rich / textual | **Ink (React for CLI)** | (无,主要 multi-channel) | **ratatui** | **自研 + neovim 风格** |
| **出品方** | Nous Research | Anthropic | Mario Zechner / badlogic | **OpenAI** | **Anomaly Corp(SST 团队)** |
| **核心定位** | Self-improving research agent | Anthropic 官方 coding agent | Personal assistant + multi-channel | OpenAI 官方 coding agent | **Open source Claude Code 替代** |
| **编程范式** | OOP(Python class)+ 装饰器 | OOP + Hooks(React 风格) | OOP(TS class) | **Rust ownership + traits + 函数组合** | **Effect-TS functional + Schema-first** |

## 编程哲学 5 流派

### 1. Python OOP 装饰器派(Hermes)

- **风格**:Python class + `@decorator` + `async def` + `dataclass`
- **错误处理**:try/except + Pydantic validation
- **典型代码**:`@delegate_task(...)` 装饰器把 function 变成可委派的 task
- **优势**:Python 生态最熟,LLM training data 最多
- **劣势**:类型系统弱,大 codebase 维护成本高
- **代表工程文化**:Nous Research(RL training friendly,跟 Atropos research codebase 一致)

### 2. React for CLI 派(Claude Code)

- **风格**:Ink(React for CLI)+ TypeScript + Bun + zod runtime validation
- **错误处理**:try/catch + zod validate + custom Error class
- **典型代码**:`useEffect` + Hooks 风格组件 + zod schema
- **优势**:Anthropic 工程化最深,前端 + backend 同栈
- **劣势**:Bun 生态相对新,Ink 调试 TUI 用 React DevTools 不直观
- **代表工程文化**:Anthropic("React-everywhere",27 个 hooks ML-validated)

### 3. TS pnpm + Embedded SDK 派(OpenClaw)

- **风格**:TS + pnpm monorepo + pi-sdk 嵌入式(Mario Zechner / badlogic)
- **错误处理**:try/catch + 自家 Result type
- **典型代码**:多 Cargo workspace 风格 pnpm + 嵌入第三方 SDK 复用
- **优势**:Personal assistant 多 channel(24+ channel)统一栈
- **劣势**:pi-sdk 嵌入耦合度高,升级 SDK 痛苦
- **代表工程文化**:Mario / badlogic(20+ 年 game dev 经验,工程稳健)

### 4. Rust ownership + traits 派(Codex)

- **风格**:Rust 17 + Cargo workspace 100+ crate + traits + 函数组合
- **错误处理**:`Result<T, E>` + `?` operator + `thiserror` enum + 强类型 Domain Error
- **典型代码**:`SessionSource::SubAgent(ThreadSpawn{depth})` enum 表达 depth chain
- **优势**:**4 家工程化最深**(100+ crates 极细拆分,每个 provider/sandbox 独立 crate),性能最高
- **劣势**:Rust 学习曲线陡,LLM training data 比 TS / Python 少
- **代表工程文化**:OpenAI(`AGENTS.md:65-75` "resist adding code to codex-core",反 god crate;模块上限 500 LoC,超 800 LoC 强制拆)

### 5. Effect-TS functional + Schema-first 派(opencode)

- **风格**:Effect-TS + `Effect.gen` + `Service` + `Layer` + `Context` + Schema-first IO
- **错误处理**:**Effect 类型系统替代异常**,`Effect.fail` / `Effect.die`,**编译期可见所有失败路径**
- **典型代码**:
  ```ts
  const PlanExitTool = Tool.define("plan_exit", Effect.gen(function* () {
    const session = yield* Session.Service
    const question = yield* Question.Service
    // ...
    if (answers[0]?.[0] === "No") yield* new Question.RejectedError()
    // ...
  }))
  ```
- **优势**:**类型系统 + Schema 强制 IO boundary**,避免 untrusted JSON / try/catch / nullable / undefined 多类陷阱
- **劣势**:Effect-TS 学习曲线陡,业内 LLM agent 项目少见用,招聘难
- **代表工程文化**:Anomaly Corp(SST 团队,极端 functional;`AGENTS.md` "Avoid try/catch where possible" + "Avoid `any` type" + "Avoid else"早返回 + "Reduce total variable count by inlining")

## 技术栈选择对 agent 设计的影响(8 个维度)

### 1. 错误处理风格 → multi-agent 隔离设计

- Python try/except(Hermes)→ MAX_DEPTH=1 简单隔离,**简单可靠**
- Rust `Result<T, E>`(Codex)→ `CodexErr::AgentLimitReached { max_threads }` 结构化错误,**可被 caller 精确捕获**,适合 actor + resume_agent 复杂模型
- Effect-TS `Effect.fail`(opencode)→ `Question.RejectedError` 用户拒绝时优雅传播,**适合 tool 触发跨 agent 切换**

### 2. 类型系统强度 → schema 表达 agent 实体

- TypeScript zod(Claude Code)→ runtime validation,27 hooks zod-validated
- Rust serde + enum(Codex)→ 编译期保证(`SessionSource::SubAgent(ThreadSpawn{depth})`)
- **Effect-TS Schema(opencode)→ 比 zod 更严**:`Schema.Literals(["subagent", "primary", "all"])` 编译期保证 `mode` 字段只能 3 值
- Python type hints(Hermes)→ runtime 弱

### 3. 异步 / 并发 → agent loop 实现

- Python `async def`(Hermes)→ asyncio loop
- TS Bun(Claude Code / opencode)→ async / await + 微任务队列
- Rust tokio(Codex)→ runtime + Future + Stream
- **Effect-TS(opencode)→ Effect runtime,可中断 / 可恢复 / 可组合**,跟 [[../concept/multi-agent-isolation]] 中 actor + resume 模型契合

### 4. 模块化 → service 拆分粒度

- Python imports(Hermes)→ 单 repo,功能在 modules
- npm workspace(Claude Code)→ 几个 package
- pnpm + pi-sdk(OpenClaw)→ extensions/* + apps/{android,ios,macos,shared} 4 native + 30+ extensions
- **Cargo workspace(Codex)→ 100+ crate 极细拆分**(每个 provider / sandbox / sdk 独立 crate)
- pnpm + Bun(opencode)→ 22 packages(opencode + core + tui + desktop + web + llm + plugin + sdk + ...)

### 5. TUI 选择 → user experience

- Hermes:rich / textual(成熟 Python TUI)
- **Claude Code:Ink(React for CLI)**— 业内独家
- Codex:**ratatui**(Rust 主流 TUI 库)
- opencode:**自研 TUI**(neovim 风格,terminal.shop 团队优化)
- 趋势:**TUI 是 LLM agent 主战场**(Anthropic / OpenAI / Anomaly 都重投 TUI)

### 6. 持久化选择 → trace / memory 模型

- Hermes:SQLite + 文件
- Claude Code:文件 + IPC
- OpenClaw:SQLite(memory wiki)
- **Codex:SQLite + JSONL append-only**(rollout/recorder.rs 完整实现)
- **opencode:Drizzle ORM + SQLite local-first**(`session.sql.ts`,snake_case schema 强约束)

**共性**:**5 家都用 SQLite,没有用 Postgres / MySQL**(都是 local-first / single-user,跟 [[../eval/langfuse]] / [[../eval/opik]] 这些 server-side platform 选 ClickHouse + 关系库截然不同)

### 7. 构建复杂度 → 团队协作成本

- Python uv:简单
- npm / Bun:中等
- pnpm + Turborepo:复杂(monorepo)
- **Cargo + Bazel + pnpm 三栈(Codex):最复杂**
- pnpm + Bun + flake.nix(opencode):中复杂(Nix 反向加成)

### 8. 工程文化 → AGENTS.md 写法

- **Hermes**:无显式 AGENTS.md(Python 风格)
- **Claude Code**:有 CLAUDE.md(用户 facing)
- **OpenClaw**:`AGENTS.md` + `extensions/*/AGENTS.md` 模块级
- **Codex**:`AGENTS.md` 17KB(LLM-readable contributor guide,反 god crate / 模块上限 500 LoC)
- **opencode**:`AGENTS.md` 4KB **极简 functional style guide**(避 try/catch / 早返回 / inline 变量 / Avoid else)

**模式**:**AGENTS.md 写得越详细的项目,LLM agent friendly 程度越高**(Codex / opencode 都有 LLM-readable)。

## Trade-off

### Python vs TypeScript vs Rust vs Effect-TS

| 选 | 适合 | 不适合 |
|---|---|---|
| **Python**(Hermes) | RL training / research / data science 集成 | 大型 production / 类型严格场景 |
| **TypeScript Bun**(Claude Code / opencode) | TUI / Web / Desktop 跨形态 / npm 生态 | 极致性能 / WASM 集成 |
| **Rust**(Codex) | 高性能 / 跨平台 / 强类型保证 | 快速 prototype / LLM training data 少 |
| **Effect-TS**(opencode) | Schema-first IO / functional 团队 / 强 boundary | OOP / Java 团队 / 招聘难 |

### `.agents/` 完整度跟工程文化

| Harness | `.agents/` 完整度 |
|---|---|
| **Opik** | 6 agents + 10 skills + 4 .mdc rules + commands + mcp + docs(最完整) |
| **opencode** | `.opencode/` 自家 dogfood:agent + command + skills + plugins + tool + themes + opencode.jsonc + tui.json |
| **Langfuse** | 16 skills + ARCHITECTURE_PRINCIPLES.md |
| **Codex** | AGENTS.md 17KB + `.codex/` 配置 |
| **Claude Code** | `.claude/{agents,commands,rules,skills,settings.json}/` |
| **Hermes / OpenClaw** | (无显式 .agents/ 目录) |

**结论**:**`.agents/` dogfood 程度跟项目对 LLM agent 友好程度强相关**。Opik / opencode 走得最深。

## SkillForge 当前选择 + 演进方向

**当前**:Java + Spring Boot + Maven + Postgres + React Frontend

**对照 5 家技术栈,SkillForge 是第 6 种路线**:**Java OOP + Spring 企业级**

**演进选项**:
1. **维持 Java/Spring 主栈**(默认)— 企业级生态成熟
2. **借鉴 functional 哲学**:虽然不能用 Effect-TS,但 SkillForge Java 代码可学:
   - 避免 `try/catch` 滥用 — 用 `Optional` / `Result` 类
   - 早返回(early return)替 `else`
   - inline 单次使用变量(opencode AGENTS.md 风格)
   - Schema-first IO(Spring `@Validated` + Jackson)
3. **借鉴 `.agents/` dogfood**:SkillForge 仓库应建 `.agents/` 目录,内置 agent + skill + mcp
4. **AGENTS.md LLM-readable**:写一份 SkillForge 自己的 AGENTS.md,LLM agent 工具(Claude Code / Codex / Cursor)都能读
5. **`.codex` + `.cursor` symlink to `.agents/`**:多 LLM 工具普适性(opencode + Opik 模式)

## 学习视角:技术栈是哲学的载体

### 5 种哲学的 1 句话总结

- **Hermes**:"Python class + @decorator + RL friendly,let researchers iterate"
- **Claude Code**:"React for CLI,Anthropic 全栈 TS,zod runtime safety"
- **OpenClaw**:"pi-sdk 嵌入式 + 24+ channel multi-platform,personal assistant"
- **Codex**:"Rust 100+ crate 极细拆分,反 god crate,工程化标杆"
- **opencode**:"Effect-TS Schema-first functional,client/server,LSP first-class,open source Claude Code 替代"

### 哲学如何影响 multi-agent 设计

- Python(Hermes)→ MAX_DEPTH=1 简单隔离
- React Hooks(Claude Code)→ 27 hooks + Fork 共享 cache
- TS pi-sdk(OpenClaw)→ 24+ channel 一致 agent runtime
- Rust enum(Codex)→ actor mailbox + control + resume_agent
- Effect-TS(opencode)→ tool 触发跨 agent + Question.ask 用户确认

**没有最好的技术栈,只有最匹配工程文化和团队的技术栈**。SkillForge Java/Spring 选 OOP + 企业级路线,**不必盲目跟 Rust / Effect-TS,但应吸收哲学(functional / Schema-first / LLM-readable AGENTS.md / dogfood `.agents/`)**。

### 对 SkillForge 的具体启示

| 借鉴点 | 来源 | 实现方式 |
|---|---|---|
| AGENTS.md LLM-readable | Codex 17KB + opencode 4KB | 写一份 SkillForge AGENTS.md |
| `.agents/` dogfood | Opik / opencode | SkillForge 仓库建 `.agents/{agents,skills,rules,commands,mcp.json}/` |
| Schema-first IO | opencode Effect-TS | Spring `@Validated` + Jackson + 自家 ResultType |
| 反 god module | Codex `core` 反 god crate | SkillForge core 模块上限,新功能优先新模块 |
| 早返回 + 避 else | opencode AGENTS.md | Java 代码 review 标准 |
| `.codex` + `.cursor` symlink | opencode + Opik | 多 LLM 工具普适性 |
| 22 国语言 README | opencode 22 国 | SkillForge 走开源后本地化 |
