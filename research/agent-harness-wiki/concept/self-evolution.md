---
type: concept
name: self-evolution
status: draft
verification: source-grounded
updated: 2026-05-16
sources:
  - "[[../source/hermes-agent]]"
  - "[[../source/claude-code]]"
  - "[[../source/openclaw]]"
  - "[[../source/codex]]"
  - "/Users/youren/myspace/hermes-agent/tools/skill_manager_tool.py"
  - "/Users/youren/myspace/hermes-agent/tools/skill_usage.py"
  - "/Users/youren/myspace/hermes-agent/agent/memory_manager.py"
  - "/Users/youren/myspace/codex/codex-rs/core/src/goals.rs"
related_harnesses:
  - "[[../harness/hermes-agent]]"
  - "[[../harness/openclaw]]"
  - "[[../harness/claude-code]]"
  - "[[../harness/codex]]"
---

# Self-Evolution(自进化)

## 定义

Agent 从**自己的会话经验**中**主动**或**被动**提取可复用的产物(skill / memory / rule / 修订的 prompt),持久化下来供未来调用,从而**避免每次任务从零重新发挥**。

> 注意:跟 **eval_loop** 是两件事 — eval 评判表现,self-evolution 反向抽知识。两者可联动(eval 失败案例可作为 self-evolution 输入),但实现层完全独立。

---

## 四类 self-evolution(关键分类,容易混淆)

> **2026-05-16 update v1**:Codex ingest 后扩展为 3 类。
> **2026-05-16 update v2**:Opik ingest 后扩展为 **4 类**。Opik Agent Optimizer 7 算法是"算法驱动自动改 prompt"形态(详见 [[prompt-optimization]]),既不是事后提炼,也不是 dev meta,更不是 budget 内嵌 — 是新的"runtime optimization 无人审"类。

| 类型 | 目标对象 | 产出 | 触发主体 | 案例 |
|---|---|---|---|---|
| **1. Agent runtime self-evolution(事后提炼)** | **终端用户** 的 agent | 改 `MEMORY.md` / `USER.md` / 增 agent-created skill / 调 agent 自身 prompt | LLM 在 session 内或后台 | OpenClaw Memory Dreaming(2026-03 源头),Hermes `commit_memory_session`,Claude Code `autoDream`(v2.1.88) |
| **2. Meta self-evolution(dev-facing)** | **开发者** | 改 `.claude/rules/` / 加新 agent / 加新 command / 改 system prompt 模板 | LLM 提议 → 人审 → 落 dev artifact | **Claude Code insight(新版)**,SkillForge `/evolve` |
| **3. Runtime budget 内嵌自约束(无 artifact)** | **当前 session 的模型行为** | 改不出 artifact,只调当前 session 输出方向 | 框架级硬约束 + 模板注入 | **Codex Goals + `token_budget` per goal** (`core/src/goals.rs`),`templates/goals/budget_limit.md` 注入让模型主动收敛 |
| **4. Algorithm-driven prompt optimization(runtime 无人审)** | **下次 session 用的 prompt** | 改 system prompt / few-shot / LLM 参数(运行时立即生效,无 dev artifact) | 算法迭代:遗传 / Bayesian / Meta-prompting / Pareto / Reflective | **Opik Agent Optimizer 7 算法**(`sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26`):EvolutionaryOptimizer / FewShotBayesianOptimizer / MetaPromptOptimizer / GepaOptimizer / ParameterOptimizer / HierarchicalReflectiveOptimizer / HRPO,详见 [[prompt-optimization]] |

**为什么要区分 4 类**:
- 类 1(runtime 提炼)改变**用户体验**(下次跟用户对话表现不同),**有 artifact**(MEMORY.md / skill)
- 类 2(meta)改变**开发流程**(下次 dev 跑 agent 时规则不同),**有 dev artifact + 人审**
- 类 3(budget 内嵌)改变**当前 session 行为**(不产出 artifact,运行时硬约束 + 模板注入引导模型收敛)
- 类 4(算法 prompt optimization)改变**下次 session 用的 prompt**(运行时自动生效,**无人审无 dev artifact**,但有 OptimizationResult 标准化对象可追溯)— **这跟类 1 不同**:类 1 是 LLM 看自己 transcript 写笔记,类 4 是算法跑 metric 改 prompt;**跟类 2 不同**:类 2 改 dev artifact 走人审,类 4 改 runtime prompt 自动生效

**容易踩的坑**:
- 把 meta 当 runtime — 比如把 `.claude/rules/` 写到 MEMORY.md 里 → 污染用户视角
- 把 runtime 当 meta — 比如把用户偏好写到 dev rule 里 → 跨用户泄漏
- 把 budget 当事后提炼 — Codex Goals 不是"压缩"也不是"提炼",而是**事前内嵌的资源约束**,模型在 session 内就要按 budget 优先级 reorder 行动
- **把 prompt optimization 跟 fine-tune 混淆** — 类 4 改 prompt 字符串(可回滚),fine-tune 改 weights(不可逆),两者完全不同(详见 [[prompt-optimization]] "跟 fine-tune 的对比")
- **把类 4 跟类 2 混淆** — Opik runtime optimizer 自动改 prompt,SkillForge `/evolve` 人审改 dev artifact,两条路线互补不冲突(详见 [[prompt-optimization]] "Opik vs SkillForge `/evolve`")

**SkillForge 必须分两套 API**:
- `agent.evolve_runtime()` — 改用户 memory / skill
- `agent.evolve_meta()` — 改 dev rules / agent / command(走 review queue)

---

## 设计空间(6 维)

| 维度 | 选项 |
|---|---|
| **触发方式** | 手动(`/evolve` 命令)/ session 结束 / 阈值触发(N 次反复模式)/ memory nudges 周期提示 / eval 失败后自动建议 |
| **输入源** | session trace(完整 transcript)/ 用户反馈 / eval 失败案例 / tool error history / user feedback explicit("以后这样做") |
| **产出物** | 新 skill(SKILL.md)/ 新 memory(MEMORY.md 条目)/ 新 rule(`.claude/rules/xxx.md`)/ 修订 system prompt 段 / 新 sub-agent 配置 / 新 slash command |
| **审核流程** | LLM-only 直接落库 / 写 review queue 人审 / LLM 提议 + 人审 + LLM 自查(双签)/ guard scan rollback |
| **回滚机制** | git 版本 / 影子运行(shadow mode 先观察)/ archive 而非 delete / use_count 阈值后才提升状态 |
| **生命周期管理** | use/view/patch 计数 / stale 判定 / 主动 archive 长期未用 / pin 关键 skill 不让 curator 动 |

---

## 各 harness 怎么实现

### [[../harness/hermes-agent]](核心卖点,源码 verified)

**触发**(多源叠加):
- `skill_manage` 工具:agent 在调试稳定流程后**主动调用**(`tools/skill_manager_tool.py:692`)
- `MEMORY.md` 周期 nudges:LLM 看 system prompt 里 memory 提示倾向沉淀
- `session_search`(辅助模型摘要历史)+ external memory `prefetch`:agent 在新 session **自动召回**相关经验
- `commit_memory_session()` 在 compaction rotation 前 trigger(`run_agent.py:8990`)
- `on_memory_write` hook(`agent/memory_manager.py:485`)→ external provider 同步 builtin memory 写入

**输入源**:
- SessionDB transcript(`hermes_state.py`)
- External memory provider 召回(`prefetch_all` `agent/memory_manager.py:287`)
- 用户 explicit 提示("记住...")

**产出物**:

#### A. Skill(SKILL.md)
`skill_manage` 工具 `actions` 集合(`tools/skill_manager_tool.py:708-741`):
| Action | 用途 |
|---|---|
| `create` | 新建 SKILL.md(frontmatter + 内容) |
| `edit` | 整文件 overwrite |
| `patch` | 字符串 search/replace |
| `delete` | 删除整个 skill 目录 |
| `write_file` | 写 supporting file(references / templates / scripts / assets) |
| `remove_file` | 删 supporting file |

**保护**:
- `_pinned_guard(name)` (`:134-152`):pinned skill 拒绝任何修改,5 处调用 (`:439, 483, 566, 613, 651`)
- skill name 安全字符校验
- SKILL.md frontmatter 合法性校验
- supporting file 路径只能写到 `references / templates / scripts / assets` 子目录
- atomic write(temp → rename)
- 可选 `skills.guard_agent_created` 配置开启 → guard scan rollback

#### B. Memory(MEMORY.md / USER.md)
`tools/memory_tool.py:107` `MemoryStore`:
- `memory_char_limit=2200, user_char_limit=1375` 字符上限(`:118`)
- MEMORY.md = agent 自己笔记(项目约定、命令别名等)
- USER.md = 用户画像(偏好、语言、工作模式等)
- 写入:加锁 → 读最新磁盘内容 → 去重 → char 上限检查 → atomic 写回
- prompt injection scan + exfiltration risk scan(防恶意写入)
- **冻结快照**:中途写不改当前 session 的 system prompt,**下个 session 才看到**(保 prompt cache)

#### C. External memory provider
8 个 plugin:honcho / mem0 / supermemory / byterover / hindsight / holographic / openviking / retaindb
各自结构化记忆格式,通过 `MemoryProvider` ABC(`agent/memory_provider.py:93-263`)

**审核流程**:
- `skill_manage` 写入有保护 + 可选 guard scan rollback,但**默认 LLM 直接写**(单租户 agent 信任模型)
- `MEMORY.md` 写入加锁 + prompt injection scan
- 无人审 queue(企业/团队场景需自行加层)

**回滚机制**:
- `skill_usage.py` sidecar 跟踪(`tools/skill_usage.py:171-179`):
  - `use_count` / `view_count` / `patch_count`
  - `last_used_at` / `last_patched_at` / `last_viewed_at`
  - `state`(active / stale / archived)
  - `pinned`(布尔,opt-out auto transitions,`:20`)
- Bump 函数:`bump_view`(`:270-274`)/ `bump_use`(`:278-284`)/ `bump_patch`(`:287-292`)
- `set_pinned`(`:308-311`)
- **仅 agent-created skill 进 sidecar** —— 内置(`.bundled_manifest`)和 Hub 安装(`.hub/lock.json`)由上游维护,curator 不动
- Curator 可 archive 长期未用,可 restore(重新激活)

**生命周期**:`active → stale → archived → restored`,`pinned` 关键 skill 锁定

### [[../harness/openclaw]](TypeScript,源码 verified,**self-evolution 源头**)

**核心发现**:OpenClaw 是 Memory Dreaming 的**源头创造者**,**Hermes `commit_memory_session` 是简化版,Claude Code `autoDream` 借鉴命名但简化实现**。演进链:

```
OpenClaw Memory Dreaming (2026-03 production)
       ↓ fork-继承
Hermes commit_memory_session (简化:无 Cron / 无评分 / 无日记)
       ↓ 共时(2026-03 v2.1.88 leak)
Claude Code autoDream (借鉴 "Dream" 命名,简化:无评分晋升 / 无日记 subagent)
```

**双系统并行**:Memory Dreaming(短期事件流晋升)+ Memory Wiki(长期结构化知识)。

#### Memory Dreaming(`extensions/memory-core/src/dreaming-*.ts`,8 文件)

**触发**(`dreaming.ts:30-40` verified):
- Cron name = `"Memory Dreaming Promotion"`,event text = `"__openclaw_memory_core_short_term_promotion_dream__"`
- `RUNTIME_CRON_RECONCILE_INTERVAL_MS = 60_000`
- Legacy Light/REM 已统一(`LEGACY_LIGHT_SLEEP_CRON_TAG` 等保留兼容)
- 2 hooks:`gateway:startup` reconcile Cron + `before_agent_reply` 检查 heartbeat event

**3 phases 扫描**(`dreaming-phases.ts::runDreamingSweepPhases`):
- Phase 1 — Daily Note 摄入(Light Sleep):读 `memory/YYYY-MM-DD.md`,固定分 0.62,每片段 ≤280 chars / 4 lines
- Phase 2 — Session Transcript 摄入(REM Sleep):读 `memory/.dreams/session-corpus/`,固定分 0.58,每扫 ≤240 msg / 文件 ≤80 / session ≤4096
- Phase 3 — Phase Signal:Light hit +0.05 / REM hit +0.08,半衰期 14 天

**6 维评分晋升**(`short-term-promotion.ts`):
```
score = frequency × 0.24    # 被召回次数(归一化)
      + relevance  × 0.30    # 召回时相关度
      + diversity  × 0.15    # 查询多样性(不同 query 数)
      + recency    × 0.15    # 时间衰减(半衰期 14 天)
      + consolidation × 0.10 # 跨天分布(不同天召回 = 更稳定)
      + conceptual × 0.06    # 概念标签丰富度
```

**晋升门槛**(AND 关系):`minScore=0.75 / minRecallCount=3 / minUniqueQueries=2`

**梦境日记 subagent**(`dreaming-narrative.ts`):
- 晋升完成后 subagent 写日记 → `DREAMS.md` append-only(managed block `<!-- openclaw:dreaming:diary:start/end -->`)
- **反元评论 prompt**:"**不说在做梦/AI/agent/LLM**",80-180 词,程序员诗人风格,流动散文(不用 Markdown 标题/列表)
- NarrativePhaseData:phase / snippets(≤12) / themes(≤6) / promotions(≤5)

#### Memory Wiki(`extensions/memory-wiki/src/` 25 files,**Karpathy llm-wiki 模式 production**)

详见 [[../concept/llm-wiki-pattern]]。

### [[../harness/claude-code]](TypeScript,源码 verified)

**autoDream** 系统(`services/autoDream/` 4 files)— Claude Code 字面意义"**做梦**":

**3-gate trigger**(`autoDream.ts:5-8, 63-65`,cheapest first):
1. **Time gate**: hours since `lastConsolidatedAt` ≥ `minHours` (default **24**)
2. **Session gate**: transcript count with mtime > lastConsolidatedAt ≥ `minSessions` (default **5**)
3. **Lock gate**: no other process mid-consolidation (`consolidationLock.ts`)

**配置覆盖**:`tengu_onyx_plover` GrowthBook config (`autoDream.ts:73-93`)
**Skip 条件** (`autoDream.ts:95-99`):
- `getKairosActive()` — KAIROS 模式跑自己的 dream
- `getIsRemoteMode()`
- `!isAutoMemoryEnabled()`
- `!isAutoDreamEnabled()`

**4-phase prompt**(`consolidationPrompt.ts` 全文 verified):
- **Phase 1 — Orient** (`:26-31`):`ls` memory dir + read entrypoint (`MEMORY.md`) + skim existing topic files
- **Phase 2 — Gather recent signal** (`:33-43`):daily logs(`logs/YYYY/MM/YYYY-MM-DD.md`)> drifted memories > narrow transcript grep `grep -rn "<narrow term>" ${transcriptDir}/ --include="*.jsonl" | tail -50`(注释:"Don't exhaustively read transcripts")
- **Phase 3 — Consolidate** (`:44-52`):write/update memory files,**绝对日期**(转 "yesterday/last week" 为具体日期),**删 contradicted facts**(如果今天的调研推翻旧 memory,源头修)
- **Phase 4 — Prune and index** (`:53-61`):keep entrypoint **≤ 200 lines AND ≤ 25KB**,demote >200 chars 的 index entry 到 topic file,resolve contradictions

**Read-only bash**(README:234):dream subagent 只能看不能改

**Forked subagent** 跑(独立 context)— 用 `runForkedAgent` (`autoDream.ts:16-17`)

**`SessionMemory`**(per-session 提取,`services/SessionMemory/sessionMemory.ts`):
- `tengu_session_memory` feature gate (`:81`)
- `shouldExtractMemory(messages)` (`:134-181`) 触发逻辑:
  - `(hasMetTokenThreshold AND hasMetToolCallThreshold)` OR
  - `(hasMetTokenThreshold AND !hasToolCallsInLastTurn)` — natural break
  - **Token threshold always required**(防过度提取)
- `extractSessionMemory()` (`:272`) sequential async,`querySource: 'session_memory'`, `forkLabel: 'session_memory'`
- `manuallyExtractSessionMemory()` (`:387`)
- Per-session memory dir mode `0o700`

**`extractMemories`** service (`services/extractMemories/prompts.ts`):
- 2 prompt 变种:`buildExtractAutoOnlyPrompt` (`:50`) / `buildExtractCombinedPrompt` (`:101`)
- 说明:"MEMORY.md is always loaded into system prompt — lines after 200 truncated" (`:78`)

**Team memory**(`services/teamMemorySync/` 5 files,**Hermes 完全没有**):
- `index.ts` + `secretScanner.ts` + `teamMemSecretGuard.ts` + `types.ts` + `watcher.ts`
- secret scanner + guard 防 secrets 共享 — **企业/Team 场景必备**

**Skills 6 LoadedFrom 来源**(`skills/loadSkillsDir.ts:67-74`):
- `commands_DEPRECATED`(legacy)/ `skills` / `plugin` / `managed` / `bundled` / `mcp`
- user vs project 实际合并到 `skills`,用 `SettingSource` (`userSettings`/`projectSettings`) 分流(`getSkillsPath()`)

**`/evolve` slash command**:SkillForge CLAUDE.md 提到(meta 层:对开发者的 rule/agent/command 建议),**未在 Claude Code v2.1.88 源码中 grep verify**(可能是 SkillForge 自己加的 command,不是从 Claude Code 抄的)。

**对比 Hermes**:
- Hermes `commit_memory_session(messages)` 在 compaction rotation **前** simple trigger
- Claude Code autoDream 是 **3-gate + 4-phase + Forked subagent** 系统化
- Claude Code 多 **Team memory + secret guard** 维度

### [[../harness/codex]] — **类 3:Goals budget 内嵌自约束**(无 artifact 提炼)

**Codex 不做事后 artifact 提炼**(没有 commit_memory_session 等价物 / 没有 autoDream 等价物 / 没有 Memory Dreaming 等价物)。它选了**完全不同的路径**:把"自约束"做成**运行时**而非**事后**。

**核心机制**(`core/src/goals.rs`,B 模式 verified):

- **ThreadGoal 是协议级一等公民**(`protocol/src/protocol.rs` `ThreadGoal` / `ThreadGoalStatus` / `ThreadGoalUpdatedEvent`)
- **每个 goal 内嵌 `token_budget`**(`goals.rs:46-55`):
  ```rust
  pub(crate) struct SetGoalRequest {
      pub(crate) objective: Option<String>,
      pub(crate) status: Option<ThreadGoalStatus>,
      pub(crate) token_budget: Option<Option<i64>>,
  }
  pub(crate) struct CreateGoalRequest {
      pub(crate) objective: String,
      pub(crate) token_budget: Option<i64>,
  }
  ```
- **5 个 OTEL metric**(`goals.rs:18-22`,`otel/src/metrics/names.rs:31-32`):
  - `codex.goal.created` / `codex.goal.completed` / `codex.goal.duration_seconds` / `codex.goal.token_count` / `codex.goal.budget_limited`
- **3 个 prompt template**(`goals.rs:57-79`):
  - `templates/goals/continuation.md` — goal 进行中提示
  - `templates/goals/budget_limit.md` — **budget 临近时注入,引导模型收敛**
  - `templates/goals/objective_updated.md` — 目标变更通知

**关键工程细节**:
- `validate_thread_goal_objective`(`protocol/src/protocol.rs`)— goal 文本格式校验,防 LLM 写歪
- `GoalContext` + `ContextualUserFragment`(`core/src/context.rs`)— goal 内嵌入 session context,让模型每个 turn 都"知道"自己在哪个 goal 内
- `UPDATE_GOAL_TOOL_NAME`(`core/src/tools/handlers/goal_spec.rs`)— 模型用专门 tool 更新 goal,这是 self-evolution 的 RPC 接口

**类 3 路线哲学**:
- **不提炼 artifact**(不写 MEMORY.md / 不增 skill / 不调 prompt)
- **不依赖人审**(不像 SkillForge `/evolve`)
- **不需要异步后台**(不像 OpenClaw Memory Dreaming / Claude Code autoDream)
- **靠模板注入引导模型自约束** — 当 `budget_limited` 信号触发,`templates/goals/budget_limit.md` 注入到下一 turn,模型主动调整(`continuation` 模式)或交接(`objective_updated` 模式)

**对照其他 3 家的位置**:
- **类 1**(runtime 提炼)在 OpenClaw Memory Dreaming / Hermes commit_memory_session / Claude Code autoDream 都有
- **类 2**(meta dev-facing)在 Claude Code insight(未 verify)/ SkillForge `/evolve` 有
- **类 3**(budget 内嵌)**只有 Codex 一家** — 这是新一类 self-evolution 形态

### EvolvingAgent(`/Users/youren/myspace/EvolvingAgent/`,待 ingest)

_(占位,从项目名推断这是个专门做自进化的 agent,待 ingest 后展开)_

### OpenClaw Memory Dreaming + QA Character Eval(已 ingest,见 [[../harness/openclaw]])

详见 [[../harness/openclaw]] frontmatter `self_evolution` 字段。

### SkillForge(自身)

**当前已知**(从 CLAUDE.md + README):
- ✅ **Eval Pipeline**:自动评测 + Prompt A/B + 自动晋升(**eval_loop**,非 self_evolution)
- ✅ `/evolve` slash command:meta 层(对开发者的 rule/agent/command 建议)
- ✅ Skill 系统:system-skills + user-skills 两层
- ✅ Reviewer agent 体系(`java-reviewer` / `compact-reviewer` / `code-reviewer` / `security-reviewer` 等)
- ❓ Agent runtime 自进化 — agent 能否自己 create skill / write memory?
- ❓ Skill 使用统计 — 是否有 sidecar 跟踪?
- ❓ Curator 机制 — 长期未用 skill 自动 archive?
- ❓ Memory nudges — 周期主动提示?

---

## Trade-off

### LLM-only 直接落库 vs 走人审

| | LLM-only | 人审队列 |
|---|---|---|
| 沉淀速度 | 快,session 内就能用 | 慢,有积压 |
| 质量风险 | 高(prompt injection / 错误经验固化 / 噪音) | 低 |
| 用户负担 | 0 | 需定期 review |
| 适用场景 | 个人 agent / memory 类小事实 | 企业 / 团队共享 skill / 关键流程 |
| **Hermes** | LLM 直接写,可选 guard scan rollback | ❌ 默认无 |
| **SkillForge** | ?(skill 创建路径未对照) | ✅ `async review system` 已有,需对照是否用于 self-evolution |

### Skill(显式步骤) vs Memory(隐式事实)

两者都是 self-evolution 产物,**形态不同**:
- **Skill** = "什么时候做什么,怎么做" — 步骤手册,适合**可复用流程**(发布流程 / PR review / 调研论文)
- **Memory** = "什么是真的" — 小事实,适合**背景知识**(用户偏好 / 项目约定 / 命令别名)

混淆代价:
- 把流程塞 memory → memory 膨胀(2200 chars 上限不够)+ 模型不容易"按步骤"执行
- 把事实塞 skill → skill 数量爆炸 + index 污染

Hermes 明确分工(`SKILL.md` frontmatter `description` + `tools/memory_tool.py` 描述):skill 是"操作手册",memory 是"长期小事实"。

### 计数跟踪粒度

| 跟踪粒度 | 价值 | 成本 |
|---|---|---|
| 完全不跟踪 | 0 维护 | 无法判断哪些 stale / 哪些重要 |
| use_count only | 简单 | 不知道 view 没触发使用的情况 |
| **use+view+patch+last_used+state**(Hermes) | 能做 curator 自动归档 + pin + restore | sidecar JSON 维护 |
| 全 trace(每次调用 timestamp + 上下文) | 能做 ML-based 推荐 | 存储 + 隐私 |

Hermes 选中等粒度(`tools/skill_usage.py:171-179`),够支持 curator 但不过度。

### Memory extraction 时机

**问题**:agent session 结束后再 extract memory,还是在 compaction rotation **前** extract?

**Hermes 选择**:在 `_compress_context` 的 `end_session` **之前** trigger(`run_agent.py:8990` `commit_memory_session(messages)`)。理由:压缩后中间历史已被摘要,memory provider 看不到原文,会丢重要事实。

---

## SkillForge 当前选择 + 演进方向

**当前已知**:
- ✅ Eval Pipeline(eval_loop,**不是** self_evolution — 区分清楚)
- ✅ `/evolve`(meta 自进化,对开发者)
- ✅ Skill 两层(system-skills + user-skills)
- ✅ Reviewer agent 体系(审核机制成熟)
- ❓ Agent runtime 自进化 API(`skill_manage` 等价)
- ❓ `skill_usage.py` 等价跟踪
- ❓ Curator 自动归档机制

**建议演进**(基于 Hermes verified):

1. **明确"meta 自进化" vs "agent runtime 自进化"边界**
   - `/evolve` 是对**开发者**的 rule/agent/command 建议(meta)
   - Hermes `skill_manage` 是对**终端用户的 agent runtime** 的 skill 建议(runtime)
   - 两层都有价值,不应混淆

2. **借鉴 `agent-created vs 内置 skill` 边界**(Hermes:`.bundled_manifest` / `.hub/lock.json` 标记)
   - 只对 agent-created 跑 curator,避免误删官方 skill
   - SkillForge `system-skills` 是非可删除的(对应 Hermes 内置),`user-skills` 才是 curator 范围

3. **借鉴 `skill_usage.py` 中粒度跟踪**(`:171-179`)
   - use+view+patch+last_used+state 是经过验证的最小可工作集合

4. **明确 review 流程分级**
   - LLM-only:个人 agent + memory 类小事实(2200/1375 chars 范围)
   - Async review queue(SkillForge 已有概念):企业 / 团队共享 skill / 高危 rule

5. **借鉴 `<memory-context>` 块化注入 + `StreamingContextScrubber`**(`agent/memory_manager.py:46-189, 65-173`)
   - 防止 self-evolution 产物在 resume 时滚雪球污染
   - 流式响应必须考虑跨 SSE chunk 的 tag 切分,否则 memory 内容会泄漏到用户可见输出

6. **借鉴 `commit_memory_session` 在 compaction rotation 前 trigger**(`run_agent.py:8990`)
   - 否则压缩可能丢失重要事实
   - SkillForge `CompactionService` 应加等价 hook

---

## 后续 lint / ingest 触发

- ⏸️ EvolvingAgent 跳过(用户已说不打磨)
- 待 ingest [[../harness/openclaw]] Memory Dreaming + QA Character Eval 系统(用户已有研究笔记 `research/OpenClaw...` 4 篇)
- 跟 [[../concept/memory]](待写)的边界:memory 是 self-evolution 产物之一,但 memory 系统本身有自己的设计空间(短期/长期/向量/symbolic/dialectic)
- SkillForge `system-skills/` 实际目录 ingest:确认 self-evolution API 形态

---

## 学习视角:理解 self-evolution 设计空间

### 这个概念存在的根本原因(motivation)

**LLM 是无状态的:每次 session 从零开始,昨天教过它的东西今天忘了。**
Self-evolution 是这个矛盾的工程化解决方案 —— **不让模型变聪明**(权重不变),
**而是让 agent 系统记住经验**(把"做过什么/学到什么"持久化为下次可消费的产物)。

**类比**:人脑通过"睡眠"巩固白天经验为长期记忆(实际神经科学过程),
而 LLM agent 通过 **autoDream / memory extraction / skill creation** 巩固会话经验为
持久产物。Claude Code 字面叫 "Dream system" 不是巧合 —— 是个**生物学启发的工程隐喻**。

### 哪些 harness 做得好(what_done_well across harnesses)

| 设计点 | 谁做得好 | 为什么 elegant |
|---|---|---|
| **3-gate cheapest-first 触发**(time → session count → lock) | Claude Code autoDream (`autoDream.ts:5-8`) | 廉价 gate 先 check,**省 95% 的 false-positive 触发** |
| **4-phase prompt 写作**(Orient/Gather/Consolidate/Prune+Index) | Claude Code `consolidationPrompt.ts` | 每 phase 一个明确动词 + 明确产物,**LLM 可执行性极高** —— prompt 工程典范 |
| **`commit_memory_session` 在 rotation 前 trigger** | Hermes (`run_agent.py:8990`) | 时序 elegant —— 旧历史还在时让 memory provider 抢救 |
| **`skill_manage` 6 actions + `skill_usage.py` 8 字段** | Hermes (`tools/skill_manager_tool.py:692` + `skill_usage.py:171-179`) | **运行时 API + 跟踪 sidecar 完整闭环** —— 不只是"能写 skill",还能"知道哪个 skill 没用过" |
| **agent-created vs 内置 skill 边界**(只对 agent-created 跑 curator) | Hermes(只 sidecar agent-created) | 安全 —— **官方 skill 由上游维护,不会被错误删除** |
| **MEMORY.md 上限 200 lines AND 25KB** | Claude Code (`consolidationPrompt.ts:55`) | 双重上限(行 + 字节)防止单一指标被绕过;**Index 不是 dump 的哲学** |
| **Team memory secret scanner + guard** | Claude Code (`services/teamMemorySync/`) | 企业场景必备 —— **共享不等于无限制共享** |
| **`pinned` flag opt-out auto transitions** | Hermes (`skill_usage.py:20`) | **保护关键 skill 不被 curator 误归档** —— 自动化系统里的"明确人类意图"出口 |
| **Read-only bash for dream subagent** | Claude Code (README:234) | dream 是反思不是执行 —— **限制权限符合任务语义** |
| **Token threshold always required** in `shouldExtractMemory` | Claude Code (`sessionMemory.ts:181`) | 单一防过度提取的硬约束 —— **多条件 OR 但 token 必须** |

### 关键概念分类(防混淆)

**经常被混在一起的三件事**:

| 概念 | 干什么 | Hermes | Claude Code |
|---|---|---|---|
| **Memory** | 长期事实存储 | MEMORY.md + USER.md(冻结快照) | memdir/ + autoMemory |
| **Self-evolution** | 从经验提取产物 | `skill_manage` + memory nudges | autoDream + SessionMemory + extractMemories |
| **Eval loop** | 评判模型表现 | RL/Atropos trajectories | analytics + transcript classifier(ML) |

**容易混淆点**:
- Eval ≠ Self-evolution。Eval 给"分数",Self-evolution 给"产物"
- Memory ≠ Self-evolution。Memory 是"存什么",Self-evolution 是"怎么从经验生成 Memory"

### 学习路径(从入门到深入)

1. **入门**:Karpathy 的 `llm-wiki.md` gist —— 理解"LLM 增量构建 wiki"的核心思想
2. **进阶**:Hermes `tools/memory_tool.py` + `MEMORY.md` —— 理解"小事实 + 冻结快照" pattern
3. **对比**:Claude Code `services/autoDream/consolidationPrompt.ts` 全文 —— 理解
   "4-phase prompt"是怎么写的
4. **深入**:Claude Code `services/SessionMemory/sessionMemory.ts:134-181` —— 理解
   "(token AND tool-call) OR (token AND natural-break)" 的触发逻辑设计
5. **高阶**:对比 Hermes `commit_memory_session` 跟 Claude Code `autoDream`
   —— 理解"同步 inline trigger" vs "异步 background consolidation"的取舍

### 反模式 / 常见错误

- ❌ **Self-evolution 跟 Eval 混淆** —— Eval 输出 score,Self-evolution 输出 skill/memory
- ❌ **把 memory 当 todo list 用** —— Memory 是长期事实,task 进度应该在 session 历史/transcript
- ❌ **LLM-only 直接落库无审核** —— prompt injection / 错误经验固化 / 噪音风险
- ❌ **Skill / Memory 不分类**(把流程塞 memory / 把事实塞 skill) —— 用错容器 ≈ 设计混乱
- ❌ **不跟踪使用频率** —— 长期会积累一堆 stale skill,无 curator 机制系统失控
- ❌ **覆盖原有 system prompt 中途** —— 破坏 prompt cache(Hermes 解决:冻结快照,下个 session 才生效)
