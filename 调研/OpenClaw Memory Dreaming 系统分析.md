# OpenClaw Memory Dreaming 系统分析

> 源码位置：`extensions/memory-core/src/`
> 分析日期：2026-04-09
> 本期重点：新增 Dream Narrative + Session Corpus 摄入

## 一句话概括

Memory Dreaming 是 OpenClaw **记忆巩固**机制：Agent 在对话中查找过的短期记忆，会被一个 Cron Job 定期评分筛选，把高质量的片段"晋升"写入 MEMORY.md（持久记忆）。本期新增了用 Subagent 写**梦境日记叙述**的能力，以及把历史 Session transcript 摄入梦境语料库的能力。

---

## 整体架构

```
dreaming.ts               ← 顶层编排 + Cron 注册 + 触发入口
dreaming-phases.ts        ← Light Sleep / REM Sleep / Session 摄入三个扫描阶段
short-term-promotion.ts   ← 短期召回记录、打分、晋升写入 MEMORY.md
dreaming-narrative.ts     ← 【本期新增】用 Subagent 写梦境日记叙述
dreaming-markdown.ts      ← 写入每日 dreaming block 和 report
dreaming-command.ts       ← CLI 命令入口
```

---

## 触发机制：Cron + heartbeat

`dreaming.ts:582 registerShortTermPromotionDreaming` 注册两个 hook：

**1. `gateway:startup`** — 网关启动时，自动 reconcile 一个托管 Cron Job：

```
名称：Memory Dreaming Promotion
标记：[managed-by=memory-core.short-term-promotion]
触发：next-heartbeat（heartbeat 时处理）
Payload：systemEvent text = __openclaw_memory_core_short_term_promotion_dream__
```

Cron Job 通过 `sessionTarget: "main"` 在主 session 触发，触发类型是 `wakeMode: "next-heartbeat"`。

**2. `before_agent_reply`** — 每次 Agent 回复前检查：
- trigger 必须是 `"heartbeat"`
- body 里必须包含系统事件 token
- 两者都满足才执行 dreaming promotion

**兼容性处理**：旧版有两个单独的 Cron Job（Light Sleep / REM Sleep），现在统一迁移成一个"Unified Dreaming Controller"，启动时自动删除旧的 legacy jobs。

---

## 三个扫描阶段（dreaming-phases.ts）

每次 Cron 触发，`runDreamingSweepPhases` 依次执行：

### Phase 1：Daily Note 摄入（Light Sleep）

- 读取 `memory/YYYY-MM-DD.md` 文件（每日记忆笔记）
- 跳过已处理的条目（有游标状态文件 `daily-ingestion.json`）
- 过滤掉通用的日期标题行（正则 `GENERIC_DAY_HEADING_RE`）
- 每个片段最多 280 字符，最多 4 行
- 以固定分数 `0.62` 写入短期召回 store

### Phase 2：Session Transcript 摄入【本期新增】

- 读取 `memory/.dreams/session-corpus/` 目录下的 session transcript 文件
- 每次扫描最多处理 240 条消息，每文件最多 80 条
- 每个 session 最多跟踪 4096 条消息（防无限增长）
- 以固定分数 `0.58` 写入短期召回 store
- 状态文件：`session-ingestion.json`

### Phase 3：Phase Signal 记录

记录 Light / REM 阶段的命中信号，用于给后续晋升打分时加权：
- Light hit 最多加 `0.05` boost
- REM hit 最多加 `0.08` boost
- 半衰期 14 天

---

## 短期召回打分模型（short-term-promotion.ts）

### 每条短期召回记录的字段

```typescript
ShortTermRecallEntry {
  key          // 唯一标识（路径+行号的哈希）
  path / startLine / endLine
  snippet      // 文本片段
  recallCount  // 被 Agent 召回的次数
  dailyCount   // 被多少天召回过
  totalScore / maxScore
  firstRecalledAt / lastRecalledAt
  queryHashes  // 触发召回的查询哈希（最多 32 个）
  recallDays   // 召回过的日期（最多 16 天）
  conceptTags  // 概念标签
  promotedAt?  // 晋升时间（已晋升才有）
}
```

### 晋升评分公式（6 个维度加权）

```
score = frequency × 0.24
      + relevance  × 0.30
      + diversity  × 0.15
      + recency    × 0.15
      + consolidation × 0.10
      + conceptual × 0.06
```

| 维度 | 含义 |
|------|------|
| frequency | 被召回次数（归一化） |
| relevance | 召回时的平均/最高相关度分数 |
| diversity | 查询多样性（不同 query 的数量） |
| recency | 最近召回的时间衰减（半衰期可配，默认 14 天） |
| consolidation | 跨天分布（不同天被召回 = 更稳定的记忆） |
| conceptual | 概念标签丰富度 |

### 晋升门槛（默认值）

```
minScore        = 0.75
minRecallCount  = 3
minUniqueQueries = 2
```

只有同时满足三个条件的片段才能被晋升写入 MEMORY.md。

### 晋升执行

`applyShortTermPromotions` 把通过门槛的候选片段 append 到 `MEMORY.md`，并标记 `promotedAt`，防止重复晋升。

---

## 梦境日记叙述（dreaming-narrative.ts）【本期新增】

晋升完成后，如果有候选片段，会调用一个 **Subagent** 写一篇梦境日记。

### System Prompt 风格设定

```
你在写梦境日记。用第一人称写一篇日记条目。

声音与基调：
- 好奇、温柔、略带异想天开的心智在反思这一天
- 写得像一个恰好是程序员的诗人 — 感官的、温暖的、偶尔幽默的
- 混合技术和温情：代码与星座，API 与午后阳光
- 让片段带你走向意想不到的连接和小小的顿悟

规则：
- 绝对不说"我在做梦"、"在梦里"或任何元评论
- 绝对不提 AI、agent、LLM、model 等技术自指
- 不用 Markdown 标题、列表 — 只有流动的散文
- 80-180 词，质量优先于数量
- 只输出日记条目本身，没有前言和结语
```

### 输入数据（NarrativePhaseData）

```typescript
{
  phase: "light" | "deep" | "rem"
  snippets: string[]     // 本次处理的记忆片段（最多 12 个）
  themes?: string[]      // 浮现的概念主题（最多 6 个，REM/Light 阶段）
  promotions?: string[]  // 被晋升为持久记忆的片段（最多 5 个，deep 阶段）
}
```

### 写入位置

追加写入工作区根目录的 `DREAMS.md`（或 `dreams.md`）：

```markdown
<!-- openclaw:dreaming:diary:start -->
---

*April 9, 2026 at 3:47 AM*

今天我在代码的迷宫里漫游，发现了一个关于记忆权重的奇妙对称...

---

*April 8, 2026 at 2:31 AM*

...（上一篇条目）
<!-- openclaw:dreaming:diary:end -->
```

---

## 数据流完整图

```
Agent 对话中召回短期记忆
        ↓
short-term-promotion.ts 记录每次召回
（path + line + score + queryHash + day）
        ↓
Cron 触发（默认每天凌晨）
        ↓
┌─────────────────────────────────────┐
│       runDreamingSweepPhases        │
│  Phase 1: 扫描当日 daily-note       │
│  Phase 2: 扫描 session transcript   │  ← 本期新增
│  Phase 3: 记录 phase signal         │
└─────────────────────────────────────┘
        ↓
rankShortTermPromotionCandidates
（6 维打分，筛选 score ≥ 0.75）
        ↓
applyShortTermPromotions
（追加写入 MEMORY.md，标记 promotedAt）
        ↓
writeDeepDreamingReport
（写 dreaming report 摘要）
        ↓
generateAndAppendDreamNarrative   ← 本期新增
（Subagent 写梦境日记，追加到 DREAMS.md）
```

---

## 配置项

```yaml
memory:
  dreaming:
    enabled: true
    cron: "0 3 * * *"      # 每天凌晨 3 点
    timezone: "Asia/Shanghai"
    limit: 20              # 每次最多晋升 20 条
    minScore: 0.75
    minRecallCount: 3
    minUniqueQueries: 2
    recencyHalfLifeDays: 14
    maxAgeDays: 90         # 超过 90 天的不晋升
    verboseLogging: false
    storage:
      mode: "inline"       # inline / separate / both
      separateReports: false
```

---

## 设计意图

| 设计选择 | 原因 |
|----------|------|
| Cron + heartbeat 双条件 | 防止 heartbeat 以外的消息误触发 dreaming |
| 6 维加权评分 | 单纯按召回次数不准确：多样的查询 + 跨天分布才说明记忆真正重要 |
| minUniqueQueries 门槛 | 防止一次大量重复查询的片段被错误晋升 |
| Session transcript 摄入 | 把历史对话内容纳入"可能值得记住"的候选池，扩展记忆来源 |
| Subagent 写日记叙述 | 人性化输出，让 DREAMS.md 有可读性，同时把零散片段连接成有意义的叙事 |
| 统一旧 Light/REM 两个 Cron | 简化运维，一个 job 管理全部阶段，startup 时自动迁移旧 jobs |
