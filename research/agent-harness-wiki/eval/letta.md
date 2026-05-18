---
type: eval-framework
name: letta
status: active
language: python + typescript

# === 版本时效 ===
last_ingested_version: "letta 0.16.8 @ commit 1131535 (2026-05-14)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/letta]]"

# === 9 维度 ===
target: |
  **Berkeley + Letta AI 出品 memory framework + Agent OS**(Apache 2.0,**MemGPT 论文** `arxiv.org/abs/2310.08560` 原班团队)。
  跟其他 memory / framework 对照:
  - [[mem0]] = **pure memory framework**(只 memory layer)
  - **Letta = memory framework + Agent OS + coding CLI(letta-code)三合一**
  - [[../harness/claude-code]] / [[../harness/opencode]] = coding agent harness
  - [[mempalace]] = 记忆宫殿专注 long-term memory

  **学术核心**:MemGPT 论文 5 概念 — Virtual context / Memory hierarchy / Inner Thoughts / Function-call memory ops / Heartbeat。
  [verified-from-source]

dataset: |
  - **`letta/personas/` + `letta/humans/`** — MemGPT 论文 persona/human 双 profile(作为 system instruction block)
  - **`letta/data_sources/`** — RAG data sources
  - **`alembic/`** + `db/` — DB schema migrations
  - Letta API 提供 dataset 管理(SaaS,详细未深读)
  [verified-from-source]

scorer: |
  ### MemGPT 论文核心评分(memory operation 准确性)
  - **`core_memory_append / replace`** — Agent 主动调 tool 修改 core memory block,**记录每次 edit**
  - **`archival_memory_insert / search`** — Archival storage(virtual context swap out)
  - **`conversation_search`** — Recall storage(对话历史)

  ### 评测维度(`services/conversation_manager.py` + `services/context_window_calculator/`)
  - **Context window 利用率**:Letta 监控 in-context vs archival 切换决策
  - **Heartbeat 心跳**:Agent 自主驱动 vs 等用户 turn
  - **Read-only block**(`constants:READ_ONLY_BLOCK_EDIT_ERROR`)防 agent 误改 hardcoded 配置

  ### 没有内置 metric 库
  Letta 主要靠 SaaS 提供 evaluation,本仓库不含 27 metric 类(对照 [[../eval/ragas]])
  [verified-from-source]

reproducibility: |
  - **`alembic/`** + `init.sql` DB migrations
  - **`compose.yaml + dev-compose.yaml + development.compose.yml + docker-compose-vllm.yaml`** 4 个 compose 配置
  - **`certs/`** TLS certs(production 配置)
  - **`nginx.conf`** reverse proxy
  - **`fern/`** OpenAPI → Python/TS SDK 自动生成
  - **`schemas/` + `serialize_schemas/`** Pydantic schema-first
  - **`block_manager_git.py` git 版本控制 memory block** — 业内独家
  [verified-from-source]

coverage: |
  ### 5 大支柱(README + 仓库结构)
  1. **Memory layer**(MemGPT 论文学术backbone)
  2. **Agent OS**(virtual context + memory hierarchy + function call memory ops + heartbeat)
  3. **Letta Code CLI**(`@letta-ai/letta-code` npm,本地 agent + 终端,支持 skills + subagents)
  4. **Letta API SaaS**(`app.letta.com`,商业模式)
  5. **Self-host**(完整 compose stack + alembic + nginx)

  ### Multi-agent(`letta/groups/` + `services/group_manager.py`)
  对照 [[../harness/codex]] `multi_agents/` / [[../harness/claude-code]] AgentTool / [[../harness/hermes-agent]] delegate / [[../harness/opencode]] 2 builtin + `@general`

  ### Composio business tool
  `functions/composio_helpers.py` 集成 Composio(Slack / Gmail / GitHub / Linear 等 100+ business tool)

  ### 不覆盖
  - 没有内置 27 RAG metric(对照 [[../eval/ragas]])
  - 没有 LLM-as-judge metric library
  - 没有 prompt optimizer(对照 [[../eval/opik]] 7 / [[../eval/dspy]] 15)
  [verified-from-source]

production_loop: |
  ### MemGPT 论文 Agent OS loop
  ```
  User input
       ↓
  In-context memory(core memory + working context)
       ↓ context full?
  → Function call: archival_memory_insert(swap out)
  → Function call: archival_memory_search(swap in)
       ↓
  LLM generate(Inner Thoughts + tool calls + message)
       ↓
  Heartbeat trigger?(自主驱动)
       ↓
  Repeat
  ```

  ### Block manager git versioning
  Memory block 改动走 git 版本控制(`services/block_manager_git.py`)— **memory 是带版本史的实体**(对照 SkillForge MEMORY-DREAM-CONSOLIDATION 走 archived_reason)
  [verified-from-source]

automation: |
  - **`letta-code` CLI**(`@letta-ai/letta-code`)— 本地 agent + 终端
  - **Letta API SaaS**(`app.letta.com`)
  - **Python + TypeScript SDK**(`letta-client` / `@letta-ai/letta-client`)
  - **`jobs/` background jobs**
  - **Composio 100+ business tool 集成**
  - **`monitoring/` + `otel/`** OpenTelemetry
  [verified-from-source]

cost: |
  - **`services/credit_verification_service.py`** — 用量计费(Letta API SaaS 商业模式)
  - **`services/context_window_calculator/`** — Context window 计算决定 LLM cost
  - **`monitoring/`** 自家 monitoring
  - **ClickHouse OTEL trace**(`services/clickhouse_otel_traces.py`)聚合
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Letta 是 **MemGPT 论文 Agent OS 路线**,跟 SkillForge platform 路线交叉

  SkillForge 当前 memory + agent loop:
  - **MEMORY-v2 + DREAM-CONSOLIDATION + LLM-SYNTHESIS** memory 闭环
  - **server-side agent loop**(Spring + Java)
  - **CTX-1 三档触发接全量估算 + 阈值配置化 + 撞窗 retry**(2026-04-30 已交付,context window 管理)

  Letta 走的是 **agent 主动 function call 操作 memory** 路线,SkillForge 走 **后台 LLM-driven memory 提议 + 人审 gate** 路线。

  ## 可借鉴(给 SkillForge memory + agent loop 加强)

  ### 高优(1-2 周)
  1. **`block_manager_git.py` git 版本控制 memory block** — **业内独家**:memory 是带版本史的实体(每次 edit 都 commit)。SkillForge 当前 archived_reason 是 archive 视角,git 版本是 history 视角,**两条路互补**
  2. **`READ_ONLY_BLOCK_EDIT_ERROR`**(`constants:READ_ONLY_BLOCK_EDIT_ERROR`)— Read-only block 概念,**防 agent 误改 hardcoded 配置**(对照 SkillForge behavior_rule / agent system prompt 是否有 read-only 保护)
  3. **`INNER_THOUGHTS_KWARG`**(MemGPT 论文 Inner Thoughts)— Agent 内部思考独立存储,**SkillForge agent 调用 LLM 时是否分开 inner monologue 字段**?(对照当前可能合并在 messages 里)

  ### 中优(1 个月)
  4. **Heartbeat 机制**(`constants:REQ_HEARTBEAT_MESSAGE`)— Agent 自主驱动 vs 等用户 turn,**SkillForge agent 是否支持 heartbeat 主动 trigger 后台动作**?(对照 cron / scheduled task,但 heartbeat 是 LLM-driven)
  5. **`core_memory_append/replace + archival_memory_insert/search` 4 个 function call** — Agent 主动操作 memory 的标准 tool 集合,**SkillForge 是否暴露这种 memory tool 给 agent**?(对照当前 memory 是 framework 自动管理还是 agent 主动操作)
  6. **`services/context_window_calculator/`** — Context window 计算独立 service,**SkillForge CTX-1 三档触发可对照**(对照当前 CTX-1 已交付)
  7. **Composio 100+ business tool 集成**(`functions/composio_helpers.py`)— SkillForge 可借鉴 Composio 集成作为 business tool ecosystem(对照 SKILL-IMPORT marketplace 路线)

  ### 不必跟
  - **Letta Code CLI 独立产品**(SkillForge 是 server-side platform,不做 CLI 产品)
  - **MemGPT 论文 virtual context 路线**(SkillForge CTX-1 已用 3 档触发 + 全量估算 + retry,跟 virtual context swap 不同思路)
  - **`@letta-ai/letta-code` skills+subagents**(SkillForge skill 体系自家更深)

  ## SkillForge 已领先
  - **Production trace + 飞轮闭环**(Letta 没有等价 ATTRIBUTION-AGENT + canary rollout)
  - **Skill marketplace + SKILL-IMPORT/BATCH**(Letta 走 Composio business tool 路线,不是 skill 一等公民)
  - **`/evolve` meta self-evolution + 人审**(Letta 走 agent 主动 memory ops,不是 dev-facing evolve)

  ## 关键决策建议

  **`block_manager_git.py` git 版本控制 memory** 是 actionable 借鉴第一名:
  - SkillForge 当前 MEMORY-DREAM-CONSOLIDATION 走 archived_reason
  - 加 git-style 版本(每次 edit 一个 commit)+ blame(谁改的 + 何时)能力
  - 工程量:中(SQLite 或 Postgres 加 memory_history 表 + 每次 update 写 history row)

  **`READ_ONLY_BLOCK` read-only memory** 是第二:
  - SkillForge behavior_rule / agent system prompt / hardcoded user preference 应有 read-only 保护
  - Letta 给 `READ_ONLY_BLOCK_EDIT_ERROR` 是工程参考

  **MemGPT 学术参考价值高**(Berkeley + ICLR 2024):**架构学**,代码不直接借鉴(SkillForge Java/Spring 跟 Letta Python/FastAPI 栈不同)

learning_takeaway: |
  ## 跟其他 memory framework 不同的是 (differentiator)
  1. **MemGPT 论文学术 backbone**(Berkeley,Charles Packer + Sarah Wooders et al.,arxiv 2310.08560)
  2. **Agent OS 定位**:不只是 memory layer,而是"LLM as Operating System"完整 OS-like 抽象
  3. **Virtual context**:Context window 不够时 swap memory(类 OS virtual memory)— 业内独家学术原创
  4. **Memory hierarchy**:Core memory + working context + recall storage + archival storage 4 层
  5. **Inner Thoughts / Inner Monologue**(`INNER_THOUGHTS_KWARG`)— 业内独家 LLM 内部思考独立字段
  6. **Function-call memory ops**(`core_memory_append/replace` / `archival_memory_insert/search` / `conversation_search`)— Agent 主动调 tool 操作 memory(对照 mem0 framework 自动 add)
  7. **Heartbeat 机制**(`REQ_HEARTBEAT_MESSAGE`)— Agent 自主驱动后台思考,**业内独家**
  8. **Read-only block**(`READ_ONLY_BLOCK_EDIT_ERROR`)— Memory block 可 read-only,业内独家
  9. **Git 版本控制 memory block**(`services/block_manager_git.py`)— 业内独家
  10. **MemGPT formerly → Letta rebrand**(2024)— 学术项目商业化
  11. **Letta Code CLI**(`@letta-ai/letta-code`)+ Letta API SaaS + Self-host 三产品线
  12. **`personas/` + `humans/` 双 profile** — User 跟 Agent 都有人格(MemGPT 论文)

  ## 优势 (strengths)
  1. **学术 backbone 强**:MemGPT paper 2023-10,ICLR 2024 引用
  2. **Python 3.12 + FastAPI + alembic + ClickHouse OTEL** 现代 Python stack
  3. **`fern/` OpenAPI SDK 自动生成**(对照 Langfuse)
  4. **Composio business tool 100+ 集成** — 不只 LLM,业务系统 ecosystem
  5. **`services/` 18 service module 拆分清晰**(archive / block / conversation / file_processor / lettuce / group / identity / job / credit / clickhouse_otel)
  6. **`compose.yaml + dev / development / vllm` 4 配置** — 多场景部署
  7. **`certs/` + `nginx.conf`** — production 完整(对照早期项目无 nginx 配置)
  8. **CITATION.cff** — 学术引用一等公民
  9. **`monitoring/` + `otel/` + `clickhouse_otel_traces.py`** — 自家可观测完整
  10. **`block_manager_git.py`** git-based memory versioning — 工程创新

  ## 哪些点做得好 (what_done_well)
  1. **`READ_ONLY_BLOCK_EDIT_ERROR` constant 明确命名** — 一眼看出意图(防 agent 误改)
  2. **`INNER_THOUGHTS_KWARG` 独立字段** — LLM 内部思考跟 user-facing message 拆开,**对话 + 推理双 channel**
  3. **`FIRST_MESSAGE_ATTEMPTS + REQ_HEARTBEAT_MESSAGE + FUNC_FAILED_HEARTBEAT_MESSAGE`** 3 个 heartbeat 常量 — 心跳机制 3 个状态
  4. **`LETTA_CORE_TOOL_MODULE_NAME + LETTA_MULTI_AGENT_TOOL_MODULE_NAME`** 2 个 tool module — 单 agent + multi-agent 分开
  5. **`block_manager.py` + `block_manager_git.py` 两个 manager** — 主 manager + git 版本独立,concern 分离
  6. **`services/lettuce/`** 命名俏皮(letta + utility 双关)
  7. **`personas/` + `humans/` 拆分** — User 跟 Agent 人格独立,**不混用同一 profile**
  8. **`local_llm/` 独立子模块** — 本地 LLM(Ollama / vLLM)跟 API LLM 拆开
  9. **`adapters/` + `interfaces/` 双层抽象** — Adapter pattern 工程化
  10. **`agent_serialization_manager.py`** — Agent 序列化独立 service,完整 state 持久化
---

# Letta(formerly MemGPT,Berkeley + Letta AI,Apache 2.0,0.16.8)

## 一句话定位

**MemGPT 论文学术 backbone + Agent OS 完整实现**。Virtual context + Memory hierarchy(core/working/recall/archival)+ Inner Thoughts + Function-call memory ops + Heartbeat + Git 版本控制 memory block(业内独家)。3 产品线:Letta Code CLI / Letta API SaaS / Self-host。

## MemGPT 论文 5 大概念

| # | 概念 | 实现 |
|---|---|---|
| 1 | **Virtual context** | Context window 不够时 swap memory(`services/context_window_calculator/`)|
| 2 | **Memory hierarchy** | Core + working context + recall + archival 4 层 |
| 3 | **Inner Thoughts** | `INNER_THOUGHTS_KWARG` 独立字段(`local_llm/constants.py`) |
| 4 | **Function-call memory ops** | `core_memory_append/replace` + `archival_memory_insert/search` + `conversation_search` |
| 5 | **Heartbeat** | `REQ_HEARTBEAT_MESSAGE / FUNC_FAILED_HEARTBEAT_MESSAGE / FIRST_MESSAGE_ATTEMPTS` |

## 跟其他 memory framework 对照

| 项目 | 路线 | 关键 |
|---|---|---|
| **mem0** | Single-pass ADD-only + multi-signal retrieval | 累加 + 检索时筛选 |
| **Letta** | MemGPT Agent OS + function-call memory ops | Agent 主动 memory 操作 |
| **MemPalace** | Verbatim + hierarchical wing/room/drawer | LongMemEval 业内最高 |
| **SkillForge** | DREAM-CONSOLIDATION + LLM-SYNTHESIS + 人审 | dedup + LLM 提议 + 人审 |

## 3 大 SkillForge actionable 借鉴

1. **`block_manager_git.py` git 版本控制 memory** — SkillForge memory 加 git-style history(每次 edit commit)
2. **`READ_ONLY_BLOCK_EDIT_ERROR`** read-only memory — SkillForge behavior_rule / agent system prompt 应有 read-only 保护
3. **`INNER_THOUGHTS_KWARG`** inner monologue 独立字段 — SkillForge agent LLM call 是否拆 inner_thoughts vs user_message?

## 待回答
- `lettuce/` 子系统具体是什么
- `block_manager_git.py` 完整 git 集成实现
- Composio integration 跟 SkillForge SKILL-IMPORT 关系
- Heartbeat 怎么避免无限 loop(对照 SkillForge ANTI-RUNAWAY 思路)
- MemGPT 论文 → Letta 产品化的工程提升点
