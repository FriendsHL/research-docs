---
name: claude-code-insights
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: claude-code (insights subsystem)
version: v2.1.88
commit: (npm sourcemap leak 2026-03-31)
license: closed-source (reverse engineered for research)
source_file: src/commands/insights.ts (3200 行)
last_ingested_at: 2026-05-17
contributed_to:
  - "[[harness/claude-code]]"
  - "[[deep/claude-code-insights]]"
  - "[[concept/attribution]]"
---

# Source — Claude Code `/insights` Command(v2.1.88 source-grounded)

> `src/commands/insights.ts` 3200 行单文件实现 — Claude Code 内置 slash command,自动分析最近 30 天 sessions,**Opus model 跑 LLM analysis** 提取 5+ 维 typed enum facets + 8 个并行 INSIGHT_SECTIONS LLM analysis + interactive HTML report。**业内最完整 Layer D + Layer E _"用户手动 review"_ 路线实现**(对照 SkillForge V3+V80 _"自动 canary + verify"_ 路线)。

---

## 1. 文件位置 + 行数

```
src/commands/insights.ts             3200 行(主文件)
src/utils/sessionStorage.ts          含 getProjectsDir / getSessionFilesWithMtime / loadAllLogsFromSessionFile
src/services/SessionMemory/prompts.ts 相关(SessionMemory 不是 insights 但提示词模式相似)
src/constants/outputStyles.ts        外围引用
src/commands.ts                      slash command 注册
```

**单文件 3200 行的设计选择**:整个 `/insights` 功能放一个文件,**reader 上下文友好**(不用跳 20 个文件追逻辑)— 跟 [[deep/claude-code-autodream-3-gate]] `services/autoDream/` 4 小文件的设计思路对比。

---

## 2. 3 层数据模型 [verified `insights.ts:228-326`]

### Layer 1 — `SessionMeta`(单 session 统计,deterministic)[code: insights.ts:228-258]

```typescript
type SessionMeta = {
  session_id: string
  project_path: string
  start_time: string
  duration_minutes: number
  user_message_count: number
  assistant_message_count: number
  tool_counts: Record<string, number>          // 每个 tool 用了几次
  languages: Record<string, number>             // 文件扩展名 → 语言 → 次数
  git_commits: number
  git_pushes: number
  input_tokens: number
  output_tokens: number
  first_prompt: string
  summary?: string
  
  // New stats (recent additions)
  user_interruptions: number                     // ⭐ 用户 Esc 取消次数
  user_response_times: number[]                  // ⭐ 用户响应时延
  tool_errors: number
  tool_error_categories: Record<string, number>
  uses_task_agent: boolean
  uses_mcp: boolean
  uses_web_search: boolean
  uses_web_fetch: boolean
  
  // Additional stats
  lines_added: number
  lines_removed: number
  files_modified: number
  message_hours: number[]                        // hour of day for each msg
  user_message_timestamps: string[]              // ⭐ ISO 时间戳(多 Claude 检测用)
}
```

**deterministic 数据**:从 jsonl 计算,**不需要 LLM**。cache 到 `~/.claude/usage-data/session-meta/`。

### Layer 2 — `SessionFacets`(单 session LLM 提取的 qualitative)[code: insights.ts:260-273]

```typescript
type SessionFacets = {
  session_id: string
  underlying_goal: string                        // ⭐ LLM 推断:用户真正想做什么
  goal_categories: Record<string, number>        // ⭐ 13 enum 之一
  outcome: string                                 // ⭐ 5 enum: fully/mostly/partially/not_achieved/unclear
  user_satisfaction_counts: Record<string, number> // ⭐ 8 enum
  claude_helpfulness: string                     // ⭐ 5 enum
  session_type: string                            // ⭐ 5 enum
  friction_counts: Record<string, number>        // ⭐ 12 enum
  friction_detail: string                         // 自由文字描述
  primary_success: string                         // 自由文字
  brief_summary: string                           // 自由文字
  user_instructions_to_claude?: string[]         // ⭐ 用户重复指令(给 MULTIPLE TIMES 算法)
}
```

**LLM-extracted 数据**:用 Opus 跑 `FACET_EXTRACTION_PROMPT`(:430-456)提取。cache 到 `~/.claude/usage-data/facets/`,**永不重算**(per-session 不变 invariant)。

### Layer 3 — `AggregatedData`(跨 session 聚合,30+ 字段)[code: insights.ts:275-326]

```typescript
type AggregatedData = {
  total_sessions: number
  sessions_with_facets: number
  date_range: { start: string; end: string }
  total_messages: number
  total_duration_hours: number
  total_input_tokens / output_tokens: number
  
  tool_counts / languages / projects: Record<string, number>
  
  // 聚合 6 类 enum 维度
  goal_categories / outcomes / satisfaction / helpfulness / session_types / friction / success: 
    Record<string, number>
  
  session_summaries: Array<{id, date, summary, goal?}>
  
  // 行为 metric
  total_interruptions / total_tool_errors: number
  tool_error_categories: Record<string, number>
  user_response_times: number[]
  median_response_time / avg_response_time: number
  
  // 特性使用率
  sessions_using_{task_agent, mcp, web_search, web_fetch}: number
  
  // 代码改动
  total_lines_added / total_lines_removed / total_files_modified: number
  
  days_active: number
  messages_per_day: number
  message_hours: number[]                         // 时间分布
  
  // ⭐ Multi-clauding 检测(独家)
  multi_clauding: {
    overlap_events: number
    sessions_involved: number
    user_messages_during: number
  }
}
```

---

## 3. 6 维 typed enum schema [verified `insights.ts:352-415` `LABEL_MAP`]

```typescript
const LABEL_MAP: Record<string, string> = {
  // === Goal categories(13 类:用户想干什么)===
  debug_investigate: 'Debug/Investigate',
  implement_feature: 'Implement Feature',
  fix_bug: 'Fix Bug',
  write_script_tool: 'Write Script/Tool',
  refactor_code: 'Refactor Code',
  configure_system: 'Configure System',
  create_pr_commit: 'Create PR/Commit',
  analyze_data: 'Analyze Data',
  understand_codebase: 'Understand Codebase',
  write_tests: 'Write Tests',
  write_docs: 'Write Docs',
  deploy_infra: 'Deploy/Infra',
  warmup_minimal: 'Cache Warmup',
  
  // === Success factors(7 类)===
  fast_accurate_search / correct_code_edits / good_explanations /
  proactive_help / multi_file_changes / handled_complexity / good_debugging
  
  // === Friction(12 类:Layer D 失败归因)===
  misunderstood_request / wrong_approach / buggy_code /
  user_rejected_action / claude_got_blocked / user_stopped_early /
  wrong_file_or_location / excessive_changes / slow_or_verbose /
  tool_failed / user_unclear / external_issue
  
  // === Satisfaction(8 类)===
  frustrated / dissatisfied / unsure / neutral /
  likely_satisfied / satisfied / happy / delighted
  
  // === Session type(5 类)===
  single_task / multi_task / iterative_refinement / exploration / quick_question
  
  // === Outcome(5 类)===
  fully_achieved / mostly_achieved / partially_achieved /
  not_achieved / unclear_from_transcript
  
  // === Helpfulness(5 类)===
  unhelpful / slightly_helpful / moderately_helpful / very_helpful / essential
}
```

**6 维 × 平均 7 enum = 共 55 个 typed enum 值** — **业内 Layer D typed enum schema 最完整实现**(对照 tau-bench 7 enum 值 / AgentOps 15 enum 值)。

---

## 4. `FACET_EXTRACTION_PROMPT` [verified `insights.ts:430-456`]

完整 prompt 模板:

```python
Analyze this Claude Code session and extract structured facets.

CRITICAL GUIDELINES:

1. **goal_categories**: Count ONLY what the USER explicitly asked for.
   - DO NOT count Claude's autonomous codebase exploration
   - DO NOT count work Claude decided to do on its own
   - ONLY count when user says "can you...", "please...", "I need...", "let's..."

2. **user_satisfaction_counts**: Base ONLY on explicit user signals.
   - "Yay!", "great!", "perfect!" → happy
   - "thanks", "looks good", "that works" → satisfied
   - "ok, now let's..." (continuing without complaint) → likely_satisfied
   - "that's not right", "try again" → dissatisfied
   - "this is broken", "I give up" → frustrated

3. **friction_counts**: Be specific about what went wrong.
   - misunderstood_request: Claude interpreted incorrectly
   - wrong_approach: Right goal, wrong solution method
   - buggy_code: Code didn't work correctly
   - user_rejected_action: User said no/stop to a tool call
   - excessive_changes: Over-engineered or changed too much

4. If very short or just warmup, use warmup_minimal for goal_category

SESSION:
<session transcript here>
```

**3 个关键 prompt 设计**:

| 设计 | 意图 |
|---|---|
| **goal_categories 只数用户 explicit 请求** | 防 Claude 自主探索污染 _"用户真正想做什么"_ 统计 |
| **satisfaction 按文字标记 explicit signal** | 防 LLM 自己猜情感,要看实际用户字面表达 |
| **friction 5 类显式列举 + 描述** | 防 LLM 给 fuzzy 描述,强制选标准 enum |
| `warmup_minimal` 边界 case | 短 session(只是 warmup)单独归类,不污染主要分析 |

---

## 5. 8 个并行 `INSIGHT_SECTIONS` [verified `insights.ts:1336-1495`]

每个 section 单独 LLM call(Opus,maxTokens=8192),输出 JSON。**并行跑节省时间**。

| Section | 用途 |
|---|---|
| `project_areas` | 4-5 个 area / project 分布 + description |
| `interaction_style` | 用户跟 Claude Code 互动模式 narrative(_"detailed upfront"_ vs _"iterate quickly"_)+ key_pattern 一句话 |
| `what_works` | 3 个 impressive workflow(用 _"you"_ 二人称)|
| `friction_analysis` | 3 个 friction category × 2 examples each |
| **`suggestions`** ⭐ | **Layer E 关键:claude_md_additions + features_to_try + usage_patterns** |
| `on_the_horizon` | 3 个未来 opportunity(parallel agents / autonomous workflows) |
| `cc_team_improvements`(Ant-only) | 给 Anthropic 团队的产品改进建议 |
| `model_behavior_improvements`(Ant-only) | 给模型团队的 behavior 改进建议 |
| `fun_ending` | _memorable qualitative moment_(增加用户阅读体验) |

---

## 6. `suggestions` section 是 Layer E 关键 [verified `insights.ts:1392-1432`]

```
## CC FEATURES REFERENCE (pick from these for features_to_try):
1. MCP Servers
2. Custom Skills
3. Hooks
4. Headless Mode
5. Task Agents

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "claude_md_additions": [
    {
      "addition": "A specific line or block to add to CLAUDE.md based on workflow patterns. 
                   E.g., 'Always run tests after modifying auth-related files'",
      "why": "1 sentence explaining why this would help based on actual sessions",
      "prompt_scaffold": "Instructions for where to add this in CLAUDE.md. 
                          E.g., 'Add under ## Testing section'"
    }
  ],
  "features_to_try": [...],
  "usage_patterns": [...]
}

IMPORTANT for claude_md_additions: 
  PRIORITIZE instructions that appear MULTIPLE TIMES in the user data. 
  If user told Claude the same thing in 2+ sessions (e.g., 'always run tests', 'use TypeScript'), 
  that's a PRIME candidate — they shouldn't have to repeat themselves.
```

⭐ **MULTIPLE TIMES 算法**:_"用户在 2+ session 反复说同一件事 → 自动建议变成 CLAUDE.md 规则"_。**Layer E 自动归因的核心 wisdom**(详见 [[deep/claude-code-insights]] §核心算法)。

---

## 7. 关键 helper 函数 [verified `insights.ts:467-1145`]

| 函数 | 行号 | 用途 |
|---|---|---|
| `getAnalysisModel` / `getInsightsModel` | :41-48 | 都返 `getDefaultOpusModel()` — 用 Opus 不用 Haiku |
| `extractToolStats` | :467 | 从 jsonl log 提取 tool usage 统计 |
| `formatTranscriptForFacets` | :831 | 格式化 transcript 给 LLM 看 |
| `SUMMARIZE_CHUNK_PROMPT` | :870 | 长 transcript 分块预 summarize |
| `loadCachedFacets` / `saveFacets` | :941, :963 | facets 持久化(`~/.claude/usage-data/facets/`)|
| `loadCachedSessionMeta` / `saveSessionMeta` | :976, :988 | meta 持久化 |
| `extractFacetsFromAPI` | :1001 | 主入口:跑 LLM 提取 facets |
| **`detectMultiClauding`** | **:1062-1143** | ⭐ Sliding window 滑窗算法检测多 Claude 并发会话 |
| `aggregateData` | :1145 | 跨 session 聚合所有 30+ 字段 |
| `generateSectionInsight` | :1572 | 单 section LLM call |
| `generateParallelInsights` | :1612 | 并行跑 8 sections |
| `generateHtmlReport` | :1947 | 生成 interactive HTML report |
| `buildExportData` | :2679 | 给后台 S3 upload 用 |
| `scanAllSessions` | :2755 | filesystem scan(no jsonl parse,fast lite scan)|
| **`generateUsageReport`** | **:2796** | ⭐ 主入口(4 阶段流程)|

---

## 8. `detectMultiClauding` 滑窗算法 [verified `insights.ts:1062-1143`]

业内独家功能:**检测用户同时跑多个 Claude Code session**(多窗口并发)。

```typescript
const OVERLAP_WINDOW_MS = 30 * 60000   // 30 分钟窗口

// 算法:把所有 session 的 user message 时间戳串成时间轴 sliding window
// 检测 pattern:s1 → s2 → s1(说明 s1 期间用户去了 s2)

return {
  overlap_events: 重叠 pair 数,
  sessions_involved: 涉及的 session 数,
  user_messages_during: 重叠期间的 user msg 数,
}
```

**洞察**:Claude Code 关心 _"用户是不是真的专注一个 session"_ — 多 Claude 并发可能稀释 Claude 帮助效率(用户 attention 切来切去)。这是 Anthropic 在 _"agent productivity 评测"_ 的细节关注。

---

## 9. `InsightsExport` 后台 S3 upload [verified `insights.ts:2655-2737`]

```typescript
export type InsightsExport = {
  metadata: {
    username: string                              // SAFEUSER / USER env
    generated_at: string
    claude_code_version: string
    date_range: { start: string; end: string }
    session_count: number
    remote_hosts_collected?: string[]             // SCP 拉的远程 Coder workspace
  }
  aggregated_data: AggregatedData                  // 完整 30+ 字段
  insights: InsightResults                         // 8 section 结果
  facets_summary?: {                               // ⭐ 6 维 enum 聚合
    total: number
    goal_categories: Record<string, number>
    outcomes: Record<string, number>
    satisfaction: Record<string, number>
    friction: Record<string, number>
  }
}
```

**`buildExportData`** 把所有数据打包,**后台 upload S3**(给 Anthropic 产品团队分析 — 注意 Ant-only `cc_team_improvements` + `model_behavior_improvements` 两个 section 就是给团队看的)。

---

## 10. Remote host data collection [verified `insights.ts:60-220`]

Ant-only(`process.env.USER_TYPE === 'ant'`):

```typescript
// 自动发现 running Coder workspace
const getRunningRemoteHosts = async () => {
  const { stdout } = await execFileNoThrow('coder', ['list', '-o', 'json'])
  return workspaces.filter(w => w.latest_build?.status === 'running').map(w => w.name)
}

// SCP 拉远程 ~/.claude/projects/
const collectFromRemoteHost = async (homespace, destDir) => {
  await execFileNoThrow('scp', ['-rq', `${homespace}.coder:/root/.claude/projects/`, tempDir])
  // 然后 merge 到 destDir(skip existing files,project name suffix with __homespace)
}
```

**意义**:Anthropic 员工通常多设备工作(local + Coder remote),`/insights` 自动汇总跨设备数据。**普通用户没这功能**(condition gate `USER_TYPE === 'ant'`)。

---

## 11. 完整 4 阶段执行流(`generateUsageReport` `:2796-`)

```
[Phase 0] (Ant-only) collectAllRemoteHostData → SCP 远程 sessions

[Phase 1] scanAllSessions
   ↓ readdir(~/.claude/projects/) 不读 jsonl, 只 stat(mtime+size)
   ↓ 按 mtime 倒序排
   ↓ yield to event loop 每 10 个 project dir (避免 block UI)
   
[Phase 2] Load SessionMeta
   ↓ META_BATCH_SIZE = 50 并行 batch
   ↓ MAX_SESSIONS_TO_LOAD = 200 上限
   ↓ cache hit → 直接用; cache miss → 解析 jsonl
   
[Phase 3] Load / Extract SessionFacets
   ↓ 每 uncached session → extractFacetsFromAPI (Opus LLM call)
   ↓ 长 transcript → summarizeTranscriptChunk 预 chunk
   
[Phase 4] aggregateData → generateParallelInsights → generateHtmlReport
   ↓ 8 sections 并行 Opus LLM call
   ↓ buildExportData → upload S3 (后台)
   ↓ 输出 HTML 到本地 + 自动打开浏览器
```

**性能优化**:
- Phase 1 lite scan(只 stat,不解析 jsonl)
- Phase 2 batch 50 并行 load cached meta
- Phase 3 facets per-session 永久 cache
- Phase 4 8 sections 并行 LLM call

---

## 12. 缓存策略 [verified]

```
~/.claude/usage-data/
├── session-meta/<session_id>.json     # Layer 1 deterministic stats cache
├── facets/<session_id>.json           # Layer 2 LLM-extracted qualitative cache
└── (HTML report 不缓存, 每次重生成)
```

**facets per-session 永久缓存** — 一个 session 的 facets 一旦提取,**不会重跑 LLM**(因为 session 内容不变)。这意味着第一次跑 `/insights` 可能要等几分钟(N sessions × Opus call),后续跑只增量提取新 sessions 的 facets,**几秒钟出 report**。

---

## 13. 关键 verified 锚点(供后续 query)

| 路径 | 主题 |
|---|---|
| `src/commands/insights.ts:41-48` | Opus model 选择(2 个 getter 都用 Opus) |
| `src/commands/insights.ts:228-258` | SessionMeta schema(deterministic stats) |
| `src/commands/insights.ts:260-273` | SessionFacets schema(LLM-extracted qualitative) |
| `src/commands/insights.ts:275-326` | AggregatedData schema(30+ 字段) |
| `src/commands/insights.ts:352-415` | LABEL_MAP — **6 维 55 enum 完整 schema** |
| `src/commands/insights.ts:430-456` | FACET_EXTRACTION_PROMPT 全文 |
| `src/commands/insights.ts:1062-1143` | detectMultiClauding 滑窗算法 |
| `src/commands/insights.ts:1336-1495` | INSIGHT_SECTIONS 8 个 + prompts 全 |
| `src/commands/insights.ts:1392-1432` | **`suggestions` section + MULTIPLE TIMES 算法**(Layer E 关键) |
| `src/commands/insights.ts:2655-2737` | InsightsExport + buildExportData(后台 upload) |
| `src/commands/insights.ts:2796+` | generateUsageReport 主入口 + 4 阶段流程 |

---

## 14. 引用 / wikilink

- 主源文件:`/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/commands/insights.ts` 3200 行
- 配套:`src/utils/sessionStorage.ts`(getProjectsDir / getSessionFilesWithMtime / loadAllLogsFromSessionFile)
- Harness 入口:[[../harness/claude-code]] §self_evolution(本次更新加 insights 章节)
- Concept:[[../concept/attribution]] §Layer E(本次加 Claude Code `/insights` 作为 _"用户手动 review"_ 路线)
- Deep:[[../deep/claude-code-insights]] depth-first 单机制深挖
- 横向 deep 对照:[[../deep/claude-code-autodream-3-gate]](Anthropic 的另一条 Layer E 路径)
