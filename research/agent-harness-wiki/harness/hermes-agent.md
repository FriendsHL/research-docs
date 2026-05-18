---
type: harness
name: hermes-agent
status: active
language: python
verification: source-grounded
ingest_mode: B
updated: 2026-05-15

# 版本时效(2026-05-16 补:schema §3.1 升级后回填)
last_ingested_version: "0.11.0 @ commit 9a1454060 (2026-04-30)"
last_ingested_at: 2026-05-15
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/hermes-agent]]"
  - "/Users/youren/myspace/hermes-agent/"
  - "[[../../Hermes源码研究/00-项目架构总览]]"
  - "[[../../Hermes源码研究/01-Agent主循环]]"
  - "[[../../Hermes源码研究/03-工具系统]]"
  - "[[../../Hermes源码研究/05-Session状态与记忆]]"
  - "[[../../Hermes源码研究/10-Skills系统]]"

agent_loop: |
  [verified-from-source]
  同步 while loop in `run_conversation()` (`run_agent.py:10135`),主循环 (`run_agent.py:10501`)。
  `class AIAgent` (`run_agent.py:864`); `__init__` (`run_agent.py:887`)。
  Budget: `max_iterations=90` 默认 (`run_agent.py:898`); `IterationBudget` 每 turn 重建
  (`run_agent.py:10233`); `consume()/refund()/remaining` 线程安全 (`run_agent.py:282-303`)。
  `_budget_grace_call` 允许预算耗尽后再说一次 (`run_agent.py:1235, 10520`)。
  耗尽兜底: `_handle_max_iterations()` 追加 user msg 让模型总结 (`run_agent.py:9951`)。
  9 阶段责任聚合在 `run_conversation()`: cleanup → messages prepare → system prompt cache →
  preflight compress → pre_llm_call+memory prefetch → 主 while → tool exec/compress →
  max-iter summary → persist + hooks + return。

tool_protocol: |
  [verified-from-source]
  `ToolRegistry` (`tools/registry.py:143`), `ToolEntry` (`tools/registry.py:77`)。
  自动发现 `discover_builtin_tools` AST 扫顶层 `register()` 调用 (`tools/registry.py:57`)。
  `register()` (`tools/registry.py:226`) / `dispatch()` (`tools/registry.py:347`)。
  `_CHECK_FN_TTL_SECONDS = 30.0` 工具可用性缓存 (`tools/registry.py:113`)。
  并发执行规则 (`run_agent.py:308-329`):
    `_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})` (line 308)
    `_PARALLEL_SAFE_TOOLS` 11 个白名单 (line 311-323):
      ha_get_state / ha_list_entities / ha_list_services /
      read_file / search_files / session_search /
      skill_view / skills_list / vision_analyze /
      web_extract / web_search
    `_PATH_SCOPED_TOOLS = frozenset({"read_file","write_file","patch"})` (line 326) —
      只在路径不重叠时可并行
    `_MAX_TOOL_WORKERS = 8` (line 329)
  Schema 部分动态: execute_code/Discord/browser_navigate 按当前可用工具重建。
  Agent-level tools `{"todo","memory","session_search","delegate_task"}` 需父 agent 状态,
  不能走 `registry.dispatch()`。

context_mgmt: |
  [verified-from-source]
  `ContextCompressor` (`agent/context_compressor.py:319`)。
  默认参数 (`agent/context_compressor.py:379-382`):
    `threshold_percent=0.50` → `threshold_tokens = max(ctx*0.50, MINIMUM_CONTEXT_LENGTH)` (line 411)
    `protect_first_n=3` (line 380)
    `protect_last_n=20` (line 381)
    `summary_target_ratio=0.20`, clamped `max(0.10, min(_, 0.80))` (line 399)
    `tail_token_budget = threshold_tokens * 0.20` (line 418)
    `max_summary_tokens = min(ctx*0.05, _SUMMARY_TOKENS_CEILING)` (line 420)

  5 触发点:
    1. preflight (turn 前估算超阈值) — `run_conversation()` 入口
    2. API context_overflow 后被动重试 — retry loop
    3. tool 执行后(基于真实 prompt tokens 或 rough estimate)
    4. 手动 `/compress <focus>` — `focus_topic` 透传到 summary prompt
    5. Gateway hygiene — 外层 message 处理时

  压缩流程(分两阶段):
    Phase A — 廉价预处理 (`_prune_old_tool_results` `agent/context_compressor.py:491`):
      a) 旧 tool result >200 chars 改一行摘要 (`[terminal] ran X -> exit N, M lines`)
      b) 重复 tool result 去重(`[Duplicate tool output ...]`)
      c) 长 `tool_call.arguments` JSON 安全截断(解析 → 截内层字符串值 → 重序列化)
    Phase B — 结构化摘要 (`compress` `:1235`):
      d) `_serialize_for_summary` 单条限 ~6000 chars (前 4000 + [truncated] + 后 1500) (`:656`)
      e) head/middle/tail 切 (protect_first_n=3, tail by token budget)
      f) `_generate_summary` 辅助模型 (`:711`)
      g) tool_call/tool_result 配对修复:孤儿 result 删,孤儿 call 补
         `[Result from earlier conversation — see context summary above]`
      h) `_sanitize_tool_pairs(compressed)` (`:1391`)

  反复压缩保护 (`agent/context_compressor.py:1397-1402`):
    `savings_pct = saved_estimate / display_tokens * 100`
    if `savings_pct < 10`: `_ineffective_compression_count += 1`
    else reset
    note "Compression skipped — last N compressions saved <10% each. ..." (`:479`)

  切新 session (`run_agent.py:8985-9013`):
    `commit_memory_session(messages)` 触发 memory extraction (`:8990`)
    `end_session(old_id, "compression")` (`:8991`)
    新 session_id = `{timestamp}_{uuid hex 6}` (`:8993`)
    新 session `parent_session_id = old_session_id` (`:8996-9001`)
    `update_system_prompt(new_session_id, new_system_prompt)` (`:9009`)
    `_last_flushed_db_idx = 0` reset flush cursor (`:9011`)
    通知 `context_engine.on_session_start(boundary_reason="compression")` (`:9022`)
    通知 `memory_provider.on_session_switch()` (`:9036+`)
  详见 [[../concept/context-compaction]]。

memory: |
  [verified-from-source]
  三层架构:

  ① **SessionDB** (`hermes_state.py:159`) — SQLite + FTS5
    sessions 表 (`hermes_state.py:50-71`): id / source / user_id / model / system_prompt /
      `parent_session_id` (FK to sessions(id), `:71`) / started_at / ended_at / end_reason /
      message_count / tool_call_count / input/output_tokens / title / api_call_count
    messages 表: role / content / tool_call_id / tool_calls / tool_name /
      reasoning / reasoning_details / codex_reasoning_items / codex_message_items
    FTS5 索引 (`hermes_state.py:104, 133`):
      `messages_fts` (English unicode61 tokenizer)
      `messages_fts_trigram` (CJK substring, ≥3 chars)
      短中文 (<3 chars) fallback LIKE
    3 个 trigger insert/delete/update 同步 FTS

  ② **Built-in memory** (`tools/memory_tool.py:107` `MemoryStore`)
    `memory_char_limit=2200, user_char_limit=1375` (`tools/memory_tool.py:118`)
    MEMORY.md = agent 自己笔记;USER.md = 用户画像
    System prompt 里 memory 是 **session 启动时的冻结快照** — 中途写不改 system prompt
    (保 prompt cache);下个 session 才看到新内容
    写入:加锁 → 读最新 → 去重 → char 上限检查 → atomic 写回
    prompt injection scan / exfiltration 风险扫描

  ③ **External memory provider** (`agent/memory_manager.py:192` `MemoryManager`)
    8 个 plugin: honcho / mem0 / supermemory / byterover / hindsight / holographic /
      openviking / retaindb
    **严格 1+1 限制** (`agent/memory_manager.py:206-228`):
      builtin 永在 + 仅 1 个 external;重复 external 注册被拒绝 + warning log
    Hooks: `prefetch`/`sync_turn` (`agent/memory_provider.py:93,115`) /
      `on_pre_compress` (`:203`) / `on_memory_write` (`:263`)
    Failure 模式 fail-open:provider 异常 warning 但不阻塞 agent
    Prefetch 整 turn 复用 cache,不多次调用 (`run_agent.py` 主循环前)

  ④ **`<memory-context>` 防污染** (`agent/memory_manager.py:46-189`)
    `build_memory_context_block(raw_context)` 包成块 (`:176-189`):
      `<memory-context>` + System note + content + `</memory-context>`
    `sanitize_context()` 一次性 strip (`:57-62`)
    `StreamingContextScrubber` 状态机跨 chunk 防泄漏 (`:65-173`)
      _OPEN_TAG / _CLOSE_TAG (`:91-92`)
      feed() 边界处理 + flush() 末尾清理 + reset() 重置
    Iron Law: **只注 api_messages,不写 messages 原始历史,不持久化 DB**
    防止 resume 时 memory 滚雪球污染

memory_extraction_on_compaction: |
  [verified-from-source]
  `commit_memory_session(messages)` 在 compaction rotation **前** 触发 (`run_agent.py:8990`)
  → 让 memory provider 在旧 session 结束前提取重要事实,避免被压缩丢掉。

multi_agent: |
  [verified-from-source]
  **仅 SubAgent (`delegate_task`),无网状 Team**。
  `MAX_DEPTH = 1` (`tools/delegate_tool.py:128`) —
    "flat by default: parent (0) -> child (1); grandchild rejected unless max_spawn_depth raised."
  Child agent 配置 (`tools/delegate_tool.py`):
    `skip_memory=True` (`:1047`) — 无 parent memory provider 访问
    `delegate_task` 在 child disabled (`:42` "no recursive delegation")
    child budget = `delegation.max_iterations` 默认 50 (独立累计,`run_agent.py:266-271`)
  父只看到 task prompt + final response observation
  `subagent_stop` hook 触发 (`hermes_cli/plugins.py:88`)

eval_loop: |
  [doc-only]
  `environments/` (Atropos RL 训练环境) + `batch_runner.py` (批量 trajectory 生成) +
  `mini_swe_runner.py` (SWE-bench mini runner) — 源码未深读
  面向 RL/research/training-data-gen
  **无 production prompt A/B 自动晋升 UI / 数据集管理 / LLM-as-judge dashboard**

self_evolution: |
  [verified-from-source]
  **核心产品卖点**。多机制叠加:

  1. `skill_manage` 工具 (`tools/skill_manager_tool.py:692` `skill_manage()`)
     Actions (`:708-741`): create / edit / patch / delete / write_file / remove_file
     `_pinned_guard` (`:134-152`) — pinned skill 拒绝修改,5 次调用位置
       (`:439, 483, 566, 613, 651`)
     SKILL.md frontmatter 校验 / supporting file 路径穿越保护 / atomic write
     可选 `skills.guard_agent_created` 配置 → guard scan rollback

  2. `skill_usage.py` sidecar 跟踪 (`tools/skill_usage.py`)
     Record fields (`:171-179`):
       use_count / view_count / patch_count /
       last_used_at / last_patched_at / last_viewed_at /
       state / pinned
     Bump 函数:
       `bump_view` (`:270-274`) — skill_view() 调用
       `bump_use` (`:278-284`) — 主动使用时
       `bump_patch` (`:287-292`) — skill_manage(patch/edit) 时
     `set_pinned` (`:308-311`)
     **仅 agent-created skill 进 sidecar** — 内置/Hub 由上游维护

  3. MEMORY.md 周期 memory nudges — system prompt 里 memory 提示 LLM 主动沉淀

  4. Cross-session `session_search` 工具 (`tools/skills_tool.py` 一致命名)
     用辅助模型摘要历史会话(不塞原文回主模型)
     `FTS5 + trigram + LIKE` 三层中文检索 (`hermes_state.py:104,133`)

  5. External memory provider 被动 `prefetch` 召回 + `sync_turn` 同步 + `on_memory_write` 桥接

  6. `agentskills.io` 开放标准兼容(README 提到)

  详见 [[../concept/self-evolution]]。

observability: |
  [doc-only]
  `~/.hermes/logs/{agent,errors,gateway}.log` profile-aware (AGENTS.md)
  `hermes logs --follow --level --session` CLI browse
  `save_trajectories=True` 可保 RL 训练数据
  TUI KawaiiSpinner + activity feed
  **未见 Langfuse 风格 trace UI / model usage dashboard / session replay UI**(SkillForge 反而更前)

hooks: |
  [verified-from-source]
  `VALID_HOOKS` **16 个** (`hermes_cli/plugins.py:75-111`):
    1. pre_tool_call  — 可 block (`:1169 get_pre_tool_call_block_message`)
    2. post_tool_call — 观察型,duration_ms
    3. transform_terminal_output — terminal 工具内部截断/清洗之前
    4. transform_tool_result — 可改写 tool result(进 messages 前)
    5. pre_llm_call — 只注 user message 不改 system prompt(保 cache)
    6. post_llm_call — 整个 user turn 结束 1 次
    7. pre_api_request / 8. post_api_request — 每次 API call 都触发(Langfuse 类 tracing 插件用)
    9. on_session_start
    10. on_session_end
    11. on_session_finalize
    12. **on_session_reset**(`:87` 笔记漏列)
    13. subagent_stop
    14. pre_gateway_dispatch — Gateway 消息分发前(skip/rewrite/allow)
    15. pre_approval_request — Approval lifecycle 观察(不能 veto)
    16. post_approval_response — choice in {once/session/always/deny/timeout}
  Fail-open: 插件异常 warning 但不阻断 agent loop
  `register_hook` (`hermes_cli/plugins.py:525`)
  `invoke_hook` (`:1052, 1160`)

safety:
  trust_model: "[verified-from-source] single-tenant operator. 不防多租户 (SECURITY.md §2). Multi-user isolation 走 OS/host 层"
  sandbox: "[verified-from-source] **7 backends** (`tools/environments/`): local(默认) / docker / ssh / modal / daytona / singularity / **vercel_sandbox**。详见 [[../concept/sandbox-isolation]](backend SPI 抽象横跨阶梯 0-5,业内最广)"
  command_approval: "[verified-from-source] `tools/approval.py` `approvals.mode` = on(默认) / auto / off / **smart**(用 aux LLM 预判, `:995`)"
  destructive_patterns: "[verified-from-source] `_DESTRUCTIVE_PATTERNS` regex (`run_agent.py:338-350`): rm/rmdir/cp/install/mv/sed -i/truncate/dd/shred/git reset/clean/checkout — 显式列出"
  path_traversal: "[verified-from-source] WRITE_DENIED_PATHS + skill file_path 解析时校验不能跳出 skill_dir"
  anti_runaway: "[verified-from-source] `max_iterations=90` + `IterationBudget` 每 turn 重置 + `_budget_grace_call` 兜底总结 + `_interrupt_requested` flag (30+ checkpoints in run_agent.py)"
  prompt_injection: "[verified-from-source] memory_tool 写入前 scan + `<memory-context>` 块化隔离 (`agent/memory_manager.py:46-189`) + `StreamingContextScrubber` 流式防泄漏 (`:65-173`)"
  secret_handling: "[verified-from-source] `redact_sensitive_text` 显示层 strip (`agent/redact.py:308`) + `RedactingFormatter` log (`:386`) + `credential_pool` 集中管理"
  execute_code_isolation: "[doc-only] child process strip API keys + RPC 调工具不直接 API + env_passthrough 显式声明"
  mcp_safety: "[verified-from-source] `_build_safe_env()` 只透传 PATH/HOME/XDG_* (`tools/mcp_tool.py:276`) + OSV malware check for npx/uvx (`tools/mcp_tool.py:1077-1080`)"
  skills_guard: "[doc-only] `tools/skills_guard.py` install 前隔离区(.hub/quarantine) + scan + audit.log"
  ssrf: "[doc-only] Gateway 平台 adapter 自带 SSRF protection + redirect 验证(SECURITY.md §4)"
  subagent_isolation: "[verified-from-source] `MAX_DEPTH=1` (`tools/delegate_tool.py:128`) + `skip_memory=True` (`:1047`) + 不能 delegate (`:42`)"
  supply_chain: "[doc-only] GitHub Actions pinned to commit SHA + supply-chain-audit.yml 阻 .pth + base64+exec (SECURITY.md §4)"

provider: |
  [verified-from-source]
  自抽象 4 种 api_mode (`run_agent.py:1034`):
    {"chat_completions", "codex_responses", "anthropic_messages", "bedrock_converse"}
  自动检测 (`run_agent.py:1036-1065`):
    provider == "openai-codex" → codex_responses
    provider == "xai" → codex_responses
    provider == "anthropic" 或 anthropic.com host → anthropic_messages
    provider == "bedrock" → bedrock_converse
    其他 → chat_completions(可能 GPT-5.x 升级到 codex_responses)
  Provider 选择参数 (`run_agent.py:908-913`):
    providers_allowed / providers_ignored / providers_order /
    provider_sort / provider_require_parameters / provider_data_collection
  Fallback chain: dict 或 list (`fallback_model`)
  Credential pool: `agent/credential_pool.py` 轮换
  Provider-specific headers: OpenRouter / Copilot / Kimi / Qwen / Codex Cloudflare
  200+ 模型: Nous Portal / OpenRouter / NIM / Xiaomi / z.ai / Kimi / MiniMax / HF / OpenAI / Anthropic / Bedrock

channels: |
  [verified-from-source]
  Gateway 单 process 多平台,**18+ adapters** (`gateway/platforms/`):
    api_server / bluebubbles / dingtalk / discord / email / feishu / feishu_comment /
    homeassistant / matrix / mattermost / qqbot / signal / slack / sms /
    telegram / webhook / wecom / weixin / whatsapp / yuanbao_media
  `BasePlatformAdapter` (`gateway/platforms/base.py:1160`)
  `handle_message` (`gateway/platforms/base.py:2372`)
  `ChannelAdapter` SPI + `ADDING_A_PLATFORM.md` 文档化扩展
  session_key 用于路由非授权边界(single-tenant trust model)
  Gateway 缓存每个 session 的 `AIAgent` + 配置签名判断复用(保 prompt cache)
  Dashboard `/chat` 通过 PTY 嵌入真实 `hermes --tui`(不重写)
  ACP 入口走 stdin/stdout JSON-RPC,output 走 stderr 保协议纯净

persistence: |
  [verified-from-source]
  SQLite `HERMES_HOME/state.db` (`hermes_state.py`)
  sessions + messages 两张核心表 (`hermes_state.py:50, 71, 98`)
  `parent_session_id` FK 支持 compaction continuation / branch / subagent chain
  JSON session log `~/.hermes/sessions/session_<id>.json` 双写
  FTS5 三层中文检索 (`hermes_state.py:104, 133`)
  Profile 隔离: `HERMES_HOME=root/profiles/<name>`
  `_apply_profile_override()` 在 `hermes_cli/main.py` 最早执行(在任何重型模块 import 前)
  Tool result budgets (`tools/budget_config.py:18-20`):
    `DEFAULT_RESULT_SIZE_CHARS = 100_000`
    `DEFAULT_TURN_BUDGET_CHARS = 200_000`
    `DEFAULT_PREVIEW_SIZE_CHARS = 1_500`
  超 size 落盘到 `/tmp/hermes-results/{tool_call_id}.txt`(`tools/tool_result_storage.py`)

cancellation: |
  [verified-from-source]
  双重: `_interrupt_requested` flag (user/gateway 中断) + `IterationBudget` (预算耗尽)
  `_interrupt_requested` 在 30+ 位置检查 (`run_agent.py:1153, 4309, 4355, 6402, 6571, 6638,
    7324, 9242, 9351, 9441, 9480, 10471, 10506, 11157, 11959, 12545` 等)
  `IterationBudget.consume()` 失败 → 跳出 while (`run_agent.py:282-288`)
  `is_interrupted` property (`run_agent.py:4773`)
  Worker thread 继承 interrupt + approval/sudo callback(并发执行段)
  Gateway 的 `/stop` / `/new` / `/approve` / `/deny` 由 runner 直接拦截**不进 agent loop**
    (`gateway/run.py` + `gateway/platforms/base.py` 两层 guard 都必须 bypass)
  Codex incomplete 连续超限 / 无效工具名自纠失败 / tool args 截断 / API retry 耗尽 /
    压缩失败 等多种 partial-error 出口

skillforge_takeaway: |
  ## 直接可借鉴(高优先级,带源码锚点)

  1. **[HIGH] `_DESTRUCTIVE_PATTERNS` regex 模式作为 bash_blocklist baseline** (`run_agent.py:338-350`)
     - 9 个高危命令模式 verified
     - SkillForge `safety.bash_blocklist` 可直接采用同样的 verbose regex
     - 优势:declarative + 易扩展

  2. **[HIGH] `MAX_DEPTH = 1`** (parent=0 → child=1, grandchild rejected) (`tools/delegate_tool.py:128`)
     - **关键安全常量**,只在源码可见(笔记未提及,A 模式 wiki 曾 hallucinate 为 2)
     - SkillForge SubAgent 设计应硬限深度 1
     - 这是 trust model 的安全关键 — 递归 delegation 是攻击面

  3. **[HIGH] `IterationBudget.refund()` for `execute_code`** (`run_agent.py:273, 290-294`)
     - SkillForge 如有"工具内调工具"机制,iteration 计费要区分内层
     - Hermes 用 refund 模式,避免内层调用吃 parent 预算

  4. **[HIGH] `StreamingContextScrubber` 跨 SSE chunk 防泄漏** (`agent/memory_manager.py:65-173`)
     - SkillForge 如做 memory context 注入,**流式响应必须考虑跨 chunk 的 tag 切分**
     - 否则 `<memory-context>` 内容会泄漏到用户可见输出 — production 级问题
     - 状态机 `_OPEN_TAG/_CLOSE_TAG` + `_max_partial_suffix` 是 verified 实现

  5. **[HIGH] MemoryManager 严格 1+1 限制** (`agent/memory_manager.py:206-228`)
     - builtin 永在 + 仅 1 个 external,重复 reject + warning
     - SkillForge 引入外部 memory provider 应同样限制,避免多 backend 冲突 + tool schema 膨胀

  6. **[HIGH] `commit_memory_session()` 在 compaction rotation 前 trigger memory extraction**
     (`run_agent.py:8990`)
     - 旧 session 结束前给 memory provider 提取重要事实的机会 — 否则压缩可能丢
     - SkillForge compact 流程可加此 hook

  7. **[HIGH] `approvals.mode = smart`** (`tools/approval.py:995`)
     - 用 aux LLM 预判是否需要 approval — 避免每次 destructive 都打扰用户
     - SkillForge 命令审批可借鉴

  8. **[HIGH] `on_session_reset` 是单独 hook** (`hermes_cli/plugins.py:87`)
     - SkillForge `/reset` 应有等价 hook 给 plugins 清理 cache

  ## 业界印证(说明 SkillForge 当前选择合理)

  - SkillForge `persistence-shape-invariant` rule ↔ Hermes `_sanitize_tool_pairs(compressed)` (`:1391`)
    + 切新 session `parent_session_id` chain (`run_agent.py:8985-9013`) 完全同源
  - SkillForge Light/Full compact ↔ Hermes Phase A 预处理 + Phase B 结构化摘要
  - SkillForge ChannelAdapter SPI ↔ Hermes 18+ platforms + `BasePlatformAdapter`
  - SkillForge Provider 抽象 + fallback ↔ Hermes 4 api_mode + fallback chain
  - SkillForge `compact-reviewer` agent ↔ Hermes 反复压缩 `<10%` 保护 + 多次压缩警告

  ## SkillForge 反而更前

  - [+] Langfuse 风格 trace + session replay + model usage dashboard — Hermes 仅日志,无 UI
  - [+] Prompt A/B 自动晋升流水线 — Hermes 偏 RL/research,无 production eval UI
  - [+] **Multi-Agent Team(网状,`TeamCreate/Send/Kill`)** — Hermes 仅 SubAgent(深度 1)

  ## 待 SkillForge 决策(建议立项)

  - **[DECIDE]** 多租户支持?Hermes 明确 single-tenant (SECURITY.md §2),SkillForge 服务端定位
    是否多租户?隔离边界在哪?(session_key 仅路由,非授权边界)
  - **[DECIDE]** 是否引入"轻量 MEMORY.md 长期事实"层?现在 SkillForge 是 pgvector + ?
    `memory_char_limit=2200, user_char_limit=1375` 是个经过验证的小尺寸 baseline
  - **[DECIDE]** session_search 用辅助模型摘要历史(而非塞原文)的范式
  - **[DECIDE]** FTS5 + CJK trigram + 短中文 LIKE 三层中文检索方案
    — SkillForge PG 检索的中文配方,看是否走 PG full-text 等价路径
  - **[DECIDE]** `skill_manage` action 集合(create/edit/patch/delete/write_file/remove_file)
    — SkillForge agent runtime 自进化的 API 形态(注意区分 meta `/evolve`)

  ## 风险/避坑

  - Hermes `run_conversation()` 单函数承担 9 阶段,~10K LOC,函数难读维护
    → SkillForge 应拆 TurnContext / ProviderCallRunner / ToolCallRunner / RecoveryPolicy
  - Prompt cache breaking 是 Iron Law — 任何"动态注入"功能必须先对照
    (中途改 system prompt = cache 失效 = 成本陡增)
  - `<memory-context>` 防污染设计精细 — SkillForge 直接做 memory 注入容易踩"resume 滚雪球"坑

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)

  - **"Self-improving" 是产品级核心而非附加** —— 整个设计围绕 learning loop 展开
    (`skill_manage` + `skill_usage.py` + memory nudges + `session_search` 跨会话搜索 +
    Honcho dialectic user modeling + `agentskills.io` 开放标准兼容)
  - **多入口共享单个同步 `AIAgent`**(7 入口 CLI/TUI/Gateway/ACP/MCP/Cron/Batch 走同一份
    `run_conversation()`),而不是每入口独立 runtime —— 行为一致性 > 入口特化
  - **Trust model 极简明确**:`SECURITY.md §2` 写"single-tenant operator",**不假装防多租户**
    —— 这个明确表态比"假装防但实际有漏洞"更负责任
  - **18+ messaging gateway 共一份 `ChannelAdapter` SPI** —— 同类 agent 大多没这么多 channel
    或者每 channel 独立 runtime,Hermes 用单一 SPI 抽象覆盖 Telegram/Discord/Slack/WhatsApp/
    Feishu/WeCom/Matrix/etc

  ## 优势(strengths)

  - **`<memory-context>` 块化 + `StreamingContextScrubber` 跨 SSE chunk 防泄漏**
    (`agent/memory_manager.py:46-189, 65-173`)—— 流式响应中 memory 内容不会泄漏到用户可见输出,
    是其他 harness(含 Claude Code)未做到的工程级细节
  - **Compaction 完整 7 步 + tool_call/result 配对修复 + `parent_session_id` chain**
    (`run_agent.py:8985-9013` + `agent/context_compressor.py:1391`)—— 有损压缩 + 历史不丢可查,
    优于 Claude Code in-place 替换的"历史合并到 boundary marker"
  - **Provider 抽象 4 api_mode 服务 200+ 模型 + fallback chain**(`run_agent.py:1034-1065`)
    —— 同类 agent 框架最简洁的 multi-provider 设计;Claude Code 单 Anthropic 反而绑死
  - **`commit_memory_session` 在 compaction rotation 前 trigger memory extraction**
    (`run_agent.py:8990`)—— 时序设计 elegant,压缩前给 memory provider 抢救事实的 last chance
  - **`MemoryManager 1+1 限制`** + **`<memory-context>` ephemeral 注入** —— 工具 schema 不膨胀
    + 写入不冲突 + 模型不困惑,**一个简单约束解决三个问题**

  ## 哪些点做得好(what_done_well)

  - **Prompt cache 作为 Iron Law 贯穿整个设计**:中途写 memory 不改 system prompt
    (`tools/memory_tool.py` 冻结快照)、Gateway agent cache by config signature、
    `/reload-skills` 默认不清 cache、continuation 不重读 memory/skills —— 高度统一的"不破缓存"
    哲学。**这种"为单一原则贯彻整个设计"的工程一致性是 elegant 的标志**
  - **依赖官方 SDK 不重发明轮子**:`openai` / `anthropic` SDK 走真实 HTTP,
    Hermes 只在 `agent/transports/*` 协议适配层抽象 —— 避免了"自己写 HTTP client + 自己维护 retry"
    的常见坑(对比有些 agent 自己手写 SDK 然后维护噩梦)
  - **`commit_memory_session` 时序** —— rotation **前** trigger memory extraction,
    是个"看似小细节实则关键"的设计。**先做摘要再分流 vs 先分流再做摘要**的选择直接决定
    "重要事实会不会丢"(详见 [[../concept/context-compaction]])
  - **`MAX_DEPTH=1` 硬限**(`tools/delegate_tool.py:128`)+ subagent `skip_memory=True` ——
    安全关键常量,直接用 Python `frozenset/Final-style` 写死,**不留 config 余地** ——
    "不可配置"反而是更负责任的安全设计
  - **明确写"single-tenant" in SECURITY.md** —— 大量 agent 项目不敢明确表态自己的 trust
    boundary,Hermes 直接写出来,**反向降低了误用风险**
  - **`<memory-context>` 状态机 scrubber** —— 处理流式跨 chunk tag 切分的 corner case,
    是只有 production 跑过才会想到的细节
  - **`run_conversation()` 长函数**(~10K LOC)虽然难维护,但**集中所有状态 + 跨入口一致**
    —— 是个有意识的取舍(注释也承认),不是无意识的烂代码;**SkillForge 不该照搬**
    但应理解"为什么不拆"
---

# Hermes Agent

## 一句话定位

Nous Research 出品的"**自进化 Python Agent**",一份 `AIAgent` (`run_agent.py:864`) 同步运行时驱动 7 个入口,把"经验→skill→记忆"的闭环作为产品级核心能力,且整套设计严格围绕"不破坏 prompt cache"展开。

## 起源与定位

- **背景**:Nous Research(以"个性化 AI"路线著称)的产品级 agent,继承自 OpenClaw 的 messaging gateway 思路并往"learning loop"方向深化
- **目标用户**:个人 + 长驻服务场景(`$5 VPS / GPU 集群 / serverless` 都跑得起)
- **核心差异化**:built-in learning loop(skill 自动创建 / skill 使用中自我改进 / memory nudges / 跨 session 搜索 / Honcho dialectic user modeling),`agentskills.io` 开放标准兼容
- **不解决**:多租户(`SECURITY.md` §2 明确 single-tenant operator)、production eval pipeline(无 prompt A/B 自动晋升 UI)
- **License**:MIT

## 核心架构

```
                    ┌──────────────────────────────────────┐
                    │   多入口 (event normalization layer)  │
                    │   CLI / TUI / Gateway (18+ chans)   │
                    │   ACP / MCP / Cron / Batch          │
                    └──────────────────┬───────────────────┘
                                       ▼
                    ┌──────────────────────────────────────┐
                    │   AIAgent (run_agent.py:864, 同步)    │
                    │   run_conversation() 9 阶段          │
                    │   while tool_use loop (:10501)       │
                    │   + 9 类异常分支 retry/fallback      │
                    └──────────────────┬───────────────────┘
                          ┌────────────┼────────────┐
                          ▼            ▼            ▼
                  ┌────────────┐ ┌──────────┐ ┌───────────┐
                  │ transports │ │ registry │ │ Session+  │
                  │ 4 api_mode │ │ +toolset │ │ Memory    │
                  │            │ │ AST scan │ │ 3 layers  │
                  └──────┬─────┘ └─────┬────┘ └──────┬────┘
                         ▼             ▼             ▼
                 200+ models      40+ tools     SQLite+FTS+
                                 (auto-disc)    MEMORY.md+
                                                external (1)
                  ┌───────────────────────────────────────┐
                  │ Lifecycle: 16 hooks (plugins.py:75)   │
                  │ Safety: 7 sandbox backends            │
                  │   + approval (smart) + redact         │
                  │   + MAX_DEPTH=1 + skip_memory subagent│
                  └───────────────────────────────────────┘
```

## 13 维度详解

详见 frontmatter,每条都带 `[code: file:line]` 锚点,可跳源码 verify。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway` 4 类分组:**直接可借鉴 8 条(高优先级)/ 业界印证 5 条 / SkillForge 更前 3 条 / 待决策 5 条 / 风险避坑 3 条**。

## 待回答的问题(lint 会扫这一节)

- `_compress_context()` 在 8920-9100 范围内,本次只读了 8918-9040 共 120 行,后续 100 行没读 — 是否还有其他重要逻辑?(待补)
- `_handle_max_iterations()` 内部具体 prompt 怎么写? (`run_agent.py:9951` 未深读)
- `agent/transports/*.py` 4 个 api_mode 各自实现细节?(本次未读)
- `tools/skills_tool.py:849` `skill_view()` 完整签名 + linked_files 处理?(只 grep 未深读)
- `delegate_tool.py:874-1369` child_depth + delegate event 完整流程?(只 grep)
- Cron `cronjob` toolset 禁用机制?(笔记提及未源码 verify)
- `agent/credential_pool.py` 轮换策略?(未读)
- 跟 [[harness/claude-code]] (待 ingest) 的 prompt cache 设计差异?
- 跟 [[harness/openclaw]] (Hermes 的前身) 的继承/改进点?
