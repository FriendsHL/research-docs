---
type: source
name: opik
origin: "/Users/youren/myspace/opik/ (Comet ML Opik, Apache 2.0)"
ingested_at: 2026-05-16
ingested_commit: 72b2f1ca89
ingested_commit_date: 2026-05-15
ingested_version_label: "Python SDK PyPI semver (pyproject.toml 标 python_version=3.10; 仓库无统一 semver,以 CHANGELOG.md 1.7.0 + commit hash 为准)"
contributed_to:
  - eval/opik.md
  - concept/agent-eval.md
  - concept/self-evolution.md
  - concept/observability.md
  - concept/prompt-optimization.md
---

# Source — opik (Comet Opik)

> Comet ML 出品,Apache 2.0,**Observability + Evaluation + Optimization 三支柱**(比 Langfuse 多 Optimization)。**40M+ traces/day** 生产规模标榜。Java + Spring Boot + MySQL + ClickHouse + Redis + 6 app + 3 SDK 异构架构。

## 仓库结构概览

```
opik/
├── apps/                                # 6 个独立 service
│   ├── opik-backend/                    # Java 17 + Spring Boot + Maven (主 backend)
│   │   └── src/main/java/com/comet/opik/{api,domain,infrastructure,utils}/
│   ├── opik-frontend/                   # React/TS + Vite + Vitest + Playwright
│   ├── opik-python-backend/             # Python 后端(SDK/optimizer 相关辅助 service)
│   ├── opik-sandbox-executor-python/    # Python 沙箱执行器(跑用户自定义 evaluator/metric)
│   ├── opik-guardrails-backend/         # Python Guardrails 独立 service
│   └── opik-documentation/              # 文档站
├── sdks/
│   ├── python/src/opik/                 # Python SDK(主)
│   ├── typescript/                      # TypeScript SDK
│   ├── opik_optimizer/                  # ✨ 独立 SDK:Agent Optimizer(4 家独家)
│   └── code_generation/                 # SDK 代码生成
├── deployment/                          # Docker / K8s / Helm
├── extensions/cursor/                   # Cursor IDE 扩展
├── tests_end_to_end/                    # Cross-stack E2E(Playwright)
├── tests_load/                          # Load tests
├── .agents/                             # ✨ 4 家最完整的 Claude Code 生态目录
│   ├── agents/        # 6 个 builtin:planner / build-fixer / code-reviewer / meta-auditor / planner / test-runner / architect
│   ├── commands/comet/  # comet 相关 slash commands
│   ├── docs/          # agent 友好文档
│   ├── mcp.json       # MCP servers(GitHub + Slack via Docker)
│   ├── rules/         # 4 个 .mdc rule:agents / code-style / git-workflow / security
│   └── skills/        # 10 个 skill:analytics-instrumentation / diagram-generation / documentation / local-dev / opik-backend / opik-frontend / playwright-e2e / python-sdk / typescript-sdk / write-docs
├── .claude/rules/                       # Claude Code 独立 rule(跟 .agents/rules 分离)
├── .codex -> .agents (symlink)          # Codex 也用 .agents/
├── .cursor -> .agents (symlink)         # Cursor 也用 .agents/
├── AGENTS.md (8KB)                      # 完整 build/test/lint/PR 命令清单
├── CHANGELOG.md                         # 版本历史(1.7.0 提及)
├── opik.sh / opik.ps1                   # 运维脚本(7 种 service profile:--infra/--backend/--guardrails/--build/--verify/--stop/--info)
└── Makefile (8KB)                       # precommit-sdks 等
```

## 关键 file:line 锚点

### Agent Optimizer(4 家独家)
- `sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26` — **7 个 optimizer 算法**(README 只提 5 个,源码实际 7 个):
  - `GepaOptimizer` — GEPA(Genetic-Pareto)
  - `MetaPromptOptimizer` — meta-prompting
  - `EvolutionaryOptimizer` — 遗传算法 prompt 进化
  - **`HierarchicalReflectiveOptimizer`** — README 没提
  - **`HRPO`** — README 没提(可能 Hierarchical Reflective Prompt Optimization)
  - `FewShotBayesianOptimizer` — few-shot + Bayesian
  - `ParameterOptimizer` — temperature/top_p Bayesian
- `sdks/opik_optimizer/src/opik_optimizer/__init__.py:27-31` — `ParameterSearchSpace` / `ParameterSpec` / `ParameterType` 参数搜索空间类型
- `sdks/opik_optimizer/src/opik_optimizer/base_optimizer.py:` — `BaseOptimizer(ABC)` + `optimize_prompt()` + `optimize_mcp()` 统一接口(`OptimizationResult` 标准返回)
- `sdks/opik_optimizer/src/opik_optimizer/agents/optimizable_agent.py` — `OptimizableAgent` 优化输入抽象
- `sdks/opik_optimizer/src/opik_optimizer/api_objects/chat_prompt.py` — `ChatPrompt.tools` 支持 MCP tool call
- `sdks/opik_optimizer/src/opik_optimizer/core/{results,evaluation,runtime,agent,llm_calls,state}.py` — 核心运行时
- `sdks/opik_optimizer/src/opik_optimizer/metrics/multi_metric_objective.py` — `MultiMetricObjective` 多 metric 优化目标
- `sdks/opik_optimizer/src/opik_optimizer/metrics/{levenshtein_accuracy,answer_correctness}.py` — 内置 metric
- `sdks/opik_optimizer/src/opik_optimizer/datasets/` — **8 个内置 benchmark dataset**:
  - `gsm8k.py` — 小学数学
  - `truthful_qa.py` — 真实性 QA
  - `halu_eval.py` — Hallucination eval
  - `ai2_arc.py` — AI2 ARC reasoning
  - `arc_agi2.py` — ARC AGI v2
  - `pupa.py` — PUPA
  - `medhallu.py` — Medical hallucination
  - `tiny_test.py` — tiny test fixture
- `sdks/opik_optimizer/README.md:9-15` — 8 个 Key Features 标榜:Standardized API / Optimizer Chaining / MCP Support / Consistent Results / Counter Tracking / Multimodal Prompts(text/image/audio/video)/ Type Safety / Backward Compatibility
- `__init__.py:35-37` — `# FIXME: Remove once LiteLLM issue is resolved` + `os.environ.setdefault("LITELLM_LOCAL_MODEL_COST_MAP", "True")` — LiteLLM 一等公民,绕 issue#16813 workaround

### Python SDK Evaluation
- `sdks/python/src/opik/evaluation/` 子模块:`engine/` / `metrics/` / `models/` / `samplers/` / `scorers/` / `suite_evaluators/` / `threads/`
- `sdks/python/src/opik/evaluation/test_case.py:` — `@dataclasses.dataclass TestCase { trace_id, dataset_item_id, task_output, dataset_item_content, mapped_scoring_inputs, dataset_item }` — **Pytest 风格 test case 是 first-class**
- `sdks/python/src/opik/evaluation/test_result.py` — `TestResult` 配套
- `sdks/python/src/opik/evaluation/metrics/` 4 大类:
  - `conversation/` + `conversation_types.py` — 对话级 metric
  - `heuristics/` — 规则级 metric
  - `llm_judges/` — LLM-as-judge metric(hallucination / moderation / answer relevance / context precision 等)
  - `ragas_metric.py` — Ragas 集成(跟 Langfuse 同名)
- `sdks/python/src/opik/evaluation/metrics/{aggregated_metric,base_metric,arguments_validator,score_result,arguments_helpers}.py` — 框架基础设施

### Java Backend Domain(`apps/opik-backend/src/main/java/com/comet/opik/domain/`)
- **Agent Configuration**(4 家里 Java 实现最完整):
  - `AgentBlueprint.java` — Agent 蓝图(模板)
  - `AgentConfig.java` — Agent 实例配置
  - `AgentConfigService.java` + `AgentConfigDAO.java` — Service + DAO 分层
  - `AgentConfigEnv.java` + `AgentConfigValue.java` — Env + Value 隔离
- **Assertion 系统**(Pytest 风格 first-class):
  - `AssertionResultDAO.java` + `AssertionResultService.java` + `AssertionResultMapper.java` — Service + DAO + Mapper 三层
- **Feedback Definition Model**(按 dataType 拆 OOP 多态):
  - `BooleanFeedbackDefinitionDefinitionModel.java`
  - `CategoricalFeedbackDefinitionDefinitionModel.java`
  - (推断还有 Numeric / Text 等,未深查)
- **Alert 系统**(4 家里独家完整 alerting):
  - `AlertDAO.java` + `AlertService.java`
  - `AlertTriggerConfigDAO.java` + `AlertTriggerDAO.java`
  - `alerts/` 子目录
- **Annotation Queue**:`AnnotationQueueDAO.java` + `AnnotationQueueService.java`
- **Comment**:`CommentDAO.java` + `CommentService.java` + `CommentEntityRef.java`
- **Comet 桥**:`ConnectBridgeService.java` — **Opik ↔ Comet 主平台互通**(用户 SkillForge 跟其他平台桥接的参考实现)
- **Cost**:`cost/` 独立子模块
- **Dashboard**:`DashboardDAO.java` + `DashboardService.java`
- **CSV / Dataset Export**:`CsvDatasetExportProcessor.java` + `CsvDatasetExportService.java` + `CsvDatasetItemProcessor.java` + `DatasetExportJobDAO.java` + `DatasetExportJobService.java` + `DatasetExportMessage.java` + `DatasetExpansionService.java`(7 个文件,export 流水线工程化深)
- **Trace attachment**:`attachment/` 子目录

### Java API(`apps/opik-backend/src/main/java/com/comet/opik/api/`)
- Agent config REST:`AgentConfigCreate.java` + `AgentConfigEnvSetByName.java` + `AgentConfigEnvUpdate.java` + `AgentConfigRemoveValues.java`
- Assertion REST:`AssertionResult.java` + `AssertionResultBatch.java` + `AssertionResultBatchItem.java` + `AssertionScoreAverage.java` + `AssertionStatus.java`
- Annotation Queue REST:`AnnotationQueue*.java`(7 个文件,含 `AnnotationQueueSearchCriteria.java`)
- Alert REST:`Alert.java` + `AlertEventType.java` + `AlertTrigger.java` + `AlertTriggerConfig.java` + `AlertTriggerConfigType.java` + `AlertType.java`
- Batch:`BatchDelete.java` + `BatchDeleteByProject.java`
- BI:`BiInformationResponse.java`
- 流式响应:`ChunkedResponseHandler.java`
- 自定义列:`Column.java`

### `.agents/` 完整 Claude Code 生态(4 家最完整)
- `.agents/agents/` **6 个 builtin agent**(每个都是真 Claude Code subagent 格式,带 example-based description + model + color + tools 白名单):
  - **planner.md**(sonnet, blue, tools=`["Read", "Grep", "Glob"]`)— 复杂 feature 分步规划,read-only 工具
  - **build-fixer.md**(haiku, yellow, tools=`["Bash", "Read", "Edit"]`)— **最小修改修编译错误**,选 haiku 模型快
  - **code-reviewer.md** — 代码审查
  - **meta-auditor.md**(描述里叫 config-auditor)— **proactive**,创建 rule/skill/agent/mcp 前先检查防 bloat
  - **planner.md / test-runner.md / architect.md**
- `.agents/skills/` **10 个 skill**:analytics-instrumentation / diagram-generation / documentation / local-dev / opik-backend / opik-frontend / playwright-e2e / python-sdk / typescript-sdk / write-docs
- `.agents/rules/` **4 个 .mdc rule**(.mdc = Markdown with Cursor metadata):agents / code-style / git-workflow / security
- `.agents/commands/comet/` — comet 相关 slash commands
- `.agents/mcp.json` — MCP servers(GitHub MCP via Docker + Slack)
- `.codex -> .agents` + `.cursor -> .agents` symlink — **完美的 LLM agent 工具普适性**
- `.github/instructions/` + `.github/copilot-instructions.md` — Copilot 专属(AGENTS.md 明确说其他 agent 忽略这俩)

### Pytest 集成 + CI/CD
- `sdks/python/src/opik/evaluation/test_case.py` — `TestCase` dataclass
- `sdks/python/src/opik/evaluation/test_result.py` — `TestResult` 配套
- AGENTS.md `## Build, Test, and Development Commands` — 完整命令清单:`./opik.sh --verify` / `scripts/dev-runner.sh --be-only-restart` / `cd apps/opik-frontend && npm run lint && npm run test` / `cd apps/opik-backend && mvn test` / `cd sdks/python && pytest tests/unit tests/e2e` / `make precommit-sdks`

### Service Profile(`opik.sh`)
- `--infra` — 只起基础设施(databases / caches)
- `--backend` — 基础设施 + backend
- `--guardrails` — Guardrails 跟任意 profile 组合(`--guardrails` / `--backend --guardrails`)
- `--build` — 重建 image
- `--verify` — 健康检查
- `--stop` — 停服务
- `--info` / `--help` — 调试

### Generated Files(`apps/opik-backend/src/main/resources/`)
- `model_prices_and_context_window.json` + `apps/opik-frontend/src/data/model_prices_and_context_window.json` — **generated artifacts,AGENTS.md 明确说"不能手动改",来自 repository updater 自动化** — 跟 Langfuse `worker/src/constants/default-model-prices.json` 同位置同思路

## 与 wiki 现有记录的关系

### 印证
- **`.agents/` 完整目录 + symlink to .codex/.cursor** 印证 [[../source/langfuse]] `.agents/` dogfood 思路(**Opik 比 Langfuse 更彻底**:完整 agents/commands/rules/skills/mcp/docs)
- **6 个 builtin agent + Claude Code subagent 标准格式(example-based description + tools 白名单)** 印证 Claude Code subagent 模式是业内事实标准
- **Ragas 集成**(`metrics/ragas_metric.py`)印证 [[../eval/langfuse]] Ragas 是 RAG eval 行业标杆
- **ClickHouse 列存** 印证 [[../concept/observability]] "> 10K traces/day 切 ClickHouse" 行业共识
- **`generated artifact` 模型定价** 印证 Langfuse + Opik 一致工程模式(集中维护 + automation 同步)

### 矛盾
- **评分模型完全不同设计哲学**:
  - [[../eval/langfuse]] `Score = ScoreSource(3) × ScoreDataType(5)` 数据抽象统一
  - Opik **Assertion(pass/fail + reason)+ Feedback Definition Model(OOP 多态,按 dataType 拆 Java 子类)**
  - **两条路线对照**(见 [[../concept/agent-eval]] 更新)
- **Self-evolution 第 4 类(Optimizer 算法自动改 prompt)** wiki 当前 [[../concept/self-evolution]] 3 类分类不够,需扩

### 新增
- **eval/opik.md**(新建,跟 [[../eval/langfuse]] 直接对照)
- **concept/prompt-optimization.md**(新建,7 算法对照)
- **`concept/self-evolution.md` 加第 4 类**(Optimizer 算法自动改 prompt)
- **`concept/observability.md` 加 Opik Java/Spring/MySQL 路线**(platform observability 第二条路)
- **`concept/agent-eval.md` 加 Opik 维度**(OOP 多态评分 / 8 内置 benchmark / Pytest first-class)

## 事实漂移 / 注意点

- **Opik 版本** 仓库 `sdks/python/pyproject.toml` 只标 `python_version=3.10`,没有统一 monorepo semver。**CHANGELOG.md 提到 1.7.0** 是当前稳定版,commit hash `72b2f1ca89` @ 2026-05-15 锁版本
- **`AGENTS.md` 顶层 + 模块级**:Opik 是真 monorepo,顶层 AGENTS.md 是 router,模块自己有 `apps/opik-backend/AGENTS.md` + `sdks/opik_optimizer/AGENTS.md` 等(本次只读了顶层)
- **README 列 5 个 optimizer,源码实际 7 个**(`__init__.py:18-26`)— **HierarchicalReflectiveOptimizer 跟 HRPO 没在 README 提及**(可能是新加 + 文档未跟上)
- **`.codex` + `.cursor` symlink**:Codex 跟 Cursor 都吃 `.agents/` 内容,但 Codex 实际是把 `.agents/` 内的 AGENTS.md 当 contributor guide(详见 [[../harness/codex]] AGENTS.md 路径约定),Cursor 实际吃的是 `.mdc` rules(`.cursor` 风格元数据)— **同一份目录被不同 agent 工具用不同方式消化**
- **Guardrails 是 Python 独立 service**(`apps/opik-guardrails-backend/opik_guardrails/`)— SkillForge guardrails 当前是 Java 内置,Opik 把它独立 service 化值得借鉴
- **Python sandbox executor** 独立 service(`apps/opik-sandbox-executor-python/`)— 用户自定义 evaluator/metric 跑在沙箱里,**SkillForge 跑用户自定义 hook/skill 应有等价 sandbox service**
- **Apache 2.0**(跟 Langfuse MIT 不同)— 商业兼容性强,Comet ML 一致选择
