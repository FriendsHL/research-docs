---
type: source
name: hermes-agent
verification: source-grounded
origin:
  - /Users/youren/myspace/hermes-agent (Hermes Agent 源码仓库)
  - /Users/youren/myspace/research-docs/research/Hermes源码研究/ (12 篇研究笔记,仅作 hypothesis 来源)
ingested_at: 2026-05-15
ingest_mode: B (source-grounded, 所有精确事实可跳 file:line)
contributed_to:
  - "[[harness/hermes-agent]]"
  - "[[concept/context-compaction]]"
  - "[[concept/self-evolution]]"

# B 模式产出 — 实际读过的源码清单(便于未来 re-verify 或 diff)
sources_read:
  docs:
    - README.md (全文)
    - AGENTS.md (全文)
    - SECURITY.md (全文)
  python:
    - "run_agent.py:260-350 (IterationBudget + parallel rules + destructive patterns)"
    - "run_agent.py:864-1190 (AIAgent.__init__ + api_mode detection)"
    - "run_agent.py:8918-9040 (_compress_context, session rotation, hooks 通知)"
    - "run_agent.py: grep 80+ key symbols 位置"
    - "hermes_cli/plugins.py:70-180 (VALID_HOOKS 16 个完整列表)"
    - "agent/context_compressor.py:319-460 (ContextCompressor.__init__)"
    - "agent/context_compressor.py:1380-1415 (anti-thrashing logic)"
    - "agent/memory_manager.py:40-240 (fence regex + StreamingContextScrubber + MemoryManager)"
    - "tools/memory_tool.py:107-220 (MemoryStore + char limits) (grep)"
    - "tools/delegate_tool.py:42,128,1047 (MAX_DEPTH, skip_memory) (grep)"
    - "tools/registry.py:57,77,113,143,226,347 (grep)"
    - "tools/skill_usage.py:171-179, 270-310 (grep)"
    - "tools/skill_manager_tool.py:134,692,708-741 (grep)"
    - "tools/skills_tool.py:674,849 (grep)"
    - "tools/approval.py:108,203,368,922,995 (grep)"
    - "tools/mcp_tool.py:276,1077-1080 (grep)"
    - "agent/redact.py:308,386 (grep)"
    - "hermes_state.py:50-159 (sessions/messages schema + FTS5 triggers)"
    - "tools/budget_config.py:18-20 (DEFAULT_RESULT/TURN/PREVIEW SIZE)"
  directories_listed:
    - "gateway/platforms/ (18+ adapter)"
    - "tools/environments/ (7 backends)"
  notes_cross_referenced:
    - "[[../../Hermes源码研究/00-项目架构总览]]"
    - "[[../../Hermes源码研究/01-Agent主循环]]"
    - "[[../../Hermes源码研究/03-工具系统]]"
    - "[[../../Hermes源码研究/05-Session状态与记忆]]"
    - "[[../../Hermes源码研究/10-Skills系统]]"
---

# Source — Hermes Agent (B 模式 source-grounded)

## 核心论点

Hermes 是 Nous Research 出品的"**自进化 AI Agent**",Python 实现,定位为 *the only agent with a built-in learning loop*。一份 `AIAgent` (`run_agent.py:864`) 同步运行时驱动 7 个入口(CLI / TUI / Gateway / ACP / MCP / Cron / Batch),核心卖点是"经验→skill→记忆"的闭环。

关键设计原则(每条都源码 verified):
1. **多入口共享 `AIAgent`**,平台只做事件标准化和 UI 回调(`run_agent.py:864` 一个类,7 个入口共用)
2. **Prompt cache 是 Iron Law** — `<memory-context>` 只注 api_messages 不进 transcript (`agent/memory_manager.py:176-189`);冻结 system prompt;`StreamingContextScrubber` 防流式跨 chunk 泄漏 (`:65-173`)
3. **Context compaction 工程级** — 5 触发点 + `threshold_percent=0.50, protect_first_n=3, protect_last_n=20, summary_target_ratio=0.20` (`agent/context_compressor.py:379-382`) + 反复 `savings_pct<10` 累计 ineffective count (`:1397-1402`) + 切新 session `parent_session_id` chain (`run_agent.py:8985-9013`) + `commit_memory_session` 在 rotation 前 extract memory (`:8990`)
4. **Self-evolution 是核心卖点** — `skill_manage` 6 actions (`tools/skill_manager_tool.py:708-741`) + `skill_usage.py` 跟踪 use/view/patch/last_used/pinned (`:171-179`) + `_pinned_guard` (`:134-152`) + 仅 agent-created skill 进 sidecar
5. **Trust model 明确**:single-tenant operator(`SECURITY.md` §2),不防多租户

## B 模式发现的事实漂移(分 3 类)

⚠️ **方法论 lesson**:原本以为"笔记错 4 处",B 模式 grep 笔记后发现实际只有 1 类是笔记真错,另外 2 处是 **A 模式 wiki 自己 hallucinate** 的(LLM 自编看似精确的事实),还有 1 处是笔记不全。区分这 3 类很重要 — A 模式不仅传递笔记错,**LLM 还会主动编错**。

| # | 维度 | 类型 | 之前的错误 | 源码实际 | 行号 |
|---|---|---|---|---|---|
| 1 | **MAX_DEPTH** | ⚠️ A 模式 wiki hallucinate | "笔记说 2" — **笔记根本没提 MAX_DEPTH** | `MAX_DEPTH = 1`(parent=0 → child=1) | `tools/delegate_tool.py:128` |
| 2 | **Terminal backends** | ❌ **笔记真错** | 笔记列 6 种(`local/docker/ssh/modal/daytona/singularity`),已修 | **7 种**(新增 `vercel_sandbox`) | `tools/environments/` ls |
| 3 | **`approvals.mode` 取值** | ⚠️ A 模式 wiki hallucinate | "笔记说 on/auto/off" — **笔记没说过 approvals.mode 取值** | on / auto / off / **smart** | `tools/approval.py:995` |
| 4 | **Gateway platforms 数** | ➖ 笔记不全(非错) | 笔记列举"Telegram/Discord/Slack/WhatsApp/飞书/企业微信/邮件/Webhook/API Server 等"但无具体数字 | **18+ adapters** | `gateway/platforms/` ls |

**结论**:
- **真笔记错 1 处**(terminal backends)→ 已在原笔记 `verified_by` 标记并修正
- **A 模式 wiki 自编错 2 处**(MAX_DEPTH / approvals.mode)→ B 模式 wiki 已纠正
- **笔记不全 1 处**(platforms 数)→ B 模式 wiki 补全为 18+

**SkillForge 借鉴的实际只有源码事实**(MAX_DEPTH=1 / approvals.mode 含 smart),不存在"笔记说错"的问题。

## B 模式新发现(笔记没记的事实)

| # | 事实 | 行号 |
|---|---|---|
| 1 | Subagent default budget = **50** (parent=90, 各自独立累计可超 parent cap) | `run_agent.py:266-271` |
| 2 | `execute_code` 通过 `IterationBudget.refund()` 不计入预算 | `run_agent.py:273, 290-294` |
| 3 | `StreamingContextScrubber` 跨 SSE chunk 防 `<memory-context>` 泄漏 | `agent/memory_manager.py:65-173` |
| 4 | `_DESTRUCTIVE_PATTERNS` 显式 regex(rm/rmdir/cp/install/mv/sed -i/truncate/dd/shred/git reset/clean/checkout) | `run_agent.py:338-350` |
| 5 | `MemoryManager` 严格 1+1 限制(builtin 必在 + 仅 1 个 external,重复注册 warning reject) | `agent/memory_manager.py:206-228` |
| 6 | `commit_memory_session(messages)` 在 compaction rotation 前 trigger memory extraction | `run_agent.py:8990` |
| 7 | `summary_target_ratio` clamp 在 `[0.10, 0.80]` | `agent/context_compressor.py:399` |
| 8 | `VALID_HOOKS` 实际包含 `on_session_reset`(笔记漏列) | `hermes_cli/plugins.py:87` |

## 关键事实(精确数字,全部 verified)

| 类别 | 事实 | 来源 |
|---|---|---|
| 语言 / 体量 | Python 3.11,`run_agent.py` 12K+ LOC,`cli.py` 11K+ LOC | AGENTS.md |
| 入口 | CLI / Ink TUI / 18+ messaging platforms / ACP / MCP / Cron / Batch | `gateway/platforms/`, AGENTS.md |
| Terminal backends | 7 种:local / docker / ssh / modal / daytona / singularity / **vercel_sandbox** | `tools/environments/` |
| LLM api_mode | 4 种 `{"chat_completions", "codex_responses", "anthropic_messages", "bedrock_converse"}` | `run_agent.py:1034` |
| Provider 自动检测 | openai-codex / xai / anthropic / bedrock / openrouter 等 | `run_agent.py:1036-1065` |
| `max_iterations` 默认 | 90 (parent) / 50 (subagent via `delegation.max_iterations`) | `run_agent.py:898, 266-271` |
| `_MAX_TOOL_WORKERS` | 8 | `run_agent.py:329` |
| `_PARALLEL_SAFE_TOOLS` | 11 个 read-only tools | `run_agent.py:311-323` |
| `_PATH_SCOPED_TOOLS` | {read_file, write_file, patch} | `run_agent.py:326` |
| `_NEVER_PARALLEL_TOOLS` | {clarify} | `run_agent.py:308` |
| Compaction 默认 | threshold 50%, protect_first 3, protect_last 20, target_ratio 20% (clamped 10-80%) | `agent/context_compressor.py:379-399` |
| 反复压缩保护 | `savings_pct < 10` 累计 `_ineffective_compression_count` | `agent/context_compressor.py:1397-1402` |
| `MAX_DEPTH` (subagent) | **1** | `tools/delegate_tool.py:128` |
| Subagent isolation | `skip_memory=True` + 不能 delegate | `tools/delegate_tool.py:1047, 42` |
| MEMORY/USER 字符上限 | 2200 / 1375 | `tools/memory_tool.py:118` |
| MemoryManager external 限制 | 同时仅 1 个 non-builtin provider | `agent/memory_manager.py:206-228` |
| `VALID_HOOKS` | **16 个** | `hermes_cli/plugins.py:75-111` |
| `approvals.mode` | on(默认)/ auto / off / smart | `tools/approval.py:995` |
| Tool result budgets | `DEFAULT_RESULT_SIZE=100K, DEFAULT_TURN_BUDGET=200K, DEFAULT_PREVIEW=1.5K` chars | `tools/budget_config.py:18-20` |
| `_CHECK_FN_TTL_SECONDS` | 30.0 (tool availability check 缓存 TTL) | `tools/registry.py:113` |
| External memory providers | 8 个(honcho / mem0 / supermemory / byterover / hindsight / holographic / openviking / retaindb) | AGENTS.md + `plugins/memory/` |
| FTS5 索引 | `messages_fts` (English) + `messages_fts_trigram` (CJK ≥3 chars) | `hermes_state.py:104, 133` |
| Profile 隔离 | `HERMES_HOME=root/profiles/<name>`, `_apply_profile_override()` 最早执行 | AGENTS.md + `hermes_cli/main.py` |

## 与现有 wiki 的关系

- **印证**:无(A 模式版本已 rollback)
- **矛盾**:旧 A 版 wiki 的 `MAX_DEPTH=2` 错的,本版 verified 为 1
- **新增**:本 ingest 是新 baseline,所有事实可跳源码,后续 harness ingest 直接对照本页 13 维 schema 实例
