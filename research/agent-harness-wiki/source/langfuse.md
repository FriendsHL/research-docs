---
type: source
name: langfuse
origin: "/Users/youren/myspace/langfuse/ (Langfuse 开源 LLM engineering platform, MIT)"
ingested_at: 2026-05-16
ingested_commit: da7d91759
ingested_commit_date: 2026-05-15
ingested_version_label: "v3.174.1"
contributed_to:
  - eval/langfuse.md
  - concept/agent-eval.md
  - concept/observability.md
---

# Source — langfuse

> 开源 LLM engineering 平台(MIT license,Y Combinator W23)。Web + Worker + ClickHouse + Postgres + Redis + S3 工程化标杆。docs 在 https://langfuse.com/docs,这里只摘需要的源码定位。

## 仓库结构概览

```
langfuse/
├── web/                              # Next.js frontend + tRPC server
│   └── src/
│       ├── features/                 # 50+ product slices (evals/experiments/prompts/datasets/playground/annotation-queues/automations/score-*/llm-*/mcp/...)
│       ├── server/                   # API + auth + tRPC
│       ├── pages/api/public/         # Public REST/OpenAPI
│       └── ee/                       # Enterprise features
├── worker/                           # Background worker (Node)
│   └── src/
│       ├── queues/                   # 26+ queue handlers
│       ├── services/{ClickhouseWriter,IngestionService,dlq}/
│       ├── api/                      # Worker HTTP API
│       ├── ee/                       # Enterprise worker logic
│       └── constants/default-model-prices.json
├── packages/shared/                  # Domain + server + interfaces
│   └── src/
│       ├── domain/                   # DDD: scores / observations / traces / prompts / datasets / score-configs / dataset-{items,run-items} / automations / webhooks / observation-field-groups / table-view-presets
│       ├── features/{evals,experiments,scores,scoreConfigs,prompts,datasets,annotation,batchAction,batchExport,comments,entitlements,folders,analytics-integrations,model-pricing}/
│       └── server/                   # clickhouse / otel / llm / repositories / queues.ts / s3 / cache / data-deletion / pricing-tiers / ingestion / redis / instrumentation / ...
├── ee/                               # Enterprise (top-level)
├── fern/                             # Fern API SDK 生成
├── specs/                            # specs
├── docker-compose*.yml               # 5 个 compose 配置
├── pnpm-workspace.yaml, turbo.json   # Monorepo
└── .agents/                          # ✨ Langfuse 自己也用 Claude Code skill 模式
    ├── AGENTS.md (-> 顶层 AGENTS.md symlink)
    ├── ARCHITECTURE_PRINCIPLES.md    # Wide Events 2.0 哲学根据地
    └── skills/                       # 16 个 skill: backend-dev-guidelines / clickhouse-best-practices / add-model-price / code-review / datadog-query-recipes / debug-issue-with-datadog / detect-prod-regressions / linear-bug-triage / pnpm-upgrade-package / changelog-writing / turborepo / agent-setup-maintenance / analyze-cloud-costs / frontend-browser-review / skill-creator
```

## 关键 file:line 锚点

### 架构哲学
- `.agents/ARCHITECTURE_PRINCIPLES.md:13-32` — **Wide Events 2.0 / Charity Majors 路线** 11 条原则:
  - "Model observations as the primary analytical unit. A trace is a correlation handle that links related observations, not the only useful entry point."
  - "Prefer wide, richly attributed events over fragmented metrics, logs, and trace records"
  - "Preserve high-cardinality context so users can slice, group, filter, and debug unknown unknowns"
  - "Favor immutable or append-oriented event records for high-volume telemetry. Updates that force read-time deduplication create hidden query costs at scale."
  - "Design storage and query paths around columnar access patterns"
  - "Real-time / near-real-time debugging workflows. Batch processing can help, but it should not make fresh production behavior invisible."
- 引用基础:Charity Majors "Observability 2.0" + Burmistrov "All you need is Wide Events, not Metrics, Logs and Traces" + Langfuse 自家 "Simplifying Langfuse for Scale"(2026-03-10)

### Score 数据模型(Eval 核心实体)
- `packages/shared/src/domain/scores.ts:4` — `ScoreSourceArray = ["API", "EVAL", "ANNOTATION"] as const;`
- `packages/shared/src/domain/scores.ts:5-11` — `ScoreSourceEnum` + `ScoreSourceDomain`(zod)+ `ScoreSourceType`
- `packages/shared/src/domain/scores.ts:13-21` — **`PublicApiCreateScoreSourceDomain`** 用 `as const satisfies readonly ScoreSourceType[]` 编译期保证外部 API 只能创建 `API` 或 `ANNOTATION`,**`EVAL` 保留给内部 evaluator output**
- `packages/shared/src/domain/scores.ts:30-40` — `isAnnotationScoreMissingConfigId` zod refine:**ANNOTATION 来源必须带 configId,除非 dataType 是 `CORRECTION`**
- `packages/shared/src/domain/scores.ts:42-44` — `CORRECTION_NAME = "output"` + `TEXT_SCORE_MAX_LENGTH = 500`
- `packages/shared/src/domain/scores.ts:46-61` — `ScoreDataTypeArray = ["NUMERIC", "CATEGORICAL", "BOOLEAN", "CORRECTION", "TEXT"]`,5 种 dataType,各有专门 zod schema(`NumericData` / `CategoricalData` / `BooleanData` / `CorrectionData` / `TextData`)

### Queue 体系(Worker 调度核心)
- `packages/shared/src/server/queues.ts:322-358` — **`enum QueueName` 32 个**:
  - Ingestion 主链:`TraceUpsert` / `IngestionQueue` / `IngestionSecondaryQueue` / `OtelIngestionQueue` / `OtelIngestionSecondaryQueue`(**主+次双队列分流**)
  - Eval pipeline:`EvaluationExecution` + `EvaluationExecutionSecondaryQueue` + `LLMAsJudgeExecution` + `CreateEvalQueue`(eval 也走双队列)
  - **`EvaluationExecutionSecondaryQueue` 注释明确**:"Separates high-throughput eval projects from other projects"(`queues.ts:327`)
  - **`IngestionSecondaryQueue` / `OtelIngestionSecondaryQueue` 注释**:"Separates high priority + high throughput projects from other projects"(`queues.ts:334,332`)
  - Experiment:`ExperimentCreate`
  - Dataset:`DatasetRunItemUpsert` / `DatasetDelete`
  - Delete:`TraceDelete` / `ScoreDelete` / `ProjectDelete`
  - Export:`BatchExport` / `CoreDataS3ExportQueue` / `MeteringDataPostgresExportQueue`
  - Integration:`PostHogIntegration[+ProcessingQueue]` / `MixpanelIntegration[+ProcessingQueue]` / `BlobStorageIntegration[+ProcessingQueue]`(**3 种 integration,都拆 入口 + processing 双 queue**)
  - Cloud:`CloudUsageMeteringQueue` / `CloudSpendAlertQueue` / `CloudFreeTierUsageThresholdQueue`
  - Data retention:`DataRetentionQueue` / `DataRetentionProcessingQueue`
  - Misc:`BatchActionQueue` / `DeadLetterRetryQueue`(**DLQ 一等公民**)/ `WebhookQueue` / `EntityChangeQueue` / `EventPropagationQueue` / `NotificationQueue`
- `packages/shared/src/server/queues.ts:360-393` — **`enum QueueJobs` 32 个对应 job 名**(queue 跟 job 1:1)
- `worker/src/queues/shardedQueueRegistry.ts` — **分片队列注册** — 多租户性能隔离机制
- `worker/src/queues/workerManager.ts` — worker 管理器

### Eval 系统
- `web/src/features/evals/server/router.ts:142,475` — `CreateEvalTemplateInputSchema` + `ctx.prisma.evalTemplate.findMany` — **EvalTemplate 有版本,跨 project 复用**
- `web/src/features/evals/server/router.ts:128-141` — `validateEvaluators(...)` — 数据库加载 evaluator 后做校验,失败的 evaluator 日志到 OTel(注释 `:123`:"All evaluators are expected to pass the validation. If an evaluator fails validation, it will be logged to Otel.")
- `web/src/features/evals/server/router.ts:214-234` — `validateEvalTemplateCanRun` — eval 跑前的 preflight
- `web/src/features/evals/server/evaluator-preflight.ts` — 完整 preflight 检查
- `web/src/features/evals/server/addDatasetRunItemsToEvalQueue.ts` — **production loop 核心**:dataset run 完成自动入 eval queue
- `web/src/features/evals/server/defaultEvalModelRouter.ts` — 默认 eval model 路由(LLM-as-judge 用哪个模型)
- `packages/shared/src/features/evals/evalConfigBlocking.ts` — eval blocking 配置(同步 vs 异步)
- `packages/shared/src/features/evals/observationForEval.ts` — observation 入 eval 的转换
- `packages/shared/src/features/evals/outputDefinition.ts` — eval 输出格式定义
- `packages/shared/src/features/evals/validateEvaluatorFilters.ts` — evaluator filter 校验
- `web/src/features/evals/components/ragas-logo.tsx` — **Ragas 集成**(RAG eval 行业标杆框架)
- `web/src/features/evals/components/eval-template-detail.tsx` + `eval-templates-table.tsx` — eval template UI 管理
- `web/src/features/evals/components/{evaluator-form,inner-evaluator-form,run-evaluator-form}.tsx` — evaluator 配置 UI

### OTel 摄入路径(Observability 2.0 实现)
- `packages/shared/src/server/otel/OtelIngestionProcessor.ts` — OTel 协议进入主处理器
- `packages/shared/src/server/otel/ObservationTypeMapper.ts` — **OTel span ↔ Langfuse observation 映射**(关键:Langfuse 把 OTel 抽象转成自己的 wide event)
- `packages/shared/src/server/otel/attributes.ts` — semantic attributes
- `worker/src/queues/otelIngestionQueue.ts` — Worker queue handler
- `worker/src/queues/otelProjectTracking.ts` — Otel project 跟踪
- `packages/shared/src/server/redis/otelIngestionQueue.ts` — Redis queue 实现

### ClickHouse(主分析库)
- `packages/shared/src/server/clickhouse/{clickhouse-logger,client,measureAndReturn,queryTracking,schema,schemaUtils}.ts` — ClickHouse 完整客户端
- `packages/shared/src/server/repositories/{traces,observations,scores,dataset-run-items,experiments,job-executions,events}.ts` + `*_converters.ts` — ClickHouse row ↔ domain 转换
- `worker/src/services/ClickhouseWriter/` — Worker 写 ClickHouse

### Domain Model(DDD)
- `packages/shared/src/domain/{automations,dataset-items,dataset-run-items,observation-field-groups,observations,prompts,score-configs,scores,table-view-presets,traces,webhooks}.ts` — 11 个 domain 实体
- `packages/shared/src/eventsTable.ts` + `observationsTable.ts` — wide event 表 schema

### Worker Features(Shared 内)
- `packages/shared/src/features/{evals,experiments,scores,scoreConfigs,prompts,datasets,annotation,batchAction,batchExport,comments,entitlements,folders,analytics-integrations,model-pricing}/` — 14 个 cross-feature shared module

### `.agents/skills/` 16 个 skill(他们自家 dogfood Claude Code)
- backend-dev-guidelines / clickhouse-best-practices / add-model-price / code-review / datadog-query-recipes / debug-issue-with-datadog / detect-prod-regressions / linear-bug-triage / pnpm-upgrade-package / changelog-writing / turborepo / agent-setup-maintenance / analyze-cloud-costs / frontend-browser-review / skill-creator

### LLM 抽象(provider 路由)
- `packages/shared/src/server/llm/fetchLLMCompletion.ts` + `compileChatMessages.ts` + `testModelCall.ts` + `types.ts` + `errors.ts` + `utils.ts` + `baseUrlValidation.ts` + `getInternalTracingHandler.ts` + `internalTraceEvents.ts` — provider-agnostic LLM 调用层
- `worker/src/constants/default-model-prices.json` — **模型定价集中维护**(每加 model 走 `.agents/skills/add-model-price/SKILL.md`)

## 与 wiki 现有记录的关系

### 印证
- **Wide Events 2.0 哲学**(`.agents/ARCHITECTURE_PRINCIPLES.md`)印证 [[../harness/codex]] `otel/` crate "5 GOAL metric + 7 compaction enum" 把 lifecycle 暴露成 event 的做法
- **`addDatasetRunItemsToEvalQueue.ts` production loop** 印证 [[../concept/agent-eval]] 8 维设计空间 "production_loop" 维度,但比当前描述完整很多
- **`@observe()` 装饰器 + 钩子做 telemetry** 跟 [[../harness/hermes-agent]] `commit_memory_session` 钩子模式同源

### 矛盾
- 当前 [[../concept/agent-eval]] 没覆盖 **Score `Source × DataType` 3×5 矩阵**(scores.ts:4,46)— scorer 维度需要扩
- 当前 wiki 没覆盖 **main + secondary 双队列分流多租户** 工程模式(`queues.ts:327,332,334`)— observability 跟 eval 章节都需补
- 当前 wiki 没 **Wide Events 2.0 vs OTel 1.0 三分法** 对比

### 新增
- **eval/langfuse.md**(新 page,首个 eval-framework 页)
- **concept/observability.md**(新 concept page,Wide Events 2.0 哲学)

## 事实漂移 / 注意点

- Langfuse 版本号 v3.174.1 是 **truthful semver**(跟 Codex `0.0.0-dev` 不同),但 monorepo 内 `web/package.json` + `worker/package.json` 都跟根 `package.json` 同版本,意味着**整个 monorepo 同步发版**
- **本地 docs/ 不存在**,只有 README + 4 国语言翻译(`README.cn.md` / `README.ja.md` / `README.kr.md`),真 docs 在 https://langfuse.com/docs
- **`AGENTS.md` 是 symlink** 指向 `.agents/AGENTS.md`,`CLAUDE.md` 是 symlink 指向 `AGENTS.md` — 这种**符号链接路由 + 真源在 `.agents/`** 是值得 SkillForge 借鉴的工程组织
- ClickHouse 主分析库,Postgres 是 metadata + auth 库 — **不是单库**,Langfuse 的存储是异构的(Postgres + ClickHouse + Redis + S3)
- **Worker queue 32 个**(`queues.ts:322-358`)+ **`worker/src/queues/` 26 个文件** + **`packages/shared/src/server/redis/` 30+ queue 实现** — queue 体系是 Langfuse 工程化最深的部分
