---
type: source
name: openclaw
verification: source-grounded
origin:
  - /Users/youren/myspace/openclaw (OpenClaw TypeScript monorepo)
  - /Users/youren/myspace/research-docs/research/OpenClaw源码研究/ (14 篇研究笔记 + diagrams)
  - /Users/youren/myspace/research-docs/research/OpenClaw {Memory Dreaming,Memory Wiki,QA Character Eval,记忆系统} 系统分析.md (4 篇专题)
ingested_at: 2026-05-16
ingest_mode: B (source-grounded)
contributed_to:
  - "[[harness/openclaw]]"
  - "[[concept/self-evolution]]"
  - "[[concept/llm-wiki-pattern]]"
  - "[[concept/agent-eval]]"
  - "[[concept/multi-agent-isolation]]"

sources_read:
  docs:
    - README.md (100 行,24+ channels + OAuth sponsors + native apps)
    - AGENTS.md (199 行,Telegraph style + 架构哲学 + prompt-cache rule + monorepo 命令)
    - CLAUDE.md (199 行,跟 AGENTS.md 同步)
  typescript:
    - "extensions/memory-core/src/dreaming.ts: grep 行号 30-650(Cron names / config / register)"
    - "extensions/memory-core/src/ ls (20+ files: dreaming-* 8 个 + concept-vocabulary + rem-evidence + memory-events)"
    - "extensions/memory-wiki/src/ ls (25 files: vault/ingest/bridge/compile/claim-health/prompt-section/tool/lint/apply/query/obsidian/etc)"
    - "extensions/ ls 第一屏 30+ extensions"
  directories_listed:
    - "openclaw root (40+ top-level files/dirs 含 dream-diary-preview-v2/v3.html)"
    - "apps/ (android / ios / macos / shared)"
    - "packages/ (5 SDK packages: plugin-sdk / plugin-package-contract / memory-host-sdk / clawdbot / moltbot)"
    - "extensions/memory-core/src/"
    - "extensions/memory-wiki/src/"
    - "extensions/ (含 horizontal cores: memory-core / image-generation-core / media-understanding-core / speech-core / video-generation-core)"
  notes_cross_referenced:
    - "[[../../OpenClaw源码研究/index]] (14 子文档)"
    - "[[../../OpenClaw源码研究/04-Agent引擎深度解析]] (pi-sdk + 双层循环)"
    - "[[../../OpenClaw Memory Dreaming 系统分析]] (6 维评分公式 + 3 phases + 梦境日记)"
    - "[[../../OpenClaw Memory Wiki 系统分析]] (Karpathy 模式 production 实现)"
    - "[[../../OpenClaw QA Character Eval 系统分析]] (双阶段 production framework)"
---

# Source — OpenClaw

## 核心论点

OpenClaw 是 personal AI assistant(TypeScript pnpm monorepo,OpenAI 等 sponsor),**Hermes 的前身**(Hermes README 明确写 `hermes claw migrate` 兼容)。**3 个关键 production 系统是其他 harness 都没有的**:

1. **Memory Dreaming**(`extensions/memory-core/src/dreaming-*.ts`,8 文件)— **Cron + heartbeat 双轨触发 + 6 维评分晋升 + 梦境日记 subagent + 反元评论 prompt** — Hermes 简化为 `commit_memory_session`,Claude Code 借鉴 "Dream" 命名但简化 4-phase prompt
2. **Memory Wiki**(`extensions/memory-wiki/src/`,25 文件)— **Karpathy llm-wiki.md 模式的 production 实现**,claim(`confidence/status/evidence/updatedAt`)+ freshness(fresh/aging/stale)+ contradiction cluster + `compile.ts` 编译 `agent-digest.json` 自动注入 system prompt + 5 Agent tools + Obsidian 兼容 + managed blocks(`<!-- openclaw:human:start/end -->`)— **跟 SkillForge 决策直接相关**(我们手做的 agent-harness-wiki 同源,但 OpenClaw 已 production)
3. **QA Character Eval**(`extensions/qa-lab/src/`)— **production agent eval framework** — 双阶段(候选并行 + Judge 排名)+ 真实 Gateway 子进程(不 mock)+ Lab Server (MCP) + YAML DSL 编译 JS AsyncFunction + Bus 消息总线 — **3 harness 里 eval_loop 最完整**(Hermes 仅 RL trajectories,Claude Code 仅 internal ML classifier)

**架构核心**:基于 **pi-sdk 嵌入式集成**(Mario Zechner / badlogic 开源,pi-ai + pi-agent-core + pi-coding-agent 三层),**不自己重写 agent loop**。

**双层 agent loop 显式拆分**:外层 `runEmbeddedPiAgent`(`src/agents/pi-embedded-runner/run.ts`)32-160 次重试容错决策 + 内层 `runEmbeddedAttempt`(`run/attempt.ts`)tool-use 失败抛 — **比 Hermes `run_conversation()` 单函数承担 9 阶段更 elegant 的职责分离**。

## B 模式发现的事实(全部 verified)

| # | 事实 | 行号 | 备注 |
|---|---|---|---|
| 1 | Memory Dreaming Cron name + event text | `extensions/memory-core/src/dreaming.ts:30-32` | `MANAGED_DREAMING_CRON_NAME = "Memory Dreaming Promotion"` |
| 2 | Legacy Light/REM 已统一 | `dreaming.ts:33-38` | legacy tag 保留为兼容 |
| 3 | Cron reconcile 间隔 | `dreaming.ts:39` | `RUNTIME_CRON_RECONCILE_INTERVAL_MS = 60_000` (1 min) |
| 4 | 6 维评分公式 | 笔记 `Memory Dreaming 系统分析.md:103-108` | freq 0.24 + rel 0.30 + div 0.15 + rec 0.15 + cons 0.10 + concep 0.06 |
| 5 | 晋升门槛 | 笔记 + `dreaming.ts:102-104` | minScore=0.75 / minRecallCount=3 / minUniqueQueries=2 |
| 6 | Phase 1 Daily Note 摄入 | 笔记 + `dreaming-phases.ts` | 固定分 0.62,每片段 ≤280 chars 4 lines |
| 7 | Phase 2 Session Transcript 摄入 | 笔记 + `dreaming-phases.ts` | 固定分 0.58,每扫 ≤240 msg,每文件 ≤80 msg,每 session ≤4096 msg |
| 8 | Phase 3 Phase Signal | 笔记 | Light hit +0.05,REM hit +0.08,半衰期 14 天 |
| 9 | 梦境日记反元评论 prompt | 笔记 `Memory Dreaming.md:140-156` | "不说在做梦/AI",80-180 词,程序员诗人 |
| 10 | DREAMS.md 格式 | 笔记 `Memory Dreaming.md:171-187` | managed block `<!-- openclaw:dreaming:diary:start/end -->` |
| 11 | Memory Wiki vault 结构 | 笔记 `Memory Wiki.md:31-50` | entities/concepts/syntheses/sources/reports + cache/agent-digest.json + claims.jsonl |
| 12 | claim 状态 enum | 笔记 `Memory Wiki.md:84` | supported / contested / contradicted / refuted / superseded |
| 13 | claim freshness 阈值 | 笔记 `Memory Wiki.md:96-99` | fresh<30d / aging 30-90d / stale>90d / unknown(no updatedAt) |
| 14 | 5 Agent tools | 笔记 `Memory Wiki.md:138-144` | wiki_status / wiki_search / wiki_get / wiki_apply / wiki_lint |
| 15 | 页面排名公式 | 笔记 `Memory Wiki.md:115` | 矛盾×6 + 未解问题×4 + min(claim,6)×2 + min(top,3) |
| 16 | QA Character Eval 入口 | 笔记 `QA Character Eval.md:30` | `character-eval.ts:393 runQaCharacterEval` |
| 17 | QA 双阶段 | 笔记 `QA Character Eval.md:32-87` | candidates 并行 → Judge 横向打分 |
| 18 | YAML 双 fence DSL | 笔记 `QA Character Eval.md:55-79` | qa-scenario + qa-flow |
| 19 | Flow DSL 编译 JS | 笔记 `QA Character Eval.md:90-94` | `new AsyncFunction(...contextNames, ...)` |
| 20 | pi-sdk 三层 | 笔记 `04-Agent引擎.md:15-19` | pi-ai (LLM) + pi-agent-core (ReAct) + pi-coding-agent (Session+工具+Retry+Compaction) |
| 21 | 双层循环显式拆分 | 笔记 `04-Agent引擎.md:38-105` | runEmbeddedPiAgent 外层 + runEmbeddedAttempt 内层 |
| 22 | 外层重试上限 | 笔记 `04-Agent引擎.md:43` | 32-160 次(由 auth profile 数动态决定) |
| 23 | 外层 5 类决策 | 笔记 `04-Agent引擎.md:50-67` | context overflow / auth / rate limit / thinking level / retry exhausted |
| 24 | thinking level 降级 | 笔记 `04-Agent引擎.md:62` | high → low → off |
| 25 | extension-agnostic core 哲学 | `AGENTS.md:41` | "Core must stay extension-agnostic" |
| 26 | Prompt-cache rule | `AGENTS.md:54` | "deterministic ordering for maps/sets/registries...Preserve old transcript bytes" |
| 27 | 24+ channels + 4 native apps | README.md:26 + `apps/` ls | WhatsApp/Telegram/.../WebChat + android/ios/macos/shared |
| 28 | extensions 30+ | `extensions/` ls 第一屏 | provider/channel/feature 全 extensions 化 |
| 29 | 5 horizontal core extensions | `extensions/{memory,image-generation,media-understanding,speech,video-generation}-core` | 跟 channels/providers 平级 |
| 30 | Plugin SDK 公开 npm | `packages/plugin-sdk` | extension-agnostic 公开契约 |

## 跟 Hermes / Claude Code 关键对比

| 维度 | OpenClaw | Hermes(基于 OpenClaw 演进) | Claude Code(借鉴命名) |
|---|---|---|---|
| **Agent loop 架构** | pi-sdk 嵌入式 + 双层循环显式拆 | 自写 sync `run_conversation()` 单函数 9 阶段 | async generator `query()` + `queryLoop()` |
| **Self-evolution** | **Memory Dreaming 完整**(6 维评分 + Cron + 梦境日记) | **简化**为 `commit_memory_session` rotation 前 trigger | autoDream 3-gate + 4-phase **借鉴 "Dream" 命名,简化** |
| **Memory Wiki** | **production 完整实现**(claim/freshness/contradiction/compile/digest auto-inject)| 仅 MEMORY.md/USER.md 简单 markdown | autoMemory + SessionMemory + extractMemories(类似但分散) |
| **Eval framework** | **production QA Character Eval**(双阶段 + Gateway 子进程 + YAML DSL) | RL/Atropos trajectories(research) | internal classifier(ML) |
| **Channels** | **24+ + 4 native apps** | 18+(SPI 抽象) | CLI/TUI + Bridge(无 messaging) |
| **Extensions** | **全 extensions 化**(30+,extension-agnostic core) | tools/+plugins/+gateway/platforms/ 三层混合 | services/+tools/+features() Bun gates |
| **Hooks** | Internal + Plugin hooks 双层(`gateway:startup` / `before_agent_reply` 等) | 16 plugin hooks | 27 zod-validated HOOK_EVENTS |
| **Native apps** | android/ios/macos/shared | 无 | 无(CLI 工具) |
| **Plugin contract** | `packages/plugin-sdk` npm 公开 | `hermes_cli/plugins.py` 内部 | feature() gate + GrowthBook |

## 演进链(self-evolution 概念修正)

**这次 ingest 最重要的发现** — 之前 [[concept/self-evolution]] 把 Hermes 当作 self-evolution 范式起点,实际:

```
OpenClaw Memory Dreaming (2026-03 production)
       ↓ fork-继承
Hermes commit_memory_session (简化:无 Cron / 无评分 / 无日记)
       ↓ 共时(2026-03 v2.1.88 leak)
Claude Code autoDream (借鉴 "Dream" 命名,简化:无评分晋升 / 无日记 subagent)
```

OpenClaw 留下 `LEGACY_LIGHT_SLEEP_CRON_TAG` / `LEGACY_REM_SLEEP_CRON_TAG` 常量证明已迭代过(legacy 是 Light/REM 分开,现在统一)。

## 与现有 wiki 的关系

- **印证**:Hermes AGENTS.md 提到 `hermes claw migrate` 命令 — OpenClaw 是 Hermes 前身已 verified
- **重大修正**:[[concept/self-evolution]] 之前的"Hermes 是 self-evolution 范式起点"叙事需改 — OpenClaw 才是源头
- **重大新增**:**OpenClaw Memory Wiki 是 Karpathy llm-wiki 模式的 production 实现**,跟我们手做的 agent-harness-wiki 同源 — 直接催生新 concept [[concept/llm-wiki-pattern]]
- **新增**:OpenClaw QA Character Eval 是 production agent eval framework — 催生新 concept [[concept/agent-eval]]
