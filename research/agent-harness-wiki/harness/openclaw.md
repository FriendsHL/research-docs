---
type: harness
name: openclaw
status: active
language: typescript
verification: source-grounded
ingest_mode: B
updated: 2026-05-16

# 版本时效(2026-05-16 补:schema §3.1 升级后回填)
last_ingested_version: "2026.4.20 (calver) @ commit 4407df6c03 (2026-04-21)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/openclaw]]"
  - "/Users/youren/myspace/openclaw/"
  - "[[../../OpenClaw源码研究/index]]"
  - "[[../../OpenClaw源码研究/04-Agent引擎深度解析]]"
  - "[[../../OpenClaw Memory Dreaming 系统分析]]"
  - "[[../../OpenClaw Memory Wiki 系统分析]]"
  - "[[../../OpenClaw QA Character Eval 系统分析]]"

agent_loop: |
  [verified-from-source]
  **基于 pi-sdk 嵌入式集成**(Mario Zechner / badlogic 开源)三层 (`笔记 04-Agent引擎.md:15-19`):
    pi-ai → 底层 LLM API 抽象(Anthropic/OpenAI/Google 等统一)
    pi-agent-core → 中层 ReAct loop
    pi-coding-agent → 上层(Session 持久化 + 工具 + Retry + Compaction)
  OpenClaw 直接 import 嵌入式集成,**不自己重写 agent loop**。

  **双层循环显式拆分**:
    外层 `runEmbeddedPiAgent` (`src/agents/pi-embedded-runner/run.ts`):
      `while(true)` 最多 32-160 次重试(auth profile 数量动态决定)
      职责:运行级容错编排,不参与 LLM 对话本身
      决策树:
        - context overflow → 触发 compaction 压缩 → continue
        - auth error → 刷新 OAuth token → continue
        - rate limit / overload → markProfile cooldown → advanceAuthProfile → continue
          - 无 profile + 有 fallback model → throw FailoverError → 换模型重跑
          - 无 profile + 无 fallback → fall-through 返回错误
        - thinking level 不支持 → 降级 high → low → off → continue
        - retry exhausted → 返回错误
        - 正常结束 → markAuthProfileGood → 返回 payloads
    内层 `runEmbeddedAttempt` (`run/attempt.ts`):
      驱动一次完整 tool-use 对话,**失败直接抛,不处理任何容错**
      流程:解析 sandbox / 加载 skills prompt / 加载 bootstrap files /
            创建 tools(createOpenClawCodingTools + MCP tools + LSP tools)/
            构建 system prompt(buildEmbeddedSystemPrompt)/
            SessionManager.run() → pi-coding-agent 真正 tool-use loop /
            订阅流式事件 subscribeEmbeddedPiSession(onPartialReply/onBlockReply/
            onToolResult/onReasoningStream)
      结束:after_agent_end hooks + context engine 维护(向量库)

  **跟 Hermes `run_conversation()` 单函数承担 9 阶段对比**:OpenClaw 的双层显式拆分更
  elegant — 内层只管"做什么",外层只管"出错怎么办",混在一起 tool-use 第 5 轮 auth 过期处理
  会极其复杂(笔记原话)。

tool_protocol: |
  [verified-from-source partial]
  `createOpenClawCodingTools()` 创建 OpenClaw 自有工具 + **MCP tools** + **LSP tools**
  (笔记 04-Agent引擎.md:84-86)。
  Plugin SDK 公开 `packages/plugin-sdk` npm 包,**extension-agnostic core 哲学**
  (`AGENTS.md:41`:"Core must stay extension-agnostic")。
  Extensions 跨进核心仅通过 `openclaw/plugin-sdk/*`、manifest、injected runtime helpers、
  documented local barrels (`api.ts`, `runtime-api.ts`)。
  Extension 不可 deep-import core 或其他 extension 内部。
  具体 tool 列表未深读源码 verify。

context_mgmt: |
  [verified-from-source partial]
  **pi-coding-agent 自带 Compaction**,在**外层循环**遇 `context overflow` 触发
  (`runEmbeddedPiAgent` 决策树,笔记 04-Agent引擎.md:54-55)。
  跟 Hermes / Claude Code 不同:OpenClaw 把 compaction 作为外层容错决策的一支,
  **不在内层 tool-use loop 内部决定**。
  `AGENTS.md:54` 明确 "**Preserve old transcript bytes when possible**" —
  prompt cache 优先。
  具体 5 触发点 / Phase 拆分 / persistence-shape 保护未深读 pi-coding-agent 源码。

memory: |
  [verified-from-source]
  **三层 + Memory Wiki extension 四件套**:

  ① **memory-core** (`extensions/memory-core/src/`,20+ files):
     短期召回 store(`ShortTermRecallEntry`,字段含 recallCount/dailyCount/totalScore/maxScore/
     queryHashes(≤32)/recallDays(≤16)/conceptTags/promotedAt)
     Dreaming 系统(详见 self_evolution)
     concept-vocabulary.ts(概念词典)
     rem-evidence.ts(REM 阶段证据)
     memory-events.ts
     flush-plan.ts / prompt-section.ts / public-artifacts.ts

  ② **memory-wiki** (`extensions/memory-wiki/src/`,25 files):
     **Karpathy llm-wiki 模式 production 实现**(详见 [[../concept/llm-wiki-pattern]]):
     vault.ts(初始化目录:entities/concepts/syntheses/sources/reports + AGENTS.md/WIKI.md)
     ingest.ts(摄入本地文件为 source 页面)
     bridge.ts(从 memory-core 自动同步 artifacts:日记/梦境报告/事件日志)
     compile.ts(编译 index + dashboard 页面 + agent-digest.json + claims.jsonl)
     claim-health.ts(claim 新鲜度评估 + 矛盾 cluster 检测)
     prompt-section.ts(从 digest 生成系统 prompt 注入片段)
     tool.ts(暴露给 Agent 的 5 个工具)
     lint.ts(vault 结构健康检查)
     obsidian.ts(Obsidian CLI 集成)
     chatgpt-import.ts(从 ChatGPT history 导入)
     corpus-supplement.ts / gateway.ts / config.ts / apply.ts / query.ts /
     import-insights.ts / import-runs.ts

  ③ **active-memory** (`extensions/active-memory/` 未深读)

  ④ **MEMORY.md / DREAMS.md** 持久输出:
     MEMORY.md 是 Dreaming 晋升结果
     DREAMS.md 是梦境日记 append-only(managed block `<!-- openclaw:dreaming:diary:start/end -->`)

  **Claim 结构**(`extensions/memory-wiki/src/claim-health.ts`):
    yaml frontmatter `claims:` 数组,每条:
      id / text / status(supported/contested/contradicted/refuted/superseded) /
      confidence(0-1)/ updatedAt /
      evidence(数组,每条 sourceId/lines/weight/note/updatedAt)
  **Freshness**:fresh(<30天)/ aging(30-90)/ stale(>90)/ unknown(无 updatedAt),
    取 claim.updatedAt、page.updatedAt、evidence[].updatedAt 中最新。
  **Contradiction cluster**:同 claimId 多页出现且 text/status 不同 → cluster 入
    reports/contradictions.md + agent-digest。

multi_agent: |
  [verified-from-source partial]
  **subagent 用于 Memory Dreaming 写日记 + QA Character Eval 候选运行**:
    Dreaming `dreaming-narrative.ts` 调用 subagent 写 80-180 词梦境日记
    QA `character-eval.ts:393` 对每个候选模型串行调用 `runQaSuite` 跑场景
  **ACP**(Agent Communication Protocol,`extensions/acpx`,笔记 06):
    跨 agent 通信协议 + Hooks 拦截能力(笔记 10 综合解析)
  具体多 agent 网状协作 / Coordinator 等模式未深读源码。

eval_loop: |
  [verified-from-source]
  **QA Character Eval**(`extensions/qa-lab/src/`)— **3 harness 里 eval_loop 最完整**:
    入口 `character-eval.ts:393 runQaCharacterEval`
    **双阶段**:
      第一阶段:候选模型串行 `runQaSuite`,每候选启动 3 服务:
        startQaLabServer() — MCP server(提供测试工具)
        startQaMockOpenAiServer() — mock 模式
        startQaGatewayChild() — **真实 Gateway 子进程**(`openclaw gateway run`,IPC 通信,
          不 mock,跑真实代码路径)
      第二阶段:Judge 模型对所有 transcript 横向打分排名
    **场景定义**:YAML 双 fence DSL(`scenario-catalog.ts`):
      ```yaml qa-scenario
      id / title / surface(telegram 等)/ objective / successCriteria
      ```
      ```yaml qa-flow
      steps[]: name / actions[]: {call, args, assert, saveAs}
      ```
    **Flow DSL 编译执行**(`scenario-flow-runner.ts`):
      把 YAML action 动态编译为 `new AsyncFunction(...contextNames, "return (${expr});")`
      然后 `await fn(...contextValues)`
    **支持 actions**:sendMessage / assert(reply contains / length) / saveAs(变量绑定)/
      ${vars.xxx} 引用
    **Bus 消息总线**(`bus-state.ts / bus-server.ts`)驱动 Agent 后续交互
    **QA Agent 身份**:"Dev C-3PO" — protocol-minded 的 QA 操作员
    **Lab Server (MCP)**:暴露记录 Agent 发出消息 / 读最新场景报告 / 检查 QA 状态
    **Report**(`report.ts`):生成 Markdown 报告
    **Judge 专用**:`manual-lane.runtime.ts` 单轮推理

self_evolution: |
  [verified-from-source]
  **核心产品卖点之一**,**self-evolution 源头**(Hermes 是简化版,Claude Code 借鉴命名)。
  **2 系统**:

  ① **Memory Dreaming**(`extensions/memory-core/src/dreaming-*.ts`,8 文件):
     **触发**(`dreaming.ts:30-40` 常量 verified):
       Cron job name = `"Memory Dreaming Promotion"`
       Cron tag = `"[managed-by=memory-core.short-term-promotion]"`
       system event = `"__openclaw_memory_core_short_term_promotion_dream__"`
       触发类型 = `sessionTarget: "main"` + `wakeMode: "next-heartbeat"`
       Reconcile interval = `RUNTIME_CRON_RECONCILE_INTERVAL_MS = 60_000` (1 min)
       Legacy(已统一):`LEGACY_LIGHT_SLEEP_CRON_TAG/EVENT_TEXT` +
         `LEGACY_REM_SLEEP_CRON_TAG/EVENT_TEXT`(自动 delete 旧 legacy jobs)
     **2 hooks 注册**(`dreaming.ts:582 registerShortTermPromotionDreaming`):
       `gateway:startup` → reconcile Cron job
       `before_agent_reply` → trigger 必 `"heartbeat"` + body 含 system event token

     **3 phases 扫描**(`dreaming-phases.ts::runDreamingSweepPhases`):
       Phase 1 — Daily Note 摄入(Light Sleep):
         读 `memory/YYYY-MM-DD.md`,跳过已处理(游标 `daily-ingestion.json`)
         过滤通用日期标题 `GENERIC_DAY_HEADING_RE`
         每片段 ≤280 chars,≤4 lines
         固定分 `0.62` 写入短期召回 store
       Phase 2 — Session Transcript 摄入(REM Sleep,本期新增):
         读 `memory/.dreams/session-corpus/` transcript
         每扫 ≤240 msg,每文件 ≤80 msg,每 session ≤4096 msg(防无限增长)
         固定分 `0.58` 写入短期召回 store
         游标 `session-ingestion.json`
       Phase 3 — Phase Signal 记录:
         Light hit 最多 +0.05 boost
         REM hit 最多 +0.08 boost
         半衰期 14 天

     **6 维评分晋升**(`short-term-promotion.ts`):
       score = frequency × 0.24 + relevance × 0.30 + diversity × 0.15
             + recency × 0.15 + consolidation × 0.10 + conceptual × 0.06
       维度含义:
         frequency 被召回次数(归一化)
         relevance 召回时的平均/最高相关度分数
         diversity 查询多样性(不同 query 的数量)
         recency 最近召回时间衰减(半衰期可配,默认 14 天)
         consolidation 跨天分布(不同天召回 = 更稳定)
         conceptual 概念标签丰富度

     **晋升门槛**(`dreaming.ts:102-104`):
       minScore = 0.75
       minRecallCount = 3
       minUniqueQueries = 2
       三个 AND 关系

     **晋升执行**(`applyShortTermPromotions`):append 候选片段到 `MEMORY.md`,
     标记 `promotedAt` 防重复晋升

     **梦境日记 subagent**(`dreaming-narrative.ts`,本期新增):
       晋升完成后调用 subagent 写日记 → DREAMS.md(or dreams.md)
       managed block `<!-- openclaw:dreaming:diary:start/end -->`
       System prompt 风格设定(笔记 verified):
         "你在写梦境日记。用第一人称..."
         声音与基调:"好奇、温柔、略带异想天开...程序员的诗人...感官的、温暖的..."
         规则:**不说"在做梦"/"梦里"/"AI"/"agent"/"LLM"** — 反元评论
              不用 Markdown 标题/列表,只有流动散文
              80-180 词
       NarrativePhaseData:phase / snippets(≤12)/ themes(≤6)/ promotions(≤5)

  ② **Memory Wiki**(structured knowledge persistence,见 memory 维度 + [[../concept/llm-wiki-pattern]]):
     **claim with evidence array + freshness + contradiction cluster**(详 frontmatter)
     **页面排名公式**(prompt 注入)(笔记 `Memory Wiki.md:115`):
       score = 矛盾数 × 6 + 未解问题数 × 4 + min(claim数,6) × 2 + min(topClaims数,3)
       最多取 4 个高分页面,每页最多 2 条 claim(confidence ↓ → freshness ↓)
     **5 Agent tools**:wiki_status / wiki_search / wiki_get / wiki_apply / wiki_lint
     **lint 分类**:structure(missing-id/duplicate/missing-type/title)/ content / health
     **三层分离原则**:
       人读 = Markdown 页面
       机器操作 = JSONL / JSON
       人工备注 = `<!-- openclaw:human:start/end -->` 块,**compile 永远不覆盖**

observability: |
  [doc-only]
  `extensions/diagnostics-otel` — **OpenTelemetry 标准化诊断**(比 Hermes/Claude Code 都更专业)
  笔记 03-Gateway 提及 health monitoring
  笔记 08 Cron + heartbeat 机制
  具体未深读

hooks: |
  [verified-from-source partial]
  **Internal Hooks + Plugin Hooks 双层**(笔记 07-Hooks 系统源码解析,本次未深读源码)
  **已 verified 的 hook 名**(从 dreaming.ts 和 笔记 04):
    `gateway:startup` — 网关启动
    `before_agent_reply` — Agent 回复前
    `before_model_resolve` — 解析 model 前
    `before_agent_start` — Agent 启动前
    `after_agent_end` — Agent 结束后
    `on_pre_compress` 类(从 pi-coding-agent)
  具体 hook 完整列表 + zod schema(像 Claude Code 那种)未 grep verify

safety: |
  [doc-only]
  Sandbox:`Dockerfile.sandbox` + `Dockerfile.sandbox-browser` + `Dockerfile.sandbox-common` 多种
  `extensions/device-pair` — 设备配对
  `INCIDENT_RESPONSE.md` 存在(根目录)
  详见 [[../concept/sandbox-isolation]] (阶梯 3 OCI 容器,dev environment per skill 形态).
  **Extension-agnostic core 哲学**(`AGENTS.md:41`):防 plugin 污染 core
  **Plugin loader/registry/contracts** 严格区分(`src/plugin-sdk/` vs `src/plugin-sdk-internal/`)
  **Memory Wiki secret guard**?— `extensions/memory-wiki/src/` 未见明显 secret scanner
    (跟 Claude Code teamMemorySync 不同),但 active-memory / corpus-supplement 可能有
  具体规则未深读

provider: |
  [verified-from-source partial]
  **extensions 30+ providers**:anthropic / anthropic-vertex / amazon-bedrock(+mantle) /
    arcee / byteplus / chutes / codex / deepseek / alibaba / cloudflare-ai-gateway /
    copilot-proxy / fal / elevenlabs / deepgram / brave / exa / duckduckgo / …
  **底层 pi-ai** 统一 LLM API 抽象(Anthropic/OpenAI/Google 等)
  **OAuth subscription**:OpenAI ChatGPT/Codex(`extensions/codex`)
  README:"prefer current flagship model from provider you trust"

channels: |
  [verified-from-source]
  **24+ messaging channels**(README:26):
    WhatsApp / Telegram / Slack / Discord / Google Chat / Signal / iMessage / BlueBubbles /
    IRC / Microsoft Teams / Matrix / Feishu / LINE / Mattermost / Nextcloud Talk / Nostr /
    Synology Chat / Tlon / Twitch / Zalo(+Zalo Personal)/ WeChat / QQ / WebChat
  **4 native apps**(`apps/`):android / ios / macos / shared
  **Gateway**(笔记 03):HTTP/WebSocket 双协议
  **Channel core**(`src/channels/`)+ Plugin SDK seams(笔记 09)

persistence: |
  [doc-only]
  workspace/(笔记 02 Session-Routing + 笔记 04 Agent 引擎)
  MEMORY.md + DREAMS.md + memory-wiki vault + session files
  短期召回 store(`short-term-promotion.ts`):key=path+line hash
  状态文件:`daily-ingestion.json` / `session-ingestion.json` / `.openclaw-wiki/state.json` /
    `.openclaw-wiki/cache/claims.jsonl` / `agent-digest.json` /
    `.openclaw-wiki/log.jsonl`(操作审计)

cancellation: |
  [doc-only]
  `subscribeEmbeddedPiSession()` 流式订阅(笔记 04)暗示支持取消
  外层 retry 循环可中断
  具体取消机制未读

skillforge_takeaway: |
  ## 直接可借鉴(高优先级,带源码锚点)

  1. **[HIGH] Memory Wiki vault 系统 = SkillForge 知识库的明确蓝本**
     (`extensions/memory-wiki/src/` 25 文件)
     - 5 工具(wiki_status/search/get/apply/lint)+ claim/evidence/freshness + 编译 digest
       自动注入 system prompt + Obsidian 兼容 + managed blocks
     - **跟我们手做的 agent-harness-wiki 同源** — 可直接照抄 OpenClaw 实现路径
     - 详见 [[../concept/llm-wiki-pattern]]

  2. **[HIGH] Memory Dreaming 6 维评分晋升公式**(`short-term-promotion.ts`)
     `freq×0.24 + rel×0.30 + div×0.15 + rec×0.15 + cons×0.10 + concep×0.06`
     - SkillForge memory 系统从"简单晋升"演进的**直接可抄公式**
     - 门槛 `minScore=0.75 / minRecallCount=3 / minUniqueQueries=2`

  3. **[HIGH] QA Character Eval 双阶段 framework**(`extensions/qa-lab/src/`)
     - 候选并行 + Judge 排名,真实 Gateway 子进程不 mock,YAML DSL 编译 JS AsyncFunction
     - **SkillForge eval pipeline production 范式**
     - 详见 [[../concept/agent-eval]]

  4. **[HIGH] 双层 agent loop 显式拆分**(`runEmbeddedPiAgent` 外 + `runEmbeddedAttempt` 内)
     - **比 Hermes `run_conversation()` 单函数承担 9 阶段更 elegant**
     - SkillForge 应拆 OuterLoop(容错决策)+ InnerLoop(tool-use 失败抛)
     - 内层不做容错 — auth 过期、context overflow、rate limit 都丢给外层处理

  5. **[HIGH] Cron + heartbeat 双轨触发**
     - `gateway:startup` reconcile Cron 独立调度 + `before_agent_reply` 检查 event routing
     - **比 Claude Code 的 in-loop 3-gate 更解耦** — Cron 不依赖 agent 跑不跑
     - SkillForge 自进化模块借鉴

  6. **[HIGH] 梦境日记 subagent + 反元评论 prompt**(`dreaming-narrative.ts`)
     - "不说在做梦/AI",80-180 词,程序员诗人风格
     - **Prompt 工程 + 用户体验的精彩结合** — SkillForge 任何"系统自反思"产出可借鉴

  7. **[HIGH] 编译 agent-digest.json 自动注入 system prompt**(`compile.ts`)
     - 知识库不是"被动等查询",而是"主动 push 高信号内容到 prompt"
     - **SkillForge memory 系统应有 compile step + prompt injection**

  8. **[HIGH] 三层分离**:人读 markdown + 机器 JSONL + 人工 managed block
     - `<!-- openclaw:human:start/end -->` 永远不被 compile 覆盖
     - SkillForge 自动维护 + 人工 override 的边界设计

  9. **[HIGH] Plugin SDK npm 公开 + extension-agnostic core**(`packages/plugin-sdk`)
     - core 严禁 plugin/provider/channel 特殊 case
     - SkillForge Skill / Tool / Provider / Channel 走 plugin/extension 路径的明确架构哲学

  10. **[MED] 4 native apps**(android/ios/macos/shared)
      - 如果 SkillForge 走 personal assistant 方向,native apps 不可少
      - 如果走 dev tool / server,可跳

  11. **[MED] `AGENTS.md` Telegraph style + repo file refs only**
      - SkillForge CLAUDE.md 给 LLM 看的项目自述风格可借鉴

  ## 业界印证(SkillForge 选择合理)

  - SkillForge `persistence-shape-invariant` ↔ OpenClaw "Preserve old transcript bytes when possible"(`AGENTS.md:54`)同源
  - SkillForge Eval Pipeline 方向 ↔ OpenClaw QA Character Eval 思路一致
  - SkillForge Skill 系统 ↔ OpenClaw extension 系统 哲学同源

  ## SkillForge 反而更前

  - [+] **Langfuse 风格 trace + replay UI** — OpenClaw 仅 diagnostics-otel(标准化但无 UI)
  - [+] **Prompt A/B 自动晋升流水线** — OpenClaw QA Eval 是 character 评测,不是 prompt A/B

  ## 待 SkillForge 决策(建议立项)

  - **[DECIDE]** Memory Wiki 是否照抄 OpenClaw(vault + claim + freshness + digest auto-inject)?
    跟 SkillForge 现有 pgvector 怎么协调?
  - **[DECIDE]** Memory Dreaming 6 维评分公式是否引入?门槛参数是否照抄?
  - **[DECIDE]** Agent loop 是否拆双层(SkillForge 是 sync Java,但分层结构可借鉴)?
  - **[DECIDE]** QA Eval framework 是否做 production agent eval(真实环境跑 + YAML DSL)?
  - **[DECIDE]** Native apps 是否做(android/ios/macos)?决定 personal vs server 定位
  - **[DECIDE]** Cron + heartbeat 双轨 vs in-loop gate — 哪种更适合 SkillForge?

  ## 风险/避坑

  - **OpenClaw 用 pi-sdk 嵌入式集成,SkillForge 已写自己 agent loop** — 不该回头重写,
    但可以学"双层显式拆分"的架构思想
  - **Memory Dreaming 6 维公式参数是经过 production 调优的,不要随意改** — SkillForge 借鉴时
    保持参数,只调 minScore/minRecallCount 等门槛
  - **TypeScript monorepo + pnpm workspace** 跟 SkillForge Java Maven 完全不同 —
    架构哲学可学,代码不能直接抄

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)

  - **Personal AI assistant 定位** —— 跑在 own devices,24+ messaging channels + 4 native apps,
    OpenAI sponsor — 同类 agent 大多是 dev tool 或 chat UI,OpenClaw 是 **multi-device + multi-channel
    personal assistant** 这一独立赛道
  - **底层 import pi-sdk 嵌入式集成** —— 不像 Hermes/Claude Code 自己重写 agent loop,
    而是 import Mario Zechner / badlogic 的开源 pi-sdk 三层(pi-ai / pi-agent-core / pi-coding-agent)
    —— **"不重发明轮子"的极致**
  - **双层 agent loop 显式拆分**(`runEmbeddedPiAgent` 外 + `runEmbeddedAttempt` 内)——
    职责分离极清晰,内层只跑 tool-use 不容错,外层 32-160 次重试容错决策树
  - **Memory Dreaming + Memory Wiki 双系统并行** —— 不只是"memory 一层",而是
    **短期事件流(Dreaming 晋升)+ 长期结构化知识(Wiki claim)** 两个产物分开
  - **Extension-agnostic core + extensions 全等地位** —— 30+ extensions(provider/channel/feature)
    走同一个 SDK 契约,**包括 memory/image-gen/speech/video 等 horizontal core 也是 extension**
  - **梦境日记 subagent 写散文** —— 不是机器生成的 log,而是程序员诗人风格的文学化日记 ——
    **systems thinking + 人文关怀的少见结合**

  ## 优势(strengths)

  - **Memory Wiki = Karpathy llm-wiki 模式的 production 实现**
    (`extensions/memory-wiki/src/` 25 files)—— 是其他 harness 都没做到的:
    claim(`confidence/status/evidence/updatedAt/weight`)+ freshness(fresh/aging/stale)+
    contradiction cluster + `compile.ts` → `agent-digest.json + claims.jsonl` →
    `prompt-section.ts` 自动注入 system prompt
  - **Memory Dreaming 6 维评分公式 + 反复迭代证据**:Legacy `LIGHT_SLEEP_CRON` 和
    `REM_SLEEP_CRON` 已统一为 `MANAGED_DREAMING_CRON`(`dreaming.ts:30-38`)——
    **不只是设计,是经过迭代的 production 产品**
  - **QA Character Eval 真实 Gateway 子进程不 mock**(`extensions/qa-lab/src/suite.ts`)——
    eval 跑真实代码路径,**比 mock-based unit tests 高一个等级**
  - **YAML DSL 动态编译 JS AsyncFunction**(`scenario-flow-runner.ts`)——
    场景定义跟代码完全解耦,**新场景写 YAML 不用改代码**
  - **Plugin SDK 公开 npm 包 + memory-host-sdk + plugin-package-contract** ——
    第三方 plugin 生态友好,**Hermes/Claude Code 都没做到这种程度的契约公开**
  - **Cron + heartbeat 双轨触发 Dreaming**(`dreaming.ts:582`)——
    Cron 独立调度不依赖 agent 在跑,heartbeat 触发只是 routing —— **比 Claude Code 的
    in-loop 3-gate 更解耦**

  ## 哪些点做得好(what_done_well)

  - **梦境日记反元评论 prompt**(`dreaming-narrative.ts`):
    "不说在做梦"/"不说梦里"/"不提 AI/agent/LLM/model"——
    **让 LLM 当文学创作者而非机器人** —— prompt 工程的杰作。同时:
    - 不用 Markdown 标题/列表 —— 流动散文
    - 80-180 词,质量优先于数量
    - 程序员的诗人 —— 感官的、温暖的、技术与温情混合
  - **6 维评分维度命名 `consolidation`**:跨天分布(不同天召回 = 更稳定的记忆)——
    **直接借用人脑记忆巩固的概念命名**,不是机械的"score weight",
    **设计语言带学科知识**
  - **页面排名公式 `矛盾×6 + 未解问题×4 + min(claim,6)×2`**(`prompt-section.ts`)——
    **矛盾 6 倍权重远高于其他**,因为矛盾是"系统失健康"的强信号,优先暴露给 agent
  - **管理块 `<!-- openclaw:human:start/end -->`**:compile 永远不覆盖 ——
    **机器自动 + 人工 override 的明确边界**,人不需要担心被覆盖
  - **`AGENTS.md` "Telegraph style"** + **repo file refs only**:`extensions/telegram/src/index.ts:80`
    —— 给 LLM 看的项目自述风格独特(对比 Hermes/Claude Code 都是叙事散文)——
    **token 经济 + 可解析**
  - **`AGENTS.md` "Preserve old transcript bytes when possible"**:prompt cache 优先 ——
    跟 Hermes "prompt cache Iron Law" 完全同源,**OpenClaw 用一行话表述了核心约束**
  - **Legacy Cron job 自动 delete + 命名空间隔离**(`dreaming.ts:33-38`)——
    迭代时不破老用户,**自动迁移而非强升级**
  - **三层分离 = 人读 markdown / 机器 JSONL / 人工 managed block**
    —— **Don't make me think** 应用到知识库:人看 markdown 直观,机器读 JSONL 高效,
    人工 override 边界清晰
  - **`AsyncFunction` 编译 YAML DSL**(`scenario-flow-runner.ts:90-94`)——
    **不写自己的 DSL parser**,直接用 JS 语言 — 编译产物就是 native JS function
  - **Workspace + state 文件分散设计**:`daily-ingestion.json` / `session-ingestion.json` /
    `.openclaw-wiki/state.json` 各管各 —— **不是大一统的 db,而是多个职责单一的状态文件** ——
    每个文件意图清晰,bug 隔离容易
  - **pi-sdk 三层分层 + 嵌入式 import**:
    pi-ai 统一 LLM(各家 API)/ pi-agent-core ReAct loop / pi-coding-agent 完整产品层 ——
    **每层职责单一,可独立替换** —— Hermes 自写所有层是耦合的,OpenClaw 三层各自独立
---

# OpenClaw

## 一句话定位

Personal AI assistant(TypeScript pnpm monorepo,OpenAI sponsor),Hermes 前身,**自创 Memory Dreaming + Memory Wiki(Karpathy llm-wiki 模式 production 实现)+ QA Character Eval(最完整 production agent eval)三大系统**,底层 pi-sdk 嵌入式集成不重写 agent loop,**双层循环显式拆分**(外层容错 + 内层 tool-use)。

## 起源与定位

- **背景**:Personal AI assistant 路线,OpenAI/GitHub/NVIDIA/Vercel sponsor
- **目标用户**:个人 multi-device + multi-channel(WhatsApp/Telegram/Slack/iMessage/etc 24+)
- **License**:MIT
- **是 Hermes 前身**:Hermes README 明确 `hermes claw migrate` 兼容(SOUL.md / MEMORY.md/USER.md / Skills / Command allowlist / Messaging settings / API keys / TTS assets / Workspace 全套迁移)
- **核心差异化** vs Hermes(继承者)+ Claude Code:
  - Memory Dreaming **完整** vs Hermes **简化**(`commit_memory_session` rotation 前 trigger)+ Claude Code **借鉴命名**(`autoDream` 4-phase 但无评分晋升)
  - Memory Wiki **production** vs Hermes/Claude Code **无对应** —— 是其他 harness 都没做到的
  - QA Character Eval **完整 framework** vs Hermes/Claude Code 仅 RL/ML classifier
  - 双层显式拆分 vs Hermes 单函数承担

## 核心架构(简化图)

```
            ┌────────────────────────────────────────────┐
            │ 入口: 24+ channels + 4 native apps + Gateway  │
            │      HTTP/WebSocket 双协议                  │
            └─────────────────────┬──────────────────────┘
                                  ▼
            ┌────────────────────────────────────────────┐
            │ 外层 runEmbeddedPiAgent (run.ts)            │
            │ while 32-160 retries                        │
            │ 决策树: context overflow → compaction       │
            │        auth → refresh OAuth                  │
            │        rate → advanceAuthProfile             │
            │        FailoverError → 换 fallback model     │
            │        thinking level 降级 high→low→off     │
            └─────────────────────┬──────────────────────┘
                                  ▼
            ┌────────────────────────────────────────────┐
            │ 内层 runEmbeddedAttempt (attempt.ts)        │
            │ SessionManager.run() → pi-coding-agent     │
            │ 失败直接抛                                   │
            └─────────────────────┬──────────────────────┘
                                  ▼
            ┌────────────────────────────────────────────┐
            │ pi-sdk 三层(嵌入式 import)                  │
            │ pi-coding-agent (Session+工具+Retry+Compact) │
            │ pi-agent-core (ReAct loop)                  │
            │ pi-ai (LLM API 抽象)                        │
            └────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │ Self-Evolution 双系统                                       │
  │ ┌─────────────────────────┐  ┌──────────────────────────┐ │
  │ │ Memory Dreaming         │  │ Memory Wiki              │ │
  │ │ Cron + heartbeat 触发    │  │ vault + claim + freshness│ │
  │ │ 3 phases 扫描            │  │ compile → digest         │ │
  │ │ 6 维评分晋升              │  │ 5 Agent tools            │ │
  │ │ DREAMS.md 日记 subagent  │  │ prompt 自动注入           │ │
  │ └─────────────────────────┘  └──────────────────────────┘ │
  └────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │ QA Character Eval (extensions/qa-lab/)                      │
  │ 候选并行 → Judge 排名                                         │
  │ 真实 Gateway 子进程(不 mock)+ Lab MCP server                │
  │ YAML DSL 编译 JS AsyncFunction                              │
  │ Bus 消息总线                                                  │
  └────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │ Extension-agnostic core + 30+ extensions                    │
  │ Plugin SDK npm 公开 (packages/plugin-sdk)                   │
  │ 5 horizontal cores: memory/image-gen/media/speech/video     │
  │ 30+ providers + 24+ channels + features                     │
  └────────────────────────────────────────────────────────────┘
```

## 13 维度详解

详见 frontmatter,带 `[code: file:line]` 锚点。

## 演进链(critical insight)

```
OpenClaw Memory Dreaming (2026-03 production)
       ↓ fork-继承
Hermes commit_memory_session (简化:无 Cron / 无评分 / 无日记)
       ↓ 共时
Claude Code autoDream (借鉴 "Dream" 命名,简化:无评分晋升 / 无日记 subagent)
```

OpenClaw legacy 常量证明已迭代(Light/REM 分开 → 统一)。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`:11 条借鉴 + 3 条业界印证 + 2 条 SkillForge 更前 + 6 条立项决策 + 3 条风险。

## 学习视角

详见 frontmatter `learning_takeaway`:6 条 differentiator + 6 条 strengths + 11 条 what_done_well。**核心 lesson**:OpenClaw 把"prompt 工程 + 人文关怀(梦境日记)+ systems thinking(extension-agnostic + 三层分离)" 三者结合,**是 agent 设计美学的标杆**。

## 待回答的问题(lint 会扫这一节)

- pi-coding-agent 的 Compaction 具体 5 触发点 / Phase 拆分?未读源码
- `createOpenClawCodingTools()` 完整 tool 清单?
- ACP 协议详细(`extensions/acpx`)?
- Internal Hooks + Plugin Hooks 完整列表(笔记 07 提及未读源码)
- `extensions/diagnostics-otel` OpenTelemetry 集成细节?
- Memory Wiki 的 secret scanner / guard(对照 Claude Code teamMemorySync)?
- Native apps `apps/{android,ios,macos}` 跟 Gateway 怎么交互?
- Plugin SDK 公开 npm 包的具体 API 表面?
