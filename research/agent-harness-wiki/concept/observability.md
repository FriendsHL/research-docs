---
type: concept
name: observability
status: draft
sources:
  - "[[../source/langfuse]]"
  - "[[../source/codex]]"
  - "[[../source/opik]]"
  - "/Users/youren/myspace/langfuse/.agents/ARCHITECTURE_PRINCIPLES.md"
  - "/Users/youren/myspace/codex/codex-rs/otel/src/"
  - "/Users/youren/myspace/codex/codex-rs/rollout/src/"
related_harnesses: [codex, langfuse, opik]
---

# Observability for Agent Systems(Wide Events 2.0 vs 三分法)

> **本页跟 [[agent-protocols]] §6 OTel 节双向**:本页讨论 _"工程实战路线选择"_(Wide Events 2.0 vs OTel 1.0 三分法),[[agent-protocols]] 把 OTel 定位在 _"协议栈第 5 层(telemetry 侧向贯穿)"_。OTel 在 wiki 内引用 21 页,是引用最广的协议 — GenAI semconv(2026-05-17 _Development_ 状态)已经把 MCP semantics 纳入。

## 定义

**Agent observability** = 把 agent 系统的运行时行为变成可检索 / 可关联 / 可分析的事件流,使得 debug / eval / 自进化 / 安全审计 4 件事都可基于同一份数据底层完成。

## 设计空间(7 维)

### 1. 基本分析单元
- **Trace 主义**(OTel 1.0 传统):**trace 是分析根**,observation/span 是 trace 的子节点。查询语言围绕 trace 展开。
- **Wide event 主义**(Langfuse / Charity Majors 路线):**observation 是分析单元**,trace 是 correlation handle(`.agents/ARCHITECTURE_PRINCIPLES.md:13`:"A trace is a correlation handle that links related observations, **not the only useful entry point**")。查询直接 slice observation。

### 2. 数据丰富度
- **三分法**(metrics / logs / traces 拆分,OTel 1.0):metric 是聚合数值,log 是文本,trace 是调用图。**查询时要 join 三家**重建一次行为。
- **Wide events**(Langfuse 哲学):**事件本身高 cardinality 高维**,所有属性都进同一行,查询时 narrow field selection + time-bounded scan 即可,**不需要 join**。

### 3. 可变性
- **Mutable**(传统 APM,metric 写完可更新)— `.agents/ARCHITECTURE_PRINCIPLES.md:第 4 条` 反对:"Updates that force read-time deduplication create hidden query costs at scale"
- **Append-only / immutable**(Langfuse / Codex 共同选)— Codex `rollout/src/recorder.rs` JSONL append-only + Langfuse trace 不可改写

### 4. 存储引擎
- **行存**(Postgres / MySQL)— OLTP 友好,大规模 trace 查询慢
- **列存**(ClickHouse / BigQuery / Druid)— OLAP 友好,大规模 trace 查询经济(Langfuse 选 ClickHouse,`.agents/ARCHITECTURE_PRINCIPLES.md:第 6 条`:"Design storage and query paths around columnar access patterns")
- **混合**(Langfuse 实际:Postgres 存 metadata + auth,ClickHouse 存 trace/observation/score,Redis 缓存 + queue,S3 大 blob)— **多库异构,各司其职**

### 5. 摄入路径
- **OTel 协议**(行业标准,Langfuse `OtelIngestionQueue` 接受)
- **自定 SDK**(Langfuse 自家 Python `@observe()` / JS `Langfuse()` client)
- **框架自动 instrumentation**(Langfuse 23+ 集成:OpenAI drop-in / Langchain callback / LlamaIndex callback / LiteLLM proxy — 用户几乎零代码)
- **JSONL 文件**(Codex `rollout/` 写本地 .jsonl,后续 replay 或 importer 摄入)

### 6. 实时性
- **离线分析为主**(传统数据仓库)— `.agents/ARCHITECTURE_PRINCIPLES.md:第 10 条` 反对:"Batch processing can help, but it should not make fresh production behavior invisible"
- **Near-real-time**(Langfuse + Codex 共同选)— production 行为应可立即查询(秒级到分钟级)
- **真实时**(`<<1s` 延迟,WebRTC + live dashboard)— Codex `realtime-webrtc/` crate 有此能力但偏 voice agent

### 7. Trace tree 表达
- **Single linear trace**(简单 LLM 应用,无 sub-agent)— 一个 trace = 一次调用
- **Nested tree**(agent 调 tool 调 sub-agent)— Langfuse / Codex 都支持任意嵌套
- **Forest / DAG**(multi-agent 协作场景)— 多 root,跨 trace 关联需自定 correlation key

## 各 harness/framework 怎么实现

### Langfuse — **Wide Events 2.0 标杆**
- `.agents/ARCHITECTURE_PRINCIPLES.md:13-32` 11 条原则,引用 Charity Majors + Burmistrov
- 基本单元:**Observation**,Trace 是 correlation handle
- 数据丰富度:wide event(高 cardinality 高维)
- 可变性:**Append-only**(immutable event records)
- 存储:ClickHouse 主分析库 + Postgres metadata + Redis queue/cache + S3 blob
- 摄入:OTel + SDK + 23+ 框架自动 instrumentation
- 实时性:Near-real-time(queue 异步 + ClickHouse 列存查询秒级)
- Score 数据模型 3×5 矩阵(`scores.ts:4-61`)统一所有 evaluator 输出
- 32 QueueName(`queues.ts:322-358`)+ main/secondary 双队列分流多租户

### [[../eval/opik]] — **Java + Spring + MySQL platform observability**(2026-05-16 新加)

Opik 跟 Langfuse 同走 Wide Events 2.0 哲学,**但技术栈完全不同**:

| 维度 | Langfuse | Opik |
|---|---|---|
| Backend | Node.js | **Java 17 + Spring Boot + Maven** |
| Main metadata DB | Postgres | **MySQL** |
| Main analytical DB | ClickHouse | ClickHouse(都用) |
| Cache + Queue | Redis | Redis(都用) |
| Blob | S3 | S3(都用) |
| Scale 标榜 | "battle-tested" | **"40M+ traces/day"** |
| 服务化粒度 | 3 个(web / worker / shared) | **6 个**(opik-backend / opik-frontend / opik-python-backend / opik-sandbox-executor-python / opik-guardrails-backend / opik-documentation) |
| Optimization 闭环 | ❌(只 experiment 对比) | **✅ `sdks/opik_optimizer/` 7 算法自动改 prompt** |
| Alert 一等公民 | ❌(只 webhook) | **✅ `AlertDAO` + `AlertEventType` + `AlertTriggerConfig*`** |
| Guardrails 一等公民 | ❌(用户自接) | **✅ 独立 service `apps/opik-guardrails-backend/`** |
| Sandbox executor | ❌ | **✅ 独立 service `apps/opik-sandbox-executor-python/`** |
| Pytest 集成 | SDK 支持 | **first-class(`test_case.py` + `test_result.py`)** |
| Ruby SDK | ❌ | **✅ via OTel** |

**Opik 走 Java/Spring 路线对 SkillForge 启示最大**(SkillForge 也是 Java + Spring 生态):
- **MySQL vs Postgres**:SkillForge 当前 Postgres,Opik 选 MySQL — 选哪个差异不大,看生态偏好
- **ClickHouse 切换**:Opik 在 Java 后端调 ClickHouse 用 `apps/opik-backend/src/main/java/com/comet/opik/infrastructure/`,SkillForge 切 ClickHouse 时可参考
- **6 个独立 app** 粒度:opik-backend(Java 主)/ opik-python-backend(Python helper)/ opik-sandbox-executor-python(用户沙箱)/ opik-guardrails-backend(独立 service)/ opik-frontend / opik-documentation — **SkillForge 微服务化方向参考**
- **40M+ traces/day Java/JVM 性能验证** — SkillForge Java 后端选型方向被验证

### [[../harness/codex]] — **工程化最深的 single-runtime observability**
- `otel/` crate 完整 OTLP 导出
- 基本单元:Trace + 5 GOAL metric(`codex.goal.{created,completed,duration_seconds,token_count,budget_limited}`)+ 7 个 compaction telemetry 枚举
- 数据丰富度:**Trace + Metric 双轨**(尚未完全转 wide events)
- 可变性:**Append-only**(`rollout/src/recorder.rs` JSONL)
- 存储:本地 JSONL + SQLite 索引 + 自动 git_info 关联
- 摄入:Rust 程序内嵌的 OTLP exporter,本地为主
- 实时性:Near-real-time(JSONL append,SQLite 索引秒级查询)
- 跟 Langfuse 互补:Codex 是 agent runtime 自带 observability,Langfuse 是中心化 platform

### [[../harness/hermes-agent]] / [[../harness/openclaw]] — **轻量自包含 trace**
- Hermes:`commit_memory_session` 钩子 + 自家 trace 格式
- OpenClaw:`extensions/memory-wiki/` + `extensions/qa-lab/` 自家 trace 模式
- 基本单元:每个项目自定
- 数据丰富度:中(取决于装饰器粒度)
- 可变性:取决于 store
- 存储:轻量(SQLite / 文件)
- 摄入:自家 SDK + 钩子
- 实时性:Near-real-time(本地)
- **没有显式 Wide Events 2.0 设计哲学**,主要服务自家 agent runtime

### [[../harness/claude-code]] — **session trace + 钩子驱动**
- Trace 主要靠 27 个 hooks 驱动 telemetry(详见 [[hooks-design]])
- 基本单元:Session
- 摄入:Hook 输出 + SDK 钩子
- 实时性:Real-time(IPC 流)
- **没有内建 server-side dashboard**,需用户自接 Langfuse / Helicone 等

### SkillForge(自身)
**当前已知**(从 CLAUDE.md):
- Langfuse 风格 trace(`@observe()` 装饰器 + SSE 流式)
- Session replay
- Trace 可观测
- ❓ 基本分析单元是 trace 还是 observation?
- ❓ 存储行存还是列存?
- ❓ 实时性多久?

## Trade-off

### Wide Events 2.0 vs OTel 1.0

| | Wide Events 2.0(Langfuse 路线) | OTel 1.0 三分法 |
|---|---|---|
| **优点** | | |
| Slice / group / filter 自由 | ✅ 不需预定义未来问题 | ❌ metric 预聚合限制查询维度 |
| High-cardinality 友好 | ✅ 高维 + 列存 + time-bounded scan | ❌ metric 高 cardinality 爆炸 |
| Debug unknown unknowns | ✅ 列存任意 slice | ❌ 必须先知道问题才能查 |
| 单一查询路径 | ✅ 一份事件流 | ❌ join metric + log + trace |
| **缺点** | | |
| 单点查询成本 | ❌ 列存 scan 一定列贵于 metric 单值 | ✅ metric 直接读 |
| 存储成本 | ❌ wide event 体量大 | ✅ metric 紧凑 |
| 历史聚合 | ❌ 实时聚合慢 | ✅ metric 预聚合即时 |
| 生态成熟度 | ❌ 较新,工具少 | ✅ Prometheus / Grafana / Datadog 完整生态 |

### 行存 vs 列存(LLM observability 场景)

| 规模 | 推荐 | 理由 |
|---|---|---|
| < 1K traces/day | Postgres 行存够 | OLTP 简单 |
| 1K-10K traces/day | Postgres + 索引调优 | 仍可撑 |
| **> 10K traces/day** | **ClickHouse 列存**(Langfuse 经验) | 行存查询变慢,scan cost 爆炸 |
| > 1M traces/day | ClickHouse + 分区 + TTL + 异步聚合 | 必须列存 + 预聚合补充 |

## SkillForge 当前选择 + 演进方向

**当前**:Langfuse 风格 trace + Postgres 持久化(SkillForge CLAUDE.md)

**演进选项**:
1. **明确 trace tree 模型** — Trace 主义 vs Observation 主义?建议 **Observation 主义**(对齐 Langfuse,且 SkillForge "Eval Pipeline" 需要 score-per-observation 而非 score-per-trace)
2. **加 OTel 摄入路径** — SkillForge 接受 OTel 协议进入,然后**用户自由选 SkillForge 自家 dashboard 还是导出到 Langfuse / Helicone** — 互通而非锁定
3. **ClickHouse 引入时机** — 走 Langfuse 经验:**< 10K traces/day Postgres 够,> 10K 切 ClickHouse**(SkillForge 现在不需要,但留 trace schema 演进余地:**字段宽松 + UUID 主键 + 时间分区友好**,后续切 ClickHouse 不破坏 API)
4. **Append-only 不变量** — SkillForge trace 表加 immutability 约束,**禁止 UPDATE,只允许 INSERT**(对齐 `.agents/ARCHITECTURE_PRINCIPLES.md:第 4 条`)
5. **Score 数据模型 3×5 矩阵** — Eval Pipeline 直接借鉴 Langfuse `ScoreSource × ScoreDataType`(详见 [[concept/agent-eval]] + [[../eval/langfuse]])

判定:**SkillForge 应明确走 Wide Events 2.0 路线**(对齐 Langfuse),但留 ClickHouse 切换余地。**OTel 进入是免费 win**(零额外工程成本,自动跟 Langfuse 互通)。

## 学习视角:Wide Events 2.0 是怎么来的

### 历史脉络(2014 → 2026)

- **2014**:Honeycomb 创始 Charity Majors 提出"observability != monitoring"
- **2018**:Charity Majors blog tag "Observability 2.0" 系列开始
- **2020**:OpenTelemetry 项目成立(metric + log + trace 三分法,继承 OpenTracing + OpenCensus)
- **2024**:Burmistrov "All you need is Wide Events, not Metrics, Logs and Traces" 论文 / 博客火
- **2026-03**:Langfuse "Simplifying Langfuse for Scale" 博客 + `.agents/ARCHITECTURE_PRINCIPLES.md` 落地为代码级硬约束
- **当前**:**Wide Events 2.0 是 LLM observability 的事实路线**(Langfuse / Helicone / Phoenix 都走类似哲学),OTel 1.0 三分法在 LLM 场景被认为是"对的工具用在错的领域"

### 为什么 LLM 场景特别适合 Wide Events

1. **LLM 调用是高 cardinality 事件**(模型名 + temperature + max_tokens + prompt hash + user_id + project_id + ...),metric 预聚合维度爆炸
2. **Debug LLM 问题需要 unknown unknowns 查询**(为什么这条 trace 输出怪?哪条 prompt 改了?)— metric 不能回答
3. **LLM 应用迭代快**,trace schema 频繁变 — wide events 天然 schema-flexible
4. **Token cost 是 per-call 高维属性** — 必须 wide event 才能 slice by model × project × user 算成本

### Codex 跟 Langfuse 的 observability 风格对比

| | Codex(runtime observability) | Langfuse(platform observability) |
|---|---|---|
| 定位 | Agent runtime 自带 | 中心化 platform |
| 摄入对象 | 自家 codex agent | 任意 LLM 应用 |
| 数据模型 | Trace + 5 GOAL metric + 7 compaction enum | Wide events(observation 主义) |
| 存储 | 本地 JSONL + SQLite | ClickHouse + Postgres + Redis + S3 |
| 查询 | replay / inspect | Web dashboard + SDK + REST API |
| **互补点** | Codex agent 生成的 trace 可走 OTel → Langfuse 集中查询 |

**Codex 跟 Langfuse 是天然合作伙伴**:Codex 是 trace 源头(agent runtime),Langfuse 是 trace 汇聚 + 分析平台。

## 反模式

- ❌ **Metric 替代 trace** — LLM 场景任何 metric 都隐藏 cardinality,无法 debug 个案
- ❌ **trace 跟 metric 双轨独立** — 实际查询必须 join 才能定位个案 → join 成本爆炸
- ❌ **Mutable trace**(改写历史)— read-time dedup 隐性 query 成本,违反 `.agents/ARCHITECTURE_PRINCIPLES.md:第 4 条`
- ❌ **预定义 dashboard 限制 slice** — 反 Charity Majors 哲学,unknown unknowns 不可查
- ❌ **Batch-only 处理让 fresh production 不可见** — 违反 `.agents/ARCHITECTURE_PRINCIPLES.md:第 10 条`
- ❌ **行存撑大规模 trace** — > 10K traces/day 后查询变 cliff
- ❌ **不接受 OTel 摄入** — 跟业内 23+ 框架生态隔离
- ❌ **Wide event 字段无上限** — 像 `TEXT_SCORE_MAX_LENGTH = 500`(`scores.ts:44`)这种字段硬上限是必要的,不然 LLM rationale 写到 10KB 一行
