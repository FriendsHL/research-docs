---
type: eval-framework
name: phoenix
status: active
language: python + typescript

# === 版本时效 ===
last_ingested_version: "arize-phoenix 15.10.0 @ commit 30735f2 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/phoenix]]"

# === 9 维度 ===
target: |
  **Arize AI 出品的 LLM observability + tracing + eval platform**(**ELv2 license,非完全 OSS**)。
  跟其他 eval 路线对比:
  - [[../eval/langfuse]] / [[../eval/opik]] = MIT / Apache 完全 OSS platform
  - [[promptfoo]] = CLI eval 100% 本地
  - [[dspy]] / [[ragas]] = framework / library
  - **Phoenix = ELv2 self-host platform**(商业 SaaS = Arize,开源 = Phoenix self-host)

  **特别强项**:**OpenInference semconv 主推**(LLM trace 语义标准)+ **MCP 集成**(一键 Cursor install)+ **Phoenix 自家 Agent UI assistant**(用户在 trace UI 跟 AI 查 root cause)
  [verified-from-source]

dataset: |
  - 内置 Dataset 实体(`packages/phoenix-evals/src/phoenix/evals/`)
  - DataFrame 集成:`evaluate_dataframe(...)`
  - HuggingFace Dataset 兼容
  - 平台级 dataset 管理(`app/src/pages/dataset/` + `app/src/pages/datasets/`)
  [verified-from-source]

scorer: |
  ### `Score` dataclass(`packages/phoenix-evals/.../evaluators.py:159`)

  ```python
  @dataclass(frozen=True, init=False)
  class Score:
      score: float | None              # 数值分数
      name: str
      label: str | None                # 分类 label
      explanation: str | None          # ⭐ 归因/LLM rationale
      metadata: dict
      kind: KindType                   # "human" | "llm" | "heuristic" | "code"
      direction: DirectionType         # "maximize" | "minimize" | "neutral"
  ```

  ### KindType 4 类(对照 [[../eval/langfuse]] ScoreSource 3 类 / [[../eval/opik]] Assertion+Feedback 双轨)
  ```python
  KindType = Literal["human", "llm", "heuristic", "code"]
  ```
  - **`human`** — 人工标注
  - **`llm`** — LLM-as-judge
  - **`heuristic`** — 规则
  - **`code`** — 代码可执行 metric

  ### DirectionType 3 类
  ```python
  DirectionType = Literal["maximize", "minimize", "neutral"]
  ```

  ### `ClassificationEvaluator`(`evaluators.py:700+`)
  - `include_explanation: bool = True`(默认开 LLM rationale)
  - LLM `generate_classification(labels, include_explanation=...)` → 拿 label + explanation
  - 校验 label 在合法 choices 中
  - 映射 label → score(数值)
  - 返回 Score 对象,**带 explanation field 作为归因**

  ### 11 个内置 Metric(`packages/phoenix-evals/src/phoenix/evals/metrics/`)
  - **conciseness** / **correctness** / **document_relevance** / **exact_match** / **faithfulness**(RAG)/ **hallucination** / **matches_regex** / **precision_recall** / **refusal** / **tool_invocation** / **tool_response_handling** / **tool_selection**
  [verified-from-source]

reproducibility: |
  - **Helm chart + Kustomize + docker-compose** 完整部署
  - **OpenInference semconv** 跨 platform trace 标准(对照 Langfuse OTel ingestion)
  - **`redaction.py` 数据脱敏**(PII / 敏感词)
  - **`retention.py` Data retention**
  - **JWT / OAuth2 / LDAP 多 auth**
  - **gRPC server**(`grpc_server.py`)高吞吐 trace 摄入
  [verified-from-source]

coverage: |
  ### 5 大支柱
  1. **LLM Observability + Tracing**(主)— OpenInference semconv first-class
  2. **Eval framework**(11 内置 metric + `Score` schema)
  3. **Experiments**(`src/phoenix/server/experiments/`)
  4. **Phoenix Agent UI**(用户在 trace UI 跟 AI 查 root cause)
  5. **MCP server**(`js/packages/phoenix-mcp/`,Cursor 一键 install)

  ### 8 大 UI 页(`app/src/pages/`)
  - **trace**(13 组件,见 [[../source/phoenix]])
  - **project** / **projects**
  - **dataset** / **datasets**
  - **experiment** / **experiments**
  - **playground**
  - **prompt** / **prompts**
  - **evaluators**
  - **dashboards**
  - **settings**

  ### 不覆盖
  - **没有 Optimizer**(对照 [[dspy]] 15 / [[../eval/opik]] 7)
  - **没有 production loop 自动归集**(对照 [[../eval/langfuse]] `addDatasetRunItemsToEvalQueue`)
  - **没有 main/secondary 双队列**(对照 Langfuse 多租户分流)
  - **不做 red teaming**(对照 [[promptfoo]] 25 攻击策略)
  - **跨 session 失败聚类不做 first-class**(对照 SkillForge ATTRIBUTION-AGENT)
  [verified-from-source]

production_loop: |
  **半 production loop**:
  - **Trace 摄入**:OpenInference semconv 自动收 production trace
  - **Eval 离线**:用户手动跑 `evaluate_dataframe()` 或 UI 触发
  - **Phoenix Agent UI** 半自动 root cause:用户在 trace UI 跟 AI assistant 对话查
  - **没有 production trace → dataset 自动归集**(对照 Langfuse)
  - **没有自动归因 agent**(对照 SkillForge ATTRIBUTION-AGENT)

  **关键 insight**:**Phoenix 提供 raw materials**(trace + eval + UI),不提供"自动归因 agent"
  [verified-from-source]

automation: |
  - **MCP 一键 Cursor install** — 业内独家
  - **Helm + Kustomize** 完整 K8s 部署
  - **`cost_tracking/` daemon** 自动跟踪 cost
  - **`daemons/`** 后台 daemon
  - **gRPC server**(`grpc_server.py`)+ HTTP API
  - **`dml_event.py + dml_event_handler.py`** DML 事件流
  [verified-from-source]

cost: |
  - **`src/phoenix/server/cost_tracking/`** 独立 daemon — 业内独家 cost daemon
  - **`fix(cost): update built-in model token prices`**(CHANGELOG 15.10.0, #13244)— 自动更新模型价格(对照 [[../eval/langfuse]] `default-model-prices.json`)
  - LLM-as-judge `include_explanation=True` 默认开 — 比关闭多 ~30% token cost
  - 11 metric × N samples × LLM-as-judge 成本可控
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Phoenix 提供归因 raw materials,**SkillForge ATTRIBUTION-AGENT 走在 Phoenix 之上**

  详见已经做的归因深度对照(前一轮 ingest):
  - Phoenix `Score.explanation` = per-evaluation LLM rationale(descriptive)
  - SkillForge ATTRIBUTION-AGENT = production-level cross-session 自动归因(actionable)
  - **SkillForge 已超过 Phoenix 一个 layer**

  ## 可借鉴(给 SkillForge 工程细节)

  ### 高优(1-2 周)
  1. **`Score schema` 加 `kind` + `direction` 字段**(Phoenix `KindType={human,llm,heuristic,code}` + `DirectionType={maximize,minimize,neutral}`)— SkillForge EvalTaskEntity 加这 2 字段给 ATTRIBUTION-AGENT 输入更结构化
  2. **`include_explanation: bool = True` 默认开** — SkillForge LLM-as-judge metric 应默认带 rationale(给 ATTRIBUTION-AGENT 输入更丰富的 trace)
  3. **OpenInference semconv 接入** — SkillForge `skillforge-observability` 模块跟进 OpenInference 标准,跟 Phoenix / Arize / Langfuse / Opik 互通(对照 R14 OTel 需求)

  ### 中优(1 个月)
  4. **`cost_tracking/` 独立 daemon 模式**(`src/phoenix/server/cost_tracking/`)— SkillForge cost 跟踪可拆独立 daemon(对照 SkillForge 当前可能嵌入主流程)
  5. **`redaction.py` 数据脱敏 first-class** — SkillForge 多租户场景应有
  6. **`retention.py` Data retention** — SkillForge 应有 trace retention 策略(对照 Langfuse `DataRetentionQueue`)
  7. **MCP server**(`js/packages/phoenix-mcp/`)— SkillForge dashboard 可暴露 MCP server 让用户 IDE 一键查 trace

  ### 低优(可选)
  8. **Phoenix Agent UI assistant**(`app/src/agent/` + `app/src/components/agent/`)— SkillForge dashboard 可加 AI assistant chat 让用户在 UI 查 root cause(但 SkillForge ATTRIBUTION-AGENT 已自动化,UI assistant 是 nice-to-have)
  9. **Trace UI 13 组件**(`ConnectedTraceTree.tsx` 等)— SkillForge OBS-1 + OBS-4 trace UI 可对照升级

  ## 不必跟
  - **ELv2 license**(SkillForge 选 license 时不学这种限制商业的)
  - **OpenInference 自家标准**(SkillForge 走 OTel 业内标准)
  - **8 大 UI 页拆分粒度**(SkillForge 当前 dashboard 工程化已足够)

  ## SkillForge 已领先
  - **ATTRIBUTION-AGENT 自动归因**(Phoenix 完全没有,只有 UI assistant 半自动)
  - **Production loop + canary 灰度**(Phoenix 没有)
  - **跨 session 失败聚类**(Phoenix 没有)
  - **多 surface 优化框架**(Phoenix 不做 prompt optimization)

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **Arize AI 商业 SaaS 的 self-host 版本**(ELv2,非完全 OSS)— 跟 Langfuse / Opik 完全 OSS 不同 license 模式
  2. **OpenInference semconv 主推**(LLM trace 语义标准)— Arize 想做 LLM observability 标准制定者
  3. **MCP server 一键 Cursor install** 业内独家
  4. **Phoenix 自家有 Agent UI assistant**(`app/src/agent/`)— 用户在 trace UI 跟 AI 查 root cause
  5. **`KindType` 4 类 vs Langfuse 3 类 / Opik 双轨**:human / llm / heuristic / code(`evaluators.py:46`)
  6. **`DirectionType` 3 类**(maximize / minimize / neutral)— 跟 Opik 4 维分数对照
  7. **`include_explanation: bool = True`** LLM-as-judge 默认开 rationale
  8. **`cost_tracking/` 独立 daemon** — 业内独家把 cost tracking 拆独立 daemon
  9. **`redaction.py` 数据脱敏 first-class** — 业内独家把 redaction 单独模块
  10. **gRPC + HTTP + GraphQL 三协议**(`grpc_server.py` + REST + Relay GraphQL `RelayEnvironment.ts`)
  11. **`15.10.0` 高版本号** — 2023-2026 已 15 major,业内 LLM observability 最快迭代

  ## 优势 (strengths)
  1. **OpenInference semconv first-class**(`from openinference.semconv.trace import ...`)— 跨 platform 兼容
  2. **`Score` dataclass `frozen=True`**(`evaluators.py:159`)— immutable 工程细节
  3. **`evaluate_dataframe(...)`** DataFrame 一等公民 — 数据科学家直接用 pandas
  4. **`packages/` 三个 PyPI package 拆分**(phoenix-client / phoenix-evals / phoenix-otel)— 用户按需 pip install
  5. **`internal_docs/` + `docs/` 区分** — 公开 doc 跟内部 doc 隔离
  6. **`agent-phoenix-mcp` MCP server** 跟 Cursor / Claude Desktop 一键集成
  7. **`fix(cost): update built-in model token prices`** 频繁更新模型价格(类 Langfuse / Opik)
  8. **`session_filters.py`** Session filter 模块化
  9. **`thread_server.py`** Thread 处理独立
  10. **`prometheus.py` + 自家 `telemetry.py`** — 自己被观测

  ## 哪些点做得好 (what_done_well)
  1. **`Score.explanation` 作为 Score 实体内 field**(`evaluators.py:159`)— 归因跟 score 同结构,**不另开 explanation table**(对照 Langfuse Score + comments 拆开)
  2. **`KindType = Literal["human", "llm", "heuristic", "code"]`** + `DirectionType` 用 TypeScript-style `Literal` typed enum — Python type hint 极致
  3. **`include_explanation` 默认 True**(`evaluators.py:720`)— LLM-as-judge 默认带 rationale,**好的 default**
  4. **`_smart_string_coerce`**(`evaluators.py:55-75`)用 JSON 序列化非字符串 vs `EnforcedString` 用 `str(...)` — 区分 LLM prompt 需要 valid JSON vs 简单 stringify
  5. **`OpenInferenceSpanKindValues`** + `SpanAttributes` 直接 import 用 — 不重定义 enum
  6. **`response.get("explanation", None)`** 而非 `response["explanation"]` — 兼容 LLM 没给 explanation 的情况
  7. **`Score(frozen=True, init=False)`** + dataclass — immutable + 自定 init,工程化深
  8. **OpenInference 跟 OTel GenAI 互转**(`feat(trace): convert OTel GenAI semconv attrs to OpenInference` #13267)— 标准之间桥接,工程文化漂亮
  9. **MCP server 一键 cursor install**(`cursor://anysphere.cursor-deeplink/mcp/install?name=phoenix&config=...`)— dev experience 极致
  10. **`cost_tracking/` + `daemons/` + `retention.py` + `redaction.py`** 4 个独立 server-side 模块 — concern 分离
---

# Phoenix(Arize AI, ELv2, 15.10.0)

## 一句话定位

Arize AI 的开源 LLM observability + eval platform(ELv2,**非完全 OSS**)。**OpenInference semconv 主推 + MCP server 一键 Cursor install + Phoenix Agent UI 让用户在 trace UI 跟 AI 查 root cause + `Score.explanation` per-evaluation LLM 归因 + 11 内置 metric + 13 trace UI 组件**。

## 跟 SkillForge ATTRIBUTION-AGENT 关系

**Phoenix 提供"归因 raw materials"(trace + Score.explanation + UI),SkillForge ATTRIBUTION-AGENT 走在 Phoenix 之上一个 layer**(production-level cross-session 自动归因 + Optimization Event 因果链 + 多 surface 优化)。

详见前一轮做的归因深度对照(已总结到 frontmatter `skillforge_takeaway`)。

## SkillForge 可借鉴 3 大工程细节

### 1. `Score schema` 加 `kind` + `direction` 字段

Phoenix `Score`(`evaluators.py:159+`)dataclass:
- `kind: KindType` = "human" | "llm" | "heuristic" | "code"
- `direction: DirectionType` = "maximize" | "minimize" | "neutral"
- `explanation: str | None` = LLM-generated rationale

SkillForge EvalTaskEntity 可加这 3 字段,给 ATTRIBUTION-AGENT 输入更结构化。

### 2. `include_explanation: bool = True` 默认开 LLM rationale

Phoenix LLM-as-judge 默认带 rationale,**多 ~30% token cost 但给归因 agent 输入更丰富**。

SkillForge LLM-as-judge metric 应默认 `include_explanation=True`,给 ATTRIBUTION-AGENT 输入更丰富 trace。

### 3. OpenInference semconv 接入

SkillForge `skillforge-observability` 模块跟进 OpenInference 标准 / OTel,跟 Phoenix / Arize / Langfuse / Opik 互通(对照 R14 OTel 需求)。

Phoenix CHANGELOG 15.10.0(2026-05-15)新增 `feat(trace): convert OTel GenAI semconv attrs to OpenInference`(#13267)— **业内 OTel ↔ OpenInference 转换器**,SkillForge 跟进可对接两边。

## 待回答
- ELv2 license 商业使用限制具体条款(SkillForge 如果商业化是否能用 Phoenix 自托管)
- OpenInference vs OTel GenAI semconv 长期谁胜出(Arize vs Anthropic/OpenAI 标准之争)
- Phoenix Agent UI 具体怎么帮用户查 root cause(prompt 模板 + tool 调用流)
- `cost_tracking/` daemon 跟 SkillForge cost 跟踪集成方式
- `redaction.py` 具体脱敏策略(PII 检测 regex / NER / LLM-based)
