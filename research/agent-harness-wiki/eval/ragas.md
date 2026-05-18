---
type: eval-framework
name: ragas
status: active
language: python

# === 版本时效 ===
last_ingested_version: "ragas @ commit 298b682 (2026-02-24,setuptools-scm 动态 version)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - 2026-02-24 之后的新 metric / integration(2-3 个月未跟进)

sources:
  - "[[../source/ragas]]"

# === 9 维度 ===
target: |
  **业内 RAG eval 标杆**(Apache 2.0,explodinggradients)。专门评测 RAG(Retrieval-Augmented Generation)+ LLM-as-judge metric library。**27+ 内置 metric** + **11+ 平台集成**(Langfuse / Opik / Helicone / LangChain / LangSmith / LlamaIndex / Bedrock / LangGraph / Griptape / OpenAI Swarm / R2R / AG-UI / MLflow)。

  跟其他 eval 关系:
  - [[../eval/langfuse]] / [[../eval/opik]] 都内置 Ragas 集成(`ragas_metric.py` / `ragas-logo.tsx`)
  - **Ragas 是 metric library + Python evaluator,不是 platform / harness**
  - 用户场景:RAG 系统(知识库 QA / 文档问答)的离线 eval

  跟 [[promptfoo]] 关系:Promptfoo 50+ assertion 中 `answerRelevance / contextFaithfulness / contextRecall / contextRelevance` 4 个就是 Ragas 风格(Promptfoo 是 CLI,Ragas 是 Python library)
  [verified-from-source]

dataset: |
  ### Dataset Schema(`src/ragas/dataset_schema.py`)
  - **`EvaluationDataset`** — 评测数据集
  - **`SingleTurnSample`** — 单轮 RAG sample(question + answer + retrieved_contexts + ground_truth)
  - **`MultiTurnSample`** — 多轮 agent sample(对照 [[tau-bench]] multi-turn)
  - 兼容 HuggingFace `datasets.Dataset`
  - **`use_v1_metrics: bool = False`**(`evaluation.py`)— 内部 v1/v2 兼容

  ### 数据流
  ```
  HuggingFace Dataset / 用户 list[dict]
       ↓
  EvaluationDataset(SingleTurnSample[] 或 MultiTurnSample[])
       ↓
  evaluate(dataset, metrics=[...], llm=..., embeddings=...)
       ↓
  EvaluationResult(每 sample × 每 metric 一个 score)
  ```
  [verified-from-source]

scorer: |
  **27+ 内置 metric**,详见 [[../source/ragas]] 完整列表。

  ### RAG 核心三角(必跑)
  1. **Faithfulness**(`_faithfulness.py`)— 答案是否基于检索 context(防 hallucination)
  2. **Context Recall**(`_context_recall.py`)— 检索 context 是否覆盖 ground truth
  3. **Context Precision**(`_context_precision.py`)— 检索 context 是否精准(无 noise)

  ### Answer 维度 3 metric
  - **Answer Correctness**(`_answer_correctness.py`)
  - **Answer Relevance**(`_answer_relevance.py`)
  - **Answer Similarity**(`_answer_similarity.py`)

  ### Multi-Modal RAG(独家)
  - **Multi-Modal Faithfulness**(`_multi_modal_faithfulness.py`)
  - **Multi-Modal Relevance**(`_multi_modal_relevance.py`)
  - **业内最早把 multi-modal RAG eval first-class**(对照 SkillForge MULTIMODAL-MVP 2026-05-14 已交付)

  ### Agent metric(非 RAG,4 个)
  - **Goal Accuracy**(`_goal_accuracy.py`)
  - **Topic Adherence**(`_topic_adherence.py`)
  - **Tool Call Accuracy**(`_tool_call_accuracy.py`)
  - **Tool Call F1**(`_tool_call_f1.py`)

  ### LLM-as-judge 框架(Aspect Critic)
  **`_aspect_critic.py`** — 给一个 aspect description(如 "tone is professional"),LLM 输出 yes/no/maybe + reason。**对照 [[promptfoo]] `llmRubric` / [[../eval/opik]] LLM-as-judge metric**

  ### Metric 抽象层级(`metrics/base.py`)
  ```python
  class Metric                  # 基类
  class MetricWithLLM(Metric)   # 需要 LLM
  class MetricWithEmbeddings    # 需要 embedding
  class SingleTurnMetric        # 单轮
  class MultiTurnMetric         # 多轮(agent)
  class ModeMetric              # 模式 metric
  ```
  [verified-from-source]

reproducibility: |
  - **`run_config: RunConfig`** 参数控制并发 / 重试 / timeout
  - **`batch_size: int | None`** 批量
  - **`cache.py`** eval cache
  - **`callbacks` + `RagasTracer`** 全 trace 可序列化
  - **`token_usage_parser`** 显式 token 用量解析(`evaluation.py`)
  - 没有 Docker 容器化
  [verified-from-source]

coverage: |
  ### 覆盖
  - **RAG eval 全栈**(retrieve / generate / multi-modal / agentic)
  - **27 metric** 覆盖 answer / context / faithfulness / NLP / factuality / aspect / agent / SQL / data
  - **11+ 平台集成** + 2 tracing(langfuse / mlflow)
  - **Optimizer**(DSPy adapter + Genetic)优化 metric prompt
  - **CI/CD-friendly**(`in_ci=True` 模式)

  ### 不覆盖
  - 没有 web UI / dashboard
  - 没有 production trace 自动归集
  - 没有 multi-tenant 隔离
  - 没有 platform 化 dataset 管理(Dataset 是 in-memory)
  - **不做 red teaming**(对照 [[promptfoo]] 25 攻击策略)
  [verified-from-source]

production_loop: |
  **Library + Notebook + CI**,不做 production loop。

  Production 闭环靠**集成方**:
  - 用户在 Langfuse 用 `metrics/ragas_metric.py` 跑 Ragas(Langfuse 提供 platform loop)
  - 用户在 Opik 跑 Ragas integration(Opik 提供 platform loop)
  - 用户在 LangSmith 跑(LangSmith 提供)

  Ragas 自己只负责 **metric 实现 + evaluator runtime**。
  [verified-from-source]

automation: |
  - **`evaluate()` 函数式 API** + Python notebook 主用
  - **CLI**(`src/ragas/cli.py`)— 简单 CLI 包装
  - **`in_ci=True`** 给 CI 模式(reduced output / non-blocking)
  - **Async executor + tqdm progress bar**(`executor.py`)
  - **`raise_exceptions: bool = False`** 默认不抛 exception(CI 友好)
  [verified-from-source]

cost: |
  - **`cost.py`** + **`token_usage_parser`**(`evaluation.py`)— token cost 跟踪
  - **`evaluation.py:25 from ragas.integrations.helicone import helicone_config`** — 默认走 Helicone proxy(用户配 API key 时)— **业内独家直接集成 Helicone**
  - Per-metric LLM 调用成本:**RAG 三角 = 3 LLM 调用 per sample**(每 100 sample × 3 metric ≈ $5-15 GPT-4)
  - `cache.py` 缓存重复 eval
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Ragas 是 **SkillForge RAG eval scenario 的 metric library**

  SkillForge 当前:
  - **EVAL-V2** 4 维分数 + dimensionStatus + Scenario / EvalRun / EvalTask
  - **MULTIMODAL-MVP**(2026-05-14)— 多模态输入 Phase 1
  - **SKILL-AB-MULTITURN-FIX**(2026-05-13)— 多轮 evaluation

  Ragas 提供 **27 个 ready-made metric**,SkillForge **不需要自己实现 RAG metric**,直接 wrap Ragas。

  ## 可借鉴(给 SkillForge EvalRun 集成)

  ### 必做(短期 1-2 周)
  1. **集成 RAG 核心三角**(Faithfulness + Context Recall + Context Precision)— SkillForge 如果用户有 RAG agent,**必须支持这 3 个 metric**(对照 Langfuse / Opik 都集成)
  2. **`SingleTurnSample` / `MultiTurnSample` Schema 对齐** — SkillForge EvalTask schema 应区分 single-turn / multi-turn,跟 Ragas Sample 兼容(便于 metric reuse)
  3. **`AspectCritic` LLM-as-judge 框架** — SkillForge LLM-as-judge metric 可借鉴 aspect description + yes/no/maybe + reason 三段输出

  ### 推荐(中期 1 个月)
  4. **Tool Call Accuracy / Tool Call F1**(`_tool_call_accuracy.py` / `_tool_call_f1.py`)— SkillForge agent eval 应有 tool 调用准确率 metric
  5. **`Goal Accuracy` + `Topic Adherence`** agent metric — 对照 SkillForge agent eval
  6. **Multi-Modal Faithfulness / Relevance**(对照 SkillForge MULTIMODAL-MVP 多模态输入)

  ### 工程细节(可借鉴)
  7. **`MetricWithLLM` + `MetricWithEmbeddings` + `SingleTurnMetric` + `MultiTurnMetric`** 多重继承抽象(`metrics/base.py`)— SkillForge Java metric 可类似 interface 分层(SingleTurnMetric + MultiTurnMetric + LLM-required + Embedding-required)
  8. **`in_ci: bool = False` 参数** — SkillForge CI 模式可加(reduced output / non-blocking exception)
  9. **`token_usage_parser`** 跟踪 token cost — SkillForge metric LLM 调用应跟踪 token

  ### 不必跟
  - **DSPy adapter + Genetic optimizer**(`optimizers/`)— SkillForge 走自家 PromptImproverService 路线
  - **`_nv_metrics.py`** NVIDIA 专用 — 不通用
  - **`_datacompy_score.py`** dataframe 对比 — 场景不通用

  ## 关键决策建议

  **如果 SkillForge 用户有 RAG agent / 知识库问答场景**:
  - **集成 Ragas Python library 作为 SkillForge metric backend**(类 Langfuse `metrics/ragas_metric.py` 集成方式)
  - 在 SkillForge Scenario YAML 配:
    ```yaml
    metrics:
      - type: ragas.faithfulness
      - type: ragas.context_recall
      - type: ragas.context_precision
    ```
  - SkillForge backend 调 Ragas Python(可能需 Python sidecar service,对照 Opik `apps/opik-python-backend/` + `opik-sandbox-executor-python/`)

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **业内 RAG eval 标杆**(2023+ 起源,被 Langfuse / Opik / Helicone / LangChain / LangSmith / LlamaIndex / Bedrock / LangGraph / Griptape / Swarm / R2R / AG-UI / MLflow 13+ 平台集成)
  2. **27+ 内置 metric**(业内最广 RAG metric 集)
  3. **RAG 核心三角**(Faithfulness / Context Recall / Context Precision)— 业内事实标准
  4. **Multi-Modal RAG metric** 独家(`_multi_modal_*.py`)— 图片 + 文本 RAG
  5. **`AspectCritic` LLM-as-judge 框架** — 给 aspect description → yes/no/maybe + reason
  6. **`evaluation.py:25` Helicone 直接 import** — 业内独家 first-class proxy 集成
  7. **DSPy + Genetic optimizer**(`optimizers/`)— RAG metric 阈值优化
  8. **Single-Turn / Multi-Turn Sample 双 schema**(`dataset_schema.py`)— 同框架覆盖 RAG + agent
  9. **Apache 2.0 license** + 完全开源,无 SaaS 版本

  ## 优势 (strengths)
  1. **被 13+ 平台集成** — 业内 RAG eval 事实接口
  2. **`MetricWithLLM` + `MetricWithEmbeddings` 多重继承** — metric 抽象层级清晰
  3. **`SingleTurnMetric` + `MultiTurnMetric` 区分** — single-turn / multi-turn 同一框架
  4. **`in_ci: bool`** + **`raise_exceptions: bool = False`** — CI 友好的默认
  5. **`callbacks` + `RagasTracer`** 全 trace 可观测
  6. **`async + tqdm`** progress 体验
  7. **`v1 → v2` 兼容**(`use_v1_metrics`)— API 演化不 break user
  8. **`cost.py` + `token_usage_parser`** — token cost 一等公民
  9. **`r2r.py` integration**(`integrations/r2r.py`)— R2R RAG framework 集成
  10. **`swarm.py` integration** — OpenAI Swarm multi-agent 集成

  ## 哪些点做得好 (what_done_well)
  1. **每个 metric 一个文件**(`_answer_correctness.py` / `_faithfulness.py` / ...)+ 下划线前缀(私有)— 工程清晰
  2. **`metrics/base.py` 6 个抽象基类**(Metric / MetricWithLLM / MetricWithEmbeddings / SingleTurnMetric / MultiTurnMetric / ModeMetric)— 多维度组合
  3. **`evaluation.py` 顶层 `evaluate(...)` 函数** + 完整 kwargs(15+ 参数)— 一个函数搞定 90% 用户场景
  4. **`integrations/` 11 个文件每个一个平台** — 不堆 if-else
  5. **`tracing/` 子目录 langfuse.py + mlflow.py** — Tracing 跟其他 integration 区分(虽都集成,但 trace 是横切)
  6. **`optimizers/dspy_adapter.py`** — DSPy 互操作,而非 fork
  7. **`_nv_metrics.py`** 单独文件给 NVIDIA — 不污染主 namespace
  8. **`use_v1_metrics`** 给 API 演化留兼容(对照 [[../eval/opik]] `app-server v1/v2`)
---

# Ragas(explodinggradients, Apache 2.0)

## 一句话定位

**业内 RAG eval 标杆**:27+ 内置 metric + RAG 三角(Faithfulness / Context Recall / Context Precision)+ 11+ 平台集成 + Multi-Modal RAG 独家。被 Langfuse / Opik / Helicone / LangSmith / LlamaIndex / Bedrock 等业内主流 platform 都集成。

## RAG 核心三角(必跑)

```
Question + Retrieved Contexts + Answer + Ground Truth
                    ↓
       ┌────────────┼────────────┐
       ↓            ↓            ↓
   Faithfulness  Context    Context
                 Recall     Precision
   答案是否基于  检索是否覆盖  检索是否精准
   检索 context  ground truth  (无 noise)
```

## SkillForge 集成路径

详见 frontmatter `skillforge_takeaway`。核心:

1. **直接 wrap Ragas Python 作为 SkillForge metric backend**(对照 Langfuse `ragas_metric.py` 模式)
2. **`SingleTurnSample` / `MultiTurnSample` schema 对齐**(对照 SkillForge EvalTask multi-turn fix)
3. **RAG 三角 + 4 agent metric(tool call accuracy / goal accuracy / topic adherence)** 优先集成

## 11+ 集成验证 Ragas 地位

```
Ragas(metric library)
  ↓ 被集成
Langfuse / Opik / Helicone / LangChain / LangSmith /
LlamaIndex / Bedrock / LangGraph / Griptape / Swarm /
R2R / AG-UI / MLflow
```

**SkillForge 集成 Ragas 是 join 业内 13 家平台 join 的事实接口**。

## 待回答
- Ragas `v1` vs `v2` metric 具体差异
- DSPy adapter 跟 Genetic optimizer 实际 RAG 优化效果
- Multi-Modal Faithfulness 具体怎么算(图文对齐?)
- Ragas 调 LLM-as-judge 默认 model(便宜的 GPT-3.5 还是 GPT-4)
