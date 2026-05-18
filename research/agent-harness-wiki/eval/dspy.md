---
type: eval-framework
name: dspy
status: active
language: python

# === 版本时效 ===
last_ingested_version: "3.2.1 @ commit 621c3a6 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/dspy]]"

# === 9 维度 ===
target: |
  **Prompt-as-program framework**(Stanford NLP, MIT, Omar Khattab 等)。**LLM 当成可编程模块,prompt + few-shot + 模型选择作为可优化 hyperparameter,用 15 种 optimizer 算法自动改进**。

  跟其他 eval 完全不同 layer:
  - [[../eval/langfuse]] / [[../eval/opik]] = **platform**(SaaS / self-host UI)
  - [[swe-bench]] / [[tau-bench]] = **benchmark**(dataset + harness)
  - **DSPy = framework**(Python library,用户 import 写代码)

  跟 [[../concept/prompt-optimization]] 的关系:**DSPy 是 prompt optimization 的学术研究产物 + 业内最早 first-class 框架(2022 DSP → 2024 DSPy)**;Opik 7 算法是 DSPy 子集的工程化包装。
  [verified-from-source]

dataset: |
  - **`dspy/datasets/`** 内置 dataset wrapper(HuggingFace `datasets` 集成)
  - 用户自定 dataset 用 **`dspy.Example`**(`primitives/example.py`)
  - 评测时通过 **`dspy.Evaluate(devset=..., metric=..., num_threads=...)`** 跑
  - 没有平台级 dataset 管理(对照 [[../eval/langfuse]] Dataset entity)
  [verified-from-source]

scorer: |
  **`dspy.Evaluate`**(`dspy/evaluate/__init__.py` 导出 `Evaluate`)— 把 metric function 跟 devset 配对:
  ```python
  evaluator = dspy.Evaluate(devset=dev, metric=accuracy_metric, num_threads=8)
  score = evaluator(my_program)
  ```

  **Metric 是 Python function**(签名 `(example, prediction, trace=None) → bool/float/Score`)— 用户自定义。
  没有内置 metric 库(对照 [[../eval/opik]] `metrics/` 4 大类 / [[ragas]] 12+ 内置 metric)— 因为 DSPy 哲学是"用户自己写 metric"。

  **Prediction 数据模型**(`primitives/prediction.py`)是核心 — `Module` 输出统一是 `Prediction` 实例,可被 metric function 消费。
  [verified-from-source]

reproducibility: |
  - **`dspy.cache = DSPY_CACHE`** 全局 LRU cache — 同 prompt + 同模型 hits cache,reruns 完全一致
  - **`save / load`**(`utils/saving.py`)序列化完整 Module state(含 optimized prompt + few-shot examples)
  - **`UNSAFE_LM_STATE_KEYS = {"api_base", "base_url", "model_list"}`**(`predict.py:22`)— `_sanitize_lm_state` 默认从 saved state 移除 unsafe keys,防 prompt injection 偷换 API endpoint(`predict.py:25-43`)
  - **`load(allow_unsafe_lm_state=True)`** 显式开关给可信文件
  [verified-from-source]

coverage: |
  **15 个 Optimizer 算法**(`teleprompt/__init__.py:1-18`):
  1. **AvatarOptimizer** — Avatar agent 优化
  2. **BetterTogether** — 多 optimizer 协作(meta)
  3. **BootstrapFewShot** — 基础 bootstrap few-shot
  4. **BootstrapFinetune** — Bootstrap + fine-tune
  5. **COPRO** — Coordinate Prompt Optimization(对照 [[../eval/opik]] MetaPromptOptimizer)
  6. **Ensemble** — 多 program ensemble(决策融合)
  7. **GEPA** — **Genetic-Pareto**(2025-07 paper,**Opik GepaOptimizer 是这个的工程化包装**)
  8. **InferRules** — 从数据 infer prompt rules
  9. **KNNFewShot** — KNN 选 few-shot examples
  10. **MIPROv2** — Multi-prompt Instruction PROposal v2(2024-06 paper)
  11. **BootstrapFewShotWithRandomSearch** — Random search baseline
  12. **BootstrapFewShotWithOptuna** — Optuna Bayesian(对照 Opik FewShotBayesianOptimizer)
  13. **LabeledFewShot** — 简单 labeled few-shot
  14. **SIMBA** — SIMBA(Simba 算法)
  15. **`bootstrap_trace_data`** — Trace data 收集 helper

  **13 个 Predict Module**(`predict/__init__.py`):
  - **Predict**(基础)/ **ChainOfThought**(CoT)/ **ReAct** / **Refine**(self-correction)/ **Retry** / **BestOfN** / **MultiChainComparison** / **Parallel** / **CodeAct** / **ProgramOfThought** / **KNN** / **Avatar** / **RLM**

  **Adapter 系统**(`dspy/adapters/`):ChatAdapter / **JSONAdapter** / **XMLAdapter** / TwoStepAdapter + typed I/O(Image / Audio / File / History / Tool / ToolCalls / **Code** / **Reasoning**)

  **不覆盖**:
  - 没有 platform UI(对照 [[../eval/langfuse]] / [[../eval/opik]])
  - 没有多租户隔离
  - 没有 production trace 自动归集(对照 [[../eval/langfuse]] `addDatasetRunItemsToEvalQueue`)
  [verified-from-source]

production_loop: |
  **DSPy 是 dev-time framework**,不是 production runtime。
  Production loop 闭环靠**用户自接** Langfuse / Opik / 自家 platform。

  典型工作流(README + 论文):
  1. 用户写 `Signature` + 用 `Module` 组合
  2. 准备 trainset / devset(`dspy.Example`)
  3. 选 Optimizer:`compiled = teleprompter.compile(student=program, trainset=trainset)`
  4. **`compiled` 是优化后 Module(含 optimal prompt + few-shot)**
  5. Save → Production deploy
  6. Production 看 metric drift → 回 dev 重新 compile

  **GEPA paper(2025-07)** 标榜 "Reflective Prompt Evolution **Can Outperform Reinforcement Learning**" — DSPy 想替代 RLHF 走 prompt-only 路线。
  [verified-from-source]

automation: |
  - **`dspy.streamify`** 流式 wrapper(`streaming/streamify.py`)
  - **`dspy.utils.asyncify / syncify`** async/sync 互转
  - **`dspy.utils.usage_tracker.track_usage`** — token usage 跟踪 decorator
  - **`dspy.utils.callback.BaseCallback`** — instrumentation hook(类 Langchain callback)
  - **`num_threads`**(`Evaluate` 参数)— 并发评测
  [verified-from-source]

cost: |
  - **`DSPY_CACHE`** 全局缓存大幅降本(reruns hit cache)
  - **`track_usage`** decorator 跟踪 token 用量
  - LiteLLM `response_cost`(via `_hidden_params`)拿 per-call cost(跟 [[tau-bench]] 同套)
  - **15 optimizer 算法跑成本差异大**:`LabeledFewShot` 最便宜(~10 LLM calls)/ `GEPA` 最贵(N generations × population size)
  [verified-from-source]

skillforge_takeaway: |
  ## 🎯 关键定位:DSPy 是 SkillForge PromptImproverService 的"学术 reference"

  SkillForge 自家路线:
  - **PromptImproverService**(已交付 EVAL-V2 / SKILL-CANARY-ROLLOUT V2)— LLM 驱动 + 人审 + canary
  - **MEMORY-LLM-SYNTHESIS**(4 类 proposal + 人审 gate)— meta self-evolution

  DSPy 走的是**完全自动 + 算法驱动**(无人审 + 离线 compile)。**SkillForge 更适合 production "人审 + 灰度回流",DSPy 适合 dev-time prompt 实验**。

  ## 可借鉴(给 SkillForge PromptImproverService 加强)

  1. **Signature 抽象**(`signatures/signature.py`)— SkillForge prompt template 可加 typed InputField / OutputField,把 prompt 从字符串升级为 typed program。**借鉴优先级:中**(SkillForge 已有 prompt versioning,Signature 是渐进升级)
  2. **`UNSAFE_LM_STATE_KEYS = {"api_base", "base_url", "model_list"}`**(`predict.py:22`)+ `_sanitize_lm_state`(`:25-43`)— **防 prompt injection 工程细节业内独家**,SkillForge 加载用户上传的 prompt / agent state 时应有同样防御
  3. **MIPROv2 + GEPA 2 个算法**(对照 Opik 7 算法):**MIPROv2 用于 instruction 优化(对照 SkillForge agent system prompt),GEPA 用于 prompt evolution(对照 SkillForge skill drafts)** — SkillForge 自家路线 + 2 算法 POC(类似 R10 决策)
  4. **`dspy.Evaluate(devset, metric, num_threads)`** 三参数 API — SkillForge EvalRun 类似,可学**`num_threads`** 并发参数(SkillForge 是否已支持?)
  5. **`Prediction` 数据模型统一**(`primitives/prediction.py`)— 所有 module 输出统一 `Prediction`,metric function 消费统一格式。**SkillForge EvalTask 输出已有 `actualOutput` 字段**,可对照确保统一性
  6. **`Adapter` 系统**(ChatAdapter / JSONAdapter / XMLAdapter / TwoStepAdapter)— SkillForge multi-provider 抽象可学 Adapter 模式,而非每 provider 一个 if-else
  7. **`Code` + `Reasoning` typed I/O**(`adapters/__init__.py`)— SkillForge 把"代码片段"和"推理过程"作为 typed value 而非纯字符串

  ## 不必跟
  - **DSPy 全套 15 algorithm**(工程量大,SkillForge 走自家 PromptImproverService 已够)
  - **DSPy 完全自动 + 无人审**(SkillForge `/evolve` + canary 灰度更安全)
  - **DSPy 没 platform UI**(SkillForge dashboard 已有)
  - **GEPA Reflective Prompt Evolution 替代 RLHF**(SkillForge 不是模型训练 platform)

  ## SkillForge 已领先
  - **Production canary 灰度回流**(SKILL-CANARY-ROLLOUT V2)— DSPy 完全没有
  - **多 surface 优化**(skill/prompt/behavior_rule/hook)— DSPy 主要 prompt(`Signature` 只能描述 input/output)
  - **MEMORY-LLM-SYNTHESIS + ATTRIBUTION-AGENT** 飞轮闭环 — DSPy 是 dev-time framework 没有

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **业内最早 first-class prompt optimization framework**(2022 DSP → 2024 DSPy ICLR Oral)
  2. **Prompt-as-program 范式**(Signature + Module + Parameter 三位一体)— **不是 prompt-engineering,是 prompt-programming**
  3. **15 个 Optimizer 算法**(业内最多)— GEPA / MIPROv2 / COPRO / SIMBA / Ensemble / Avatar / BetterTogether ...
  4. **13 个 Predict Module**(ReAct / CoT / Refine / Retry / BestOfN / CodeAct / PoT / MultiChainComparison / Parallel)— composable
  5. **不是 platform / benchmark,是 framework** — 跟 4 家 eval(Langfuse / Opik / SWE-bench / tau-bench)都不同 layer
  6. **`UNSAFE_LM_STATE_KEYS` 防 prompt injection**(`predict.py:22`)— 业内独家工程细节
  7. **Adapter pattern**(Chat / JSON / XML / TwoStep)+ typed I/O(Image / Audio / File / Code / Reasoning)— 多模态 + 多协议统一抽象

  ## 优势 (strengths)
  1. **`compile` 语法**:`compiled = optimizer.compile(student=program, trainset=...)`— prompt 优化跟编译类比,直觉
  2. **GEPA 2025-07 paper(`arxiv.org/abs/2507.19457`)**:"Reflective Prompt Evolution Can Outperform Reinforcement Learning"— DSPy 自家发表
  3. **MIPROv2 2024-06 paper**(`arxiv.org/abs/2406.11695`)+ DSPy Assertions 2023-12(`arxiv.org/abs/2312.13382`)— 学术 backing 完整
  4. **`DSPY_CACHE` 全局 LRU**(`clients/`)— reruns 大幅省钱
  5. **LiteLLM 多 provider**(50+ provider)
  6. **`bootstrap_trace_data`** — 把执行 trace 作为 optimizer 输入,**自动 bootstrap labeled data**
  7. **`SandboxSerializable`**(`primitives/sandbox_serializable.py`)— sandbox 序列化,跑 untrusted code 安全
  8. **`code_interpreter.py` + `python_interpreter.py`** — Python REPL 工具(类 Codex Code Mode 但更简化)

  ## 哪些点做得好 (what_done_well)
  1. **`Predict(Module, Parameter)` 多重继承**(`predict/predict.py:46`)— Module 是组合单元,Parameter 是 optimizer 看到的 unit,**一个类两个 role 漂亮**
  2. **`Signature` 用 docstring + InputField / OutputField 写**(类似 dataclass)— Pythonic
  3. **`UNSAFE_LM_STATE_KEYS = {"api_base", "base_url", "model_list"}`** 用 set + 默认 sanitize + warning + `allow_unsafe_lm_state=True` 显式开关 — **secure by default + opt-out for trusted**
  4. **`from dspy.predict import *` + `from dspy.teleprompt import *`** 顶层 `__init__` 全 re-export — 用户 `import dspy` 就拿到全部 API
  5. **GEPA 子模块**(`teleprompt/gepa/`)— 一个算法一个子目录,工程清晰
  6. **`BootstrapRS = BootstrapFewShotWithRandomSearch`**(`__init__.py`)— alias 短名给老用户兼容
  7. **`Adapter` 把 LM I/O 协议层 abstract**(JSON / XML / TwoStep) — 业内最早做 I/O adapter 的 framework 之一
  8. **`callbacks` instrumentation hook**(类 Langchain callback) — 业内统一模式
---

# DSPy(Stanford NLP, MIT, ICLR 2024 Oral)

## 一句话定位

**Prompt-as-program framework**:把 LLM 当成可编程模块,prompt + few-shot + 模型选择作为可优化 hyperparameter,用 15 种 optimizer(teleprompter)算法自动改进。"Compiling Declarative Language Model Calls into Self-Improving Pipelines"。

## 起源
- **2022-12** DSP paper(`arxiv.org/abs/2212.14024`)— Demonstrate-Search-Predict
- **2023-10** DSPy ICLR 2024 Oral(`arxiv.org/abs/2310.03714`)
- **2023-12** DSPy Assertions(`arxiv.org/abs/2312.13382`)
- **2024-06** MIPROv2(`arxiv.org/abs/2406.11695`)
- **2025-07** **GEPA: Reflective Prompt Evolution Can Outperform RL**(`arxiv.org/abs/2507.19457`)— DSPy 自家发表

## 核心抽象:Signature + Module + Parameter

```python
import dspy

# Signature 描述 task,不写 prompt
class QASignature(dspy.Signature):
    """Answer questions concisely."""
    context: str = dspy.InputField()
    question: str = dspy.InputField()
    answer: str = dspy.OutputField()

# Module 是可组合单元
qa = dspy.ChainOfThought(QASignature)

# Optimizer compile
optimizer = dspy.teleprompt.MIPROv2(metric=accuracy_metric)
compiled_qa = optimizer.compile(student=qa, trainset=trainset)

# 用
result = compiled_qa(context=ctx, question=q)
print(result.answer)
```

## 15 Optimizer 跟 Opik 7 算法对照

详见 frontmatter `scorer`+ source 锚点。**Opik 7 算法基本是 DSPy 15 算法的子集**:
- Opik `GepaOptimizer` = DSPy GEPA(2025-07)
- Opik `FewShotBayesianOptimizer` = DSPy `BootstrapFewShotWithOptuna`
- Opik `MetaPromptOptimizer` ≈ DSPy `COPRO`
- Opik `EvolutionaryOptimizer` ≈ DSPy GEPA / SIMBA

**Opik 的工程化加值**:`OptimizationResult` 标准化对象 + Optimizer Chaining + Counter Tracking + 8 内置 benchmark dataset。**Opik = DSPy 学术成果的 production 工程化包装**。

## SkillForge 校准

详见 frontmatter `skillforge_takeaway`。最核心 3 条:

1. **`UNSAFE_LM_STATE_KEYS` 防 prompt injection** — 加载用户上传 prompt / agent state 时 sanitize,**业内独家工程细节,SkillForge 应学**
2. **Signature 抽象** — SkillForge prompt template 渐进升级为 typed program(中优先级)
3. **MIPROv2 + GEPA 2 算法 POC** — 对照 R10 决策(`/evolve` + Opik runtime Optimizer 互补),DSPy 给 2 个具体算法 reference

## 待回答
- DSPy `code_interpreter.py` vs Codex Code Mode V8 isolate 对比(都让 LLM 跑代码)
- `bootstrap_trace_data` 跟 SkillForge production trace → dataset 自动归集对比
- `dspy.Example` schema 跟 [[../eval/langfuse]] DatasetItem 对比
- DSPy `Assertion` 跟 Opik Assertion(`AssertionResultDAO.java`)对比
- GEPA paper 具体算法步骤(Genetic + Pareto front 怎么用)
