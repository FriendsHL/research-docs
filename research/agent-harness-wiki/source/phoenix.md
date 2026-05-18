---
type: source
name: phoenix
origin: "/Users/youren/myspace/phoenix/ (Arize AI, ELv2)"
ingested_at: 2026-05-16
ingested_commit: 30735f2
ingested_commit_date: 2026-05-15
ingested_version_label: "arize-phoenix 15.10.0(CHANGELOG.md)"
contributed_to:
  - eval/phoenix.md
  - concept/observability.md
  - concept/attribution.md
---

# Source — Phoenix(Arize AI)

> **Arize AI 出品**(ELv2 license,**非完全 OSS**)— **LLM observability + tracing + eval platform**。MCP-enabled(`badge.mcpx.dev?status=on`)+ Cursor MCP install one-click。开源 Phoenix 是 Arize 商业 SaaS 的 self-host 版本。**OpenInference semconv 主推者**(LLM trace 语义标准)。

## 仓库结构概览

```
phoenix/
├── packages/                       # 公开 Python packages(PyPI 发布)
│   ├── phoenix-client/             # Client SDK
│   ├── phoenix-evals/              # ⭐ Eval framework(Score / Evaluator)
│   └── phoenix-otel/               # OTel 集成
├── src/phoenix/                    # 主 server(internal)
│   ├── server/                     # FastAPI server
│   │   ├── agents/                 # ⭐ Phoenix 自家也有 agent
│   │   ├── api/                    # REST + GraphQL
│   │   ├── experiments/
│   │   ├── cost_tracking/          # 成本跟踪 daemon
│   │   ├── daemons/                # 后台 daemon
│   │   ├── retention.py            # Data retention
│   │   ├── redaction.py            # 数据脱敏
│   │   ├── grpc_server.py          # gRPC server
│   │   ├── thread_server.py        # Thread 处理
│   │   ├── openapi/
│   │   ├── middleware/
│   │   └── ...(auth / bearer_auth / oauth2 / ldap / encryption / rate_limiters / telemetry)
│   ├── trace/                      # ⭐ Trace 模型
│   ├── tracers.py                  # Tracer 实现
│   ├── db/                         # DB layer
│   ├── core/                       # Core
│   ├── datasets/
│   ├── experimental/
│   ├── metrics/
│   ├── session/
│   ├── utilities/
│   └── version.py
├── app/                            # ⭐ React frontend(Vite + TypeScript)
│   ├── src/agent/                  # Agent context + chat
│   ├── src/components/agent/       # Agent UI
│   ├── src/pages/                  # 8 大页:trace / project / dataset / experiment / playground / prompt / settings / evaluators / dashboards
│   └── src/openInference/          # OpenInference 集成
├── js/                             # JS SDK(`packages/phoenix-mcp`)
├── docs/                           # 公开文档(arize.com/docs/phoenix/)
├── internal_docs/                  # 内部文档
├── examples/                       # 用户示例
├── helm/                           # Helm chart
├── kustomize/                      # Kubernetes manifests
├── api_reference/                  # API reference docs
├── docker-compose.yml
├── docs.json
├── AGENTS.md
├── CLAUDE.md
└── pyproject.toml
```

## 关键 file:line 锚点

### Phoenix Evals(`packages/phoenix-evals/src/phoenix/evals/`)

**核心模块** — 这是 Phoenix 归因实现所在。

#### `Score` dataclass(`evaluators.py:159+`)
```python
@dataclass(frozen=True, init=False)
class Score:
    score: float | None              # 数值分数
    name: str                        # metric 名
    label: str | None                # 分类 label
    explanation: str | None          # ⭐ 归因/理由(LLM-generated)
    metadata: dict                   # 含 model 等
    kind: KindType                   # "human" | "llm" | "heuristic" | "code"
    direction: DirectionType         # "maximize" | "minimize" | "neutral"
```

**关键 type alias**(`evaluators.py:46-47`):
```python
KindType = Literal["human", "llm", "heuristic", "code"]
DirectionType = Literal["maximize", "minimize", "neutral"]
```

#### `ClassificationEvaluator`(`evaluators.py:720+`)

```python
def __init__(
    self,
    name: str,
    llm: LLM,
    prompt_template,
    choices,
    include_explanation: bool = True,   # ⭐ 默认开 LLM rationale
    input_schema: Optional[type[BaseModel]] = None,
    direction: DirectionType = "maximize",
    ...
):
    self.include_explanation = include_explanation
    # ... label_score_map 构建
```

**`_evaluate` 工作流**(`:743-770`):
```python
response = self.llm.generate_classification(
    prompt=prompt_filled,
    labels=self.labels,
    include_explanation=self.include_explanation,   # ⭐ 给 LLM 拿理由
    ...
)
label = response["label"]
explanation = response.get("explanation", None)    # ⭐ 拿 LLM rationale

return [Score(
    score=score,
    name=self.name,
    label=label,
    explanation=explanation,                        # ⭐ 归因附 Score
    metadata={"model": self.llm.model},
    kind=self.kind,
    direction=self.direction,
)]
```

**核心 insight**:**Phoenix 的"归因" = per-Score LLM-as-judge explanation field**(每个 LLM-as-judge score 自带 rationale)。

#### 11 个内置 Metric(`packages/phoenix-evals/src/phoenix/evals/metrics/`)
- `conciseness.py` / `correctness.py`
- `document_relevance.py`
- `exact_match.py`
- `faithfulness.py`(RAG)/ `hallucination.py`
- `matches_regex.py`
- `precision_recall.py`
- `refusal.py`
- `tool_invocation.py` / `tool_response_handling.py` / `tool_selection.py`

#### Evaluator 类层级(`evaluators.py`)
```python
class Evaluator(ABC)
class LLMEvaluator(Evaluator)
class ClassificationEvaluator(LLMEvaluator)   # ⭐ 主类

# Helper
def create_evaluator(...)
def create_classifier(...)
def bind_evaluator(...)
def evaluate_dataframe(...)
```

### Phoenix Trace UI(`app/src/pages/trace/`)— 13 组件

| 组件 | 作用 |
|---|---|
| `TracePage.tsx` | 主 trace 页 |
| `TraceDetails.tsx` | trace 详情面板 |
| **`ConnectedTraceTree.tsx`** | **视觉化 trace 树**(span 父子关系) |
| `TraceFeedbackActionToolbar.tsx` | 人工标注 trace |
| `TraceHeaderRootSpanAnnotations.tsx` | Root span 注释 |
| `TracePaginationContext.tsx` + `TraceDetailsPaginator.tsx` | 分页 |
| `SessionDetailsTracesView.tsx` + `SessionDetailsTraceList.tsx` | Session 内 trace |
| `TraceNotesTableCell.tsx` | trace notes |
| `TracesTable.tsx` | 项目级 trace 列表 |
| `TransferTracesButton.tsx` | trace 跨项目转移 |
| `ProjectTraceCountSparkline.tsx` | 趋势图 |
| `PlaygroundRunTraceDialog.tsx` | Playground run trace |

### Phoenix Agent(`app/src/agent/` + `app/src/components/agent/`)

**Phoenix 自家也有 Agent**(meta!):
- `agent/context/agentContextTypes.ts`
- `agent/context/deriveRouteContexts.ts`
- `agent/context/useAdvertiseAgentContext.ts`
- `agent/chat/buildAgentChatRequestBody.ts`
- `agent/chat/handleAgentToolCall.ts`
- `agent/chat/interruptToolCalls.ts`
- `agent/chat/shouldSendAutomatically.ts`
- `agent/extensions/toolRegistry.ts`
- `agent/extensions/capabilities.ts`
- `components/agent/useAgentChat.ts`
- `components/agent/useGenerateSessionSummary.ts`     # 自动生成 session summary
- `components/agent/sessionSummaryUtils.ts`
- `components/agent/groupMessageParts.ts`
- `components/agent/agentModelConfig.ts`
- `components/agent/useAgentChatPanelState.ts`
- `components/agent/useAssistantAgentEnabled.ts`
- `components/agent/useActiveChatPanelWhileMounted.ts`
- `components/agent/toolPartTypes.ts`

**关键**:Phoenix UI 内置 AI assistant,**用户在 trace UI 跟 AI assistant 对话查 root cause**(类 SkillForge ATTRIBUTION-AGENT 但是 UI-driven 而非 production cron)。

### Server-side 模块(`src/phoenix/server/`)

| 模块 | 作用 |
|---|---|
| `app.py` / `main.py` | FastAPI app 入口 |
| `grpc_server.py` | gRPC server(高吞吐 trace 摄入) |
| `cost_tracking/` | ⭐ Cost tracking daemon |
| `daemons/` | 后台 daemon |
| `retention.py` | Data retention |
| `redaction.py` | 数据脱敏(PII / 敏感词) |
| `experiments/` | Experiment management |
| `agents/` | Phoenix 自家 Agent server-side |
| `dml_event_handler.py` + `dml_event.py` | DML 事件流 |
| `jwt_store.py` | JWT store |
| `oauth2.py` + `ldap.py` + `bearer_auth.py` | 多 auth |
| `rate_limiters.py` | Rate limit |
| `encryption.py` | 加密 |
| `email/` | Email 通知 |
| `prometheus.py` | Prometheus metric |
| `telemetry.py` | 自家 telemetry |
| `thread_server.py` | Thread 处理 |
| `session_filters.py` | Session 过滤 |
| `openapi/` | OpenAPI spec |
| `cli/` | CLI |
| `middleware/` | 中间件 |

### OpenInference Semconv(`evaluators.py:30`)

```python
from openinference.semconv.trace import OpenInferenceSpanKindValues, SpanAttributes
```

**Arize 主推 OpenInference**(`openinference-spec/openinference`)— LLM trace 语义标准,跟 OpenTelemetry GenAI semconv 并行。

CHANGELOG `15.10.0`(2026-05-15)有 `feat(trace): convert OTel GenAI semconv attrs to OpenInference`(#13267)— **Arize 把 OTel GenAI 标准 normalize 到 OpenInference**。

### MCP 集成(README badge `mcpx.dev?status=on`)

- `js/packages/phoenix-mcp/` — Phoenix 自家 MCP server
- README 一键 Cursor MCP install:
  ```
  cursor://anysphere.cursor-deeplink/mcp/install?name=phoenix&config=...
  ```
- **业内独家 LLM observability 提供 MCP server**(让 Cursor / Claude Desktop 直接查 Phoenix trace)

### Helm + Kustomize 部署

- `helm/` Helm chart
- `kustomize/` Kubernetes manifests
- `docker-compose.yml` 本地 dev

## 与 wiki 现有

### 印证
- **Wide Events 2.0 哲学** — Phoenix 同走(observation as primary unit)
- **ClickHouse / Postgres / Redis 异构存储** — 跟 Langfuse / Opik 同套
- **OpenInference + OpenTelemetry semconv** — Arize 主推 OpenInference,跟 Langfuse OTel ingestion 互通
- **MCP 集成** — Phoenix MCP server 跟 [[../harness/codex]] / [[../harness/opencode]] / SkillForge MCP-CLIENT-MVP(2026-05-07 已交付)互通

### 矛盾
- **Phoenix `Score.explanation` = per-evaluation LLM rationale**(descriptive)vs **SkillForge ATTRIBUTION-AGENT = production-level cross-session 自动归因**(actionable)— **不同 layer**(详见 [[../concept/attribution]] 待新建)
- **Phoenix ELv2 license**(非完全 OSS)vs Langfuse MIT / Opik Apache 2.0 — license 选型差异

### 新增
- **`eval/phoenix.md`**(新建)
- **扩 [[../concept/observability]]**:加 Phoenix Arize OpenInference 路线 + Phoenix UI 13 trace 组件 + Phoenix 自家 Agent assistant
- **(可选)新建 [[../concept/attribution]]**:Phoenix Score.explanation vs SkillForge ATTRIBUTION-AGENT 深度对照(已在前一轮分析)

## 事实漂移 / 注意点

- **ELv2 license**(Elastic License v2)— **非完全 OSS**,商业使用受限(用户大企业要看 license)
- **Arize 是 Phoenix 商业 SaaS**,开源 Phoenix 是 self-host 版本(类 ELK)
- **15.10.0 高版本号** — Phoenix 迭代快,2023-2026 已 15 major versions
- **OpenInference vs OTel GenAI** 竞争中(Arize 主推 OpenInference,Anthropic / OpenAI 主推 OTel GenAI)— Phoenix `15.10.0` 加 OTel → OpenInference 转换器是回应
- **MCP 一键 Cursor install** 是营销 + dev experience 双重产物
- **Phoenix 自家有 Agent UI assistant** — 用户在 trace UI 跟 AI 对话查 root cause
- **`cost_tracking/` daemon + `redaction.py`** — Phoenix 工程化深(对照 Langfuse cost dashboard + Opik 4 维分数 cost dimension)
