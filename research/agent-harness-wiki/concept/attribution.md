---
type: concept
name: attribution
status: draft
sources:
  - "[[../eval/phoenix]]"
  - "[[../eval/tau-bench]]"
  - "[[../eval/agentops]]"
  - "[[../eval/zep-graphiti]]"
  - "[[../eval/mempalace]]"
  - "[[../eval/promptfoo]]"
  - "[[../eval/dspy]]"
  - "[[../eval/helicone]]"
related_harnesses: []
related_evals: [phoenix, tau-bench, agentops, zep-graphiti, mempalace, promptfoo, dspy, helicone, langfuse, opik]
---

# Attribution — Agent 归因专题(5 Layer + 7 步流程)

> 把 wiki 已 ingest 的 8+ 个项目的归因实现整合,形成完整 attribution landscape。SkillForge ATTRIBUTION-AGENT(飞轮 V3 + V80 完整 state machine,已交付 2026-05-15)在最深一层(Layer E),业内 OSS 没有等价产品。

## 定义

**Agent Attribution** = 从 agent 行为结果反推**为什么会这样**:
- **失败原因**:这条 trace 失败,因为 LLM 想错了 / tool 调错了 / context 缺了 / prompt 模糊了?
- **改进归因**:这个 metric 变好,因为 prompt 改了 / skill 新版本上线 / behavior_rule 加了?
- **影响归因**:这条 fact 是哪次对话产生的?后续被哪条 superseded?

跟 [[../concept/observability]] 关系:
- **Observability** = **收集** trace / span / 事件(数据)
- **Attribution** = **解读** trace / 反推因果(意义)
- 两者协作:**observability 是 attribution 的 raw materials**

## 5 Layer Attribution Landscape

```
┌──────────────────────────────────────────────────────────────────┐
│ Layer E: Production 自动 / 用户驱动归因 + actionable             │
│  → 2 条路线:                                                       │
│    路线 A (自动 + canary): SkillForge V3+V80(业内独家)          │
│    路线 B (用户 review):  ⭐ Claude Code /insights(2026-05-17 加) │
│  → 部分 Layer E 候选:                                              │
│    - AgentScope Trinity-RFT(改 model weights 路线)              │
│    - MetaGPT AFlow(改 workflow 路线 ICLR'25 oral)                │
│    - Cognee memify_pipelines(memory feedback loop)              │
│    - DSPy GEPA + Opik Optimizer(改 prompt algorithm)            │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │
┌──────────────────────────────────────────────────────────────────┐
│ Layer D: Eval-output 级 fault 归因(typed enum)                  │
│  → 单次失败 → LLM 分析 → fault author / fault type / description │
│  → tau-bench / DSPy / Promptfoo trace-level assertion             │
│  → AgentOps `AGENT_THINKING/ACTION/DECISION` 三态                 │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │
┌──────────────────────────────────────────────────────────────────┐
│ Layer C: Memory / KG 级 fact 归因(provenance)                  │
│  → 每条 fact 追溯到产生它的 episode(raw conversation)           │
│  → Validity windows(fact 何时为真 / 何时被覆盖)                 │
│  → Graphiti / MemPalace / mem0(v2 隐式)                         │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │
┌──────────────────────────────────────────────────────────────────┐
│ Layer B: Per-evaluation rationale(score-level explanation)      │
│  → LLM-as-judge 输出 score + LLM-generated rationale             │
│  → Phoenix Score.explanation field 默认开                        │
│  → Ragas AspectCritic / Opik Assertion / Promptfoo llmRubric    │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │
┌──────────────────────────────────────────────────────────────────┐
│ Layer A: Trace + 人工探索归因(observability UI)                │
│  → 收集 trace + span + 事件,人工在 UI 看 trace tree            │
│  → AgentOps(agent-level)/ Phoenix / Langfuse / Opik(LLM-level)│
│  → Helicone(Gateway-first)                                       │
└──────────────────────────────────────────────────────────────────┘
```

## 7 步完整归因流程(从 Layer A 到 Layer E)

```
[Step 1] Trace 收集(Observability,Layer A 基础)
   ↓
[Step 2a] 单次归因(短链路)──┐
   ↓                       │
   ✅ 简单场景到这就够        │
   ✅ tau-bench / Phoenix    │
                            ▼
                  [Step 2b] 自动标注(长链路起点)
                            ↓
                            生产 session 自动 outcome 标注
                            (signal-based + LLM + human override)
                            ↓
                       [Step 3] 跨 session 聚类(找 pattern)
                            ↓
                            按 failure_class × tool × agent × skill 桶聚
                            ↓
                       [Step 4] 归因决策层(LLM-driven)
                            ↓
                            决策"该改哪个 surface,改成什么"
                            ↓
                       [Step 5] Candidate 生成(multi-surface)
                            ↓
                            skill / prompt / behavior_rule / hook 各自生成新版本
                            ↓
                       [Step 6] A/B + Canary 灰度
                            ↓
                            离线 A/B + 生产灰度回流
                            ↓
                       [Step 7] 验证 + 回灌
                            ↓
                            生产指标对比 → promote / rollback / verify
```

## 各 Layer 详细实现(wiki ingest verified)

### Layer A:Trace observability + 人工探索

#### AgentOps(agent-level OTel)— [[../eval/agentops]]

**[verified-from-source]** `agentops/semconv/span_kinds.py`:

```python
# 12 SpanKind(:7-19)
WORKFLOW / SESSION / TASK / OPERATION / AGENT / TOOL / LLM /
CHAIN / TEXT / GUARDRAIL / HTTP / UNKNOWN

# Legacy 3 sub-kind(:25-27)— ⭐ 业内独家 agent action 三态归因
AGENT_THINKING  # agent 想错了
AGENT_ACTION    # agent 做错了
AGENT_DECISION  # agent 决策错了
```

**4 装饰器强制 agent 语义层级**:`@session → @agent → @operation → @task`。

**OpenTelemetry 完整集成**(`instrumentation/OpenTelemetry.md`):
- `weakref.WeakKeyDictionary()` 存 span context
- Explicit parent context propagation
- `_get_parent_context(trace_obj)` helper

#### Phoenix(LLM-level OpenInference)— [[../eval/phoenix]]

**[verified-from-source]** `app/src/pages/trace/` 13 trace UI 组件:
- `ConnectedTraceTree.tsx`(视觉化 trace 树)
- `TraceFeedbackActionToolbar.tsx`(人工标注)
- `TraceHeaderRootSpanAnnotations.tsx`(root span 注释)
- **Phoenix Agent UI assistant**:用户在 trace UI 跟 AI 对话查 root cause(半自动)

#### Langfuse / Opik / Helicone

详见各 eval 页。共同点:**收集 + UI + 人工探索**,**没有自动归因决策**。

### Layer B:Per-evaluation rationale

#### Phoenix Score.explanation(业内最规范)

**[verified-from-source]** `packages/phoenix-evals/.../evaluators.py:159+`:

```python
@dataclass(frozen=True, init=False)
class Score:
    score: float | None
    name: str
    label: str | None
    explanation: str | None        # ⭐ LLM-generated rationale
    metadata: dict
    kind: KindType                 # "human" | "llm" | "heuristic" | "code"
    direction: DirectionType       # "maximize" | "minimize" | "neutral"
```

**`include_explanation: bool = True` 默认开**(`evaluators.py:720`)— **多 30% token cost 但归因输入更丰富**。

#### Ragas AspectCritic — [[../eval/ragas]]

`_aspect_critic.py` — 给 aspect description(e.g. "tone is professional")→ LLM 输出 yes/no/maybe + reason。

#### Opik Assertion / Promptfoo llmRubric / DSPy

各 framework 都把 LLM-as-judge rationale 作为输出。

### Layer C:Memory / KG provenance

#### Graphiti Episodes + Validity windows — [[../eval/zep-graphiti]]

**[verified-from-source]** Graphiti 0.29.0 arxiv 2501.13956:

```python
class Edge:  # (推断,基于 README + graphiti_core/edges.py)
    source_node_id: str
    target_node_id: str
    fact: str
    valid_from: datetime      # ⭐ fact 何时为真
    valid_to: datetime | None # ⭐ fact 何时被 supersede
    source_episode_id: str    # ⭐ 追溯到产生 fact 的 raw data
```

**核心概念**:Episode(raw conversation / document / event)→ LLM extract entities + relationships + validity windows → Graph。**每个 fact 都可 audit**。

#### MemPalace Verbatim + Local KG — [[../eval/mempalace]]

`mempalace/knowledge_graph.py` — SQLite-backed Temporal KG + validity windows + invalidate + timeline。Verbatim 存储所有原文(终极 provenance)。

#### mem0 v2(隐式时间戳)— [[../source/mem0]]

`mem0/memory/main.py` ADD-only 累加 + Temporal Reasoning(查询时间感知)— **没有显式 validity windows**,靠 retrieval 时按时间排序 + LLM 推断当前事实。

### Layer D:Eval-output fault 归因(typed enum)

#### tau-bench `auto_error_identification.py`(业内首批)— [[../eval/tau-bench]]

**[verified-from-source]** `auto_error_identification.py:31-66`:

```python
class FaultAuthor(Enum):                       # ⭐ 3 种归因主体
    USER = "user"
    AGENT = "agent"
    ENVIRONMENT = "environment"

class FaultType(Enum):                          # ⭐ 4 类错误
    CALLED_WRONG_TOOL = "called_wrong_tool"
    USED_WRONG_TOOL_ARGUMENT = "used_wrong_tool_argument"
    GOAL_PARTIALLY_COMPLETED = "goal_partially_completed"
    OTHER = "other"

class FaultAssignmentResult(BaseModel):
    task_id: int
    author: FaultAuthor                         # typed enum
    description: str                            # LLM free-form

class GradingStrategy(Enum):                    # ⭐ 2 种 grading 视角
    ACTIONS = "actions"                         # 比对 action sequence
    OUTPUTS = "outputs"                         # 比对 output keys
```

**LLM-driven 离线归因**:把 user instruction + ground truth + trajectory → LLM → `FaultAssignmentResult`。

#### Promptfoo Trace-level Assertion(独家)— [[../eval/promptfoo]]

**[verified-from-source]** `src/assertions/`:
- `traceErrorSpans.ts` — trace 中 error span 数 ≤ X
- `traceSpanCount.ts` — trace 总 span 数在区间
- `traceSpanDuration.ts` — span 总耗时上限

**业内独家:trace span 直接当 assertion 输入做归因**。

#### DSPy GEPA Reflective Prompt Evolution — [[../eval/dspy]]

GEPA(2025-07 arxiv 2507.19457)— Reflective prompt evolution 隐含归因(LLM 反思 prompt 哪里改得不好)。

#### AgentOps SpanKind 归因维度 — [[../eval/agentops]]

`AGENT_THINKING / AGENT_ACTION / AGENT_DECISION` 三态(Legacy SpanKind):**agent 想错 vs 做错 vs 决策错** — 跟 tau-bench `FaultType` 维度互补:

| Framework | 归因维度 | 视角 |
|---|---|---|
| **tau-bench** | `FaultAuthor × FaultType` 3×4 | 谁错了 × 错什么 |
| **AgentOps** | `AGENT_THINKING / ACTION / DECISION` 3 | 在做什么 |
| **组合** | tau × AgentOps = 完整 |

### Layer E:Production 自动归因 + 用户驱动归因 — 2 条互补路线

**2026-05-17 校准**:Layer E 不是 SkillForge 唯一独家。**Claude Code `/insights`** 走另一条 _"用户驱动 review"_ 路线,**Anthropic 同时 ship 两条 self-improvement 路径**(`/insights` 用户驱动 + autoDream 自动后台)。SkillForge 独家在 _"自动 multi-surface + canary + verify 完整闭环"_。

#### 路线 A — SkillForge V3 + V80 自动 canary 闭环(改 system 内部 surface)— [verified-from-source]

完整 5 步飞轮(详见 [[concept/agent-memory]] 校准章节):

```
① 标注(SessionAnnotationService → t_session_annotation)
   signal-based(Smart Import 6 类 reason)+ llm-based + human override
   ↓
② 聚类(PatternClusteringService → t_session_pattern)
   按 failure_class × tool × agent × skill 桶聚
   ↓
③ 归因 agent(AttributionAgent sub-agent)
   LLM-driven 决策"该改哪个 surface,改成什么"
   ↓
④ Candidate(V4 multi-surface)
   OptimizableSurface<V> + AbstractAbEvalRunner<V> Template Method
   ↓
⑤ A/B + Canary(V2 SKILL-CANARY-ROLLOUT)
   生产灰度 + 指标回流
   ↓
⑥ 验证 + 回灌(V80 state machine)
```

**V80 完整 state machine**(`V80__create_optimization_event.sql`):

```
proposal_pending
   → proposal_approved / proposal_rejected
   → candidate_created
   → ab_running
   → ab_passed / ab_failed
   → canary_started
   → promoted / rolled_back
   → verified
```

**关键字段**(verified):
- `surface_type`:skill / prompt / behavior_rule(V3 + V4 multi-surface)
- `confidence DECIMAL(3,2) + CHECK 0..1` — agent **自我 reject at < 0.5**
- `cooldown_expires_at TIMESTAMPTZ` — 24h cooldown 防同 pattern 重复触发
- `attribution_session_id VARCHAR(36)` — 链接 curator agent session
- `candidate_skill_id` + `candidate_prompt_version_id` + `ab_run_id` + `canary_id` — 各 surface 候选实体引用

**SkillForge 自动 + canary + verify multi-surface 闭环业内独家**。Phoenix 提供 raw materials(Score.explanation + trace UI),tau-bench 提供 typed enum schema。**`/insights` 走另一条用户驱动路线**(见路线 B)。

#### 路线 B — Claude Code `/insights` 用户驱动 review(改 CLAUDE.md 用户文档)— [verified-from-source 2026-05-17]

`src/commands/insights.ts` 3200 行单文件 — Anthropic 内置 slash command。详细深挖见 [[../deep/claude-code-insights]]。

**核心机制**:
- **6 维 55 typed enum 跨 session 聚合**(LABEL_MAP `:352-415`)— 业内 Layer D typed enum schema 最完整(对照 tau-bench 7 / AgentOps 15)
- **Opus model 跑 8 INSIGHT_SECTIONS LLM analysis**(`:1336-1495`)— 含 project_areas / interaction_style / what_works / friction_analysis / **suggestions ⭐** / on_the_horizon
- ⭐ **MULTIPLE TIMES algorithm**(`:1430` prompt 注释):_"用户在 2+ session 反复说同一件事 → 自动建议变成 CLAUDE.md 规则"_
- **`suggestions` section** 输出 3 类:`claude_md_additions`(改用户 config 建议)+ `features_to_try` + `usage_patterns`
- **设计哲学**:LLM 给建议,**用户决策 100%**。不自动改 CLAUDE.md。

**Layer E 完成度**(对照 SkillForge):

| 步骤 | SkillForge | Claude Code `/insights` |
|---|---|---|
| ① 自动标注 | ✅ SessionAnnotationService | ✅ Opus 自动从 jsonl 提取 facets |
| ② 跨 session 聚类 | ✅ PatternClusteringService | ✅ AggregatedData 30+ 字段 |
| ③ 归因 LLM 决策 | ✅ AttributionAgent | ✅ 8 INSIGHT_SECTIONS + suggestions |
| ④ Multi-surface candidate | ✅ 自动 skill/prompt/behavior_rule | ⚠️ 只生成 _建议_,**不自动改 CLAUDE.md**(用户决策) |
| ⑤ A/B + Canary | ✅ V2 SKILL-CANARY-ROLLOUT | ❌ 无 |
| ⑥ State machine | ✅ V80 8 状态 | ❌ 无 |

**社区生态补 ④⑤⑥**:
- `yahav10/claude-insights` CLI:把 `/insights` HTML report 转 actionable artifact(skill/rule/hook configs 自动生成)
- `rebelytics/one-skill-to-rule-them-all`:meta-skill 持续观察 sessions + 建议改进 skills(包括 meta-skill 自己)
- `aviadr1/claude-meta`:CLAUDE.md 当 learning system,one prompt 让 Claude 从 mistake 学
- MindStudio `learnings.md` pattern:session start 读 + session end 更新

**业内 Layer E 全景**(`/insights` 加入后)— 详见 [[../deep/claude-code-insights]] §横向校准:

| 项目 | 完成度 | 改什么 surface | 决策权 |
|---|---|---|---|
| **SkillForge V3+V80** | ⭐ 完整闭环 | skill/prompt/behavior_rule/hook | 系统(用户审 proposal)|
| **Claude Code `/insights`** | ⭐ ①②③ 完整 + ④ 建议 | CLAUDE.md 用户文档 | **用户 100%** |
| **AgentScope Trinity-RFT** | ⭐ 改 weights 闭环 | model weights(RL fine-tune) | 系统 |
| **MetaGPT AFlow** | 🟡 workflow 自动生成 | agent workflow | 系统 |
| **DSPy GEPA / Opik Optimizer** | 🟡 改 prompt(用户 metric) | prompt | metric 决定 |
| **Cognee memify** | 🟡 memory feedback loop | memory weight + KG | 系统 |
| **OpenClaw Memory Dreaming** | 🟡 memory promotion | memory state(short→durable)| 系统 |
| **Claude Code autoDream** | 🟡 memory consolidation | memory file(汇总不归因)| 系统 |
| **Hermes commit_memory_session** | ❌ Layer C only | memory extract | 用户手动 |

## 各 Layer 项目覆盖矩阵

| 项目 | Layer A | Layer B | Layer C | Layer D | Layer E |
|---|---|---|---|---|---|
| **Phoenix** | ✅(13 UI 组件)| ✅(Score.explanation)| — | — | — |
| **Langfuse** | ✅(SDK + trace UI)| ✅(Score 3×5 矩阵)| — | — | — |
| **Opik** | ✅(SDK + trace + 4 维分数)| ✅(Assertion + LLM-as-judge)| — | — | — |
| **Helicone** | ✅(Gateway baseURL 切换)| — | — | — | — |
| **AgentOps** | ✅(agent-level 12 SpanKind) | — | — | ✅(3 sub-kind)| — |
| **tau-bench** | — | — | — | ✅ first-class(`auto_error_identification.py`)| — |
| **Promptfoo** | — | ✅(llmRubric)| — | ✅(trace-level assertion)| — |
| **DSPy** | — | — | — | ✅(GEPA Reflective)| — |
| **Ragas** | — | ✅(AspectCritic)| — | — | — |
| **Graphiti** | — | — | ✅(Episodes + validity windows)| — | — |
| **MemPalace** | — | — | ✅(verbatim + KG)| — | — |
| **mem0** | — | — | 🟡(隐式时间戳)| — | — |
| **Letta(MemGPT)** | — | — | 🟡(memory blocks + git history)| — | — |
| **LangChain/LangSmith** | ✅(callback)| ✅(LangSmith Eval,闭源)| — | — | — |
| **SkillForge** | ✅(OBS-2/4 + Hybrid RRF)| ✅(EVAL-V2 4 维分数)| 🟡(extractionBatchId 粗粒度)| 🟡(ATTRIBUTION-AGENT 部分 typed)| ✅ **业内独家**(V80 完整 state machine)|

## SkillForge 真差距清单(归因维度)

### 已领先(verified)
- **Layer E 完整自动归因**(V80 state machine + V83 multi-surface)— **业内独家**
- **Layer A trace + Hybrid RRF + L0/L1 分级**(对照 [[concept/agent-memory]])
- **MEMORY-LLM-SYNTHESIS 4 类 proposal + 人审 gate**(业内独家)

### 真差距(对照 5 layer)

| Layer | 差距 | 借鉴对象 | 优先级 |
|---|---|---|---|
| Layer A | **`AGENT_THINKING / ACTION / DECISION` 3 sub-kind** | AgentOps `span_kinds.py:25-27` | 🔴 高 |
| Layer A | **`GUARDRAIL` 一等公民 SpanKind** | AgentOps `span_kinds.py:17` | 🔴 高 |
| Layer A | **`@operation` / `@task` 显式装饰器层级** | AgentOps `sdk/decorators/` | 🟡 中 |
| Layer B | **`include_explanation: bool = True` LLM-as-judge 默认开 rationale** | Phoenix `evaluators.py:720` | 🔴 高 |
| Layer C | **Episode provenance**(`source_session_id + source_message_id`)| Graphiti / R-MEM-3 | 🔴 高(对应 R-MEM-3) |
| Layer C | **Validity windows**(`valid_from + valid_to`)| Graphiti / R-MEM-2 | 🔴 高(对应 R-MEM-2) |
| Layer D | **`FaultAuthor × FaultType` typed enum schema** | tau-bench `auto_error_identification.py:31-66` | 🟡 中 |
| Layer D | **Trace-level assertion**(`traceErrorSpans / traceSpanCount / traceSpanDuration`)| Promptfoo `src/assertions/` | 🟡 中 |
| Layer E | (已业内最深,无差距)| — | — |

## 反模式

- ❌ **只做 Layer A 没有 Layer B-E**(observability-only,**没有 actionable 归因输出**)— Phoenix / Langfuse / Opik / Helicone 单独都不够,需 SkillForge V3 这种闭环
- ❌ **跳过 Layer A 直接 Layer E**(没有 trace 数据无法归因)— SkillForge OBS-2/4 是 ATTRIBUTION-AGENT 前置
- ❌ **归因 output 是 free-form description**(无 typed enum)— tau-bench `FaultAuthor × FaultType` 模式给 agent 输出加结构化
- ❌ **完全人工 Layer A 人工探索**(不 scalable)— Phoenix Agent UI 半自动是过渡,SkillForge V3 自动归因是终极
- ❌ **归因不闭环**(只输出"该改什么"但不真改 + canary + verify)— SkillForge V2 SKILL-CANARY-ROLLOUT + V80 state machine 才完整

## 关键 insight(给 SkillForge 用)

1. **归因 5 layer 不互斥**:SkillForge 已是 Layer E 唯一 OSS,但**仍需补 Layer A/B/C/D 的工程细节**(3 sub-kind + GUARDRAIL + include_explanation + Episode provenance + typed enum)
2. **Layer A + Layer D 组合归因输入最丰富**:trace 维度(WHAT happened)+ fault enum(WHY failed)
3. **`AGENT_THINKING/ACTION/DECISION` × `FaultAuthor/FaultType`** 矩阵覆盖 12 种细分归因 — 给 ATTRIBUTION-AGENT 输入升级
4. **trace span 本身是归因信号**(Promptfoo 路线)— trace 不只是 raw data,**span duration / error rate / count 直接当 eval signal**(对照 [[../concept/observability]] Wide Events 哲学:observation 是分析单元)
