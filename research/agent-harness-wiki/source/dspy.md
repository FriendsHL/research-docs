---
type: source
name: dspy
origin: "/Users/youren/myspace/dspy/ (Stanford NLP, MIT)"
ingested_at: 2026-05-16
ingested_commit: 621c3a6
ingested_commit_date: 2026-05-15
ingested_version_label: "3.2.1(`dspy/__metadata__.py:__version__`)"
contributed_to:
  - eval/dspy.md
  - concept/prompt-optimization.md
  - concept/prompt-as-program.md
---

# Source — DSPy

> **Stanford NLP 出品**(MIT,Omar Khattab et al.)— "**Compiling Declarative Language Model Calls into Self-Improving Pipelines**"(ICLR 2024)。**LLM 当成可编程模块,把 prompt + few-shot + 模型选择当成可优化 hyperparameter**。15+ teleprompter(optimizer)+ 13+ predict module + Signature 抽象。

## 仓库结构概览

```
dspy/
├── dspy/                            # 主包
│   ├── __init__.py                  # 入口(re-export 所有 module)
│   ├── __metadata__.py              # __version__="3.2.1"
│   ├── adapters/                    # LM I/O 适配:ChatAdapter / JSONAdapter / XMLAdapter / TwoStepAdapter / Image / Audio / File / History / Tool / ToolCalls / Code / Reasoning
│   ├── clients/                     # LM client(LiteLLM-based + cache)
│   ├── datasets/                    # 内置 dataset
│   ├── dsp/                         # 旧 DSP(legacy)+ utils/settings
│   ├── evaluate/                    # Evaluate 框架(metric + DataFrame eval)
│   ├── experimental/                # 实验性 feature
│   ├── predict/                     # 13 predict module(见下)
│   ├── primitives/                  # base_module / Example / Module / Prediction / code_interpreter / python_interpreter / sandbox
│   ├── propose/                     # Prompt proposal(GEPA 用)
│   ├── retrievers/                  # Retriever(BM25 / ColBERTv2 / 其他)
│   ├── signatures/                  # Signature / field / utils — DSPy 核心抽象
│   ├── streaming/                   # streamify
│   ├── teleprompt/                  # 15 optimizers(见下)
│   └── utils/                       # asyncify / syncify / callback / saving / usage_tracker / logging
├── docs/                            # 文档站
├── tests/                           # 测试
├── pyproject.toml                   # PyPI 包定义
└── uv.lock                          # uv 锁
```

## 关键 file:line 锚点

### 核心抽象:Signature(`signatures/signature.py`)

DSPy 的**核心创新** — 把 LLM 调用从 prompt-engineering 抽象成 **typed input/output declaration**:

```python
# Signature 表达"任务签名"
class QASignature(dspy.Signature):
    """Answer questions concisely based on the context."""
    context: str = dspy.InputField()
    question: str = dspy.InputField()
    answer: str = dspy.OutputField()
```

不写 prompt,描述 task → DSPy 自动生成 prompt + few-shot examples + 输出 parser。

### 13 个 Predict Modules(`dspy/predict/`)

```
__init__.py
aggregation.py
avatar/                          # Avatar(agent-like)
best_of_n.py                     # Best-of-N sampling
chain_of_thought.py              # CoT
code_act.py                      # Code-Act(LLM 写代码)
knn.py                           # KNN-based prediction
multi_chain_comparison.py        # Multi-chain 对比
parallel.py                      # Parallel execution
parameter.py                     # Parameter abstraction(给 optimizer)
predict.py                       # Basic Predict module(`Predict(Module, Parameter):`)
program_of_thought.py            # PoT(LLM 写程序)
react.py                         # ReAct(`arxiv.org/abs/2210.03629`)
refine.py                        # Refine(iterative self-correction)
retry.py                         # Retry on failure
rlm.py                           # RLM?(retriever-augmented LM)
```

每个 module 都继承 `Module` + `Parameter` — **Parameter** 是 optimizer 看到的 unit。

### 15 个 Optimizer(`dspy/teleprompt/__init__.py:1-18`)

**"Teleprompter" 是 DSPy 对 optimizer 的命名**:

```python
__all__ = [
    "AvatarOptimizer",                       # Avatar agent 优化
    "BetterTogether",                        # 多 optimizer 协作
    "BootstrapFewShot",                      # Bootstrap few-shot(基础)
    "BootstrapFinetune",                     # Bootstrap + fine-tune
    "COPRO",                                 # Coordinate Prompt Optimization
    "Ensemble",                              # 多 program ensemble
    "GEPA",                                  # ⭐ Genetic-Pareto(Jul'25 paper)
    "KNNFewShot",                            # KNN-based few-shot 选择
    "MIPROv2",                               # ⭐ MIPRO v2(`arxiv.org/abs/2406.11695`)
    "BootstrapFewShotWithRandomSearch",      # Random search
    "BootstrapFewShotWithOptuna",            # Optuna(Bayesian)
    "LabeledFewShot",                        # 简单 labeled few-shot
    "InferRules",                            # Rule inference
    "SIMBA",                                 # SIMBA optimizer
    "bootstrap_trace_data",                  # Trace data bootstrap
]
```

跟 [[../eval/opik]] Opik Optimizer 7 算法对照:

| 维度 | DSPy(15) | Opik(7) |
|---|---|---|
| 算法数 | **15**(更多) | 7 |
| EvolutionaryOptimizer 等价 | **GEPA**(Genetic-Pareto,2025-07 paper) | EvolutionaryOptimizer |
| FewShotBayesianOptimizer 等价 | **BootstrapFewShotWithOptuna** | FewShotBayesianOptimizer |
| MetaPromptOptimizer 等价 | **COPRO**(Coordinate Prompt Optimization)| MetaPromptOptimizer |
| Bayesian instruction 等价 | **MIPROv2** | (无显式等价) |
| 完整 codebase | Stanford,2022 起源 | 2024 后期 |
| Ecosystem 成熟度 | **业内最成熟 prompt optimization 框架** | 工程化更深(标准化 OptimizationResult / Chaining) |

### 关键论文(README:Citations)

- **[Jul'25] GEPA**: Reflective Prompt Evolution Can Outperform RL(`arxiv.org/abs/2507.19457`)— DSPy 自家 GEPA optimizer 的论文
- **[Jun'24] MIPROv2**(`arxiv.org/abs/2406.11695`)
- **[Oct'23] DSPy: Compiling Declarative LM Calls**(`arxiv.org/abs/2310.03714`,ICLR 2024)
- **[Dec'23] DSPy Assertions**(`arxiv.org/abs/2312.13382`)— 计算约束
- **[Dec'22] DSP**(`arxiv.org/abs/2212.14024`)— Demonstrate-Search-Predict(DSPy 前身)

### Adapter 系统(`dspy/adapters/`)

LM I/O 桥接层:
- **ChatAdapter**(主)— OpenAI chat completion 风格
- **JSONAdapter** — 强制 JSON 输出
- **XMLAdapter** — XML 输出
- **TwoStepAdapter** — 两步生成(先 plan 再 act)
- **typed I/O**: Image / Audio / File / History / Tool / ToolCalls / Code / Reasoning

### Predict 基础类(`dspy/predict/predict.py:46-50`)

```python
class Predict(Module, Parameter):
    """Basic DSPy module that maps inputs to outputs using a language model.

    Args:
        signature: The input/output signature describing the task.
        callbacks: Optional list of callbacks for instrumentation.
        **config: Default keyword arguments forwarded to the underlying LM.
    """
```

**`Predict(Module, Parameter)` 多重继承** — `Module` 是组合单元,`Parameter` 是 optimizer 看到的可优化 unit。

### `UNSAFE_LM_STATE_KEYS`(`predict.py:22`)

```python
UNSAFE_LM_STATE_KEYS = {"api_base", "base_url", "model_list"}
```

**`_sanitize_lm_state`**(`:25-43`)默认从 saved state 移除 api_base / base_url / model_list,防 LM state 反序列化时偷换 API endpoint(prompt injection 防御)。

### 完整 ecosystem 组件

- **`dspy.clients`** — LM client(LiteLLM-based,`DSPY_CACHE` 全局缓存)
- **`dspy.streaming.streamify`** — 流式 wrapper
- **`dspy.utils.usage_tracker.track_usage`** — token usage 跟踪
- **`dspy.utils.asyncify / syncify`** — async/sync 互转
- **`dspy.utils.saving.load`** — 序列化(警告 UNSAFE_LM_STATE_KEYS)
- **`dspy.cache = DSPY_CACHE`** — 全局 LRU cache
- **`SandboxSerializable`** — sandbox 序列化(用户跑 untrusted code 时)
- **`code_interpreter.py` + `python_interpreter.py`** — Python REPL 工具

## 与 wiki 现有

### 印证
- **GEPA**(`dspy/teleprompt/gepa/`)印证 [[../eval/opik]] `GepaOptimizer` 是 GEPA paper(`arxiv.org/abs/2507.19457`,**Jul'25 DSPy 自家发表**)
- **BootstrapFewShotWithOptuna** 印证 [[../eval/opik]] `FewShotBayesianOptimizer` 同 Bayesian 路线
- **LiteLLM 多 provider** 印证 [[../eval/opik]] / [[tau-bench]] 都用 litellm
- **Pydantic typed I/O** 印证业内 schema-first 路线

### 矛盾
- **DSPy 算法 15 个 vs Opik 7 个** — Opik 是 DSPy 的"工程化标准化版本"(Opik OptimizationResult 标准化对象 + Optimizer Chaining 是 DSPy 没有的工程产物)
- **DSPy 是 framework**(用户 import dspy 写代码)vs **Opik 是 platform**(用户在 UI 跑 + dashboard)— **不同 layer**

### 新增
- **`eval/dspy.md`**(新建)— framework 类(对比 [[../eval/opik]] / [[../eval/langfuse]] platform 类)
- **扩 [[../concept/prompt-optimization]]** — 加 DSPy 15 算法 + Opik 7 算法完整对照表
- **(可选)新建 `concept/prompt-as-program.md`** — Stanford 提出的"prompt = program"范式(Signature + Module + Parameter 三位一体),对比传统 prompt-as-string

## 事实漂移 / 注意点

- **DSPy 3.2.1**(`__metadata__.py`)— 跟 [[../eval/opik]] 1.7.0 / [[../eval/langfuse]] v3.174.1 同时期成熟
- **不是 platform** — DSPy 是 Python framework,**没有 web UI / dashboard / multi-tenant**
- **跟 Opik Optimizer 关系**:
  - Opik `GepaOptimizer` = DSPy GEPA 算法 + Opik 工程化包装
  - Opik 7 算法是"DSPy 15 算法的子集 + 工程化"
  - **DSPy 是 prompt optimization 学术研究产物,Opik 是 production 工程化产物**
- **`UNSAFE_LM_STATE_KEYS`** 防 prompt injection 工程细节业内独家
- **Code-Act + Program-of-Thought + Code Interpreter** 集成 — DSPy 把 "LLM 写代码" 当 first-class predict module
