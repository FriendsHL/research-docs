---
type: deep
name: claude-code-compact-tool-result
status: B-mode source-grounded
harness: claude-code
mechanism: cache-editing microcompact + COMPACTABLE_TOOLS allowlist + time-based trigger + 3-UUID preserved-segment relink
updated: 2026-05-17
sources:
  - "[[harness/claude-code]]"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/compact/microCompact.ts:1-530"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/compact/compact.ts:1-1705"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/compact/cachedMicrocompact.ts (referenced)"
---

# Claude Code Compact Tool Result — cache-editing 不删本地消息只发 cache_edits 给 server,保 prompt cache 命中

> 要点:Claude Code 的 _"context compaction"_ 不是简单 _"删除老 tool_results"_。生产 Claude API 有 prompt cache(写每个 1M token cache 极贵,读 cache 极便宜),naive 删除消息会**整个 cache prefix 失效**。Claude Code 解法:**cached microcompact 路径不改本地 message,只发 `cache_edits` block 给 API**,让 server-side 删除 cached tool_results,本地 message 顺序保持完整 — **prompt cache 仍命中**。
>
> 这页深挖 3 个反直觉工程决策:**(1) `COMPACTABLE_TOOLS` 是 8 个工具的显式 allowlist**,Memory / SubAgent / Skill 调用不可被 compact;**(2) Time-based trigger** 在 cache expire 之前主动清,而非 token-pressure trigger;**(3) `annotateBoundaryWithPreservedSegment` 用 3 个 UUID(head/anchor/tail)做 message DAG relink**,让 compact 之后 loader 能修复 parent_uuid 链不破。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通用 Claude Code 一段时间会发现:
- 长 session 里 token 用量**没爆**,但你又没手动 `/compact`
- 老的 `Read tool_result`(几千字符内容)**消失了**,被 `[Old tool result content cleared]` 替代
- 但 `Memory` tool 的 tool_result **从不消失**(skill 调用的也不消失)
- prompt cache 命中率**没受太大影响**(API 账单不暴涨)

这就是 microcompact 在工作。**它跟 `/compact` 是 2 个机制**:
- `/compact`(`compact.ts:387` `compactConversation`)— 用户主动触发,LLM 总结整个对话历史
- microcompact(`microCompact.ts`)— **自动 background**,只删除老的 tool_result 内容

为什么这个机制容易被忽略:
1. 用户**看不到**(本地 message 没改,只是 server-side cache 被编辑)
2. 实现非常分散(`microCompact.ts:1-530` + `cachedMicrocompact.ts` + `compact.ts:1705` + `timeBasedMCConfig.ts` + `apiMicrocompact.ts`)
3. 跟 prompt cache 强耦合(不懂 Anthropic prompt cache 看不出门道)
4. 业内大部分 framework **没有这个层次的优化**(直接 token-trigger 删消息)

**对 production LLM agent 的启示**:**context 管理不是单一 strategy**。Claude Code 有 5 级 compaction(详见 [[harness/claude-code]] context_mgmt 维度):
1. **Cached microcompact**(本页主题)— 不改本地,只发 cache_edits,server 删 cached tool_result
2. **Time-based microcompact** — cache 已经 expire 时,主动清空 tool_result content
3. **Auto-compact** — token 压力时 LLM 总结对话
4. **Manual `/compact`** — 用户触发(可带 customInstructions)
5. **Reactive 413 recovery** — API 返回 prompt-too-long 时回退重试

这页聚焦第 1+2 级。

---

## 2. 机制 — `COMPACTABLE_TOOLS` allowlist + 2 路径

### 2.1 `COMPACTABLE_TOOLS` 8 工具显式 allowlist

```typescript
[code: microCompact.ts:40-50]
// Only compact these tools
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,        // 文件读
  ...SHELL_TOOL_NAMES,         // Bash / shell
  GREP_TOOL_NAME,              // grep
  GLOB_TOOL_NAME,              // glob
  WEB_SEARCH_TOOL_NAME,        // web search
  WEB_FETCH_TOOL_NAME,         // web fetch
  FILE_EDIT_TOOL_NAME,         // file edit
  FILE_WRITE_TOOL_NAME,        // file write
])
```

**8 个工具**全是 _"输出大 + 一次性 + 可重新调用"_ 的工具。明显**不在**列表里的:
- `Memory` tool(用户长期记忆)
- `Skill` tool(技能调用)
- `Task` tool(子 task 协调,见 [[deep/claude-code-autodream-3-gate]])
- `AgentTool` (SubAgent spawn)
- `TodoWrite` (任务进度)
- `AskUserQuestion` (用户提问)

**为什么这 8 个 vs 那些**:
- ✅ **Compactable**:输出可重得(可以重新 read / grep / search),内容大,旧的不再相关
- ❌ **Not compactable**:语义密集 / 一次性不可重得(Memory 是用户输入的事实,Skill 调用是 first-class 上下文,SubAgent 输出包含子任务全部信息,TodoWrite 是 plan state)

**对照** [[harness/agentscope]] `CompactableToolRegistry.java:9-15`(SkillForge 路径):同款 _"Memory / SubAgent 不可 compact"_ pattern,但 SkillForge 是 inverse(whitelist 风:_"哪些可 compact"_),opencode 是 PRUNE_PROTECTED(blocklist 风:_"哪些 protected"_)。Claude Code 是 allowlist 风,精确控制哪 8 个。

### 2.2 `cachedMicrocompactPath` — 不删消息,发 cache_edits

```typescript
[code: microCompact.ts:305-379]
async function cachedMicrocompactPath(messages, querySource): Promise<MicrocompactResult> {
  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  
  // Second pass: register tool results grouped by user message
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (
          block.type === 'tool_result' &&
          compactableToolIds.has(block.tool_use_id) &&
          !state.registeredTools.has(block.tool_use_id)
        ) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }
  
  const toolsToDelete = mod.getToolResultsToDelete(state)
  
  if (toolsToDelete.length > 0) {
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits     // ⭐ 不修改 message, 排队给 API layer
    }
    ...
    // Return messages unchanged - cache_reference and cache_edits added at API layer
    return { messages: unchanged }
  }
}
```

**关键设计**:**返回 `messages` 完全不变**。`pendingCacheEdits` 排队,在下一次 API call 时 attach 到 request,server 解析 `cache_edits` block,在 server-side cache 删除 cached tool_results。

**为什么 prompt cache 不破**:
- Anthropic prompt cache 按 message 顺序匹配(prefix match)
- 如果本地 message 顺序变了,cache miss
- Cache editing API 让 server 知道 _"删除这些 tool_result_id 的内容,但保留 message 顺序"_
- Cache prefix 仍然 hit,**只是删除部分内容**

业内对照:OpenAI / 其他 provider 没有 cache editing API,所以这个机制**只在 Claude API 工作**。Codex / Aider 等 framework 无法复制。

### 2.3 Time-based trigger — 不等 token 压力,主动清

```typescript
[code: microCompact.ts:261-269]
// Time-based trigger runs first and short-circuits. If the gap since the
// last assistant message exceeds the threshold, the server cache has expired
// and the full prefix will be rewritten regardless — so content-clear old
// tool results now, before the request, to shrink what gets rewritten.
// Cached MC (cache-editing) is skipped when this fires: editing assumes a
// warm cache, and we just established it's cold.
const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
if (timeBasedResult) {
  return timeBasedResult
}
```

**这段注释是 production engineering wisdom 集大成**。翻译:
- Anthropic prompt cache 有 TTL(~5 分钟空闲就 expire)
- 如果用户离开 5 分钟回来,**整个 prefix 都要重写**
- 既然要重写,**何不在重写前先清空老 tool_result**?重写时少传 tokens
- **不能用 cache-editing**(cache 已 cold,editing 没用)— fallback 到 _"本地替换内容"_ 路径

**触发时机**:不是 token 满了才 trigger,**是检测到 cache 即将/已经 expire 时 proactive trigger**。

业内对照:大部分 framework 是 _"reactive compact"_(token 满了才动),Claude Code 是 _"proactive + reactive 双路径"_。

### 2.4 Time-based 替换内容

```typescript
[code: microCompact.ts:36]
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

被 clear 的 tool_result 内容替换为这条 placeholder。Tool_use 本身 + 元数据保留(LLM 看到 _"我之前调用过 Read X.py,但内容已清"_),只是内容空了。

**为什么留 placeholder 而非完全删除**:
- 删除会破坏 tool_use → tool_result 配对(LLM API 会拒绝)
- Placeholder 让 LLM 知道 _"我做过这件事,但具体输出忘了,需要时重新 read"_

---

## 3. `annotateBoundaryWithPreservedSegment` — 3-UUID 修复 message DAG link

普通 compact(`compact.ts:387` `compactConversation`)做的事更激进:**LLM 总结老消息,删除老消息,保留新消息**。但这会破坏 _message UUID 链_:

```
原 message 链(parent_uuid):
  msg-1 → msg-2 → msg-3 → msg-4 → msg-5 → msg-6
  
compact 后:
  msg-1 → [SUMMARY] → msg-5 → msg-6
  
但 msg-5.parent_uuid 还指向 msg-4 (已删) — 断链!
```

`compact.ts:340-367` 的 `annotateBoundaryWithPreservedSegment` 做修复:

```typescript
[code: compact.ts:349-367]
export function annotateBoundaryWithPreservedSegment(
  boundary: SystemCompactBoundaryMessage,
  anchorUuid: UUID,
  messagesToKeep: readonly Message[] | undefined,
): SystemCompactBoundaryMessage {
  const keep = messagesToKeep ?? []
  if (keep.length === 0) return boundary
  return {
    ...boundary,
    compactMetadata: {
      ...boundary.compactMetadata,
      preservedSegment: {
        headUuid: keep[0]!.uuid,        // ⭐ 保留段的第一条
        anchorUuid,                      // ⭐ 锚点 (boundary 前面的 SUMMARY)
        tailUuid: keep.at(-1)!.uuid,    // ⭐ 保留段的最后一条
      },
    },
  }
}
```

**3 个 UUID 做的事**:
- `headUuid`:保留段第一条 message 的 UUID(`msg-5`)
- `anchorUuid`:它应该挂在哪里(`SUMMARY` 或 boundary 本身)
- `tailUuid`:保留段最后一条(`msg-6`)— 用于后续 message 找 parent

**loader 加载时**:看到 `preservedSegment` metadata,**patch** `msg-5.parent_uuid = anchorUuid`(让它指向 SUMMARY 而非 deleted msg-4),并修复 `anchorUuid 的其他 children → tailUuid` 链。

**注释明确说**(`compact.ts:341-348`):
> _"Preserved messages keep their original parentUuids on disk (dedup-skipped); the loader uses this to patch head→anchor and anchor's-other-children→tail."_

**为什么不直接改 message 的 parent_uuid?**
- Message 文件是 dedup-skipped(同 UUID 跨 session 共享),改一处影响全局
- preservedSegment metadata 是 _"加载时 patch"_,**不破坏 on-disk 真实链**

这是 _"persistent data structure"_ 思想 — 不变 on-disk,变 in-memory view。

---

## 4. WHY — 为什么这么设计

### 4.1 为什么用 cache_edits 而非删本地消息?

**核心约束**:Anthropic prompt cache 是按 byte-level prefix 匹配。本地任何改动 → cache miss → 重写 cost = 长 prefix 的 input tokens × $$$$。

```
传统做法:
  本地删 5 个 tool_result → message 长度变 → cache miss
  → 整个 prefix 重新算 cache → cost N × write_cost
  
Cache editing:
  本地 message 不变 → cache 仍 prefix 匹配
  → server 看到 cache_edits → 删 cached tool_result content
  → cost 几乎 0 (cache hit + 微小 edit overhead)
```

**Hot conversation 节省效果**(代码注释 `compact.ts:432-434`):
> _"3P default: true — forked-agent path reuses main conversation's prompt cache. Experiment (Jan 2026) confirmed: false path is 98% cache miss, costs ~0.76% of fleet cache_creation (~38B tok/day), concentrated in ephemeral envs (CCR/GHA/SDK) with cold GB cache and 3P providers where GB is disabled."_

**关键数字**:false 路径(不共享 cache)有 **98% cache miss**,每天浪费 **38 B tokens** 写 cache。Claude Code 把这个数字 measured 出来,然后切到 true 默认。**这是 production-tested**。

### 4.2 为什么 allowlist 8 个工具?

3 个分类标准:

| 工具 | 输出大小 | 可重得? | 语义密度 | Compactable? |
|---|---|---|---|---|
| FileRead | 大(整文件)| ✅(重读)| 低(纯文本)| ✅ |
| Bash | 大(stdout)| 部分(同命令可重跑,但有副作用)| 低 | ✅ |
| Grep | 大(命中列表)| ✅ | 低 | ✅ |
| Glob | 中(文件列表)| ✅ | 低 | ✅ |
| WebFetch | 大(HTML/JSON)| ✅(URL 重 fetch)| 低 | ✅ |
| WebSearch | 中(搜索结果)| ✅ | 低 | ✅ |
| FileEdit | 小(确认 message)| N/A(side effect 已发生)| 中 | ✅(确认信息可丢) |
| FileWrite | 同 FileEdit | 同 | 中 | ✅ |
| **Memory** | **小**(用户事实)| **❌**(用户输入)| **高** | ❌ |
| **Skill** | **中**(技能内容)| **❌**(skill 已 install)| **高** | ❌ |
| **SubAgent** | **大**(子 task 输出)| **❌**(spawn 一次,昂贵)| **高** | ❌ |
| **TodoWrite** | **小**(plan state)| **N/A** | **高** | ❌ |
| **AskUserQuestion** | **小** | **❌**(用户输入)| **高** | ❌ |

**规则**:_"输出可重得 + 语义低密 + 一次性"_ 的工具 = compactable。

### 4.3 为什么 time-based 而非 token-based?

如前 §2.3 — **如果 cache 已经 expire,反正要重写,不如先清理**。token-based trigger 是 _"token 满了被动反应"_,time-based 是 _"cache 即将 expire 主动准备"_。

两者**互补**,不是替代。代码 `:267-269` 注释:
> _"Cached MC (cache-editing) is skipped when this fires: editing assumes a warm cache, and we just established it's cold."_

Cache editing 假设 cache 是 warm,time-based 在 cache 已 cold 时跑(用 fallback 路径)。

### 4.4 为什么不直接改 message 的 parent_uuid?

如前 §3 — message file 是 _"内容寻址"_(UUID hash),改一处影响全局(其他 session 引用同 UUID 也变)。`preservedSegment` metadata 是 _"on-disk 不变 + 加载时 view 变换"_,这是 functional / persistent data structure 思想。

---

## 5. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | Claude Code 怎么防 | 代码位置 |
|---|---|---|
| **Naive 删消息破 prompt cache,每次 compact 浪费 token** | Cache editing API,本地 message 不变,只删 server-side cached content | `microCompact.ts:305-379` |
| **删除老 tool_result 时 tool_use 变孤儿(API 拒绝)** | 保留 tool_use + 替换 content 为 placeholder `[Old tool result content cleared]` | `microCompact.ts:36` |
| **误删 Memory / SkillUse / SubAgent 输出(高语义密度)** | `COMPACTABLE_TOOLS` Set 显式 allowlist 8 个,其他不动 | `microCompact.ts:40-50` |
| **`/compact` 总结老消息后 message_uuid 链断** | `annotateBoundaryWithPreservedSegment` 3-UUID metadata + loader patch | `compact.ts:340-367` |
| **改 message file 影响其他 session(dedup-shared)** | metadata-only(loader-time patch),on-disk 不动 | `compact.ts:341-343` |
| **compact 自己请求 hit prompt-too-long(总结输入太长)** | `truncateHeadForPTLRetry` 截前最老 API-round 重试,`MAX_PTL_RETRIES = 3` | `compact.ts:227,243-` |
| **Cache 已 expire 时还 cached MC(无效)** | Time-based trigger 先判断,cache cold 时跳 cached MC 用 fallback | `microCompact.ts:261-292` |
| **Output style 变体被 cache MC 排除(latent bug)** | `isMainThreadSource` prefix-match 而非 exact-match("repl_main_thread:outputStyle:..." 也算 main thread)| `microCompact.ts:243-251` |
| **Forked sub-agent 注册 tool_result 污染主 thread cache state** | `cachedMicrocompactPath` 只在 main thread 跑,sub-agent 不进 | `microCompact.ts:272-286` |
| **Image / document 估 token 不准** | `IMAGE_MAX_TOKEN_SIZE = 2000` 不论 format 都按 2000 算 | `microCompact.ts:38, 152-153` |

**10 类 production 故障显式 cover** — 跟 [[deep/claude-code-autodream-3-gate]] 同款,production-tested signal 在源码注释里。

---

## 6. 横向校准 — 5 家 framework tool result 压缩对比

| | Claude Code | Hermes | opencode | AgentScope/SkillForge | Codex |
|---|---|---|---|---|---|
| **核心策略** | ⭐ Cache editing API(server-side 删)| 本地 `_prune_old_tool_results` 替换 | `PRUNE_PROTECTED_TOOLS = ["skill"]` blocklist | `CompactableToolRegistry` allowlist | 3 套 compact + InitialContextInjection |
| **Allowlist / Blocklist** | ✅ Allowlist 8 工具 | 隐式(按 tool name pattern) | ✅ Blocklist(skill only protected) | ✅ Allowlist(default 11 + per-agent override) | ? |
| **保留 prompt cache** | ⭐ **是**(cache_edits API) | 否(改本地 = cache miss) | 否 | 否 | 部分(InitialContextInjection 训练对齐位置) |
| **Time-based trigger** | ⭐ 有(cache expire 前主动)| 无(token-based) | 无 | 无 | 无 |
| **Placeholder text** | `[Old tool result content cleared]` | `[Duplicate tool output ...]` | ? | ? | ? |
| **配对修复(orphan tool_use)** | `annotateBoundaryWithPreservedSegment` | `_sanitize_tool_pairs:1391` | 隐式 | 隐式 | 隐式 |
| **5 级 compaction** | ⭐ 是(cached MC / time MC / auto / manual / reactive 413)| 1 级(commit_memory_session) | 1 级 | 1 级 | 3 套(compact / compact_remote / compact_remote_v2) |

**核心洞察**:**只有 Claude Code 有 cache-editing 路径**,因为只有 Claude API 支持这个 server-side 操作。其他 framework 即使想做也做不了(API 不支持)。

**这是 _"端到端控制 stack 的优势"_** — Anthropic 既做模型 + API + Claude Code,可以在 API 层加 cache_edits 这种 niche feature 给自家 client 用。OpenAI 不会给 Codex 加 cache_edits,因为 OpenAI 自己用 ChatGPT 不需要。

---

## 7. 学习视角 — 写这页让我学到的 3 件事

1. **Prompt cache 是 production agent 的核心成本中心**:每次 compact / context manipulation 都要问 _"这会不会破 cache?"_。Claude Code 5 级 compaction 全部围绕 _"cache 友好"_ 设计,这才是为什么这套机制复杂(而不是 over-engineering)。
2. **Time-based proactive 比 token-based reactive 高级一级**:_"等问题发生再处理"_ 的工具(token full → compact)只是基本盘。_"预测问题发生时机提前准备"_(cache 即将 expire → 提前清)是另一个层级。**任何 production agent 系统都该考虑:有哪些状态变化是 _可预测_ 的?**
3. **`allowlist` 比 `blocklist` 强**:opencode `PRUNE_PROTECTED_TOOLS = ["skill"]` blocklist 默认假设 _"所有 tool 都可 compact 除非显式 protected"_,Claude Code `COMPACTABLE_TOOLS` allowlist 默认假设 _"所有 tool 都不可 compact 除非显式列出"_。**新增 tool 时,allowlist 是默认安全**(忘记加 → 不被错误 compact),blocklist 是默认危险(忘记加 → 被错误 compact)。这是 _"安全默认"_(secure by default)思想。

---

## 引用链路

- 源码:`/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/compact/`
  - `microCompact.ts:1-530` 完整(COMPACTABLE_TOOLS allowlist + cachedMicrocompactPath + time-based trigger)
  - `compact.ts:1-1705` 关键(annotateBoundaryWithPreservedSegment :349 + truncateHeadForPTLRetry :243 + compactConversation :387)
  - `cachedMicrocompact.ts`(referenced,本次未深读 — registerToolResult / createCacheEditsBlock / isModelSupportedForCacheEditing 等 helper)
  - `timeBasedMCConfig.ts`(referenced)+ `apiMicrocompact.ts`(referenced)
- Harness 入口:[[harness/claude-code]] §context_mgmt
- 横向 concept:[[concept/context-compaction]] / [[concept/runtime-context-injection]] §9 (cache-aware 设计)
- 横向 deep:[[deep/claude-code-autodream-3-gate]] (autoDream 共享 `runForkedAgent` + `createCacheSafeParams` cache 路径) / [[deep/codex-actor-resume-agent]] (Codex `InitialContextInjection` 路线对照)
