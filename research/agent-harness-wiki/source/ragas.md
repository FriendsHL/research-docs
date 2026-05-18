---
type: source
name: ragas
origin: "/Users/youren/myspace/ragas/ (explodinggradients, Apache 2.0)"
ingested_at: 2026-05-16
ingested_commit: 298b682
ingested_commit_date: 2026-02-24
ingested_version_label: "(动态 version via setuptools-scm,以 commit 为准)"
contributed_to:
  - eval/ragas.md
  - concept/agent-eval.md
  - concept/rag-evaluation.md
---

# Source — Ragas

> **业内 RAG eval 标杆**(Apache 2.0,explodinggradients)。**27+ 内置 metric**(answer correctness / faithfulness / context recall / precision / hallucination 等)+ 多 LLM 集成 + **被 Langfuse / Opik 都集成**。

## 仓库结构概览

```
ragas/
├── src/ragas/
│   ├── __init__.py                  # 入口
│   ├── _analytics.py                # 匿名 telemetry
│   ├── async_utils.py
│   ├── backends/                    # 持久化后端
│   ├── cache.py                     # eval cache
│   ├── callbacks.py                 # 回调
│   ├── cli.py                       # CLI
│   ├── config.py
│   ├── cost.py                      # 成本跟踪
│   ├── dataset.py + dataset_schema.py  # Dataset(EvaluationDataset / SingleTurnSample / MultiTurnSample)
│   ├── embeddings/                  # Embedding wrapper(OpenAI / HuggingFace / ...)
│   ├── evaluation.py                # ⭐ 主入口
│   ├── exceptions.py
│   ├── executor.py                  # Async 并发 Executor
│   ├── experiment.py                # Experiment management
│   ├── integrations/                # 11 个集成(见下)
│   ├── llms/                        # LLM wrapper(LangChain-based)
│   ├── losses.py
│   ├── messages.py
│   ├── metrics/                     # 27 个 metric(见下)
│   ├── optimizers/                  # ⭐ Prompt optimization(DSPy + Genetic)
│   ├── prompt/                      # Prompt templating
│   └── run_config.py
├── docs/                            # MkDocs 文档站
├── examples/
├── tests/
├── pyproject.toml                   # setuptools-scm 动态 version
└── Makefile
```

## 关键 file:line 锚点

### 27 个内置 Metric(`src/ragas/metrics/`)

| 类别 | Metric |
|---|---|
| **Answer 维度** | `_answer_correctness.py` / `_answer_relevance.py` / `_answer_similarity.py` |
| **Context 维度(RAG 核心三角)** | `_context_precision.py` / `_context_recall.py` / `_context_entities_recall.py` |
| **Faithfulness(RAG 核心)** | `_faithfulness.py` / **`_multi_modal_faithfulness.py`** / `_noise_sensitivity.py` |
| **Multi-modal RAG** | `_multi_modal_faithfulness.py` / `_multi_modal_relevance.py` |
| **NLP metric** | `_bleu_score.py` / `_rouge_score.py` / `_chrf_score.py` / `_string.py` |
| **Factuality** | `_factual_correctness.py` |
| **Aspect-based** | `_aspect_critic.py` / `_domain_specific_rubrics.py` / `_instance_specific_rubrics.py` / `_simple_criteria.py` |
| **Summarization** | `_summarization.py` |
| **Agent metric(non-RAG)** | `_goal_accuracy.py` / `_topic_adherence.py` / `_tool_call_accuracy.py` / `_tool_call_f1.py` |
| **SQL** | `_sql_semantic_equivalence.py` |
| **DataCompy** | `_datacompy_score.py`(dataframe 对比) |
| **NVIDIA proprietary** | `_nv_metrics.py`(NVIDIA NeMo 专用) |

**RAG 核心三角**(`evaluation.py` 默认推荐):
1. **Faithfulness** — 答案是否基于检索 context
2. **Context Recall** — 检索 context 是否覆盖 ground truth
3. **Context Precision** — 检索 context 是否精准(无 noise)

### Multi-Turn / Single-Turn 双模式

`dataset_schema.py`:
```python
SingleTurnSample      # 单轮 RAG
MultiTurnSample       # Multi-turn agent(对照 [[tau-bench]])
```

`MetricWithLLM` / `SingleTurnMetric` / `MultiTurnMetric` 抽象基类(`metrics/base.py`):
```python
class Metric(...)
class MetricWithLLM(Metric)
class MetricWithEmbeddings(Metric)
class SingleTurnMetric(...)
class MultiTurnMetric(...)
class ModeMetric(...)
```

### 11 个 Integration(`src/ragas/integrations/`)— 业内最广

```
ag_ui.py              # AG-UI(generative UI 框架)
amazon_bedrock.py     # AWS Bedrock
griptape.py           # Griptape framework
helicone.py           # ← 被 evaluation.py:25 直接 import
langchain.py          # LangChain
langgraph.py          # LangGraph
langsmith.py          # LangSmith
llama_index.py        # LlamaIndex
opik.py               # ← Opik 集成(被 [[../eval/opik]] 反向集成)
r2r.py                # R2R RAG framework
swarm.py              # OpenAI Swarm
tracing/              # tracing 子模块
   ├── langfuse.py    # Langfuse tracing
   └── mlflow.py      # MLflow tracing
```

**11 个 integration 等于 11 家平台都集成 Ragas**:**业内 RAG eval 标杆地位坐实**。

### `evaluation.py:25` Helicone 是 first-class

```python
from ragas.integrations.helicone import helicone_config
```

主 `evaluate()` 函数直接 import Helicone config — Ragas + Helicone 紧密集成,**Helicone 是 LLM 调用 proxy,Ragas eval 自动走 Helicone 跟踪 token cost**。

### Optimizers(`src/ragas/optimizers/`)

```
__init__.py
base.py                    # BaseOptimizer
dspy_adapter.py           # ⭐ DSPy adapter(对照 [[dspy]])
dspy_llm_wrapper.py       # DSPy LLM wrapper
dspy_optimizer.py         # DSPy 优化器
genetic.py                # 遗传算法
utils.py
```

**Ragas 集成 DSPy + 自家 Genetic optimizer 优化 prompt**:**比 Opik Agent Optimizer 早做 prompt optimization** 在 RAG eval 场景。

### `evaluation.py` 主入口签名

```python
def evaluate(
    dataset: t.Union[EvaluationDataset, Dataset],
    metrics: list[Metric] | None = None,
    llm: BaseRagasLLM | LangchainLLM | None = None,
    embeddings: BaseRagasEmbeddings | LangchainEmbeddings | None = None,
    callbacks: Callbacks = None,
    in_ci: bool = False,
    run_config: RunConfig | None = None,
    token_usage_parser: TokenUsageParser | None = None,
    raise_exceptions: bool = False,
    column_map: dict[str, str] | None = None,
    show_progress: bool = True,
    batch_size: int | None = None,
    experiment_name: str | None = None,
    use_v1_metrics: bool = False,
) -> EvaluationResult
```

**关键参数**:`in_ci` 给 CI 模式优化(reduced output / non-blocking)— 工程细节漂亮。

### `executor.py` Async 并发(Ragas 性能关键)

跟 [[../eval/langfuse]] worker queue + [[promptfoo]] 并发模式不同 — Ragas 用 Python async + `tqdm` progress bar。

### `_aspect_critic.py` Aspect Critic(Ragas 独家)

**LLM 当 aspect critic**:给一个 aspect description(如 "tone is professional"),LLM 输出 yes/no/maybe + reason — **对照 [[promptfoo]] `llmRubric`,但 Ragas 加了 multi-aspect 批量评测**。

### Multi-Modal RAG metric(独家)

- `_multi_modal_faithfulness.py`
- `_multi_modal_relevance.py`

**业内最早把 multi-modal RAG eval 做成 first-class**(图片 + 文本 RAG)。

## 与 wiki 现有

### 印证
- **Langfuse `metrics/ragas_metric.py` 集成** + **Opik 内置 Ragas integration**(`web/src/features/evals/components/ragas-logo.tsx`)— 印证 Ragas 是 RAG eval 行业标杆
- **DSPy adapter**(`optimizers/dspy_adapter.py`)印证 Ragas + DSPy 互通
- **LangChain / LangSmith 集成** 印证 LangChain 生态主流

### 矛盾
- **Ragas Optimizer**(DSPy + Genetic)跟 [[../eval/opik]] Optimizer 7 算法 / [[dspy]] 15 算法**部分重叠**:
  - Ragas `dspy_optimizer.py` = DSPy 优化器的 RAG-focused 包装
  - Ragas `genetic.py` = 跟 GEPA / Opik EvolutionaryOptimizer 同 family
  - Ragas optimization 主要给 metric 阈值调优,不是 prompt evolution(场景不同)

### 新增
- **`eval/ragas.md`**(新建)— RAG eval 专属页
- **扩 [[../concept/agent-eval]]**:加 Ragas "RAG eval 三角"(faithfulness / context recall / context precision)+ Multi-turn metric(对照 SingleTurnSample / MultiTurnSample)
- **(可选)新建 `concept/rag-evaluation.md`**:RAG eval 设计空间(query / retrieval / generation 三段)

## 事实漂移 / 注意点

- **2026-02-24 commit** 比其他项目稍旧(SWE-bench / tau-bench 2026-03,DSPy / Promptfoo / Langfuse 2026-05)— Ragas 节奏稍慢但稳定
- **`v1` vs `v2` metric**(`use_v1_metrics: bool = False` 在 `evaluate()`)— 内部有版本兼容
- **依赖 `langchain_core`** + `datasets` HuggingFace — 比较重的 Python 生态依赖
- **`_analytics.py` track_was_completed** 匿名 telemetry — 用户可关
- **没有 web UI / dashboard** — 纯 Python library(对照 [[../eval/langfuse]] / [[../eval/opik]])
- **Multi-modal RAG 是独家方向**(`_multi_modal_*.py`)— SkillForge MULTIMODAL-MVP(2026-05-14 已交付)可对照
- **Helicone 直接 import**(`evaluation.py:25`)— 默认 LLM 调用经过 Helicone proxy(用户配 API key 时)
