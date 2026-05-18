---
name: agentscope
type: harness
status: active
language: python
last_ingested_version: 1.0.20
last_ingested_at: 2026-05-17
last_commit: b840fa927ad409aeddb9df359631ed586f2ae830
community_known_newer: false
known_post_ingest_features:
  - "AgentScope 2.0 roadmap (2026-04 Discussion #1441) — major API revision pending"

sources:
  - "[[source/agentscope]]"
  - "/Users/youren/myspace/agentscope/"
  - "https://arxiv.org/abs/2402.14034"

agent_loop: |
  `ReActAgent` (`agent/_react_agent.py` 1137 行) async streaming.
  Model generation + tool execution 双层 interrupt 协议 (`asyncio.CancelledError`).
  Chunk-level `is_interrupted` flag (`:698`) + in-flight tool_call fake-result 补全 (`:633-654`)
  保证 resume 时 history 无孤儿 tool_use.

tool_protocol: |
  `Toolkit.register_tool_function(func)` 通用注册 (JSON tool_use).
  **MCP-as-local-function 独家** (`mcp/_mcp_function.py` + `HttpStatelessClient.get_callable_function`) —
  把 MCP server tool 抽出为 Python callable,可单独调用 / 注册 toolkit / 包成 high-level tool 三选一.
  Multi-modality / coding / text_file tool 内置子目录.

context_mgmt: |
  **SummarySchema** (`_react_agent.py:43-87`) — 5 字段 pydantic 强类型压缩:
  task_overview / current_state / important_discoveries / next_steps / context_to_preserve,
  each Field(max_length=200-300, description=...). 业内罕见强类型 (vs Claude Code 自由文本 summary).
  Memory compression module (2026-01 ship).

memory: |
  **双层显式分离** (`memory/_long_term_memory/` + `_working_memory/`) — 业内罕见.
  `LongTermMemoryBase` (`_long_term_memory_base.py` 94 行) 抽象 + `InMemoryMemory` 工作记忆.
  ReMe 集成 (2025-11) 作为 long-term 一种实现.
  跟 RAG `KnowledgeBase` 分离 (KB 是 retrievable doc store, memory 是 conversation history).

multi_agent: |
  **MsgHub** (`pipeline/_msghub.py`) — 多 agent 消息总线, 配合 sequential_pipeline / chat_room.
  **A2A protocol** (`a2a/` 5 文件 + `_nacos_resolver.py` Alibaba 服务发现) — 跨进程 / 跨主机 agent 通信开放标准.
  A2A formatter (`formatter/_a2a_formatter.py`) message-level 协议适配.
  跟 Codex actor + mailbox 是 in-process / Claude Code SubAgent 是 in-process,
  AgentScope A2A 是 **跨进程 / 跨主机** — 不同 isolation 边界.

eval_loop: |
  内置 `evaluate/` 完整 framework:
  `_benchmark_base.py` + `_evaluator/` + `_evaluator_storage/` + `_metric_base.py` + `_solution.py` + `_task.py`.
  **`_ace_benchmark/` 内置 benchmark suite** (ACE = AgentScope eval).
  跟 ragas / promptfoo 等独立 eval 框架不同 — 内嵌于 agent 框架本身.

self_evolution: |
  **Trinity-RFT** (`tuner/` 10 文件 + Trinity model provider) — 业内独家把 agentic RL training 收编进 agent 框架.
  `_algorithm.py` / `_dataset.py` / `_judge.py` (LLM-as-judge) / `_model.py` / `_tune.py` / `_workflow.py` +
  `model_selection/` + `prompt_tune/` 子目录.
  6 个 production sample (Math 75→85% / Frozen Lake 15→86% / Werewolf 50→80% 等,真实数字).
  跟 Hermes/Claude Code 的 "memory dreaming / dedup / archive" self-evolution 完全不同方向 —
  AgentScope 是 **改 model weights**, Hermes/CC 是 **改 memory / rule / agent definition**.

observability: |
  OTel-based tracing (`tracing/` 7 files).
  6 decorator: `trace / trace_llm / trace_reply / trace_format / trace_toolkit / trace_embedding`
  覆盖 LLM call + agent reply + formatter + toolkit + embedding 5 类 span.
  跟 SkillForge OBS-2 (LLM-only span) 互补 — AgentScope 在 framework 层 instrument.

hooks: |
  **极简 1 文件** `hooks/_studio_hooks.py` (Studio 是 AgentScope 自家 IDE / 可视化工具).
  vs Claude Code 27 hook events / opencode 27 — 显著少.
  PlanNotebook 自带 `register_plan_change_hook` (plan-domain-specific).
  印证 [[concept/hooks-design]] "hooks 数量不必衡量成熟度" 观察.

safety: |
  ? 未 verify. 需 grep "safety|guardrail|prompt_injection|sandbox".
  跟 Codex 3 平台 sandbox / Claude Code 5 permission mode + YOLO ML 对比未完成.

provider: |
  **6 model + Trinity** (`model/`):
  Anthropic (Claude) / DashScope (Qwen, **Alibaba 自家**) / Gemini / Ollama / OpenAI / **Trinity** (RL training model).
  7 provider-specific formatter + truncated_formatter_base 截断基类.

channels: |
  **Realtime voice (`realtime/`) + TTS (`tts/`)** — 业内罕见 voice first-class.
  ReActAgent 直接含 TTS context manager + 流式 audio synthesis (`_react_agent.py:578-617`).
  Web UI (Studio).
  CLI (examples 跑法).
  跨进程 A2A.

persistence: |
  **4 session backend** (`session/`):
  - `_json_session.py` — local JSON
  - `_redis_session.py` — Redis
  - **`_tablestore_session.py`** — Alibaba Cloud Tablestore (NoSQL serverless)
  - `_session_base.py` — abstract base
  统一接口: `save_session_state(session_id, user_id, **state_modules_mapping: StateModule)`.
  StateModule 抽象 (`module/_state_module.py:20-152`) 让任意 stateful 模块组合都能序列化.

cancellation: |
  **业内最完整 interrupt 协议**:
  1. `asyncio.CancelledError` 作为信号
  2. Chunk-level `is_interrupted` flag (`_react_agent.py:698`) — model generation 和 tool execution 都可被中断
  3. Interrupt 后 **fake ToolResultBlock 补全 memory** (`:639-653` "The tool call has been interrupted by the user.") — 防 resume 时 history 孤儿
  4. `handle_interrupt()` async post-processing (`:799-816`) 含 LLM-friendly 续接话术
  跟 Codex `resume_agent` + Claude Code Esc cancel 形成三家路线对照.

skillforge_takeaway: |
  4 大可借鉴 + 1 大决策候选:

  🔴 **R-AS-1 StateModule 抽象统一持久化**(业内最干净)— SkillForge `SubAgentRunEntity` + `MemoryEntity` + `t_session` + 各 ToolUseRecord 当前各自序列化,
  可借鉴 StateModule PyTorch nn.Module 风格统一抽象 + custom_to_json / custom_from_json per-attr 适配.
  ROI: cross-environment migration / A/B testing / 时间旅行 debug 同一套抽象支撑.

  🔴 **R-AS-2 Interrupt fake-tool-result 补全**(直接打到生产)—
  SkillForge `SubAgentStartupRecovery.java:35-95` 现有 crash recovery 处理 RUNNING → CANCELLED 转换,
  但 in-session Esc cancel 时 in-flight tool_call 怎么补 fake result 让 LLM resume 时不蒙未 verify.
  AgentScope `_react_agent.py:639-653` 给了精确模板.

  🟡 **R-AS-3 SummarySchema 强类型压缩**(防 hallucination)—
  SkillForge V21 CONTEXT-COMPACT 当前自由文本 summary, 可加 pydantic 5 字段 schema
  (task_overview / current_state / important_discoveries / next_steps / context_to_preserve)
  + max_length + description LLM prompt. 工作量 < 半周.

  🟡 **R-AS-4 Memory long_term + working 显式分离**(架构清晰)—
  SkillForge MemoryEntity 三态 (observation/reflection/optimized) 是按 lifecycle 分,
  AgentScope 是按 lifetime scope 分. 两者正交可叠加. 工作量 = 加一个 dimension flag.

  🟢 **决策候选: Trinity-RFT agentic RL training** —
  SkillForge 远期是否走 "production trace → RL fine-tune candidate agent" 路径? 
  当前 SkillForge LLM-SYNTHESIS + 人审 gate 是 self-evolution V1-V2,
  Trinity-RFT 是 V3 候选(改 model weights vs 改 memory/rule).
  风险: 需 model training infra + Qwen / open weights provider,SkillForge 当前 Claude/OpenAI 为主.
  建议先观察 production benchmark,不优先做.

  **❌ 不该跟**:
  - A2A + Nacos 跨进程 agent — SaaS 单进程暂不需要
  - TTS / realtime voice — SkillForge 文本主线,与产品定位不符
  - Studio 可视化 IDE — SkillForge dashboard 已是 React/Vue 路线
---

# Harness — AgentScope 1.0.20

> 13 维详见 frontmatter. 本页 narrative 重点:**StateModule 抽象 + interrupt 协议 + Trinity-RFT** 是 AgentScope 三大独家点,
> 远比 "Alibaba 出的 multi-agent 框架" 这种表面描述精确 — 跟 5 个已 ingest harness 横向有显著差异化设计选择.

---

## 1. 项目定位(一句话)

**Alibaba 通义实验室生产级 Python agent 框架**,围绕 `StateModule` PyTorch 风格抽象 + 4 backend session 持久化 + Trinity-RFT 训练闭环 + A2A 跨进程协议设计,Apache 2.0 / 25.2k stars / arxiv 2402.14034 学术 backbone.

---

## 2. 三大独家技术点 — 横向 5 harness 对照

### 2.1 StateModule 抽象(`module/_state_module.py:20-152`)

| | AgentScope | Claude Code | Codex | Hermes | OpenClaw | opencode |
|---|---|---|---|---|---|---|
| **agent state 统一抽象** | ✅ `StateModule` (nn.Module 风格 + nested + custom JSON 转换) | ⚠️ 散在 Session/Memory/Subagent 各自类 | ⚠️ Rust actor 内部 mailbox state | ⚠️ Python dict 散管 | ⚠️ 各 ext 自管 | ⚠️ Effect-TS Schema 类型化但无统一抽象 |
| **per-attribute JSON 序列化** | ✅ `custom_to_json / custom_from_json` | ❌ | ❌ | ❌ | ❌ | ✅ (Effect-TS Schema decoder/encoder) |
| **持久化 backend** | **4**: JSON / Redis / Tablestore / abstract | 1: 本地文件 | 1+: rollout JSONL + SQLite | 1: 本地 | 1: Drizzle/SQLite | 1: Drizzle/SQLite |

**为什么这是独家**:Python 框架做 `nn.Module` 风格 state 抽象的人极少 — 大部分 Python agent 框架(LangChain / AutoGen / CrewAI)用 Pydantic schema 或纯 dict,**没有自动追踪 sub-module + 递归 state_dict** 这套.

详见 [[concept/agent-state-separation]] 深度分析.

### 2.2 Interrupt 协议(`_react_agent.py:575-655, 687-699, 799-816`)

业内最完整 5 步:

1. **信号统一**:`asyncio.CancelledError` 作为 cancel 信号
2. **双层检测**:model generation streaming(`:586` async for content_chunk)+ tool execution chunk(`:688-699` async for chunk)都监听 cancel
3. **chunk 自带 flag**:`chunk.is_interrupted` (`:698`)— 流式 tool 输出途中可中断
4. **fake result 补全**(`:639-653`):in-flight tool_call 自动注入 `"The tool call has been interrupted by the user."` ToolResultBlock
5. **handle_interrupt 话术续接**(`:799-816`):提供 LLM-friendly continuation `"I noticed that you have interrupted me. What can I..."` 

**对照**:
- Codex `resume_agent` — 有 cross-agent pause/resume,but in-loop chunk interrupt 未确认
- Claude Code Esc cancel — 有 UI 层取消,fake tool result 补全 in-message 未 verify
- Hermes — 单 LLM 流式取消粗粒度

**对 SkillForge 启示**:`SubAgentStartupRecovery.java` 是服务重启 crash recovery,**in-session user-initiated cancel 是否补 fake tool_result 给 LLM 让 history 不孤儿** 是 R-AS-2 verify 重点.

### 2.3 Trinity-RFT(`tuner/` + `model/_trinity_model.py`)

业内独家把 **agentic RL training pipeline** 收编进 agent 框架:

- `_algorithm.py` RL 算法层
- `_judge.py` LLM-as-judge 评分(给 reward signal)
- `_dataset.py` training dataset 抽象
- `_workflow.py` 训练流程编排
- `model_selection/` + `prompt_tune/` 两条优化轴
- **Trinity model provider**(`_trinity_model.py`)— model 可同时处于 inference + RL training 模式

**6 个 sample 真实数字**(README):
- Math (Qwen3-0.6B):75% → 85%
- Frozen Lake (Qwen2.5-3B):15% → 86%(+71pp)
- Learn to Ask (Qwen2.5-7B):47% → 92%(+45pp)
- Werewolf (Qwen2.5-7B):win rate 50% → 80%
- AIME-24 (Qwen3-0.6B):20% → 60%(+40pp)

**对照** self-evolution 路线 4 家:
1. AgentScope = **改 model weights**(RL fine-tune)
2. Hermes / OpenClaw = **改 memory**(commit_memory_session + memory dreaming)
3. Claude Code = **改 agent skills / rules**(autoDream + rule synthesis)
4. SkillForge = **改 memory + skill + agent + rule 四件套**(MEMORY-LLM-SYNTHESIS + 人审 gate)

四条路线**正交可叠加**,详见 [[concept/self-evolution]](待扩展加 AgentScope 路线)。

---

## 3. 模块结构 narrative(26 模块 → 4 大职责)

```
              ┌────────────────────────┐
              │   StateModule 核心     │  module/
              │  (nn.Module for state) │
              └───────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼────┐      ┌─────▼─────┐    ┌─────▼─────┐
   │  Agent  │      │  Memory   │    │   Plan    │
   │ ReAct + │      │ Long+Work │    │ Notebook  │
   │ A2A +   │      │ ReMe 集成 │    │ 904 行    │
   │Realtime │      └───────────┘    └───────────┘
   └────┬────┘
        │
        ├──── Model (6 provider + Trinity)
        ├──── Formatter (7 per-provider)
        ├──── Tool + MCP-as-local-function
        ├──── RAG (KnowledgeBase)
        ├──── TTS + Realtime
        └──── Tracing (OTel)

   ┌────────────────────────────────────────┐
   │  Pipeline (MsgHub + chat_room)         │  多 agent 协调
   │  A2A (5 files + Nacos resolver)        │  跨进程协议
   └────────────────────────────────────────┘

   ┌────────────────────────────────────────┐
   │  Session (JSON/Redis/Tablestore)       │  4 backend 持久化
   │  Evaluate (_ace_benchmark)             │  内置评测
   │  Tuner (Trinity-RFT)                   │  RL 训练闭环
   └────────────────────────────────────────┘
```

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[harness/codex]] | 印证 + 校准 | Codex `resume_agent` + actor mailbox 是 in-process 路线;AgentScope A2A + Nacos 是 **跨进程**路线 — 两套不冲突 |
| [[concept/multi-agent-isolation]] | 扩展 | 新增 "跨进程隔离" 维度(in-process vs cross-process vs cross-host) |
| [[concept/agent-memory]] | 印证 + 扩展 | long_term + working 显式分离是 5 路线之外的第 6 个 dimension(scope-based) |
| [[concept/self-evolution]] | 重大扩展 | Trinity-RFT 是 V4 self-evolution 路线("改 weights" vs "改 memory/rule/skill") |
| [[concept/hooks-design]] | 印证 | 极简 1 hook 文件再次印证 "hooks 数量不衡量成熟度" |

---

## 5. 借鉴清单(R-AS-1~R-AS-4 + 决策候选)

详见 `skillforge_takeaway` frontmatter. 总结:

| ID | 优先级 | 工作量 | ROI |
|---|---|---|---|
| **R-AS-1** StateModule 抽象 | 🔴 P1 | 2-4 周 | 高 — 撬动 A/B / 跨环境迁移 / 时间旅行 debug |
| **R-AS-2** Interrupt fake-tool-result | 🔴 P1 | < 1 周 | 高 — 直接打到生产 (verify 后) |
| **R-AS-3** SummarySchema 强类型 | 🟡 P2 | < 半周 | 中 — 防 compaction hallucination |
| **R-AS-4** Memory long+work 双层 | 🟡 P2 | < 半周 | 中 — 架构清晰 |
| **决策** Trinity-RFT agentic RL | 🟢 远期 | 数月 | 待观察 — V4 self-evolution 候选 |

---

## 6. 已知 post-ingest 风险

- **AgentScope 2.0 roadmap**(2026-04 Discussion #1441)预告大改 API,本次 ingest 是 1.0.20 — 2026 下半年需 re-verify
- Anthropic Agent Skill 支持(2025-11)未深入 verify SkillForge 对照 Anthropic Skill 标准
- `_studio_hooks.py` 内部细节未读,可能含 Studio IDE-specific 限制

---

## 7. 引用

- [[source/agentscope]] 完整源码锚点
- [[concept/agent-state-separation]] depth-first 概念深挖
- arxiv 2402.14034 学术 backbone
- GitHub: `agentscope-ai/agentscope` 25.2k stars / Apache 2.0
