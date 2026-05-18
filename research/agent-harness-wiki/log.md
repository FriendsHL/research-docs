---
name: log
type: meta
updated: 2026-05-15
---

# Log — Agent Harness & Eval Wiki

> 时间线,append-only。每条带"触发问题"栏,几个月后回看可知哪些问题答过、答案在哪。
> 解析:`grep "^## \[" log.md` 可拿到完整列表。

---

## [2026-05-15] bootstrap | 初始化 wiki

**触发问题**: 每次调研都要重做 harness/eval 框架研究,知识不沉淀;同时 SkillForge 迭代需要业界对照视角。

**改动**:
- 新建目录 `research/agent-harness-wiki/`
- 新建 [[purpose]] [[schema]] [[index]] [[log]]
- schema 定 13 个维度对齐 SkillForge 模块
- 候选 raw source 已在 [[index]] 列出

**takeaway**: 边界已定 — LLM 仅在 `agent-harness-wiki/` 内写,`/Users/youren/myspace/` 下所有项目均可读。Hermes 源码研究作为首个 ingest 目标。

---

## [2026-05-15] ingest | hermes-agent (A 模式 — 已 rollback)

**触发问题**: 同上,首次 ingest 用文档+笔记驱动。

**改动**: 4 个文件(source/hermes-agent.md、harness/hermes-agent.md、concept/context-compaction.md、concept/self-evolution.md)。

**rollback 原因**: 用户指出笔记是 2026-04-30 写的,Hermes 半月内发了 0.7-0.11 多版本,源码可能漂移。决定改 wiki-ingest skill 加 B 模式(source-grounded verification),所有精确数字/常量/函数名带 `[code: file:line]` 锚点,然后重做。

---

## [2026-05-15] meta | wiki-ingest skill 升级到 B 模式

**触发问题**: 第一次 ingest 暴露"事实保真度"问题 — 仅基于笔记/文档,无法验证精确常量。SkillForge 决策需要可追溯到源码的事实。

**改动**:
- `~/.claude/skills/wiki-ingest/SKILL.md` 重写,加 6 条 B 模式规则:
  1. 源码项目必读至少 5-8 个核心入口模块
  2. 精确数字/常量/函数名必带 `[code: file:line]` 锚点
  3. 维度内容首字段标 `[verified-from-source]`/`[doc-only]`/`[notes-only]` 三档
  4. Step 1 显式列证据不足维度
  5. Token 软上限 150K input
  6. Step 1 新增"已读 .py 入口"清单 + 笔记/源码差异清单

**takeaway**: A 模式(文档/笔记驱动)适合广度扫描和非源码资料,B 模式(源码驱动)是 production 决策的默认。**lint 规则补充**: harness 页 `[doc-only]`/`[notes-only]` 超过 3 条 = warning。

---

## [2026-05-15] ingest | hermes-agent (B 模式 source-grounded)

**触发问题**: 用 source-grounded 重做 Hermes 调研,精确常量/函数/路径必须可跳源码验证。重点回答:SkillForge 在 multi_agent depth、`_DESTRUCTIVE_PATTERNS` 黑名单、流式 memory-context 防泄漏、IterationBudget refund 机制、approval smart 模式、MemoryManager 1+1 限制上,从 Hermes 源码能拿到哪些 production-ready 做法?

**模式**: B (source-grounded verification)
**已读源码**: 17+ `.py` 文件段 + 2 个目录 ls + 3 份项目文档 + 6 篇笔记交叉验证,~180K input tokens(略超 150K 软上限但远低于 1M ceiling)

**改动**:
- 新建 [[source/hermes-agent]] (源摘要 + 已读源码清单 + 4 处笔记/源码差异 + 8 个新事实)
- 新建 [[harness/hermes-agent]] (13 维全填,每维带 `[code: file:line]` 锚点 + 证据等级标签)
- 新建 [[concept/context-compaction]] (7 维设计空间,Hermes Phase A/B + 切新 session + 反复压缩保护带行号)
- 新建 [[concept/self-evolution]] (6 维设计空间,`skill_manage` 6 actions + `skill_usage.py` 8 字段 record 带行号)
- 更新 [[index]] 加入 4 个 🔒 B 模式标签的页面

**B 模式发现的事实漂移(4 处,分 3 类)**:
1. **笔记真错** ❌ Terminal backends `6 → 7`,新增 `vercel_sandbox` (`tools/environments/`)
   — 已修笔记 `00-项目架构总览.md:389,422` + `index.md:45` + 加 `verified_by` frontmatter
2. **A 模式 wiki 自编错** ⚠️ `MAX_DEPTH=1`(parent=0→child=1),笔记**根本没提**,A 模式 wiki 自己 hallucinate 为 2 (`tools/delegate_tool.py:128`)
3. **A 模式 wiki 自编错** ⚠️ `approvals.mode` 实际 on/auto/off/**smart** 4 种,笔记**没说过 approvals.mode 取值**,A 模式 wiki 自己编为 on/auto/off (`tools/approval.py:995`)
4. **笔记不全** ➖ Gateway platforms 实际 18+,笔记只列举几个无具体数字 (`gateway/platforms/`)

**重要方法论 lesson**:A 模式不仅传递笔记错(case 1),**LLM 还会主动 hallucinate 看似精确的事实**(case 2/3)。B 模式价值更强 — 同时检出"笔记真错"和"LLM 自编错"。

**B 模式新发现(8 个,笔记没记)**:
- Subagent default budget = 50 (`run_agent.py:266-271`)
- `IterationBudget.refund()` for execute_code (`run_agent.py:273, 290-294`)
- `StreamingContextScrubber` 跨 SSE chunk 防泄漏 (`agent/memory_manager.py:65-173`)
- `_DESTRUCTIVE_PATTERNS` 显式 regex 9 个高危命令 (`run_agent.py:338-350`)
- `MemoryManager` 严格 1+1 限制 (`agent/memory_manager.py:206-228`)
- `commit_memory_session` 在 compaction rotation **前** trigger memory extraction (`run_agent.py:8990`)
- `summary_target_ratio` clamp `[0.10, 0.80]` (`agent/context_compressor.py:399`)
- `VALID_HOOKS` 16 个含 `on_session_reset` (`hermes_cli/plugins.py:87`,笔记漏列)

**takeaway**: B 模式立刻发现 4 处笔记漂移 + 8 个新事实,**MAX_DEPTH 这种安全核心常量笔记错了**(SkillForge SubAgent 若按笔记设计深度 2 就走弯路了)。SkillForge **直接可借鉴 8 条(高优先级)** + **业界印证 5 条** + **SkillForge 更前 3 条** + **待立项决策 5 条**。最高优先级:`_DESTRUCTIVE_PATTERNS` 直接作为 `bash_blocklist` baseline、`StreamingContextScrubber` 跨 chunk 防泄漏、`commit_memory_session` 在 rotation 前 trigger memory extraction、`approvals.mode=smart` 用 aux LLM 预判 approval。

---

## [2026-05-15] ingest | claude-code v2.1.88 (B 模式 source-grounded)

**触发问题**: Claude Code v2.1.88 作为 production-grade agent platform,在 hooks (27 个)/ compaction (5 级 pipeline)/ memory (autoDream + SessionMemory + Team + secret guard)/ multi-agent (Fork 继承 vs SubAgent 隔离 vs Coordinator 网状)/ permission (5 modes + YOLO classifier ML + iron_gate) 上,跟 Hermes 有哪些差异?哪些对 SkillForge 是 production-ready 可借鉴的具体设计?Fork vs SubAgent trade-off 该如何在 SkillForge 中体现?

**模式**: B (source-grounded verification)
**已读源码**: 19 处 `.ts/.tsx` 文件段 + 10 个目录 ls + README 全文 + 1 篇本地笔记 + token 用量 ~350K input(超 150K 软上限,远低于 1M ceiling)

**改动**:
- 新建 [[source/claude-code]](19 处源码引用 + 4 类新事实 + 13 行 Hermes 对比表)
- 新建 [[harness/claude-code]](13 维全填 + 12 条 SkillForge 启示 + 跟 Hermes 对照表 + 13 处行号锚点)
- 新建 [[concept/multi-agent-isolation]](5 维 trade-off + Hermes 完全隔离 vs Claude Code 两种都给的设计哲学对比 + SkillForge 决策建议)
- 更新 [[concept/context-compaction]] 加 Claude Code 5 级 pipeline + reactive 413 recovery + in-place vs 切新 session 对比
- 更新 [[concept/self-evolution]] 加 Claude Code autoDream 3-gate+4-phase + SessionMemory 触发逻辑 + Team memory + secret guard + Skills 6 来源
- 更新 [[index]] 加 4 个新页 + 2 个 concept 页升级 emoji 标记

**关键发现(11 个)**:
1. Claude Code v2.1.88 是 **production-grade agent platform**,远超 CLI 工具 — 40+ tools / 27 hooks / 30+ services
2. **HOOK_EVENTS 27 个**,zod-validated literal union (`entrypoints/sdk/coreTypes.ts:25-53`),比 Hermes 多 11 个(PostToolUseFailure/StopFailure/PostCompact/SubagentStart/PermissionRequest/Denied/TeammateIdle/TaskCreated/Completed/Elicitation/Result/ConfigChange/WorktreeCreate/Remove/InstructionsLoaded/CwdChanged/FileChanged)
3. **5 级 compaction pipeline**(budget/snip/microcompact/collapse/autocompact)vs Hermes 单 ContextCompressor — 分层更细
4. **`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`** + 注释 BQ 数据 1,279 sessions × 50+ failures = ~250K wasted API calls/day(`autoCompact.ts:71`)
5. **autoDream 3-gate**(time:24h, sessions:5, lock)+ **4-phase prompt**(Orient/Gather/Consolidate/Prune+Index)— `services/autoDream/`
6. **Fork subagent 继承 parent system prompt + context + cache**(`AgentTool.tsx:485`)— **跟 Hermes `MAX_DEPTH=1 + skip_memory=True` 完全相反**
7. **SessionMemory 触发逻辑**:(token AND tool-call) OR (token AND natural break),token 始终 required(`sessionMemory.ts:134-181`)
8. **Team memory secret scanner + guard**(`services/teamMemorySync/secretScanner.ts, teamMemSecretGuard.ts`)— Hermes 完全没有
9. **5 Permission modes**(default/plan/autoAccept/bypassPermissions/auto)+ **YOLO classifier ML** + `tengu_iron_gate_closed` 高危 gate
10. **Coordinator Mode 网状** + Workers async + `<task-notification>` XML 标准化(`coordinator/coordinatorMode.ts:145-156`)
11. **Skills 6 LoadedFrom 来源**:commands_DEPRECATED/skills/plugin/managed/bundled/mcp(`loadSkillsDir.ts:67-74`)

**SkillForge takeaway**: 12 条直接借鉴(高 + 中优先级)+ 4 条业界印证 + 4 条 SkillForge 反而更前 + 6 条立项决策(包括 **SubAgent 是否同时支持 Fork+隔离两种模式**、Compaction 拆 5 级、autoDream 模式、Team memory + secret guard、Permission 细分 5 modes)+ 4 条风险避坑

**takeaway**: Claude Code v2.1.88 是 SkillForge production-grade 路径的最佳镜子 — 27 hooks / 5 级 compaction / autoDream / Fork+SubAgent+Coordinator 三模式 multi-agent / Team memory + secret guard 都是 SkillForge 可直接对照演进的设计。**两份 ingest 之后形成的 wiki 已经有 3 个 concept 页(context-compaction / self-evolution / multi-agent-isolation),足以支撑 SkillForge 在这三个核心模块的设计决策**。

---

## [2026-05-15] meta | wiki 状态 snapshot(2 harness + 3 concept)

**触发问题**: 跑完 Hermes + Claude Code 两个 B 模式 ingest 后,wiki 当前形态怎么样?能否支撑 SkillForge 实际决策?

**当前 wiki**:
- 2 harness 页(hermes-agent / claude-code),都带 `[code:line]` 锚点 + 证据等级标签 + 跟 SkillForge 对照
- 3 concept 页(context-compaction / self-evolution / multi-agent-isolation),都跨 2 harness 对比
- 2 source 页(原始资料引用 + 已读源码清单)
- 4 meta 文件(purpose/schema/index/log)
- 总行数 ~3000+

**已检出笔记错误**:1 处真错(terminal backends 6→7)+ 已 verified-by 标记
**已检出 A 模式 wiki 自编错误**:2 处(MAX_DEPTH / approvals.mode)+ 已纠正

**takeaway**: 后续 ingest 优先级:OpenClaw(Hermes 前身,看继承关系)> Aider(代码 agent,SEARCH/REPLACE 范式独特)> Cline / opencode > Codex。每个 ingest 预估 50-90 分钟 B 模式。当前 wiki 已能回答"hooks/compaction/memory/multi-agent" 4 大核心问题 70%。

---

## [2026-05-16] meta | 加 learning_takeaway 维度

**触发问题**: wiki 之前过度偏 SkillForge(13 维 schema 直接对齐 SkillForge 模块,`skillforge_takeaway` 一等公民,log "触发问题"全 SkillForge 视角)。但用户原意:wiki **不只服务工程决策,也服务个人学习** —— 关心"这个 harness 跟其他不同的是什么?优势在哪?哪些点做得好?"。

**改动**:
- `schema.md` 加 `learning_takeaway` 三段规范(differentiator / strengths / what_done_well),跟 `skillforge_takeaway` 平级
- `purpose.md` 把"个人学习"提到完全平级,加"理解 harness 设计空间"作为主目的,加"用户笔记是学习路径记录(有的细致有的粗糙),wiki 以源码为准、笔记作参考"明确表态
- 回填 5 个页 `learning_takeaway` / "学习视角"段:
  - `harness/hermes-agent.md`:4 条 differentiator + 5 条 strengths + 7 条 what_done_well
  - `harness/claude-code.md`:8 条 differentiator + 6 条 strengths + 8 条 what_done_well
  - `concept/context-compaction.md`:加 motivation(类比工作记忆/长期记忆)+ 8 行 across-harness 对比表 + 学习路径 5 步 + 5 条反模式
  - `concept/self-evolution.md`:加 motivation(类比睡眠巩固记忆)+ 10 行 across-harness 对比表 + 3 概念分类表(Memory/Self-evolution/Eval)+ 学习路径 5 步 + 6 条反模式
  - `concept/multi-agent-isolation.md`:加 motivation(类比 OS 进程/线程)+ 6 行 across-harness 对比表 + 设计哲学对比 + 学习路径 5 步 + 5 条反模式
- `~/.claude/skills/wiki-ingest/SKILL.md` 加 learning_takeaway 规则:Step 1 模板新增 5.1 段、Step 2 强制要求、`learning_takeaway` 升为一等公民、concept 页加"学习视角"段

**takeaway**: wiki 现在双视角并行 — **工程视角**(skillforge_takeaway:可借鉴/印证/更前/待决策/风险)+ **学习视角**(learning_takeaway:differentiator/strengths/what_done_well)。后续 ingest 都同时产出两份。**核心方法论**:学习视角让我能回答"为什么这个设计 elegant",工程视角让我能回答"SkillForge 抄哪条"。两个问题答案不一样 — 比如 Hermes 的"`run_conversation()` 10K LOC 单函数"是学习视角的 what_done_well(集中状态有意识取舍),但是工程视角的"风险/避坑"(SkillForge 不该照搬)。

---

## [2026-05-16] ingest | openclaw (B 模式 source-grounded)

**触发问题**: OpenClaw 作为 Hermes 前身,**Memory Dreaming 源头 + Memory Wiki(Karpathy llm-wiki 模式 production 实现)+ QA Character Eval(最完整 production agent eval)**,如何改写 [[concept/self-evolution]] 的演进叙事?Memory Wiki 跟我们手做的 agent-harness-wiki 几乎同源,SkillForge 直接照抄 OpenClaw 是不是最高效路径?

**模式**: B (source-grounded verification)
**已读源码**: README/AGENTS.md/CLAUDE.md + `extensions/memory-core/src/dreaming.ts` grep + `extensions/memory-wiki/src/` 25 files ls + `extensions/` 30+ ls + `apps/` 4 native + 14 篇本地笔记交叉验证。Token 用量 ~700K input(70% of 1M ceiling)

**改动**:
- 新建 [[source/openclaw]](30 处源码引用 + 跟 Hermes/Claude Code 对比表 + 演进链)
- 新建 [[harness/openclaw]](13 维全填,双视角:11 条 skillforge_takeaway + 6+6+11 条 learning_takeaway)
- 新建 [[concept/llm-wiki-pattern]](8 维设计空间 + Karpathy 原始 pattern + nashsu/llm_wiki + OpenClaw production + 我们手做的对比)
- 新建 [[concept/agent-eval]](8 维设计空间 + OpenClaw QA Character Eval verified + Hermes/Claude Code 对比)
- 更新 [[concept/self-evolution]] 加 OpenClaw Memory Dreaming 完整链路(6 维公式 + 3 phases + Cron+heartbeat + 梦境日记 subagent + 反元评论 prompt)+ **修正演进链** OpenClaw → Hermes → Claude Code
- 更新 [[concept/multi-agent-isolation]] 加 OpenClaw subagent 用途有限(Dreaming narrative + QA Eval)+ 设计哲学(完全隔离同 Hermes)
- 更新 [[index]] 加 4 个新页 + 升级 multi-agent-isolation 标签

**B 模式核心发现(8 个)**:
1. **OpenClaw 是 Memory Dreaming 源头**:`MANAGED_DREAMING_CRON_NAME = "Memory Dreaming Promotion"` (`dreaming.ts:30`),legacy 常量证明已迭代(Light/REM 分开 → 统一)
2. **Memory Wiki = Karpathy llm-wiki 模式的 production 实现**:`extensions/memory-wiki/src/` 25 files 含 vault/ingest/bridge/compile/claim-health/prompt-section/tool/lint/obsidian — **跟我们手做的 agent-harness-wiki 同源**
3. **6 维评分晋升公式**:`freq×0.24 + rel×0.30 + div×0.15 + rec×0.15 + cons×0.10 + concep×0.06`,门槛 `minScore=0.75/minRecallCount=3/minUniqueQueries=2`
4. **梦境日记 subagent 反元评论 prompt**:"不说在做梦/AI/agent/LLM",80-180 词,程序员诗人风格 — **prompt 工程美学**
5. **双层 agent loop 显式拆分**:`runEmbeddedPiAgent` 外层 32-160 retries 容错决策 + `runEmbeddedAttempt` 内层 tool-use 失败抛 — **比 Hermes 单函数 9 阶段更 elegant**
6. **QA Character Eval 真实 Gateway 子进程**:`startQaGatewayChild()` IPC 通信不 mock,YAML DSL 编译 JS `new AsyncFunction()` — **3 harness 里 eval_loop 最完整**
7. **基于 pi-sdk 嵌入式集成**(Mario Zechner / badlogic 开源)三层:pi-ai + pi-agent-core + pi-coding-agent — **不重发明轮子**
8. **Extension-agnostic core + Plugin SDK npm 公开**:`packages/plugin-sdk`,30+ extensions 走同一契约,**比 Hermes/Claude Code 第三方友好**

**演进链修正**:
```
OpenClaw Memory Dreaming (2026-03 production,源头)
       ↓ fork-继承
Hermes commit_memory_session (简化:无 Cron / 无评分 / 无日记)
       ↓ 共时
Claude Code autoDream (借鉴 "Dream" 命名,简化:无评分晋升 / 无日记 subagent)
```

**takeaway**: OpenClaw 是 wiki 当前 3 个 harness 里**美学最高 + production 最完整**的:Memory Wiki(Karpathy 模式 production)+ Memory Dreaming(6 维评分 + 梦境日记)+ QA Character Eval(真实环境 + YAML DSL)是三个其他 harness 都未做到的系统。**SkillForge 知识库直接路径**:沿 OpenClaw Memory Wiki 路线演进(claim/freshness/contradiction/compile/auto-inject prompt),而不是从零设计。**核心 lesson**:OpenClaw 把 "prompt 工程 + 人文关怀(梦境日记)+ systems thinking(extension-agnostic + 三层分离)" 三者结合,**是 agent 设计美学的标杆**。

---

## [2026-05-16] meta | wiki 状态 snapshot(3 harness + 5 concept)

**当前 wiki**:
- 3 harness 页(hermes-agent / claude-code / **openclaw**),全 B 模式 source-grounded + 双视角
- 5 concept 页(context-compaction / self-evolution / multi-agent-isolation / **llm-wiki-pattern** / **agent-eval**)
- 3 source 页
- 4 meta 文件(purpose / schema / index / log)
- 总行数预估 4500+

**已检出**:
- 笔记真错 1 处(Hermes terminal backends 6→7,已修)
- A 模式 wiki 自编错 2 处(已纠)
- **演进链修正**:OpenClaw 是 self-evolution 源头(不是 Hermes)

**核心收获**:wiki 已能回答 hooks / compaction / memory / multi-agent / self-evolution / eval / llm-wiki-pattern 7 大核心问题 85%+

**后续优先级**(剩余主要 harness):
- Aider(SEARCH/REPLACE 范式,无 OpenAI tool_use,代码 agent 独特路径)
- Cline(VSCode 插件)/ opencode / Codex
- Langfuse / Opik / Inspect AI(eval framework 专项)

---

## [2026-05-16] meta | 加 community-knowledge 证据等级 + 版本时效机制

**触发问题**: 用户告知 Claude Code v2.1.88(我 ingest 的版本)已经过时,近期新增 **goal / agent view / agent manager / insight**(meta self-evolution:从 session 提取 → 改 rule/加 skill)。源码不可访问,只能社区资讯/推断,**无法 B 模式 verify**。怎么把这些放进 wiki 不破坏证据等级规则?

**改动**:
- `schema.md`:加第 4 档证据等级 **`[community-knowledge]`**(必须标信息来源 + 时间)+ frontmatter 加版本时效字段(`last_ingested_version` / `last_ingested_at` / `community_known_newer` / `known_post_ingest_features`)+ 4 条新 lint 规则
- `harness/claude-code.md`:frontmatter 加 `last_ingested_version: v2.1.88` + `community_known_newer: true` + `known_post_ingest_features` 4 项 + 末尾加 **「已知近期演进(post v2.1.88,unverified)」** section,列 goal/agent-view/agent-manager/insight 4 特性,每条带证据等级 + 推断机制 + 对 SkillForge 决策 + verify 优先级
- `concept/self-evolution.md`:加 **「两类 self-evolution(关键分类)」** 段,显式区分:
  - **Agent runtime self-evolution**(终端用户,改 MEMORY.md/skill):OpenClaw Memory Dreaming + Hermes commit_memory_session + Claude Code autoDream
  - **Meta self-evolution (dev-facing)**(开发者,改 .claude/rules/ / 加 agent / command):**Claude Code insight(新版)** + SkillForge `/evolve`
  - 容易踩的坑:meta vs runtime 混淆 → 跨视角污染
- `~/.claude/skills/wiki-ingest/SKILL.md`:加第 4 档 + 版本时效字段 + 「已知近期演进」section 模板

**核心方法论 lesson**:
- B 模式有"源码可访问"前提,**源码不可访问时不能强 verify**,但又**不该完全放弃信号**
- `[community-knowledge]` 第 4 档让"知道有这事但不准"的事实**进 wiki 但隔离主维度**,**保持主体可信度**
- 版本时效机制让 wiki 自带"知道自己 stale"能力,**自然演化** — 等新版可 ingest 时升级 `[community-knowledge]` → `[verified-from-source]`

**Claude Code 4 新特性最重要的是 `insight`**:**直接同 SkillForge `/evolve`** — meta self-evolution(dev-facing,改 rule/加 skill),verify 优先级最高。未来 Claude Code 新版可访问时,优先 ingest `insight` 实现细节。

**takeaway**: wiki 治理机制升级 — 现在能优雅处理"已知但 unverified"信号。任何 harness 发新版,标 `[community-knowledge]` + 列 `known_post_ingest_features`,下次 re-ingest 时升级为 verified。这套机制**不只解决 Claude Code 一次** — 是 wiki 长期可维护性的关键基础设施。

---

## [2026-05-16] ingest | codex (B 模式 source-grounded)

**触发问题**: Codex 作为 OpenAI 官方 + Rust 实现,在哪些维度上是工程化标杆?对 SkillForge 的 skill 编排(是否引入 JS host 范式)、goal 内嵌 token_budget、multi-agent actor 模型这 3 个决策意味着什么?

**改动**:
- 新建 [[harness/codex]] — 13 维 + 双视角 takeaway,版本锁 commit 302149d979 @ 2026-05-15(monorepo `0.0.0-dev`,无 semver)
- 新建 [[source/codex]] — 100+ crates 结构图 + 50+ file:line 锚点
- 新建 [[concept/code-mode-orchestration]] — V8 isolate LLM-writes-JS 范式,4 家里独家
- 新建 [[concept/hooks-design]] — 三家 hooks 数量阶梯(Codex 8 / Hermes 16 / Claude Code 27),`output_spill` 防爆量 + `plugin_hook_declarations` 白名单注册
- 新建 [[concept/agent-interop]] — `external-agent-sessions/` 跨 harness 迁移机制(SHA256 ledger + 50 条上限 + 30 天窗口)
- 更新 [[concept/multi-agent-isolation]] — 加 Codex 第三条路:actor 模型 + ConfigLayer 层叠 + `resume_agent`(`DEFAULT_AGENT_MAX_THREADS=6`,`agent/registry.rs:65-76` depth chain)
- 更新 [[concept/context-compaction]] — 加 Codex 3 套 compact + provider 路由(`should_use_remote_compact_task`)+ `InitialContextInjection::{BeforeLastUserMessage, DoNotInject}` 训练对齐 + 7 telemetry 枚举
- 更新 [[concept/self-evolution]] — 扩展为 3 类:Codex Goals + `token_budget` per goal + 5 OTEL metric 是新一类(runtime budget 内嵌自约束,无 artifact)
- 更新 [[index]] — 加 harness/codex + 3 个新 concept + source/codex

**关键 B 模式 verified 事实(50+ 个 file:line 锚点见 [[source/codex]])**:
- **Code Mode**:`PUBLIC_TOOL_NAME="exec"` + `WAIT_TOOL_NAME="wait"`(`code-mode/src/lib.rs:36-37`),`DEFAULT_EXEC_YIELD_TIME_MS=10_000` / `DEFAULT_MAX_OUTPUT_TOKENS_PER_EXEC_CALL=10_000`(`runtime/mod.rs:23-25`),`rusty_v8` + `v8_enable_sandbox`(`code-mode/Cargo.toml:13,28`)
- **Hooks 8 lifecycle**(`hooks/src/lib.rs:18-27`),6 个 dispatch matcher(`:34-41`),独立 crate,`output_spill.rs` 防输出爆量,`plugin_hook_declarations` 白名单注册,`legacy_notify.rs` 向后兼容
- **Goals 一等公民**(`core/src/goals.rs`):`SetGoalRequest` / `CreateGoalRequest` 带 `token_budget: Option<i64>`,5 OTEL metric `codex.goal.{created,completed,duration_seconds,token_count,budget_limited}`,3 prompt template(`continuation.md` / `budget_limit.md` / `objective_updated.md`)
- **Multi-agent**:`DEFAULT_AGENT_MAX_THREADS = Some(6)`(`config/mod.rs:180`),`SessionSource::SubAgent(ThreadSpawn{depth})` 链(`agent/registry.rs:65-76`),v1 + v2 双 API(`Feature::MultiAgentV2`),50 个预定义代号 + 2 个 builtin role TOML(`awaiter` 单一职责 + `explorer`),`resume_agent` 独家
- **Compact**:`compact.rs:585` + `compact_remote.rs:387` + `compact_remote_v2.rs:456` 三套,`should_use_remote_compact_task(provider)` 路由(`:65`),`InitialContextInjection::{BeforeLastUserMessage, DoNotInject}`(`:59-63`)训练对齐,`COMPACT_USER_MESSAGE_MAX_TOKENS=20_000`(`:48`),7 telemetry 枚举(`:17-23`)
- **External agent migration**:`SESSION_IMPORT_MAX_COUNT=50` + `SESSION_IMPORT_MAX_AGE=30 天`(`detect.rs:10-11`),`external_agent_session_imports.json` SHA256 ledger 防重(`ledger.rs:12-26`),`NOTE_MAX_LEN=2000` + `TOOL_RESULT_MAX_LEN=4000` 截断(`records.rs:14-16`)
- **OTEL 工程化标杆**:独立 `otel/` crate(metrics/events/otlp/trace_context/targets),`codex.goal.*` 5 metric(`otel/src/metrics/names.rs:31-32`),`rollout/` JSONL + SQLite 双轨持久化 + 自动 git_info(`rollout/src/recorder.rs:1,50-58`)
- **3 平台沙箱**:`sandboxing/src/{bwrap,landlock,seatbelt}.rs` + 3 个 `.sbpl` Seatbelt policy + `policy_transforms` 抽象层 + Windows 独立 crate
- **Memory 拆 3 crate**:`memories/{read,write,mcp}/`,`protocol/src/memory_citation.rs` 协议级 citation

**对其他 3 家的影响**:
- [[concept/multi-agent-isolation]]:多 agent 三家路线现在完整 —— Hermes/OpenClaw 完全隔离 / Claude Code 隔离+Fork 两选 / **Codex actor + resume(第三条路)**
- [[concept/context-compaction]]:Codex 是 7 维设计空间里"训练对齐注入位置"和"provider capability 路由"的代表
- [[concept/self-evolution]]:扩展为 3 类(原 2 类:runtime 提炼 / meta dev-facing,新加:**runtime budget 内嵌自约束**)

**takeaway**: Codex = 工程化标杆。**8 个关键借鉴**给 SkillForge:goals + token_budget per goal、8 lifecycle hook 精简、`output_spill` 防爆、`plugin_hook_declarations` 白名单、compact provider 路由、InitialContextInjection 训练对齐、ConfigLayerStack role 层叠、multi-agent actor + resume。**3 条 SkillForge 不要走**:每 provider 一个 crate(抽象成本高)、app-server v1+v2 长期并存(技术债)、V8 嵌入(JVM 生态不现实,可考虑 GraalVM JS)。**1 个新范式技术 POC 候选**:V8 Code Mode(LLM 写 JS 调工具)给 SkillForge skill 编排第二选项,优先级中等。

---

## [2026-05-16] ingest | langfuse (B 模式 source-grounded)

**触发问题**: Langfuse 作为业内最广泛部署的 LLM observability+eval 平台,production loop(trace→dataset→eval→score 单线打通)是怎么工程化的?Score 数据模型(`ScoreSource × ScoreDataType` 3×5 矩阵)和 main/secondary 双队列分流多租户对 SkillForge 的 Eval Pipeline + Prompt A/B + 自动晋升 + Observability 4 个模块意味着什么?

**改动**:
- 新建 [[eval/langfuse]] — 首个 eval-framework 页,9 维 + 双视角 takeaway,版本锁 commit da7d91759 @ 2026-05-15(v3.174.1 semver)
- 新建 [[source/langfuse]] — Web/Worker/packages/shared pnpm monorepo 结构图 + 40+ file:line 锚点
- 新建 [[concept/observability]] — Wide Events 2.0 vs OTel 1.0 三分法,7 维设计空间,Langfuse + Codex 两家 observability 风格对比
- 更新 [[concept/agent-eval]] — 8 维设计空间每个维度补 Langfuse "Platform 路线" 选项,加 Langfuse vs OpenClaw QA Character Eval 对比章节,frontmatter 加 [[eval/langfuse]] 引用
- 更新 [[index]] — Eval 区填充 langfuse + 加 concept/observability + source/langfuse

**关键 B 模式 verified 事实(40+ 个 file:line 锚点见 [[source/langfuse]])**:
- **Wide Events 2.0 哲学**(`.agents/ARCHITECTURE_PRINCIPLES.md:13-32`)11 条原则,引用 Charity Majors + Burmistrov + Langfuse 自家 2026-03-10 blog:"Model observations as the primary analytical unit. A trace is a correlation handle... not the only useful entry point."
- **Score 3×5 矩阵**(`packages/shared/src/domain/scores.ts:4,46`):`ScoreSource = ["API","EVAL","ANNOTATION"]` × `ScoreDataType = ["NUMERIC","CATEGORICAL","BOOLEAN","CORRECTION","TEXT"]`
- **Public API EVAL source 编译期保留**(`scores.ts:13-21`):`PublicApiCreateScoreSourceDomain = z.enum([API, ANNOTATION] as const satisfies readonly ScoreSourceType[])` — 外部 API 永远不能创建 `EVAL` source,防伪造内部 evaluator output(业内独家工程细节)
- **ANNOTATION 必须带 configId**(`scores.ts:30-40`)— zod refine `isAnnotationScoreMissingConfigId`,REST 同步 400 + ingestion 异步 drop 共用一份 predicate
- **`CORRECTION_NAME = "output"` + `TEXT_SCORE_MAX_LENGTH = 500`**(`scores.ts:42-44`)
- **32 QueueName**(`queues.ts:322-358`)+ 32 对应 QueueJobs(`:360-393`),**7 处 main + secondary 双队列分流多租户**(`IngestionQueue`+`IngestionSecondaryQueue` / `OtelIngestionQueue`+`OtelIngestionSecondaryQueue` / `EvaluationExecution`+`EvaluationExecutionSecondaryQueue`),源码注释明确"Separates high-throughput eval projects from other projects"
- **DLQ 一等公民**:`DeadLetterRetryQueue`(`queues.ts:353`)+ `worker/src/services/dlq/` + `dlqRetryQueue.ts` redis 实现
- **Production loop 单线打通**:`web/src/features/evals/server/addDatasetRunItemsToEvalQueue.ts` — production trace → dataset → eval → score 全自动
- **EvalTemplate 版本控制**(`router.ts:142,475`)— eval template 跟 prompt 一样版本化,改 prompt 才能 reproducible 重跑历史 eval
- **OTel 摄入路径**:`OtelIngestionProcessor.ts` + `ObservationTypeMapper.ts`(OTel span ↔ Langfuse observation 映射)
- **存储异构 4 库**:Postgres(metadata + auth)+ ClickHouse(主分析库)+ Redis(queue + cache)+ S3(blob)
- **23+ 框架集成**:OpenAI/Langchain/LlamaIndex/Haystack/LiteLLM/Vercel AI SDK/Mastra/Instructor/DSPy/Mirascope/Ollama/Bedrock/AutoGen/Flowise/Langflow/Dify/OpenWebUI/Promptfoo/LobeChat/Vapi/Inferable/Gradio/Goose/smolagents/CrewAI
- **Ragas 集成**(`web/src/features/evals/components/ragas-logo.tsx`)— RAG eval 行业标杆显式集成
- **`AGENTS.md` symlink + 真源在 `.agents/`** + **`.agents/skills/` 16 个 dogfood Claude Code skill**(Langfuse 自家 codify dev workflow,印证 SkillForge skill 系统方向)

**对其他记录的影响**:
- [[concept/agent-eval]]:8 维设计空间补全(每个维度加 Langfuse "Platform 路线"),OpenClaw runtime framework 跟 Langfuse platform 互补关系明确
- 新 [[concept/observability]]:Wide Events 2.0 vs OTel 1.0 对比 + Langfuse 跟 Codex 两家 observability 风格(中心化 platform vs runtime self-contained)的天然合作 + SkillForge 演进建议(走 Wide Events 路线 + ClickHouse 引入时机 > 10K traces/day + OTel 摄入免费 win)
- [[harness/codex]]:印证 Codex `otel/` crate 5 GOAL metric + 7 compaction enum 跟 Wide Events 哲学一致

**takeaway**: Langfuse = **LLM Eval / Observability 工程化标杆**。**10 条关键借鉴**给 SkillForge:Score 3×5 矩阵抽象、Public API EVAL source 编译期保留、ANNOTATION configId 约束、`TEXT_SCORE_MAX_LENGTH=500` 硬上限、main+secondary 双队列分流多租户、production loop `addDatasetRunItemsToEvalQueue` 单线打通、EvalTemplate 版本控制、OTel ingestion path、Audit log + EntityChangeQueue、DLQ 一等公民。**3 条 SkillForge 不要走**:1 库通吃(Langfuse 4 库异构,Postgres 不该撑全场景)、ad-hoc score 格式(3×5 矩阵已覆盖 95% 场景)、过早做集成生态(Langfuse 23+ 是 3+ 年迭代,SkillForge 早期专注自家 channel)。**3 个 SkillForge 领先点**:Prompt A/B + 自动晋升 first-class(Langfuse 没做)、server-side agent loop 引擎(Langfuse 只 telemetry)、统一 Provider/Channel SPI。**ClickHouse 切换阈值**:Postgres 撑 < 10K traces/day,> 10K 切 ClickHouse(Langfuse production 经验)。

---

## [2026-05-16] ingest | opik (B 模式 source-grounded)

**触发问题**: Opik 作为 Comet ML 出品的老牌商业平台 + Apache 2.0 开源,跟 Langfuse 同赛道但多了 Optimization 第三支柱(`sdks/opik_optimizer/` 7 种算法 + 8 内置 benchmark dataset)。对 SkillForge 的 Eval Pipeline → Self-Evolution → Optimizer 闭环、Assertion+Feedback OOP 多态评分模型、Java/Spring 生态匹配度,3 个决策意味着什么?

**改动**:
- 新建 [[eval/opik]] — 9 维 + 双视角 takeaway,版本锁 commit 72b2f1ca89 @ 2026-05-15(1.7.0)
- 新建 [[source/opik]] — 6 app + 3 SDK monorepo 结构图 + 40+ file:line 锚点
- 新建 [[concept/prompt-optimization]] — 6 维设计空间,Opik 7 算法对照表 + 业内对比(DSPy/PromptBreeder/APE/OPRO)+ Optimizer Chaining 实战推荐 + 跟 fine-tune 对比
- 更新 [[concept/self-evolution]] — 扩展为 4 类:加 类 4 "Algorithm-driven prompt optimization(runtime 无人审)",Opik 7 算法是案例
- 更新 [[concept/agent-eval]] — 8 维设计空间补 Opik 路线选项(TestCase / 8 benchmark / Assertion+Feedback OOP 多态 / LiteLLM / OptimizationResult / Optimizer 闭环),加 Opik vs Langfuse 设计哲学对比 + Opik vs OpenClaw QA Character Eval 互补关系
- 更新 [[concept/observability]] — 加 Opik Java + Spring + MySQL 路线作为 platform observability 第二条路(对照 Langfuse Node + Postgres)
- 更新 [[index]] — Eval 加 opik + 加 concept/prompt-optimization + source/opik

**关键 B 模式 verified 事实(40+ file:line 锚点见 [[source/opik]])**:
- **7 个 Optimizer 算法**(`sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26`)— **README 只提 5 个,源码实际 7 个**(HierarchicalReflectiveOptimizer 跟 HRPO 未在 README):
  - EvolutionaryOptimizer / FewShotBayesianOptimizer / MetaPromptOptimizer / GepaOptimizer / ParameterOptimizer / HierarchicalReflectiveOptimizer / HRPO
  - 统一 `BaseOptimizer(ABC).optimize_prompt()` + `OptimizationResult` 标准返回
  - Optimizer Chaining + MCP Support(`ChatPrompt.tools`)+ Counter Tracking + Multimodal Prompts
- **8 内置 benchmark dataset**(`sdks/opik_optimizer/src/opik_optimizer/datasets/`):gsm8k / truthful_qa / halu_eval / ai2_arc / arc_agi2 / pupa / medhallu / tiny_test — **4 家里独家**
- **Assertion + FeedbackDefinitionModel 双轨 OOP 多态评分**(`apps/opik-backend/.../domain/`):
  - Assertion:`AssertionResultDAO.java` + `AssertionResultService.java` + `AssertionResultMapper.java` 三层(Pytest 风格 pass/fail + reason)
  - Feedback:`BooleanFeedbackDefinitionDefinitionModel.java` + `CategoricalFeedbackDefinitionDefinitionModel.java`(按 dataType 拆 Java 子类)
- **完整 Alert 系统**(4 家独家):`AlertDAO` + `AlertService` + `AlertTriggerConfigDAO` + `AlertTriggerDAO` + Java API `Alert.java` + `AlertEventType.java` + `AlertTrigger.java` + `AlertTriggerConfig.java` + `AlertTriggerConfigType.java` + `AlertType.java`
- **6 个独立 app**(opik-backend Java + opik-frontend React + opik-python-backend + opik-sandbox-executor-python + opik-guardrails-backend + opik-documentation)— **服务化拆分粒度业内最细**
- **`.agents/` 4 家最完整 dogfood**:6 个 builtin agent(planner / build-fixer / code-reviewer / meta-auditor / test-runner / architect,真 Claude Code subagent 标准格式 + tools 白名单 + model 选择)+ 10 个 skill(analytics-instrumentation / clickhouse-best-practices / opik-backend / playwright-e2e / ...)+ 4 个 .mdc rule + commands/comet + mcp.json + `.codex -> .agents` + `.cursor -> .agents` symlink
- **Pytest first-class**(`sdks/python/src/opik/evaluation/test_case.py`)— `TestCase dataclass { trace_id, dataset_item_id, task_output, ... }` + `test_result.py` 配套
- **`opik.sh` 7 种 service profile**:`--infra` / `--backend` / `--guardrails` / `--build` / `--verify` / `--stop` / `--info`
- **3 SDK**:Python + TypeScript + **Ruby via OTel**(4 家里只 Opik 提供 Ruby)
- **`ConnectBridgeService.java`**:Opik ↔ Comet 主平台桥(商业产品策略 + 桥接模式参考)
- **40M+ traces/day** 生产规模标榜 + Java 17 + Spring Boot + ClickHouse + MySQL + Redis + S3

**对其他记录的影响**:
- [[concept/self-evolution]]:从 3 类扩展为 4 类(加 "Algorithm-driven prompt optimization runtime 无人审"),**SkillForge `/evolve` 跟 Opik runtime optimizer 是互补两条路线**(SkillForge meta dev-facing + Opik runtime optimization 不冲突)
- [[concept/agent-eval]]:8 维设计空间补全 Opik "OOP 多态评分 + Pytest first-class + 8 benchmark + Optimizer 闭环"维度,Opik vs Langfuse 设计哲学对比(JS/TS 数据抽象 vs Java OOP 多态)+ Opik vs OpenClaw QA Character Eval 互补
- [[concept/observability]]:platform observability 现在有 Langfuse(Node + Postgres)+ Opik(Java + MySQL)两条路线,**SkillForge Java 生态选 Opik 路线对照度最高**
- [[concept/prompt-optimization]](新):Opik 7 算法是 SkillForge Self-Evolution 第 4 类的具体技术参考,跟 SkillForge `/evolve` meta 路线互补

**takeaway**: Opik = **eval framework + Optimization 闭环工程化标杆**(40M+ traces/day, Java/Spring 生态, Apache 2.0)。**12 条关键借鉴**给 SkillForge:7 算法 Optimizer / 8 benchmark / OptimizationResult 标准化 / Assertion+Feedback OOP 多态 / Pytest first-class / `.agents/` 完整 dogfood / 6 builtin agent / Alert 系统 / Guardrails 独立 service / Sandbox executor 独立 service / Online Evaluation Rules / `opik.sh` 7 种 service profile。**4 条 SkillForge 不要早期照搬**:7 算法完整复刻(工程量大,先 POC 1-2 个 GepaOptimizer + FewShotBayesianOptimizer)/ Feedback OOP 多态(Java 子类爆炸风险,起步用 Langfuse 数据抽象)/ 40M+ traces/day 规模(SkillForge < 10K 阶段不需要)/ Python helper backend 独立 service(优先 Java 内嵌)。**2 条 SkillForge 领先**:`/evolve` meta self-evolution 跟 Opik runtime optimizer 互补 / 统一 SPI 抽象比 Opik 集成 callback 更轻量。**关键决策**:**SkillForge Java 生态匹配度 Opik > Langfuse,可借鉴更多;但 Score 数据模型起步建议用 Langfuse 数据抽象**(避免 Java 子类爆炸),后续按需 evolve 子类。

---

## [2026-05-16] ingest | opencode (B 模式 source-grounded)

**触发问题**: opencode 作为 MIT 开源 + Anomaly Corp(SST 创始人团队)出品的 Claude Code 替代品,LSP first-class + Client/Server 架构 + Effect-TS 重度 functional + 2 builtin agent Tab 切换 — 对 SkillForge 的 server-side Agent 平台 + 多 channel + coding agent 工具集 4 个模块意味着什么?

**改动**:
- 新建 [[harness/opencode]] — 13 维 + 双视角 takeaway,版本锁 1.15.0 @ commit 2d6bedecd
- 新建 [[source/opencode]] — 22 packages monorepo 结构图 + 40+ file:line 锚点
- 新建 [[concept/lsp-integration]] — LSP first-class 集成,opencode 4 家独家,7 维设计空间 + IDE plugin 路线对比 + SkillForge coding agent 真正缺口
- 新建 [[concept/agent-runtime-stacks]] — 5 harness 技术栈横向对比(Python / TS Bun / TS pnpm + pi-sdk / Rust / TS Bun + Effect-TS)+ 5 种编程哲学 + 8 维度技术选型对 agent 设计的影响 + SkillForge Java/Spring 第 6 路线借鉴方向
- 更新 [[concept/multi-agent-isolation]] — 加 opencode 第 4 条路:Tab 切换 builtin + `PlanExitTool` 工具触发跨 agent 切换 + Permission ruleset 跟 agent 绑定 + Agent 自动生成
- 更新 [[concept/context-compaction]] — 加 opencode 路线:6 关键常量 + 7 段 SUMMARY_TEMPLATE + 4 规则 + `PRUNE_PROTECTED_TOOLS=["skill"]` + Bus 事件驱动 + projectors CQRS
- 更新 [[concept/code-mode-orchestration]] — 加 opencode "tool-level mode" 作为 Code Mode 的对立面(Code Mode 是 LLM 写代码,opencode 是 LLM 调切换工具)
- 更新 [[index]] — Harness 加 opencode + 加 concept/lsp-integration + concept/agent-runtime-stacks + source/opencode

**关键 B 模式 verified 事实(40+ file:line 锚点见 [[source/opencode]])**:
- **2 builtin agent + `@general` subagent**(README "Agents")— build(full-access)+ plan(read-only)+ Tab 键切换 + `@general` mention 调用
- **`PlanExitTool` 工具触发跨 agent 切换**(`tool/plan.ts:14-50`)— **业内独家**:plan agent 完成后,LLM 调 `plan_exit` 工具,工具用 `Question.Service.ask` 问用户 Yes/No,Yes 切到 build agent + 用 plan 文件作为起点
- **LSP first-class**(`packages/opencode/src/lsp/` 6 模块 + `tool/lsp.ts`)— 4 家独家,Effect-TS Schema 类型化 LSP 协议(Position / Range / Symbol / DocumentSymbol)
- **`session/compaction.ts:42-48` 关键常量**:`PRUNE_MINIMUM=20_000` / `PRUNE_PROTECT=40_000` / `TOOL_OUTPUT_MAX_CHARS=2_000` / **`PRUNE_PROTECTED_TOOLS=["skill"]`**(独家:skill 调用 first-class memory)/ `DEFAULT_TAIL_TURNS=2` / `MIN_PRESERVE_RECENT_TOKENS=2_000` / `MAX_PRESERVE_RECENT_TOKENS=8_000`
- **`SUMMARY_TEMPLATE` 7 段强制 Markdown**(`compaction.ts:50-87`):Goal / Constraints & Preferences / Progress(Done/In Progress/Blocked)/ Key Decisions / Next Steps / Critical Context / Relevant Files
- **4 条 SUMMARY 规则**(`compaction.ts:88-92`):"Keep every section, even when empty" + "Use terse bullets, not prose paragraphs" + "Preserve exact file paths, commands, error strings, and identifiers" + **"Do not mention the summary process or that context was compacted"**(防 LLM summary 自我提及压缩,业内独家)
- **`Agent.Info` Effect-TS Schema**(`agent.ts:29-46`):`mode: ["subagent", "primary", "all"]` + `permission: Permission.Ruleset` + `model: {modelID, providerID}`
- **`Service.generate()`**(`agent.ts:50-54`)— 用 `ai` SDK `streamObject` 从 description 自动生成 agent `{identifier, whenToUse, systemPrompt}`
- **6 个 prompt template variant**(`prompt/{compaction,explore,scout,summary,title}.txt` + `generate.txt`)
- **OpenTelemetry first-class**(`agent.ts:25` `@effect/opentelemetry/Tracer` 在 agent loop 直接 import)
- **Bus 事件驱动**(`compaction.ts:31-37`)`BusEvent.define("session.compacted", {sessionID})`
- **22 packages monorepo**(opencode + core + tui + desktop + web + llm + plugin + sdk + identity + console + containers + docs + enterprise + extensions + function + http-recorder + script + slack + storybook + ui + web)
- **22 国语言 README**(业内最广,opencode 22 / Opik 11 / Langfuse 4 / Hermes 1)
- **MIT license** + Anomaly Corp(SST 团队 Jay)+ npm `opencode-ai`
- **AGENTS.md 4KB 极简 functional style guide**:Avoid try/catch / Avoid `any` / Avoid else 早返回 / Reduce variable count by inlining / functional array methods / Drizzle snake_case / "ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE"
- **Effect-TS 重度集成**:`Effect.gen` + `Service` + `Layer` + `Context` + Schema-first IO,业内 LLM agent 项目少见
- **`apply_patch` 跟 Codex 同款** patch 协议
- **Provider-agnostic**:`packages/llm/src/{providers,protocols}/` Claude / OpenAI / Google / 本地模型 + OpenCode Zen 商业 proxy
- **Client/Server 架构**:`packages/opencode/src/{server,control-plane,bus}/`,**TUI / Desktop / Web / Mobile 都是 client**(4 家独家)
- **Drizzle ORM + SQLite local-first**(`session.sql.ts` snake_case schema 强约束)
- **SST infrastructure**(`infra/{app,console,enterprise,monitoring,secret,stage}.ts`)— SST 团队 dogfood SST

**对其他记录的影响**:
- [[concept/multi-agent-isolation]]:多 agent 现在有 4 条完整路线 — 完全隔离(Hermes/OpenClaw)/ 隔离+Fork(Claude Code)/ Actor+resume(Codex)/ Tab+工具触发(opencode)
- [[concept/context-compaction]]:加 opencode 极简 SUMMARY_TEMPLATE + 4 规则 + projectors CQRS 路线
- [[concept/code-mode-orchestration]]:opencode 是 Code Mode 的对立面 — Code Mode 把"工具编排"上移到代码层,opencode 把"agent mode 切换"上移到工具层,**两条思路都减少 framework hard-code,把决策上移给 LLM**
- 新 [[concept/lsp-integration]]:LSP first-class 是 SkillForge coding agent 真正缺口
- 新 [[concept/agent-runtime-stacks]]:5 harness 技术栈横向对比,SkillForge Java/Spring 是第 6 条路线

**takeaway**: opencode = **MIT 开源 Claude Code 替代品的工程化标杆**(Anomaly Corp / SST 团队 / Bun + Effect-TS)。**10 条关键借鉴**给 SkillForge:**LSP first-class 集成**(借鉴优先级最高,SkillForge coding agent 真正缺口)/ Client/Server 架构 + 多 client / **Permission ruleset 跟 agent 绑定** / **`PlanExitTool` 工具触发跨 agent 切换** / `PRUNE_PROTECTED_TOOLS=["skill"]` skill first-class memory / 7 段 SUMMARY_TEMPLATE 强制结构 / `apply_patch` 跟 Codex 同款 / **`Service.generate()` 自动生成 agent** / Anthropic prompt cache 抽象 / OpenTelemetry first-class。**3 条 SkillForge 不要照搬**:Effect-TS(Java/Spring 不能用,只能学哲学)/ 22 packages 极细拆分(SkillForge 早期不需要,SST 团队 7+ 年积累)/ opencode 不做 eval/self-evolution/Optimizer(SkillForge 定位更广,不能完全照搬纯 coding agent)。**3 条 SkillForge 领先**:Eval Pipeline + Prompt A/B + 自动晋升 / Self-Evolution `/evolve` / 统一 Provider/Channel SPI 抽象。**关键决策**:**LSP 集成是 SkillForge coding agent next horizon**,opencode `lsp/` 6 模块是开源参考实现,JVM 嵌 LSP client(JSON-RPC over stdio)技术上没问题。

---

## [2026-05-16] meta | wiki 健康检查 + 3 项修补

**触发问题**: 7 ingest 完成(5 harness + 2 eval + 12 concept + 7 source + log 14 entry,8273 行),wiki 整体健康度怎么样?早期 ingest 页是否需要补 schema 后加的字段?

**改动**(3 项非阻塞修补):

1. **`harness/hermes-agent.md` 补版本时效字段**(schema §3.1 升级前 ingest 的页):
   - `last_ingested_version: "0.11.0 @ commit 9a1454060 (2026-04-30)"`
   - `last_ingested_at: 2026-05-15`
   - `community_known_newer: false`
   - `known_post_ingest_features: []`

2. **`harness/openclaw.md` 补版本时效字段**(同上):
   - `last_ingested_version: "2026.4.20 (calver) @ commit 4407df6c03 (2026-04-21)"`
   - `last_ingested_at: 2026-05-16`
   - `community_known_newer: false`
   - `known_post_ingest_features: []`

3. **`harness/claude-code.md` 加"已知近期演进 (post v2.1.88, unverified)"章节**:把 frontmatter `known_post_ingest_features` 列表的 4 个新功能(goal / agent-view / agent-manager / insight)在正文展开,每个标 `[community-knowledge]`,并标注 verify 优先级(HIGHEST insight / HIGH goal / MEDIUM agent-view + agent-manager),给下次新版 ingest 留 actionable 路线图。

**健康检查关键发现**:
- ✅ **27/27 页面全 index 收录** + 0 孤儿页 + 跨页关键事实(Hermes MAX_DEPTH=1 / Codex DEFAULT_AGENT_MAX_THREADS=6 / Hooks 8/16/27 阶梯)100% 一致
- ✅ skillforge_takeaway 全 7 页 ≥ 35 行 + learning_takeaway 三 section(differentiator/strengths/what_done_well)100% 齐全
- ✅ log.md 13/14 entry 三段式(bootstrap entry 不需要触发问题,合理)
- 🔶 doc-only > 3 warning 3 处:claude-code 9 / hermes-agent 6 / openclaw 4(A 模式遗留 + 部分模块未深 verify,未来 re-ingest 时升级)
- ✅ 引用网络密集:hermes-agent 入站 14 / openclaw 13 / claude-code 12 / codex 9 / langfuse 9 / opik 7 / opencode 4(最新)
- ✅ 7 个新 concept 入站 3-5 次(健康分布)

**下次 ingest 建议**:
- **Claude Code 新版可访问时立即 ingest**,优先 verify `insight`(对应 SkillForge `/evolve` meta self-evolution)+ `goal`(对照 Codex Goals)
- doc-only > 3 的 3 页(claude-code / hermes-agent / openclaw)在新版 ingest 时升级 doc-only → verified-from-source

**takeaway**: Wiki 健康整体优秀,无 blocker / error。**结构 100% / 引用 100% / 跨页一致性 100%**,3 项修补已完成把 schema 后加字段补齐 + claude-code community-knowledge 4 个新功能从 frontmatter 列表升级为正文 actionable 章节。下一步可继续 ingest(superpowers / 业内 benchmark)或写 overview thesis 总结 evolving wiki 的核心判断。

---

## [2026-05-16] meta | 写 overview.md 全局 thesis 首版

**触发问题**: 7 ingest 完成后(5 harness + 2 eval + 12 concept),wiki 急需一个全局 thesis 把跨页 insight 汇总,作为后续 ingest 的回灌锚点 + 后续读 wiki 时的"5 分钟内看完核心判断"入口。

**改动**:
- 新建 [[overview]] — 全局 thesis 首版,5 个 section + 6 大块内容
- 更新 [[purpose]] — Evolving thesis section 从"初始为空"占位升级为 1 段精炼总结(7 学习判断 + 4 类 SkillForge 决策),指向 [[overview]] 完整版
- 更新 [[index]] — overview.md 从"未生成"标记升级为"🎯 全局 thesis 首版"

**[[overview]] 内容结构**(6 大块):
1. **Wiki 状态 snapshot**(2026-05-16):27 页 / 8345 行 / 15 log entry / 100% 跨页一致性 / 调研对象覆盖度
2. **学习视角 7 个核心判断**:
   - 判断 1:Multi-agent 已分化为 4 条路线(隔离 / Fork / Actor+resume / Tab+工具触发)
   - 判断 2:Self-evolution 4 类形态(runtime 提炼 / meta dev-facing / runtime budget 内嵌 / algorithm-driven prompt optimization)
   - 判断 3:Compaction 7 维设计空间各家路线丰富(Codex 训练对齐 + opencode SUMMARY_TEMPLATE)
   - 判断 4:Hooks 数量阶梯反映工程文化(Codex 8 < Hermes 16 < Claude Code 27)
   - 判断 5:Wide Events 2.0 vs OTel 1.0 三分法(LLM observability 事实路线)
   - 判断 6:Code Mode V8 isolate 是真新范式但工程门槛高(OpenAI 独家)
   - 判断 7:`.agents/` dogfood 程度跟 LLM agent 友好度强相关
3. **SkillForge 视角 8 个关键决策建议**:
   - 优先级 1(必做短期):Score 数据模型起步 Langfuse 抽象 / 8 hook + output_spill / Main+Secondary 双队列
   - 优先级 2(重要中期):LSP first-class / Goals + token_budget / Production loop 单线打通
   - 优先级 3(演进长期):`/evolve` + Opik Optimizer 互补 / `.agents/` dogfood + AGENTS.md
   - 不要走的路 5 条 + SkillForge 已领先 3 条
4. **Wiki 维护原则 6 条**(从实践提炼):B 模式必须 / 双视角 takeaway 不可偏废 / 版本时效 + 4 级证据让 wiki 自带 "知道自己 stale" 能力 / 跨页一致性靠 grep + 关键常量 / Iron Law 写入边界 / Two-Step Chain-of-Thought ingest
5. **Open Questions / 待 ingest 候选**:紧急(Claude Code insight verify)+ 用户后续候选(superpowers / SWE-bench / τ-bench / Inspect AI / DSPy 等 9 个)+ 6 个 wiki 当前未覆盖维度(Skills 横向对比 / MCP 集成深度 / Permission Sandbox 模型 / Auth Identity / Plugin SDK / Cancellation)
6. **元判断**(meta-thesis):Wiki 价值 = 跨 harness 横向连接密度 × 每个事实 verified 程度 × 双视角(学习+SkillForge)覆盖完整度

**takeaway**: Wiki 从 "ingest 产物集合" 升级为 "evolving thesis 活产物"。后续任何 ingest 都应**回灌 [[overview]]**:新发现支持现有判断 → 强化;反证 → 修正;新增维度 → 扩展。**核心元判断**:wiki 价值 = 跨 harness 横向连接密度 × verified 程度 × 双视角覆盖度,任何后续 ingest 应强化这 3 个维度。

---

## [2026-05-16] ingest | 7 项目批量(SWE-bench / tau-bench / DSPy / Promptfoo / Ragas / Phoenix / Helicone / LangChain)

**触发问题**: 用户选定的"评测 + 归因"7 项目集体 ingest 到 wiki,作为后续 SkillForge 决策的指导手册。每个项目深度 source + eval 一对页面 + 跟 SkillForge 当前状态校准(基于 docs/delivery-index.md 实际数据)。

**改动**:

### 新建 14 页(7 source + 7 eval)
- [[source/swe-bench]] + [[eval/swe-bench]] — 业内 coding benchmark 标杆,4 变体 + 8 语言 + Regression-aware 4 test category
- [[source/tau-bench]] + [[eval/tau-bench]] — Multi-turn Tool-Agent-User benchmark + LLM-simulated user + **`auto_error_identification.py` 业内首批 benchmark 自动归因**(对照 SkillForge ATTRIBUTION-AGENT 同方向)
- [[source/dspy]] + [[eval/dspy]] — Stanford prompt-as-program framework,15 optimizer + 13 predict module + `UNSAFE_LM_STATE_KEYS` 防 prompt injection
- [[source/promptfoo]] + [[eval/promptfoo]] — CLI-first eval + 50+ assertion(含独家 trace-level)+ 25+ red-team 策略 + GitHub Action code-scan
- [[source/ragas]] + [[eval/ragas]] — RAG eval 标杆 + 27 内置 metric + 被 13 平台集成 + Multi-Modal RAG 独家
- [[source/phoenix]] + [[eval/phoenix]] — Arize observability + Score.explanation per-evaluation 归因 + OpenInference semconv 主推 + MCP server 一键 Cursor install
- [[source/helicone]] + [[eval/helicone]] — AI Gateway + Observability 二合一 + OpenAI API baseURL 兼容 proxy + 100+ models 1 API key
- [[source/langchain]] + [[eval/langchain]] — 业内规模最大 framework + 5 大产品生态 + v1 重写 + Callback System 业内事实标准

### 更新 index.md
- Eval 区从 2 个加到 **9 个**(langfuse / opik / swe-bench / tau-bench / dspy / promptfoo / ragas / phoenix / helicone / langchain)
- Source 区从 6 个加到 **13 个**

### 关键 SkillForge 校准发现(基于 SkillForge docs/delivery-index.md 实际数据)
- **R7 Production loop 已在做**(SkillForge 飞轮 V1+V2+V3+V4 已落齐,剩 V5+V6 pending)
- **R5 LSP 已在需求名单**(deferred 名单)
- **R10 `/evolve` + Optimizer 互补**:SkillForge 走 LLM 驱动 + 人审路线(`MEMORY-LLM-SYNTHESIS`/`PromptImproverService`/`SkillDraftService`/**`ATTRIBUTION-AGENT`**)
- **SkillForge ATTRIBUTION-AGENT** 已在 Phoenix `Score.explanation` 之上一个 layer(production-level cross-session 自动归因 vs Phoenix per-evaluation LLM rationale)
- **tau-bench `auto_error_identification.py` `FaultAuthor × FaultType` typed enum schema** 可借鉴给 SkillForge ATTRIBUTION-AGENT 输出加 typed enum(对照当前 free-form description)

### 关键 verified 事实(精选)
- **SWE-bench `FAIL_TO_PASS/FAIL_TO_FAIL/PASS_TO_PASS/PASS_TO_FAIL` 4 test category**(`constants/__init__.py:40-43`)— regression-aware 评分模型
- **tau-bench `LLMUserSimulationEnv`**(`envs/user.py:36-54`)— SkillForge EVAL-DYNAMIC-USER-SIM Phase 2/3 直接参考
- **tau-bench `auto_error_identification.py` `FaultAuthor` Enum `USER/AGENT/ENVIRONMENT` + `FaultType` Enum 4 类** — 业内首批 benchmark 自动归因
- **DSPy 15 optimizer**(`teleprompt/__init__.py:1-18`)+ **`UNSAFE_LM_STATE_KEYS = {"api_base", "base_url", "model_list"}`**(`predict.py:22`)防 prompt injection
- **Promptfoo 50+ assertion**(`src/assertions/`)— 含独家 `traceErrorSpans/traceSpanCount/traceSpanDuration` trace-level assertion
- **Promptfoo 25+ red-team 策略**(`src/redteam/strategies/`)— crescendo/GCG/GOAT/Hydra/multilingual/mathPrompt/promptInjections
- **Ragas 27 metric** + **RAG 核心三角**(Faithfulness + Context Recall + Context Precision)+ 被 13 平台集成
- **Phoenix `Score` dataclass**(`evaluators.py:159+`):score + name + label + **explanation**(LLM rationale)+ metadata + **kind**(human/llm/heuristic/code)+ **direction**(maximize/minimize/neutral)
- **Phoenix `include_explanation: bool = True`**(`evaluators.py:720`)LLM-as-judge 默认开 rationale
- **Helicone AI Gateway** `https://ai-gateway.helicone.ai` OpenAI API baseURL 兼容 + 100+ models + 1 API key
- **LangChain v1.3.1**(`libs/langchain_v1/pyproject.toml`)2026 重写 + `init_chat_model("provider:model")` 字符串路由 + `Runnable | pipe` 业内标志抽象

### 关键 SkillForge actionable 借鉴(已 deduplicate SkillForge 已有的)
- **`UNSAFE_LM_STATE_KEYS` 防 prompt injection 模式**(DSPy,业内独家)— SkillForge 加载用户 prompt/state 时 sanitize
- **`include_explanation: bool = True` LLM-as-judge 默认开 rationale**(Phoenix)— SkillForge LLM-as-judge metric 应默认带 rationale,给 ATTRIBUTION-AGENT 输入更丰富
- **`KindType` 4 类 + `DirectionType` 3 类**(Phoenix)— SkillForge EvalTaskEntity 加 kind + direction 字段
- **3 个 trace-level assertion**(Promptfoo `traceErrorSpans/traceSpanCount/traceSpanDuration`)— SkillForge OBS-2/OBS-4 trace 模型已就绪,可直接加
- **RAG 核心三角**(Ragas Faithfulness/ContextRecall/ContextPrecision)— SkillForge 用户 RAG 场景应集成
- **`FaultAuthor + FaultType` typed enum schema**(tau-bench)— SkillForge ATTRIBUTION-AGENT 输出 schema 加 typed enum
- **`buildPricesOpenRouter.py` 自动同步价格**(Helicone)— SkillForge 模型定价自动化
- **Callback System / Tracer 抽象**(LangChain)— SkillForge 自家 LifecycleHooks N3 已交付,可对照 LangChain callback 模板确保业内兼容

**takeaway**: 7 项目集体 ingest 完成,wiki 现在覆盖 **5 harness + 9 eval(原 2 + 新 7)+ 12 concept + 13 source**。基于 SkillForge 实际状态(用户告知 + docs/delivery-index.md 读取),**SkillForge 已超过业内多数项目几个 layer**(production loop 飞轮 / ATTRIBUTION-AGENT / 多 surface 优化 / Skill marketplace 一等公民 / MULTIMODAL / MCP / PROMPT-CACHE-MVP)。剩余可借鉴 actionable 工程细节集中在: `UNSAFE_LM_STATE_KEYS` 防 prompt injection / `include_explanation` 默认开 / `KindType`+`DirectionType` Score schema / Trace-level assertion / RAG 三角 / `FaultAuthor`+`FaultType` 归因 enum / 自动同步价格 / Callback System 业内兼容。

---

## [2026-05-16] ingest | 5 memory framework 批量(mem0 + Letta + MemPalace + LangMem + Zep+Graphiti)

**触发问题**: 用户要看记忆相关项目(含"记忆宫殿"),GitHub 上 5 个最知名 memory framework 集中 ingest,作为 SkillForge memory 系统(MEMORY-v2 + DREAM-CONSOLIDATION + LLM-SYNTHESIS)演进的指导。

**改动**:

### 新建 10 页(5 source + 5 eval)
- [[source/mem0]] + [[eval/mem0]] — mem0 v2.0.2,业内最广用 memory framework + 2026-04 新算法 LongMemEval 94.8 + **`mem0/openclaw/dream-gate.ts` 跟 OpenClaw plugin 联动**(重大发现)
- [[source/letta]] + [[eval/letta]] — Letta(formerly MemGPT)0.16.8,Berkeley + Agent OS + virtual context + Inner Thoughts + Heartbeat + **`block_manager_git.py` git 版本控制 memory block**(业内独家)+ READ_ONLY_BLOCK
- [[source/mempalace]] + [[eval/mempalace]] — MemPalace bb31396,**LongMemEval 业内最高分**(raw 96.6% / Hybrid 98.4% / Hybrid+rerank ≥99%)+ Verbatim 存储 + Wings/Rooms/Drawers + 零 API call + 29 MCP tools + Auto-save Claude Code hooks · ⚠️ **`mempalace.tech` 是 imposter site**(README CAUTION)
- [[source/langmem]] + [[eval/langmem]] — LangMem 0.0.30,LangChain ecosystem 内 memory utilities + Background ReflectionExecutor(Local + Remote)+ `namespace=tuple` + Prompt Optimization 一体
- [[source/zep-graphiti]] + [[eval/zep-graphiti]] — Zep faf2ace(legacy 归档,主转 Cloud SaaS)+ Graphiti 0.29.0(arxiv 2501.13956)— **Temporal KG + validity windows + Episodes provenance + 4 graph DB driver(Neo4j/FalkorDB/Kuzu/Neptune)+ Cross Encoder reranking** 业内独家

### 新建 1 个 concept
- [[concept/agent-memory]] — **10 维设计空间 + 5 framework + SkillForge 横向对比矩阵 + 10 条 actionable 借鉴清单(R-MEM-1 ~ R-MEM-10)**

### 更新 index.md
- Eval 区从 9 个加到 **14 个**(原 9 + 新 5)
- Source 区从 13 个加到 **18 个**
- Concept 区加 [[concept/agent-memory]]

### 关键 verified 事实(精选)
- **mem0 v2 LongMemEval 94.8**(+27 from old)+ LoCoMo 91.6 + BEAM(1M) 64.1 — 2026-04 新算法 5 大变化(Single-pass ADD-only / Agent-generated facts first-class / Entity linking / Multi-signal retrieval / Temporal Reasoning)
- **mem0 `Memory` + `AsyncMemory` 双类**(`memory/main.py:331, :1795`)
- **MemPalace LongMemEval R@5 raw 96.6% / Hybrid v4 98.4% / Hybrid+rerank ≥99%**(业内最高) + LoCoMo R@10 88.9% + ConvoMem 92.9% + MemBench R@5 80.3%
- **MemPalace 29 MCP tools**(业内 memory framework MCP 集成最深)+ Temporal KG SQLite + Auto-save Claude Code hooks
- **Letta MemGPT 5 大概念**:Virtual context + Memory hierarchy + Inner Thoughts(`INNER_THOUGHTS_KWARG`)+ Function-call memory ops + Heartbeat(`REQ_HEARTBEAT_MESSAGE`)
- **Letta `block_manager_git.py` git 版本控制 memory block**(业内独家)+ `READ_ONLY_BLOCK_EDIT_ERROR` read-only block
- **LangMem 3 大组件**(Core API + Memory tools agent-facing + Background ReflectionExecutor 异步)+ Prompt Optimization 4 种 kind 一体
- **Graphiti Temporal KG**(arxiv 2501.13956,2025-01)+ Validity windows + Episodes provenance + 4 graph DB driver + Cross Encoder reranking

### 3 大重大发现(影响 wiki 其他章节)

1. **mem0 + OpenClaw 协作 ecosystem**:`mem0/openclaw/dream-gate.ts` + `openclaw-plugin-sdk.d.ts` + `openclaw.plugin.json` — mem0 仓库内置 OpenClaw plugin SDK,**OpenClaw Memory Dreaming 实际可能是 mem0 + OpenClaw 共建**(待 verify [[harness/openclaw]] `extensions/memory-core/src/dreaming.ts` 是否 import mem0)
2. **`mempalace.tech` 是 imposter site**(MemPalace README 顶部 CAUTION 警告)— 之前 WebSearch 看到的 "Milla Jovovich + Ben Sigman 创建" 信息来自 imposter,**作废**。真实创建者待 verify
3. **Zep 主仓库已转 Cloud-only**(`legacy/` Go source 归档,主 source 转 Zep Cloud SaaS),**Graphiti 是 OSS 接续的 Temporal KG framework** — 跟其他 framework 同档活跃

### SkillForge 校准(基于 docs/delivery-index.md)

SkillForge 当前 memory(已交付):
- **N1 Memory 向量检索**(2026-04-16)
- **MEMORY-v2 写入/召回/淘汰**(2026-04-27)
- **MEMORY-DREAM-CONSOLIDATION**(2026-05-08,embedding dedup + archived_reason)
- **MEMORY-LLM-SYNTHESIS**(2026-05-11,LLM 驱动 4 类 proposal + 人审 gate)

**SkillForge 走的是第 6 条 memory 路线**(其他 5 framework 都没有的"人审 gate" + LLM 提议)。

### 10 条 actionable 借鉴清单(R-MEM-1 to R-MEM-10,按优先级)

**优先级 1(必做,1-2 周)**:
- R-MEM-1:跑 LongMemEval benchmark(SkillForge vs mem0 94.8 / MemPalace 96.6 raw)
- R-MEM-2:Validity windows(`valid_from + valid_to`,自动 invalidate 旧事实)
- R-MEM-3:Episode provenance(`source_event_id`,每 fact 追溯到产生它的 session/message)

**优先级 2(重要,1 个月)**:
- R-MEM-4:Entity linking(独立 entity store collection)
- R-MEM-5:Multi-signal retrieval(semantic + BM25 + entity)
- R-MEM-6:Hierarchical scope(Wings/Rooms/Drawers 三层)
- R-MEM-7:git 版本控制 memory(`block_manager_git.py`)

**优先级 3(演进,长期)**:
- R-MEM-8:Cross encoder reranking
- R-MEM-9:Read-only block(防 agent 误改 hardcoded 配置)
- R-MEM-10:Ontology(prescribed + learned)

### SkillForge 已领先 4 条
1. MEMORY-LLM-SYNTHESIS 人审 gate(业内独家,5 framework 都没有)
2. MEMORY-DREAM-CONSOLIDATION embedding dedup + archived_reason(其他 framework 累加)
3. Production loop + 飞轮 + ATTRIBUTION-AGENT(其他 framework 是 memory layer,SkillForge 是 platform)
4. Skill marketplace + 多 channel + multi-provider SPI

### SkillForge 不该走的路 5 条
1. 完整集成 Graphiti / mem0 / Letta(infrastructure + 语言栈成本高)
2. 切 Verbatim 存储(SkillForge dedup 已优化路线)
3. 切 ADD-only 累加(SkillForge 人审 gate 路线更安全)
4. 引入 Neo4j 等 graph DB(SkillForge pgvector 已够)
5. 依赖 LangChain ecosystem

**takeaway**: 5 memory framework ingest 完成,wiki 现在覆盖 **5 harness + 14 eval(原 9 + 新 5)+ 13 concept + 18 source**。**SkillForge memory 系统已是业内独家"LLM-SYNTHESIS proposal + 人审 gate + DREAM-CONSOLIDATION dedup"第 6 路线**,跟其他 5 framework 完全互补。**3 大重大发现**(mem0+OpenClaw 联动 / mempalace.tech imposter / Zep Cloud-only)修订 wiki 已有页面。**10 条 actionable 借鉴**按优先级排好,**R-MEM-1 跑 LongMemEval benchmark** 是 next action(决定哪些 R 真的要做)。

---

## [2026-05-16] meta | SkillForge memory B 模式 verify(从 doc-only 升级)

**触发问题**: R-MEM 10 条清单第一版基于 SkillForge `docs/delivery-index.md` 推断(doc-only),用户问"为什么是推断" — 启动 B 模式 verify(对照 Hermes ingest 时 A 模式 hallucinate MAX_DEPTH=2 vs 实际 1 的教训)。

**改动**:
- 更新 [[concept/agent-memory]] — SkillForge 校准章节从"推断"升级到 `[verified-from-source]` + file:line 锚点
- 修订 R-MEM-5 优先级:从"必做(1-2 周)"改为"R-MEM-4 配套(< 半周)"— SkillForge 已实现 Hybrid + RRF
- 新发现 SkillForge 已领先 6 条(原 4 条)

**B 模式 verify 关键源文件**(7 文件 / 4 migration):
- `skillforge-server/src/main/java/com/skillforge/server/entity/MemoryEntity.java`(282 行)
- `skillforge-server/src/main/java/com/skillforge/server/service/MemoryService.java`(`:130, :139, :651, :853, :886, :711-730, :940`)
- `skillforge-server/src/main/java/com/skillforge/server/repository/MemoryRepository.java`
- `V7__memory_vector_search.sql`(pgvector HNSW + tsvector GIN 双索引)
- `V25__sprint1_memory_snapshot_visibility_attribution.sql`(t_memory_snapshot batch rollback)
- `V29__memory_v2_status_importance.sql`(status/importance/last_score)
- `V66__memory_archived_reason.sql`(DREAM-CONSOLIDATION)
- `V68__memory_llm_synthesis.sql`(LLM-SYNTHESIS schema 完整 + t_memory_proposal)
- `V80__create_optimization_event.sql`(ATTRIBUTION-AGENT 因果链 state machine)
- `V83__multi_surface_attribution_link.sql`(V4 multi-surface 飞轮)
- `V85__create_simulator_trial_seed_user_simulator.sql`(EVAL-DYNAMIC-USER-SIM Phase 1)

**重大发现**(B 模式 verify 后 wiki 没记录的 SkillForge 能力):
1. **Hybrid FTS + Vector + RRF retrieval 已实现**(`MemoryService.java:853-908`)— RRF_K=60 业内标准 + FTS_LIMIT=20 + VEC_LIMIT=20 + L1_TOP_K=8
2. **L0/L1 类型分级 retrieval 业内独家**(`MemoryService.java:711-730`)— L0_TYPES=[preference,feedback] 头部注入 + L1_TYPES=[knowledge,project,reference] Hybrid recall
3. **`memory_kind` 三态分类**(observation/reflection/optimized,V68 + MemoryEntity.java:82-104)— 跟 t_memory.type 业务分类正交,业内独家
4. **V80 t_optimization_event 完整 state machine**(proposal_pending → proposal_approved → candidate_created → ab_running → ab_passed → canary_started → promoted → verified)— ATTRIBUTION-AGENT 因果链工程深度业内独家
5. **V85 EVAL-DYNAMIC-USER-SIM Phase 1** — 对照 tau-bench `LLMUserSimulationEnv`,SkillForge 已开始做

**校准结果**:
- ✅ R-MEM-1/2/3/4/6/7/8/9/10 确认真差距(9 条)
- 🟢 R-MEM-5 大部分已完成,只缺 entity signal(降优先级,R-MEM-4 配套)
- ✅ SkillForge 已领先从 4 条扩展到 **6 条**(新增 memory_kind 三态 + L0/L1 分级 hybrid)

**对其他记录的影响**:
- 修订 [[concept/agent-memory]] SkillForge 校准章节 + R-MEM-5 优先级 + 已领先 6 条
- Wiki 之前的 "SkillForge 推断纯 semantic" 是 **A 模式 hallucinate**(跟 Hermes MAX_DEPTH=2 同类错误),已修正

**takeaway**: B 模式 verify 是 wiki 核心方法 — **没读源码就不要写"推断"事实**。这次 verify 把 R-MEM 清单从 80% doc-only 推断升级到 95% verified,真差距清单更可靠。**下次任何对 SkillForge 的判断都应先 grep 源码再写**(避免 A 模式 hallucinate)。

---

## [2026-05-16] ingest | agentops(B 模式 source-grounded)

**触发问题**: 用户问归因正常流程,要看 AgentOps(专门 agent observability,跟其他 LLM-level observability 不同抽象层)— 跟 [[concept/observability]] 已有 Phoenix / Langfuse / Opik / Helicone 4 路线补完,**形成 5 路线 observability landscape**。

**改动**:
- 新建 [[source/agentops]] — 30+ file:line 锚点
- 新建 [[eval/agentops]] — 9 维 + 双视角 takeaway,**LongMemEval 不公布**(observability 不是 eval framework)
- 更新 [[index]] — Eval 区 + Source 区分别加 1 个

**关键 verified 事实**:
- **12 SpanKind**(`agentops/semconv/span_kinds.py:7-19`):WORKFLOW / SESSION / TASK / OPERATION / AGENT / TOOL / LLM / CHAIN / TEXT / **GUARDRAIL** / HTTP / UNKNOWN — 业内独家含 GUARDRAIL 一等公民
- **Legacy 3 sub-kind**(`span_kinds.py:23-43`):**`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION`** — 业内独家 agent action 三态归因维度
- **4 装饰器**:`@session / @agent / @operation / @task`(`sdk/decorators/__init__.py`)
- **9 agentic framework instrumentation**(ag2 / agno / crewai / google_adk / haystack / langgraph / openai_agents / smolagents / xpander)+ 5 LLM provider + **mem0 instrumentation**(`providers/mem0/` 4 文件)
- **TraceContext class**(`sdk/core.py:40-56`)+ **`is_init_trace` flag** 区分 framework 自动 vs 用户显式
- **`weakref.WeakKeyDictionary()`** OTel context propagation 工程模板(`OpenTelemetry.md`)
- **完整 OSS stack**:SDK + Dashboard(Next.js + Cypress + Bun)+ ClickHouse + OTel Collector + Grafana + compose.yaml
- **`llms.txt` 顶层 LLM-readable doc**(类 robots.txt 但给 LLM)
- **`uv` 包管理 + `justfile`** Astral / Rust 生态现代 stack

**重大发现**:**mem0 是 memory framework 协同 hub**!继上次 mem0 + OpenClaw 联动发现(`mem0/openclaw/dream-gate.ts`),本次新增 **mem0 + AgentOps 联动**(`agentops/instrumentation/providers/mem0/` 4 文件完整 instrument mem0)。

**5 路线 observability landscape 完整形成**(已加 [[concept/observability]] 维护):
- Layer A trace + 人工探索:**AgentOps**(agent-level)/ Phoenix / Langfuse / Opik(LLM-level)/ Helicone(Gateway-first)
- Layer B per-evaluation rationale:Phoenix Score.explanation / Ragas / Opik Assertion / Promptfoo llmRubric
- Layer C memory KG provenance:Graphiti Episodes / MemPalace / mem0(隐式)
- Layer D fault typed enum:tau-bench FaultAuthor × FaultType / DSPy GEPA
- Layer E production 自动归因因果链:**SkillForge ATTRIBUTION-AGENT V3 + V80**(业内独家)

**SkillForge 3 大可借鉴 actionable**:
1. **`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 3 sub-kind**(agent action 三态归因)— SkillForge OBS-2 `t_llm_span.kind` 加 agent-level 细分,给 ATTRIBUTION-AGENT 归因输入更精细
2. **`GUARDRAIL` 一等公民 SpanKind** — SkillForge SafetyHook + behavior_rule 加 guardrail span 标识
3. **`@operation` / `@task` 显式层级** — OBS-1 session × trace 合并加 task / operation 显式层级

**takeaway**: AgentOps 是**业内独家专门 agent-level observability** — 不是 LLM-level / trace-level,而是 agent semantic 层级(SESSION → AGENT → TASK / OPERATION 4 层)。**对 SkillForge 真价值是 agent action 三态归因 + GUARDRAIL 一等公民 SpanKind**(给 ATTRIBUTION-AGENT V3 输入更精细 typed enum)。**没有自动归因(Layer E)**,只提供归因 raw materials(Layer A)。SkillForge V3 ATTRIBUTION-AGENT 走得更远(production 自动归因 + 因果链 + multi-surface 闭环)。

---

## [2026-05-16] verify+concept | R-HARNESS B 模式 verify SkillForge 源码 + 新建 [[concept/agent-harness]]

**触发问题**: 5 harness ingest 后(Hermes / Claude Code / OpenClaw / Codex / opencode),需要给 SkillForge 提 actionable 借鉴清单 R-HARNESS-1~R-HARNESS-10。跟 R-MEM 一样,doc-only 推断不够 — 需要 grep SkillForge `skillforge-core/` + `skillforge-server/` 源码把"推断"升级为 `[verified-from-source]`。

**改动**:
- 新建 [[concept/agent-harness]] — **10 维 cross-harness 设计空间 + R-HARNESS-1~R-HARNESS-10 verified clipboard + SkillForge 已领先 10 条 + 不该走的路 5 条**(283 行)
- 更新 [[index]] — Concept 区加 [[concept/agent-harness]] 行

**B 模式 verify 7 项关键词**(`grep -rEni "..." skillforge-core skillforge-server --include="*.java" | grep -v test`):
1. LSP / pylsp / tsserver / jdt.ls / textdocument — **业务代码 0 命中**(只有 `toolsPrompt` / `writeToolSpan` 名字 hit,与 LSP 无关)
2. goals / token_budget — **AgentLoopEngine.java:1363-1384** 4-axis exit + **EvalScenarioEntity.java:76-244** businessGoal field(V84+)
3. prune / compact / protected_tool — **CompactableToolRegistry.java:9-15** _"Tools NOT in the whitelist (e.g. **Memory, SubAgent**) are left untouched"_ + per-agent override
4. SubAgent / spawn_agent — **完整生产系统**:`subagent/` 4 module + `SubAgentRunEntity` + **`SubAgentStartupRecovery.java:35-95` 服务重启 crash recovery**(业内罕见)
5. AGENTS.md / cursorrules — **0 命中**(只支持 CLAUDE.md,`SystemPromptBuilder.java:34` 拼接顺序)
6. HookEvent — **`HookEvent.java:13-18` 仅 5 events**(SESSION_START / USER_PROMPT_SUBMIT / POST_TOOL_USE / STOP / SESSION_END,**缺 PreToolUse / PreCompact / Notification**)
7. SkillSource — **`SkillSource.java:20-28` 8 enum values**(UPLOAD / SKILL_CREATOR / DRAFT_APPROVE / EVOLUTION_FORK / SKILLHUB / CLAWHUB / GITHUB / FILESYSTEM,**业内最多**)

**verify 后校准结果**:
- 🔴 P1 必做 **2 条**(R-HARNESS-1 LSP / R-HARNESS-6 PreToolUse+PreCompact hooks),工作量 1-3 周/项
- 🟡 P2 配套 **3 条**(R-HARNESS-2 Goals 协议级 < 半周 / R-HARNESS-4 Fork 模式 < 1 周 / R-HARNESS-5 AGENTS.md+session migration 1-2 周)
- ✅ 已实现 **2 条**(R-HARNESS-3 pruning whitelist **已 ≥ opencode** + per-agent override / R-HARNESS-7 SkillSource 8 enum **业内最多**)— 原推断错误,校准
- 🔵 P3 不做 **3 条**(R-HARNESS-8 Code Mode V8 / R-HARNESS-9 Memory Wiki / R-HARNESS-10 27 hooks 全套)

**SkillForge 已领先 10 条**(从 R-MEM verify 后 6 条扩到 R-HARNESS verify 后 10 条):8 SkillSource enum / DRAFT_APPROVE+EVOLUTION_FORK 治理 / SubAgent crash recovery / Async SubAgent result delivery / per-agent compactable override / `LlmCallContext` 跨线程 explicit pass / 4-axis loop exit / `businessGoal` eval scenario / CLAUDE.md context segment / Agent-authored hooks 三层(user/system/agent)。

**takeaway**: R-HARNESS verify 揭示 SkillForge **远比推断版本成熟** — SubAgent 已是 production 系统含 crash recovery,Skill 装载源 8 enum 业内最多,Compactable whitelist 含 per-agent override 比 opencode 强。**真正缺口收敛到 2 条**(LSP first-class + PreToolUse/PreCompact hooks),工作量从原推断 10+ 项降到 5 项可做(2 P1 + 3 P2 配套)。**校准方法论**:doc-only 推断常常**严重低估** SkillForge 现状(R-MEM 校准 4→6,R-HARNESS 校准 6→10),所有 actionable 清单必须 B 模式 verify 后再交付。

---

## [2026-05-16] concept | runtime-context-injection — depth-first 写法实验

**触发问题**: 用户反馈 wiki 深度不够 — _"感觉 Wiki 其实并不详细深入,感觉还是参考的意义,并不能实际提升和学习"_。同时点名 3 个具体担心:tool 压缩 / `system_reminder` / dream memory。盘点后发现 `<system-reminder>` 是真 gap(命中全是 Codex `InitialContextInjection`,Claude Code SR 完全没单独整理),决定用这页做 depth-first 写法实验。

**改动**:
- 新建 [[concept/runtime-context-injection]](358 行)— **depth-first 模式**:不堆 catalog,讲机制 + 工程动机 + 失败模式 + 抗 hallucination
- 更新 [[index]] — Concept 区加入

**关键 verified 事实**(grep Claude Code v2.1.88 源码,20+ file:line 锚点):
- **协议级契约**:`prompts.ts:131-134/190` LLM 被显式训练 _"tags contain useful info, automatically added by system, bear no direct relation to specific tool results"_ — 3 个 contract bit 防 3 种失败
- **机制三件套**:`messages.ts:3097-3099` wrapInSystemReminder + `:1797-1817` ensureSystemReminderWrap 幂等 + `:1819-1860` smooshSystemReminderSiblings 合并 sibling 进 tool_result(防 Gap E/F + A/B `sai-20260310-161901` 验证)
- **23 attachment 5 大功能类**:A Mode 维持(plan/auto 三档/两档)/ B Memory 抗老化 / C Skill 发现 / D Task 协调 / E Team+SubAgent+8 Hook 类型
- **memoryAge staleness 警告**(`memdir/memoryAge.ts:33-42`):_"the citation makes the stale claim sound more authoritative, not less"_ — 反 "citation 越精确 stale 时越危险" 悖论,核心 fix:**memory 自己带 age + freshness note,且 age 必须用人类可读"47 days ago"而非 ISO timestamp**(_"Models are poor at date arithmetic"_)
- **5 budget 常量带 prod reasoning**(`attachments.ts:265-293`):MAX_MEMORY_BYTES=4096 / MAX_SESSION_BYTES=60KB / TURNS_BETWEEN_ATTACHMENTS=5 / FULL_REMINDER_EVERY_N_ATTACHMENTS=5 / TURNS_BETWEEN_REMINDERS=10 — 每个数字配 "production 观察 → 工程决策" 注释
- **跟 prompt cache 协同**(`messages.ts:3711-3713`):稳定 header attachment-creation time 存,后续不重算 — _"rendered bytes are stable across turns (prompt-cache hit)"_

**对 Codex `InitialContextInjection` 校准**:**完全不同机制**!Codex 是 _"compact 完成后 summary 注入位置"_(BeforeLastUserMessage / DoNotInject),Claude Code SR 是 _"runtime 任意时刻 in-context augment"_。前者 1 种内容 + 训练对齐位置,后者 23+ 内容 + 协议级训练契约。**这两个混在一起是常见误读**,本页第 §6 显式校准。

**SkillForge R-INJECT 借鉴清单**:
- 🔴 P1:R-INJECT-1 SR 协议层(`grep system-reminder skillforge-server skillforge-core` 业务代码 0 命中)/ R-INJECT-2 memoryAge staleness 警告(防 hallucination 直接打到 SkillForge MEMORY 路径)
- 🟡 P2:R-INJECT-3 task/mode reminder cadence / R-INJECT-5 prompt cache 协同 audit / R-INJECT-6 hook→reminder channel

**takeaway**:这页写完用户反馈得到正面验证 — depth-first 比 wide-first 更难写但**更有学习价值**。3 个关键洞察:1) system prompt 是 _"训练 LLM 怎么读 tool result"_ 而非 _"告诉 LLM 你是助手"_,Anthropic 5 行 prompt 撬动整个 SR 协议;2) **production 常量带注释 reasoning** 是 _"真在 prod 跑过"_ 最快信号(比 stars / docs 都准);3) 抗 hallucination 不靠模型本身,靠 **channel 设计**(memoryAge "47 days ago" 触发 staleness reasoning 比 ISO timestamp 强 — 来自 prod user reports,非设计直觉)。

**用户实际开发经历记录**(用户 2026-05-16 亲口分享):_"system reminder 也是我在开发过程中遇到了一些问题然后想解决办法和探究出来的,一开始我不清楚有这个能力。还有 prompt cache 虽然一开始我知道这个,但是也是后期才开始做的。"_ — 这印证了本页价值:**SR 是个 builder 撞坑才会发现的 protocol,wiki 应该让下一个 builder 在第一次撞坑之前就知道**。

**meta(关于 wiki 自己)**:这页是对 _"wiki 不够深入"_ 反馈的直接回应。Depth-first 与现有 wide-first 页面差异:
- Wide-first(e.g. [[concept/agent-harness]]):横向 5 harness × 10 维 + [verified-from-source] 清单 — 适合**做决策**
- Depth-first(本页):单一机制 + 工程动机 + 失败模式 + 抗 hallucination — 适合**学习+迁移**
- 未来扩展应**两种共存**:harness 入口页 wide,concept 入口页 depth。已识别需要 depth 化的页:[[concept/agent-memory]](dream/consolidate 机制)、[[concept/context-compaction]](tool-result 压缩 phase 流程)、[[concept/observability]](OTel 集成深度)。

---

## [2026-05-17] ingest | AgentScope 1.0.20 — 第 6 个 harness + depth-first 写法 #2

**触发问题**: 用户指 "agentScope 这个项目 这个应该是阿里的"。AgentScope 是 Alibaba 通义实验室出品 25.2k stars,**5 大独家维度 wiki 当前全没覆盖**:Trinity-RFT agentic RL training(改 model weights 的 self-evolution V4)/ A2A 跨进程 protocol / Realtime voice + TTS / Anthropic Skill 跨家标准 / **StateModule PyTorch nn.Module 风格 agent state 抽象**(撬动 5 个 capability)。**B1 顺序**:先单独 ingest AgentScope,再做 RAG 主线(LlamaIndex + GraphRAG)。

**改动**:
- 新建 [[source/agentscope]] 30+ file:line 锚点(模块全清单 + 关键代码片段)
- 新建 [[harness/agentscope]] 13 维 frontmatter + narrative(三大独家点深度对照 + 模块结构图 + 借鉴 R-AS-1~R-AS-4 清单)
- 新建 [[concept/agent-state-separation]] **depth-first 写法 #2**(386 行,聚焦 `StateModule` 132 行如何撬动 5 capability)
- 更新 [[index]] — Harness / Concept / Source 区分别加 1 行

**关键 verified 事实**(grep AgentScope 1.0.20 @ commit b840fa9):
- **StateModule 核心**(`module/_state_module.py:20-152` 完整 132 行):PyTorch `nn.Module` 风格 — `__setattr__` 自动追踪 sub-StateModule 进 `_module_dict: OrderedDict` / `state_dict()` 递归 / `load_state_dict(strict=True)` 支持 schema 演化 / `register_state(custom_to_json, custom_from_json)` per-attr JSON converter / `_attribute_dict` 显式注册(Pythonic explicit > implicit)
- **Interrupt 协议 5 步**(`_react_agent.py:575-655, 687-699, 799-816`):asyncio.CancelledError 信号 + chunk-level `is_interrupted` flag(model gen + tool exec 双层检测)+ fake `ToolResultBlock(output="The tool call has been interrupted by the user.")` 自动补全防 history 孤儿 + `handle_interrupt` LLM-friendly 续接话术 `"I noticed that you have interrupted me..."`
- **SummarySchema**(`_react_agent.py:43-87`):5 字段 pydantic 强类型 memory 压缩(task_overview / current_state / important_discoveries / next_steps / context_to_preserve,各 Field max_length=200-300 + description)— vs Claude Code 自由文本 summary 防 hallucination
- **Session 4 backend**:`_json_session.py` / `_redis_session.py` / **`_tablestore_session.py`**(Alibaba Cloud NoSQL serverless)/ `_session_base.py:7-48` 用 `**state_modules_mapping: StateModule` kwarg 收集任意命名 StateModule 实现统一抽象
- **PlanNotebook**(`plan/_plan_notebook.py` 904 行):一等公民 plan,9 个 async method(create / revise / update_subtask / finish_subtask / view / recover_historical_plan / get_current_hint)+ 2 hook 注册(`register_plan_change_hook` / `remove_plan_change_hook`)— 比 Claude Code TodoWrite / Codex goals.rs 都结构化
- **Trinity-RFT** 业内独家 agentic RL(`tuner/` 10 文件 + `model/_trinity_model.py` + 6 production sample 真数字):Math 75→85% / Frozen Lake 15→86%(+71pp)/ Learn to Ask 47→92%(+45pp)/ Werewolf 50→80% / AIME-24 20→60%(+40pp)
- **A2A protocol**(`a2a/` 5 文件):跨进程 / 跨主机 agent 通信 + **`_nacos_resolver.py` Alibaba Nacos 服务发现**独家
- **MCP-as-local-function**(`mcp/_mcp_function.py` + `HttpStatelessClient.get_callable_function`):MCP server tool 抽出为 Python callable,3 种用法:直接调用 / 注册 toolkit / 包成 high-level tool
- **6 model + Trinity provider**:Anthropic / DashScope(Qwen,自家)/ Gemini / Ollama / OpenAI / Trinity(RL training)
- **7 per-provider formatter** + truncated_formatter_base 截断基类
- **Memory 双层**:`_long_term_memory/` + `_working_memory/`(业内罕见显式 scope 分离,跟 SkillForge observation/reflection/optimized lifecycle 分离正交)
- **Hooks 极简 1 文件**:`hooks/_studio_hooks.py` — vs Claude Code 27 / opencode 27 显著少,印证 _"hooks 数量不衡量成熟度"_

**SkillForge R-AS 借鉴清单**(4 + 1):
- 🔴 P1 **R-AS-1 StateModule 抽象统一持久化**(2-4 周,撬动跨环境迁移 + A/B + 时间旅行 + RL dataset)— 详细 6 步实施路径在 [[concept/agent-state-separation]] §8
- 🔴 P1 **R-AS-2 Interrupt fake-tool-result 补全**(< 1 周)— SkillForge `SubAgentStartupRecovery.java:35-95` 是 crash recovery,**in-session user-initiated cancel 是否补 fake tool_result 未 verify**,AgentScope `_react_agent.py:639-653` 给精确模板
- 🟡 P2 **R-AS-3 SummarySchema 强类型压缩**(< 半周,防 hallucination)— SkillForge V21 CONTEXT-COMPACT 自由文本 summary,可加 pydantic 5 字段 schema
- 🟡 P2 **R-AS-4 Memory long_term + working 显式分离**(< 半周)— SkillForge memory_kind 三态按 lifecycle 分,可加 scope dimension 正交
- 🟢 决策候选 **Trinity-RFT agentic RL**:V4 self-evolution 路径(改 model weights vs SkillForge V1-V3 改 memory/rule/skill),需 Qwen / open weights provider 不优先

**❌ 不该跟**:A2A + Nacos 跨进程(SaaS 单进程不需要)/ TTS realtime voice(SkillForge 文本主线)/ Studio 可视化 IDE(已有 dashboard 路线)

**meta**:depth-first 写法实验 #2 验证经验:
- [[concept/runtime-context-injection]](写法 #1)聚焦**协议**(`<system-reminder>` 5 类失败模式 + 训练契约)
- [[concept/agent-state-separation]](写法 #2)聚焦**抽象**(`StateModule` 132 行如何撬动 5 capability + PyTorch 影响痕迹 line-by-line)
- 共同点:不堆 catalog,讲机制 + 工程动机 + 横向校准 + 学习视角
- 新发现:**framework discipline > abstraction power**(StateModule 132 行不复杂,但所有 26 模块自觉 inherit 才是真力量来源)
- 这印证了用户反馈:"参考意义但不能提升和学习" 的问题在于 catalog 写法把 framework wisdom 拆碎,depth-first 才能让 wisdom 完整还原

**takeaway**:AgentScope 是第 6 个 ingested harness,带 3 个**业内独家**视角(StateModule / Trinity-RFT / A2A+Nacos)。对 SkillForge **2 条 P1 直接打到生产**(R-AS-1 + R-AS-2,共 < 4 周)+ 1 条 V4 self-evolution 远期候选(Trinity-RFT)。下一步 B1 第二阶段:RAG 主线(LlamaIndex + GraphRAG 双 ingest)。

---

## [2026-05-17] ingest | Dify — RAG 主线 Step 1 / 6,第 7 个 harness

**触发问题**: 用户点名 Dify(114k stars,业内 #2 RAG/agent framework,产品定位 = production AI app platform)。**A2 方案**(Dify + LlamaIndex + RAGFlow + Microsoft GraphRAG + Cognee + Pathway 6 项目并行 clone)第一个 ingest。Dify 跟 SkillForge **产品定位最近**(都是业务方 + 多租户 + Visual UI),其他 5 dev tool harness(Hermes/CC/OpenClaw/Codex/opencode)是开发者 CLI 工具,产品对照空缺由 Dify 填补。

**改动**:
- 新建 [[source/dify]] 30+ file:line 锚点(31 sub-modules + 560 Python files cataloged)
- 新建 [[harness/dify]] 13 维 + narrative(4 大独家点 + R-DIFY-1~R-DIFY-6 借鉴清单)
- 更新 [[index]] — Harness / Source 区分别加 1 行

**关键 verified 事实**(grep + read Dify @ commit 9d0906c):
- **31 sub-modules in `api/core/`**(560 Python files):agent / app / rag / tools / workflow / mcp / memory / datasource / model_manager.py / provider_manager.py / indexing_runner.py / plugin / moderation / ops / telemetry / logging / llm_generator / prompt / repositories(DDD)/ schemas / entities / errors / context / contexts / helper / callback_handler / extension / external_data_tool / trigger / db / hosting_configuration.py / base / + 多个 utility
- **`RetrievalMethod` 4 枚举**(`retrieval_methods.py:1-20`):SEMANTIC_SEARCH / FULL_TEXT_SEARCH / HYBRID_SEARCH / KEYWORD_SEARCH + 静态 helper `is_support_semantic_search` / `is_support_fulltext_search` 显式判断 method-feature 兼容
- **`RerankMode` 2 枚举**(`rerank_type.py`):RERANKING_MODEL(用 reranker model,Cohere 等)/ WEIGHTED_SCORE(加权融合)— 对照 SkillForge `MemoryService.java:853-908` RRF(Reciprocal Rank Fusion)
- **`api/core/rag/` 14 子模块**:cleaner / data_post_processor / datasource / docstore / embedding(含 cached_embedding.py)/ entities / extractor / index_processor(factory + base + processor/ + constant/)/ models / pipeline(仅 queue.py)/ rerank(model + weighted)/ retrieval(4 method)/ splitter(text_splitter + fixed_text_splitter)/ summary_index(独家)— 跟 mem0/Letta/MemPalace/Zep+Graphiti 5 memory 路线**正交**
- **`services/rag_pipeline/` 9 文件 DSL service 独立层**:`pipeline_generate_service.py` / `pipeline_template/` / `rag_pipeline_dsl_service.py` / `rag_pipeline_manage_service.py` / `rag_pipeline_task_proxy.py` / `rag_pipeline_transform_service.py` / `rag_pipeline.py` / `transform/` + `entity/`
- **Workflow node_factory.py 188+ 行**:`@lru_cache register_nodes()` 双源 import(`graphon.nodes` 外部 + `core.workflow.nodes` 内置 7 类:agent/datasource/knowledge_index/knowledge_retrieval/trigger_plugin/trigger_schedule/trigger_webhook)/ `resolve_workflow_node_class(node_type, node_version)` 双键索引 / `_LazyNodeTypeClassesMapping:172-213` MutableMapping cache snapshot + delete tracking + override(runtime 动态替换 / 隐藏)
- **5 Agent runner**(`api/core/agent/`):base_agent_runner / cot_agent_runner / cot_chat_agent_runner / cot_completion_agent_runner / fc_agent_runner — **显式分离 CoT(3 变体)vs Function Calling 路线** 给老模型/prompt-only 国产模型保留 CoT 路径
- **6 Tool 类型**(`api/core/tools/`):builtin_tool / custom_tool / mcp_tool / plugin_tool / **`workflow_as_tool`(业内独家)** / __base + ToolEngine + ToolManager + ToolLabelManager + signature.py
- **`workflow_as_tool` 独家抽象**:已发布 workflow 包成 callable tool 给 agent 调用 — **nested workflow composition**,LangChain Tool / OpenAI Function 都没有
- **极简 memory**:`api/core/memory/token_buffer_memory.py` 1 文件 — Dify **第 6 memory 路线"pure-RAG-as-memory"**(长期 context 全走 RAG knowledge base,不做 mem0/Letta/MemPalace 风 entity/reflection/summarization)
- **5 datasource type**(`api/core/datasource/`):local_file / online_document / online_drive / website_crawl / __base
- **`api/core/moderation/` 内容审核独立模块**(业内罕见 first-class)
- **多 harness dogfood**:`.claude/skills/` 6 个(backend-code-review / component-refactoring / e2e-cucumber-playwright / frontend-code-review / frontend-query-mutation / frontend-testing)+ `.agents/skills/` 7 个(add how-to-write-component / tailwind-css-rules)+ `.codex` 空 marker + `.gemini/` 配置 + **`CLAUDE.md → AGENTS.md` symlink 业内独家细节**

**SkillForge R-DIFY 借鉴清单**(6 条):
- 🔴 P1 **R-DIFY-1 Visual Workflow + DSL 双轨**(YAML ↔ UI 双向 sync,Dify `rag_pipeline_dsl_service.py` 路径)— 产品差异化高 ROI
- 🔴 P1 **R-DIFY-2 workflow_as_tool 抽象**(1-2 周)— SkillForge 当前 skill 调 skill 无显式抽象,加 nested tool 比 sub-session 轻量
- 🟡 P2 **R-DIFY-3 100+ provider 统一 manager**(收敛到 3-5 家,1-2 周)— 商业化阶段
- 🟡 P2 **R-DIFY-4 Moderation first-class 模块**(< 1 周)— safety hook 散在各层,可独立化
- 🟢 P3 **R-DIFY-5 RAG 4 retrieval method × 2 rerank mode 抽象**(远期参考)
- 🟢 P3 **R-DIFY-6 多 harness dogfood 套路 7 skill 模板**(< 半周)— `AGENTS.md ↔ CLAUDE.md symlink` 协议是高 ROI 细节

**❌ 不该跟**:Visual builder UI 主线(SkillForge 是 Dashboard 不是拖拽)/ 100+ provider 自己接(3 家足够)/ Workflow engine 100% 抄(SkillForge skill+agent 更细粒度)

**meta**:Dify 是**第 7 个 ingested harness**,跟之前 6 个 dev tool 路线(Hermes/CC/OpenClaw/Codex/opencode/AgentScope)+ 1 个 framework(AgentScope)不同,**第一个 production application platform 产品形态**。给 SkillForge 提供商业 / 多租户 / Visual / 用户友好度对照 — 这块在 R-MEM / R-HARNESS verify 流程中之前是空白。

**takeaway**:Dify 让 wiki 产品视角完整(开发者工具 vs 业务应用平台两端都覆盖)。**R-DIFY-2 workflow_as_tool 是直接打到 SkillForge 架构的 P1**(SkillForge 当前 skill 调 skill 无抽象,加完直接撬动 nested composition 能力)。下一步:LlamaIndex(标杆 framework)+ RAGFlow(deep document understanding 角度)+ Microsoft GraphRAG(KG paradigm)4 必做剩 3 个,然后 Cognee + Pathway 应做 2 个,最后写 [[concept/rag-design-space]] depth-first synthesis。

---

## [2026-05-17] ingest | RAG 主线 Step 2-6 / 6:LlamaIndex + RAGFlow + MS GraphRAG + Cognee + Pathway 批量

**触发问题**: 继续 RAG 主线 A2 方案(6 项目并行 clone 后顺序 ingest),Dify 完成后批量做剩余 5 个,为最终 [[concept/rag-design-space]] depth-first synthesis 收集材料。

**改动**(每个项目 source + 13 维 frontmatter 页):
- 新建 [[source/llama_index]] + [[eval/llama_index]] — 业内 RAG framework 标杆
- 新建 [[source/ragflow]] + [[harness/ragflow]] — Deep document understanding + gVisor
- 新建 [[source/ms-graphrag]] + [[eval/ms-graphrag]] — KG paradigm 起点 + ICLR'26 calibration
- 新建 [[source/cognee]] + [[eval/cognee]] — Memory control plane,跟 SkillForge memory 路线最近
- 新建 [[source/pathway]] + [[eval/pathway]] — Streaming RAG 唯一(Rust + Differential Dataflow)
- 更新 [[index]] — Eval / Source 区批量加 5 + 5 = 10 行
- **5 + 5 = 10 个新 wiki 页 + 累计约 2500 行新内容**

**关键 verified 事实速查**:

**[[eval/llama_index]]**(LlamaIndex 0.14.22 @ 23bd65a):
- 818 MB modular monorepo:llama-index-core 56 modules + 23 integration category
- **104 LLMs / 78 vector_stores / 159 readers / 66 embeddings / 68 tools** integration count
- **18+ Query Engine** explicit type(citation/router/sub_question/multistep/sql_join/flare/multi_modal/jsonalyze/pandas/transform/retry/cogniswitch/knowledge_graph/graph 等)— RAG composition primitive 业内最广
- **18+ Index 类型** 含 **Property Graph 一等公民**(`indices/property_graph/` 完整 KG 内置)
- Workflow 已抽离到独立 `workflows` PyPI(`workflow.py` 1 行 import)
- **Composable Memory 5 类** + memory_blocks 模块化
- LlamaIndex 官方集成 graph_rag 只 1 个:`llama-index-graph-rag-cognee`(跟 Cognee 联姻)

**[[harness/ragflow]]**(RAGFlow 0.25.4 @ 09d45046,**80.6k stars 业内 #2**):
- **Deep document understanding 核心护城河**:`deepdoc/parser/` 16 文件格式 parser(mineru + paddleocr + docling + 13 其他)+ `deepdoc/vision/` 10 vision 模块(table_structure_recognizer + layout_recognizer + OCR)
- **15 模板按文档类型 chunking**(`rag/app/{book,email,laws,manual,naive,paper,picture,presentation,qa,resume,table,tag,audio,one}.py`)
- **双流派 GraphRAG**:`rag/graphrag/general/`(Microsoft 风)+ `rag/graphrag/light/`(LightRAG 风)— 用户选
- **RAPTOR + tree query decomposition** 内置(`rag/raptor.py` + `rag/advanced_rag/tree_structured_query_decomposition_retrieval.py`)
- **gVisor sandbox**(`agent/sandbox/` Docker Compose + executor_manager + sandbox_base_image + sandbox_spec.md)— 业内 RAG framework **唯一显式 gVisor production-grade isolation**
- **Agent Canvas DSL**(`agent/canvas.py` Graph class:components map + downstream/upstream)+ 20+ workflow component(Iteration/Loop/Switch/Exit_loop)
- **中文 NLP 一等公民**(`rag/nlp/rag_tokenizer.py + surname.py + synonym.py + term_weight.py`)
- **OpenClaw 生态集成**:**2026-03-24 RAGFlow Skill on OpenClaw** (clawhub.ai/yingfeng/ragflow-skill)— 跟之前 ingest 的 [[harness/openclaw]] 印证 ecosystem 协同
- **10+ 国语言 README** 业内最广
- Memory 模块新(2025-12-26)

**[[eval/ms-graphrag]]**(Microsoft GraphRAG @ de531f0a):
- arxiv 2404.16130 _"From Local to Global: A Graph RAG Approach"_ — **KG RAG paradigm 起点论文实现**
- 8 sub-package monorepo + 主包 14 模块
- **4 Search Mode 核心创新**(`query/structured_search/`):basic vanilla / local entity 邻域 / **global community report 汇总** / **drift_search**(2025 新:Dynamic Reasoning + Inference with Flexible Traversal)
- **Community Report Primitive 关键创新**:LLM 自动写社区摘要,绕过 entity-level 查询,适合 holistic question
- KG 7 数据模型(Entity/Relationship/Community/CommunityReport/Covariate/TextUnit/Document)
- 21 Indexing Workflow + 13 Operation 细粒度 pipeline
- **`prompt_tune/` 自动 prompt 调优** 内置
- ⚠️ README 明确 _"not officially supported product"_(demo)
- ⚠️ **ICLR'26 GraphRAG-Benchmark calibration**:_"GraphRAG frequently underperforms vanilla RAG on many real-world tasks"_ — **KG RAG 未必更好,SkillForge 不要默认走 KG 路线**

**[[eval/cognee]]**(Cognee 1.0.9 @ 4ca1d0c2):
- **"Memory control plane for AI Agents - Brain behind your Agents"** — 跟 SkillForge memory **路线最近**
- 25 modules / 17 task category / 22 retriever
- **memify_pipelines 7 个**:apply_feedback_weights / apply_frequency_weights / consolidate_entity_descriptions / create_triplet_embeddings / **persist_agent_trace_feedbacks_in_knowledge_graph ⭐** / **persist_sessions_in_knowledge_graph ⭐** / memify_default_tasks
- **22 Retriever 显式建模**:5 种 graph_completion 变体(基础/上下文扩展/CoT/分解/摘要)+ Cypher + Temporal + Triplet + jaccard + lexical + natural_language 等
- **RDF/XML Ontology Resolver**(`modules/ontology/{rdf_xml,matching_strategies,base_ontology_resolver}.py`)— 业内罕见
- **`cognee/skill.md` 业内独家**:仓库内自带 Anthropic Skill 标准 wrapper,可被 Claude/agent 直接 install + call
- 8 DB infrastructure:cache / graph (Kuzu+Neo4j) / **unified** / **hybrid** / relational / vector + dataset_database_handler + dataset_queue
- 内置 BEAM benchmark(`cognee/eval_framework/run_beam_eval.py`)
- **LlamaIndex 官方集成包 `llama-index-graph-rag-cognee`** — LlamaIndex 选的 KG memory partner

**[[eval/pathway]]**(Pathway @ 510c6daf,**BSL 1.1** NavAlgo SAS):
- **"Live Data Framework"** — Python ETL + LLM/RAG streaming framework
- **业内 streaming-RAG 唯一**(其他都 batch + 手工增量)
- Python frontend + **Rust 核心引擎**(src/) + vendored `external/differential-dataflow/`(Frank McSherry Microsoft Research Naiad NSDI 2013 论文)
- 文档库变化只重算 delta,index 实时反映 — 业内独家 framework-level incremental compute
- `python/pathway/xpacks/llm/` 11 文件完整 RAG xpack:document_store + embedders + llms + **mcp_server** + parsers + prompts + question_answering + rerankers + servers + splitters + vector_store
- `python/pathway/stdlib/{temporal,statistical}/` 时序 + 统计 first-class
- 14 integration_tests category:airbyte / db_connectors / gdrive / iceberg / kafka / **rag_evals** / s3 / sharepoint 等
- 4 production RAG template:Adaptive RAG / Multimodal RAG with gpt-4o / Private RAG (Ollama+Mistral) / Unstructured-to-SQL
- ⚠️ **BSL 1.1 license 商用需 NavAlgo 授权**

**SkillForge R-XX 借鉴清单速查**(具体在各页 frontmatter `skillforge_takeaway`):

| 来源 | R-ID | 项 | 优先级 | 工作量 |
|---|---|---|---|---|
| LlamaIndex | R-LI-1 | Query Engine 18+ 种 explicit type 抽象 | 🟡 P2 远期 | — |
| LlamaIndex | R-LI-2 | Composable memory blocks(功能维度) | 🟡 P2 | < 1 周 |
| LlamaIndex | R-LI-3 | Property Graph 一等公民(⚠️ ICLR'26 警示)| 🟢 P3 | 远期 |
| RAGFlow | R-RF-1 | Deep document understanding 16 parser | 🟡 P2 | 远期 |
| **RAGFlow** | **R-RF-2** | **gVisor Sandbox 集成**(P1 safety,SkillForge skill engine 直接打)| 🔴 P1 | **2-3 周** |
| RAGFlow | R-RF-3 | Template-based chunking 15 文档类型 | 🟡 P2 | 远期 |
| RAGFlow | R-RF-4 | RAPTOR + tree query decomposition | 🟡 P2 | 远期 |
| MS GraphRAG | R-GR-1 | 4 Search Mode 抽象(basic/local/global/drift)| 🟡 P2 | 远期 |
| MS GraphRAG | R-GR-2 | Community Report Primitive | 🟡 P2 | 远期 |
| **MS GraphRAG** | **R-GR-CALIBRATION** | ⚠️ **KG RAG 未必更好(ICLR'26 paper),SkillForge 不要默认走 KG 路线** | 🔴 **必读** | — |
| **Cognee** | **R-COG-1** | **Agent trace + feedback 自动入 KG**(直接打 ATTRIBUTION-AGENT v3 缺口)| 🔴 P1 | **1-2 周** |
| **Cognee** | **R-COG-2** | **Decorator-based agent memory API**(@cognify 风) | 🔴 P1 | **1 周** |
| Cognee | R-COG-3~7 | 22 Retriever / Unified+Hybrid DB / Ontology / skill.md / BEAM benchmark | 🟡 P2 / 🟢 P3 | 远期 |
| Pathway | R-PW-1 | Differential Dataflow **设计思想**(memory dedup 增量化)| 🟡 P2 | < 1 周 design 评估 |
| Pathway | R-PW-2 | Temporal + Statistical stdlib 设计 | 🟡 P2 | 远期 |
| Pathway | R-PW-3 | RAG xpack as MCP server pattern | 🟢 P3 | < 半周 |

**RAG 主线 SkillForge 真正直接打到生产的 P1**(3 条):
- 🔴 **R-DIFY-2** workflow_as_tool / skill_as_tool 抽象(1-2 周)
- 🔴 **R-RF-2** gVisor Sandbox 集成(2-3 周)
- 🔴 **R-COG-1** Agent trace + feedback 自动入 KG(1-2 周)
- 🔴 **R-COG-2** Decorator-based agent memory API(1 周)

**4 条共 5-8 周工作量** = 给 SkillForge 完整覆盖 **skill 复用 / 代码安全沙箱 / 闭环 memory 自演化 / skill-internal memory 抽象** 4 大产品能力短板。

**meta**:5 个 RAG ingest 全部完成,**已积累足够材料写 [[concept/rag-design-space]] depth-first synthesis**(下一步)。6 个 RAG 项目横向对照矩阵:
| | Dify | LlamaIndex | RAGFlow | MS GraphRAG | Cognee | Pathway |
|---|---|---|---|---|---|---|
| Stars | 114k | 46.5k | 80.6k | demo | growing | 50k+ |
| License | Dify OSL | MIT | Apache 2.0 | MIT | Apache 2.0 | **BSL 1.1** |
| 类型 | Platform | Framework | Platform | Demo | Framework | Framework |
| KG 强度 | 弱 | Property Graph | 双流派 GraphRAG | 核心 | 核心 | 弱 |
| Streaming | 弱 | 弱 | Ingestion pipeline | 弱 | sync module | **核心** |
| Doc parsing | 浅 | 159 reader | **深** | 浅 | 中 | 中 |
| MCP | partial | partial | ✅ | ❌ | ✅ | ✅ |
| 跟 SkillForge 距离 | 产品近 | 抽象层近 | 工程近 | paradigm 远 | **memory 最近** | 设计思想可借 |

**takeaway**:6 个 RAG project ingest 完整覆盖 _"production platform(Dify/RAGFlow)/ framework(LlamaIndex)/ paradigm(MS GraphRAG)/ memory(Cognee)/ streaming(Pathway)"_ 5 个维度,**给 SkillForge memory v3 决策 + RAG 工程化 + safety + 闭环 self-improving memory 4 条 P1 借鉴**(R-DIFY-2 + R-RF-2 + R-COG-1 + R-COG-2)。下一步:写 [[concept/rag-design-space]] depth-first 合成所有材料 + ICLR'26 GraphRAG-Benchmark calibration。

---

## [2026-05-17] concept | [[concept/rag-design-space]] — depth-first 写法 #3 synthesis

**触发问题**: 6 个 RAG ingest 完成后,需要 depth-first synthesis 把材料合到一个 concept 页,给 SkillForge memory v3 一个决定性 borrow clipboard,而不是分散在 6 个 source/eval/harness 页里 catalog 形式。

**改动**:
- 新建 [[concept/rag-design-space]] **depth-first 写法 #3 synthesis**(372 行)— 跟 [[concept/runtime-context-injection]](写法 #1 协议)和 [[concept/agent-state-separation]](写法 #2 抽象)同体系,本页是写法 #3 跨多 project synthesis
- 更新 [[index]] — Concept 区加入

**核心结构**:
- §1 三件 calibration:**SkillForge memory 不是 RAG over documents 而是 RAG over agent state** + **ICLR'26 警示 KG 未必更好** + **RAG framework ≠ memory framework**
- §2 **8 维设计空间**:文档 parsing / chunking / index type / retrieval pattern / rerank / indexing 时态 / synthesis / feedback loop
- §3 **6 项目 × 8 维位置图**(每项目最强维度标 🟢)— 关键发现 _"没有一个 8 维全强,SkillForge cherry-pick 按维度"_
- §4 SkillForge memory v3 决策建议 — 已有 ✅ / 真缺口 🔴 / 不要做 ⚠️ 三档
- §5 **R-RAG-1~R-RAG-12 决定性清单**(取代分散的 R-DIFY/R-LI/R-RF/R-GR/R-COG/R-PW)
- §6 跟现有 wiki 关系盘点
- §7 学到 3 件事

**🔴 R-RAG P1 必做 4 条**(SkillForge 5-8 周直接打到生产 4 大能力短板):
- **R-RAG-1** Agent trace + user feedback → memory 自动 pipeline(Cognee `persist_agent_trace_feedbacks_in_KG`)
- **R-RAG-2** Decorator-based skill-internal memory API(Cognee `@cognify`)
- **R-RAG-3** workflow_as_tool / skill_as_tool 抽象(Dify)
- **R-RAG-4** gVisor sandbox 集成(RAGFlow `agent/sandbox/`)

**⚠️ 反直觉关键发现**:
1. **SkillForge memory 不是普通 RAG over documents** — 是 RAG over agent state(MemoryEntity / SkillEntity / SubAgentRunEntity / LlmTraceEntity 等 structured records)。这意味着 80% 的 _"deep document understanding"_ 工程对 SkillForge **不直接 applicable**,但**抽象层(retrieval pattern / KG / temporal)+ 闭环工程(sandbox / incremental / feedback loop)**是直接借鉴
2. **ICLR'26 GraphRAG-Benchmark paper** 明确警示 _"GraphRAG frequently underperforms vanilla RAG on many real-world tasks"_ — SkillForge **不要默认走 KG memory v3 路线**,先把 vanilla(已有 hybrid FTS+Vector+RRF)做扎实
3. **6 个 RAG project 没有一个 8 维全强**:Cognee 在 feedback loop 强 / Dify 在 workflow 抽象强 / RAGFlow 在 parsing + sandbox 强 / LlamaIndex 在 retrieval pattern 抽象强 / MS GraphRAG 在 KG paradigm 强 / Pathway 在 streaming 强 — SkillForge **必须按维度 cherry-pick**,不要全套抄某一家

**meta(关于 wiki 自己)**:depth-first 写法 3 版完成,各有侧重:
- 写法 #1([[concept/runtime-context-injection]])聚焦**协议**(`<system-reminder>` 5 类失败 + 训练契约)
- 写法 #2([[concept/agent-state-separation]])聚焦**抽象**(`StateModule` 132 行如何撬动 5 capability)
- **写法 #3([[concept/rag-design-space]])聚焦 synthesis 跨多 project**(6 RAG framework 8 维设计空间 + ICLR'26 paper calibration + SkillForge cherry-pick clipboard)
共同点:不堆 catalog,讲机制 + 工程动机 + 横向校准 + 学习视角 + SkillForge actionable

**takeaway**:RAG 主线 B1 方案 6 项目 + 1 paper calibration + 1 synthesis concept = **完整 RAG ingest 第二阶段交付**。累计本 session **11 个新 wiki 页 ~ 3300 行新内容**(5 source + 3 harness + 2 eval + 1 concept synthesis)。**给 SkillForge memory v3 + RAG 工程化的决定性 P1 清单** = R-RAG-1~4(共 5-8 周),完整 14 项跨页 R-XX 借鉴 clipboard 已建立(R-HARNESS / R-AS / R-INJECT / R-DIFY / R-LI / R-RF / R-GR / R-COG / R-PW / R-RAG)。下一步建议:跟用户对齐**全栈 borrow roadmap 合并**(把分散 R-XX 清单出 SkillForge 1 张 prioritized backlog)。

---

## [2026-05-17] meta+deep | 新增 `deep/` 类型 + 3 demo 单机制深挖

**触发问题**:用户反馈 _"harness 的细节太粗了 这个你怎么处理?"_。回顾发现:之前承诺 depth-first 重写 3 concept 页,但 8 个 harness 页本身从未被 depth 重写。结构性原因 — schema.md 强制 13 维 frontmatter 把 harness 页推向 breadth 模式。**3 个 concept 页(runtime-context-injection / agent-state-separation / rag-design-space)其实就是 deep 雏形** — 现在正式化为 `deep/` 子目录类型。

**改动**:
- 更新 [[schema]] §2 目录结构 + §3.4b 加 `deep/` 类型规范(必须含 §WHY + §失败模式 + 双向 wikilink 连回 harness 页 + B 模式 file:line)
- 新建 [[deep/claude-code-autodream-3-gate]](329 行)— Claude Code 后台 memory consolidation 深挖
- 新建 [[deep/codex-actor-resume-agent]](326 行)— Codex Rust actor model + resume_agent_from_rollout 深挖
- 新建 [[deep/opencode-lsp-first-class]](317 行)— opencode LSP-as-agent-tool 深挖
- 更新 [[index]] — 新增 Deep 区块

**3 个 demo deep 页的设计选择**:

**[[deep/claude-code-autodream-3-gate]]**(`src/services/autoDream/` 4 文件 550 行 verify):
- **3 反直觉发现**:(1) lock 文件 mtime 同时承担"上次时间戳"+"互斥锁"(`consolidationLock.ts:1-2`)/ (2) 3-gate 按 syscall cost 严格升序 + scan throttle 防 _"Gate 1 开 Gate 2 闭"_ 退化场景(`autoDream.ts:54-56`)/ (3) 4-phase prompt 强类型 procedural instruction(vs Hermes 自由 prompt / AgentScope pydantic schema)
- **10 类 production 故障**显式 cover 在源码注释 + prompt 内容里(ext4 birthtime 问题 / PID reuse guard / race detection / stale memory citation 反 authority 悖论 / relative date drift / ...)
- 跟 [[concept/runtime-context-injection]] §9 prompt cache 协同呼应(`createCacheSafeParams`)

**[[deep/codex-actor-resume-agent]]**(`core/src/agent/` 6 文件 6000+ Rust verify):
- **3 反直觉发现**:(1) Mailbox 双 channel(mpsc 传消息 + watch::Sender<u64> 传 seq)解决"receiver 异步等通知而非 poll"(`mailbox.rs:11-49`)/ (2) `trigger_turn: bool` flag 区分 push/pull 消息节省 LLM call(`mailbox.rs:63-66`)/ (3) `resume_agent_from_rollout` BFS 队列 + `DirectionalThreadSpawnEdgeStatus::{Open, Closed}` 区分 shutdown vs close(`control.rs:448-523`)
- **SpawnReservation RAII pattern**(`registry.rs:294-340`)Rust borrow checker 强制 cleanup 防 slot 泄漏 — 编译器保证而非 try/finally 约定
- **CAS loop lock-free 实现 `agent_max_threads` 限额**(`registry.rs:275-291`)
- 跟 [[concept/agent-state-separation]] AgentScope StateModule(PyTorch nn.Module 风)+ Claude Code SubAgent(in-memory)三家 multi-agent state 模型对照
- Codex 是业内唯一把 Erlang BEAM 哲学搬到 Rust agent runtime

**[[deep/opencode-lsp-first-class]]**(`tool/lsp.ts:1-113` + `lsp/lsp.ts:1-150` + `lsp/language.ts:1-121` verify):
- **3 反直觉发现**:(1) LSP-as-agent-tool 是 13 行 Effect-TS Schema 的事,真复杂度在 `server.ts` 2065 行 per-language config / (2) `touchFile` before query 确保 server 看 unsaved buffer(`tool/lsp.ts:80`)/ (3) Tool input 用 1-based(LLM-ergonomics)+ 内部 -1 转 LSP spec 0-based(`tool/lsp.ts:64`)
- **9 LSP operations as agent tool**(goToDefinition / findReferences / hover / documentSymbol / workspaceSymbol / goToImplementation / prepareCallHierarchy / incomingCalls / outgoingCalls)— 业内独家(Claude Code / Codex / Cursor / Aider 都没有把 9 op 完整暴露)
- **State { clients, servers, broken, spawning } 4 字段设计意图清晰**:broken 黑名单防反复重试 spawn 失败 server / spawning Promise 占位防 race
- **SymbolKind 26 → 8 filter** 防 `workspaceSymbol` query 撑爆 context
- 跟 5 家 coding agent(Claude Code / Codex / Cursor / Aider / opencode)3 流派(grep-only / AST tree-sitter / LSP)对照

**meta(关于 wiki 自己)**:Deep 写法验证 — **跟 concept 写法的 4 大差异**:

| | concept/ | deep/ |
|---|---|---|
| 范围 | 跨多 harness synthesis 或 N 维设计空间 | **单 harness 单机制** |
| 触发 | 多个 harness ingest 后做 cross 比较 | 一个 harness ingest 后挑值得深挖的机制 |
| 结构 | 8 维 / 5 路线 / 6 项目对照 表格 | **§机制 + §WHY + §失败模式 + §横向校准 + §学习视角** |
| 价值 | 决策(选 framework / pattern) | **学习 + 迁移**(理解 production engineering wisdom) |

**关键收获**:写 3 个 demo deep 页让我自己学到的:
- **autoDream**:_"production engineering wisdom 在源码注释里,不在 README claim 里"_(10 类故障显式 cover)
- **Codex actor**:_"Rust borrow checker 是 multi-agent state 管理的 dividend"_(RAII vs try/finally)
- **opencode LSP**:_"协议成熟度 = 工程红利"_(LSP 9 年 spec 让 113 行 tool 撬动 IDE 级 capability)

**takeaway**:3 个 deep page 模式验证成功,体量(各 ~300 行)+ 深度(每个 8 节深挖)+ 学习价值都达标。**剩余 5 个待写**:claude-code-compact-tool-result / codex-code-mode-v8-isolate / openclaw-memory-dreaming-6-dim / ragflow-deepdoc-table-recognizer / dify-workflow-as-tool。

**新工作模式确立**:每个新 harness ingest 后 = 1 个 wide-first harness 页(quick reference)+ 2-3 个 depth-first deep 页(挑最有 teaching 价值的机制)。Schema lint 规则相应更新。下一步:跟用户对齐先做哪 2-3 个 deep page,或是否批量推进剩 5 个。

---

## [2026-05-17] deep batch | 完整 5 个 deep page 批量交付

**触发问题**:用户 _"继续吧。全都搞了吧。然后 后面我学习下看看"_ — 完整 8 个 deep page 集合的最后 5 个,准备给用户学习用。

**改动**(5 个新 deep page 共 1831 行):
- 新建 [[deep/claude-code-compact-tool-result]](345 行)— Claude Code microCompact + cache-editing API + COMPACTABLE_TOOLS allowlist + time-based proactive trigger
- 新建 [[deep/codex-code-mode-v8-isolate]](325 行)— Codex V8 isolate + 8 JS primitive + delete 4 dangerous globals + async pause/resume
- 新建 [[deep/openclaw-memory-dreaming-6-dim]](409 行)— OpenClaw 6 维加权评分(`relevance 0.30 > frequency 0.24`)+ 3 phase sleep cycle(light/deep/REM)+ 梦境日记反元评论 subagent
- 新建 [[deep/ragflow-deepdoc-table-recognizer]](364 行)— RAGFlow 6 label table structure + 11 label page layout + 9 cell type + 2D 重构修复算法
- 新建 [[deep/dify-workflow-as-tool]](388 行)— Dify workflow nested as callable tool + 8 type 自动映射 + workflow_call_depth 递归跟踪 + pause 禁用
- 更新 [[index]] Deep 区追加 5 行详细描述

**累计 8 个 deep page 一览**(3073 行,跨 5 harness):

| Deep page | 行数 | 来源 harness | 核心机制 | 3 反直觉发现 |
|---|---|---|---|---|
| claude-code-autodream-3-gate | 402 | Claude Code | 3-gate cheapest-first + lock-as-mtime 双语义 + 4-phase prompt | lock 文件 mtime 双语义 / 3-gate 按 syscall cost 排序 / 4-phase 强类型 procedural prompt |
| **claude-code-compact-tool-result** | **345** | Claude Code | cache-editing 不删本地 + COMPACTABLE_TOOLS 8 allowlist + time-based proactive | cache_edits API 保 prompt cache / time-based vs token-based / allowlist > blocklist |
| codex-actor-resume-agent | 447 | Codex | Mailbox 双 channel + RAII SpawnReservation + BFS resume agent tree | mpsc+watch 双 channel / RAII 编译器强制 cleanup / BFS resume open edge |
| **codex-code-mode-v8-isolate** | **325** | Codex | V8 isolate + 8 JS primitive + delete 4 dangerous globals | LLM 写 JS 而非 JSON / delete console/Atomics/SAB/WebAssembly / store/load 显式 > 隐式 |
| opencode-lsp-first-class | 393 | opencode | 9 LSP ops as agent tool + 1-based LLM-ergonomics + touchFile before query | LSP-as-tool 13 行 + 80+ 语言 / 1-based input + 内部 -1 / State 4 字段意图清晰 |
| **openclaw-memory-dreaming-6-dim** | **409** | OpenClaw | 6 维加权评分(relevance 0.30 最重) + 3 phase sleep cycle + 梦境日记反元评论 | relevance > frequency 反直觉 / 神经科学不是 metaphor 是 algorithm / 反元评论 prompt |
| **ragflow-deepdoc-table-recognizer** | **364** | RAGFlow | 6 label table struct + 11 label layout + 9 cell type + 2D 重构 + 3 backend(含 Ascend 国产) | 6 label 含 spanning cell / 2 阶段 garbage detection / 国产硬件支持是必修 |
| **dify-workflow-as-tool** | **388** | Dify | workflow as Tool + 8 type 自动映射 + workflow_call_depth 递归跟踪 + pause 禁用 | NOTE inline comment 解释意图 / 显式 mapping > 通用 reflection / 递归累加 + downstream max check |

**5 harness 覆盖** + **跨 26 反直觉发现** + **每个 deep page 含 §WHY + §失败模式(10-13 类显式 cover)+ §横向校准 + §学习视角**。

**最终模式 5 大共性 production engineering wisdom**(从 8 deep page 提炼):
1. **production wisdom 在源码注释 / inline NOTE 里,不在 README claim 里**(autoDream/compact/workflow_as_tool 都有大量 inline 注释解释非平凡决策)
2. **失败模式必须 explicit cover**(每页 10+ 类 production 故障在代码 + 注释里显式 handle)
3. **协议成熟度 = 工程红利**(LSP 9 年 spec / V8 sandbox / cache_edits API 都是 _"站在巨人肩膀上"_)
4. **显式 > 隐式**(workflow_call_depth 递归累加 / store/load 显式 cell 间传值 / 8 类型显式 mapping / lock body PID 显式)
5. **反直觉决策需要 measured evidence**(Claude Code _"98% cache miss"_ Jan 2026 实验 / OpenClaw `relevance > frequency` 权重反直觉)

**用户学习指南**:8 个 deep page **可独立读**(不依赖 harness 页),建议阅读顺序:
- **入门**:claude-code-autodream-3-gate(3-gate scheduler + lock-as-mtime 简单且 production)
- **算法**:openclaw-memory-dreaming-6-dim(数学化 + 神经科学启发,有趣)
- **架构**:codex-actor-resume-agent(Rust actor model + RAII pattern)
- **协议层**:codex-code-mode-v8-isolate(LLM 写 JS 革命性范式)
- **工程深度**:ragflow-deepdoc-table-recognizer(production CV + 国产硬件 + 跨页 garbage)
- **组合性**:dify-workflow-as-tool(workflow 作 nested composition)
- **专项**:claude-code-compact-tool-result(prompt cache + microCompact) / opencode-lsp-first-class(LSP-as-tool 单一机制)

**takeaway**:深度学习路径完整建立。**8 个 deep page = 跨 5 harness × 8 个具体机制 × 平均 8 节深挖**。Wiki 学习面价值 _"参考意义但不能提升"_ 反馈得到系统性回应。用户后续学习可独立读任一篇,也可按 harness / 主题串读。

---

## [2026-05-17] ingest | Agent Team / Manager 3 项目主线(CrewAI + AG2 + MetaGPT)+ depth-first 写法 #4 synthesis

**触发问题**: 用户跳过 autoResearch 后,转向 _"agent team 或 agents manager"_ 方向。Wiki 之前 multi-agent 章节聚焦的是 _spawn / mailbox / fork_ 这种**底层 primitive**(Claude Code SubAgent / Codex actor / AgentScope MsgHub / Dify workflow_as_tool 已 cover),缺 _"team / manager / role / SOP"_ 这种**组织行为学**上一层抽象。选 A 方案:3 ingest(CrewAI + AG2 + MetaGPT)+ 1 concept synthesis。

**改动**:
- 新建 [[source/crewai]] + [[harness/crewai]](6 个 file:line 锚点表 + 4 大独家点)
- 新建 [[source/ag2]] + [[eval/ag2]](formerly Microsoft AutoGen)
- 新建 [[source/metagpt]] + [[harness/metagpt]](_"软件公司 as multi-agent system"_)
- 新建 [[concept/agent-team-patterns]] **depth-first 写法 #4 synthesis**(跨 3 个新 ingest + 5 个已 ingest harness)
- 更新 [[index]] Harness / Eval / Source / Concept 4 个区分别加 1 行

**关键 verified 事实**:

**[[harness/crewai]]**(CrewAI v1.14.5a6 @ a95d2676,30k+ stars):
- 5-package monorepo(cli + crewai + crewai-core + crewai-files + crewai-tools)
- 主包 `lib/crewai/src/crewai/` 26 子模块
- **Crew = team of agents** + **`Process` enum**(sequential / hierarchical / **consensual TODO 未实现**)
- **Task DAG via `context: list[Task]`**(类型安全 Python 引用,不是 string template)
- **3 维 memory scope**(short_term / long_term / entity)— 跟 5 路线 memory framework + Dify pure-RAG + Cognee memify 共存,第 7 路线 _"scope-based"_
- **4 lifecycle callback**(step / task / before_kickoff / after_kickoff)
- `manager_llm` / `manager_agent` hierarchical 模式 + `delegations: int` 跟踪委派次数
- A2A protocol(Google)+ MCP 集成
- Flow(event-driven workflow)+ Crew 双抽象(正交)
- 3 种 task structured output(json / pydantic / native response_model)

**[[eval/ag2]]**(AG2 formerly AutoGen v0.13.0 @ ce95893,40k+ stars):
- 学术起点(arxiv 2308.08155),2026 分叉到 AG2ai
- **`ConversableAgent` 4577 行庞大基类** — AssistantAgent / UserProxyAgent / GroupChatManager / 20+ contrib agent 全继承
- **GroupChat dataclass** + **5+1 speaker selection 模式**(auto / manual / random / round_robin / Callable + eligibility filter)
- **状态机式 transition 控制**(`allowed_or_disallowed_speaker_transitions: dict` + `speaker_transitions_type: "allowed"|"disallowed"`)
- **Auto selection 4 prompt template 处理 LLM 边界 case**(返多个名字 retry / 返没名字 retry / 返非法名字 retry)— production-tested
- **`func_call_filter`** — function call 时按 function name 自动选 speaker
- **`agentchat/group/` 新版 declarative abstraction**(condition-based transitions:patterns/targets/handoffs/on_condition/llm_condition/guardrails/safeguards)
- **`contrib/` 20+ advanced pattern**:**Society of Mind**(Minsky 1986 神经科学经典)+ **Swarm** + **CaptainAgent** + multimodal/llava/math/retrieve/web_surfer agent
- **`opentelemetry/` OTel 内置**(40k+ stars 项目里独家)

**[[harness/metagpt]]**(MetaGPT @ 11cdf466,45k+ stars,arxiv 2308.00352):
- _"软件公司 as multi-agent system"_ 范式起点
- **Team + Role + Environment + SOP** 四件套
- **16+ 预定义 role**:Architect / Engineer / ProductManager / ProjectManager / QAEngineer / TeamLeader / DataAnalyst / Researcher / CustomerService / Sales / Searcher / Teacher / TutorialAssistant / Assistant / InvoiceOCRAssistant
- **25+ 预定义 Action**:design_api / fix_bug / project_management / debug_error / etc
- **⭐ `investment` + `cost_manager.max_budget` + `NoMoneyException` 三件套 = 业内独家 cost-aware multi-agent**(`team.py:92-100`,超预算自动 raise)
- **3 RoleReactMode**(REACT / BY_ORDER / PLAN_AND_ACT)
- **Action vs Tool 双层抽象**(Action = role 职责 / Tool = 底层执行)
- **7 种 environment**(software / android / mgx / minecraft / stanford_town / werewolf / api)
- **`ext/` 7 学术领域**:**AFlow ICLR 2025 oral**(_"Automating Agentic Workflow Generation"_,ranking #2 LLM-based Agent) / **Stanford Smallville Generative Agents** / **Werewolf 经典 multi-agent benchmark** / **SPO arxiv 2502.06855 + AOT 2502.12018**
- 商业产品 **MGX**(`mgx.dev`,2025-02 launch,_"world's first AI agent development team"_)
- Team.serialize / deserialize 完整 → `SERDESER_PATH/team/team.json`

**[[concept/agent-team-patterns]]** synthesis 核心:

**3 条 multi-agent 哲学路线**(同问题 3 个不同 metaphor):
- CrewAI **Task-driven**(任务驱动)
- AG2 **Conversation-driven**(对话驱动)
- MetaGPT **Role-play-driven**(角色扮演 + SOP)

**4 大组织模式 × 9 项目分布矩阵**:Sequential / Hierarchical / GroupChat / Role-play

**5 个 production engineering wisdom**:
1. **预算 enforce**(MetaGPT 独家:investment + NoMoneyException)
2. **5+1 speaker selection**(AG2 独家:5 模式 + Callable + eligibility filter)
3. **状态机式 transition 控制**(AG2 独家)
4. **Task DAG via 实例引用**(CrewAI 独家:类型安全)
5. **SOP + 预定义 role**(MetaGPT 业务方友好极致)

**R-TEAM-1~R-TEAM-8 综合借鉴清单**(汇总 R-CW + R-AG + R-MG):
- 🔴 P1 必做 2 条:**R-TEAM-1** investment + NoMoneyException 预算 enforce(1-2 周,production multi-agent 必备)/ **R-TEAM-2** Task DAG via `context: list[Task]`(< 1 周,类型安全)
- 🟡 P2 应做 3 条:R-TEAM-3 Process/RoleReactMode enum / R-TEAM-4 Speaker selection 5+1 / R-TEAM-5 Auto selection 4 retry prompt template(< 半周)
- 🟢 P3 远期 3 条:R-TEAM-6 Action vs Tool 双层 / R-TEAM-7 状态机 transition / R-TEAM-8 AFlow + Werewolf benchmark

**⚠️ 不该跟**:
- 庞大基类 inheritance(AG2 ConversableAgent 4577 行违反 composition over inheritance)
- 20+ 特化 role / 20+ contrib agent(过度 niche)
- 商业 SaaS 上报(CrewAI Plus / MGX)
- SDLC 预定义 role marketplace(MetaGPT 太特化)

**meta(关于 wiki 自己)**:Depth-first 写法验证扩 4 版,各侧重:
- 写法 #1 [[concept/runtime-context-injection]] — 协议(`<system-reminder>`)
- 写法 #2 [[concept/agent-state-separation]] — 抽象(StateModule)
- 写法 #3 [[concept/rag-design-space]] — 跨多 project synthesis(6 RAG framework × 8 维 + ICLR'26 paper calibration)
- **写法 #4 [[concept/agent-team-patterns]] — 跨多 project + 已 ingest 多 harness synthesis**(3 个新 ingest + 5 个已 ingest = _"哲学路线 + 组织模式 + production wisdom"_ 三层 synthesis)

**关键反直觉发现**:_"Multi-agent 框架根本差异不在有没有 agent class,而在用什么 metaphor 组织 agent"_(Task / Conversation / Role-play 是 3 个不同 mental model,引出完全不同的 abstractions + APIs + ergonomics)。**任何 framework 设计第一步是选 metaphor**,后面 API 都跟着 metaphor 自然涌现。

**takeaway**:Agent Team / Manager 主线交付完整。**3 个新 ingest(CrewAI / AG2 / MetaGPT)+ 1 concept synthesis = 跨 9 个项目的 multi-agent 全景**(3 新 + 5 已 ingest + 1 LangGraph 间接)。Wiki multi-agent 章节从 _"primitive only"_ 扩到 _"Team / organization 也完整"_。SkillForge 远期 multi-agent 路径有 8 条具体借鉴(R-TEAM-1~8)+ 2 条 P1 直接打到生产(R-TEAM-1 预算 enforce + R-TEAM-2 Task DAG)。

---

## [2026-05-17] ingest+update | Claude Code `/insights` 命令深度 verify + Layer E 校准

**触发问题**: 用户 wiki-query 问 _"现在归因都是怎么做的?"_ → 讨论到 Layer E _"自动改用户 surface"_ 闭环 → 用户提到 Claude Code 有个 insight 能力 _"感觉是我想要的"_ → 让我去网上 + SDK 找。结果发现 Claude Code v2.1.88 内置 `/insights` slash command(`src/commands/insights.ts` 3200 行单文件),wiki 原标 `[community-knowledge]`,本次正式 verify 升级为 `[verified-from-source]`。

**改动**:
- 新建 [[source/claude-code-insights]] B 模式 catalog(280 行,30+ file:line 锚点)
- 新建 [[deep/claude-code-insights]] depth-first 单机制深挖(341 行,跟 8 个已有 deep page 同 quality)
- 更新 [[harness/claude-code]] §self_evolution insights 章节 — 从 `[community-knowledge]` 推断升级为 `[verified-from-source]` 完整实现描述
- 更新 [[concept/attribution]] §Layer E — 加 Claude Code `/insights` 作为 **路线 B 用户驱动 review**(对照 SkillForge V3+V80 **路线 A 自动 canary**),完整业内 Layer E 9 项目矩阵
- 更新 [[index]] — 新增 deep / source 两行

**关键 verified 事实**(grep + read `src/commands/insights.ts:1-3200`):

**核心抽象 — 3 层数据模型**:
- **SessionMeta**(`:228-258`)— deterministic stats(tool_counts / languages / git_commits / tokens / response_times / interruptions / tool_errors / multi_clauding 时间戳)
- **SessionFacets**(`:260-273`)— LLM-extracted qualitative(goal_categories / outcome / satisfaction / friction / helpfulness / session_type 6 维 + 自由文字 + ⭐ `user_instructions_to_claude: string[]` 给 MULTIPLE TIMES 算法)
- **AggregatedData**(`:275-326`)— 跨 session 聚合 30+ 字段

**6 维 55 typed enum 完整 LABEL_MAP**(`:352-415`):
- **goal_categories** 13 类(debug_investigate / implement_feature / fix_bug / refactor_code / configure_system / create_pr_commit / analyze_data / understand_codebase / write_tests / write_docs / deploy_infra / warmup_minimal / ...)
- **friction** 12 类(misunderstood_request / wrong_approach / buggy_code / user_rejected_action / claude_got_blocked / user_stopped_early / wrong_file_or_location / excessive_changes / slow_or_verbose / tool_failed / user_unclear / external_issue)
- **satisfaction** 8 类(frustrated / dissatisfied / unsure / neutral / likely_satisfied / satisfied / happy / delighted)
- **outcome** 5 类(fully/mostly/partially/not_achieved + unclear_from_transcript)
- **helpfulness** 5 类(unhelpful / slightly_helpful / moderately_helpful / very_helpful / essential)
- **session_type** 5 类(single_task / multi_task / iterative_refinement / exploration / quick_question)
- **success** 7 子类(fast_accurate_search / correct_code_edits / good_explanations / proactive_help / multi_file_changes / handled_complexity / good_debugging)

**业内对照**:Claude Code 55 enum > AgentOps 15 > tau-bench 7 — **业内 Layer D typed enum schema 最完整 OSS 实现**。

**FACET_EXTRACTION_PROMPT** 4 条 CRITICAL GUIDELINES(`:430-456`):
1. goal_categories 只数用户 explicit 请求(防 Claude autonomous 探索污染)
2. satisfaction 按文字字面 signal(_"Yay!"_ → happy / _"I give up"_ → frustrated)
3. friction 5 类显式列举 + 描述
4. warmup_minimal 边界 case 单独归类

**8 INSIGHT_SECTIONS 并行 LLM call**(`:1336-1495`):
- project_areas / interaction_style / what_works / friction_analysis / **suggestions ⭐** / on_the_horizon / fun_ending(+ Ant-only:cc_team_improvements / model_behavior_improvements)

**MULTIPLE TIMES algorithm 核心 wisdom**(`:1430` prompt 注释):
> _"PRIORITIZE instructions that appear MULTIPLE TIMES in the user data. If user told Claude the same thing in 2+ sessions (e.g., 'always run tests', 'use TypeScript'), that's a PRIME candidate — they shouldn't have to repeat themselves."_

**核心机制 — `suggestions` section 输出 3 类**:
- `claude_md_additions`(addition + why + prompt_scaffold)— 直接告诉用户 CLAUDE.md 加什么
- `features_to_try`(从 5 个 CC FEATURE REFERENCE:MCP / Custom Skills / Hooks / Headless Mode / Task Agents 选)
- `usage_patterns`(title + suggestion + copyable_prompt)

**Cache 策略 + 性能**:
- `~/.claude/usage-data/{session-meta,facets}/<session_id>.json` per-session 永久 cache
- 首次跑 N=100 sessions:~$5 + 10 min(50 sessions 没 cache × Opus call)
- 后续跑 N=100(95 cached + 5 新):~$1 + 30 秒
- INSIGHT_SECTIONS 每次重跑(基于最新 aggregate)

**detectMultiClauding 滑窗算法**(`:1062-1143`):30 min OVERLAP_WINDOW_MS,sliding window 检测 pattern _"s1 → s2 → s1"_,业内独家检测多 Claude 并发会话。

**Opus model 而非 Haiku**(`:41-48`)— 2 个独立 getter `getAnalysisModel` + `getInsightsModel` 都返 `getDefaultOpusModel()`,Anthropic 不省成本。

**Remote host data collection**(`:60-220`)— Ant-only SCP 拉远程 Coder workspace `~/.claude/projects/`,跨设备数据汇总。

**InsightsExport + S3 upload**(`:2655-2737`)— Ant-only 后台 upload 给 Anthropic 产品团队分析(`cc_team_improvements` + `model_behavior_improvements` 两 section 就是给团队看的)。

**设计哲学** — _"用户驱动 self-improvement"_:
- LLM 生成 `claude_md_additions` 建议
- **不自动改 CLAUDE.md**(用户 100% 决策)
- 3 个 reasons:侵入性 = 信任成本 / CLAUDE.md 是用户产物 / 建议 vs 决策分离

**Layer E 校准**:Layer E 不是 SkillForge 唯一独家,**Anthropic 同时 ship 两条 self-improvement 路径**:
- `/insights` = 用户驱动 review(_"建议而非改"_,适合 CLI 工具改 user config)
- autoDream = 自动后台 memory consolidation(详见 [[../deep/claude-code-autodream-3-gate]])
- SkillForge V3+V80 独家在 _"自动 multi-surface + canary + verify 完整闭环"_(改 system 内部 surface)

**Layer E 9 项目矩阵**(添加到 [[concept/attribution]] §Layer E):

| 项目 | 完成度 | 改什么 surface | 决策权 |
|---|---|---|---|
| SkillForge V3+V80 | ⭐ 完整闭环 | skill/prompt/behavior_rule/hook | 系统 |
| **Claude Code `/insights`** | ⭐ ①②③ 完整 + ④ 建议 | **CLAUDE.md 用户文档** | **用户 100%** |
| AgentScope Trinity-RFT | ⭐ 改 weights 闭环 | model weights(RL fine-tune) | 系统 |
| MetaGPT AFlow(ICLR'25 oral)| 🟡 workflow 自动生成 | agent workflow | 系统 |
| DSPy GEPA / Opik Optimizer | 🟡 改 prompt | prompt | metric 决定 |
| Cognee memify | 🟡 memory feedback loop | memory weight + KG | 系统 |
| OpenClaw Memory Dreaming | 🟡 memory promotion | memory state | 系统 |
| Claude Code autoDream | 🟡 memory consolidation | memory file | 系统 |
| Hermes commit_memory_session | ❌ Layer C only | memory extract | 用户手动 |

**社区生态**(围绕 `/insights` 补 ④⑤⑥):
- `yahav10/claude-insights` CLI:把 HTML report 转 actionable artifact
- `rebelytics/one-skill-to-rule-them-all`:meta-skill 持续观察
- `aviadr1/claude-meta`:CLAUDE.md 当 learning system
- MindStudio `learnings.md` pattern:session start 读 + end 更新

**R-INSIGHT 借鉴清单(给 SkillForge)**:
- 🔴 P1 **R-INSIGHT-1** MULTIPLE TIMES algorithm(1-2 周,SkillForge 已有 V80 state machine,只加 _"跨 session 重复识别"_ 上游 source)
- 🔴 P1 **R-INSIGHT-2** Per-trace LLM-extracted facets 永久 cache + aggregate regenerate(< 1 周)
- 🟡 P2 **R-INSIGHT-3** 6 维 typed enum schema 给 ATTRIBUTION-AGENT 输入更结构化(1 周)
- 🟡 P2 **R-INSIGHT-4** _"建议而非改"_ 路径(SkillForge dashboard 加 trace pattern review 入口,2-3 周)
- 🟢 P3 R-INSIGHT-5 detectMultiClauding 滑窗算法(短期参考)

**R-INSIGHT-1 + R-INSIGHT-2 = SkillForge Layer E 从 _"自动单路线"_ 升级到 _"自动 + 用户驱动双路线"_,共 2-3 周工作量**。

**meta(关于 wiki 自己)**:Depth-first 写法第 9 个 deep page,首次出现 _"原 `[community-knowledge]` 升级为 `[verified-from-source]`"_ 路径 — 验证了 Iron Law B 模式 source-grounded 是 wiki 长期价值,community knowledge 一旦有源码必须 verify。**关键收获**:_"reading source 比 reading docs/blog 价值高 10 倍"_ — 3200 行源码读完才发现 6 维 55 enum + MULTIPLE TIMES + per-session cache + Opus 选择等 production wisdom,blog 都看不到。

**takeaway**:Claude Code `/insights` 完整深度 ingest 完成。**用户驱动 Layer E** 是 Anthropic 押注的 _"agent self-improvement"_ 第 2 条路径,跟自动 autoDream 互补,跟 SkillForge V3+V80 _"自动 canary 改 system surface"_ 是不同产品定位的不同设计哲学(_"建议而非改"_ vs _"自动 + verify"_)。**没有最优路线,两种都要**。SkillForge 远期补 _"建议而非改"_ 路径(R-INSIGHT-4)+ MULTIPLE TIMES algorithm(R-INSIGHT-1)是 quick win。

---

## [2026-05-17] meta | 新增 [[backlog]] — agentic 缺口结构化清单

**触发问题**:_"看下我们当前还有哪些项目没有囊括在内,尤其是 agentic 相关的"_ — 用户想先把缺口模块准备好,再一个一个分析。

**改动**:
- 新建 [[backlog]](meta 类型,wiki 根目录)— 跟 [[log]]/[[index]] 同级
- 索引 ~70 条候选 ingest 目标,按 P0/P1/P2/parked + planned/in-progress/done/parked 状态机管理
- 用户钦点 2 条 P0:
  1. `concept/sandbox-isolation` — 6 个 harness 引用 sandbox 但无横切 anchor 页(codex 3 平台 / ragflow gVisor / hermes 7 backends / opencode shell+pty / openclaw Dockerfile / dify plugin),需对照 9 个外部实现(E2B / Daytona / Modal / Vercel Sandbox / gVisor / Firecracker / Cloudflare / WebContainers / bubblewrap+landlock+seatbelt)
  2. `concept/a2a-protocol` + 可选 `deep/google-a2a-spec` — [[harness/agentscope]]+[[harness/crewai]] 都 verify 集成但无独立协议页,需跟 MCP/ACP/AGNTCY/AG-UI 横向定位
- 6 大维度缺口梳理:
  - Harness:Aider/OpenHands/smolagents/LangGraph/OpenAI Agents SDK(P0)+ 10 条 P1 + 10 条 P2 + 8 条闭源 doc-only
  - Eval:Braintrust/Inspect AI/Weave(P0)+ 5 条 P1
  - Benchmark:AgentBench/GAIA/OSWorld/WebArena/AgentDojo(P0)+ 14 条 P1 — wiki 现在 agentic benchmark 维度大量空白
  - RAG:Haystack/LightRAG/HippoRAG/R2R 等 7 条
  - 协议:MCP deep + agent-protocols 4 层栈(P0)+ Computer Use 等
  - Memory + 已有素材升级(Hermes 12 章 / Workspace Agents / Swarm)
- [[index]] Meta 区加 backlog 入口

**takeaway**:wiki 第一次有了**未完成计划清单**(向前看)— 之前只有 log(向后看)+ index(已存在)。这个补全是 wiki 进入 _"持续 ingest 模式"_ 的前提:LLM 下次进来不用再扫一遍 _"还缺什么"_,直接取 P0 planned 条目走 `/wiki-ingest`。Sandbox + A2A 两个 concept 页是**当前最高 ROI 的横切补全**(横向连接已 verify 的 6+2 个 harness 引用,把 dangling reference 收紧成 anchor)。

---

## [2026-05-17] ingest | [[concept/sandbox-isolation]] — backlog P0 #1 起手

**触发问题**:_"6 个 harness 页 sandbox 字段已 verify 但相互独立,缺横向 anchor:gVisor / Firecracker / V8 isolate / OCI 容器 / Linux namespaces 谁是同类,SkillForge 选哪条?"_ — 来自 [[backlog]] §0.1。

**改动**:
- 新建 [[concept/sandbox-isolation]] — 8 维设计空间 + 6 阶梯隔离机制 + 6 harness verified + 9 外部 doc-only 对照矩阵 + Trade-off 横切 + SkillForge 条件性选型 + 学习视角三段
- 更新 [[backlog]] §0.1 + §8 sandbox-isolation 状态:planned → done(2026-05-17)
- 更新 [[index]] Concept 区索引行
- 6 个 harness 反向加链(在 sandbox 段加 `详见 [[../concept/sandbox-isolation]]`):codex / ragflow / hermes-agent / opencode / openclaw / dify

**模式**:hybrid(B-mode for 6 wiki harness anchors,直接复用 verified file:line / doc-only for 9 external systems via WebFetch 2026-05-17)
**已读源码**:6 个 wiki harness 页(复用 anchors)+ 9 个外部 doc URL ~10K tokens

**关键发现**:
1. **6 阶梯隔离机制有明确顺序**:0 无隔离 → 1 Linux namespaces(bwrap+landlock)→ 2 macOS Seatbelt/Windows AppContainer → 3 OCI 容器 → 4 gVisor userspace kernel → 5 microVM Firecracker。5' V8 isolate / WebAssembly 是单语言层,**不能单独承担 sandbox 责任**(必须叠加在 1-5 之上)。
2. **重要校准 — Cloudflare Sandbox 用 container,不是 V8 isolate**(2026-05-17 doc 明示)— 修正常见 community 误解。
3. **Firecracker 125ms cold start + <5 MiB overhead 改变 _"sandbox 必须慢"_ 的工程默认假设** — 反直觉地比 Docker 快、比 VM 轻,直接催生 E2B / Vercel Sandbox / Cloudflare Workers / Fly.io 整批产品。
4. **业内 sandbox 阶梯分布显著不均**:
   - 1-2(OS 系统调用层):[[harness/codex]] 一家
   - 3(OCI 容器):[[harness/openclaw]] + Daytona + Cloudflare Sandbox
   - 4(gVisor):[[harness/ragflow]] 业内 RAG framework 唯一
   - 5(microVM Firecracker):E2B / Vercel Sandbox / Fly.io / AWS Lambda — **cloud AI sandbox 清一色 Firecracker**
5. **[[harness/hermes-agent]] 7 backends SPI 抽象** 是承认 _"sandbox 不是技术问题,是部署问题"_ 的工程哲学差异 — 不做 implementation,做 abstraction。
6. **[[harness/dify]] _"承认不做"_** — 用架构约束(workflow 不跑代码)规避整块 sandbox 复杂度,是合法工程决策。
7. **[[harness/codex]] `policy_transforms.rs` 业内唯一显式 OS abstraction layer** — 3 平台真做完(Linux bwrap+landlock / macOS 3 个 .sbpl / Windows 独立 crate)+ V8 isolate 作为第二层叠加,**是 _"open-source coding agent 跨平台 sandbox 模板"_**。

**SkillForge 启示**:`Code Sandbox` 实现路径暂标 `?`(用户对齐:后续读 `/Users/youren/myspace/skillforge` 源码再补)。条件性建议清单:
- 场景 A(只调 typed tool):不上 sandbox(Dify 路线,工程成本 0)
- 场景 B(LLM-generated shell):bwrap+landlock+seatbelt + 跨平台抽象(Codex 路线)
- 场景 C(用户代码 single-tenant):**gVisor**(RAGFlow R-RF-2 路线,production-ready 最低门槛)← 最有可能推荐
- 场景 D(SaaS multi-tenant):**Firecracker microVM 或接 E2B/Vercel** — 不自建,接 SaaS 工程门槛低 10x
- 场景 E(V8 Code Mode):**不推荐** — 工程成本极高且仍需外层 OS sandbox 兜底,ROI 不如 C/D

**takeaway**:**Sandbox 是 trust model 的物理实例化** — 同一个 harness 在 _"信任本地用户 / 信任 LLM 输出 / 不信任跨租户"_ 三种 trust model 下选不同阶梯,这是 wiki 第一次把 sandbox 拉回 _"决策树根 = trust model"_ 这个第一性。**最高 ROI 的工程胜利是 Firecracker microVM 的 125ms cold start + 5 MiB overhead** —— 让 _"per-request fresh sandbox"_ 从理论变现实,改变了 _"sandbox 慢且重"_ 的默认认知。这页是 [[backlog]] P0 #1,起手后续 #2 = `concept/a2a-protocol`。

---

## [2026-05-17] ingest | [[concept/agent-protocols]] — backlog P0 #2 起手(扩为 5 层栈)

**触发问题**:_"wiki 18 个 harness/eval 页频繁引用 MCP / 4 页引用 A2A / 21 页引用 OTel,但协议本身缺 anchor —— 5 个 agent 互通协议(MCP / A2A / ACP / AGNTCY / AG-UI)+ OTel telemetry 的定位关系是什么?谁是同层谁是上下层?ACP 还在吗?"_ — 来自 [[backlog]] §0.2 + §5(本次合并解决,a2a-protocol + agent-protocols 同一页)。

**改动**:
- 新建 [[concept/agent-protocols]] — **5 层协议栈** anchor:5 协议详节(MCP/A2A/AGNTCY/AG-UI 各占大节 + OTel 侧向贯穿)+ 8 维设计空间矩阵 + 4 个 wiki 内集成阶梯对照 + Trade-off 横切 + SkillForge 5 协议条件性建议 + 学习视角三段 + 7 项待回答问题
- 更新 [[backlog]] §0.2(a2a-protocol)+ §5(agent-protocols)+ §8 状态:planned → done(2026-05-17,合并到同一页)
- 更新 [[index]] Concept 区
- 反向加链 [[concept/agent-interop]] §定义(下一步)+ [[concept/observability]] OTel 段(下一步)

**模式**:hybrid(21+ wiki 内 verified 引用直接复用 + 5 个外部 protocol spec doc-only WebFetch 2026-05-17)
**已读数据**:18 MCP 集成页 + 4 A2A 集成页 + 21 OTel 集成页(grep)+ 6 外部 doc(modelcontextprotocol.io/introduction + spec/2025-06-18 / a2a-protocol.org/latest / agentcommunicationprotocol.dev / agntcy.org / docs.ag-ui.com / opentelemetry.io/docs/specs/otel + semconv/gen-ai)~40K tokens

**用户钦点扩展(2026-05-17)**:原本计划 4 层栈(MCP/A2A/AGNTCY/AG-UI),用户提示 _"OTel 也是协议层"_,扩为 **5 层栈**(4 互通 + 1 贯穿)。OTel wiki 内引用 21 页超过 MCP 18 页,是 wiki 内引用最广的协议。

**关键发现**:
1. **5 层栈定位明确,4 互通 + 1 贯穿**:Tool(MCP)/ Agent(A2A)/ 编排(AGNTCY)/ UI(AG-UI)各占一层,OTel 侧向贯穿。
2. **ACP 已 merge 进 A2A under Linux Foundation**(2026-05-17 a2a-protocol.org 明示) — 修正常见 _"5 个协议碎片化"_ 误解。Linux Foundation 扮演 _"协议聚合 hub"_。
3. **OTel GenAI semconv 已经把 MCP semantics 纳入** —— 协议之间 _"我覆盖你的语义"_ 是 elegant 的合作姿态,跟 _"我替代你"_ 完全不同。
4. **3 个协议联盟健康姿态证据**:(a) ACP→A2A 合并 / (b) AGNTCY 明示 _"middleware above A2A/MCP"_ 不替代 / (c) OTel GenAI semconv 纳入 MCP semantics — 在标准之争史上罕见。
5. **MCP 1 年从发布到事实标准**(2024-11 → 2026-05 wiki 18 家集成) —— USB-C 类比 + 3+3 primitives 对称 + LSP-inspired + Anthropic dogfood 4 个因素共同促成。
6. **MCP 6 primitives 对称设计**:Server 3(Resources/Prompts/Tools)+ Client 3(Sampling/Roots/Elicitation) —— 避免 server/client 不对等,允许 server 反向 query。
7. **MCP 安全 4 原则文档明示 _"protocol cannot enforce"_** —— 把协议局限写进 spec,implementor MUST 落实。这种诚实姿态罕见。
8. **A2A 协议层强约束 _"不共享 internal memory/tools/logic"_** — 跟 [[concept/multi-agent-isolation]] _"完全隔离"_ 路线对应,**协议层就站完全隔离一边**。
9. **AG-UI human-in-the-loop interrupts 是协议级一等公民** —— 跟 [[harness/codex]] `request_user_input` 协议事件同思路,把 _"人介入"_ 做成 protocol-level 而不是后置约束。
10. **OTel GenAI semconv 还在 Development 状态** —— _"AI agent 的 telemetry 标准化滞后于 agent 协议化"_,MCP 已 production 18 月,GenAI semconv 还在草案。**先有 agent → 才有 agent telemetry standardization** 是 _"协议演化次序"_ 的常态。
11. **AG-UI 1st-party 集成 10+ framework 是惊人 traction**(Microsoft / Google ADK / AWS / Mastra / Pydantic AI / Agno / LlamaIndex / AG2),做法是 _"协议先 partner 再 spec"_,CopilotKit 拉 LangGraph/CrewAI 入伙后发协议。

**SkillForge 启示**(5 协议条件性建议,待 SkillForge 源码 verify 后升级为 R-PROTO-N):
- 🔴 P0 必做:**MCP**(wiki 18 家事实标准,不集成 = 跟不上 baseline)+ **OTel**(wiki 21 家最广,跟现有 [[concept/observability]] Wide Events 2.0 路线兼容,GenAI semconv 含 MCP)
- 🟡 P1 应做:**A2A**([[harness/agentscope]] + Nacos 跨进程是 SaaS multi-tenant agent team 现成方案)
- 🟢 P2 观察:**AG-UI**(如 SkillForge 有 frontend agent 交互需求,低风险标准选择)
- ⚪ P3 不必跟:**AGNTCY**(middleware 层工程门槛高,要 Cisco-level multi-org 部署才需要)

**takeaway**:**Agent 协议栈是 2025-2026 协议联盟健康姿态最好的例子** — 5 个协议明示彼此 _"complementary"_ 不竞争 + ACP→A2A 合并 + OTel 纳入 MCP semantics 两个合并案例,反面是 W3C / OpenStack 早期 API 碎片化。**SkillForge 必做 P0 双协议是 MCP + OTel** — 两个 wiki 内引用最广的协议,行业已经形成事实标准,不接 = 跟不上 baseline。**最有指导意义的洞察是 _"5 个协议之所以能共存,是因为它们各自承认自己的局限,且 Linux Foundation / CNCF 做协议聚合 hub"_** — 这是协议政治学的健康姿态,值得 SkillForge 内部多协议路线(provider SPI / channel SPI 等)借鉴。这页是 [[backlog]] P0 #2,起手后续 #3 = harness/aider 或 #4 = `deep/mcp-spec-deep` deep dive。

---

## [2026-05-18] update | [[concept/agent-protocols]] §10 SkillForge 选型扩展 — 工程边界 + skill 封装决策

**触发问题**:用户连环 3 问 — _"(1) A2A 也是把 agent 调用注入 function_calling 吗?(2) CC TeamCreate/SendMessage 遵循 A2A 吗?(3) 现在把调 agent 封装成 skill 后,A2A 还有用吗?"_ — 直接戳到协议存在的工程边界,需要把答案 anchor 进 wiki。

**改动**:
- [[concept/agent-protocols]] §10 新增 5 个子节:
  - §10.4 工程决策框架 — **3 个边界**(同进程 / 同信任域跨进程 / 跨信任域)mental model
  - §10.5 A2A 落地两条路线 — **路线 A agent-as-tool(crewai)vs 路线 B agent-as-actor(agentscope)**,两条路线 A2A 语义保留度对照
  - §10.6 **校准:CC Coordinator ≠ A2A**(消除常见误解)— 5 维对照表(传输 / 发现 / 信任模型 / Task 状态机 / 部署边界)
  - §10.7 **skill 封装 vs A2A 决策表** — 7 场景判定 + 3 个 skill 搬不过来的 A2A 设计点(INPUT_REQUIRED 反向 query / IP 保护 / 公开 discovery)+ SkillForge _"现在不必急,但保留切换点"_ 3 条建议
  - §10.8 **时间线判断** — 2026 不必 / 2027-2028 必选 / 2030+ marketplace 基础设施
- [[concept/multi-agent-isolation]] §Coordinator Mode 反向加链 `[[agent-protocols]] §10.6` 校准

**模式**:对话式深度 verify(用户连环 3 问驱动 + grep [[harness/claude-code]] Coordinator Mode + agentscope/crewai A2A 集成路线 + wiki 内 already-verified facts)
**已读数据**:[[harness/claude-code]] TeamCreate/SendMessage/Coordinator grep + [[concept/multi-agent-isolation]] Coordinator 段 + 已有 A2A 5 段 WebFetch 数据 ~5K 增量

**关键发现**:
1. **MCP 跟 A2A 在 function_calling 适配上不对等** — MCP fine-grained per-tool 天然装得进;A2A 三种 invocation 模式只有 `message/send` 同步模式装得进 function_calling,`message/stream` SSE 和 webhook push **装不下 INPUT_REQUIRED 反向 query 语义**。
2. **A2A 业内有两条落地路线**:agent-as-tool(crewai 路线,降级)vs agent-as-actor(agentscope 路线,完整语义)。两条路本质 trade-off — **简单 vs 语义完整**。
3. **CC TeamCreate/SendMessage 不是 A2A** — 是 in-process `<task-notification>` XML 协议,**同信任域 + 同进程**;A2A 假设跨信任域 + 跨网络。**解决相同问题但 transport 完全不同**。
4. **协议选型的 mental model 是 _"跨什么边界"_** — 同进程 skill 够 / 同信任域跨进程 MCP-style RPC 够 / 跨信任域 A2A 才是 native。
5. **skill 封装搬不过来 3 个 A2A 设计点**:(a) Task 状态机 + INPUT_REQUIRED / (b) 跨组织 IP 保护(skill 内部能看到 prompt)/ (c) `.well-known` 公开 discovery。
6. **SkillForge 现在不必急做 A2A,但要做架构准备**:skill 抽象保留 _"远程/本地"_ 切换点 + skill 元数据对齐 Agent Card schema + skill SDK 支持 long-running affordance。**这 3 条让 _"未来加 A2A 不痛"_**。
7. **A2A 时间线 — 2026 探索期 / 2027-2028 跨厂商 SaaS 集成必选 / 2030+ marketplace 基础设施**。MCP 是 LLM↔Tool 标准化,A2A 是 Agent↔Agent 跨域标准化 — 两者类比 OpenAPI:不强制内部用,但跨域是 default 选择。

**takeaway**:**协议存在的边界 = 跨什么信任域**。MCP/A2A/AGNTCY/AG-UI 都不是 _"必须用"_ 协议,而是 _"跨某种边界时的 default 选择"_。SkillForge 当前阶段不必急于落 A2A,但**skill 抽象设计层应该对齐 A2A 思路**(name + capability + examples + supportedContentTypes + INPUT_REQUIRED 语义),这样未来 _"提供 agent 给外部 SaaS"_ 或 _"集成第三方 agent"_ 时,**几乎零成本切换**。这个洞察补强了 §10 选型节,让 wiki 第一次能精确回答 _"我的产品什么时候必须接 A2A"_。

---

## [2026-05-18] ingest | [[deep/mcp-spec-deep]] — backlog §5 P0 deep dive 起手

**触发问题**:用户连环 4 问 — _"(1) MCP server 反向沟通是不是靠 tool result + agent loop 比 Sampling/Elicitation 主流?(2) tool+skill 已有的情况下 MCP 还有用吗?(3) host 加载 client 时把 toolList 注册到 function_calling — 这流程对吗?(4) LSP 是什么?有把 LSP 作为 agent tool 的,跟 MCP 的灵感渊源讲清楚"_ — 4 个问题都需要 wiki 内 anchor,合并落成 deep 页。

**改动**:
- 新建 [[deep/mcp-spec-deep]] — schema §3.4b deep 页规范 9 节齐全:
  - §1 重要性(3 容易被低估的点)
  - §2 机制详解(2.1 LSP 白话背景 + 2.2 3 角色 + 2.3 6 primitives 含 visibility 区别 + 2.4 lifecycle + 2.5 stdio/Streamable HTTP transport)
  - §3 WHY(LSP 思路 / server 不信任 by default / 6 primitives 对称设计 / LLM-app-agnostic)
  - §4 失败模式(prompt injection via tool desc / 看完整对话泄漏 / DNS rebinding / schema injection / session hijack 5 类)
  - §5 横向校准(vs LSP / vs A2A / vs OpenAPI/gRPC raw / vs LangChain framework Tool)
  - §6 工程实战 4 大反直觉(4 个 Q&A 提炼:tool result + agent loop 主流 / skill+MCP 不互斥 / toolList 7 步注册 / LSP 白话故事)
  - §7 学习视角(显式约定价值 / 协议跟实现层划清 / 反向调用是分水岭 / spec 复杂度 ≠ 工程复杂度 / LSP 第二次胜利)
  - §8 **SkillForge 联动 R-HARNESS-1 🔴 P1 必做** 3 阶段路线 + Phase 4 中期 MCP server 双向暴露
  - §9 引用链路
- 更新 [[backlog]] §5 — `deep/mcp-spec-deep` planned → done(2026-05-18)
- 更新 [[index]] Deep 区加索引行
- 反向连接 [[concept/agent-protocols]] §2 MCP 节作为父概念

**模式**:doc-only(spec 2025-06-18 + 18 wiki harness/eval verified 集成 + 2 个补 verify:CC LSPTool tool 列表 + concept/lsp-integration SkillForge 需求)
**已读数据**:6 个 MCP spec 章节 WebFetch + grep CC LSPTool + grep SkillForge R-HARNESS-1 ~15K tokens 增量

**关键发现(4 反直觉)**:
1. **Sampling/Elicitation 生态采用慢,主流走 tool result + agent loop** —— `isError: true` + 描述性 text 让 LLM 自己在下一个 agent loop iteration retry,client/server 都零实现成本。Sampling/Elicitation 真正不可替代场景只有 2:(a) server 内 recursive agent 用 host LLM 推理(无自家 API key)/ (b) 敏感 token 走 UI 不走对话历史。
2. **MCP vs skill 不互斥,Claude Code 三者同时跑**:builtin tools(40+ 原生)+ Skills(自定义同进程)+ MCP client(接外部)。**MCP 价值不在 _"提供新能力"_,在 _"标准化跨进程暴露能力"_**。
3. **toolList 注册流程 3 个易错点**:(a) 数据源是 Server 不是 Client(Client 只是 transport 代理);(b) 多 Server 同名 tool host 加前缀 disambiguate(`server_a__read_file`),**spec 没规定**;(c) **只有 Tool 进 function_calling**,Resource 进 context,Prompt 用户触发 — **3 primitives 不同 visibility 是 MCP 精细化设计**。
4. **LSP 是 MCP 直接灵感源**:2014 M×N 噩梦(10 语言 × 6 IDE = 60 plugin)→ 2016 LSP _"前厨后厨分离"_ → `rust-analyzer` 1 个 server 跨所有 IDE 复用 → M+N。2024-11 MCP 把同型问题搬到 LLM 域:**M capability × N LLM app → M+N**。**MCP spec 直接借走 4 设计**:JSON-RPC over stdio + `initialize` + capability negotiation + stateful session。
5. **LSP 第二次胜利 — 2025 agent 借用 LSP 拿代码语义**:opencode `lsp/` 6 模块 first-class(`tool/lsp.ts` 9 LSP operations 暴露给 agent)+ CC `LSPTool`(wiki 未深挖)。grep / regex 派 vs LSP 派的精度差 — _"rename User.id → user_id"_ 时 grep 300 处误改 vs LSP 精确 12 处。
6. **MCP 跟 LSP 信任模型关键差异**:LSP server **trusted**(开发者主动装),MCP server **untrusted**(假设任意第三方)。所以 MCP 加 3 道安全栏 — server 不能看完整对话 + tool annotation 默认不可信 + user consent flow。
7. **MCP `Streamable HTTP` 单 endpoint 双模式工程胜利** — POST 既可同步返 JSON 又可打开 SSE 流 + Mcp-Session-Id session 管理 + SSE event ID Resumability + 4 安全 MUST(防 DNS rebinding 攻击 localhost server)。

**SkillForge 联动**:Wiki R-HARNESS-1 🔴 P1 必做(各 1-3 周)— LSP 接入 SkillForge tool 层。**双路线借鉴**:opencode(已 verify 完整 LSP-as-agent-tool)+ CC LSPTool(wiki 待补 verify)。中期 Phase 4 可选 _"SkillForge skill → MCP server 暴露"_ 跨 LLM host 复用,跟 LSP `rust-analyzer` 跨 IDE 复用同型。

**takeaway**:**MCP 不是简单的 tool API 标准,是 LSP 思路在 LLM 时代的精确复用 + 安全演化版**。**LSP 第二次胜利证明协议跨场景复用价值 9 年不衰**。**spec 给了 Sampling/Elicitation 精细机制但生态主流走 tool result + agent loop** — 这是 SkillForge 接入 MCP 的实操指南:client 实现先支持 tool result error,Sampling/Elicitation 当 P2。这页是 wiki 第一篇 **doc-only deep page**(之前 9 篇都是 source-grounded),证明 _"协议 spec + verified 集成"_ 也能撑 deep 深度,只要锚点够多(18 wiki 集成 + LSP 双重渊源 + SkillForge R-HARNESS-1 联动)。下一步可做 `deep/google-a2a-spec`(把 A2A 学习版讲解落成对称 deep 页)。

---

## [2026-05-18] ingest | [[deep/google-a2a-spec]] — backlog §0.2 + §5 deep dive 完成 + 协议栈双 deep 对称

**触发问题**:用户继上一轮 _"看下 mcp 协议"_ 后说 _"继续吧"_ — 把之前 A2A 学习版讲解落成跟 [[deep/mcp-spec-deep]] 对称的 deep 页,**让 wiki 协议栈第 1 层(MCP)+ 第 2 层(A2A)都有深挖**。

**改动**:
- 新建 [[deep/google-a2a-spec]] — schema §3.4b deep 页规范 9 节齐全,跟 mcp-spec-deep 对称结构:
  - §1 重要性(3 容易被低估的点 — A2A 不是 MCP agent 版 / Task 状态机假设人介入是常态 / ACP 合并是 standardization 健康姿态)
  - §2 机制详解(2.1 跟 MCP 根本错位 mental model + 2.2 4 件套抽象 + 2.3 Agent Card 含 RFC 8615 两层 card + 2.4 Task 8 状态机 + 2.5 Message+Part + 2.6 8 JSON-RPC 方法 + 2.7 3 invocation 模式 + 2.8 invoice 完整 walkthrough)
  - §3 WHY(Opaque Collaboration 协议层 IP 保护 / Long-Running by Default / Human-in-the-Loop First-Class / Agent Card 借 OpenAPI / LF 协议聚合 hub)
  - §4 失败模式(跨组织 IP 泄漏 / 长任务断线 / 恶意 Agent Card 注册 / Replay / 同步思维错误用 A2A 5 类)
  - §5 横向校准(vs MCP / vs ACP 合并 / vs REST OpenAPI / vs OpenAI Assistants API / vs ChatGPT Plugins historical)
  - §6 工程实战 wiki 内 2 家落地路线对照(crewai agent-as-tool 降级 vs agentscope agent-as-actor 完整)
  - §7 学习视角(信任边界是设计 root / Long-Running first-class / 协议合并比竞争健康 / Opaque Collaboration 改变集成谈判 / OpenAPI OAuth 经验复用)
  - §8 SkillForge 联动 3 阶段路线(2026 准备期 / 2027 client / 2027-2028 server)+ 跟 [[concept/agent-team-patterns]] R-TEAM-* 联动暗示
  - §9 引用链路
- 更新 [[backlog]] §0.2 — `deep/google-a2a-spec` planned → done(2026-05-18)
- 更新 [[index]] Deep 区加索引行(放 mcp-spec-deep 上方,协议栈第 2 层 deep 在前直观)
- 反向连接 [[concept/agent-protocols]] §3 A2A 节作为父概念(下一步)

**模式**:doc-only(A2A v1.0 spec 3 个章节 WebFetch 已在 §3 阶段 fetched + 4 wiki harness/eval verified 集成 grep + 复用 §10 已经写的 mental model)
**已读数据**:零新增 WebFetch(完全复用上一阶段已 fetch 的 a2a-protocol.org / spec / discovery / streaming-and-async 4 段)+ grep [[harness/agentscope]] A2A 集成段 + [[harness/crewai]] A2A 集成段 + [[eval/ag2]] 对照 + [[concept/agent-state-separation]] 跨进程 isolation 引用 ~5K tokens 增量

**关键发现**:
1. **A2A 跟 MCP 是 _"借用工具" vs "委托任务"_ 根本错位** — 不是 MCP agent 版,是不同问题域。真实 system 经常同时跑两个协议。
2. **Task 状态机 INPUT_REQUIRED + AUTH_REQUIRED 是协议层 _"假设人介入是常态"_** — 跟 MCP `tools/call` _"call→return"_ 完全不同。3 个独立项目(A2A + Codex + AG-UI)同时把 human-in-the-loop 做成 protocol-level,**协议设计共识**。
3. **Agent Card `extendedAgentCard` 两层 card 设计** — public card 露 skill name,鉴权后获取详细 schema/auth。类比 OAuth public discovery vs token-required endpoint,**借力 web 开发者已有 mental model**。
4. **Opaque Collaboration 协议层强制 IP 保护** — `"agents collaborate without sharing internal memory/tools/logic"` 把 _"双方不暴露 prompt"_ 写进 spec,**降低跨组织集成谈判成本**。跟 [[concept/multi-agent-isolation]] _"完全隔离 vs 共享 parent context"_ 直接对应,A2A 协议层就站完全隔离一边。
5. **3 invocation 模式覆盖秒到天级**:同步 `message/send`(秒级) / SSE `message/stream`(分钟级) / Webhook push(小时-天级)。**`tasks/resubscribe` MUST 第一事件返当前 Task 状态** — 防订阅间隙信息丢失,**长任务断线重连工程化的 elegant 设计**。
6. **ACP merge 进 A2A under Linux Foundation** 是 standardization 罕见健康姿态 — LF 扮演 _"协议聚合 hub"_,避免 W3C 式碎片化。**对照 SkillForge:如果同型对手协议出现,主动合并 / 互通比互掐健康**。
7. **同步思维错误用 A2A 是真实工程坑** — 工程师以为 A2A 是 _"agent 版 REST"_,只用 `message/send` 同步,**完全浪费 Task 状态机 + INPUT_REQUIRED + streaming + webhook 4 杀器**。这是 §10.5 路线 A agent-as-tool 降级的根因。
8. **wiki 内 2 家 verified 走完全不同路线 verified 路线 A/B 真实 trade-off**:[[harness/crewai]] agent-as-tool(降级)vs [[harness/agentscope]] agent-as-actor(完整语义)。**SkillForge 选哪条决定跟 [[concept/agent-team-patterns]] R-TEAM-* 耦合深度**。

**SkillForge 联动(本页 §8)**:
- Phase 1 准备期(2026):Skill 抽象保留 _"远程/本地"_ 切换点 + Skill 元数据对齐 Agent Card schema + Skill SDK 支持 INPUT_REQUIRED 语义
- Phase 2(2027):A2A client 接 Devin/Replit/Jules 等第三方 agent SaaS
- Phase 3(2027-2028):A2A server `.well-known/agent-card.json` 暴露 SkillForge skill 给外部 SaaS 客户
- **关键提醒**:SubAgent / Team / A2A 应该作为 _"3 个 isolation 层级"_ 统一设计,**不要做成 3 套独立抽象**

**takeaway**:**A2A 是 _"第一个协议级假设 agent 之间互不信任"_ 的标准** — Opaque Collaboration + Long-Running by Default + Human-in-the-Loop First-Class 三件套从 _"跨信任域"_ 这个 root 长出来。**借 OpenAPI / OAuth 经验复用**(Agent Card / extendedAgentCard 两层)降低 web 开发者学习成本。**ACP 合并进 A2A 案例值得 SkillForge 借鉴**:协议碎片化是生态毒药,主动合并 / 互通比互掐健康。**协议栈 §2 MCP + §3 A2A 双 deep 完成,wiki 协议层结构完整** — 接下来可补 sandbox / a2a /protocols 之外的其他 backlog(harness/aider / harness/openhands / 等)。

---

## [2026-05-18] ingest | [[harness/smolagents]] — backlog §1 P0 起手 + [[concept/code-mode-orchestration]] 升级 cross-harness synthesis

**触发问题**:用户接受推荐 — _"smolagents 是 code-mode 路线 _'理念旗手'_,跟 Codex Code Mode V8 isolate _'工程实现旗手'_ 对照,Python AST interpreter vs JavaScript V8 isolate 两条 code-mode sandbox 路线根本差异是什么?业内罕见 _'AST interpreter 进程内'_ 路线工程上如何兼顾安全 + 0 cold start?跟 [[concept/sandbox-isolation]] 9 外部 sandbox 怎么校准?"_

**改动**:
- 新建 [[harness/smolagents]] — B 模式 source-grounded,frontmatter 13 维齐 + skillforge_takeaway 5 条 R-SMOL-N + learning_takeaway 三段
- 新建 [[source/smolagents]] — 源码摘要 + 7 文件 read 清单 + 30+ file:line 锚点
- 更新 [[backlog]] §1 P0 — smolagents planned → done(2026-05-18)
- 更新 [[index]] Harness + Source 区索引行
- **[[concept/code-mode-orchestration]] §"各 harness 怎么实现" 重大升级** — 从 single-source(Codex 1 家)升级 cross-harness synthesis(Codex + smolagents 2 家),新 11 维矩阵对照(范式 / Sandbox / Sandbox 选项数 / 编排粒度 / 跨调用状态 / 并行 / 流式 / 中断 / 安全防御 / 工具发现 / 定位)
- **[[concept/sandbox-isolation]] §2 阶梯 5' 扩展** — 加 smolagents `LocalPythonExecutor` AST interpreter 作为业内罕见路线 verified anchor,跟 V8 isolate / WebAssembly 同类但独立路线
- **[[concept/sandbox-isolation]] §5 外部矩阵升级** — E2B / Modal 从 [doc-only] 升级 wiki 内 verified 集成(via smolagents `E2BExecutor` + `ModalExecutor`)

**模式**:B 模式 source-grounded
**已读源码**:7 个核心文件(README + agents.py + local_python_executor.py + remote_executors.py + mcp_client.py + memory.py + pyproject.toml)~115K input tokens / 150K 软上限 — 接近上限但安全
**版本**:smolagents 1.26.0.dev0 @ commit 025b6ad(2026-05-14)

**关键发现**:
1. **smolagents = code-mode _"理念旗手"_,Codex = code-mode _"工程实现旗手"_** — 两家形成 Python/JS 双路线对照。smolagents 2024-12 launch blog _"agents that think in code"_ 业内最早公开倡导;Codex 2025 V8 isolate 100+ crates 工程实现深。
2. **业内罕见 AST interpreter 进程内 Python sandbox** — 不是 subprocess + venv + restrict,**是 `ast.parse` + recursive `evaluate_ast` 节点级解释**(`local_python_executor.py:1583`)。**0 cold start + 4 道防御**:`BASE_PYTHON_TOOLS` 40+ 白名单 + `DANGEROUS_MODULES` 9 黑名单(builtins/io/multiprocessing/os/pathlib/pty/shutil/socket/subprocess/sys)+ `DANGEROUS_FUNCTIONS` 黑名单 + `nodunder_getattr` 反射防御 + **4 硬限额**(MAX_OPERATIONS=10M / WHILE=1M / TIME=30s / OUTPUT=50K)。
3. **6 executor `executor_type: Literal[...]` Sandbox 抽象业内最强类型化** — `agents.py:1535` 6 选项(local/blaxel/e2b/modal/docker/wasm),比 [[harness/hermes-agent]] 7 backends string SPI 类型安全。**`create_python_executor()` factory 模式是 production sandbox 模板**。
4. **`FinalAnswerException(BaseException)` 跨 local/remote 一致 control flow** — `local_python_executor.py:1572` 抛 + `remote_executors.py:71 FINAL_ANSWER_EXCEPTION = "FinalAnswerException"` 常量跨边界一致。**elegant Pythonic 跨进程边界抽象**。
5. **`CallbackRegistry` hooks-by-step-type 第 3 种 hooks 哲学** — `memory.py:280` `register(step_cls, callback)` 按 step 类型注册,**不是事件命名(CC 27 PreToolUse/PostToolUse)也不是 lifecycle(Hermes 16)**。跟 5 step types 1:1 绑定,**新加 step type 自动有 hook 槽位**。
6. **`AgentMemory.return_full_code()` 拼接所有 code_action 成单 script** — `memory.py:273` code-mode 独有 replay/debug 设计,**JSON tool_use harness 拼不出来**。
7. **MCP `structured_output` FutureWarning v1.25 默认改 True** — `mcp_client.py:91-100` 显式追 MCP spec 2025-06-18 outputSchema,**协议演化对接证据**,跟 [[concept/agent-protocols]] §2 MCP 节联动。
8. **HF Hub 持久化路线业内独家** — `agent.push_to_hub("user/agent_name")` / `from_hub` / `from_space` 借力 HF model registry。
9. **`managed_agents + remote executor` 已知缺口**(`agents.py:1608` raise Exception)— smolagents 工程诚实,显式 raise 而非偷偷凑合;**对 SkillForge 是反例,必须设计 multi-agent + 远程 sandbox 兼容**。
10. **`additional_authorized_imports = ["*"]` warning**(`agents.py:1578-1582`)— 用户主动放权时 logger.log 提醒,**安全姿态诚实**。
11. **极致 minimalism — 19 文件 ~13K LOC**;`agents.py` 自称 _"~1000 lines"_ 实际 1814,但核心抽象保留 minimalism(对照 Codex 100+ crates)。

**SkillForge 启示**(R-SMOL-1 至 5):
- 🔴 P1 R-SMOL-1 — `executor_type: Literal[...]` Sandbox 抽象模板(1 周,SkillForge 应 `executor_type: Literal["jvm-native","docker","e2b","gvisor","modal"]` 对应 sandbox-isolation §10 5 场景)
- 🟡 P2 R-SMOL-2 — Code Mode Python 路线 POC 参考(2-4 周技术评估,**不建议直接抄 AST interpreter 到 JVM**,可考虑 GraalVM polyglot 路线)
- 🟡 P2 R-SMOL-3 — `CallbackRegistry` hooks-by-step-type 第 3 种 hooks 哲学(2 周混合 lifecycle + step-type 双 surface)
- 🟡 P2 R-SMOL-4 — `FinalAnswerException` 跨 local/remote 一致 control flow(0.5-1 周)
- 🟢 P3 R-SMOL-5 — `AgentMemory.return_full_code()` replay 思路(可选,只对 code-mode 有用)

**takeaway**:**smolagents 是 wiki 内 verified 第 2 家 code-mode harness(对照 Codex),让 [[concept/code-mode-orchestration]] 从 single-source 升级到 cross-harness synthesis**。最大 insight:**code-mode 两条路线(Python AST interpreter vs JavaScript V8 isolate)在 sandbox 选项数 / 安全防御策略 / 中断协议 / 状态管理 4 个维度都不同,但底层都是 _"承认进程内不安全 → 叠加多道防御"_ 工程诚实姿态**。smolagents 6 executor `Literal` 抽象 + `FinalAnswerException` 跨边界一致协议 + `CallbackRegistry` hooks-by-step-type 第 3 种哲学 都是值得 SkillForge 借鉴的 elegant 设计。**最高 ROI 是 R-SMOL-1 sandbox 抽象**(1 周,直接对齐 sandbox-isolation §10 SkillForge 5 场景)。下一步可做:`harness/aider`(coding agent 老牌 vs smolagents code-mode 对照)/ `deep/smolagents-ast-interpreter`(单点深挖业内罕见 AST interpreter sandbox)/ 或继续 backlog §1 P0(openhands / langgraph / openai-agents-sdk)。


---

## [2026-05-18] ingest | [[harness/openhuman]] — 用户钦点 newly trending 项目

**触发问题**:用户 _"我看有个 openhuman 的项目 你看看呢"_ → web 查证后确认是 tinyhumansai/openhuman(2026-05-13 launch,5 天 GitHub Trending,7.8k stars),跟 backlog §1 P0 `harness/openhands` 不是同一项目。是 wiki 内首家 "Personal AI superintelligence" niche harness。

**改动**:
- 新建 [[harness/openhuman]] — B 模式 source-grounded,frontmatter 13 维完整 + R-OH-1~6 + learning_takeaway 三段
- 新建 [[source/openhuman]] — 8 文件 read + 20+ file:line 锚点
- 更新 [[backlog]] — 用户钦点新条目,标 done(2026-05-18)
- 更新 [[index]] Harness + Source 区索引行

**模式**:B 模式 source-grounded(Early Beta + project-in-flux 双标签)
**已读源码**:8 文件(README + AGENTS.md + Cargo.toml + agent/README + memory/tree/schemas.rs + tokenjuice/mod.rs + 6 处 ls)+ grep cancel/abort + learning/agent README ~150K input tokens(达到软上限,Step 2 紧凑写)
**版本**:openhuman 0.53.49 (core) / 0.53.45 (app),Early Beta status

**关键发现 11 条**:
1. **业内首家 _"Personal AI superintelligence"_ niche harness** — wiki 内 12 harness 都不是这个定位
2. **Memory Tree 17 JSON-RPC controllers + SQLite + Obsidian vault 双写 + 20min auto-fetch** — Karpathy llm-wiki 模式 wiki 内**第 3 家** verified production(对照 openclaw / mempalace)
3. **TokenJuice 业内独家 _"context compression as library + 3-layer rule overlay"_** — `vincentkoc/tokenjuice` Rust port,Builtin via include_str! → User ~/.config/ → Project .tokenjuice/ 优先级递增,v1 library only
4. **Composio 桥接 100+ OAuth _"借平台拿生态"_ 路线** — 自写只 7 个 integration,跟 dify/ragflow 自建路线对比
5. **14 built-in agent archetypes + SandboxMode/ToolScope enum + workspace TOML loading** — 跟 [[harness/claude-code]] subagent 思路一致 Rust 强类型化
6. **`harness/fork_context.rs` KV-cache reuse** — wiki 内**第 2 家 Fork-style multi-agent verified**(对照 CC)
7. **`harness/interrupt.rs` InterruptFence + check_interrupt + InterruptedError** Rust RAII graceful cancellation — 跟 agentscope interrupt 协议思路相邻
8. **Learning 路线业内独特**:不是 prompt-evolution,**是对 user 认知 reflect**(reflection.rs + user_profile.rs + stability_detector.rs + tool_tracker.rs + transcript_ingest/)
9. **Tauri 2.x + Rust core in-process tokio task + per-launch hex token + CEF Chromium** — wiki 内首家 Tauri verified
10. **桌面 mascot + Google Meet 真实参与者** channel 业内独家
11. **QuickJS skills runtime 已 removed**(AGENTS.md:14)— execution surfaces rebuild 中,**标 [community-knowledge / project-in-flux],6 个月后 re-ingest**

**SkillForge 启示** R-OH-1~6:🟡 P2 Memory Tree 双写模式 + 🟡 P2 TokenJuice compaction-as-library + 🟡 P2 SandboxMode/ToolScope enum + 🟢 P3 Composio 桥接 + 🟢 P3 fork_context KV-cache + 🟢 P3 User Profile Reflection

**待反向加链**:[[concept/llm-wiki-pattern]] 加 openhuman 第 3 家 + [[concept/agent-memory]] 加 openhuman 第 7 条路线 + [[concept/multi-agent-isolation]] 加 fork_context 第 2 家 Fork verified — 留待下次更新或拆单独 task

**takeaway**:**OpenHuman 是 wiki 内首家 "Personal AI superintelligence" niche harness**(2026-05-13 newly launched 5 天上 trending),填补 wiki 在 _"个人 AI / 长期 memory / OAuth-heavy / voice-native"_ 类目空白。**3 大 differentiator**(Memory Tree 双写 / TokenJuice library 抽象 / Composio 桥接)都给 SkillForge 提供借鉴模板,虽然 server-side B2B 定位让 _"桌面 mascot + Google Meet"_ channel 路线不抄,但 **Memory + Compaction + Sub-agent enum-driven** 3 条工程模式值得借鉴。**Early Beta + QuickJS removal in-flux** 提醒 wiki 6 个月后 re-ingest 看演化。

---

## [2026-05-18] ingest | [[harness/langgraph]] — backlog §1 P0(本日第 3 个 P0)

**触发问题**:用户继 smolagents + openhuman 后选 C 继续 P0 — LangChain 出品的 stateful agent graph,wiki 内 [[eval/langchain]] 已有但 langgraph 作 harness 视角缺。Token 紧(~150K 已超软上限),压缩版 ingest。

**改动**:
- 新建 [[harness/langgraph]] — 13 维 + R-LG-1~4 + learning_takeaway 三段(紧凑写)
- 新建 [[source/langgraph]] — 简洁版
- 更新 [[backlog]] §1 P0 — langgraph done(2026-05-18)
- 更新 [[index]] Harness + Source 区

**模式**:B 模式 source-grounded(token 紧,只读 3 文件 + 多 ls)
**已读源码**:AGENTS.md(完整 dependency graph)+ state.py grep class boundaries + checkpoint base 行数 + ls 8 包结构 ~10K 增量
**版本**:1.2.0 @ 076e2a3(2026-05-12)

**关键发现**:
1. **业内 _"Pregel + 状态图"_ agent execution model 旗手** — 跟 ReAct loop / event loop / actor model 形成第 4 种 paradigm
2. **8 包 monorepo dependency graph 清晰**(AGENTS.md:34-53)— checkpoint base 是核心,4 上游;prebuilt+sdk-py 分层;sdk-js standalone
3. **StateGraph 4 generic type parameter** `Generic[StateT, ContextT, InputT, OutputT]` — 业内 type-safe agent state machine 最强
4. **StateGraph → CompiledStateGraph 编译/运行分离** — LLVM IR 思路,跟 Codex V8 isolate JS 编译同源
5. **Checkpoint 独立 package + 3 backends + conformance test suite** — 业内唯一 _"persistence first-class + 给 3rd-party 兜底测试"_,SkillForge 借鉴模板
6. **prebuilt 双层 SDK** — 底层 StateGraph 灵活 + 高层 chat_agent_executor/tool_node 快速上手
7. **callbacks.py LangChain ecosystem 复用** — 不重新发明 observability,工程克制

**SkillForge 启示** R-LG-1~4:
- 🟡 P2 R-LG-1 Checkpoint as first-class concern + 3 backends + conformance test 模式(2-3 周)
- 🟡 P2 R-LG-2 StateGraph 4 generic type parameter(Java/Kotlin 自然适配,1-2 周)
- 🟢 P3 R-LG-3 Pregel execution model POC(长期评估)
- 🟢 P3 R-LG-4 prebuilt 双层 SDK 模式

**takeaway**:LangGraph 是 wiki 协议层 + 4 个 multi-agent framework + 第 4 种 agent execution paradigm 的关键 anchor。**最高 ROI 是 R-LG-1 Checkpoint as first-class concern** — SkillForge persistence 当前是 PostgreSQL + pgvector,抽出 BaseCheckpointSaver SPI + conformance test suite 给跨产品 SDK 复用是直接借鉴。**24 小时内累计 7 个 P0 done**(sandbox-isolation / agent-protocols / mcp-spec-deep / google-a2a-spec / smolagents / openhuman / langgraph)。Token 极紧,建议下次 fresh session 继续。

---

## [2026-05-18] meta | Step 4 overview 升级 + E1 + E2 bundled

**触发问题**:上次 session 跑 `/wiki-lint` 出 4 E + 11 W + 123 R-XXX,Step 3 清完 W1 反向加链积压后,overview 时效已严重落后(2026-05-16 第 3 版基于 5 harness + 14 eval + 13 concept,实际 13 + 21 + 22 + 11 deep),且 E1(index 漏 2 行)+ E2(5 个 eval schema 漂移)还没修。本次把 overview 升级 + E1 + E2 三件套打包。

**改动**:
- 更新 [[overview]] 第 3 → **第 4 版**(2026-05-18):
  - §1 snapshot table 重写(13 + 16 eval + 5 framework + 22 + 11 + 35 = 102 主页)+ 调研对象覆盖度补 8 新 harness + eval 6 路线分类(加 framework)+ memory 7 路线(加 openhuman)+ RAG 6 路线新表
  - §2 判断 7 条 → **10 条**(判断 1 multi-agent 4 → 5 路线;新加判断 8 paradigm 4 种 / 判断 9 deep 页 §WHY + §失败模式 / 判断 10 协议层 5 栈)
  - §3 加 R-XXX P0/P1 15 条 lint-report 联动表
  - §5 Open Questions 清空 13 项已完成,加 aider/openhands/openai-agents-sdk 3 个 P0 候选
  - §6 元判断 3 条 → **6 条**(加 deep 页类型 / 24 小时 reshape 7 个 concept / 123 R-XXX 是 SkillForge 决策具象化)
- E1 修复:[[index]] §Concept 加 [[concept/attribution]] + [[concept/trace-based-eval]] 2 行(102/102 收录)
- E2 修复:[[schema]] §3 加 §3.2b `type: framework`(13 维 harness 复用 + 不强制 8 维 eval)+ §8 lint 规则同步 + §9 Changelog v2;5 个 eval 文件改 `type: eval` → `type: framework`(ag2 / cognee / llama_index / ms-graphrag / pathway)
- 更新 [[lint-report-2026-05-18]] 标 E1/E2/Step 4 done + 健康度表(索引完整性 🟡 98% → ✅ 100%;新加 schema 一致性 + overview 时效性 两行)

**模式**:meta(不算新 ingest),纯 wiki 维护性 commit
**已读源码**:0(纯文档操作)
**版本**:N/A

**关键发现**:
1. **schema v2 决策(选 B)解决"eval/ 内 5 个 framework 误标"语义错配** — 避免反向链翻新,但精确区分 eval-framework vs framework(memory/RAG/multi-agent/streaming)
2. **overview §2 判断 9 deep 页 §WHY + §失败模式** — 把 deep 页"为什么贵"的元判断显化,SkillForge 内部文档可学这个结构
3. **§3 R-XXX 联动表** — 把 8 条 overview 综合决策跟 15 条 page-level R-XXX 区分清楚,前者是 thesis,后者是 actionable
4. **§5 Open Questions 清单清空 13 项** — 24 小时密集 ingest 后,backlog §1 P0 几乎全消化,只剩 aider / openhands / openai-agents-sdk 3 个未收

**takeaway**:**24 小时密集 ship 已经让 overview 严重过时,Step 4 升级是把"密集 ship"沉淀成"全局判断"的关键动作**。下次 fresh session 起手就拿到 13 + 16 + 5 + 22 + 11 + 35 = 102 主页 + 10 条核心判断 + 15 条 R-XXX P0/P1,直接进入 Step 5(下一个 P0 harness)无需重读 wiki。**E3/E4 safety verify 留下次清(30-60 分钟,grep 源码)**。
