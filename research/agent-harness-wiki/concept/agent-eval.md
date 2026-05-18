---
type: concept
name: agent-eval
status: draft
verification: source-grounded
updated: 2026-05-16
sources:
  - "[[../source/openclaw]]"
  - "[[../source/langfuse]]"
  - "[[../source/opik]]"
  - "[[../../OpenClaw QA Character Eval 系统分析]]"
related_harnesses:
  - "[[../harness/openclaw]]"
  - "[[../harness/hermes-agent]]"
  - "[[../harness/claude-code]]"
related_evals:
  - "[[../eval/langfuse]]"
  - "[[../eval/opik]]"
---

# Agent Eval(Agent 评测框架)

## 定义

**评测 agent 而不是评测单点 LLM**。Agent 评测需要:
- **完整环境**:不只是 prompt → response,而是 prompt → tools → 多轮交互 → 最终状态
- **场景驱动**:固定起点 + 用户行为模拟 + 成功标准
- **多维度评分**:正确性 + 一致性 + 风格 + 工具使用合理性 + 安全
- **可重现**:相同输入相同环境多次跑结果稳定(seed/temperature 锁定)

跟传统 LLM benchmark(MMLU/GSM8K)的区别:**LLM benchmark 是单轮 IO,agent eval 是完整 session loop**。

---

## 设计空间(8 维)

> 2026-05-16 update v1:加入 Langfuse 后,每个维度补"Platform 路线"选项
> 2026-05-16 update v2:加入 Opik 后,每个维度补"OOP 多态 + Optimization 路线"选项

| 维度 | 选项 |
|---|---|
| **评测对象** | 单 LLM 单点 / agent session 多轮 / multi-agent 协作 / 长期记忆持续测 / **observation 单元(Langfuse Wide Events 路线)** / **TestCase(Opik Pytest 风格 `test_case.py`)** |
| **数据集来源** | 人工编 / **production trace 自动归集(Langfuse `addDatasetRunItemsToEvalQueue`)**/ synthetic / 用户反馈 / eval 失败案例 / **8 内置 benchmark dataset(Opik 独家:gsm8k / truthful_qa / halu_eval / ai2_arc / arc_agi2 / pupa / medhallu / tiny_test)** |
| **场景定义** | hardcoded test case / YAML DSL(OpenClaw)/ JSON schema / 自然语言 + LLM 编译 / **EvalTemplate 版本化模板(Langfuse)** / **Pytest TestCase + dataset 驱动(Opik)** |
| **执行环境** | mock 全部 / mock 部分 / **真实进程 + IPC**(OpenClaw 路线)/ **SDK 装饰器 `@observe()/@track`**(Langfuse + Opik 路线)/ **沙箱独立 service(Opik `apps/opik-sandbox-executor-python/`)** |
| **评分方式** | rule-based / execution-based / LLM-as-judge / 人工 annotation / 综合 / **Score 统一抽象(Langfuse `ScoreSource × ScoreDataType` 3×5 矩阵)** / **Assertion + Feedback OOP 多态(Opik Java 子类继承,`BooleanFeedbackDefinitionDefinitionModel` / `CategoricalFeedbackDefinitionDefinitionModel`)** |
| **判官模型** | 同款 / 更强模型(production judge)/ 多 judge 投票 / **provider-agnostic router**(Langfuse `defaultEvalModelRouter.ts`)/ **LiteLLM 一等公民(Opik Optimizer 强依赖)** |
| **可复现性** | seed + temperature 锁定 / docker / VM / 模型版本锁定 / 完整快照 / **EvalTemplate 版本 + prompt 版本 + 模型定价版本三轨绑定**(Langfuse + Opik 都做)/ **OptimizationResult 标准化对象(Opik,re-run 可对比)** |
| **production loop** | 离线 only / production 失败回灌 eval / prompt A/B + 自动晋升 / **trace → dataset → eval → score 全自动**(Langfuse)/ **trace → dataset → eval → Optimizer 自动改 prompt → promote(Opik 4 家独家闭环,详见 [[prompt-optimization]])** |

---

## 各实现怎么做

### [[../harness/openclaw]] QA Character Eval(**production 最完整**)

`extensions/qa-lab/src/`,**两阶段、多模型横向对比、真实环境**。

**入口**:`character-eval.ts:393 runQaCharacterEval`

**第一阶段:候选模型并行**

```
对每个候选模型串行调用 runQaSuite()
↓
启动 3 个服务:
  startQaLabServer() — MCP server(提供测试工具 API)
  startQaMockOpenAiServer() — Mock OpenAI(mock 模式用)
  startQaGatewayChild() — **真实 Gateway 子进程**(完整 `openclaw gateway run`,IPC 通信)
↓
读取 qa/scenarios/index.md pack 配置
↓
启动 QA Agent("Dev C-3PO" protocol-minded QA 操作员)
注入场景 kickoffTask 作第一条消息
↓
执行 Flow DSL(scenario-flow-runner.ts)
↓
Bus 消息总线驱动后续交互
```

**场景定义 YAML 双 fence**:

```markdown
```yaml qa-scenario
id: character-vibes-gollum
title: Gollum 角色对话
surface: telegram
objective: 保持角色一致性并完成用户任务
successCriteria:
  - 使用 gollum 口吻回复
  - 不暴露是 AI
```

```yaml qa-flow
steps:
  - name: 发送消息
    actions:
      - call: sendMessage
        args: ["你好,我的宝贝"]
      - assert: reply contains "precious"
      - saveAs: firstReply
  - name: 追问
    actions:
      - call: sendMessage
        args: ["${vars.firstReply} 你喜欢什么?"]
      - assert: reply.length > 10
```
```

**Flow DSL 动态编译执行**(scenario-flow-runner.ts):

```typescript
const fn = new AsyncFunction(...contextNames, `return (${expr});`)
await fn(...contextValues)
```

**支持 actions**:`sendMessage` / `assert(reply contains / length / regex)` / `saveAs` / `${vars.xxx}` 引用 / `${env.xxx}`

**第二阶段:Judge 横向打分**

- 所有候选 transcript 收集
- Judge 模型(更强算力)读所有 transcript 横向打分
- `manual-lane.runtime.ts` 单轮推理(Judge 专用)
- `report.ts` 生成 Markdown 报告

**关键设计**:
- **真实 Gateway 子进程**(`startQaGatewayChild()`),不是 mock —— 跑真实代码路径
- **Lab Server 是 MCP server**(`lab-server.ts`)—— 暴露测试辅助工具给 Gateway 内 Agent(记录消息 / 读最新场景报告 / 检查 QA 状态)
- **Bus 消息总线**(`bus-state.ts / bus-server.ts`)—— Agent 发出的消息走 bus,测试代码可订阅

### [[../harness/hermes-agent]] eval

`[doc-only]`(本次未深读 environments/ 源码):

- **environments/** Atropos RL 训练环境(面向 research / RL training)
- **batch_runner.py** 批量 trajectory 生成
- **mini_swe_runner.py** SWE-bench mini runner

**特点**:面向 **RL training data generation + research benchmark**,**不是 production agent QA**。
**无**:对外 framework UI / dataset 管理 / prompt A/B 自动晋升 / LLM-as-judge dashboard。

### [[../harness/claude-code]] eval

`[doc-only]`:
- **内部 transcript classifier**(AFK mode ML — README 提及)
- **YOLO classifier**(permission ML — `hooks/toolPermission/handlers/interactiveHandler.ts:25, 483`)
- **`services/analytics/` + GrowthBook + `tengu_*` events** 大量
- **ANT-only**:`~/.config/claude/dump-prompts/` debug prompt dumping

**特点**:**production analytics + ML classifier**(不是 framework),数据驱动决策但**对外用户无暴露**。

### [[../eval/langfuse]] — **Platform 级 eval 标杆**(2026-05-16 新加)

**Langfuse 跟 OpenClaw QA Character Eval 完全不同定位**:
- OpenClaw QA Character Eval = **agent runtime 自带 framework**,跑自家 agent 真实 Gateway
- Langfuse = **中心化 platform**,接任意 LLM 应用的 trace,跑 eval(LLM-as-judge / annotation / API),出 Score

**关键 file:line 锚点**(详见 [[../eval/langfuse]] 和 [[../source/langfuse]]):

#### Score 数据模型(3 × 5 矩阵)
- `scores.ts:4` — `ScoreSourceArray = ["API", "EVAL", "ANNOTATION"]`
- `scores.ts:46` — `ScoreDataTypeArray = ["NUMERIC", "CATEGORICAL", "BOOLEAN", "CORRECTION", "TEXT"]`
- **`scores.ts:13-21` Public API EVAL source 保留**(`as const satisfies readonly ScoreSourceType[]` 编译期 subset)— 外部 API 永远不能创建 `EVAL` source,防伪造内部 evaluator output
- `scores.ts:30-40` ANNOTATION 必须带 configId 除非 CORRECTION dataType
- `scores.ts:44` `TEXT_SCORE_MAX_LENGTH = 500` 防 LLM-as-judge rationale 无界

#### Worker Queue 体系(32 个)
- `queues.ts:322-358` 32 个 QueueName,**7 处 main + secondary 双队列分流**:
  - `IngestionQueue` + `IngestionSecondaryQueue`(`:333-334`)
  - `OtelIngestionQueue` + `OtelIngestionSecondaryQueue`(`:331-332`)
  - `EvaluationExecution` + `EvaluationExecutionSecondaryQueue`(`:326-327`,注释明确"Separates high-throughput eval projects from other projects")
- `queues.ts:353` DLQ 一等公民:`DeadLetterRetryQueue`
- `queues.ts:355-356` Audit 流:`EntityChangeQueue` + `EventPropagationQueue`

#### Production Loop 单线打通
- `web/src/features/evals/server/addDatasetRunItemsToEvalQueue.ts` — production trace → dataset → eval → score 全自动
- `defaultEvalModelRouter.ts` — LLM-as-judge 用哪个模型
- `evaluator-preflight.ts` — eval 跑前检查
- `EvalTemplate` 版本控制(`router.ts:142,475`)— 跟 prompt 一样版本化

#### Ragas / 行业框架集成
- `web/src/features/evals/components/ragas-logo.tsx` Ragas RAG eval 集成
- 23+ 框架自动 instrumentation(OpenAI/Langchain/LlamaIndex/Haystack/LiteLLM/Vercel AI/Mastra/DSPy/CrewAI/AutoGen/Goose/smolagents...)

**Langfuse vs OpenClaw QA Character Eval 的根本区别**:

| 维度 | OpenClaw QA(runtime framework) | Langfuse(platform) |
|---|---|---|
| 定位 | Agent 自家 framework | 接任意 LLM 应用的中心化 platform |
| 目标 | 自家 agent 不退化 | 任何用户 LLM 应用 |
| 数据模型 | YAML DSL 编译 JS scenario | Score 统一抽象(3×5 矩阵) |
| 触发 | 手工 / CI 跑 | Production trace 自动入 |
| Evaluator | scenario 内 assert + character scoring | LLM-as-judge + annotation + API + custom |
| 存储 | 自家 db / 文件 | ClickHouse + Postgres + Redis + S3 |
| 多租户 | 单 agent / 单租户 | 多租户 + main/secondary 双队列 |
| 集成生态 | 自家 | 23+ 框架 |

**互补关系**:OpenClaw QA Character Eval 这种"自家 framework"+ Langfuse 这种"中心化 platform" **完全可以共存** — 自家 framework 跑完出 score → 通过 OTel / SDK 写到 Langfuse → 中心化看板看历史趋势。

### [[../eval/opik]] — **OOP 多态评分 + 7 算法 Optimizer**(2026-05-16 新加)

**Opik 跟 Langfuse 同赛道但走完全不同路线**:

| 维度 | Langfuse | Opik |
|---|---|---|
| Backend | Node.js | **Java + Spring Boot** |
| Main DB | ClickHouse + Postgres | **ClickHouse + MySQL** |
| 评分模型 | `Score = Source(3) × DataType(5)` 数据抽象 | **Assertion + FeedbackDefinitionModel OOP 多态**(Java 子类继承) |
| SDK | Python + JS/TS | Python + TS + **Ruby via OTel** |
| Optimizer | ❌ 无 | **✅ 7 算法独立 SDK**(`sdks/opik_optimizer/`) |
| 内置 benchmark | ❌ | **✅ 8 个(gsm8k / truthful_qa / halu_eval / ...)** |
| Pytest 集成 | SDK 支持 | **first-class(`test_case.py` + `test_result.py`)** |
| Alert 系统 | webhook | **完整 alerting(`AlertDAO` + `AlertEventType` + `AlertTriggerConfig*`)** |
| Guardrails | ❌(用户自接) | **✅ 独立 service(`apps/opik-guardrails-backend/`)** |
| Sandbox executor | ❌ | **✅ 独立 service(`apps/opik-sandbox-executor-python/`)** |
| `.agents/` dogfood | 16 skills | **6 agents + 10 skills + 4 rules + commands + mcp + docs(最完整)** |

**关键 file:line 锚点**(详见 [[../eval/opik]] 和 [[../source/opik]]):

#### Assertion + Feedback 双轨(`apps/opik-backend/src/main/java/com/comet/opik/domain/`)
- Assertion:`AssertionResultDAO.java` + `AssertionResultService.java` + `AssertionResultMapper.java`(Pytest 风格 pass/fail + reason)
- Feedback Definition Model:`BooleanFeedbackDefinitionDefinitionModel.java` + `CategoricalFeedbackDefinitionDefinitionModel.java`(按 dataType 拆 Java 子类)

#### 7 个 Optimizer 算法(`sdks/opik_optimizer/src/opik_optimizer/__init__.py:18-26`)
- `EvolutionaryOptimizer` / `FewShotBayesianOptimizer` / `MetaPromptOptimizer` / `GepaOptimizer` / `ParameterOptimizer` / `HierarchicalReflectiveOptimizer` / `HRPO`
- 统一 `BaseOptimizer(ABC).optimize_prompt()` + `OptimizationResult` 标准返回
- **Optimizer Chaining**:一个 optimizer 输出 → 下一个输入
- 详见 [[prompt-optimization]] 8 维设计空间

#### 8 个内置 benchmark dataset(`sdks/opik_optimizer/src/opik_optimizer/datasets/`)
- `gsm8k.py` / `truthful_qa.py` / `halu_eval.py` / `ai2_arc.py` / `arc_agi2.py` / `pupa.py` / `medhallu.py` / `tiny_test.py`

#### Pytest 集成(`sdks/python/src/opik/evaluation/`)
- `test_case.py` — `TestCase dataclass`
- `test_result.py` — `TestResult` 配套

#### Alert 系统(4 家独家)
- Java DAO:`AlertDAO.java` + `AlertTriggerConfigDAO.java` + `AlertTriggerDAO.java`
- API:`Alert.java` + `AlertEventType.java` + `AlertTrigger.java` + `AlertTriggerConfig.java` + `AlertTriggerConfigType.java` + `AlertType.java`

#### `.agents/` 完整 Claude Code 生态(4 家最深 dogfood)
- 6 个 builtin agent(planner / build-fixer / code-reviewer / meta-auditor / test-runner / architect),每个真 subagent 标准格式(example-based description + model + color + tools 白名单)
- 10 个 skill(analytics-instrumentation / clickhouse-best-practices / opik-backend / playwright-e2e / ...)
- 4 个 .mdc rule + `.codex -> .agents` + `.cursor -> .agents` symlink

**Opik vs Langfuse 设计哲学根本差异**:
- **Langfuse**:把 LLM eval **统一成数据抽象**(Score 实体,3×5 矩阵,**TypeScript 类型系统硬约束**)
- **Opik**:把 LLM eval **拆成 OOP 实体**(Assertion / Feedback 各种子类,**Java 多态 + IDE refactor 友好**)
- **两条路线都能扩展,选哪条取决于生态**:JS/TS 项目走 Langfuse 数据抽象,Java/Spring 项目走 Opik OOP 多态

**Opik 跟 OpenClaw QA Character Eval 互补**:
- OpenClaw QA = agent runtime 自家 framework + YAML DSL scenario
- Opik = 中心化 platform + Pytest 风格 + Optimizer 闭环
- **可共存**:OpenClaw 跑完出 score → 通过 OTel / SDK 写到 Opik → 中心化看板 + Optimizer 自动改 prompt

### SkillForge eval pipeline(已知)

从 SkillForge README:
- **Automated eval, prompt A/B testing, promotion pipelines** —— 跟 Anthropic 的 evals 风格类似
- **Self-improving agents** —— 评测结果直接驱动 prompt 改进
- 具体实现未对照(本次未 ingest SkillForge 源码)

**对照 Langfuse 后的具体建议**(详见 [[../eval/langfuse]] frontmatter `skillforge_takeaway`):
1. **Score 数据模型抽象**:直接照搬 `ScoreSource × ScoreDataType` 3×5 矩阵
2. **Public API EVAL source 保留**:防外部伪造 evaluator output
3. **Main + Secondary 双队列分流**:多租户必备
4. **`addDatasetRunItemsToEvalQueue` production loop**:Prompt A/B + 自动晋升的开源参考
5. **EvalTemplate 版本控制**:eval 跟 prompt 一样版本化

---

## Trade-off

### Mock 全部 vs 真实进程

| | Mock 全部 | **真实进程**(OpenClaw)|
|---|---|---|
| 速度 | 快 | 慢(进程启动开销)|
| 覆盖 | 仅测 logic | 测真实代码路径,含 race condition / IPC / 异步 |
| 维护成本 | mock 库要跟代码同步 | 无,跑真实 |
| Bug 发现 | 仅 mock 范围内 | **包括 production 才会出的 bug** |
| 推荐场景 | 单元测试 | **Agent eval / E2E** |

OpenClaw 选真实进程是**关键工程判断** —— agent 太多 emergent behavior,mock 永远不够。

### YAML DSL vs hardcoded test case

| | hardcoded(传统 unit test) | **YAML DSL**(OpenClaw) |
|---|---|---|
| 写场景 | 写代码 | 写 YAML |
| 代码-数据耦合 | 紧 | 松 |
| 非开发者写场景 | 难 | 可能(写 YAML) |
| 动态扩展 | 改代码重 build | 加 YAML 不改代码 |
| 风险 | 无 | YAML 语法错或 expr 错难调试 |

OpenClaw `new AsyncFunction(...)` 编译 YAML expr 为 JS —— **直接复用 JS 语言,不写自己的 DSL parser**。

### LLM-as-judge vs rule-based

| | Rule-based | LLM-as-judge |
|---|---|---|
| 客观性 | 高(deterministic) | 中(模型主观)|
| 灵活性 | 低(必须能 grep 出来) | 高(语义理解) |
| 成本 | 0 | 每次跑要 inference |
| 适合 | "tests pass" / "exit 0" / 数值断言 | 风格 / 一致性 / 用户体验 |

OpenClaw 用 LLM-as-judge(Judge 模型横向打分),适合 **character eval**(角色一致性 / 风格)。
SWE-bench 用 rule-based(代码 run pass)适合 **正确性 eval**。

---

## SkillForge 当前选择 + 演进方向

**当前已知**:
- Eval Pipeline:Automated eval + Prompt A/B + 自动晋升(从 README 可见)
- 但具体形态未对照源码

**建议演进**(基于 OpenClaw verified 实现):

1. **真实 Gateway 子进程跑** —— 借鉴 `startQaGatewayChild()` 路径,避免 mock 错位
2. **YAML DSL 编译执行** —— SkillForge 是 Java,可用 GraalJS 等价 `new AsyncFunction()`
3. **双阶段:候选并行 + Judge 排名** —— SkillForge prompt A/B 升级为 N-way 候选
4. **Lab Server (MCP)** 暴露测试工具给 Agent —— Agent 在 eval 模式下行为可观察 / 可干预
5. **YAML qa-scenario + qa-flow 双 fence** —— scenarios 跟代码解耦,**业务人员可写场景**

---

## 学习视角

### 这个概念存在的根本原因(motivation)

**Agent 评测跟 LLM 评测是两件事**。LLM benchmark 测"单轮 IO 准不准",agent eval 测"完整 session 跑下来好不好"。
agent 涉及:tool use / multi-turn / memory / 用户协作 / 异步事件 / 错误恢复 —— **mock 全部就失真了**。

**类比**:LLM benchmark ≈ 笔试。agent eval ≈ 模拟真实工作场景的实操考试。
笔试满分不代表能上岗,实操才能验证。

### 哪些 harness 做得好

| 设计点 | 谁做得好 | 为什么 elegant |
|---|---|---|
| **真实 Gateway 子进程跑** | OpenClaw QA Character Eval | 不 mock —— **测真实代码路径,捕 production bug** |
| **YAML DSL 编译 JS AsyncFunction** | OpenClaw `scenario-flow-runner.ts` | 不写自己的 parser,**复用 JS native** |
| **Lab Server (MCP) 暴露测试工具给 Agent** | OpenClaw `lab-server.ts` | Agent 行为可观察可干预 —— eval 工具就是 agent 工具的扩展 |
| **双阶段:候选并行 + Judge 排名** | OpenClaw | **多 candidate 横向比较** 比"单次跑分"信号强 |
| **production trace 录制 → eval data** | Claude Code transcript classifier(理论) | 真实用户行为 > 人工编 |
| **场景跟代码解耦** | OpenClaw YAML | **scenarios 可由 PM/QA 写,不需要开发** |

### 学习路径

1. **入门**:理解为什么 LLM benchmark 不够 —— 看 SWE-bench / τ-bench / GAIA 的论文
2. **进阶**:看 OpenClaw `extensions/qa-lab/` 笔记 —— 理解 production agent eval framework 形态
3. **对比**:看 Claude Code transcript classifier 设计(ML auto-approval)—— 理解 production data-driven 视角
4. **深入**:对照 SkillForge eval pipeline 实际实现 —— 看是否需要演进到 OpenClaw 形态
5. **高阶**:看 Inspect AI / Langfuse Eval / OpenAI Evals 等 framework —— 看业界 dataset + judge + reproducibility 设计

### 反模式

- ❌ **Mock 全部跑 agent eval** —— 失去 production race condition / IPC 覆盖
- ❌ **单次跑分** —— 单次结果方差大,应跑 N 次取统计
- ❌ **场景 hardcoded in test 代码** —— 跟业务逻辑紧耦合,非开发者无法写
- ❌ **只用 rule-based** —— 风格 / 一致性 / UX 无法 grep
- ❌ **只用 LLM-as-judge** —— 客观正确性 / 代码 run pass 应该用 rule
- ❌ **Eval 跟 production 完全隔离** —— production 失败案例应该回灌 eval(self-evolution 闭环)
- ❌ **不锁 seed / model version** —— 不可复现
