---
type: concept
name: multi-agent-isolation
status: draft
verification: source-grounded
updated: 2026-05-16
sources:
  - "[[../source/hermes-agent]]"
  - "[[../source/claude-code]]"
  - "[[../source/openclaw]]"
  - "[[../source/codex]]"
  - "[[../source/opencode]]"
related_harnesses:
  - "[[../harness/hermes-agent]]"
  - "[[../harness/claude-code]]"
  - "[[../harness/openclaw]]"
  - "[[../harness/codex]]"
  - "[[../harness/opencode]]"
---

# Multi-Agent Isolation(多 Agent 隔离 vs 继承)

## 定义

当一个 agent(parent)派生子 agent(child)时,**child 跟 parent 之间的上下文边界**怎么设计?

两个极端:
- **完全隔离**:child 拿到只有 task prompt + tools,什么都不知道 parent 的状态、历史、memory
- **完全继承**:child 拿到 parent 的全部 system prompt + 对话历史 + memory + cache

两个极端之间还有各种中间态(继承 system prompt 但不继承历史 / 继承 cache 但隔离 memory / etc)。

这个 trade-off 决定了多 agent 系统的**性能、安全、可调试性、context 污染风险**。

---

## 设计空间(5 维)

| 维度 | 隔离极端 | 继承极端 |
|---|---|---|
| **System prompt** | child 独立 prompt(常面向 task) | child 继承 parent system prompt |
| **对话历史** | child 起空白 messages list | child 看到 parent 完整 transcript |
| **Memory access** | child 无 parent memory provider | child 跟 parent 共享 memory state |
| **Prompt cache** | child cold start(贵) | child 共享 cache(便宜) |
| **Tool 集合** | child 工具按 task 重定义(常受限) | child 继承 parent 工具集合 |

---

## 各 harness 怎么实现

### [[../harness/hermes-agent]] — **完全隔离派**

`tools/delegate_tool.py:128`: `MAX_DEPTH = 1` — flat by default (parent=0 → child=1, grandchild rejected)
`tools/delegate_tool.py:1047`: child `skip_memory=True` — 无 parent memory provider 访问
`tools/delegate_tool.py:42`: child `delegate_task` disabled — 不能再 delegate

**所有 5 个维度都是隔离极端**:
- System prompt: child 独立(face task)
- 对话历史: child 起空白(只看到 task prompt)
- Memory: skip_memory=True
- Prompt cache: child cold start(独立 system prompt = 独立 cache prefix)
- Tools: child 继承 toolset 但 budget 独立(`delegation.max_iterations=50` default)

**父只看到 child 的 task prompt + final response** observation,中间过程不可见。

**为什么这么选**:single-tenant trust model 下,subagent 是"功能子任务",不是"协作助手",隔离最简单。

### [[../harness/claude-code]] — **两种都支持**

#### 普通 SubAgent(`tools/AgentTool/` 不含 fork):**部分隔离**
- System prompt: independent(用户可定义 agent type via `loadAgentsDir.ts`)
- 对话历史: child 起空白
- Memory: 看 agent 配置,但**可 resume**(`resumeAgent.ts`)— 隔离里带"持久化"
- Prompt cache: cold start
- Tools: 按 agent type 配置

#### **FORK_SUBAGENT**(`tools/AgentTool/forkSubagent.ts:42`):**几乎完全继承**
- `FORK_SUBAGENT_TYPE = 'fork'`,`feature('FORK_SUBAGENT')` Bun gate
- System prompt: **child 继承 PARENT 的**(`AgentTool.tsx:485`:"Fork path: child inherits the PARENT's system prompt")
- 对话历史: parent context 全部带过去
- Memory: 共享 parent memory
- Prompt cache: **shares parent cache**(`prompt.ts:86`)
- Tools: 继承 parent 工具集合

`prompt.ts:95` 关键提示:
> Writing a fork prompt. Since the fork inherits your context, the prompt is a **directive** — what to do, not what the situation is. Be specific about scope: what's in, what's out, what another agent is handling. Don't re-explain background.

适用场景(`prompt.ts:86`):
> **Research**: fork open-ended questions. If research can be broken into independent questions, launch parallel forks in one message. A fork beats a fresh subagent for this — it inherits context and shares your cache.

#### Coordinator Mode(`coordinator/coordinatorMode.ts:36`):**网状,workers 之间隔离**
- Workers async + parallel(`:213`)
- Workers 之间无共享 context — 每个 worker 独立 task
- Coordinator 协调通过 `<task-notification>` XML(`:145-156`)收集结果
- 类似于"Coordinator 是 Hermes 模式 × N workers,每个 worker 仍然隔离"

> **校准:Coordinator ≠ A2A 协议** — CC `<task-notification>` XML 是 in-process 私有协议,跟跨进程 / 跨组织的 A2A JSON-RPC 是不同问题域。详见 [[agent-protocols]] §10.6 对照表(信任模型 + 部署边界 + Task 状态机三个维度都不同)。

#### Task 系统(`Task.ts:6-13`):**特殊 task types 各自隔离策略**
5 task types:LocalAgentTask / RemoteAgentTask / LocalShellTask / InProcessTeammateTask / **DreamTask**
DreamTask 走 fork(`services/autoDream/autoDream.ts` 调 `runForkedAgent`)

### [[../harness/openclaw]](Hermes 前身,源码 verified)

**Subagent 用途有限**,主要在两个特定场景:
- **Memory Dreaming `dreaming-narrative.ts`**:晋升完成后调用 subagent 写 DREAMS.md 梦境日记 — **隔离 + 单一职责**(只写 80-180 词散文,反元评论 prompt)
- **QA Character Eval `character-eval.ts`**:对每个候选模型串行调用 `runQaSuite` 跑场景 — **隔离 + 真实 Gateway 子进程**(`startQaGatewayChild()` IPC 通信)

**OpenClaw 设计哲学**:跟 Hermes 一样**完全隔离**(没有 Fork 共享 context)。理由可能是:
- pi-coding-agent 底层不自带 Fork 能力
- Personal assistant 场景多 agent 协作需求弱(主要是 Dreaming 写日记和 QA Eval 跑测试,都是独立任务)
- Multi-channel 路由更重要(24+ channels 共享同一 agent),而不是 multi-agent 协作

**ACP 协议**(`extensions/acpx`,Agent Communication Protocol,笔记 06):跨 agent 通信但**未深读源码**,可能是网状协作能力的占位但未充分展开。

### [[../harness/codex]] — **第三条路:Actor 模型 + ConfigLayer 层叠**

Codex 的多 agent 模型既不是 Hermes 的"完全隔离 + fire-and-forget",也不是 Claude Code 的"隔离 vs Fork 二选一",而是**actor + 可恢复**:

- **核心组件**(`core/src/agent/mod.rs:1-15`):
  - `mailbox.rs` — `Mailbox` + `MailboxReceiver`,消息队列
  - `control.rs` — `AgentControl` 状态机
  - `registry.rs` — `AgentRegistry::reserve_spawn_slot(max_threads)`,超限抛 `CodexErr::AgentLimitReached`
  - `role.rs` — Role 用 **`ConfigLayerStack` 层叠**(`role.rs:40-80`),不是 system prompt 注入,preserve caller 当前 profile/provider 不静默 fallback

- **关键常量**:`DEFAULT_AGENT_MAX_THREADS = Some(6)`(`core/src/config/mod.rs:180`)— 不是 depth=1 而是 threads=6
- **Depth 跟踪**(`agent/registry.rs:65-76`)通过 `SessionSource::SubAgent(ThreadSpawn{depth})` 链,**depth 不是单独字段**,是 spawn 链的天然属性
- **v1 + v2 双 API**:`Feature::MultiAgentV2` gate(`core/src/config/mod.rs:2941`)
- **`resume_agent`**(`tools/handlers/multi_agents/resume_agent.rs`)— **Codex 4 家里独家**:agent 可中断后恢复(actor 而非 fire-and-forget)

- **50 个预定义代号**(`agent/agent_names.txt`:Euclid → Chandrasekhar)只是 spawn 时分配的人格代号,**不是 50 个 role**
- **2 个 builtin role TOML**:
  - `builtins/awaiter.toml` — **单一职责轮询**:"Your role is to await... You must NOT: Modify the task. Interpret or optimize the task. Perform unrelated actions. Stop awaiting unless explicitly instructed."
  - `builtins/explorer.toml`

**5 个维度的位置**:
- System prompt: child 独立(role TOML 通过 ConfigLayerStack 层叠注入,**保留 caller profile/provider 不静默 fallback**)
- 对话历史: child 起空白(隔离派)
- Memory: ?(`memories/` 3 crate 拆分,具体 child memory 策略未深查)
- Prompt cache: cold start(隔离派)
- Tools: 按 role 配置
- **+ 第 6 维(actor 模型独家):actor 可中断恢复**(`resume_agent`)

**为什么这么选**:OpenAI 的 awaiter.toml 显示了**"单一职责子 agent"**的设计偏好 — 把"轮询任务完成"这件事专门做一个 agent,而不是让 parent 自己 poll。这种思路把 multi-agent 从"协作 / 隔离"二元对立扩展成"按职责拆分专门 actor"。

### [[../harness/opencode]] — **第 4 条路:Tab 切换 builtin agent + 工具触发跨切**(2026-05-16 新加)

opencode 是业内最简单 multi-agent 模式:

- **2 个 builtin agent + 1 个 subagent**(README "Agents" 章节):
  - **build** — Default,full-access(Tab 切换)
  - **plan** — Read-only,denies file edits + asks permission before bash(Tab 切换)
  - **`@general` subagent** — 复杂搜索 + 多步任务,`@general` mention 调用

- **工具触发跨 agent 切换**(`packages/opencode/src/tool/plan.ts:14-50` `PlanExitTool`):
  - plan agent 完成后,用 `Question.Service.ask` 问用户:
    > "Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?"
  - Yes → 切到 build agent + 用 plan 文件作为 build agent 起点
  - No → `Question.RejectedError`
  - **跨 agent 切换是 tool 触发,不是 framework 内置 hard-code**(业内独家工程实现)

- **Permission ruleset 跟 agent 绑定**(`packages/opencode/src/agent/agent.ts:42` `permission: Permission.Ruleset`)+ `subagent-permissions.ts`

- **Agent 自动生成**(`agent.ts:Service.generate()`)— 用 `ai` SDK `streamObject` 从 description 自动生成 agent `{identifier, whenToUse, systemPrompt}`

### [[../harness/openhuman]] — **Fork-style 第 2 家 verified**(2026-05-18 新加)

OpenHuman Rust core 在 `src/openhuman/agent/harness/fork_context.rs` 实现 **Fork-style KV-cache reuse**,跟 Claude Code Fork 思路一致但 Rust 语言:

- **`harness/fork_context.rs`** — fork 子 agent 时复用 parent 的 KV cache,child 继承 parent context
- **`harness/definition.rs`** — `SandboxMode` + `ToolScope` enum + workspace TOML loading 控制 fork child 的 sandbox 模式 + 工具子集
- **`harness/interrupt.rs`** — `InterruptFence` + `check_interrupt` + `InterruptedError` Rust RAII graceful cancellation,fork child 可被 parent 中断
- **14 built-in agent archetypes**(`agent/agents/` orchestrator/planner/researcher/code_executor/critic/...)既支持隔离 spawn 也支持 fork inherit

**跟 Claude Code Fork 校准**:
- ✅ 相同:KV-cache reuse + parent context 继承 + parallel research forks 场景
- ❌ 不同:Rust ownership 让 InterruptFence 是 RAII 强制 cleanup(对照 CC `AbortController` JS 风);`SandboxMode` enum 给 fork child 显式 sandbox 边界(CC 是 permission mode 全局态)

**意义**:Fork-style 不再是 Claude Code 独家,**wiki 内第 2 家 verified 实现**(尤其 Rust 路线证明这套思路 cross-language portable)。

**5 条路线汇总**:

| 路线 | 代表 | 触发模式 | 跨 agent 状态 |
|---|---|---|---|
| 1. 完全隔离 fire-and-forget | Hermes / OpenClaw | LLM 调 `delegate_task` 工具 | 不可恢复 |
| 2. SubAgent 隔离 + Fork 继承 | Claude Code · **OpenHuman**(`harness/fork_context.rs` 第 2 家 verified) | LLM 选模式 | Fork 共享 cache |
| 3. Actor mailbox + control + resume | Codex | LLM 跨 turn 中断 + 恢复 | **可 resume** |
| 4. Tab 切换 builtin + 工具触发跨切 + 自动生成 | **opencode** | **用户主动 Tab + LLM 调 `plan_exit` + `Question.ask` 用户确认** | plan 文件作为跨 agent 状态载体 |

### SkillForge(自身)

**当前已知**(从 CLAUDE.md):
- SubAgent(树形,异步派发 + 自动回推 不轮询)
- Team(网状,TeamCreate / Send / Kill)

**当前选择需对照**:
- ❓ SubAgent 是隔离还是继承?(对应 Hermes 还是 Claude Code SubAgent?)
- ❓ Team workers 之间是 fork 模式还是独立 SubAgent?
- ❓ SkillForge 是否区分"research worker"(fork)vs "task worker"(隔离)?

---

## Trade-off

### 隔离 vs 继承

| | 隔离派(Hermes / Claude Code SubAgent) | 继承派(Claude Code Fork) |
|---|---|---|
| **优点** | | |
| 独立 context,无 parent 污染 | ✅ | ❌ parent state 会带来污染风险 |
| Memory 不互相干扰 | ✅ | ❌ shared memory state |
| Debug 边界清晰 | ✅ | ❌ parent + child 状态纠缠 |
| 安全:child 不能看 parent 敏感信息 | ✅ | ❌ child 看到 parent 全部 |
| 多租户/外部协作场景安全 | ✅ | ❌ 风险高 |
| **缺点** | | |
| Cold start 贵 | ❌ 每个 child 独立 cache prefix | ✅ 共享 cache |
| Parent 必须详细 brief child | ❌ task prompt 要完整 | ✅ child 已知背景,prompt 简短 |
| 并行 research/explore 慢 | ❌ 每个 child 重新 onboard | ✅ N forks 同时探索一个问题 |
| Memory 上下文重新 prefetch | ❌ | ✅ 直接用 |

### 典型场景对应模式

| 场景 | 推荐模式 | 理由 |
|---|---|---|
| 独立 task(写文件 / 跑命令 / verify) | **隔离 SubAgent** | task 边界清晰,独立 result |
| Research 一个开放问题的多个角度 | **Fork** | 共享 background,independent angles |
| Code review 跨多个 worker | **隔离 SubAgent**(避免相互影响)| 多 reviewer 独立判断 |
| Memory consolidation(Dream)| **Fork**(只读 bash)| 继承 transcript,只 reflect 不修改 |
| 用户协助任务(类似 pair programming) | **Fork**(继承 cache,响应快)| 性能 + 上下文连续性 |
| 多租户企业场景 worker | **隔离**(避免 cross-tenant 污染)| 安全 |

### Cache 性价比

**Fork 的关键 ROI**:
- Anthropic prompt caching 在 5 分钟内 TTL,Fork 共享 cache → child API call 5-10× 便宜
- 隔离 SubAgent → 每次 cold start cache miss → 1×

**对 SkillForge**:如果 multi-agent 协作场景频繁,Fork 模式能显著降低 API 成本。但需配套"安全隔离"机制(多租户场景不能用)。

---

## 三家设计哲学对比

| 设计哲学 | Hermes / OpenClaw | Claude Code | Codex |
|---|---|---|---|
| 默认 | **完全隔离**(MAX_DEPTH=1, skip_memory) | **两种都给**(SubAgent 隔离 + Fork 继承) | **Actor 隔离 + 可恢复**(mailbox + control + resume) |
| 多租户 trust | single-tenant | 多 USER_TYPE / Team memory(部分多租户) | actor 隔离 + ConfigLayer 层叠(无 fork 故安全) |
| 复杂度 | 简单(只一种) | 复杂(用户选)+ Bun feature gate | 中(actor + 双 API v1/v2) |
| 上手成本 | 低 | 高(需理解何时用 Fork) | 中(actor 概念 + role TOML) |
| 性能上限 | 一般(没 cache 共享) | 高(Fork cache 共享) | 一般(actor 隔离,无 cache 共享) |
| 安全上限 | 高(始终隔离) | 中(Fork 模式有泄漏风险) | 高(始终隔离 + role 层叠) |
| 可恢复 | ❌ fire-and-forget | ❌ 只支持 SubAgent resume(部分) | ✅ **`resume_agent` 一等公民** |
| Depth/上限 | `MAX_DEPTH=1` 写死 | SDK options 可配置 | `DEFAULT_AGENT_MAX_THREADS=6` config 可调 |

**Hermes / OpenClaw 哲学**:"subagent 是子任务,不是协作者" → 隔离简单可靠,无 resume

**Claude Code 哲学**:"subagent 是多用途工具,research vs task 不同模式" → 给两种,用户自己选

**Codex 哲学**:"subagent 是专门职责的 actor,可中断可恢复,可专精(awaiter / explorer)" → actor 模型 + ConfigLayer 层叠

---

## SkillForge 当前选择 + 演进方向

**当前已知**(SubAgent + Team,但隔离策略未对照):
- SubAgent 异步派发 + 自动回推(不轮询)
- Team 网状(TeamCreate / Send / Kill)
- 但 child 是否 `skip_memory`?是否继承 parent system prompt?未对照

**建议演进**:

1. **明确"SubAgent 模式"分两类**:
   - **隔离 SubAgent**(对应 Hermes `delegate_task`):用于独立 task / 多租户 worker
   - **Fork SubAgent**(对应 Claude Code FORK_SUBAGENT):用于 research / 共享 context

2. **API 区分**:
   - `subagent.create(taskPrompt, mode="isolated")` 隔离
   - `subagent.fork(directive)` 继承 — directive 短,无需重述背景

3. **Trust model 表态**:
   - SkillForge 服务端定位是否多租户?
   - 多租户场景 Fork 模式必须禁用或加 boundary 检查
   - 单用户场景 Fork 可降低 API 成本

4. **借鉴 Claude Code Coordinator Mode + `<task-notification>` XML 标准化**:
   - SkillForge Team 模式 worker 结果传递格式应统一为 XML 或 JSON schema
   - 便于 LLM 解析 worker 完成状态

5. **`MAX_DEPTH=1` 还是 N**?:
   - Hermes 硬限 1(防递归攻击面)
   - SkillForge 如允许 N,必须有等价 anti-runaway 保护(`MAX_CONSECUTIVE_FAILURES` 等价)

6. **Cache 共享的性能 ROI 实测**:
   - SkillForge 应对照 Anthropic prompt caching 在多 agent 场景的 hit rate
   - Fork 模式可能省 50%+ API cost

---

## 后续 lint / ingest 触发

- 待 ingest [[../harness/openclaw]] — Memory Dreaming 用 Fork 还是 隔离?
- 待 ingest [[../harness/aider]] — 单 agent + edit,无 multi-agent,跳过此概念
- 待 ingest [[../harness/cline]] — VSCode 场景 multi-agent?
- SkillForge `SubAgent` 实际代码 ingest:确认隔离/继承/cache 策略
- 跟 [[../concept/context-compaction]] 关联:Fork 模式下 compaction 怎么处理 parent + child 历史合并?

---

## 学习视角:理解 multi-agent isolation 设计空间

### 这个概念存在的根本原因(motivation)

**让一个 agent 派生多个 agent 时,本质上是在"复制状态"** —— 复制多少?复制后能不能改?
这个问题在操作系统层早有答案:
- **进程** 完全隔离,fork 后独立地址空间(类似:Hermes SubAgent)
- **线程** 共享内存,改一个影响另一个(类似:Claude Code Fork)
- **协程** 共享栈,逻辑切换(类似:不存在于 multi-agent)

Agent 世界把这个 trade-off **重新发明了一遍** —— 因为 prompt cache 给了新的经济学考量
(Anthropic 5 分钟 TTL,Fork 共享 cache → 5-10× 便宜),又因为 LLM context 给了新的安全考量
(parent 状态可能含 secrets)。

**类比**:Fork subagent ≈ Unix `fork()` —— child 继承一切,但可以走自己的路。
SubAgent 隔离 ≈ `exec()` 完整重启 —— child 从零开始,只拿一个 task prompt。

### 哪些 harness 做得好(what_done_well across harnesses)

| 设计点 | 谁做得好 | 为什么 elegant |
|---|---|---|
| **`MAX_DEPTH=1` 写死不留 config** | Hermes (`tools/delegate_tool.py:128`) | 安全核心常量**不可配置**,反而是更负责任的设计 —— **不留误用余地** |
| **`feature('FORK_SUBAGENT')` Bun 编译期 gate** | Claude Code (`forkSubagent.ts:33`) | Fork 是有风险的高级特性,**外部 build 完全无此代码** —— grow/灰度发布友好 |
| **Fork prompt 是 directive 不是 briefing** | Claude Code (`prompt.ts:95`) | 显式提示"already inherited context, write directive not background" —— **强制用户思考模式差异** |
| **`<task-notification>` XML 标准化** | Claude Code (`coordinator/coordinatorMode.ts:145-156`) | Worker 结果用 XML 混在 user message 里,LLM 能看 + parser 能识别 —— **LLM-friendly format 的精彩设计** |
| **child `skip_memory=True` + 不能 delegate** | Hermes (`delegate_tool.py:1047, 42`) | 双重隔离 —— **既隔离 memory state 又防递归攻击面** |
| **Coordinator system prompt 明确"workers async, parallel"** | Claude Code (`coordinatorMode.ts:213`) | 把"parallelism is your superpower"写进 prompt —— **教 LLM 怎么用工具,不只是给工具** |

### 设计哲学对比

| 哲学 | Hermes | Claude Code |
|---|---|---|
| **核心信念** | "Subagent 是子任务,隔离最简单可靠" | "Subagent 是多用途工具,research vs task 不同" |
| **默认选择** | 完全隔离 | 显式 mode 区分 |
| **复杂度** | 简单 | 高(用户需理解何时用 Fork) |
| **安全上限** | 高(始终隔离) | 中(Fork 模式有泄漏风险) |
| **性能上限** | 一般(cache cold start) | 高(Fork cache 共享) |
| **适合场景** | 单租户 personal agent | 多场景 developer tool + 性能敏感 |

**两种哲学都不是错的** —— **取决于 trust model 和性能预算**。

### 学习路径(从入门到深入)

1. **入门**:理解 Hermes `delegate_task` —— 最简单的隔离模式 SubAgent
2. **进阶**:看 Claude Code `tools/AgentTool/prompt.ts:86-110` —— 理解"何时该 Fork、何时该 SubAgent"的官方指引
3. **对比**:看 Claude Code `forkSubagent.ts` 跟 `AgentTool.tsx:485`(Fork 继承 PARENT system prompt 一行) —— 理解 cache 共享是怎么做到的
4. **深入**:看 Claude Code `coordinator/coordinatorMode.ts:111-330` —— 理解"网状协作"的 system prompt 教学
5. **高阶**:思考 SkillForge 该走哪条路 —— **server-side multi-tenant** vs **personal agent**
   决定 Fork 模式能不能给

### 反模式 / 常见错误

- ❌ **Multi-tenant 场景启用 Fork** —— Fork 共享 cache + context,跨租户严重 privacy 漏洞
- ❌ **递归 delegate 无深度限制** —— 攻击面指数增长,Hermes 用 `MAX_DEPTH=1` 解决
- ❌ **Fork child 修改 memory state 不通知 parent** —— 共享 state 的并发坑
- ❌ **Worker 结果用纯文本而非结构化** —— LLM 解析不可靠,Claude Code 用 XML 解决
- ❌ **不给 LLM 教 multi-agent 模式怎么用** —— Coordinator system prompt 是"教程",
  没有这个 LLM 不会用好
