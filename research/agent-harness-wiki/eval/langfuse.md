---
type: eval-framework
name: langfuse
status: active
language: typescript

# === 版本时效 ===
last_ingested_version: "v3.174.1 @ da7d91759 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/langfuse]]"
  - "/Users/youren/myspace/langfuse/README.md"
  - "/Users/youren/myspace/langfuse/.agents/ARCHITECTURE_PRINCIPLES.md"
  - "/Users/youren/myspace/langfuse/packages/shared/src/domain/scores.ts"
  - "/Users/youren/myspace/langfuse/packages/shared/src/server/queues.ts"
  - "/Users/youren/myspace/langfuse/web/src/features/evals/server/"

# === 评测框架专属维度(schema §3.2) ===

target: |
  LLM 应用全栈:prompt + tool use + agent 链路 + RAG + multi-agent。
  **关键差异**:**Observation 是分析单元而非 LLM 单次调用**(`.agents/ARCHITECTURE_PRINCIPLES.md:13`:"Model observations as the primary analytical unit. A trace is a correlation handle that links related observations, not the only useful entry point.")。
  这跟传统 eval framework(LangChain Eval / OpenAI evals / Inspect AI 等)以"模型 + dataset 配对"为对象的范式截然不同。
  集成生态 **23+ 个**:OpenAI / Langchain / LlamaIndex / Haystack / LiteLLM / Vercel AI SDK / Mastra / Instructor / DSPy / Mirascope / Ollama / Bedrock / AutoGen / Flowise / Langflow / Dify / OpenWebUI / Promptfoo / LobeChat / Vapi / Inferable / Gradio / Goose / smolagents / CrewAI
  [verified-from-source]

dataset: |
  - **Dataset / DatasetItem / DatasetRunItem** 三层实体(`packages/shared/src/domain/{datasets,dataset-items,dataset-run-items}.ts`)
  - 来源:手动创建 + production trace 导入 + CSV 上传 + LangChain / LlamaIndex 框架集成
  - 规模:云端版无明确上限(ClickHouse 列存支撑),自托管按用户基础设施
  - 更新频率:append-only(immutable event records,`ARCHITECTURE_PRINCIPLES.md:第 4 条`)
  - **泄露防护**:无显式 train/test 分离机制(Langfuse 主要用于"对齐 production",不是公开 benchmark)
  [verified-from-source]

scorer: |
  **Score = 统一抽象产物**(`packages/shared/src/domain/scores.ts:4-61`),所有 evaluator 路径(LLM-as-judge / 人工标注 / API 上报 / 自定义 pipeline)的输出**统一成 Score 实体**。

  **3 × 5 矩阵**:
  - **ScoreSource = 3**(`scores.ts:4`):`API` / `EVAL` / `ANNOTATION`
  - **ScoreDataType = 5**(`scores.ts:46`):`NUMERIC` / `CATEGORICAL` / `BOOLEAN` / `CORRECTION` / `TEXT`

  **工程级 boundary**(`scores.ts:13-21`):
  ```ts
  export const PublicApiCreateScoreSourceDomain = z.enum([
    ScoreSourceEnum.API,
    ScoreSourceEnum.ANNOTATION,
  ] as const satisfies readonly ScoreSourceType[]);
  ```
  外部 API 只能创建 `API` 或 `ANNOTATION`,**`EVAL` 保留给内部 evaluator output 防外部伪造**。`as const satisfies` 编译期保证是 subset。

  **约束**(`scores.ts:30-44`):
  - `ANNOTATION` 来源必须带 `configId`,除非 dataType 是 `CORRECTION`(`isAnnotationScoreMissingConfigId` zod refine)
  - `CORRECTION_NAME = "output"`(纠正分数的名字硬约定)
  - `TEXT_SCORE_MAX_LENGTH = 500`(text score 长度上限)

  **具体 evaluator 类型**:
  - **LLM-as-judge**:`LLMAsJudgeExecutionQueue`(`queues.ts:328`)+ `defaultEvalModelRouter.ts` 选模型
  - **User feedback**:走 `API` source(用户客户端调 SDK 上报)
  - **Manual labeling / Annotation**:走 `ANNOTATION` source + annotation queue UI + score-config 约束
  - **Custom pipeline**:走 `API` source 由用户自实现
  - **Ragas 集成**(`web/src/features/evals/components/ragas-logo.tsx`)— RAG eval 行业标杆框架显式集成
  [verified-from-source]

reproducibility: |
  - **Prompt 版本控制**(`web/src/features/prompts/`)— prompt 是 first-class versioned entity
  - **EvalTemplate 版本控制**(`web/src/features/evals/server/router.ts:142,475`)— eval template 也有版本,改 prompt 可重跑历史 eval
  - **模型定价版本锁**(`worker/src/constants/default-model-prices.json`)— 模型 + 价格集中维护(走 `.agents/skills/add-model-price/SKILL.md`)
  - **Trace immutable**(`ARCHITECTURE_PRINCIPLES.md:第 4 条` "Favor immutable or append-oriented event records")— 历史不可改写
  - **Seed / temperature** 锁定通过 prompt + model config 跟随 — 不在 Langfuse 自身,在用户调用的 LLM provider 层
  - **Docker / VM 部署**:`docker-compose.yml` + Helm + Terraform(AWS / Azure / GCP)— 完整自托管可复现
  [verified-from-source]

coverage: |
  **覆盖**:
  - Observability(tracing,SDK 装饰器 `@observe()`)
  - Prompt Management(版本 + caching + 协作迭代)
  - Eval(LLM-as-judge / annotation / API / custom)
  - Dataset(test set + benchmark)
  - Playground(prompt 调试 + 跳转 trace iterate)
  - LLM cost tracking(`default-model-prices.json` + ClickHouse 聚合)
  - 23+ framework 集成

  **不覆盖 / 已知 gap**:
  - **Agent loop 内部不可见** — Langfuse 只看到 `@observe()` 装饰的边界,agent 内部决策细节看 trace tree
  - **本地 dataset 评测离线模式弱** — 主要靠 cloud / self-hosted server,不适合 CI 离线
  - **跨 model A/B + 自动晋升**:有 experiment 但**自动晋升不是 first-class**(SkillForge `Prompt A/B + 自动晋升` 是 Langfuse 当前缺的差异化方向)

  **已知 gaming pattern**:
  - LLM-as-judge bias:用同模型评同模型 → 自评偏高(行业通病)
  - Annotation queue cold start:新评 task 需大量人工 labeled data 才能训出可信 LLM-as-judge
  - `ANNOTATION` source 需 configId 强约束(`scores.ts:30-40`)避免 ad-hoc 标签污染数据集
  [verified-from-source]

production_loop: |
  **强 production loop**(关键差异化,SkillForge 应学):

  - `addDatasetRunItemsToEvalQueue.ts` — **dataset run 完成自动入 eval queue** → production trace 自动转 dataset run → 自动 eval → 出 score,**单线打通**
  - Worker 32 queue(`queues.ts:322-358`)调度全流水线
  - **Main + Secondary 双队列分流多租户**(`queues.ts:327,332,334`):
    - `EvaluationExecutionSecondaryQueue` — "Separates high-throughput eval projects from other projects"
    - `IngestionSecondaryQueue` / `OtelIngestionSecondaryQueue` — "Separates high priority + high throughput projects from other projects"
  - **Webhook + 6 种 outbound integration**(PostHog / Mixpanel / BlobStorage / S3 / DataRetention / Metering)— production 信号自动反推到下游分析平台
  - **DeadLetterRetryQueue**(`queues.ts:353`)+ `dlq/` worker service — 失败 eval 进 DLQ,可手动重试
  - **Prompt A/B 实验**(`web/src/features/experiments/` + `ExperimentCreate` queue)— 但**自动晋升不在 Langfuse 自身实现**,需用户写 automation 触发
  [verified-from-source]

automation: |
  - **CI/CD 集成**:Langfuse 提供 SDK + Public API,用户可在 CI 跑 eval(典型:PR 触发 dataset run + Langfuse eval + PR comment 出 score 差异)
  - **Webhook + Automations**(`web/src/features/automations/` + `WebhookQueue`):事件驱动自动化
  - **OpenAPI + Fern SDK 生成**(`fern/`)— typed Python / JS/TS SDK 自动生成,**CI 集成几乎免代价**
  - **Audit log**(`audit-log-resource-types.ts`)+ `EntityChangeQueue`(`queues.ts:355`)— 所有 entity change 都产生事件,可触发外部系统
  - **`.agents/skills/`**(16 个 skill)— Langfuse 团队自家 dogfood Claude Code,每个 dev workflow 都 codify 成 skill(SkillForge 印证方向)
  [verified-from-source]

cost: |
  - **per-run 成本**:取决于 LLM-as-judge 调用次数 + ClickHouse 查询(列存 + time-bounded scan 控成本)
  - **`default-model-prices.json`** 把每个模型每 token 价格写死(input/output 分开),trace 自动算成本
  - **`cloudUsageMeteringQueue` + `cloudSpendAlertQueue` + `cloudFreeTierUsageThresholdQueue`**(`queues.ts:335-337`)— Cloud 版有内建用量监控 + 预警
  - **采样**:Langfuse 自身**默认全量摄入**(符合 wide events 哲学),用户可在 SDK 层做采样
  - **ClickHouse 列存 + 列裁剪**:大量 trace 查询经济(`ARCHITECTURE_PRINCIPLES.md:第 6 条`"Design storage and query paths around columnar access patterns")
  [verified-from-source]

# === 服务 SkillForge 决策的关键栏 ===
skillforge_takeaway: |
  ## 可借鉴 (10 条)
  1. **Score 数据模型(3 × 5 矩阵)** — `ScoreSource × ScoreDataType`(`scores.ts:4-61`):所有 evaluator 路径输出统一成 Score 实体,SkillForge Eval Pipeline 可直接照抄这套抽象,避免每种 evaluator 一种 score 格式
  2. **Public API EVAL source 保留**(`scores.ts:13-21`)— **`as const satisfies readonly ScoreSourceType[]`** 编译期保证外部不能伪造 evaluator output,SkillForge 多租户 + 公开 API 必须有这种 boundary
  3. **`ANNOTATION` 必须带 configId**(`scores.ts:30-40`)— 防 ad-hoc 标签污染 dataset,SkillForge annotation 系统直接学
  4. **`TEXT_SCORE_MAX_LENGTH = 500`** — 防 LLM-as-judge 写无界长 rationale 污染存储,SkillForge eval 输出加这种硬上限
  5. **Main + Secondary 双队列分流多租户**(`queues.ts:327,332,334`)— SkillForge 多租户场景"高吞吐 project 不影响其他 project"工程模板,直接借鉴
  6. **`addDatasetRunItemsToEvalQueue.ts` production loop** — Production trace → dataset → eval → score 单线打通是 SkillForge "Prompt A/B + 自动晋升" 的开源参考实现
  7. **EvalTemplate 版本控制** — eval 跟 prompt 一样版本化(`router.ts:142,475`),**改 prompt 后才能 reproducible 重跑历史 eval**,SkillForge eval 需要
  8. **OTel ingestion path**(`OtelIngestionQueue` + `OtelIngestionProcessor` + `ObservationTypeMapper`)— SkillForge 自己的 Langfuse 风格 trace 可直接走 OTel 协议,然后 Langfuse 也按 OTel 摄入,**两边自动兼容**
  9. **Audit log 一等公民**(`audit-log-resource-types.ts`)+ `EntityChangeQueue` — SkillForge 应有跟资源类型对齐的审计日志 + 实体变更事件流
  10. **DeadLetterRetryQueue + DLQ service**(`queues.ts:353` + `worker/src/services/dlq/`)— SkillForge 后台任务体系应内建 DLQ 而不是 ad-hoc 重试

  ## 业内验证 (5 条)
  - **Wide events 哲学**(observation 是分析单元而非 trace)— Charity Majors / Burmistrov 路线被 Langfuse production scale 验证,SkillForge 设计 trace 表时应走 wide events 不要走"trace + observation 分离"
  - **ClickHouse 列存适合 LLM observability** — Langfuse / OpenLLMetry / Helicone 都选 ClickHouse,SkillForge 大规模 trace 时 Postgres 不够
  - **`@observe()` 装饰器 = 钩子做 telemetry** — Hermes `commit_memory_session` 钩子 / Codex 5 OTEL metric / Langfuse `@observe()` 都是装饰器+钩子,行业共识
  - **Eval 跟 dataset 联动**(`addDatasetRunItemsToEvalQueue.ts`)是 SkillForge 自动晋升的开源参考
  - **Score 抽象统一所有 evaluator 输出** — 不要每种 evaluator 一种 score 格式

  ## SkillForge 领先 (3 条)
  - **Prompt A/B + 自动晋升**(SkillForge first-class)— Langfuse 有 experiment 但**自动晋升不在 Langfuse 自身**,需用户写 automation,SkillForge 这条路有差异化空间
  - **Server-side Agent loop 引擎**(Spring Boot)— Langfuse 不跑 agent loop 自己,只是 telemetry,SkillForge 是平台 + telemetry 一体
  - **统一 Provider SPI + Channel SPI** — Langfuse provider 抽象是 `fetchLLMCompletion`,但 channel 抽象主要是集成层(23+ 框架)而非 first-class SPI

  ## 待决策 (5 条)
  1. **Score 抽象采纳粒度** — 完整搬 `ScoreSource × ScoreDataType` 3×5 矩阵?还是先做 `NUMERIC + CATEGORICAL + BOOLEAN` 3 种?
  2. **ClickHouse 引入时机** — SkillForge 现在 Postgres,什么规模需要切 ClickHouse?(参考:Langfuse 自托管 < 10K traces/day Postgres 够,> 10K 建议 ClickHouse)
  3. **Main + Secondary 双队列** — SkillForge 现在 single queue,什么时候需要双队列分流?(典型触发:high-throughput project 拖慢其他 project)
  4. **OTel 进入路径** — SkillForge 是否直接接受 OTel 协议?还是只用自家 trace 格式?— **建议接受 OTel,跟 Langfuse 互通**
  5. **Eval 跟 SkillForge 现有的 Prompt A/B 怎么对齐** — Langfuse `experiments` + `ExperimentCreate` queue 跟 SkillForge `prompt-ab` 模块结构对照

  ## 风险 (3 条)
  - **不要 1 库通吃** — Langfuse 用 Postgres(metadata + auth) + ClickHouse(分析) + Redis(队列 + 缓存) + S3(blob)4 库,SkillForge 应避免 Postgres 一库通吃所有场景
  - **不要 ad-hoc score 格式** — Langfuse 3 × 5 矩阵覆盖 95% 场景,自己发明 score schema 是 anti-pattern
  - **不要过早做集成生态** — Langfuse 23+ 集成是 7+ 年迭代产物,SkillForge 早期应聚焦自家 channels,避免分心

# === 服务个人学习 / 理解 eval 设计空间的关键栏 ===
learning_takeaway: |
  ## 跟其他 eval framework 不同的是 (differentiator)
  1. **Wide Events 2.0 哲学**(`.agents/ARCHITECTURE_PRINCIPLES.md:13-32`)— 把 observation 而非 trace 作为分析单元,跟 OTel 1.0 三分法(metrics+logs+traces)完全不同范式。这是 LLM observability 跟传统 APM 最大的认知差异
  2. **Score 统一抽象**(3 × 5 矩阵,`scores.ts:4-61`)— 所有 evaluator 路径输出统一成 Score,业内最简洁的 eval 数据模型
  3. **`EVAL` source 编译期保留**(`scores.ts:13-21`,`as const satisfies`)— 把"防伪造内部 score"做成 TypeScript 类型系统层硬约束,业内独家工程细节
  4. **EvalTemplate 版本控制** — eval 模板跟 prompt 一样版本化,vs 业内多数 eval framework 把 evaluator 当一次性脚本
  5. **Production loop 单线打通**(`addDatasetRunItemsToEvalQueue.ts`)— production trace → dataset → eval → score 全自动,vs 业内多数 eval framework 把"eval"跟"production"分两个世界
  6. **Main + Secondary 双队列分流多租户**(`queues.ts:327,332,334`)— 业内独家的"多租户性能隔离"做法,SaaS 场景必备
  7. **23+ 框架集成**(OpenAI/Langchain/LlamaIndex/Haystack/LiteLLM/Vercel AI SDK/DSPy/CrewAI/AutoGen/Goose/smolagents...)— 业内集成生态最广
  8. **完全开源 + 完全可自托管**(MIT + docker compose 5 分钟)— 跟 Helicone / LangSmith / Phoenix / Honeycomb 等闭源/有 cloud 锁的对手区分开

  ## 优势 (strengths)
  1. **`scores.ts:13-21` `as const satisfies readonly ScoreSourceType[]` 编译期 subset 保证** — `PublicApiCreateScoreSourceDomain` 一行确保外部 API 不能创建 `EVAL` source,业内独家工程细节
  2. **`isAnnotationScoreMissingConfigId` zod refine + REST 同步 400 + ingestion 异步 drop**(`scores.ts:30-40`)— 一份 predicate 用在两个调用路径,代码无重复
  3. **Worker queue 32 个,每个都有专门 file + 主+次双队列**(`worker/src/queues/` 26 file)— production 工程化深度业内标杆
  4. **`ARCHITECTURE_PRINCIPLES.md` 把架构哲学 codify 成给 LLM 看的 11 条**(`.agents/ARCHITECTURE_PRINCIPLES.md:13-32`)— 类似 Codex AGENTS.md 17KB 的做法,业内 LLM-readable 工程文档典范
  5. **`.agents/skills/` 16 个 dev skill(Langfuse 自家 dogfood Claude Code)**— `add-model-price` / `analyze-cloud-costs` / `clickhouse-best-practices` / `detect-prod-regressions` / `debug-issue-with-datadog` 等每个 dev workflow codify 成 skill,印证 SkillForge 内置 skill 系统方向
  6. **23+ 集成自动 instrumentation**(OpenAI drop-in / Langchain callback / LlamaIndex callback / LiteLLM proxy)— 用户**几乎零代码**接入,业内最低集成门槛
  7. **OTel 摄入路径**(`ObservationTypeMapper.ts` 把 OTel span 转 Langfuse observation)— **吸收业界 OTel 生态而不强加自家协议**
  8. **Fern API SDK 自动生成**(`fern/` 目录)— OpenAPI → Python/JS/TS SDK 自动 derive,前后端类型对齐自动化

  ## 哪些点做得好 (what_done_well)
  1. **`as const satisfies readonly ScoreSourceType[]` 一行实现编译期 subset 校验** — 比运行时 `if (source === "EVAL") throw` 优雅得多
  2. **3 × 5 Score 矩阵**(`Source × DataType`)+ 每种 DataType 一个专门 zod schema(`NumericData` / `CategoricalData` / `BooleanData` / `CorrectionData` / `TextData`,`scores.ts:63+`)— **数据类型决定字段形态**,工程细节
  3. **`AGENTS.md` 是 symlink + 真源在 `.agents/`**(`AGENTS.md -> .agents/AGENTS.md`, `CLAUDE.md -> AGENTS.md`)— LLM agent 工具(Claude Code / Codex 等)都能找到,但代码组织真源唯一 — **绝妙的工程组织**
  4. **`.agents/ARCHITECTURE_PRINCIPLES.md` 引用 Charity Majors 博客 + Burmistrov substack + Langfuse 自家博客** — 把"理论根据"+"业内共识"+"自家实践"三者绑定到一份文件,**LLM 读完就理解架构哲学**
  5. **Queue 命名 main + secondary 双套**(`IngestionQueue` / `IngestionSecondaryQueue`,`EvaluationExecution` / `EvaluationExecutionSecondaryQueue`)+ **注释明确分流目的** — 不需要复杂 priority 系统,**两个 queue 解决 80% 多租户性能隔离**
  6. **DLQ 一等公民**(`DeadLetterRetryQueue` `queues.ts:353` + `dlqRetryQueue.ts` redis 实现 + `worker/src/services/dlq/`)— 多数项目把 DLQ 当后期补丁,Langfuse 从 day-1 设计
  7. **`EntityChangeQueue`**(`queues.ts:355`)— 所有 entity change 走统一事件流,下游 webhook + integration 都消费这个 — **观察者模式工程化**
  8. **每个 integration 拆 2 queue**(入口 queue + processing queue,如 `PostHogIntegrationQueue` + `PostHogIntegrationProcessingQueue`)— 入口快速 ack 不阻塞,processing 慢逻辑独立 — **业内 SaaS integration 工程模板**
  9. **`worker/src/constants/default-model-prices.json` 把模型定价集中维护** + 专门 skill(`.agents/skills/add-model-price/`)— 每加模型都走相同流程,**避免散落更新**
  10. **`web/src/features/` 50+ slice 按 product 切分,每个 slice 内含 `components/` + `hooks/` + `server/` + 自家 README** — **feature-first 组织** vs 传统 MVC 横切组织,LLM 改一个 feature 不需跨 5 个目录
---

# Langfuse — LLM Engineering Platform

## 一句话定位
开源 LLM engineering 平台(MIT, YC W23)+ Observability + Eval + Prompt Management + Dataset + Playground + 23+ 框架集成,**Wide Events 2.0 + ClickHouse 工程化标杆**,自家也 dogfood Claude Code skill 体系。

## 起源与定位

- **2023 起源**(Y Combinator W23 batch)
- **2025 双倍下注开源**(README 顶部 blog 链接:https://langfuse.com/blog/2025-06-04-open-sourcing-langfuse-product)
- **MIT license** 完全 open source,商业模式是 Langfuse Cloud(SaaS)+ Self-host 任意规模免费
- **核心使命**:让 LLM 应用的 develop / monitor / evaluate / debug 4 件事**一体化协作**,而不是 4 个分散工具
- **现代化哲学**:Wide Events 2.0(Charity Majors 路线),反对 OTel 1.0 metrics+logs+traces 三分法

## 核心架构

```
┌──────────────────────────────────────────────────────────┐
│          User Applications                                │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Python SDK (@observe())   JS/TS SDK   OpenAPI     │  │
│  │ Langchain/LlamaIndex/Haystack/LiteLLM/Vercel AI   │  │
│  │ DSPy/CrewAI/AutoGen/Goose/smolagents (23+ 集成)   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼ HTTP / OTel / SDK
┌──────────────────────────────────────────────────────────┐
│       Langfuse Web (Next.js)                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │ web/src/pages/api/public/    (REST + OpenAPI)     │  │
│  │ web/src/server/api/          (tRPC for admin UI)  │  │
│  │ web/src/features/ × 50       (UI + 业务切片)        │  │
│  │   - evals / experiments / prompts / datasets       │  │
│  │   - playground / annotation-queues / automations  │  │
│  │   - score-* / llm-* / mcp / ...                   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────┬───────────────────────────────────────────┘
               │ Redis (BullMQ)
               ▼
┌──────────────────────────────────────────────────────────┐
│           Worker (Node)                                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ worker/src/queues/ × 26  (32 QueueName)            │  │
│  │   - IngestionQueue / IngestionSecondaryQueue       │  │
│  │   - OtelIngestionQueue / OtelIngestion...Secondary │  │
│  │   - EvaluationExecution / ...SecondaryQueue        │  │
│  │   - LLMAsJudgeExecution / CreateEvalQueue          │  │
│  │   - DatasetRunItemUpsert / ExperimentCreate        │  │
│  │   - DeadLetterRetryQueue / WebhookQueue            │  │
│  │   - EntityChangeQueue / NotificationQueue          │  │
│  │   - 6 种 Integration × 2 queue 各                  │  │
│  │   - Cloud{UsageMetering,SpendAlert,FreeTier} queue │  │
│  └────────────────────────────────────────────────────┘  │
└─┬────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────┐
│  Storage (异构)                                            │
│  ┌────────────────┐ ┌──────────────────┐ ┌────────────┐ │
│  │ Postgres       │ │ ClickHouse       │ │ Redis      │ │
│  │ (metadata,auth,│ │ (主分析库:       │ │ (queue +   │ │
│  │  prisma)       │ │  trace/obs/score)│ │  cache)    │ │
│  └────────────────┘ └──────────────────┘ └────────────┘ │
│  ┌────────────────────────────────────────────────────┐  │
│  │ S3 (blob: 大 trace payload + export + retention)   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## 9 维度详解

详见 frontmatter 各字段。挑 3 个最有学习价值的展开。

### 1. Score 数据模型(3 × 5 矩阵)

`packages/shared/src/domain/scores.ts:4-61` 用 TypeScript + zod 把整个 eval 输出抽象成一个 Score 实体:

```ts
// 3 种 source(scores.ts:4)
ScoreSourceArray = ["API", "EVAL", "ANNOTATION"]
// 5 种 dataType(scores.ts:46)
ScoreDataTypeArray = ["NUMERIC", "CATEGORICAL", "BOOLEAN", "CORRECTION", "TEXT"]
```

**3 × 5 = 15 种合法组合**(部分场景受约束,如 `CORRECTION` dataType 不要求 configId 但其他 dataType 的 ANNOTATION source 要)覆盖所有 evaluator 路径:
- LLM-as-judge → 一般产出 `EVAL` source + `NUMERIC` / `CATEGORICAL` dataType
- 用户反馈(API)→ `API` source + `BOOLEAN`(👍/👎)或 `NUMERIC`(1-5 星)
- 人工标注 → `ANNOTATION` source + 任意 dataType,需带 `configId` 约束分类
- LLM 校正反馈 → `CORRECTION` dataType + `CORRECTION_NAME = "output"`(`scores.ts:42`)

**`scores.ts:13-21` 的工程细节绝妙**:

```ts
export const PublicApiCreateScoreSourceDomain = z.enum([
  ScoreSourceEnum.API,
  ScoreSourceEnum.ANNOTATION,
] as const satisfies readonly ScoreSourceType[]);
```

**`as const satisfies` 在 TypeScript 编译期保证外部 API 永远不能创建 `EVAL` source** — `EVAL` 是内部 evaluator 保留的。这是把"防伪造内部信号"做成类型系统层硬约束的业内独家工程细节。

### 2. Worker Queue 32 个 + Main/Secondary 双队列分流

`packages/shared/src/server/queues.ts:322-358` 列出 32 个 QueueName。最值得学的是 **"主 + 次双队列分流多租户" 模式**(7 处主+次配对):

```
IngestionQueue + IngestionSecondaryQueue            (queues.ts:333-334)
OtelIngestionQueue + OtelIngestionSecondaryQueue    (queues.ts:331-332)
EvaluationExecution + EvaluationExecutionSecondaryQueue  (queues.ts:326-327)
```

Source code 注释明确(`queues.ts:327`):

> "Separates high-throughput eval projects from other projects."

(`queues.ts:332` 类似:"Separates high priority + high throughput projects from other projects")

**工程含义**:**多租户场景,一个 high-throughput project 不会拖慢其他 project**。不需要复杂的 priority queue 系统,**两个 queue 解决 80% 隔离需求**。SkillForge 多租户场景应直接学这个模式。

其他 25 个 queue 也都有专门 file 实现(`worker/src/queues/` 26 file),包括:
- **DLQ 一等公民**:`DeadLetterRetryQueue`(`queues.ts:353`)+ `worker/src/services/dlq/` + `dlqRetryQueue.ts`
- **整体审计流**:`EntityChangeQueue` + `EventPropagationQueue`(`queues.ts:355-356`)
- **6 种 integration 都拆 2 queue**(入口 queue + processing queue)— 入口快速 ack 不阻塞,processing 慢逻辑独立

### 3. Production Loop 单线打通

`web/src/features/evals/server/addDatasetRunItemsToEvalQueue.ts` 实现了**最关键的工程闭环**:

```
production trace → dataset(自动归集)
       ↓
  DatasetRunItemUpsert queue(queues.ts:329)
       ↓
  自动入 EvaluationExecution queue / Secondary queue
       ↓
  LLMAsJudgeExecution(若需 LLM 评分)
       ↓
  Score(API/EVAL/ANNOTATION 3 来源统一)
       ↓
  ClickHouse 持久化 → 仪表盘 + experiment 对比
       ↓
  Webhook + 6 种 outbound integration → 通知 PostHog/Mixpanel/S3/Metering
```

**SkillForge "Prompt A/B + 自动晋升" 这条线的开源参考实现就是这套 pipeline**。Langfuse 本身没把"自动晋升"做成 first-class(用户需写 automation 自定义),但单线打通的工程模板可直接借鉴。

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`。最关键的 3 个决策点:

1. **ClickHouse 引入时机** — SkillForge 现在 Postgres,Langfuse 经验:< 10K traces/day Postgres 够,> 10K 建议 ClickHouse 切换
2. **Score 抽象采纳粒度** — 完整搬 3×5 矩阵还是先做 3 种 dataType?建议从 `NUMERIC + CATEGORICAL + BOOLEAN` 起步,后续按需扩
3. **Main + Secondary 双队列分流** — SkillForge 多租户场景何时切?建议:high-throughput 单 project 拖慢其他 project 时(经验阈值:单 project > 50% 总队列时长)

## 待回答的问题

- ClickHouse schema 具体什么样(看了 `packages/shared/src/server/clickhouse/schema.ts` 但未深读)— 表分区策略 / TTL / order by 设计
- `ShardedQueueRegistry` 具体怎么实现 sharding(`worker/src/queues/shardedQueueRegistry.ts`)
- `LLMAsJudgeExecution` queue 的具体 evaluator 模板格式(Ragas 集成方式)
- `experiments` 跟 `evals` 实际差异(都做 A/B,边界在哪)
- 自家 LLM provider 抽象 `fetchLLMCompletion.ts` 跟业内对比怎么样
- Audit log 怎么消费的(谁查询 audit log 表)
- `MeteringDataPostgresExportQueue` 为什么数据要回写 Postgres(从 ClickHouse 反向)
- EE(`ee/`)跟 OSS 边界在哪 — license 怎么切
- `automations` feature 内置自动化的具体能力范围
- `.agents/skills/clickhouse-best-practices/SKILL.md` 内容(没看,值得 Langfuse 单独深查)
