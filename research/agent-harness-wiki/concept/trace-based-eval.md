---
type: concept
name: trace-based-eval
status: draft
sources:
  - "[[../eval/promptfoo]]"
  - "[[../eval/phoenix]]"
  - "[[../eval/agentops]]"
  - "[[../eval/langfuse]]"
  - "[[../eval/opik]]"
  - SkillForge skillforge-observability(verified)
related_evals: [promptfoo, phoenix, agentops, langfuse, opik, helicone]
related_concepts: [observability, agent-eval, attribution]
---

# Trace-based Eval — Trace 数据作为评测信号

> **wiki 此前的盲区**:wiki ingest 的 platform observability 项目(Phoenix / Langfuse / Opik / Helicone / AgentOps)都把 trace 当 raw materials(给人看 / 给 LLM 看 / 做归因 raw input),**只有 Promptfoo 显式把 trace metric 当 first-class assertion 输入**(3 个 trace-level assertion)。
>
> **本 concept 把 trace 显式作为评测信号的独立维度**,补全 [[agent-eval]] 8 维设计空间(扩为 9 维)。

## 定义

**Trace-based Eval** = 把 trace / span 的 metric(shape / timing / error / cost)**作为 assert 输入参与 pass/fail 判定**,而不是只作为 raw data 给 observability dashboard。

跟 [[observability]] / [[attribution]] / [[agent-eval]] 关系:

| 概念 | 角色 | 例子 |
|---|---|---|
| **Observability** | 收集 raw trace | 写 t_llm_trace + t_llm_span |
| **Attribution** | 反推因果 | 找 "为什么失败" |
| **Trace-based Eval** | **trace metric 作为 assert 输入** | **"trace span count ≤ 50" 通过/失败** |
| **Agent Eval** | 整体评测框架 | LLM-as-judge + assertion + dataset |

## 4 类 Trace-based Eval Signal

```
[Signal 1: Shape]
   ├─ Span count(总 span 数)
   ├─ Tool call count
   ├─ Event count
   ├─ Nesting depth(trace 树深度)
   └─ Branch count(并行 span 数)

[Signal 2: Timing]
   ├─ Total duration(trace 全长)
   ├─ Per-span latency
   ├─ Latency p50 / p99
   ├─ Wait time(等待外部 API)
   └─ Time-to-first-token(streaming)

[Signal 3: Error]
   ├─ Error span count
   ├─ Error rate per kind(llm vs tool vs event)
   ├─ Retry count
   ├─ Error path(从 root → first error)
   └─ Status(running / ok / error / cancelled)

[Signal 4: Cost]
   ├─ Total token(input + output)
   ├─ Cache hit ratio(read / creation)
   ├─ Total cost(USD)
   ├─ API call count
   └─ Cost per surface(skill / prompt / agent)
```

**每类 signal 都可以是 assert 输入**(如 `traceSpanCount < 50 AND total_duration_ms < 30000 AND error_span_count == 0`)。

## 业内实现对比(verified)

### Promptfoo(业内唯一 first-class 实现)— [[../eval/promptfoo]]

**[verified-from-source]** `src/assertions/`:

```
traceErrorSpans.ts          # Signal 3:trace 中 error span 数 ≤ X
traceSpanCount.ts           # Signal 1:trace 总 span 数在区间
traceSpanDuration.ts        # Signal 2:span 总耗时上限
traceUtils.ts               # Trace 数据处理工具
```

**业内独家**:把 trace span 直接当 assertion 输入。其他 framework 的"trace-level assertion"主要是:
- LangSmith:闭源 SaaS,evaluator 可读 trace 但无 first-class API
- Langfuse:可在 prompt 里访问 trace 但要自定 evaluator function
- Phoenix:可在 evaluator 加 trace context 但靠用户写 Python

**仅 Promptfoo 3 个 trace assertion 是 YAML config 直接配**(无需写 evaluator function)。

### Phoenix(隐式 trace eval)— [[../eval/phoenix]]

`Score.metadata: dict` 字段可放 trace 信息,但**没有显式 trace-as-eval-signal API**。trace 主要给 Phoenix Agent UI assistant 用户对话查 root cause(半自动 attribution)。

### Langfuse / Opik / AgentOps(只 observability)

trace 收集 + UI 探索 + 跟 LLM-as-judge 联动,**但没有显式 trace metric assertion**。

### SkillForge OBS-2/4(verified, 已就绪 + 大部分没用)

**[verified-from-source]** `skillforge-observability/.../LlmTraceEntity.java`:

```java
@Column(name = "total_duration_ms")     long totalDurationMs;    // ⭐ Signal 2 timing
@Column(name = "tool_call_count")       int toolCallCount;        // ⭐ Signal 1 shape
@Column(name = "event_count")           int eventCount;           // ⭐ Signal 1 shape
@Column(name = "total_input_tokens")    int totalInputTokens;     // ⭐ Signal 4 cost
@Column(name = "total_output_tokens")   int totalOutputTokens;    // ⭐ Signal 4 cost
@Column(name = "total_cost_usd")        BigDecimal totalCostUsd;  // ⭐ Signal 4 cost
@Column(name = "status")                String status;            // ⭐ Signal 3 error
                                          // running | ok | error | cancelled
@Column(name = "error")                 String error;             // ⭐ Signal 3 error
```

**SkillForge OBS-2 trace 表已经有 8 个 trace metric 字段**,但 **EVAL-V2 4 维分数(quality/efficiency/latency/cost)是 eval 主路径**,**trace metric 没作为 first-class assertion 输入**。

跟 [[../eval/promptfoo]] 3 trace assertion 对照,**SkillForge 数据全有,差 assertion 接入层**。

## SkillForge OTel 集成度(重要校准)

> 用户提到"我设计的时候是有这个参考",经 grep verify:**SkillForge 是 OTel-inspired 但不是 OTel-native**。

**[verified-from-source]**(grep `skillforge-observability/src` + `skillforge-server/src`):

| 维度 | SkillForge 现状 | OTel 标准 |
|---|---|---|
| grep `OpenTelemetry` / `otel` | **0 命中** | — |
| trace_id 格式 | `length=36` **UUID**(`LlmSpanEntity.java:18`) | **32 hex chars**(W3C Trace Context) |
| span_id 格式 | `length=36` UUID | **16 hex chars** |
| Resource attributes | 自家业务字段(`origin / source / total_duration_ms`)| **OTel semconv**(`service.name / deployment.environment / ...`) |
| Span kind | `kind` 3 类(`llm / tool / event`)+ `event_type` 4 类 | **OTel 6 类**(INTERNAL / SERVER / CLIENT / PRODUCER / CONSUMER + 隐式)|
| Span attributes | `attributes_json` JSONB(自家 JSON)| **OTel structured attribute keys**(`gen_ai.system / gen_ai.request.model / ...`)|
| 传输协议 | `LlmTraceWriteRequest` 自家 REST API(`api/LlmTraceWriteRequest.java`)| **OTLP**(gRPC / HTTP) |
| 写入 | `PgLlmTraceStore.upsertTrace` 原生 SQL | **OTel SDK BatchSpanProcessor** |
| **概念对齐?** | ✅ trace + span + parent_span_id chain + duration + status | ✅ |
| **协议兼容?** | ❌ **不能接受 OTLP / 不能 export 到 Phoenix 等** | — |

**关键 insight**:**SkillForge 借鉴 OTel 概念**(trace/span/parent_span_id/duration/status 等),**但用自家 schema + 自家协议** — 这意味着:
- ✅ 内部观测体系完整
- ❌ **不能直接跟 Phoenix / Langfuse / Opik / AgentOps 互通**(需要桥)

### 业内独家:OBS-4 `root_trace_id`(跨 trace 跨 agent 串联)

**[verified-from-source]** `LlmTraceEntity:31-35`:

```java
@Column(name = "root_trace_id", length = 36)
private String rootTraceId;
// OBS-4 M0: 跨 agent / 跨 session trace 串联根。
// 同一 user message 内主 agent 所有 trace + 派出的所有 subagent
// (含递归 child of child) 全部共享同一 root_trace_id。Immutable。
```

**OTel 没有原生等价**:
- OTel `parent_span_id chain` 只能在**单个 trace 内**(同 trace_id)
- 跨 trace / 跨 agent 串联需要 OTel `links`(W3C Span Link)或自定 attribute
- **SkillForge `root_trace_id` 是更强抽象**(immutable trace 树根,跨 trace_id),**业内独家**

## 9 维设计空间(扩展 [[agent-eval]] 8 维)

> 原 8 维详见 [[agent-eval]]。本节加第 9 维 **"trace-as-eval-signal"**。

| 维度 | 选项 |
|---|---|
| **(9 新加)trace-as-eval-signal** | 不用(只当 observability raw)/ 隐式 evaluator function 自取 / **first-class assertion**(Promptfoo `traceErrorSpans / traceSpanCount / traceSpanDuration`)/ **OBS-2 字段直接 SQL filter**(SkillForge 已就绪) |

## R-OBS-N 借鉴清单(给 SkillForge 用)

### 🔴 优先级 1:Trace-as-eval-signal 工程化(短期 1-2 周)

| # | 借鉴 | 来源 | Verified 现状 |
|---|---|---|---|
| **R-OBS-1** | **Trace metric 作为 EvalRun 一等 assertion**(对照 Promptfoo `traceErrorSpans/Count/Duration`)| Promptfoo `src/assertions/` | SkillForge OBS-2 已有 `total_duration_ms / tool_call_count / event_count / status` 字段,**差 assertion 接入层** |
| **R-OBS-2** | **EvalTaskEntity 加 trace metric 字段**(对照当前 4 维 quality/efficiency/latency/cost) | 自家工程 | SkillForge EVAL-V2 4 维已部分覆盖,加 `trace_span_count / trace_error_count / trace_depth` 等 |
| **R-OBS-3** | **ATTRIBUTION-AGENT 输入加 trace shape signal** | Promptfoo / AgentOps | V80 t_optimization_event 当前只看 confidence,可加 trace 维度归因 |

### 🟡 优先级 2:OTel 协议层(中期 1 个月)

| # | 借鉴 | 价值 |
|---|---|---|
| **R-OBS-4** | **加 OTel-compatible 输出**(BatchSpanProcessor + OTLP exporter):SkillForge trace → 转 OTel format → Phoenix / Langfuse / AgentOps 都可接 | 一行配置就能跟业内 5+ observability platform 互通(LangChain Callback / Phoenix OpenInference 等都靠 OTel) |
| **R-OBS-5** | **加 OTel-compatible 输入**(OTLP ingestion endpoint):用户在第三方 framework(LangChain / CrewAI / LangGraph)用,trace 通过 OTel 进 SkillForge | 让 SkillForge 接非自家 agent 的 trace |
| **R-OBS-6** | **span attribute 跟 OTel semconv 对齐**(`gen_ai.system` / `gen_ai.request.model` / etc.) | Phoenix / OpenInference / AgentOps 一致 |

### 🔵 优先级 3:细分 SpanKind(对照 AgentOps)

| # | 借鉴 | 来源 |
|---|---|---|
| **R-OBS-7** | **`agent.thinking / agent.action / agent.decision` 3 sub-kind** | AgentOps `span_kinds.py:25-27` |
| **R-OBS-8** | **`GUARDRAIL` 一等公民 SpanKind** | AgentOps `span_kinds.py:17` |

## SkillForge 已领先(verified)

1. **`root_trace_id` 跨 trace 跨 agent 串联**(OBS-4)— **业内独家**(OTel 无原生等价)
2. **`total_duration_ms / tool_call_count / event_count / status / error` trace aggregate 字段**(OBS-2)— 数据完整,**差 assertion 接入层**
3. **`origin = production | eval` 流量分流**(EVAL-V2 M3a) — 业内独家流量标签
4. **`source = live | legacy` ETL 兼容**(`LlmSpanSource enum`)— 历史 trace 数据迁移机制
5. **完整 blob_ref 机制**(`input_blob_ref / output_blob_ref / raw_sse_blob_ref / blob_status`)— 大 trace payload 分离存储,trace metric 表保持 narrow row

## SkillForge 真差距(对照 5 framework)

### 主要差距
- **trace metric 没作为 EvalRun first-class assertion**(R-OBS-1 / R-OBS-2)
- **不接受 OTLP / 不输出 OTLP**(R-OBS-4 / R-OBS-5)— 跟业内 platform 互通需要桥
- **`agent.thinking / agent.action / agent.decision` 3 sub-kind 缺**(R-OBS-7)
- **`GUARDRAIL` SpanKind 缺**(R-OBS-8)

### 不该担心的"差距"(假象差距)
- **trace_id UUID 而非 OTel hex**:**不影响功能**,只影响协议兼容(R-OBS-4 解决)
- **`attributes_json` JSONB 而非 OTel structured attributes**:**功能等价**,只是 schema 不同
- **自家 REST API 而非 OTLP**:**不影响内部使用**,只影响外部互通

## 关键决策建议

### Question 1:SkillForge 是否要升级 OTel-native?

**答案取决于 SkillForge 商业模式**:

| 场景 | 建议 |
|---|---|
| **SkillForge 只服务自家 agent**(closed ecosystem)| **不升级 OTel-native**,自家 schema 更轻量 |
| **SkillForge 想接第三方 agent**(LangChain / CrewAI 用户的 trace)| **升级 OTel 协议层**(R-OBS-4 / 5 / 6)|
| **SkillForge 想跟业内 platform 互通**(Phoenix / Langfuse export)| **升级 OTel 输出**(R-OBS-4) |
| **SkillForge 想做 ATTRIBUTION-AGENT 商业输出**(让别家用)| **升级 OTel-native**(用户 trace 进来才能归因)|

### Question 2:Trace-as-eval-signal 优先做哪个?

**推荐 R-OBS-1 + R-OBS-2 优先**(短期 1-2 周):
- SkillForge OBS-2 字段已就绪(`total_duration_ms / tool_call_count / event_count / status / error`)
- 只需**EvalRun + EvalTaskEntity 加 trace metric assertion 接入层**
- 让 Scenario YAML 可配 `trace_span_count_max: 50` / `total_duration_max_ms: 30000` / `error_span_count: 0`
- **跟 ATTRIBUTION-AGENT 配套**:trace metric 不达标 → ATTRIBUTION-AGENT 自动归因 → improvement candidate

## 反模式

- ❌ **只收 trace 不当 eval signal**(Phoenix / Langfuse / Opik / Helicone 5 家观测的 trace 都没作 first-class eval signal)
- ❌ **trace + LLM-as-judge 是两条独立轨**(应融合:trace metric 不达标时 LLM-as-judge 输出"为什么慢/为什么失败"的 rationale)
- ❌ **OTel-native or 自家二选一**(应 layered:**自家 schema 内部用 + OTel 协议层外部互通**,SkillForge 当前缺第二层)
- ❌ **`root_trace_id` 跟 OTel parent_span_id 互斥**(应共存:**单 trace 内 parent_span_id chain + 跨 trace 用 `root_trace_id`**,SkillForge 已这么做)
