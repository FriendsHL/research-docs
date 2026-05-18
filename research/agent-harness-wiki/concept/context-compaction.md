---
type: concept
name: context-compaction
status: draft
verification: source-grounded
updated: 2026-05-16
sources:
  - "[[../source/hermes-agent]]"
  - "[[../source/claude-code]]"
  - "[[../source/codex]]"
  - "[[../source/opencode]]"
  - "/Users/youren/myspace/hermes-agent/agent/context_compressor.py"
  - "/Users/youren/myspace/hermes-agent/run_agent.py"
  - "/Users/youren/myspace/codex/codex-rs/core/src/compact.rs"
  - "/Users/youren/myspace/opencode/packages/opencode/src/session/compaction.ts"
related_harnesses:
  - "[[../harness/hermes-agent]]"
  - "[[../harness/claude-code]]"
  - "[[../harness/codex]]"
  - "[[../harness/opencode]]"
---

# Context Compaction(上下文压缩)

## 定义

当 agent 一次对话的 messages 总长度逼近模型 context window 时,**用辅助模型把中间历史摘要成结构化交接信息,保留头尾原文 + 替换中间**,使后续对话可以继续而不被 provider 因 context overflow 拒绝。

> 不是"删除旧消息",不是"截断 tool result",也不是"重启对话"。它是有结构、有边界保护、有持久化协议的工程。

---

## 设计空间(7 维)

| 维度 | 选项 |
|---|---|
| **触发时机** | preflight(进 loop 前预估)/ context_overflow 后被动 / tool 执行后(防大输出爆炸)/ 手动(`/compress <focus>`)/ Gateway hygiene(消息平台外层) |
| **压什么** | 全部历史 / 只 middle(head+tail 保护)/ 只 tool result / 只重复内容 / focus-aware |
| **压成什么** | 自由文本摘要 / 结构化字段摘要(Active Task / Completed / Files / Critical Context) / 编程式裁剪(一行 placeholder 替换)/ 重复去重 / 长 JSON 安全截断 |
| **怎么接回** | 替换 transcript in-place / **切新 session(parent_session_id chain)** / 标记"压缩边界"插条特殊 message / 重建 system prompt |
| **边界保护** | tool_call/tool_result 配对修复(孤儿 result 删 / 孤儿 call 补 stub)/ 不切割 tool batch / persistence-shape 字节一致 / 不破 prompt cache |
| **质量保护** | 反复压缩节省 <X% 倾向停止 / 摘要失败 fallback marker / 多次压缩警告 / focus topic 优先保留 |
| **下游通知** | `memory_provider.on_pre_compress(messages)`(让 memory 抢救重要事实)/ `context_engine.on_session_start(boundary_reason="compression")` / `memory_provider.on_session_switch()` / todo snapshot 重新追加 |

---

## 各 harness 怎么实现

### [[../harness/hermes-agent]](Python,源码 verified)

**触发** — 5 个点:
- Preflight(turn 前估算)
- API context_overflow 后被动
- Tool 执行后(基于真实 prompt_tokens 或 rough estimate)
- 手动 `/compress <focus>`,`focus_topic` 透传到 summary prompt(`run_agent.py:8922-8924`,注释明确"Inspired by Claude Code's `/compact <focus>`")
- Gateway hygiene 外层

**Phase A — 廉价预处理**(`agent/context_compressor.py:491` `_prune_old_tool_results`):
1. 旧 tool result >200 chars 改一行摘要(`[terminal] ran X -> exit N, M lines` / `[read_file] read X.py from line N (M chars)` 等)
2. 重复 tool result 去重 → `[Duplicate tool output — same content as a more recent call]`
3. 长 `tool_call.arguments` JSON 安全截断(解析 JSON → 截内层字符串值 → 重序列化,避免字符串硬切产生坏 JSON 致 provider 400)

**Phase B — 结构化摘要**(`agent/context_compressor.py:1235` `compress`):
4. `_serialize_for_summary` 单条限 ~6000 chars(前 4000 + `[truncated]` + 后 1500)(`:656`)
5. head/middle/tail 切:`protect_first_n=3` (`:380`),tail 按 token budget(`tail_token_budget = threshold_tokens * 0.20`, `:418`)
6. `_generate_summary` 辅助模型生成结构化 summary(Active Task / Completed Actions / Relevant Files / Critical Context / Remaining Work,`:711`)
7. `_sanitize_tool_pairs(compressed)` 修复孤儿 tool_call/tool_result 配对(`:1391`)

**默认参数**(`agent/context_compressor.py:379-399`):
- `threshold_percent = 0.50`
- `protect_first_n = 3`
- `protect_last_n = 20`
- `summary_target_ratio = 0.20`(clamp `max(0.10, min(_, 0.80))`,`:399`)
- `threshold_tokens = max(ctx * 0.50, MINIMUM_CONTEXT_LENGTH)`(`:411`)
- `max_summary_tokens = min(ctx * 0.05, _SUMMARY_TOKENS_CEILING)`(`:420`)

**反复压缩保护**(`agent/context_compressor.py:1397-1402`):
```python
savings_pct = (saved_estimate / display_tokens * 100) if display_tokens > 0 else 0
self._last_compression_savings_pct = savings_pct
if savings_pct < 10:
    self._ineffective_compression_count += 1
else:
    self._ineffective_compression_count = 0
```
当 `savings_pct < 10` 累计若干次,note "Compression skipped — last N compressions saved <10% each. ..."(`:479`)。

**接回**(`run_agent.py:8918-9040` `_compress_context`):
- L8940: `memory_manager.on_pre_compress(messages)` — 通知 external memory 抢救重要事实
- L8945: `context_compressor.compress(messages, ...)`
- L8977-8979: 追加 `todo_snapshot` 到 compressed messages
- L8981-8983: invalidate + rebuild + cache system prompt
- L8990: **`commit_memory_session(messages)` 在 rotation 前 trigger memory extraction**(关键!)
- L8991: `end_session(old_id, "compression")`
- L8993: 新 session_id = `{YYYYMMDD_HHMMSS}_{uuid hex 6}`
- L8996-9001: 新 session `parent_session_id = old_session_id`,继承 source/model
- L9005-9006: title auto-numbering(continuation session)
- L9009: `update_system_prompt(new_session_id, new_system_prompt)`
- L9011: `_last_flushed_db_idx = 0` reset flush cursor(避免按旧 history 长度跳过前面消息)
- L9022-9028: `context_engine.on_session_start(boundary_reason="compression", old_session_id=...)`
- L9036+: `memory_provider.on_session_switch(reason="compression")` — provider 缓存 per-session 状态刷新

**质量保护**:
- 摘要模型失败 → 插 fallback marker + `_emit_warning("⚠ Compression summary failed: ...")`(`run_agent.py:8955`)
- 配置的 aux summary model 失败但 main model 恢复成功 → 仍然 warning,避免悄悄 fallback(`run_agent.py:8966-8975`)

### [[../harness/claude-code]](TypeScript,源码 verified)

**5 级 pipeline** (`services/compact/` 11 files,跟 Hermes 单 ContextCompressor 完全不同的分层架构):

| 级别 | 文件 | 干什么 |
|---|---|---|
| 1. budget | `query/tokenBudget.ts` + `feature('TOKEN_BUDGET')` Bun gate | token 预算追踪 + `taskBudgetRemaining` 跨 boundary (`query.ts:282-292`) |
| 2. snip | `tools/SnipTool/` | 历史 snippet 提取,模型主动用 |
| 3. microcompact | `services/compact/microCompact.ts` + `apiMicrocompact.ts` + `timeBasedMCConfig.ts`(GrowthBook 时间触发)| 频繁小裁剪,主循环间隔超阈值触发 |
| 4. collapse | `utils/collapseReadSearch.ts` + `utils/collapseHookSummaries.ts` | 折叠 read/search 工具结果摘要 + hook 摘要 |
| 5. autocompact | `services/compact/autoCompact.ts` | 完整 compaction(类似 Hermes ContextCompressor)|

**默认阈值常量**(`services/compact/autoCompact.ts:62-71`):
- `AUTOCOMPACT_BUFFER_TOKENS = 13_000`(autocompact 在 ctx 还剩 13K 时触发)
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20_000`(UI warning)
- `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`
- `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` + 注释:**BQ 2026-03-10:1,279 sessions × 50+ 连续失败 = ~250K wasted API calls/day**
- env override: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`
- threshold = `effectiveContextWindow - 13_000` (`autoCompact.ts:103-105`)

**主 compact 函数**(`services/compact/compact.ts`,1568 行):
- `compactConversation` (`:387`) — 主入口
- `partialCompactConversation` (`:772`)
- `truncateHeadForPTLRetry` (`:243`) — **reactive 413 recovery**(API 返回 413 后被动压缩)
- `mergeHookInstructions` (`:374`) — hook 可注入 compaction 摘要指令
- `annotateBoundaryWithPreservedSegment` (`:349`) — suffix-preserving boundary marker(reactive / session-memory paths)
- `buildPostCompactMessages` (`:330`)
- `createCompactCanUseTool` (`:1125`) — Fork agent 跑 compaction 时的 canUseTool
- `createPostCompactFileAttachments` (`:1415`) + `createPlanAttachmentIfNeeded` / `createSkillAttachmentIfNeeded` / `createPlanModeAttachmentIfNeeded` / `createAsyncAgentAttachmentsIfNeeded`

**切新 session vs in-place**:Claude Code 用 **in-place + boundary annotation**(`annotateBoundaryWithPreservedSegment`),不切新 session — 跟 Hermes `parent_session_id` chain 的设计相反。这意味着 Claude Code:
- 用 `mergeHookInstructions` 让 PreCompact / PostCompact hook 可观察(zod-validated,`coreTypes.ts:37-38`)
- Reactive recovery 直接在原 transcript 上做(`truncateHeadForPTLRetry`)
- 跨 compaction boundary 用 `taskBudgetRemaining` 计 server 看不到的预算

**`/compact <focus>`**:Hermes 注释明确说"inspired by Claude Code's `/compact <focus>`"(`run_agent.py:8924`)— 验证了 Claude Code 是源头。详细 focus_topic 透传逻辑笔记 06 提及未深 verify。

### [[../harness/openclaw]](待 ingest)

_(占位,Hermes 的"前身",compaction 设计应高度相似)_

### [[../harness/codex]](Rust,源码 verified)

**3 套并存的 compact 实现**(同 codebase 内共存,这是 4 家里独家):
- `core/src/compact.rs` (585 行,local inline 实现)
- `core/src/compact_remote.rs` (387 行,远端 compaction 旧版)
- `core/src/compact_remote_v2.rs` (456 行,远端 compaction 新版)

**路由策略**(`compact.rs:65-67`):
```rust
pub(crate) fn should_use_remote_compact_task(provider: &ModelProviderInfo) -> bool {
    provider.supports_remote_compaction()
}
```
按 provider 是否支持 server-side compaction 路由 local/remote — OpenAI Responses API 走 remote,其他 provider 走 local。**把 provider capability 当成路由依据**,而不是 if-else 散落 — 工程化漂亮。

**注入位置区分**(`compact.rs:59-63`,4 家里独家):
```rust
pub(crate) enum InitialContextInjection {
    BeforeLastUserMessage,  // mid-turn compact 用
    DoNotInject,            // pre-turn / manual compact 用
}
```
注释明确(`compact.rs:54-58`):
> "Mid-turn compaction must use `BeforeLastUserMessage` because **the model is trained to see the compaction summary as the last item in history after mid-turn compaction**; we therefore inject initial context into the replacement history just above the last real user message."
> "Pre-turn/manual compaction variants use `DoNotInject`: they replace history with a summary and clear `reference_context_item`, so the next regular turn will fully reinject initial context after compaction."

**把"模型训练时见到的位置"作为代码级硬约束**,而不是埋在 prompt template 里 — 这是行业其他 3 家都没有的精细对齐。

**关键常量**(`compact.rs:46-48`):
- `SUMMARIZATION_PROMPT = include_str!("../templates/compact/prompt.md")`
- `SUMMARY_PREFIX = include_str!("../templates/compact/summary_prefix.md")`
- `COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000`

**Telemetry 完整**(`compact.rs:17-23`)— 7 个枚举:`CodexCompactionEvent` / `CompactionImplementation` / `Phase` / `Reason` / `Status` / `Strategy` / `Trigger`,**业内最细 telemetry**。每次 compact 都打全维度事件,可观测性是 4 家里最好的。

**Hook 协作**:`core/src/hook_runtime.rs` 暴露 `run_pre_compact_hooks` + `run_post_compact_hooks`,跟 hooks 8 个事件中的 `PreCompact` / `PostCompact`(`hooks/src/lib.rs:22-24`)对齐。

**7 维位置**:
- **触发时机**:turn 内被动 + 手动 + provider 决定 local/remote
- **压什么**:全部历史(不像 Hermes 有 Phase A 廉价预处理)
- **压成什么**:辅助模型自由摘要(`templates/compact/prompt.md`)+ 强制前缀(`summary_prefix.md`)
- **怎么接回**:in-place 替换 + **训练对齐的注入位置**(BeforeLastUserMessage / DoNotInject)— 4 家里独家
- **边界保护**:`reference_context_item` 在 manual 模式 clear,让下一 turn 重新 inject — 跟 Hermes 切新 session 思路相似,但走 in-place
- **质量保护**:未发现 "<X% 倾向停止" 等价物(Hermes 有 `_ineffective_compression_count`)
- **下游通知**:8 lifecycle hook 的 `PreCompact` / `PostCompact` 给 plugin 干预的机会,plugin 可在 PostCompact 做 memory extraction

### [[../harness/opencode]](TypeScript / Bun / Effect-TS,源码 verified)— 2026-05-16 新加

**关键常量**(`packages/opencode/src/session/compaction.ts:42-48`):
- `PRUNE_MINIMUM = 20_000` tokens(触发 prune 下限)
- `PRUNE_PROTECT = 40_000` tokens(保护区间)
- `TOOL_OUTPUT_MAX_CHARS = 2_000`(单 tool output 字符上限,跟 Codex `compact_remote.rs` 相近)
- **`PRUNE_PROTECTED_TOOLS = ["skill"]`** — **业内独家**:Skill 调用历史是 first-class memory,绝对不被 prune
- `DEFAULT_TAIL_TURNS = 2`(默认保留尾部 turn 数)
- `MIN_PRESERVE_RECENT_TOKENS = 2_000` / `MAX_PRESERVE_RECENT_TOKENS = 8_000`(保留尾部 token 区间)

**`SUMMARY_TEMPLATE` 7 段强制 Markdown 结构**(`compaction.ts:50-87`)— 比 Hermes Phase B 5 段更细:
- **Goal**(single-sentence task summary)
- **Constraints & Preferences**(用户约束 / 偏好 / 规格)
- **Progress**(Done / In Progress / Blocked 三 sub-section)
- **Key Decisions**(decision and why)
- **Next Steps**(ordered next actions)
- **Critical Context**(重要技术事实 / 错误 / 开放问题)
- **Relevant Files**(file or directory path: why it matters)

**4 条 SUMMARY 规则**(`compaction.ts:88-92`):
1. "Keep every section, even when empty."(防 LLM 跳 section)
2. "Use terse bullets, not prose paragraphs."(防 LLM 写散文)
3. "Preserve exact file paths, commands, error strings, and identifiers when known."(防 LLM 改 identifier)
4. **"Do not mention the summary process or that context was compacted."**(防 LLM 在 summary 里说"我刚被压缩了" — **业内独家工程细节**)

**Bus 事件驱动**(`compaction.ts:31-37`):`BusEvent.define("session.compacted", {sessionID})` — compaction 是事件总线发的事件,所有 listener 可订阅

**`overflow.ts`** — `isOverflow` + `usable` 判定 context 满

**`projectors.ts` + `projectors-next.ts`** — **CQRS / event sourcing 风格状态投影器**,事件流 → 投影 → 当前状态(类 [[../eval/langfuse]] Wide Events 哲学)

**7 维位置**:
- **触发时机**:`isOverflow` 判定 + `PRUNE_MINIMUM=20_000` 下限触发
- **压什么**:全部历史(skill 工具输出受 `PRUNE_PROTECTED_TOOLS = ["skill"]` 保护)
- **压成什么**:7 段强制 Markdown summary(由 `SUMMARY_TEMPLATE` + 4 条规则约束)
- **怎么接回**:in-place 替换 + 保留尾部 `DEFAULT_TAIL_TURNS=2` + `MIN/MAX_PRESERVE_RECENT_TOKENS`
- **边界保护**:Schema-first(Effect-TS Schema 强制 IO boundary)+ skill tool 输出绝对保留
- **质量保护**:**"Do not mention summary process"** prompt 级硬约束 + Schema validate
- **下游通知**:`BusEvent("session.compacted")` 事件总线全员通知

### SkillForge(自身)

来自 `/Users/youren/myspace/skillforge/CLAUDE.md` + git 提交历史:
- **Light compact + Full compact** 两档(`CompactionService` + `SessionMemoryCompactStrategy`)
- **`persistence-shape-invariant` Iron Law**:`ChatService` 持久化 Message 跟 Engine 内存 messages list 同位置 message JSON 字节必须一致(rule `persistence-shape-invariant.md`,来源 Q2 bdb0453 反例 + Q3 cc87776 修 + b2c7039 guard)
- **`identity-column-on-rewrite` rule**:加 `t_session_message` identity 列必扩 `snapshotXByseqNo + patchX` preserve(来源 Q1 a4100f7 trace_id wipe regression)
- **`compact-reviewer` agent** 系统提示内嵌 8 条 compact 不变量(INV-1 ~ INV-8)
- Light compact 有 index-alignment limitation(已知)

**SkillForge ↔ Hermes 对照**:

| 设计点 | Hermes | SkillForge | 状态 |
|---|---|---|---|
| 摘要 + head/tail 保护 | Phase B + protect_first_n=3, protect_last_n=20 | Full compact 同模式 | ✅ 同源 |
| tool_call/result 配对修复 | `_sanitize_tool_pairs` (`:1391`) | INV-1 enforce | ✅ 同源 |
| persistence shape 一致 | implicit(SQLite row 即 message) | 显式 invariant + reviewer agent | ⚠️ SkillForge 更严格 |
| identity 列保留 | 无 identity 列概念(SQLite rowid 即主键) | 显式 invariant + 4 步 preserve | ⚠️ SkillForge 独有 |
| 切新 session | `parent_session_id` chain | ? 需对照 | ❓ |
| 反复压缩 <10% 保护 | `_ineffective_compression_count` | ? | ❓ |
| Memory extraction 在 rotation 前 | `commit_memory_session` (`:8990`) | ? | ❓ |
| Focus topic 透传 | `/compress <focus>` | ? `/compact` 是否接受 focus? | ❓ |
| 摘要失败 fallback | static fallback marker + warning | ? | ❓ |

---

## Trade-off

### 替换 transcript in-place vs 切新 session

| | in-place | 切新 session(parent chain) |
|---|---|---|
| 实现 | 简单,messages array 整段 replace | 复杂,需 session id 父子关系 + 跨 session resume |
| 历史可查 | 丢失 | 保留(旧 session `/resume` 仍可) |
| 数据库 schema | 不需要 `parent_session_id` | 需要 |
| 用户感知 | 透明 | UI 需 projection 隐藏压缩链 |
| **Hermes** | ❌ | ✅ |
| **SkillForge** | ?(需对照 `CompactionService` 实际) | ? |

### 反复压缩保护

不加保护 → 长会话反复触发,每次节省递减但成本不变 → "压了也省不了多少,但每轮都在压"的死循环。
**Hermes**:最近若干次 `savings_pct < 10` 倾向不再自动压,note 提示用户 `/new` 或 `/compress <focus>`(`agent/context_compressor.py:1397-1402, 479`)

### 主动 focus topic

`/compress agent loop` → 摘要时优先保留 agent loop 相关内容。
**为什么有价值**:压缩本质是有损,用户知道下一步要做什么,焦点信息应保留。Hermes 注释明确这是抄 Claude Code 的(`run_agent.py:8924`)。

### Memory extraction 时机

如果在压缩**后**才让 memory provider 提取重要事实,中间历史已被摘要丢失原文。
**Hermes 的选择**:`commit_memory_session(messages)` 在 `end_session` **前**触发(`run_agent.py:8990`),保证 memory provider 看得到原始未压缩的 messages。

---

## SkillForge 当前选择 + 演进方向

**当前已知**(从 CLAUDE.md):
- ✅ Light + Full 两档 compact
- ✅ `persistence-shape-invariant` Iron Law(超出 Hermes 范围,SkillForge 更严格)
- ✅ tool_call/result 配对维护(INV-1)
- ✅ `identity-column-on-rewrite` 模式(SkillForge 独有)
- ✅ `compact-reviewer` agent 强制审核
- ❓ 切新 session(parent_session_id)还是 in-place?
- ❓ 反复压缩 <10% 保护?
- ❓ Memory extraction 在 rotation 前的 hook?
- ❓ `/compact` 是否接受 focus topic?

**建议演进**(基于 Hermes verified 实现):
1. **借鉴反复压缩 <10% 保护**(`agent/context_compressor.py:1397-1402`)— 避免长会话压缩死循环
2. **借鉴 `commit_memory_session` 在 rotation 前 trigger memory extraction**(`run_agent.py:8990`)— 否则压缩可能丢重要事实
3. **`/compact` 接受 `<focus>` 参数** — 压缩有损,焦点应让用户/agent 决定保留什么
4. **`memory_provider.on_pre_compress` hook**(`agent/memory_provider.py:203`)— 压缩前给 memory provider 一个抢救重要事实的 last chance
5. **摘要失败 fallback marker + warning emit** — 避免悄悄 fallback 把问题藏起来
6. **`tail_token_budget` 而非 `protect_last_n`**(`agent/context_compressor.py:418`)— 按 token 而非按消息数保护 tail,因为消息长度方差大

---

## 后续 lint / ingest 触发

- 待 ingest [[../harness/openclaw]](Hermes 前身)
- 待 ingest [[../harness/aider]]:Aider 用 SEARCH/REPLACE 不走 OpenAI tool_use,compaction 形态可能完全不同
- 待 ingest [[../harness/cline]](VSCode 插件 agent):IDE 场景 compaction 触发点是否更频繁?
- SkillForge `CompactionService` 实际代码 ingest:确认 ? 标记的 unknown

---

## 学习视角:理解 context compaction 设计空间

### 这个概念存在的根本原因(motivation)

**LLM context window 是有限的物理资源,但对话/任务可以是无限增长的逻辑实体。**
Compaction 是这个矛盾的工程化解决方案 —— 不是"减少使用",而是"在保留有效信息的前提下,
把无界对话压缩到有界 window 里"。

**类比**:人脑工作记忆只有 ~7 个 chunks,但人类一天能处理上千条信息 —— 靠的是:
- **遗忘**(放弃不重要的)
- **抽象**(把多条具体经验合并成一条规律)
- **外化**(写笔记/相册)

Compaction 三种策略恰好对应:**裁剪** = 遗忘,**摘要** = 抽象,**切新 session(`parent_session_id`)** = 外化(旧 session 仍可 resume 查)。

### 哪些 harness 做得好(what_done_well across harnesses)

| 设计点 | 谁做得好 | 为什么 elegant |
|---|---|---|
| **多触发点 cascade**(preflight / overflow / tool-后 / 手动 / Gateway hygiene) | Hermes(5 触发点) | 不是单一 timing,而是"在多个时机检测"—— **早压晚压都不浪费机会** |
| **分层 pipeline**(轻预处理 → 重摘要) | Claude Code(5 级)+ Hermes(Phase A 预处理 + Phase B 摘要) | **不是"all or nothing"** —— 廉价 cleanup 先做,昂贵摘要后做 |
| **`memory_provider.on_pre_compress` hook** | Hermes(`agent/memory_provider.py:203`) | 压缩前给外部系统 last chance 抢救事实 —— **时序边界的 elegant 处理** |
| **`commit_memory_session` 在 rotation 前 trigger** | Hermes(`run_agent.py:8990`) | 同上,但更具体 —— "保 session 完整性"前先"抢救核心事实" |
| **反复压缩 <10% saving 保护** | Hermes(`context_compressor.py:1399`)+ Claude Code `MAX_CONSEC_FAILURES=3` | 防 "压了也省不了多少但每轮都在压"的死循环 —— **anti-thrashing 是工程系统的标志** |
| **tool_call/tool_result 配对修复** | Hermes `_sanitize_tool_pairs` (`:1391`)+ Claude Code `annotateBoundaryWithPreservedSegment` (`:349`) | OpenAI tool calling 格式严格,压缩切边界容易破格式 —— **修复孤儿配对是 production 级细节** |
| **`/compact <focus>` 接受 focus topic** | Claude Code 首创(Hermes 抄)| 压缩有损,**让用户/agent 决定焦点**比"通用摘要"保留更多有用信息 |
| **真实生产数据驱动设计** | Claude Code `MAX_CONSEC_FAILURES = 3` 注释 BQ 数据 | "1,279 sessions × 50+ failures = 250K wasted API calls/day" —— **决策有数据,不靠拍脑袋** |

### 学习路径(从入门到深入)

1. **入门**:看 Hermes `agent/context_compressor.py:319-460`(`__init__` 默认参数)—— 理解
   threshold / protect_first / protect_last / target_ratio 4 个关键参数
2. **进阶**:看 Hermes `_compress_context` (`run_agent.py:8918-9040`)—— 理解压缩跟 session
   rotation / system prompt rebuild / memory provider 通知的**完整集成**
3. **对比**:看 Claude Code `services/compact/autoCompact.ts:62-71`(阈值常量)+ 跟 Hermes
   对照 —— 理解"buffer-based threshold" vs "ratio-based threshold"两种风格
4. **深入**:看 Claude Code `services/compact/compact.ts:243` `truncateHeadForPTLRetry` ——
   理解 reactive 413 recovery 的 fallback 路径
5. **高阶**:对比 Hermes "切新 session" vs Claude Code "in-place + boundary annotation"
   —— 理解两种"保留历史可查"的工程选择

### 反模式 / 常见错误

- ❌ **简单删除最旧消息** —— 看似简单,但会丢失任务背景,模型继续工作时困惑
- ❌ **只压缩 tool result 不压缩对话** —— 长对话场景压缩不够
- ❌ **压缩中途修改 system prompt** —— 破坏 prompt cache,成本陡增
- ❌ **不修复 tool_call/tool_result 配对** —— provider 直接返回 400
- ❌ **不限制反复压缩** —— "压了也省不了多少但还在压"的死循环
