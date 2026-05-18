---
type: deep
name: claude-code-insights
status: B-mode source-grounded
harness: claude-code
mechanism: 6-dimension typed enum facets (55 enum 值) + MULTIPLE-TIMES prioritization 算法 + 8 parallel INSIGHT_SECTIONS Opus LLM analysis + per-session facets immutable cache + Layer E "用户手动 review" 路线
updated: 2026-05-17
sources:
  - "[[harness/claude-code]]"
  - "[[source/claude-code-insights]]"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/commands/insights.ts:1-3200"
---

# Claude Code `/insights` — 业内最完整的 Layer D + Layer E _"用户手动 review"_ 路线

> 要点:Claude Code 内置 `/insights` slash command(`src/commands/insights.ts` 3200 行单文件)是**业内 OSS Layer D typed enum + Layer E 自动归因建议最完整实现**。**6 维 × 55 个 enum 值**(goal_categories 13 / friction 12 / satisfaction 8 / outcome 5 / helpfulness 5 / session_type 5 + success 7)远超 tau-bench(7 个 enum)+ AgentOps(15 个 enum)。**`MULTIPLE TIMES` prioritization 算法**(`:1430`)是核心 wisdom — _"用户在 2+ session 反复说同一件事 → 自动建议变成 CLAUDE.md 规则"_。
>
> 这页深挖 3 个反直觉工程决策:**(1) 为啥 6 维 55 enum 而不是 tau-bench 风简洁 3 enum?**;**(2) 为啥 `/insights` 设计上 _不自动改你的 CLAUDE.md_,只 generate 建议?**;**(3) 为啥 facets per-session 永久 cache,但 INSIGHT_SECTIONS 每次重跑?**
>
> 跟 [[deep/claude-code-autodream-3-gate]](Anthropic Layer E 的另一条路径)对照:**autoDream = 后台持续 memory consolidation,`/insights` = 用户主动 review session pattern**。同公司,两条互补 self-improvement 路径。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通用 Claude Code 的人,**`/insights` 是个低使用率 slash command** — 大部分人不知道有这玩意,或者觉得是 _"花哨的可视化"_。读完源码会发现这是 Anthropic 押注 _"用户驱动 self-improvement"_ 的核心 product feature:

**业内对照**:
| Framework | Layer E "用户 review" 模式 |
|---|---|
| **SkillForge V3 + V80** | 自动 candidate → A/B → canary → verify(用户审 proposal,但不 review trace pattern)|
| **Claude Code `/insights`** | ⭐ **用户主动 review trace pattern + LLM 建议改 CLAUDE.md(决策权 100% 在用户)** |
| **OpenClaw memory dreaming** | 自动 cron + 6 维评分 + 自动 promote memory(用户不 review)|
| **DSPy GEPA / Opik Optimizer** | 用户写 metric + 自动 optimize prompt(用户 review metric 不 review trace)|
| **MetaGPT AFlow** | 自动 workflow 生成(用户 review workflow output)|

**`/insights` 独特**:**LLM 不直接改你的 config / prompt / skill**,而是 _"生成建议 + 把决策权完整给用户"_。这是设计哲学,不是技术限制。

为什么容易被忽略:
1. **Slash command 入口隐藏**(不是 always-on,要用户主动跑)
2. **生成 HTML report,不是 actionable artifact**(用户要自己解析)
3. **每次跑 N 个 Opus LLM call,成本高**(facets per-session 后才 cache,首次跑可能 30 min + 几刀)
4. 源码 3200 行单文件,**初读门槛高**

**对 production agent platform 设计的启示**:**_"用户驱动 self-improvement"_ 跟 _"自动 self-improvement"_ 不是替代,是 _2 条互补路径_**。Anthropic 同时押 autoDream(自动)+ `/insights`(用户驱动)— SkillForge 当前重 _自动_ 缺 _用户驱动_,可以补 `/insights` 等价物。

---

## 2. 机制 1 — 6 维 55 typed enum 是怎么设计出来的

业内对照:
| Framework | enum 维度数 | 总 enum 值数 |
|---|---|---|
| **tau-bench** | 3(FaultAuthor / FaultType / GradingStrategy)| **7** |
| **AgentOps** | 2(SpanKind 12 + Legacy 3 sub-kind) | 15 |
| **Phoenix Score** | 2(KindType 4 + DirectionType 3)| 7 |
| **Claude Code `/insights`** | ⭐ **6**(goal / outcome / satisfaction / friction / helpfulness / session_type + success 子维度) | ⭐ **55** |

**为什么 Claude Code 55 而别人 7-15?**

3 个原因:

### 2.1 Claude Code 数据集大 = enum 可细分

Anthropic 内部跑 `/insights` 看的是**全公司 + 全 Coder workspace 的真实 Claude Code 使用数据**(`detectMultiClauding` + `getRunningRemoteHosts` 路径)。数据量级支撑 55 个 enum 值统计意义。**tau-bench 是 benchmark,数据有限,7 enum 够**。

### 2.2 不同维度有不同用途 — 不能合并

```
goal_categories  → "用户想做什么?" (input characterization)
outcome          → "做成了吗?" (success/failure binary-ish)
satisfaction     → "用户满意吗?" (sentiment,跟 outcome 不一样:做成了但不满意是可能的)
friction         → "失败的话, 哪里出问题?" (failure mode classification)
helpfulness      → "Claude 帮上忙了吗?" (跟 satisfaction 不一样:用户满意但 Claude 帮的不多是可能的)
session_type     → "这是什么类型 session?" (single_task vs exploration 等)
```

**6 维正交** — 合并会丢信息。设计师选择 _"细粒度可分析"_ vs _"简洁好记"_,Claude Code 选前者。

### 2.3 LLM 提取友好 — 5 enum 一次 prompt 容易

FACET_EXTRACTION_PROMPT 让 Opus 一次性提取所有 6 维(`:430-456`)。Opus 处理 6 维 × 5-13 enum 选择没问题,**结构化输出方便后续 aggregate**。

**如果只 7 enum**:LLM 出错率高(因为 enum 太粗,LLM 觉得 _"不准确"_ 倾向写自由文字)。

---

## 3. 机制 2 — MULTIPLE TIMES Prioritization(Layer E 核心 wisdom)

`insights.ts:1430` 的 prompt 注释:

```
IMPORTANT for claude_md_additions: 
  PRIORITIZE instructions that appear MULTIPLE TIMES in the user data. 
  If user told Claude the same thing in 2+ sessions 
  (e.g., 'always run tests', 'use TypeScript'), 
  that's a PRIME candidate — they shouldn't have to repeat themselves.
```

**翻译成算法**:

```
扫所有 sessions
  ↓
For each session: 提取 user_instructions_to_claude: string[] (SessionFacets 字段)
  ↓
Aggregate 跨 session — count occurrences of each instruction (semantic dedup, not exact match)
  ↓
For instructions with count >= 2:
   → 建议变成 CLAUDE.md addition
  ↓
Output:
  - addition: "Always run tests after modifying auth-related files"
  - why: "Based on 5 sessions where you said variations of this"
  - prompt_scaffold: "Add under ## Testing section"
```

### 3.1 为什么 _"重复 = 该固化"_ 是 wisdom?

3 个 production 真理:

| 观察 | 意义 |
|---|---|
| **用户重复说同一件事 → 痛点** | 用户在跟系统 _协议错位_。系统不知道这条 default 是什么,每次要用户说一遍 |
| **重复成本 = N × token + N × cognitive load** | 把 N 次重复变成 1 次写 CLAUDE.md,节省 N-1 次 |
| **重复说明用户 endorse** | 跟 _"用户说了一次"_ 不一样,_"说了 N 次"_ 是用户真心确认 — **置信度 = N 倍** |

**业内对照**:
- **mem0** ADD-only:存所有 fact,**不区分 _"用户重复 vs 用户说一次"_**(信号被埋)
- **MemPalace Verbatim**:存所有原话,**手动 review 时才发现重复**(被动)
- **LangMem reflection**:LLM 反思 memory,**重复识别隐含在 reflection prompt 但不显式**
- **Claude Code `/insights`** ⭐:**显式 prompt 算法 + 业务规则**(_"appear MULTIPLE TIMES"_ 是 first-class signal)

### 3.2 _"appear MULTIPLE TIMES"_ 是 LLM 算的,不是 deterministic

注意:`MULTIPLE TIMES` 不是简单 `count >= 2`,而是 **LLM 在 prompt 里自己判断 semantic similarity**:
- _"Always run tests"_ 跟 _"don't forget to run tests"_ 算同一条
- _"use TypeScript"_ 跟 _"please write in TS"_ 算同一条

**LLM 做 semantic dedup,not exact-match dedup**。这意味着算法有 LLM 的 fuzziness,但更智能。

### 3.3 给 SkillForge 启示

SkillForge MEMORY-LLM-SYNTHESIS 现在是 _"每次 session 后 LLM 提议 memory"_,**没有 _"跨 session 重复识别 → 自动建议固化"_ 的显式路径**。

`R-INSIGHT-1`(可加进 SkillForge backlog):
```
跨 session aggregate user_instructions
  ↓
LLM 识别 semantic 重复 (>= 2 次)
  ↓
自动生成 CLAUDE.md / behavior_rule 建议
  ↓
进 V80 state machine (proposal_pending → approved/rejected)
```

**工作量**:1-2 周(SkillForge 已有 V80 state machine,只需加 _"重复识别"_ 上游 source)。

---

## 4. 机制 3 — Cache 策略:Facets immutable / INSIGHT_SECTIONS regenerate

```
~/.claude/usage-data/
├── session-meta/<session_id>.json     ← deterministic stats, cache forever
├── facets/<session_id>.json           ← LLM-extracted qualitative, cache FOREVER
└── (HTML report 不缓存, 每次重生成)
```

### 4.1 为什么 facets 永久 cache?

**Per-session facets 是 invariant**:一个 session 跑完了,内容不变,LLM 提取的 facets 也不变。**重新跑 Opus 没意义,浪费钱**。

### 4.2 为什么 INSIGHT_SECTIONS 每次重跑?

**Aggregate 是 time-varying**:你昨天加了 5 个 session,今天跑 `/insights` 看 _"过去 30 天 friction"_ 跟上周看的不一样。所以 8 个 INSIGHT_SECTIONS 的 LLM analysis **必须重跑**(基于最新 aggregate)。

### 4.3 Cache 节省效果

```
首次跑 (N=100 sessions, 50 没 cache):
  - 50 × Opus facets extraction = ~$5
  - 8 × Opus section analysis = ~$0.5
  - 总 = ~$5.5, 时间 ~10 min

后续跑 (N=100 sessions, 95 已 cache, 5 新增):
  - 5 × Opus facets extraction = ~$0.5
  - 8 × Opus section analysis = ~$0.5
  - 总 = ~$1, 时间 ~30 s
```

**Cache 让 `/insights` 从 _"小时级 + 几刀"_ 降到 _"几十秒 + 几毛"_**。这是为什么 Anthropic 敢 ship 给用户 — production cost feasible。

### 4.4 给 SkillForge 启示

`R-INSIGHT-2`:任何 _"per-trace LLM 分析"_ 都该 _per-trace cache + aggregate regenerate_:
- SkillForge `LlmTraceEntity` LLM-as-judge eval 应该 per-trace cache
- 每次跑 dashboard 时只跑 aggregate(快 + 便宜)

---

## 5. WHY — 3 个核心设计选择

### 5.1 为什么 `/insights` _不_ 自动改 CLAUDE.md?

直觉想法:_"既然 LLM 已经生成 claude_md_additions 建议,为啥不直接 append 到用户的 CLAUDE.md?"_

Anthropic 选 _"不自动改"_,3 个原因:

| 原因 | 解释 |
|---|---|
| **侵入性 = 信任成本** | 自动改用户 config = 用户必须 100% 信任 LLM。**LLM hallucination 一次,用户永远不开 `/insights`** |
| **CLAUDE.md 是用户产物** | CLAUDE.md 是用户花精力 curate 的项目 _"协议文档"_。LLM 自动改 = 改 _"用户的话"_,文化上不接受 |
| **建议 vs 决策分离** | LLM 给建议,用户做决策 — _"LLM-augmented 而非 LLM-driven"_ |

**对照 SkillForge V3 + V80**:SkillForge 走 _"自动 A/B + canary + verify"_ 路线,自动改的是 _"系统内部 candidate"_(不是 CLAUDE.md 用户文档),所以可以更激进。**两个产品定位不同**(SkillForge 平台 vs Claude Code CLI),设计哲学不同 reasonable。

### 5.2 为什么 6 维 enum 而不是 3 维 enum?

如前 §2.2 — 维度合并丢信息,跟用户**真实 mental model**不匹配:_"做成了但不满意"_ 是常见场景,合并 outcome + satisfaction 看不见这个 case。

### 5.3 为什么 8 INSIGHT_SECTIONS 而不是 1 个 mega-prompt?

3 个原因:
- **Parallel speed** — 8 LLM call 并行 = 总时间 ≈ 1 call 时间
- **Per-section context isolation** — 每个 section LLM 只看跟它相关的 aggregate,不会被无关数据干扰
- **Independent failure** — 1 个 section 失败不影响其他 7 个(robustness)

---

## 6. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | `/insights` 怎么防 | 代码位置 |
|---|---|---|
| **首次跑成本爆炸(N sessions × Opus)** | 每 10 个 project dir yield event loop;META_BATCH_SIZE=50 并行;MAX_SESSIONS_TO_LOAD=200 上限 | `:2755-2790, :2820+` |
| **facets 重复 LLM 提取浪费钱** | per-session 永久 cache | `:941-988` |
| **长 transcript 超 LLM context** | `SUMMARIZE_CHUNK_PROMPT` 预 chunk summarize | `:870, 881-941` |
| **Session 没 valid 时间戳污染分析** | `hasValidDates(log)` filter | `:730` |
| **同 session 多 branch 重复算** | `deduplicateSessionBranches` | `:812` |
| **多 Claude 并发会话被忽略** | `detectMultiClauding` 滑窗算法独立 metric | `:1062-1143` |
| **LLM 把 Claude 自主行为算成用户 goal** | FACET_EXTRACTION_PROMPT explicit guideline 1:_"DO NOT count Claude's autonomous codebase exploration"_ | `:434-437` |
| **LLM 猜情感不准** | FACET_EXTRACTION_PROMPT guideline 2:基于用户字面表达(_"Yay!"_ → happy)| `:439-444` |
| **Friction 描述太 vague** | FACET_EXTRACTION_PROMPT guideline 3:5 类显式列举 + 描述 | `:446-451` |
| **Warmup session 污染数据** | `warmup_minimal` 单独 goal category | `:453` |
| **跑过预算撑爆账单(N sessions × $0.10)** | (没显式防,但 cache 极大降本)| (cache pattern) |
| **生成建议跟用户实际不符** | MULTIPLE TIMES algorithm — 只建议出现 2+ 次的(高置信度)| `:1430` |
| **8 sections 个别失败拖累整体** | 并行 + 独立 failure(`generateSectionInsight` per-section error handling)| `:1572, 1612` |
| **LLM Generate 自由文字 build trust 困难** | features_to_try 强制从 5 个 CC FEATURE REFERENCE 里选 | `:1396-1416` |

---

## 7. 横向校准 — Layer E 4 家路线对照

| | Claude Code `/insights` | SkillForge V3+V80 | Claude Code autoDream | DSPy GEPA |
|---|---|---|---|---|
| **触发方式** | ⭐ 用户主动 slash command | 自动(session 完成后)| ⭐ 自动 (autoDream 3-gate 后台) | 用户写 metric + 跑 optimizer |
| **改什么 surface** | **CLAUDE.md 用户文档**(建议)| skill / prompt / behavior_rule / hook(自动) | memory file(汇总不归因)| prompt(自动) |
| **typed enum 维度** | ⭐ **6 维 55 enum** | 3 维(approved/rejected/auto)| 0 | 0(LLM reflection 风) |
| **跨 session 聚类** | ⭐ AggregatedData 30+ 字段 | PatternClusteringService | ❌(单 session 内 4-phase) | ❌(per metric run) |
| **MULTIPLE TIMES 算法** | ⭐ 独家(`:1430`) | ❌(MEMORY-LLM-SYNTHESIS 不识别重复)| ❌ | ❌ |
| **自动改用户 config** | ❌ **设计上拒绝**(用户决策) | ✅ 自动改 skill / prompt | ✅ 自动改 memory file | ✅ 自动改 prompt |
| **决策权** | ⭐ **用户 100%** | 系统(用户审 proposal)| 系统(autoDream 全自动)| 系统(metric 决定)|
| **失败影响范围** | 0(不改东西)| skill rollback / canary 隔离 | memory 文件可手动修 | prompt 版本控制 |

**核心 insight**:**`/insights` 是 _"用户驱动 self-improvement"_ 范式的代表**,跟 _"自动 self-improvement"_ 是**两条互补路径**,不是替代。

**业内现状**:大部分 framework 走 _"自动"_ 路线(易卖 + 易 demo),**`/insights` 是少数派 _"建议而非改"_ 路线**。但 production user 真实接受度 — _"建议路线"_ 反而高(因为信任成本低)。

---

## 8. 学习视角 — 写这页让我学到的 3 件事

1. **_"建议路线"_ 不是 _"自动路线"_ 的弱化版,是另一条产品哲学**:Anthropic 同时 ship autoDream(自动)+ `/insights`(建议),不是 `/insights` 是过渡。**两条路线适合不同场景**:
   - 自动路线:系统知道改什么不会出错(memory consolidation / skill canary)
   - 建议路线:改的是用户 _"协议文档"_(CLAUDE.md / config),决策必须给用户

2. **MULTIPLE TIMES algorithm 是 production wisdom 的浓缩**:_"重复 = 痛点 = 该固化"_ 是 4 行 prompt 注释,但浓缩了 _"用户跟系统协议错位"_ 的 production 观察。**任何 _"用户跟系统反复同一交互"_ 的场景,都该问:_"这条该固化为系统 default 吗?"_**

3. **6 维 55 enum 不是 over-engineering,是 _"数据量级匹配粒度"_**:大量真实数据可以支撑细 enum,小数据只能粗。**设计 evaluation enum 时,先问 _"我有多少数据?"_,再决定粒度**。

---

## 9. R-INSIGHT 借鉴清单(给 SkillForge)

| ID | 项 | 优先级 | 工作量 |
|---|---|---|---|
| **R-INSIGHT-1** | **MULTIPLE TIMES algorithm**(跨 session 重复识别 → 自动建议 CLAUDE.md / behavior_rule)| 🔴 P1 | 1-2 周(已有 V80 state machine,只加上游 source) |
| **R-INSIGHT-2** | **Per-trace LLM-extracted facets 永久 cache + aggregate regenerate** | 🔴 P1 | < 1 周(给 LlmTraceEntity 加 facets table) |
| **R-INSIGHT-3** | **6 维 typed enum schema**(goal/outcome/satisfaction/friction/helpfulness/session_type)给 SkillForge ATTRIBUTION-AGENT 输入更结构化 | 🟡 P2 | 1 周(扩 `t_session_annotation` schema) |
| **R-INSIGHT-4** | **_"建议而非改"_ 路径**(/insights 平行存在于 SkillForge dashboard,给用户主动 review trace pattern 的入口) | 🟡 P2 远期 | 2-3 周 |
| **R-INSIGHT-5** | **detectMultiClauding 滑窗算法**(检测用户同时跑多个 agent session) | 🟢 P3 | 短期参考 |

**R-INSIGHT-1 + R-INSIGHT-2 共 2-3 周 = SkillForge Layer E 从 _"自动单路线"_ 升级到 _"自动 + 用户驱动双路线"_**。

---

## 引用链路

- 源码:`/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/commands/insights.ts:1-3200`(完整 single file)
  - `:228-258` SessionMeta schema
  - `:260-273` SessionFacets schema
  - `:275-326` AggregatedData schema
  - `:352-415` LABEL_MAP — 6 维 55 enum 完整
  - `:430-456` FACET_EXTRACTION_PROMPT 全文
  - `:1062-1143` detectMultiClauding 滑窗
  - `:1336-1495` 8 INSIGHT_SECTIONS prompts
  - `:1430` ⭐ MULTIPLE TIMES algorithm 核心 prompt
  - `:2655-2737` InsightsExport schema + S3 upload
  - `:2796+` generateUsageReport 主入口
- 配套 source:[[source/claude-code-insights]]
- Harness:[[harness/claude-code]] §self_evolution(已更新 insights 章节)
- Concept:[[concept/attribution]] §Layer E(已加 Claude Code `/insights` _"用户手动 review"_ 路线)
- 横向 deep 对照:[[deep/claude-code-autodream-3-gate]](Anthropic 另一条 Layer E 路径 — autoDream 自动后台 vs `/insights` 用户驱动)
- 社区生态:`rebelytics/one-skill-to-rule-them-all`(meta-skill 自学习)/ `aviadr1/claude-meta`(CLAUDE.md 持续学习)/ `yahav10/claude-insights`(把 HTML report 转 actionable artifact CLI)
