---
type: eval-framework
name: opik
status: active
language: java + python + typescript

# === 版本时效 ===
last_ingested_version: "1.7.0 @ commit 72b2f1ca89 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/opik]]"
  - "/Users/youren/myspace/opik/README.md"
  - "/Users/youren/myspace/opik/AGENTS.md"
  - "/Users/youren/myspace/opik/sdks/opik_optimizer/"
  - "/Users/youren/myspace/opik/sdks/python/src/opik/evaluation/"
  - "/Users/youren/myspace/opik/apps/opik-backend/src/main/java/com/comet/opik/domain/"
  - "/Users/youren/myspace/opik/.agents/"

# === 9 维度 ===

target: |
  LLM 应用全栈 + **特别关注 RAG**(内置 hallucination / moderation / Answer Relevance / Context Precision metrics)+ Agentic system(README 显式提及 "complex agentic systems")+ Production scale("40M+ traces/day" 标榜)。

  4 大支柱(README):
  - **Comprehensive Observability**(LLM call + conversation + agent activity 深度 tracing)
  - **Advanced Evaluation**(prompt eval + LLM-as-judge + experiment management)
  - **Production-Ready**(dashboard + online evaluation rules)
  - **Opik Agent Optimizer + Opik Guardrails**(独立 SDK + 独立 service)

  跟 Langfuse 关键区别:**Opik 主打 Optimization 第三支柱**(4 家里独家)。
  [verified-from-source]

dataset: |
  - **自家 Dataset 实体**(`apps/opik-backend/.../domain/` `Dataset*` Java DAO)+ CSV import/export
  - **8 个内置 benchmark dataset**(`sdks/opik_optimizer/src/opik_optimizer/datasets/`)**4 家里独家**:
    - `gsm8k.py` — 小学数学
    - `truthful_qa.py` — 真实性 QA
    - `halu_eval.py` — Hallucination eval
    - `ai2_arc.py` — AI2 ARC reasoning
    - `arc_agi2.py` — ARC AGI v2(挑战性 reasoning)
    - `pupa.py` — PUPA
    - `medhallu.py` — Medical hallucination
    - `tiny_test.py` — tiny test fixture
  - Public API 通过 Python SDK `sdks/python/src/opik/datasets/` 拉服务端或本地 dataset
  - 来源:手动 + production trace 导入 + CSV + **业内 benchmark zero-config**(Opik 独家)
  - 更新频率:append-only(trace)+ versioned(dataset)
  [verified-from-source]

scorer: |
  **Assertion + Feedback 双轨设计**(跟 Langfuse 数据统一抽象路线完全不同):

  ### 1. Assertion(Pytest 风格 first-class)
  - Java backend:`AssertionResultDAO.java` + `AssertionResultService.java` + `AssertionResultMapper.java` 三层
  - Java API:`AssertionResult.java` + `AssertionResultBatch.java` + `AssertionResultBatchItem.java` + `AssertionScoreAverage.java` + `AssertionStatus.java`
  - Python SDK:`sdks/python/src/opik/evaluation/test_case.py` — `TestCase dataclass { trace_id, dataset_item_id, task_output, dataset_item_content, mapped_scoring_inputs, dataset_item }`
  - **Pytest 风格 assertion + reason,而非 Langfuse 风格数值 Score**

  ### 2. Feedback Definition Model(OOP 多态,按 dataType 拆 Java 子类)
  - `BooleanFeedbackDefinitionDefinitionModel.java`
  - `CategoricalFeedbackDefinitionDefinitionModel.java`
  - (推断还有 Numeric / Text 等子类)
  - **每个 dataType 一个 Java 子类继承同接口** — 跟 Langfuse 单 `ScoreDataType` 枚举不同

  ### 3. 4 类 evaluation metric(`sdks/python/src/opik/evaluation/metrics/`)
  - **conversation/** + **conversation_types.py** — 对话级 metric
  - **heuristics/** — 规则级 metric(可程序判定)
  - **llm_judges/** — LLM-as-judge metric(hallucination / moderation / answer relevance / context precision)
  - **ragas_metric.py** — Ragas 集成(跟 Langfuse 同名)

  ### 4. Aggregated + Multi-metric
  - `metrics/aggregated_metric.py` — 多 metric 聚合
  - `opik_optimizer/metrics/multi_metric_objective.py` — `MultiMetricObjective`(优化器多目标)
  [verified-from-source]

reproducibility: |
  - **Prompt + experiment + dataset 三轨版本控制**
  - **Optimizer `OptimizationResult` 标准化对象**(`sdks/opik_optimizer/src/opik_optimizer/base_optimizer.py` 多处返回)— reruns 可对比
  - **Model 定价集中维护**:`apps/opik-backend/src/main/resources/model_prices_and_context_window.json` + `apps/opik-frontend/src/data/model_prices_and_context_window.json` — **AGENTS.md 明确说 generated artifact 不能手动改**
  - **Docker Compose** 7 种 service profile(`./opik.sh --infra/--backend/--guardrails/...`)+ **Helm 自托管**完整可复现
  - **Pytest integration**(`test_case.py` + `test_result.py`)— CI 跑 eval 出 TestResult,assertion 直接挂 PR
  [verified-from-source]

coverage: |
  **覆盖 5 大类**(跟 Langfuse 4 大类相比多 Optimization):
  - Observability(SDK 装饰器 `@track` + auto-instrumentation,30+ 框架包括最新 Google ADK / AutoGen / Flowise AI)
  - Evaluation(Datasets + Experiments + Pytest 集成)
  - **Optimization**(`sdks/opik_optimizer/` 7 种算法)
  - **Guardrails**(独立 `apps/opik-guardrails-backend/`)
  - Online Evaluation Rules(production trace 流上跑 LLM-as-judge 规则)

  **已知 gap**:
  - **Agent Optimizer chaining 还在早期**(README 标 "Optimizer Chaining"但 base_optimizer.py 尚未深度集成 chain DSL)
  - **没有 Langfuse 风格的 23+ 框架 native instrumentation**(Opik 30+ 集成偏 SDK callback 而非 drop-in replacement)
  - **Ruby SDK 是 OTel-only**(不是原生 SDK,跟 Python/TS 是两个等级)

  **已知 gaming pattern**:
  - 7 种 optimizer 算法跑 prompt evolution,**可能 overfit 单个 dataset**(Opik 给出 8 个 benchmark 帮 cross-validate)
  - LLM-as-judge 自家模型评自家模型 bias(行业通病)
  [verified-from-source]

production_loop: |
  **Production loop + Optimizer 闭环**(4 家里独家的"自动改 prompt"环节):

  ```
  Production trace → @track decorator + auto-instrumentation
       ↓
  ClickHouse 持久化(40M+ traces/day 规模)
       ↓
  Online Evaluation Rules(LLM-as-judge 在流上跑)
       ↓
  Alert / Webhook(`AlertDAO` + `AlertTriggerConfig*`)
       ↓
  Dataset 自动归集(失败 trace + low-score trace)
       ↓
  Experiment(Pytest CI 跑 + assertion)
       ↓
  Opik Agent Optimizer(7 种算法自动改 prompt)←—— 4 家独家这一环
       ↓
  OptimizationResult + Optimizer Chaining(一个 optimizer 结果给下一个)
       ↓
  Promote(人审)→ 回滚或新 prompt 上线
  ```

  关键:**Opik 是唯一把"自动改 prompt"做成 first-class** 的 eval framework。Langfuse experiment 只做对比不改;Opik 给完整闭环:trace → eval → optimizer → promote。
  [verified-from-source]

automation: |
  - **Pytest 集成 first-class**(`sdks/python/src/opik/evaluation/test_case.py` + `test_result.py`)— CI 跑 eval 直接 assertion → fail PR
  - **`opik.sh` 7 种 service profile** — CI 自动起服务(`--infra` / `--backend` / `--guardrails`)
  - **`make precommit-sdks`** — 跨 SDK pre-commit checks
  - **Online Evaluation Rules** — production trace 流上自动跑 LLM-as-judge
  - **Alert 系统**(`AlertDAO` + `AlertEventType` + `AlertTriggerConfig*` + `AlertType`)— 4 家里独家完整 alerting
  - **MCP 集成**(`.agents/mcp.json` GitHub + Slack via Docker)
  - **Optimizer Chaining**:一个 optimizer 输出作为下一个输入,**多算法 pipeline 自动跑**
  - **PyTest fixtures + Vitest + Playwright + Maven/JUnit** 4 套测试框架统一在 `./opik.sh` 下
  [verified-from-source]

cost: |
  - **`apps/opik-backend/.../domain/cost/`** 独立 Java cost domain 子模块
  - **`model_prices_and_context_window.json`** 集中维护(`apps/opik-backend/src/main/resources/` + `apps/opik-frontend/src/data/`)— 双份同步,**generated artifact** 不能手动改
  - **40M+ traces/day** 经济性是 selling point — ClickHouse 列存 + Java backend(JVM 性能)
  - **Optimizer 跑成本**:7 种算法 × 8 内置 dataset cross-validation 是非平凡成本,Opik 通过 `Counter Tracking`(`opik_optimizer` README:"Built-in LLM and tool call counters for monitoring usage")让用户可控
  - **采样**:SDK 层支持
  [verified-from-source]

# === SkillForge 决策的关键栏 ===
skillforge_takeaway: |
  ## 可借鉴 (12 条)
  1. **Opik Agent Optimizer 7 算法**(`sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26`)— **SkillForge Self-Evolution 演进方向**,从 `/evolve` meta self-evolution 进化到"自动改 prompt"runtime optimization,详见 [[../concept/prompt-optimization]]
  2. **OptimizationResult 标准化对象** — 所有 optimizer 统一返回类型,Optimizer Chaining 可串联,SkillForge 自进化结果应有等价标准化
  3. **8 个内置 benchmark dataset**(gsm8k / truthful_qa / halu_eval / ai2_arc / arc_agi2 / pupa / medhallu / tiny_test)— SkillForge 应内置一组业内 benchmark,用户 zero-config 跑
  4. **Assertion + Feedback Definition Model 双轨**(Java OOP 多态):**SkillForge Java 生态匹配度最高的评分模型**,适合 Spring Boot 实现。比 Langfuse 数据统一抽象更"Java-natural"
  5. **Pytest 集成 first-class**(`test_case.py` + `test_result.py`)— SkillForge 应有 JUnit 等价物,CI 直接 fail PR
  6. **`.agents/` 完整目录**(agents/commands/rules/skills/mcp/docs 全套)— SkillForge 仓库应 dogfood 这套结构,**.codex + .cursor symlink to .agents** 印证多 LLM 工具普适
  7. **6 个 builtin agent**(planner/build-fixer/code-reviewer/meta-auditor/test-runner/architect)— SkillForge 应内置类似 agent,**特别是 meta-auditor proactive 防 bloat 模式**值得学
  8. **Alert 系统**(`AlertDAO` + `AlertEventType` + `AlertTriggerConfig*` + `AlertType`)— SkillForge 应有完整 alert,不只 webhook
  9. **Guardrails 独立 service**(`apps/opik-guardrails-backend/`)— SkillForge guardrails 当前 Java 内置,可独立服务化(对照 Codex `secrets/` + `process-hardening/` + `execpolicy/` 拆 crate)
  10. **Python sandbox executor 独立 service**(`apps/opik-sandbox-executor-python/`)— SkillForge 跑用户自定义 hook/skill/evaluator 应有等价 sandbox service
  11. **Online Evaluation Rules** — production trace 流上跑 LLM-as-judge 规则,SkillForge "实时质量监控"路线
  12. **`opik.sh` 7 种 service profile**(`--infra` / `--backend` / `--guardrails` 等)— SkillForge 部署脚本可学这种 modular 启动模式

  ## 业内验证 (5 条)
  - **Java + Spring Boot + ClickHouse + MySQL** 是 LLM eval 平台经过验证的路线(Opik 40M+ traces/day)— SkillForge 当前 Java + Postgres 路线方向对,后续 ClickHouse 切换有 Opik 模板
  - **Pytest / JUnit 集成 CI** 是 eval framework 主流(Opik 一等公民,Langfuse SDK 支持)
  - **LiteLLM 是 provider 抽象层事实标准**(Opik Optimizer 强依赖,Langfuse 也支持)
  - **Ragas 集成** Opik + Langfuse 都做,RAG eval 行业标杆
  - **`.agents/` dogfood Claude Code 模式** Opik 跟 Langfuse 都做,业内事实标准

  ## SkillForge 领先 (2 条)
  - **`/evolve` meta self-evolution + 人审 + 落 dev artifact** SkillForge 路线 → Opik Agent Optimizer 是"runtime optimization 无人审",**两条路线互补**,SkillForge 应保留 meta 路线 + 引入 runtime optimizer
  - **统一 ChannelAdapter SPI** — Opik 主要靠 30+ 集成 callback,SkillForge SPI 抽象更轻量

  ## 待决策 (5 条)
  1. **Score 数据模型选哪条路** — Langfuse 数据统一抽象(3×5 矩阵)vs Opik OOP 多态(Java 子类继承)— **SkillForge Java 生态建议起步 Langfuse 数据抽象,后续按需 evolve Java 子类**
  2. **是否引入 Opik Agent Optimizer 等价物** — 7 算法 + OptimizationResult + Optimizer Chaining 工程量大,**先做技术 POC**(用 Python 跑 EvolutionaryOptimizer + GepaOptimizer 两种,看效果再投入)
  3. **6 SDK / app 拆分粒度** — Opik 拆 6 个 app(backend/frontend/python-backend/sandbox-executor/guardrails/docs)对比 Langfuse 3 个(web/worker/shared)— SkillForge 微服务化方向参考 Opik
  4. **Pytest CI 集成时机** — SkillForge eval pipeline 是否一开始就提供 JUnit 集成?(建议:是,降低用户接入门槛)
  5. **Comet bridge 对照** — Opik `ConnectBridgeService.java` 跟 Comet 主平台桥接,SkillForge 跟其他平台(Langfuse / Opik / Helicone)互通可学这种桥模式

  ## 风险 (4 条)
  - **7 个 optimizer 算法工程深度大**,SkillForge 不要早期完整复刻 — 选 1-2 个最有效的(GepaOptimizer + FewShotBayesianOptimizer)起步
  - **Opik OOP 多态评分模型** Java 子类越加越多容易类爆炸 — SkillForge 起步用 Langfuse 数据抽象更安全
  - **40M+ traces/day 规模** 不是 SkillForge 早期目标 — Opik 工程化为大规模设计,SkillForge < 10K traces/day 阶段不用照搬
  - **`apps/opik-python-backend` 单独存在意义**:Java backend 调 Python 跑 LLM/SDK 相关逻辑,SkillForge 是否需要类似 Python helper backend(给 Python 用户写的 evaluator/metric 跑沙箱)— **建议起步不要,优先 Java 内嵌**

# === 服务个人学习 / 理解 eval 设计空间的关键栏 ===
learning_takeaway: |
  ## 跟其他 eval framework 不同的是 (differentiator)
  1. **Optimization 第三支柱**(README slogan:"Open-source AI Observability, Evaluation, and **Optimization**")— 4 家里独家,**eval framework + 自动改 prompt 闭环**,跟 Langfuse"只对比不改"完全不同范式
  2. **`sdks/opik_optimizer/` 7 算法独立 SDK** — `EvolutionaryOptimizer` + `FewShotBayesianOptimizer` + `MetaPromptOptimizer` + `GepaOptimizer` + `ParameterOptimizer` + `HierarchicalReflectiveOptimizer` + `HRPO`,每种算法不同 prompt evolution 哲学
  3. **8 个内置 benchmark dataset**(gsm8k / truthful_qa / halu_eval / ai2_arc / arc_agi2 / pupa / medhallu / tiny_test)— 4 家里独家,zero-config 跑业内 benchmark
  4. **Assertion + Feedback Definition Model 双轨**(Java OOP 多态,vs Langfuse 数据统一抽象)— 两种评分模型设计哲学根本不同
  5. **Pytest 集成 first-class**(`test_case.py` + `test_result.py`)— CI fail PR,Langfuse 主要靠 SDK 提交而非 PyTest fixture
  6. **6 个独立 app**(opik-backend Java + opik-frontend React + opik-python-backend + opik-sandbox-executor-python + opik-guardrails-backend + opik-documentation)— 服务化拆分粒度业内最细
  7. **`.agents/` 完整 Claude Code 生态**(agents/commands/rules/skills/mcp/docs 全套 + .codex/.cursor symlink)— 4 家里 dogfood 程度最深
  8. **6 个 builtin agent** 都是真 Claude Code subagent 标准格式(example-based description + model + color + tools 白名单)— planner(sonnet/blue/read-only)+ build-fixer(haiku/yellow/minimal-edit)是工程化标杆
  9. **Online Evaluation Rules**(production 流上跑 LLM-as-judge)— Langfuse 没有显式等价物
  10. **Comet ML 母公司桥**(`ConnectBridgeService.java`)— 跟主平台互通,商业产品策略
  11. **Apache 2.0**(vs Langfuse MIT)+ 商业 retention 路线(Comet Cloud 收费)
  12. **Ruby SDK via OTel**(4 家里只有 Opik 提供)

  ## 优势 (strengths)
  1. **OptimizationResult 标准化对象**(`base_optimizer.py` 多处返回类型)— 所有 optimizer 统一接口,Optimizer Chaining 可串联,RAII-style 工程
  2. **`opik.sh` 7 种 service profile** — CI / local dev 灵活组合(`--infra` 只起 DB / `--backend --guardrails` 全功能)
  3. **MCP 集成完整**:`.agents/mcp.json` 配 GitHub + Slack via Docker,`ChatPrompt.tools` 优化器支持 MCP tool call
  4. **`as const` Type Safety**(README:"Full type hints and validation for robust development")— 全 type hint
  5. **Backward Compatibility**(`opik_optimizer` README:"All original parameters preserved through kwargs extraction" + "Deprecation Warnings: Clear warnings for deprecated parameters with migration guidance")— 工程文化
  6. **Multimodal prompts**(text/image/audio/video,OpenAI content parts)— 跨模态优化原生支持
  7. **`Counter Tracking`** — Optimizer 跑 LLM + tool call 自动计数,**用户能准确知道优化成本**
  8. **`AGENTS.md` 8KB router-style + 模块级 AGENTS.md** — 跟 Codex 17KB AGENTS.md 一致工程文化(LLM-readable contributor guide)
  9. **`.codex -> .agents` + `.cursor -> .agents` symlink** — 完美的多 LLM 工具普适性
  10. **`.agents/skills/` 10 个 dev workflow skill**(包括 analytics-instrumentation + clickhouse-best-practices)— Langfuse 同模式,Opik dev workflow 更精细
  11. **6 个 builtin agent 用不同 model**(planner=sonnet / build-fixer=haiku)— **model 选择是 agent 设计的一部分**,工程细节
  12. **`tools 白名单`**(planner.md:48 `tools: ["Read", "Grep", "Glob"]`)— builtin agent 工具收紧,**read-only agent 强制 read-only**

  ## 哪些点做得好 (what_done_well)
  1. **`build-fixer.md` model=haiku + 单一职责 "fix builds with minimal changes, no refactor"** — 跟 Codex `awaiter.toml` 单一职责 builtin 一致哲学(`builtins/awaiter.toml`:"You must NOT: Modify the task. Interpret or optimize the task.")
  2. **`meta-auditor.md`(config-auditor)proactive 触发,在创建 rule/skill/agent/mcp 前先审** — 防 .agents/ 目录 bloat,工程文化漂亮
  3. **`planner.md` tools 限 `["Read", "Grep", "Glob"]` 完全 read-only** — 规划 agent 不写代码,职责清晰
  4. **`AGENTS.md:## Generated Files` 显式标记** `model_prices_and_context_window.json` "不能手动改,来自 repository updater 自动化" — 防止 PR 改 generated artifact 引入 drift
  5. **`AGENTS.md:## Ignored Surfaces` 显式标记** `.github/instructions/` 跟 `.github/copilot-instructions.md` 是 Copilot 专属,其他 agent 忽略 — **多 agent 工具兼容性的 explicit guard rail**
  6. **`opik.sh --info` 调试模式** — 运维脚本自带 debug
  7. **7 算法 OptimizationResult 统一返回**(`base_optimizer.py`)+ Optimizer Chaining — 跟 Codex `ConfigLayerStack` 层叠思路一致(用结构化结果让组件可组合)
  8. **`Multimodal Prompts` OpenAI content parts** 跨 optimizer 共享 — 一份多模态抽象 7 算法都用
  9. **`Counter Tracking` 自动计数 LLM + tool call** — 跟 Codex 5 GOAL OTEL metric / Langfuse `default-model-prices.json` 同精神(运行时可观测)
  10. **`Backward Compatibility` 通过 kwargs extraction + Deprecation Warning** — 工程文化(API 演进不 break user code)
---

# Opik(Comet Opik)— LLM Engineering Platform

## 一句话定位

Comet ML 出品的开源 LLM engineering 平台(Apache 2.0),**Observability + Evaluation + Optimization 三支柱**(比 Langfuse 多 Optimization 第三支柱)。Java + Spring Boot + 6 个独立 app + 3 SDK(Python/TS/Ruby via OTel)+ **`sdks/opik_optimizer/` 7 算法独立 SDK**(4 家独家)+ `.agents/` 完整 Claude Code 生态(4 家最深 dogfood)+ 40M+ traces/day 生产规模。

## 起源与定位

- **Comet ML 母公司**(2017+ 起源,ML 实验跟踪老牌,W&B 同类商业公司)
- **2024+ Opik 开源**(Apache 2.0)— 跟 Comet 主平台(Cloud 收费)互通(`ConnectBridgeService.java`)
- **核心使命**:让 LLM 应用的 **observability + eval + optimization** 形成完整闭环,**不只是看 + 测,还能自动改**
- **现代化哲学**:用 Optimizer 算法自动 prompt evolution,**`eval → optimizer → promote` 闭环替代手工 prompt engineering**

## 核心架构

```
┌────────────────────────────────────────────────────────────────┐
│         User Applications                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Python SDK (@track)   TypeScript SDK    Ruby (via OTel)  │  │
│  │ 30+ 集成:Google ADK / AutoGen / Flowise / Langchain ...  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬───────────────────────────────────────┘
                         │ HTTP / OTel / SDK
                         ▼
┌────────────────────────────────────────────────────────────────┐
│    apps/opik-frontend (React/TS + Vite)                        │
│       └─ Playwright E2E tests                                  │
└────────────────────────┬───────────────────────────────────────┘
                         │ REST/API
                         ▼
┌────────────────────────────────────────────────────────────────┐
│    apps/opik-backend (Java 17 + Spring Boot + Maven)           │
│                                                                 │
│    com/comet/opik/                                              │
│      ├ api/        REST DTO/Controller                          │
│      ├ domain/                                                  │
│      │    ├ Agent{Blueprint,Config,ConfigDAO,ConfigService}     │
│      │    ├ Assertion{ResultDAO,ResultService,ResultMapper}     │
│      │    ├ FeedbackDefinitionModel(OOP 多态:Boolean/Categorical)│
│      │    ├ Alert{DAO,Service,TriggerConfigDAO,TriggerDAO}      │
│      │    ├ Annotation{QueueDAO,QueueService,QueueReviewer}     │
│      │    ├ Comment{DAO,Service,EntityRef}                      │
│      │    ├ ConnectBridgeService (Comet ↔ Opik 桥)              │
│      │    ├ cost/                                               │
│      │    ├ CsvDatasetExport*  Dataset*Export*Job*              │
│      │    └ Dashboard{DAO,Service}                              │
│      ├ infrastructure/                                          │
│      ├ utils/                                                   │
│      └ OpikApplication.java (Spring Boot 入口)                  │
└─┬──────────────────────────────────────────────────────────────┘
  │
  ├─→ apps/opik-python-backend/    (Python helper for SDK)
  ├─→ apps/opik-sandbox-executor-python/  (用户自定义 evaluator 沙箱)
  └─→ apps/opik-guardrails-backend/(Python Guardrails 独立 service)
  │
  ▼
┌────────────────────────────────────────────────────────────────┐
│  Storage(异构 3 库)                                              │
│  ┌──────────┐ ┌──────────────────┐ ┌────────────┐               │
│  │ MySQL    │ │ ClickHouse       │ │ Redis      │               │
│  │ (metadata│ │ (主分析:40M+/day)│ │ (queue +   │               │
│  │  + auth) │ │  trace/obs/score │ │  cache)    │               │
│  └──────────┘ └──────────────────┘ └────────────┘               │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ sdks/opik_optimizer/  ✨ 独立 SDK (4 家独家)                     │
│                                                                 │
│  src/opik_optimizer/                                            │
│    __init__.py:18-26  ✨ 7 算法                                  │
│      ├ EvolutionaryOptimizer          (遗传算法)                 │
│      ├ FewShotBayesianOptimizer       (few-shot + Bayesian)      │
│      ├ MetaPromptOptimizer            (meta-prompting)           │
│      ├ GepaOptimizer                  (GEPA Genetic-Pareto)      │
│      ├ ParameterOptimizer             (temperature/top_p Bayes)  │
│      ├ HierarchicalReflectiveOptimizer (README 未提)             │
│      └ HRPO                            (README 未提)             │
│    base_optimizer.py  BaseOptimizer(ABC) + optimize_prompt()    │
│    core/  results/evaluation/runtime/agent/llm_calls/state      │
│    metrics/  multi_metric_objective / answer_correctness ...    │
│    datasets/  ✨ 8 内置 benchmark                                │
│      gsm8k / truthful_qa / halu_eval / ai2_arc / arc_agi2 /     │
│      pupa / medhallu / tiny_test                                │
└────────────────────────────────────────────────────────────────┘
```

## 9 维度详解

详见 frontmatter 各字段。挑 3 个最有价值的展开。

### 1. Agent Optimizer 7 算法(4 家独家)

`sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26` 完整列出 7 个 optimizer 算法,**每个不同的 prompt evolution 哲学**:

| 算法 | 类型 | 适用场景 |
|---|---|---|
| **EvolutionaryOptimizer** | 遗传算法 | 大搜索空间,无梯度可用 |
| **FewShotBayesianOptimizer** | Bayesian + few-shot | 数据少,需 sample-efficient |
| **MetaPromptOptimizer** | Meta-prompting | LLM 自己改 prompt(LLM-as-meta) |
| **GepaOptimizer** | GEPA(Genetic-Pareto) | 多目标 Pareto front 优化 |
| **ParameterOptimizer** | Bayesian(参数空间) | 调 temperature / top_p / max_tokens |
| **HierarchicalReflectiveOptimizer** | 反思层级 | 复杂任务 hierarchical reflection |
| **HRPO** | (待查具体算法) | (HRP Optimization?) |

**关键工程细节**(见 [[../concept/prompt-optimization]]):
- 统一 `BaseOptimizer(ABC)` + `optimize_prompt()` 接口
- 统一 `OptimizationResult` 返回类型 → **Optimizer Chaining**(一个 optimizer 输出给下一个用)
- **MCP Support**:`ChatPrompt.tools` 让 optimizer 优化带 MCP tool 调用的 agent
- **Counter Tracking**:LLM + tool call 自动计数(用户清楚知道优化成本)
- **Multimodal Prompts**:OpenAI content parts(text/image/audio/video)跨 optimizer 共享
- 8 个内置 benchmark dataset 帮 cross-validate(防 overfit 单 dataset)

这是 SkillForge **Self-Evolution 演进方向的开源参考**(详见 [[../concept/self-evolution]] 第 4 类)。

### 2. Assertion + Feedback 双轨评分(Java OOP 多态)

跟 Langfuse `Score = ScoreSource × ScoreDataType` 数据抽象不同,Opik 用 **Java OOP 多态**:

**Assertion**(Pytest 风格,`apps/opik-backend/.../domain/AssertionResultDAO.java`):
- pass/fail + reason
- 直接对应 Pytest `assert` 语句
- `TestCase` dataclass(`sdks/python/src/opik/evaluation/test_case.py`)包含 trace_id + dataset_item_id + task_output

**Feedback Definition Model**(按 dataType 拆 Java 子类):
- `BooleanFeedbackDefinitionDefinitionModel.java`
- `CategoricalFeedbackDefinitionDefinitionModel.java`
- (推断还有 Numeric / Text 等)

**两条路线 trade-off**:

| | Langfuse 数据抽象(`scores.ts:4-61`) | Opik OOP 多态 |
|---|---|---|
| 优势 | 简洁,1 个 Score table,3×5 矩阵覆盖 | Java-natural,IDE refactor 友好 |
| 劣势 | 类型系统弱(TS 风格) | 类爆炸风险(每加 dataType 一个 Java 子类) |
| 适合 | TypeScript / Node.js 生态 | Java / Spring / OOP 生态 |
| SkillForge 建议 | 起步用 Langfuse 抽象,后续 evolve 子类 | |

详见 [[../concept/agent-eval]] 更新章节。

### 3. `.agents/` 完整 Claude Code 生态(4 家最深 dogfood)

`.agents/{agents,commands,rules,skills,docs}/` + `mcp.json` + `.codex -> .agents` + `.cursor -> .agents` symlink。

**6 个 builtin agent**(每个都是真 Claude Code subagent 标准格式):

```yaml
# .agents/agents/planner.md frontmatter (truncated)
---
name: planner
description: |
  Use this agent when the user needs a detailed implementation plan...
  <example>
  Context: User starting a complex feature
  ...
  </example>
model: sonnet
color: blue
tools: ["Read", "Grep", "Glob"]  # ← read-only 工具白名单
---
```

工程细节:
- **planner = sonnet + blue + tools=["Read", "Grep", "Glob"]** — 规划 agent 用强模型 + read-only
- **build-fixer = haiku + yellow + tools=["Bash", "Read", "Edit"]** — 修编译错误用快模型 + 最小工具,prompt 明确"不重构、不加功能"
- **meta-auditor(config-auditor)= proactive** — 创建 rule/skill/agent/mcp 前先审,**防 .agents/ 目录 bloat**

这是业内最深的 LLM agent 工具普适性 dogfood:
- Claude Code 看 `.agents/` 直接用
- Codex 通过 `.codex -> .agents` symlink 看
- Cursor 通过 `.cursor -> .agents` symlink 看 `.mdc` rule
- GitHub Copilot 独立用 `.github/instructions/` + `.github/copilot-instructions.md`(AGENTS.md 明确说"其他 agent 忽略这俩")

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway`。最关键的 3 个决策点:

1. **Score 数据模型选哪条路** — Langfuse 数据抽象 vs Opik OOP 多态,**SkillForge Java 生态建议起步用 Langfuse 抽象,后续 evolve 子类**
2. **是否引入 Agent Optimizer** — 7 算法工程量大,**先做技术 POC**(选 GepaOptimizer + FewShotBayesianOptimizer 2 种实测)
3. **6 SDK / app 拆分** — Opik 服务化粒度比 Langfuse 细,SkillForge 微服务化方向参考 Opik

## 待回答的问题

- `HRPO` 完整名(Hierarchical Reflective Prompt Optimization?)+ 具体算法
- `HierarchicalReflectiveOptimizer` 跟 `MetaPromptOptimizer` 区别
- `apps/opik-python-backend/` 跟 `apps/opik-sandbox-executor-python/` 边界(都是 Python service,职责拆分?)
- Java backend 怎么调 Python service(REST?gRPC?Message queue?)
- `FeedbackDefinitionModel` 完整子类列表(推断有 Numeric / Text,未 verify)
- Opik 跟 Comet 主平台桥具体协议(`ConnectBridgeService.java`)
- `apps/opik-guardrails-backend/` 跟 Java backend 怎么集成(Guardrails 是 sync block 还是 async warn)
- `tests_load/` load test 怎么 reproduce 40M+ traces/day 标榜
- 7 个 optimizer 跑同一个任务效果对比(benchmark 数据未查)
- ClickHouse schema(Opik 跟 Langfuse 都用 ClickHouse,schema 对比未做)
