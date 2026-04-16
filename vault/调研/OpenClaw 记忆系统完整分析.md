# OpenClaw 记忆系统完整分析

> 源码位置：`extensions/memory-core/`
> 分析日期：2026-04-09

## 全局架构概览

OpenClaw 的记忆系统分为三层：

```
┌─────────────────────────────────────────────────────────────┐
│                       Agent 对话层                           │
│  memory_search / memory_get 工具 + 系统 Prompt 注入          │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                      索引 & 检索层                            │
│  SQLite（chunks + chunks_vec + chunks_fts）                  │
│  混合召回：向量搜索 + BM25 关键词搜索                          │
│  后处理：时间衰减 + MMR 多样性重排                             │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│                      记忆巩固层（Dreaming）                   │
│  短期召回追踪 → Cron 评分 → 晋升写入 MEMORY.md               │
│  梦境日记叙述（DREAMS.md）                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 第一层：记忆文件结构

### 文件类型和语义

| 文件路径 | 类型 | 语义 |
|---------|------|------|
| `MEMORY.md` | 持久记忆 | 长期核心知识，不会时间衰减 |
| `memory/*.md` | 话题文件 | 按主题分类的持久知识，不衰减 |
| `memory/YYYY-MM-DD.md` | 日记记忆 | 每日短期记录，按日期衰减 |
| Session transcripts | 对话语料 | 历史对话，可被索引和检索 |

**关键设计**：`MEMORY.md` 和 `memory/*.md` 是 "evergreen"（常青）文件，时间衰减不对它们生效。只有带日期的文件（`memory/2026-04-09.md`）会随时间衰减。

---

## 第二层：索引与检索

### 2.1 SQLite 数据库结构

使用 Node.js 内置 `DatabaseSync`（原生 SQLite），每个 Agent workspace 一个 db 文件。

```sql
-- 文本 chunk 表：存储分块文本和向量
chunks (
  id TEXT PRIMARY KEY,
  path TEXT,          -- 相对文件路径（如 MEMORY.md）
  start_line INT,
  end_line INT,
  text TEXT,          -- chunk 原文
  embedding BLOB,     -- Float32Array 序列化的向量
  model TEXT,         -- 生成 embedding 时用的模型
  source TEXT         -- 来源标识（如 "memory"）
)

-- 向量索引表（sqlite-vec 扩展）
chunks_vec (
  id TEXT,
  embedding FLOAT[N]  -- N 维向量，用于 vec_distance_cosine()
)

-- 全文搜索表
chunks_fts (          -- SQLite FTS5 虚表
  id, path, source, start_line, end_line, text, model
  -- tokenizer: unicode61（英文）或 trigram（中日韩）
)

-- Embedding 缓存表
embedding_cache (     -- 避免重复调用 embedding API
  ...
)
```

**PRAGMA 设置**：每次打开连接都设 `busy_timeout = 5000`，多进程并发时不立即报 `SQLITE_BUSY`，而是等待 5 秒。

### 2.2 Embedding 提供商

支持自动选择和手动配置：

| 提供商 | 说明 |
|--------|------|
| OpenAI | `text-embedding-3-small` 等 |
| Voyage | Anthropic 推荐的高质量向量 |
| Gemini | Google embedding API |
| Mistral | Mistral embedding API |
| Ollama | 本地运行的 embedding 模型 |
| Local | 本地模型文件（gguf） |

自动选择按 `autoSelectPriority` 排序，优先 local/Ollama（离线可用），无法加载再降级到远程。

### 2.3 混合召回流程（hybrid.ts）

每次 `memory_search` 调用：

```
用户 query
    ├─ 1. Embedding 查询向量
    │      ↓
    │  searchVector()
    │  SELECT ... vec_distance_cosine(v.embedding, ?) ... ORDER BY dist ASC
    │  → 返回 [{ id, path, startLine, endLine, snippet, vectorScore }]
    │
    └─ 2. BM25 关键词查询
           ↓
       searchKeyword()
       SELECT ... FROM chunks_fts WHERE chunks_fts MATCH ? ORDER BY bm25() ASC
       → 返回 [{ id, path, startLine, endLine, snippet, textScore }]

              ↓
       mergeHybridResults()
       score = vectorWeight × vectorScore + textWeight × textScore
       （同一 id 的结果合并，只有一侧命中则另一侧得分为 0）

              ↓
       applyTemporalDecayToHybridResults()    （可选，默认关闭）
       score × e^(-λ × ageDays)

              ↓
       applyMMRToHybridResults()              （可选，默认关闭）
       多样性重排

              ↓
       最终返回排序后的召回结果
```

### 2.4 BM25 关键词搜索细节

- 英文/通用：FTS tokenizer `unicode61`，查询格式 `"token1" AND "token2"`（精确词组匹配）
- 中日韩：FTS tokenizer `trigram`，按 trigram 切词
  - 短 CJK（< 3 字符）降级为 `LIKE '%term%'` 子串匹配
  - 长 CJK token 进 FTS MATCH

BM25 排名值经 `bm25RankToScore(rank)` 转换为 `[0,1]` 分数：
```
rank < 0：relevance / (1 + relevance)   ← SQLite BM25 负值表示相关
rank ≥ 0：1 / (1 + rank)               ← 退化情况
```

### 2.5 时间衰减（temporal-decay.ts）

默认关闭，开启后对带日期的文件按指数衰减：

```
score_decayed = score × e^(-λ × ageDays)
λ = ln(2) / halfLifeDays
```

- 默认半衰期 30 天（30 天后分数衰减为原来一半）
- `MEMORY.md` 和 `memory/*.md`（非日期文件）跳过衰减
- 对于非日期文件，取文件的 `mtime` 作为时间戳

### 2.6 MMR 多样性重排（mmr.ts）

基于 Carbonell & Goldstein (1998) 的 Maximal Marginal Relevance 算法，默认关闭。

**算法**：贪心迭代选择，每轮选 MMR 分数最高的：
```
MMR(i) = λ × relevance(i) - (1-λ) × max_sim(i, already_selected)
```

相似度计算（Jaccard）：
- 英文：按 ASCII 字母数字 token 分词
- 中日韩：CJK 单字 unigram + 相邻字符 bigram（严格按原文位置，不跨词制造虚假 bigram）

**参数**：
- `lambda = 0.7`（默认）：70% 权重给相关性，30% 给多样性
- `lambda = 1.0`：纯按相关性排序（等同不用 MMR）
- `lambda = 0.0`：纯按多样性排序

---

## 第三层：记忆巩固（Dreaming）

### 3.1 短期召回追踪

每次 `memory_search` 返回结果后，`tools.ts` 异步（fire-and-forget）调用：

```typescript
void recordShortTermRecalls({
  workspaceDir,
  query,
  results: surfacedResults,  // 实际展示给用户的结果
  timezone,
}).catch(() => {})            // 永远不阻塞召回本身
```

追踪数据存在 `memory/.dreams/short-term-recall.json`：

```typescript
ShortTermRecallEntry {
  key           // path:startLine:endLine 的哈希
  path / startLine / endLine
  snippet       // 片段文本
  recallCount   // 总召回次数
  dailyCount    // 被召回的天数
  totalScore / maxScore     // 累积/最大相关度
  firstRecalledAt / lastRecalledAt
  queryHashes   // 触发召回的 query 哈希（最多 32 个）
  recallDays    // 召回发生的日期列表（最多 16 天）
  conceptTags   // 从 snippet 提取的概念标签
  promotedAt?   // 已晋升则有此字段
}
```

### 3.2 晋升评分模型（6 维加权）

```
score = frequency    × 0.24   // 召回次数（归一化）
      + relevance    × 0.30   // 平均/最高相关度
      + diversity    × 0.15   // 不同查询的数量（query 多样性）
      + recency      × 0.15   // 时间衰减（半衰期 14 天）
      + consolidation× 0.10   // 跨天分布（多天被召回 = 更稳定）
      + conceptual   × 0.06   // 概念标签丰富度
```

**晋升门槛（默认）**：
```
minScore         ≥ 0.75
minRecallCount   ≥ 3
minUniqueQueries ≥ 2
```

三条件同时满足才能晋升，防止单次大量重复查询的片段被错误提升。

### 3.3 Cron 触发流程

```
gateway:startup
    ↓
reconcileShortTermDreamingCronJob()
    ↓ 注册/更新 Cron Job（统一 Dreaming Controller）
    ↓ 自动清理旧版 Light Sleep / REM Sleep 两个 legacy jobs

每天凌晨（默认 cron 表达式）
    ↓ Cron 触发 → heartbeat → before_agent_reply hook
    ↓ 检查 systemEvent token 匹配

runDreamingSweepPhases()
    ├─ Phase 1: Daily Note 摄入（daily-ingestion.json 游标）
    │   读 memory/YYYY-MM-DD.md，score=0.62，写入 short-term-recall
    ├─ Phase 2: Session Transcript 摄入（session-ingestion.json 游标）
    │   读 memory/.dreams/session-corpus/*.md，score=0.58
    └─ Phase 3: Phase Signal 记录（phase-signals.json）
        Light hit: 最多 +0.05 boost
        REM hit:   最多 +0.08 boost

rankShortTermPromotionCandidates()
    ↓ 6 维评分，过滤 score < 0.75

applyShortTermPromotions()
    ↓ 追加到 MEMORY.md，标记 promotedAt

writeDeepDreamingReport()
    ↓ 写摘要 report

generateAndAppendDreamNarrative()（如有候选片段）
    ↓ Subagent 写梦境日记 → 追加到 DREAMS.md
```

---

## 工具层：Agent 侧接口

### memory_search

```typescript
// 参数
{
  query: string        // 自然语言查询
  maxResults?: number  // 默认系统控制
  corpus?: "memory" | "wiki" | "all"  // 语料库范围
}

// 内部执行
1. Embedding query → 向量搜索
2. 关键词搜索（BM25）
3. 合并 → 时间衰减 → MMR → 截断
4. 异步记录召回（短期促进追踪）
5. 返回 [{ path, startLine, endLine, snippet, score, source }]
```

### memory_get

```typescript
// 按路径读特定文件的特定行范围
{
  lookup: string      // 文件路径或 id
  fromLine?: number
  lineCount?: number
  corpus?: "memory" | "wiki" | "all"
}
```

### 系统 Prompt 注入（prompt-section.ts）

每次 Agent 对话前自动注入：

```
## Memory Recall
Before answering anything about prior work, decisions, dates, people, preferences,
or todos: run memory_search on MEMORY.md + memory/*.md + indexed session transcripts;
then use memory_get to pull only the needed lines.

Citations: include Source: <path#line> when it helps the user verify memory snippets.
```

---

## 完整数据流

```
用户问："上周你说的那个 API 设计是什么？"
         ↓
  memory_search("API 设计 上周")
         ↓
  ┌── 向量搜索 ──────────────────────────────────────┐
  │  query → embedding → cosine sim → top-K          │
  └── 关键词搜索 ────────────────────────────────────┘
  │  "API" AND "设计" AND "上周" → BM25 → top-K      │
         ↓
  合并（weighted sum）→ 时间衰减 → MMR 去重 → 截断
         ↓
  返回结果给 Agent
         ↓
  异步记录召回 → short-term-recall.json
         ↓（每天凌晨 Cron）
  评分候选 → 晋升到 MEMORY.md → 写梦境日记到 DREAMS.md
```

---

## 设计意图总结

| 设计选择 | 原因 |
|----------|------|
| SQLite 而非向量数据库 | 零依赖，嵌入式运行，本地优先 |
| 混合召回（向量 + BM25） | 纯向量对精确词组弱；纯关键词对语义不理解；混合两者互补 |
| Evergreen vs 日期文件分类 | MEMORY.md 是核心知识不该衰减；日记是时效信息应该衰减 |
| 时间衰减默认关闭 | 保守：误衰减比没衰减危险，用户可按需开启 |
| MMR 默认关闭 | 有计算开销，仅在结果重复度高时有价值 |
| 召回追踪 fire-and-forget | 不能让追踪延迟阻塞 Agent 的实际回答 |
| 6 维评分而非仅次数 | 单次集中查询 ≠ 真正重要；多样查询 + 跨天分布才是稳定记忆的信号 |
| Dreaming 用 Cron + heartbeat 双条件 | 防止普通消息误触发；heartbeat 确保在主 session 里有序执行 |
| 梦境日记叙述 | 把零散片段连接成叙事，提升 DREAMS.md 可读性，同时也是对记忆的一次再组织 |
