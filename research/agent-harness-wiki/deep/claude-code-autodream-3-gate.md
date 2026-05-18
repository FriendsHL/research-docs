---
type: deep
name: claude-code-autodream-3-gate
status: B-mode source-grounded
harness: claude-code
mechanism: 3-gate cheapest-first scheduler + 4-phase consolidation prompt + lock-file as mtime tracking
updated: 2026-05-17
sources:
  - "[[harness/claude-code]]"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/autoDream/autoDream.ts:1-324"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/autoDream/config.ts:1-21"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/autoDream/consolidationLock.ts:1-140"
  - "/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/autoDream/consolidationPrompt.ts:1-65"
---

# Claude Code autoDream — 一个 lock 文件 + 3 个 gate + 4 个 phase 如何撑起 background memory consolidation

> 要点:autoDream(`src/services/autoDream/` 4 文件共 550 行)是 Claude Code **后台 memory consolidation** 的全部实现。读完发现 3 个 production engineering wisdom 反直觉:**(1) 一个 lock 文件的 mtime 同时承担"上次 consolidation 时间戳"和"互斥锁"双重语义**;**(2) 3 个 gate 按 cost 严格排序**(stat → listdir → atomic write)— 大部分轮次只跑第 1 个 stat 就退出;**(3) 4-phase prompt 模板是 Anthropic 把"做 memory consolidation 像 PR review 一样规范化"的尝试**。
> 
> 这页是 wiki 第一个 deep/ 类型,聚焦 _"autoDream 是怎么 work 的"_,跟 [[concept/self-evolution]] 多 harness 横向 + [[harness/claude-code]] 13 维 catalog 互补。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通用 Claude Code 的人,长 session 后**自动**会发现 `~/.claude/memory/CLAUDE.md` 里的内容**变得更整洁、更去重、更带时间标记**。但**没人告诉你这是怎么发生的**。

实际触发路径:
- 每次 user prompt 提交 → `stopHooks` → `executeAutoDream` → 跑 3 个 gate
- 3 个 gate 全过 → fork 一个 sub-agent 跑 4-phase `/dream` prompt
- Sub-agent **只读 memory + transcript**,**写 memory file**(Bash 受限只读命令)
- 整个 consolidation 在 background,用户无感

**为什么容易被忽略**:
1. 跟 `/compact`(in-context compaction)不是一回事 — autoDream 是 **persistent memory consolidation**,compact 是 conversation context 压缩
2. 跟 user-triggered `/dream` 共享同一个 4-phase prompt,但 _auto_ 路径有 3-gate scheduler
3. 失败时**静默回退**(rollback mtime),用户看不到错误
4. 实现在 4 个小文件里(autoDream.ts + config.ts + consolidationLock.ts + consolidationPrompt.ts),没自家命名空间(不是 `MemoryConsolidator/`)

**对 production memory system 的启示**:**memory consolidation 不该出现在 user-visible 路径,它是后台进程**。autoDream 把这事做到极致:整个机制对用户透明,只有最终 `~/.claude/memory/` 里的文件越来越整洁。

---

## 2. 机制 — 3-gate cheapest-first scheduler

源码 `autoDream.ts:5-9` 开头注释直接写:

```typescript
// Gate order (cheapest first):
//   1. Time: hours since lastConsolidatedAt >= minHours (one stat)
//   2. Sessions: transcript count with mtime > lastConsolidatedAt >= minSessions
//   3. Lock: no other process mid-consolidation
```

这 3 个 gate 按 **syscall cost** 严格升序排列。完整流程(`autoDream.ts:125-191`):

```
                    Every user prompt
                          │
                          ▼
              ┌─ isGateOpen() ─────────────────┐
              │  KAIROS active?  →  return     │  ← 4 个 enable 前置 (无 syscall)
              │  Remote mode?    →  return     │
              │  autoMem disabled? → return    │
              │  autoDream disabled? → return  │
              └────────────────┬───────────────┘
                               │
                               ▼
              ┌─ Gate 1: Time ─────────────────┐
              │  readLastConsolidatedAt()       │  ← 1 个 stat syscall, mtime of lock file
              │  hoursSince < minHours (24h)?   │
              │    → return                     │
              └────────────────┬───────────────┘
                               │
                               ▼
              ┌─ Gate 1.5: Scan throttle ──────┐
              │  sinceScanMs < 10 min?          │  ← 内存比较, 0 syscall
              │    → return                     │     防止 Gate 2 抖动
              └────────────────┬───────────────┘
                               │
                               ▼
              ┌─ Gate 2: Sessions ─────────────┐
              │  listSessionsTouchedSince(...)  │  ← listdir + parallel stat (重)
              │  filter mtime > lastAt          │
              │  count < minSessions (5)?       │
              │    → return                     │
              └────────────────┬───────────────┘
                               │
                               ▼
              ┌─ Gate 3: Lock ─────────────────┐
              │  tryAcquireConsolidationLock()  │  ← stat + read + write + verify (最重)
              │  Stale 1h? Dead PID? → reclaim │
              │  Held by live PID? → return    │
              └────────────────┬───────────────┘
                               │
                               ▼
                  fork sub-agent + 4-phase prompt
                  registerDreamTask(...) → 跟踪
                  failure → rollbackConsolidationLock(priorMtime)
                  success → mtime stays at now (= new lastConsolidatedAt)
```

### 2.1 Gate 1 — Time(最便宜,一个 stat)

```typescript
[code: autoDream.ts:131-141]
let lastAt: number
try {
  lastAt = await readLastConsolidatedAt()  // 1 syscall: stat(lockPath)
} catch (e) { ... return }
const hoursSince = (Date.now() - lastAt) / 3_600_000
if (!force && hoursSince < cfg.minHours) return  // default 24h
```

**关键**:`lastConsolidatedAt` **就是 lock 文件的 mtime**。`readLastConsolidatedAt()`(`consolidationLock.ts:29-36`):

```typescript
export async function readLastConsolidatedAt(): Promise<number> {
  try {
    const s = await stat(lockPath())
    return s.mtimeMs
  } catch {
    return 0
  }
}
```

一个 stat。**每个 user prompt 跑一次**。如果 hoursSince < 24h,直接 return。**绝大部分 turn 在这里就停了**。

### 2.2 Gate 1.5 — Scan throttle(0 syscall,内存比较)

```typescript
[code: autoDream.ts:55-56, 143-151]
const SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000
...
const sinceScanMs = Date.now() - lastSessionScanAt
if (!force && sinceScanMs < SESSION_SCAN_INTERVAL_MS) {
  logForDebugging(`[autoDream] scan throttle — time-gate passed but last scan was ${...}s ago`)
  return
}
lastSessionScanAt = Date.now()
```

**这是 production engineering 智慧**。注释直接写明动机(`autoDream.ts:54-55`):

> _"Scan throttle: when time-gate passes but session-gate doesn't, the lock mtime doesn't advance, so the time-gate keeps passing every turn."_

**翻译**:如果你今天到了 25h 但 session 没攒够 5 个,**Gate 1 每个 turn 都过**,但 Gate 2 每个 turn 都不过。这样**每个 turn 都跑 listdir(`listSessionsTouchedSince`)** — 浪费。

**修法**:加一个**内存 throttle**,每 10 分钟最多扫一次 sessions。无 syscall,纯内存比较。

### 2.3 Gate 2 — Sessions(listdir + 多个 stat)

```typescript
[code: autoDream.ts:154-171]
let sessionIds: string[]
try {
  sessionIds = await listSessionsTouchedSince(lastAt)
} catch (e) { ... return }
const currentSession = getSessionId()
sessionIds = sessionIds.filter(id => id !== currentSession)
if (!force && sessionIds.length < cfg.minSessions) return  // default 5
```

`listSessionsTouchedSince`(`consolidationLock.ts:118-124`):

```typescript
export async function listSessionsTouchedSince(sinceMs: number): Promise<string[]> {
  const dir = getProjectDir(getOriginalCwd())
  const candidates = await listCandidates(dir, true)  // listdir + parallel stat
  return candidates.filter(c => c.mtime > sinceMs).map(c => c.sessionId)
}
```

**注释里的小细节**(`consolidationLock.ts:114-117`):

> _"Uses mtime (sessions TOUCHED since), not birthtime (0 on ext4)."_  
> _"Caller excludes the current session."_  
> _"Scans per-cwd transcripts — it's a skip-gate, so undercounting worktree sessions is safe."_

- **不用 birthtime**:因为 ext4 上 birthtime = 0,跨文件系统不可靠
- **排除当前 session**:它 mtime 总是新的,会污染计数
- **per-cwd 扫描足够**:即使漏掉 worktree session,gate 错误偏保守(少 trigger)是安全的

### 2.4 Gate 3 — Lock(最重,但只在前 2 gate 都过才跑)

```typescript
[code: autoDream.ts:173-190]
let priorMtime: number | null
if (force) {
  priorMtime = lastAt
} else {
  try {
    priorMtime = await tryAcquireConsolidationLock()
  } catch (e) { ... return }
  if (priorMtime === null) return
}
```

`tryAcquireConsolidationLock`(`consolidationLock.ts:46-84`)做的事:

```
1. stat(lockPath) + readFile(lockPath) 并发                ← 2 syscall
2. if mtime < 1h ago AND PID is live → 别人在跑 → return null
3. mkdir(memoryDir, recursive)                              ← 1 syscall (idempotent)
4. writeFile(lockPath, our PID)                             ← 1 syscall
5. readFile(lockPath) + parseInt + 比较 PID                ← 1 syscall (race detection)
6. 不是我们写的 → 我们输了 race → return null
7. 返回 priorMtime (供 rollback 用)
```

**关键 production engineering**:
- **PID stale guard**(`consolidationLock.ts:18-19`):`HOLDER_STALE_MS = 60 * 60 * 1000`。即使 PID 还活着,如果 lock 持续 > 1h,reclaim。防卡死。
- **Race detection on write**(`consolidationLock.ts:76-81`):写完立刻 read-back 验证 PID 是不是自己 — 因为同时两个 reclaimer 都写,**last writer wins** Linux fs 语义,Loser bails。
- **Memory dir 可能不存在**(`consolidationLock.ts:71`):mkdir recursive 是 idempotent,放在 writeFile 之前。

---

## 3. 4-phase Consolidation Prompt(forked sub-agent 跑的 prompt)

源码 `consolidationPrompt.ts:10-65` 完整 prompt template。4 个 phase:

```
Phase 1 — Orient
  - ls memory directory                    ← 看现状
  - read entrypoint (CLAUDE.md)             ← 理解 index
  - skim existing topic files               ← 防止建重复
  - if logs/ or sessions/ subdirectories,
    review recent entries

Phase 2 — Gather recent signal
  - Priority order:
    1. Daily logs (logs/YYYY/MM/YYYY-MM-DD.md)  ← append-only stream
    2. Existing memories that drifted (与现在代码冲突的事实)
    3. Transcript search (narrow grep, not exhaustive)
       `grep -rn "<narrow term>" ${transcriptDir}/ --include="*.jsonl" | tail -50`

Phase 3 — Consolidate
  - 合并 new signal 进 existing topic files (避免 near-duplicate)
  - Convert relative dates to absolute dates    ← "yesterday" → "2026-05-16"
  - Delete contradicted facts                   ← 修在 source

Phase 4 — Prune and index
  - Update entrypoint to stay < MAX_ENTRYPOINT_LINES (~500) and < 25KB
  - Each entry one line < 150 chars
  - Remove pointers to stale/wrong/superseded memories
  - Demote verbose entries (line > ~200 chars → shorten + move detail to topic file)
  - Resolve contradictions
```

**核心设计选择**:**强类型 4-phase 而非自由 reflection prompt**。对照其他 framework:
- **Hermes** `commit_memory_session` 是一个 LLM call,自由 prompt
- **OpenClaw** memory dreaming 有 6 维评分 + 梦境日记 subagent(但 prompt 自由度更高)
- **AgentScope** SummarySchema(`_react_agent.py:43-87`)5 字段 pydantic 强类型
- **Claude Code** 4-phase prompt 是 _step-by-step procedural instruction_,跟 OpenClaw 6 维 + AgentScope pydantic 是 3 种不同 _"约束 LLM consolidation 行为"_ 的路径

**Tool 约束**:`autoDream.ts:216-218` 在 prompt 末尾 inject:

```
**Tool constraints for this run:** Bash is restricted to read-only commands
(`ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail`, and similar).
Anything that writes, redirects to a file, or modifies state will be denied.
Plan your exploration with this in mind — no need to probe.
```

forked agent 的 `canUseTool = createAutoMemCanUseTool(memoryRoot)` 物理上限制只能写 memory dir + 读全局 — **memory consolidation 不能改用户代码**(safety 默认开)。

---

## 4. WHY — 为什么这么设计

### 4.1 为什么 mtime 同时承担"时间戳"和"锁"双重语义?

正常实现会有 2 个文件:`.last-consolidated`(时间戳)+ `.consolidate-lock`(互斥锁)。Claude Code **合并成 1 个**(`consolidationLock.ts:1-2`):

```
// Lock file whose mtime IS lastConsolidatedAt. Body is the holder's PID.
```

**好处**:
- **少 1 个文件**:文件系统 inode 数量减少
- **少 1 个 stat**:每个 turn 跑 Gate 1 只 stat 一个文件,不是两个
- **原子性**:lock acquire 成功 = mtime = now = lastConsolidatedAt 同时刷新(单文件 writeFile 是原子的)
- **rollback 语义清晰**:fork failure → `utimes(lockPath, priorMtime, priorMtime)` 把 mtime 推回去 + 清空 body — 一个 syscall 修复两个语义

**坏处**:有人可能不知道 mtime 还兼具 lock 时间戳,误读代码。`consolidationLock.ts:1-2` 注释就是防这个。

### 4.2 为什么 3 gate 排序是 time → sessions → lock?

按 **syscall cost** 升序:

| Gate | Cost | 跑哪个 syscall |
|---|---|---|
| 1 Time | O(1) | 1 stat |
| 1.5 Scan throttle | 0 | 内存比较 |
| 2 Sessions | O(N sessions) | 1 listdir + N stat |
| 3 Lock | O(1) but contention-sensitive | 2 read + 1 mkdir + 1 write + 1 read (5 syscall) |

如果倒过来排(lock first),**每个 user prompt 都会写一次 lock 文件**(即使 time 还没到)→ 浪费 + 干扰其他 process。

**业内 engineering rule of thumb**:_"Filter cheapest first, escalate only when needed."_ Claude Code 注释直接把这个写出来。

### 4.3 为什么有 scan throttle?

如前 §2.2 — 防 _"time gate 长期开,但 session gate 长期闭,导致每个 turn 都跑 listdir 浪费"_ 的退化场景。

这是**实际跑过 production 才会发现的 bug**:没有 throttle,在 _"已经 25h 但 session 不够 5 个"_ 的窗口期,每个 user prompt 都跑 listdir。10 min throttle 把它限制为最多每 10 分钟 1 次。

**Builder lesson**:gate-based scheduler 必须考虑 _"gate 1 过,gate 2 不过"_ 的退化场景。

### 4.4 为什么 sub-agent fork 而非主 loop 跑?

`autoDream.ts:224-233` 用 `runForkedAgent({...})` fork 一个 sub-agent 跑 consolidation:

```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: prompt })],
  cacheSafeParams: createCacheSafeParams(context),    // ⭐ 跟主 loop 共享 prompt cache
  canUseTool: createAutoMemCanUseTool(memoryRoot),    // ⭐ 物理限制只能改 memory
  querySource: 'auto_dream',                          // ⭐ telemetry 区分
  forkLabel: 'auto_dream',                            // ⭐ trace 区分
  skipTranscript: true,                               // ⭐ 不污染主 transcript
  overrides: { abortController },                     // ⭐ 用户可 kill
  onMessage: makeDreamProgressWatcher(taskId, ...),   // ⭐ progress 推 UI
})
```

**7 个 sub-agent 参数都有明确意图**:
- `cacheSafeParams`:跟主 loop 共享 prompt cache(系统 prompt 一致,cache hit)— 跟 [[concept/runtime-context-injection]] §9 prompt cache 协同呼应
- `canUseTool`:物理 tool 隔离(不能改 user code)
- `querySource` + `forkLabel`:telemetry 标识
- `skipTranscript`:不污染主 conversation
- `abortController`:用户从 bg-tasks dialog kill
- `onMessage` watcher:把 progress 推 UI(`makeDreamProgressWatcher` 在 `:281-313` 实现)

**对比 inline 跑**:
- ❌ 污染主 transcript
- ❌ 主 loop 阻塞
- ❌ 用户难 kill(只能 Esc 整个 loop)
- ❌ Tool 安全控制困难(主 loop 已有完整权限)

Sub-agent fork 是干净解。

---

## 5. 失败模式 — 这个机制防了哪些 bug

源码注释 + 实现细节透露的 8 类失败场景:

| 失败模式 | autoDream 怎么防 | 代码位置 |
|---|---|---|
| **2 个 process 同时 consolidate** | Lock with PID check + race detection | `consolidationLock.ts:46-84` |
| **Process crash 后 lock 卡死** | `HOLDER_STALE_MS = 1h` 后 reclaim 即使 PID 还活 | `consolidationLock.ts:18-19, 60-66` |
| **PID reuse 后误以为是同一 process** | PID stale guard + read-back verify | `consolidationLock.ts:74-81` |
| **Fork failure 后 mtime 误以为 success** | `rollbackConsolidationLock(priorMtime)` 推回 mtime + 清空 body | `autoDream.ts:269-270` / `consolidationLock.ts:91-108` |
| **User 主动 kill 后 rollback 重复** | `if (abortController.signal.aborted) return` — DreamTask.kill 已经 rollback,不重复 | `autoDream.ts:262-265` |
| **Gate 1 开 Gate 2 闭 → 每 turn listdir 浪费** | `SESSION_SCAN_INTERVAL_MS = 10min` throttle | `autoDream.ts:54-56, 143-151` |
| **当前 session mtime 永远是新的污染 session count** | `sessionIds.filter(id => id !== currentSession)` | `autoDream.ts:164-165` |
| **Forked agent 改 user code(safety)** | `canUseTool = createAutoMemCanUseTool(memoryRoot)` 物理限制 + Bash 受限 read-only | `autoDream.ts:227, 216-218` |
| **Consolidation 把 stale memory 当 fact** | 4-phase prompt Phase 3 显式要求 _"Delete contradicted facts — if today's investigation disproves an old memory, fix it at the source"_ | `consolidationPrompt.ts:50-51` |
| **Relative date drift("yesterday" 几天后含义变)** | 4-phase prompt Phase 3 要求 _"Convert relative dates to absolute dates so they remain interpretable after time passes"_ | `consolidationPrompt.ts:50` |

**10 类 production 故障 全部被代码注释或 prompt 内容**显式 cover — 这是 _"production-tested"_ 的最直接 signal,远比 README claim 可信。

---

## 6. 横向校准 — 其他 harness 怎么处理 memory consolidation

| Harness | 触发机制 | Consolidation 内容 | Sub-agent fork? | Lock? |
|---|---|---|---|---|
| **Claude Code autoDream** | ⭐ 3-gate scheduler(time + sessions + lock)| 4-phase prompt(orient/gather/consolidate/prune) | ✅(`runForkedAgent` + cacheSafeParams + restricted Bash)| ✅(PID lock + mtime double-duty)|
| **OpenClaw memory dreaming** | 用户触发 / cron(详情见 [[harness/openclaw]]) | 6 维评分晋升 + 梦境日记 subagent | ✅(`dreaming-narrative.ts`)| 未深 verify |
| **Hermes commit_memory_session** | 用户触发 `/commit_memory_session` | 自由 LLM prompt | 未明确 fork | 未明确 lock |
| **AgentScope** | 用户触发 + SummarySchema 强类型压缩 | pydantic 5 字段 schema 化(task_overview / current_state / important_discoveries / next_steps / context_to_preserve) | 未明确 | 未明确 |
| **Codex** | Code Mode V8 内可调 / `/compact` | 3 套 compact + `InitialContextInjection` | 未明确 | 未明确 |
| **mem0/Letta/MemPalace/Cognee** | API call / decorator | 各自不同(LLM-extract / verbatim / function-call / memify 7 pipeline)| ❌(library 风,不 fork)| ❌ |

**关键差异**:
- **Claude Code 是业内唯一显式 3-gate scheduler + lock-as-mtime 工程化 background consolidation**
- 其他 framework 多数是 user-triggered consolidation(没有 _"何时自动跑"_ 的逻辑)
- AgentScope pydantic schema 化 + Claude Code 4-phase prompt 是两条不同 _"约束 LLM 行为"_ 路径,都比 Hermes 自由 prompt 严格

---

## 7. 学习视角 — 写这页让我学到的 3 件事

1. **一个 lock 文件能身兼数职**:mtime 同时是 _"上次时间戳"_ + _"互斥锁信号"_,body 同时是 _"PID guard"_ + _"race detector"_。Production engineering 让单文件承担多语义,代价是注释要清楚(`consolidationLock.ts:1-2`)。Claude Code 做对了。
2. **Gate 排序是 cost-aware**:`stat → listdir → atomic write` 严格升序。这种 "filter cheapest first" 是 production code review 标准模式,但在 agent runtime 里**很多 framework 不上心**(把所有 check 揉一起,每 turn 全跑)。
3. **失败模式必须在 source 注释里显式列出**:`consolidationLock.ts` 18 行注释 + `autoDream.ts` 54 行注释 涵盖了 ext4 birthtime 问题、scan throttle 退化场景、PID reuse guard、race detector 等 _"production 跑过才会发现的边缘 case"_。**判断一个机制是不是 production-tested,最快的信号是看源码注释而非 README claim**。

---

## 引用链路

- 源码:`/Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/services/autoDream/`
  - `autoDream.ts:1-324` 完整 (主流程 + 3 gate + 4 phase prompt 调用 + progress watcher)
  - `config.ts:1-21` `isAutoDreamEnabled` 双源(settings.json + GrowthBook tengu_onyx_plover)
  - `consolidationLock.ts:1-140` 完整(lock-as-mtime + PID guard + rollback)
  - `consolidationPrompt.ts:1-65` 完整 4-phase prompt template
- Harness 入口:[[harness/claude-code]] §self_evolution
- 横向 concept:[[concept/self-evolution]] / [[concept/agent-memory]] / [[concept/runtime-context-injection]] §9 (prompt cache 协同)
- 横向 deep:[[deep/agentscope-state-module]](待写,SummarySchema 强类型对照)
