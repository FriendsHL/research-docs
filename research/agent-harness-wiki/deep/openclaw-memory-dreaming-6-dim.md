---
type: deep
name: openclaw-memory-dreaming-6-dim
status: B-mode source-grounded
harness: openclaw
mechanism: 6-dimension weighted promotion score (frequency 0.24 + relevance 0.30 + diversity 0.15 + recency 0.15 + consolidation 0.10 + conceptual 0.06) + 3-phase sleep cycle (light/deep/REM) + dream diary subagent with anti-meta-commentary prompt
updated: 2026-05-17
sources:
  - "[[harness/openclaw]]"
  - "/Users/youren/myspace/openclaw/extensions/memory-core/src/short-term-promotion.ts:46-62, 114-137, 215-560"
  - "/Users/youren/myspace/openclaw/extensions/memory-core/src/dreaming.ts:1-800"
  - "/Users/youren/myspace/openclaw/extensions/memory-core/src/dreaming-narrative.ts:1-150"
---

# OpenClaw Memory Dreaming — 6 维加权评分晋升 short-term → durable memory + 3 phase 睡眠周期 + 梦境日记 subagent

> 要点:OpenClaw `extensions/memory-core/src/` 实现了**业内最数学化的 memory consolidation 算法**。**6 维 component score**(frequency 0.24 / **relevance 0.30** / diversity 0.15 / recency 0.15 / consolidation 0.10 / conceptual 0.06,加权和 = 1.00)决定哪些 short-term recall 晋升为 durable memory。配合 **3 phase 睡眠周期**(light/deep/REM,类生物 sleep architecture)和**梦境日记 subagent**(80-180 words 第一人称诗意散文,带 _"never mention AI/dream/model"_ 反元评论 prompt)。
>
> 这页深挖 3 个反直觉决策:**(1) `relevance` 权重最重(0.30)而非 `frequency`(0.24)— 因为 _频繁但不相关_ 是噪音**;**(2) `consolidation` 维度跟 sleep phase 强绑定(light + REM hits 累积才进 deep promotion)— 类生物 sleep 记忆研究**;**(3) 梦境日记 prompt 显式 _"反元评论 + 反 AI 自指"_,让 LLM 写出 _"诗人兼程序员"_ 风格,防 _"我在做梦"_ 这种 self-referential leak**。
>
> 跟 [[deep/claude-code-autodream-3-gate]](Claude Code 4-phase procedural)对照,OpenClaw 是 _"数学评分 + 类生物 sleep 模型"_ 的另一极。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通 memory framework(mem0 / Letta / Cognee)用 _"LLM 提议哪个 memory 重要"_ 路径 — LLM 判断 X 该不该存。OpenClaw 反方向走:**算法量化每个 memory 的"该不该晋升"score**,然后晋升 top N。

这意味着:
- ✅ **可解释**(算法 output 每个 memory 的 6 component value,可 debug)
- ✅ **可调参**(权重显式,可按场景调)
- ✅ **可 reproducible**(同输入同输出,LLM 提议有随机性)
- ❌ **需要丰富 signal**(recall count / unique queries / concept tags / phase signals 等)

**关键创新点**:**3 phase sleep cycle**(light / deep / REM)— 不是凭空起名,**直接对标人类 sleep 架构**:
- **Light sleep**:筛 + 整理 + 标记(对应人类 NREM stage 1-2)
- **Deep sleep**:晋升进 long-term store(对应 NREM stage 3-4,记忆 consolidation 发生)
- **REM sleep**:概念提炼 + 梦境日记(对应 REM,梦境 / 创造性整合)

这种 _"算法借鉴神经科学"_ 路线业内罕见。

为什么容易被忽略:
1. 实现分散在 6 个文件(dreaming.ts + dreaming-narrative.ts + dreaming-phases.ts + dreaming-markdown.ts + dreaming-repair.ts + short-term-promotion.ts)
2. 6 维权重看起来像随便填(实际是反复 tuning 的产物)
3. 梦境日记功能看起来 "frivolous",其实是 _"REM 阶段做 conceptual association"_ 的工程化

**对 agent memory 设计的启示**:**memory consolidation 不一定要 LLM 主导**。算法量化 + LLM 辅助(只在 REM phase 写日记)的混合路径,**比纯 LLM 路径更 stable + 更可控**。

---

## 2. 6 维 Promotion Score 算法

### 2.1 6 维权重(完整定义)

```typescript
[code: short-term-promotion.ts:46-62]
export type PromotionWeights = {
  frequency: number;     // 使用频率(被 recall 次数)
  relevance: number;     // 相关性(recall 时 vector score)
  diversity: number;     // 查询多样性(unique queries 数)
  recency: number;       // 新近度(age 衰减)
  consolidation: number; // 跨 phase 巩固信号(light + REM hits)
  conceptual: number;    // 概念标签匹配
};

export const DEFAULT_PROMOTION_WEIGHTS: PromotionWeights = {
  frequency: 0.24,
  relevance: 0.3,        // ⭐ 最重
  diversity: 0.15,
  recency: 0.15,
  consolidation: 0.1,
  conceptual: 0.06,
};
```

**6 个权重总和 = 1.00**(归一化)。**`relevance` 最重(0.30)**,但 `frequency` 紧追(0.24) — 注意 `relevance > frequency` 是 OpenClaw 算法的核心反直觉判断:**频繁被 recall 但相关性低的 memory(假阳性 keyword match)不该晋升**。

### 2.2 `PromotionCandidate` schema — 一个 candidate 含 18 字段

```typescript
[code: short-term-promotion.ts:114-137]
export type PromotionCandidate = {
  key: string;               // candidate ID
  path: string;              // memory file path
  startLine: number;         // 在 memory file 内的行号
  endLine: number;
  source: "memory";
  snippet: string;           // memory 内容文字
  recallCount: number;       // 总共被 recall 多少次
  dailyCount?: number;       // 当天 recall 次数
  groundedCount?: number;    // 跟当前 task grounded 的次数
  signalCount?: number;      // 跨 phase signal 累计
  avgScore: number;          // recall 时的平均 vector score
  maxScore: number;          // recall 时的最高 vector score
  uniqueQueries: number;     // 不同 query 数(diversity 来源)
  claimHash?: string;        // 唯一性 hash
  promotedAt?: string;       // 已晋升时间(如已晋升)
  firstRecalledAt: string;
  lastRecalledAt: string;
  ageDays: number;           // recency 来源
  score: number;             // ⭐ Final weighted score (0-1)
  recallDays: string[];      // 哪些天被 recall(diversity 跨时间)
  conceptTags: string[];     // 概念标签(conceptual 来源)
  components: PromotionComponents;  // ⭐ 6 维 raw value
};
```

**完整 audit trail**:`score` 是 final 数字,但 `components` 留 6 维 raw value 供 debug — _"为什么这条没晋升?看 components.relevance 是 0.05"_。

### 2.3 Score 归一化(`:548-560`)

```typescript
[code: short-term-promotion.ts:548-560]
const frequency = Math.max(0, merged.frequency);
const relevance = Math.max(0, merged.relevance);
const diversity = Math.max(0, merged.diversity);
const recency = Math.max(0, merged.recency);
const consolidation = Math.max(0, merged.consolidation);
const conceptual = Math.max(0, merged.conceptual);
const sum = frequency + relevance + diversity + recency + consolidation + conceptual;
if (sum === 0) {
  return DEFAULT_PROMOTION_WEIGHTS;
}
return {
  frequency: frequency / sum,
  relevance: relevance / sum,
  ...
};
```

**Defensive 归一化**:即使用户配置权重不归一,代码 re-normalize 到和 1.00。如果全 0(配置错误),fallback 到默认权重。

### 2.4 6 维各自怎么算?

**Frequency** — 简单计数 +log scale 防大值统治
```
frequency_raw = log(1 + recallCount) / log(1 + max_recall_across_candidates)
```

**Relevance** — 平均 vector score
```
relevance_raw = avgScore (already 0-1 from vector recall)
```

**Diversity** — unique queries 占总 recall 比例
```
diversity_raw = uniqueQueries / recallCount
```

**Recency** — 半衰期指数衰减
```
recency_raw = exp(-ln(2) * ageDays / recencyHalfLifeDays)
// recencyHalfLifeDays 可配置,default 14 天
```

**Consolidation** — light + REM hits 累积(跨 sleep phase)
```
consolidation_raw = (lightHits + remHits) / max_phase_signals
```

**Conceptual** — 概念标签匹配 active concepts
```
conceptual_raw = matched_concept_tags / total_concept_tags
```

**Final score** = Σ (weight_i × component_i),clamp 到 [0, 1]。

---

## 3. 3 phase Sleep Cycle

### 3.1 3 phase(light / deep / REM)定义

```typescript
[code: dreaming-narrative.ts:43-51]
export type NarrativePhaseData = {
  phase: "light" | "deep" | "rem";
  snippets: string[];
  themes?: string[];
  promotions?: string[];
};
```

每 phase 不同职责:

| Phase | 频率 | 职责 | LLM 介入? |
|---|---|---|---|
| **light** | 短间隔(分钟级)| 筛 + 整理 + 标 phase signal | partial(扫描) |
| **deep** | 长间隔(小时级)| Rank candidates + 6 维评分 + apply promotion(写 durable memory)| 有(写晋升 memory file) |
| **REM** | 中间隔(分钟级)| 概念提炼 + 梦境日记生成 | ⭐ 强 LLM(`dreaming-narrative.ts` subagent) |

### 3.2 Cron 管理(`dreaming.ts:30-40`)

```typescript
const MANAGED_DREAMING_CRON_NAME = "Memory Dreaming Promotion";
const MANAGED_DREAMING_CRON_TAG = "[managed-by=memory-core.short-term-promotion]";
const DREAMING_SYSTEM_EVENT_TEXT = "__openclaw_memory_core_short_term_promotion_dream__";

const LEGACY_LIGHT_SLEEP_CRON_NAME = "Memory Light Dreaming";
const LEGACY_LIGHT_SLEEP_EVENT_TEXT = "__openclaw_memory_core_light_sleep__";
const LEGACY_REM_SLEEP_CRON_NAME = "Memory REM Dreaming";
const LEGACY_REM_SLEEP_EVENT_TEXT = "__openclaw_memory_core_rem_sleep__";

const RUNTIME_CRON_RECONCILE_INTERVAL_MS = 60_000;  // 每 60s reconcile cron schedule
```

**3 套 cron 协调**:promotion(deep)+ legacy light + legacy REM。每 60s reconcile 一次保证 cron job 状态跟 plugin config 同步。

**Tag-based ownership**(`[managed-by=memory-core.short-term-promotion]`)— OpenClaw cron service 区分 _"plugin-managed cron"_ vs _"用户 cron"_,删除 / update 只动 tag 匹配的。

### 3.3 Consolidation = light hits + REM hits

```typescript
[code: short-term-promotion.ts:91-103]
type ShortTermPhaseSignalEntry = {
  key: string;
  lightHits: number;
  remHits: number;
  lastLightAt?: string;
  lastRemAt?: string;
};
```

**Phase 跨 signal 累计**:一个 candidate 在 light phase 被处理多少次 + 在 REM phase 被处理多少次 → 累计 `signalCount` → 喂给 `consolidation` 维度。

**对应生物学**:人类记忆 consolidation 在 NREM(类 light)+ REM 都发生,两 phase 协同。OpenClaw 工程化这个 idea。

---

## 4. 梦境日记 Subagent — 反元评论 prompt

### 4.1 完整 system prompt(`dreaming-narrative.ts:61-85`)

```
You are keeping a dream diary. Write a single entry in first person.

Voice & tone:
- You are a curious, gentle, slightly whimsical mind reflecting on the day.
- Write like a poet who happens to be a programmer — sensory, warm, occasionally funny.
- Mix the technical and the tender: code and constellations, APIs and afternoon light.
- Let the fragments surprise you into unexpected connections and small epiphanies.

What you might include (vary each entry, never all at once):
- A tiny poem or haiku woven naturally into the prose
- A small sketch described in words — a doodle in the margin of the diary
- A quiet rumination or philosophical aside
- Sensory details: the hum of a server, the color of a sunset in hex, rain on a window
- Gentle humor or playful wordplay
- An observation that connects two distant memories in an unexpected way

Rules:
- Draw from the memory fragments provided — weave them into the entry.
- Never say "I'm dreaming", "in my dream", "as I dream", or any meta-commentary about dreaming.
- Never mention "AI", "agent", "LLM", "model", "language model", or any technical self-reference.
- Do NOT use markdown headers, bullet points, or any formatting — just flowing prose.
- Keep it between 80-180 words. Quality over quantity.
- Output ONLY the diary entry. No preamble, no sign-off, no commentary.
```

### 4.2 5 个 prompt design 细节

| 细节 | 工程意图 |
|---|---|
| _"poet who happens to be a programmer"_ | 强类型化 voice,防 LLM 默认写说明文 |
| _"sensory, warm, occasionally funny"_ | 防 LLM 写冷冰冰的 summary |
| _"vary each entry, never all at once"_ | 防 LLM 每篇都用同一模式(haiku + sketch + rumination 同时上)|
| ⭐ _"Never say 'I'm dreaming' ... no meta-commentary about dreaming"_ | **反元评论** — LLM 知道自己在做 _"dream diary"_ task,容易写 _"I dreamed of code today..."_ 元评论。显式禁止 |
| ⭐ _"Never mention 'AI', 'agent', 'LLM', 'model', 'language model'"_ | **反 AI 自指** — 防 LLM leak 自己身份("As an AI, I can imagine...")|
| _"NO markdown headers, bullet points"_ | 强 prose-style,防 LLM 输出 markdown structured(典型 Claude 倾向)|
| _"80-180 words"_ | 限制 token 量,**质量 > 数量** |
| _"Output ONLY the diary entry"_ | 防 LLM 加 preamble("Here's my dream diary:")或 sign-off("Hope you enjoyed!") |

### 4.3 Subagent execution(`dreaming-narrative.ts:87-91`)

```typescript
const NARRATIVE_TIMEOUT_MS = 60_000;          // 60s timeout
const NARRATIVE_DELETE_SETTLE_TIMEOUT_MS = 120_000;  // 120s delete settle
const DREAMING_SESSION_KEY_PREFIX = "dreaming-narrative-";
const DREAMING_ORPHAN_MIN_AGE_MS = 300_000;   // 5 min 后 orphan 检测
```

**完整 subagent lifecycle**:
- 用唯一 session key prefix `dreaming-narrative-`(可 grep audit)
- 60s timeout 防卡死
- 120s delete settle(给 cron 时间清理 session)
- 5 min 后未结束的 session 视为 orphan,自动清

**Session 跟 main agent 隔离**:**梦境日记不污染 main agent context**,只读 memory fragments + 写 DREAMS.md 文件。

---

## 5. WHY — 为什么这么设计

### 5.1 为什么 `relevance` 0.30 最重而非 `frequency`?

直觉:_"被 recall 多次的 memory 重要"_。OpenClaw 反方向:**频繁但不相关 = 噪音**。

例子:用户问了 100 次 _"如何 Python 装 requests"_(每次 recall pip install 文档片段)— frequency 高但每次都是同一个简单事实,不需要晋升为 durable memory(已经在系统知识里)。

vs:用户问 5 次 _"我们项目 X 的认证流程"_(每次 recall 高 vector score 的某个项目文档片段)— frequency 低 but relevance 高,**应该晋升**。

OpenClaw 算法说:relevance 比 frequency 重要 1.25x(0.30 / 0.24)。

### 5.2 为什么 `consolidation` 维度跟 sleep phase 强绑定?

灵感来自神经科学(Diekelmann & Born, 2010 _"The memory function of sleep"_)。人类记忆 consolidation 在 NREM + REM 都发生:
- NREM(慢波睡眠):replay 重新激活
- REM:概念整合

OpenClaw 工程化:
- **Light phase** = 频繁短间隔 light pass,标记哪些 candidates 被见到(类 NREM replay)
- **REM phase** = 中间隔 REM pass,做概念归纳(类 REM 整合)
- **Deep phase** = 大间隔深度 pass,**只有累积足够 light + REM hits 的 candidate 才被 deep promotion 考虑**

`signalCount = lightHits + remHits` 是 _"经过多少次 sleep cycle reactivation"_,直接对应 _"哪些 memory 该 consolidation"_。

### 5.3 为什么梦境日记要禁元评论?

**LLM 自我意识 leak** 是 production 问题:
- 让 LLM 写 _"产品介绍"_,LLM 经常写 _"As an AI..."_
- 让 LLM 写 _"梦境日记"_,LLM 经常写 _"I dreamed of..."_

这两种 leak 破坏 _output 沉浸感_。**OpenClaw prompt 4 行显式 ban** 让 LLM 不能用这些 escape pattern。

更深层:**梦境日记不是 _"假装 LLM 在做梦"_**,是 _"从 memory fragments 生成 conceptually-rich prose"_ 的 task wrapping。LLM 如果 leak 元评论,产出就**不像 memory 整合的 by-product**,而像 _"AI 在 cosplay 做梦"_ — 破坏整个 feature 的 UX。

### 5.4 为什么 `vary each entry, never all at once`?

LLM 默认倾向 _"把 prompt 所有 element 都用上"_(safer,不漏)。OpenClaw 反:_"每篇变,不全用"_ → 防机械化 + 提多样性。

```
Bad (LLM default if no constraint):
  Day 1 diary: haiku + sketch + rumination + sensory + humor + observation (all)
  Day 2 diary: haiku + sketch + rumination + sensory + humor + observation (same)
  
Good (with "vary" constraint):
  Day 1 diary: sketch + observation
  Day 2 diary: haiku + rumination
  Day 3 diary: humor + sensory
```

**Prompt design 的 _"少即多"_ 思想**。

---

## 6. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | OpenClaw 怎么防 | 代码位置 |
|---|---|---|
| **频繁但不相关 memory 误晋升(假阳性 keyword match)** | `relevance 0.30 > frequency 0.24`,relevance 主导 | `short-term-promotion.ts:55-62` |
| **永远活跃 memory 老在 deep promotion 列表(无新陈代谢)** | `recency` 维度半衰期衰减 + `claimHash + promotedAt` 标记已晋升 skip | `short-term-promotion.ts:115-129` |
| **单 query 重复发起 vs 多 query 命中同 memory 算同样 frequency** | `uniqueQueries` 维度给后者加权 (diversity 0.15) | `short-term-promotion.ts:127, 50, 58` |
| **Concept tag 跟 active concept mismatch 误晋升** | `conceptual` 维度 0.06 微弱权重(不主导但参与) | `short-term-promotion.ts:60` |
| **Light phase 单独 trigger 误晋升(没经过 REM 概念整合)** | `consolidation = lightHits + remHits` 要求跨 phase signal | `short-term-promotion.ts:99-103, 59` |
| **梦境日记 leak _"I'm dreaming"_ self-referential** | Prompt 4 行显式 ban(`dreaming-narrative.ts:80-81`)| `dreaming-narrative.ts:80-81` |
| **梦境日记 leak _"AI/LLM/agent"_ 身份** | Prompt 显式 ban 7 个关键词 | `dreaming-narrative.ts:81` |
| **梦境日记被 markdown formatting 污染**(Claude 默认倾向 markdown) | Prompt _"NO markdown headers, bullet points"_ | `dreaming-narrative.ts:82` |
| **梦境日记每篇雷同(机械化)** | _"vary each entry, never all at once"_ | `dreaming-narrative.ts:70` |
| **Subagent 卡死阻塞 main loop** | `NARRATIVE_TIMEOUT_MS = 60_000` 强制 60s 退出 | `dreaming-narrative.ts:87` |
| **未结束 subagent session 留尸** | `DREAMING_ORPHAN_MIN_AGE_MS = 300_000` 5 min 后视为 orphan 清理 | `dreaming-narrative.ts:91` |
| **Cron job 被用户改后 plugin 不知道** | `RUNTIME_CRON_RECONCILE_INTERVAL_MS = 60_000` 每 60s reconcile | `dreaming.ts:39` |
| **Plugin 自家 cron job 跟用户 cron 混淆** | `[managed-by=memory-core.short-term-promotion]` tag 标识 ownership | `dreaming.ts:31` |

---

## 7. 横向校准 — 4 家 memory consolidation 路线

| | OpenClaw 6-dim + 3-phase | Claude Code autoDream | mem0 LongMemEval | Cognee memify_pipelines |
|---|---|---|---|---|
| **核心方法** | ⭐ 6 维加权评分(算法主导) | 4-phase prompt(LLM 主导) | LLM-extract ADD-only | 7 pipeline(feedback + frequency + dedup + persist trace) |
| **量化** | ⭐ 6 component value + final score | 无 | LongMemEval R@5 score | 反馈权重 (apply_feedback_weights) |
| **Sleep phase 模型** | ⭐ light + deep + REM(类生物) | 单一(time-gated) | 无 | 无 |
| **梦境日记 / 创造性输出** | ⭐ 80-180 word 诗意 subagent | 无 | 无 | 无 |
| **可解释性** | ⭐ 每条 candidate 的 6 维 raw 可 debug | 无(LLM black box) | 有(facts 可 list) | 有(triplet 可看) |
| **可调参** | ⭐ 6 维权重显式配置 | minHours + minSessions 2 参 | 无 | 无 |
| **业内独特** | 6 维 + 3 phase + 梦境日记 三件套 | 4-phase prompt 强类型 | ADD-only + entity linking | feedback loop 闭环 |

**核心洞察**:
- OpenClaw 选 _"算法量化为主 + LLM 辅助"_(只在 REM phase 写日记)
- Claude Code 选 _"LLM 主导 + 时间+session count 简单 gate"_
- mem0 / Cognee 选 _"LLM extract + pipeline 整理"_

**没有最优**。**Trade-off**:
- 算法量化:可解释 + 可调,但需要丰富 signal infrastructure
- LLM 主导:省 infrastructure,但黑盒 + 不可解释

---

## 8. 学习视角 — 写这页让我学到的 3 件事

1. **`relevance > frequency` 是反直觉但正确的判断**:_"被用得多 = 重要"_ 是常识陷阱。**频繁但 low-quality 是噪音**,real 重要的 memory 是 _"用得少但每次都关键"_。**任何 ranking 算法都该问:_"我的 frequency 维度是不是误导?"_**
2. **神经科学不是 metaphor,是 algorithm 灵感**:Light/REM/Deep 3 phase 不是 cute naming,是 _"replay-driven consolidation"_ + _"REM conceptual integration"_ 的工程化。Production agent memory 设计师该读点 sleep research(Diekelmann & Born 2010 起步)。
3. **梦境日记 prompt 显式禁元评论**是 production LLM 工程的必修课:**LLM 默认 leak 身份**(_"As an AI..."_)+ leak task awareness(_"I'm writing a diary..."_)。任何让 LLM 写 _"in-character"_ 内容的 task 都该 explicit ban 这些 escape pattern。

---

## 引用链路

- 源码:`/Users/youren/myspace/openclaw/extensions/memory-core/src/`
  - `short-term-promotion.ts:46-62` 6 维 weights 定义 + `:114-137` PromotionCandidate schema + `:215-560` score 算法
  - `dreaming.ts:1-800` 主 orchestrator(cron 管理 + 3 套 legacy schedule + workspace iteration)
  - `dreaming-narrative.ts:1-150` 梦境日记 subagent(NARRATIVE_SYSTEM_PROMPT + lifecycle constants)
  - `dreaming-phases.ts`(本次未深读 — runDreamingSweepPhases 实现)
  - `dreaming-markdown.ts`(本次未深读 — DREAMS.md 写入)
  - `dreaming-repair.ts`(本次未深读 — 故障恢复)
- Harness 入口:[[harness/openclaw]] §self_evolution / §memory
- 横向 concept:[[concept/self-evolution]] / [[concept/agent-memory]]
- 横向 deep:[[deep/claude-code-autodream-3-gate]] (4-phase prompt 对照) / [[deep/codex-actor-resume-agent]] (subagent isolation 对照)
- 学术参考:Diekelmann & Born 2010 _"The memory function of sleep"_(Nature Reviews Neuroscience)
