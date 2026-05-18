---
name: runtime-context-injection
type: concept
status: B-mode source-grounded
updated: 2026-05-16
sources:
  - "[[../claude code 源码/]]"
  - "[[harness/claude-code]]"
  - "[[harness/codex]]"
  - "[[source/codex]]"
---

# Runtime Context Injection — Claude Code `<system-reminder>` 与 Codex `InitialContextInjection`

> **不要把这两个混为一谈**。它们解决的是**完全不同的问题**,在 agent 生命周期里也处于**不同位置**。我第一遍调研时把它们都归到"compact"语义里,后来 grep `messages.ts:1789-1859` 才看清楚 — Claude Code 的 `<system-reminder>` 是**运行时随时**插入 instruction / memory / skill / task hint 的协议级机制,Codex 的 `InitialContextInjection` 是**compact 重启上下文**后注入 summary 的**注入位置选择**。前者是 in-context augmentation,后者是 post-compact rehydration。

---

## 1. 为什么这个 concept 重要(也很容易被忽略)

普通用 Claude Code / Claude API 的人,**根本不知道这个机制存在** — 它对用户隐藏。但你只要在 production 里跑 agent 跑得久了,就会撞上几类痛:

1. LLM 忘了上下文里的关键约束(memory / skill / 当前 mode)
2. compaction 把信息丢了,新的 LLM call 不知道之前发生了什么
3. 有些 instruction(比如 plan mode 限制、auto mode 边界)需要**每 N 轮重复**才不被遗忘
4. 子 agent / tool / hook 想往主 loop 传一条提醒,但又不能伪装成 user message
5. 注入的内容跟 prompt cache 冲突,每次都 invalidate cache

`<system-reminder>` 是 **Claude Code 对这 5 类问题的工程化解法的统称**,**业内独家的协议级方案**。其他 harness 要么没这个层(Hermes / OpenClaw / opencode)、要么解了 1-2 类(Codex 只解 compact rehydration,见 §6)。

> 用户实际开发经历(2026-05-16):"system reminder 也是我在开发过程中遇到了一些问题然后想解决办法和探究出来的,一开始我不清楚有这个能力。还有 prompt cache 虽然一开始我知道这个,但是也是后期才开始做的。"

**这就是为什么这个 wiki 要写。** 不是为了 catalog `<system-reminder>` 有多少种 attachment(那是结果),而是为了让下一个调研者**在第一次撞坑之前**就知道:1) 这个能力存在,2) 它在 prompt 里是个**协议级契约**,3) 不是普通 string concat。

---

## 2. 协议级契约 — 它首先是一个 system prompt 规则

Claude Code 在 system prompt 里**显式告诉 LLM** `<system-reminder>` 该怎么读:

```
[code: src/constants/prompts.ts:131-134]
function getSystemRemindersSection(): string {
  return `- Tool results and user messages may include <system-reminder> tags.
<system-reminder> tags contain useful information and reminders.
They are automatically added by the system, and bear no direct relation
to the specific tool results or user messages in which they appear.`
}
```

简化版还有第二个变体(`prompts.ts:190`):

> "Tool results and user messages may include `<system-reminder>` or other tags. Tags contain information from the system. They bear no direct relation to the specific tool results or user messages in which they appear."

**关键 3 个 contract bit**:

| 契约 | 防的失败模式 |
|---|---|
| "useful information and reminders" | LLM 把它当噪音忽略 |
| "automatically added by the system" | LLM 把它当成用户的实际话回应 |
| **"bear no direct relation to..."** | LLM 把它跟相邻 tool_result / user message 错误关联(e.g. tool 报错时 reminder 恰好出现 → LLM 误以为是 tool 的错误信息) |

**这是 trained 行为契约**,不是普通 string。你直接拿一个未经 Claude Code 系统 prompt 训练的 LLM,塞 `<system-reminder>...</system-reminder>`,它**可能**会照规矩处理,但**不可靠**。这条 trained behavior 是为什么 Claude Code 的 `<system-reminder>` 工作得好、而别家直接抄 tag 名字未必能 work 的原因。

---

## 3. 机制 — `wrap` + `idempotent` + `smoosh-siblings` 三件套

源码核心在 `src/utils/messages.ts`,3 个互锁的纯函数:

### 3.1 `wrapInSystemReminder` — 单条 wrapper

```typescript
[code: src/utils/messages.ts:3097-3099]
export function wrapInSystemReminder(content: string): string {
  return `<system-reminder>\n${content}\n</system-reminder>`
}
```

简单到不行 — 就是字符串包裹。**真正的工程在剩下两个**。

### 3.2 `ensureSystemReminderWrap` — 幂等保护

```typescript
[code: messages.ts:1797-1817]
function ensureSystemReminderWrap(msg: UserMessage): UserMessage {
  const content = msg.message.content
  if (typeof content === 'string') {
    if (content.startsWith('<system-reminder>')) return msg  // 已包裹,跳过
    ...
  }
  ...
  const newContent = content.map(b => {
    if (b.type === 'text' && !b.text.startsWith('<system-reminder>')) {
      changed = true
      return { ...b, text: wrapInSystemReminder(b.text) }
    }
    return b
  })
  ...
}
```

**为什么需要幂等**:`<system-reminder>` 内容**可能**被多次 normalize pass 处理(attachment → message → mergeUserMessages → hoist → final pass),如果不幂等就会嵌套 `<system-reminder><system-reminder>...`,LLM 看到嵌套 tag 会困惑。

**`startsWith` 而非 includes**:用 prefix 检测是因为 `<system-reminder>` 一定在 text block 开头(wrapper 控制) — 这让幂等检测是 O(1) 且不被中间偶然出现的 `<system-reminder>` 字符串误伤。

### 3.3 `smooshSystemReminderSiblings` — 这才是真功夫

```typescript
[code: messages.ts:1819-1860+]
/**
 * Final pass: smoosh any `<system-reminder>`-prefixed text siblings into the
 * last tool_result of the same user message. Catches siblings from:
 * - PreToolUse hook additionalContext (Gap F: attachment between assistant and
 *   tool_result → standalone push → mergeUserMessages → hoist → sibling)
 * - relocateToolReferenceSiblings output (Gap E)
 * - any attachment-origin text that escaped merge-time smoosh
 */
```

**这个函数在源码注释里直接讲了它防的 bug** — 我把它翻译成大白话:

```
正常 user message 长这样:
{
  role: "user",
  content: [
    { type: "tool_result", ... },
    { type: "tool_result", ... }
  ]
}

但是某些边界情况(PreToolUse hook 注入 additionalContext / tool reference 重定位)
会产出 sibling text block:

{
  role: "user",
  content: [
    { type: "text", text: "<system-reminder>..." },  ← 孤立 sibling
    { type: "tool_result", ... }
  ]
}

后果:LLM 看到一个 user 消息里 system-reminder 是独立 block,
跟下面 tool_result 不在一起 — 跟 contract "bear no direct relation"
反而矛盾(它们被 LLM 可能解释为有关联,因为是同一 user message)。

修复:smoosh 把 sibling text **合并进最后一个 tool_result 的 content**。
这样 system-reminder 跟 tool_result 内嵌在一起,LLM 已经被 trained
知道 "tool_result 里可能有 system-reminder,且 bear no direct relation"。
```

**这是 production 级细节** — `Gap E` `Gap F` 这种命名是 internal bug tracker 编号,说明 Anthropic 内部对这套机制做了 A/B 验证(`sai-20260310-161901, Arm B`,`messages.ts:1830` 注释提到)。

---

## 4. 注入触发的 5 大功能类(23+ attachment types)

不要去背 23 种 attachment type — 应该理解**它们解决的 5 类问题**:

| # | 功能类 | 典型 attachment.type | 防的失败 |
|---|---|---|---|
| **A** | **Mode 维持**(state reminder) | plan mode 三档(`getPlanModeV2Instructions` / `Sparse` / `SubAgent`,`messages.ts:3294/3380/3414`)+ auto mode 两档(`getAutoModeFullInstructions` / `Sparse`,`:3440/3448`) | LLM 在长 session 中忘了当前 mode 的边界(plan mode 不能写非 plan file / auto mode 可以连续动作但不能搞 destructive) |
| **B** | **Memory 注入 + 抗老化** | `relevant_memories`(`:3708-3721`)+ `nested_memory`(`:3700-3706`)+ `compact_file_reference`(`:3592-3598`) | 长 session 内存 drift / compaction 后 LLM 不知道某 file 之前看过 |
| **C** | **Skill / Tool 发现** | `skill_discovery`(`:3506`)+ `invoked_skills`(`:3644-3661`)+ `skill_listing`(`:3728-3737`)+ `dynamic_skill`(`:3723-3727`) | LLM 不知道有这个 skill 可用 / skill 调用后忘了上下文里它的指令 |
| **D** | **Task / Plan 协调** | `todo_reminder`(`:3663`)+ `task_reminder`(`:3680`)+ `plan_file_reference`(`:3636`) | LLM 不用 TodoWrite / TaskCreate 跟踪进度,长任务断片 |
| **E** | **Team / SubAgent 协作** | `teammate_mailbox`(`:3457`)+ `team_context`(`:3467`) + 8 hook 类型(`:1033-1040` `hook_blocking_error` / `hook_cancelled` / `hook_error_during_execution` / `hook_non_blocking_error` / `hook_success` / `hook_system_message` / `hook_additional_context` / `hook_stopped_continuation`) | 子 agent 想给主 loop 传消息但又不是 user 真实输入 / hook 结果给 LLM 看但又不能当 user 话回应 |

**关键洞察**:`<system-reminder>` 是 **agent 内**的**非用户输入但需要 LLM 看的 channel**。本质上是给 agent loop 加了一条**带 trained contract 的 metadata bus**。其他 harness 要么没这个 bus(直接拼字符串到 user message,LLM 容易当成用户的话),要么有但没 trained contract(LLM 不知道该怎么对待)。

### 4.A 一个递归彩蛋:写这份文档的过程中我自己就收到了 2 个 `<system-reminder>`

```
<system-reminder>
The task tools haven't been used recently. If you're working on tasks
that would benefit from tracking progress, consider using TaskCreate...
</system-reminder>
```

这就是上表 **D 类 task_reminder**(`messages.ts:3680-3699`),用 `Object Permanence + reminder cadence` 模式触发 — Claude Code 检测到我用了 TaskCreate 但好一段时间没 update,推送提醒。**我作为 LLM 看到这条之后,内部的反应正是被 contract 训练过的**:不当成用户话回应、不在结果里 mention 它、看完判断是否相关再行动。这条递归观察证明了:`<system-reminder>` 是个 working production protocol,在你 ingest 它本身的源码时,它在你身上**正在工作**。

---

## 5. 抗 hallucination 的精彩工程 — memoryAge 的 staleness contract

这是我**没预料到**的发现 — `memdir/memoryAge.ts:33-42`:

```typescript
[code: src/memdir/memoryAge.ts:33-42]
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''   // 新鲜不警告 (noise)
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

**这条 staleness 警告会被注入到 `<system-reminder>` 里,跟着每条 >1 day 老的 memory 出现**(`memoryAge.ts:49-53` 的 `memoryFreshnessNote` 直接包裹 `<system-reminder>` tag)。

注释里的动机更狠(`memoryAge.ts:28-32`):

> "Motivated by user reports of **stale code-state memories** (file:line citations to code that has since changed) being asserted as fact — **the citation makes the stale claim sound more authoritative, not less**."

**翻译**:用户反馈 Claude 把 7 天前的 memory(里面写"`foo.py:42 has function bar()`")当成今天的现实状态去判断,结果代码早改了。Claude 自己看到 `file:line` citation **反而更相信**,因为引用看起来权威。Anthropic 的 fix 是**让 memory 自己带 staleness warning**,而且 warning 必须用 `<system-reminder>` channel 才能让 LLM "正确地不当成 user 输入 + 仍然认真看"。

**这条工程的 takeaway**:
1. **memory 不是 fact**,是 "point-in-time observation"
2. 时间元数据(`memoryAge`)必须可见且**人类可读**(`memoryAge.ts:10-14` 的注释:"Models are poor at date arithmetic — a raw ISO timestamp doesn't trigger staleness reasoning the way '47 days ago' does")
3. **citation 越精确,stale 时越危险** — 这反直觉,但是 production 验证过的

---

## 6. Codex `InitialContextInjection` — 完全不同的东西

Codex 在 `core/src/compact.rs:59-67` 有这个枚举:

```rust
[code: codex/core/src/compact.rs:59-63]
pub(crate) enum InitialContextInjection {
    BeforeLastUserMessage,
    DoNotInject,
}
```

**它的角色**:`compact()` 把对话压缩后,得到一个 summary,这个 summary 要重新塞回新对话作为"上文"。**塞在哪里?** Codex 用这个枚举做决定:

- `BeforeLastUserMessage`:塞在最后一条 user message **之前**(mid-turn compact 用,因为下一步是 LLM call,这个位置最像"对话延续")
- `DoNotInject`:不塞(pre-turn compact 或 user manual `/compact`,这种情况 summary 通过别的路径出现)

**注释里的动机**(`compact.rs:59-63`):"模型训练时见到的位置"决定 injection point — 这是**训练对齐**(training distribution alignment),不是 in-context reminder。

### 6.1 关键区别 — 一张表说清

| 维度 | Claude Code `<system-reminder>` | Codex `InitialContextInjection` |
|---|---|---|
| **何时注入** | Runtime 任意时刻(每条 LLM call 前都可能 reroll attachments) | **只在 compact 完成后**注入 summary |
| **注入什么** | mode / memory / skill / task / hook 结果 — 5 大类 23 种 | 只有 1 种:compact summary |
| **怎么标识** | XML 风格 `<system-reminder>...</system-reminder>` tag + system prompt 训练契约 | 没有 tag — summary 作为 plain user message 出现,只是位置不同 |
| **LLM 是否被训练知道** | **是**(prompt rule + Anthropic 训练) | 否,LLM 不知道这是 "system 注入的" |
| **节奏控制** | `TURNS_BETWEEN_ATTACHMENTS=5 / FULL_REMINDER_EVERY_N_ATTACHMENTS=5 / TURNS_BETWEEN_REMINDERS=10`(`attachments.ts:265-293`) | 一次性,compact 后只注入一次 |
| **解决的问题** | "LLM 忘了 + 上下文 augment + sub-channel 通信" | "compact 完了对话怎么接得上" |

**所以**:Claude Code `<system-reminder>` 和 Codex `InitialContextInjection` 是**正交问题的正交方案**。如果 SkillForge 同时要这俩能力,就要**分别**做,不能合并。

---

## 7. Budget 协议 — 没人会告诉你的 5 个数字

这部分我也是 grep 才看到的(`attachments.ts:262-293`):

```typescript
[code: src/utils/attachments.ts:262-293]
TURNS_BETWEEN_ATTACHMENTS: 5,           // 每 5 轮才注入一次
FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,   // 每 5 次注入有 1 次是 full(其他 sparse 短版)
MAX_MEMORY_LINES: 200,                  // 每个 memory 文件最多 200 行
MAX_MEMORY_BYTES: 4096,                 // 每个 memory 文件最多 4KB byte cap
MAX_SESSION_BYTES: 60 * 1024,           // 整个 session 累计最多注入 60KB,超了停止 prefetch
VERIFY_PLAN_REMINDER_CONFIG.TURNS_BETWEEN_REMINDERS: 10,  // plan verify 提醒每 10 轮一次
```

每个数字背后都有**工程决策**(`attachments.ts:269-289` 注释把动机写出来了):

| 数字 | 决策 | 工程 reasoning |
|---|---|---|
| `MAX_MEMORY_BYTES = 4096` | 单文件 4KB | "Line cap alone doesn't bound size (200 × 500-char lines = 100KB). The surfacer injects up to 5 files per turn via `<system-reminder>`, **bypassing the per-message tool-result budget**, so a tight per-file byte cap keeps aggregate injection bounded (5 × 4KB = 20KB/turn)." |
| `MAX_SESSION_BYTES = 60 * 1024` | 累计 60KB / session | "over a long session the selector keeps surfacing distinct files — ~26K tokens/session observed in prod. Cap the cumulative bytes: once hit, **stop prefetching entirely**. Budget is ~3 full injections" |
| `TURNS_BETWEEN_ATTACHMENTS = 5` | 每 5 轮注入一次 | 不要 every-turn 因为 noise,不要 never 因为 drift |
| `FULL_REMINDER_EVERY_N_ATTACHMENTS = 5` | 5 次注入有 1 次 full,4 次 sparse | sparse mode 是"see full instructions earlier" 一句话(见 `getPlanModeV2SparseInstructions` 全文 1 行,`getAutoModeSparseInstructions` 同),full 是几百字。token 节省 + 关键 reminder 仍周期性 full |

**业内类似常量**:opencode 的 compaction 6 常量(`session/compaction.ts:42-92`)是同源思路,Codex 的 `COMPACT_USER_MESSAGE_MAX_TOKENS=20_000` 是同一类。**这种"production 常量带注释 reasoning"是判断一个 harness 是否真在 prod 跑过的最快信号**。

---

## 8. 失败模式清单 — `<system-reminder>` 实际上是哪些坑的解药

倒推:既然 `<system-reminder>` 工程这么复杂,它一定是修一堆 bug 修出来的。源码注释里能挖到的:

| 失败模式 | 修法 | 源码位置 |
|---|---|---|
| **嵌套 wrap** | `ensureSystemReminderWrap` startsWith 检测 + 幂等 | `messages.ts:1800/1808` |
| **PreToolUse hook 注入 standalone sibling**(Gap F) | smoosh-siblings 合并进 tool_result | `messages.ts:1822-1824` |
| **tool reference relocate 产出 sibling**(Gap E) | smoosh-siblings 合并 | `messages.ts:1825` |
| **真实 user input 被误并** | smoosh 用 `startsWith('<system-reminder>')` 做 discriminator,**只合并 SR-prefixed**,真实输入留独立 sibling | `messages.ts:1827-1831` + A/B `sai-20260310-161901` |
| **prompt cache 失效** | `relevant_memories` 用 attachment-creation 时存的 header,而非每次重算 — "rendered bytes are stable across turns (prompt-cache hit)"(`messages.ts:3711-3713`)| `messages.ts:3711-3719` |
| **memory stale 被引用为权威**(staleness 反authority 悖论) | memoryAge 注入 freshness 警告 | `memoryAge.ts:28-42` |
| **LLM 把 reminder 当 user 话回应** | system prompt 训练契约 + tag 命名 | `prompts.ts:131-134 / 190` |
| **reminder 太频繁 noise / 太少 drift** | 5/5/10 节奏 + full/sparse 双档 | `attachments.ts:265-293` |
| **memory 累积超 session budget** | `MAX_SESSION_BYTES=60KB`,超了停 prefetch | `attachments.ts:288` |

---

## 9. 跟 prompt cache 的协同(用户后期才意识到的)

这是用户(2026-05-16)亲口说的:_"prompt cache 虽然一开始我知道这个,但是也是后期才开始做的。"_

`<system-reminder>` 的几个机制**正好是为了不打破 prompt cache**:

1. **稳定 header**(`messages.ts:3711-3713`):`memoryHeader(m.path, m.mtimeMs)` 在 attachment 创建时就计算好存起来,**后续每轮渲染时不重算** — 因为重算会因时间戳格式变化导致 byte-level 不一致,prompt cache miss。
2. **idempotent wrap**:重复包裹会让 byte 不一致;幂等保证多 pass 后输出一致。
3. **smoosh 进 tool_result**:reminder 合并进 tool_result 而非作为新 sibling block,**减少 message 数量**,降低 prompt cache 的 message-boundary 敏感性。

**takeaway**:任何 runtime context injection 系统,**设计第一天就要考虑 prompt cache 边界**。Anthropic 在 Claude API 提供 prompt cache 后,Claude Code 整个 `<system-reminder>` 协议都是 cache-aware 设计 — 这不是后置 patch,是核心约束。SkillForge 如果加 runtime injection,**必须**先 verify prompt cache 边界。

---

## 10. SkillForge 当前现状 + 借鉴清单

> Iron Law:SkillForge 源码只读,以下基于 [verified-from-source]。

### 10.1 SkillForge 当前 runtime injection 能力 [verified-from-source]

- **CLAUDE.md 注入**(`SystemPromptBuilder.java:34/86/462`)— 拼 system prompt 时一次性注入,**非 runtime**
- **Context segment breakdown**(`ContextBreakdownService.java:142` claude_md segment)— 静态 segment,**非 runtime injection**
- **`<system-reminder>` tag** — 0 命中(`grep "system-reminder\|systemReminder"` skillforge-server / skillforge-core 业务代码无命中)
- **`compact_file_reference` 类似机制** — 0 命中
- **memory staleness warning** — `MemoryEntity.java` 有 `memory_kind` 三态(observation/reflection/optimized)但**无 age-based staleness 警告注入**

### 10.2 R-INJECT 借鉴清单(P1-P3)

| # | 借鉴项 | SkillForge gap | 工作量 | 优先级 |
|---|---|---|---|---|
| **R-INJECT-1** | **`<system-reminder>` 协议层**(tag + system prompt contract + wrap/smoosh) | 完全无 — 需要协议层从零设计,Java 包装层 `RuntimeReminderInjector` + system prompt 加 contract bit | 1-2 周 | 🔴 P1 |
| **R-INJECT-2** | **memoryAge staleness warning** — memory 注入时按 ageDays 加 freshness note(`>1d` 触发) | `MemoryEntity` 有 `created_at` / `updated_at`,可计算 age;但**无 staleness note 注入逻辑** | < 3 天 | 🔴 P1(直接防 hallucination) |
| **R-INJECT-3** | **Task reminder / Mode reminder cadence**(每 N 轮注入) | 完全无 — 需要在 `AgentLoopEngine` 加 turn counter + injection trigger | < 1 周 | 🟡 P2 |
| **R-INJECT-4** | **Sparse/full 双档 reminder**(每 5 次注入 1 次 full) | 不存在 — 但价值是 token 节省,SkillForge 已有 prompt cache 是更高优先级 | 1 周 | 🟢 P3 |
| **R-INJECT-5** | **跟 prompt cache 协同**(稳定 header / 幂等 wrap / cache-boundary-aware) | SkillForge **已有** prompt cache(用户确认 "后期才开始做的"),需要审查 cache boundary 是否被 runtime injection 破坏 | < 半周 audit + 修 | 🟡 P2 |
| **R-INJECT-6** | **Hook → reminder channel**(hook 结果走 SR channel 而非伪装 user message) | SkillForge `LifecycleHookDispatcher.dispatchCollecting` 返回结果当前如何注入未 verify,需要 grep | 0.5-2 周 | 🟡 P2 |

**注意**:R-INJECT-1 不要"伪 system-reminder"。如果用 OpenAI / 其他非 Claude provider,这个 tag 没有 trained contract,**裸抄是没用的**。需要在 system prompt 里**先训练 LLM 怎么读 tag**,或者用 provider-specific 的 instruction wrapping(Codex `InitialContextInjection` 走的是另一条路)。

---

## 11. 回到学习视角 — 写这份文档让我自己学到的 3 件事

1. **system prompt 是"训练 LLM 怎么读 tool result"的地方**,不是"告诉 LLM 你是助手"。Claude Code 用 5 行 prompt 教会 LLM "tags contain info, bear no direct relation",撬动整个 runtime augmentation 协议。
2. **production 常量带注释 reasoning** 是判断"这个 harness 真在 prod 跑过"的最快信号。比 stars / commits / docs 都准。
3. **抗 hallucination 不靠模型本身,靠 channel 设计**。memoryAge 的 "47 days ago" 比 ISO timestamp 触发 staleness reasoning 更好 — 这种发现来自 prod user reports,不是设计直觉。

---

## 引用链路

- 主要源码:`/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/`
  - `constants/prompts.ts:131-134, 190` 系统 prompt 训练契约
  - `utils/messages.ts:1789-1860, 3097-3134, 3294-3722` wrap / smoosh / 23 attachment switch
  - `utils/attachments.ts:265-293` 5 budget constant
  - `memdir/memoryAge.ts:1-53` staleness warning
- 对照 Codex:`codex/core/src/compact.rs:59-63` `InitialContextInjection`(`[[source/codex]]`)
- 跨页:[[concept/context-compaction]] / [[concept/hooks-design]] / [[concept/agent-memory]] / [[harness/claude-code]] / [[harness/codex]]
