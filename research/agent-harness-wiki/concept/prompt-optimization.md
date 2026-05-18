---
type: concept
name: prompt-optimization
status: draft
sources:
  - "[[../source/opik]]"
  - "/Users/youren/myspace/opik/sdks/opik_optimizer/src/opik_optimizer/__init__.py"
  - "/Users/youren/myspace/opik/sdks/opik_optimizer/README.md"
related_harnesses: []
related_evals: [opik]
---

# Prompt Optimization — Algorithm-driven prompt evolution

> **4 家独家提示**:本概念目前**只在 Opik 一家**作为 first-class SDK 实现(`sdks/opik_optimizer/`)。其他 3 家(Hermes / Claude Code / OpenClaw / Codex / Langfuse)都没有 dedicated optimizer 算法,**Langfuse 有 experiment 做对比但不自动改 prompt**。

## 定义

**Prompt Optimization** = 用算法(遗传、Bayesian、Meta-prompting、Pareto、Reflection 等)在 **dataset + metric** 输入下,**自动迭代 prompt** 直到 metric 收敛,**不需要人工 prompt engineering**。

跟 [[self-evolution]] 的关系:
- self-evolution 是更宽概念,**3 类已知 + 1 类新加**(详见 [[self-evolution]] 类 4)
- prompt-optimization 是 self-evolution 类 4(runtime optimization 无人审,算法自动改 prompt)的**具体技术实现**

跟 traditional prompt engineering 的关系:
- traditional:**人类**根据直觉改 prompt → 跑 eval → 看结果 → 再改
- prompt-optimization:**算法**根据 dataset metric 改 prompt → 跑 eval → 算法 update → 再改

## 设计空间(6 维)

### 1. 搜索算法
- **遗传算法 / 进化算法**(`EvolutionaryOptimizer`)— 维护 population,crossover + mutation,适合大搜索空间无梯度
- **Bayesian Optimization**(`FewShotBayesianOptimizer` / `ParameterOptimizer`)— 概率代理模型 + acquisition function,sample-efficient,适合数据少
- **Meta-prompting**(`MetaPromptOptimizer`)— **LLM 自己改 prompt**(LLM-as-meta-optimizer),利用 LLM 对自己的元能力
- **GEPA(Genetic-Pareto)**(`GepaOptimizer`)— 多目标 Pareto front 优化,适合多 metric trade-off
- **Hierarchical Reflective**(`HierarchicalReflectiveOptimizer` / `HRPO`)— LLM 多层 self-reflection 改 prompt
- **Gradient-based** — 不在 Opik(LLM 是黑盒,gradient 不可得;只有 fine-tune 场景适用)

### 2. 优化目标
- **单 metric**:accuracy / exact_match / answer_correctness
- **多 metric**(`MultiMetricObjective` `opik_optimizer/metrics/multi_metric_objective.py`)— Pareto 或 weighted combination
- **多模态 metric**(text + image + audio + video,OpenAI content parts)

### 3. 优化对象
- **Prompt 字符串**(主):system prompt / user prompt template
- **Few-shot examples**(`FewShotBayesianOptimizer`):选最佳的 N 个 examples
- **LLM 参数**(`ParameterOptimizer`):temperature / top_p / max_tokens / frequency_penalty
- **Agent 配置**(`OptimizableAgent` `opik_optimizer/agents/optimizable_agent.py`):整个 agent(prompt + tools + behavior)
- **MCP tools**(`ChatPrompt.tools`):带 MCP 工具调用的 agent

### 4. 数据集
- **用户自定 dataset**(Opik / Langfuse Dataset 实体)
- **内置 benchmark**(Opik 8 个独家):
  - `gsm8k.py` — 小学数学推理
  - `truthful_qa.py` — 真实性 QA
  - `halu_eval.py` — Hallucination 检测
  - `ai2_arc.py` — AI2 ARC reasoning
  - `arc_agi2.py` — ARC AGI v2(挑战性)
  - `pupa.py` — PUPA
  - `medhallu.py` — Medical hallucination
  - `tiny_test.py` — 调试 fixture
- **Synthetic dataset**(LLM 生成,Opik 部分支持)

### 5. 算法可组合性
- **Standalone**(单算法跑到收敛)
- **Optimizer Chaining**(Opik first-class)— 一个 optimizer `OptimizationResult` 作为下一个 optimizer 输入
  - 典型组合:`EvolutionaryOptimizer`(大搜索)→ `FewShotBayesianOptimizer`(精炼)→ `ParameterOptimizer`(参数微调)
- **Concurrent**(并行多 optimizer 跑同任务,选 best)
- **Hierarchical**(`HierarchicalReflectiveOptimizer` 内部多层反思)

### 6. 成本控制
- **Counter Tracking**(Opik `opik_optimizer` README:"Built-in LLM and tool call counters for monitoring usage")— 每次 optimizer call 自动计数 LLM + tool call
- **Sample budget**:限定最大 LLM 调用次数
- **Early stopping**:metric 收敛 / no improvement for N rounds 提前停
- **Multi-fidelity**:先用小 dataset 粗选,再用大 dataset 精选(Opik 未明示但 best practice)

## Opik 7 算法对照表

| 算法 | 类型 | 适用 | 关键 paper / 来源 |
|---|---|---|---|
| **EvolutionaryOptimizer** | GA | 大搜索空间无梯度 | 经典 genetic algorithm |
| **FewShotBayesianOptimizer** | BO + few-shot | 数据少 | 业内 Bayesian few-shot 标准 |
| **MetaPromptOptimizer** | LLM-as-meta | LLM 改 LLM | OPRO / APE / PromptBreeder 相关 |
| **GepaOptimizer** | GEPA(Genetic-Pareto) | 多目标 Pareto | Pareto evolutionary 文献 |
| **ParameterOptimizer** | Bayesian(参数) | 调 temp/top_p | 通用 hyperparameter optimization |
| **HierarchicalReflectiveOptimizer** | LLM hierarchical reflection | 复杂任务多层反思 | Reflexion / Self-Refine 相关 |
| **HRPO** | (待 verify) | (待 verify) | (源码未深读) |

**关键工程统一性**(对照 [[../source/opik]] `__init__.py:18-26`):
- **统一接口**:`BaseOptimizer(ABC)` + `optimize_prompt()` / `optimize_mcp()`
- **统一返回**:`OptimizationResult` 标准对象
- **统一 metric 接口**:`MultiMetricObjective`
- **统一 dataset 接口**:8 个内置 + 用户 Dataset 都过 `opik.datasets.*`

## 与其他 framework 的关系

### Opik vs 业内开源 prompt optimization 框架

| Framework | 算法数 | 集成 dataset | Optimizer Chaining | MCP support | 成本控制 |
|---|---|---|---|---|---|
| **Opik Agent Optimizer** | **7**(独家) | **8 内置 benchmark** | ✅ first-class | ✅ `ChatPrompt.tools` | ✅ Counter Tracking |
| **DSPy**(Stanford) | 多个(Teleprompter)| ❌ 用户自定 | 有但抽象不同 | ❌ | 用户自管 |
| **PromptBreeder**(DeepMind) | 1(meta-prompt evolutionary) | benchmark 论文用 | ❌ | ❌ | ❌ |
| **APE**(automatic prompt engineering) | 1(LLM gradient) | benchmark 论文用 | ❌ | ❌ | ❌ |
| **OPRO**(Google) | 1(LLM-as-optimizer) | benchmark 论文用 | ❌ | ❌ | ❌ |
| **Langfuse** | 0(只 experiment 对比) | ✅ dataset | ❌ | ❌ | ❌ |
| **MLflow / W&B** | 0(experiment tracking) | ❌ | ❌ | ❌ | ❌ |

**结论**:Opik 是**业内 prompt optimization 工程化最完整的开源框架**,7 算法 + 标准化接口 + 8 benchmark + Optimizer Chaining + MCP + Counter Tracking 全栈。

### Opik vs SkillForge `/evolve`

| | Opik Agent Optimizer | SkillForge `/evolve` |
|---|---|---|
| 触发 | 用户在 Notebook / SDK 跑 | 用户 `/evolve` slash command |
| 改什么 | prompt 字符串 / few-shot / 参数 / agent config | rule / agent / command / skill(meta dev artifact) |
| 自动度 | **完全自动**(算法迭代) | LLM 提议 → **人审** → 落 dev artifact |
| 数据驱动 | dataset + metric | session trace + lint |
| 频率 | 离线跑(可能小时级) | session 后即时 |
| 目标 | runtime prompt quality | dev workflow improvement |
| 风险 | overfit dataset | meta artifact drift |

**两条路线互补**:Opik 是 **runtime optimization 无人审**,SkillForge 是 **meta self-evolution 有人审**。**SkillForge 应保留 `/evolve` 路线 + 引入 Opik-style runtime optimizer 作为第二条路**。

## Trade-off

### 算法选择 trade-off

**没有最好的算法,只有最适合任务的算法**:

| 任务特征 | 推荐算法 | 理由 |
|---|---|---|
| 大搜索空间(prompt 多个独立段) | EvolutionaryOptimizer | GA 适合大空间 |
| 数据少(< 100 example) | FewShotBayesianOptimizer | Bayesian sample-efficient |
| LLM 模型强(GPT-4+) | MetaPromptOptimizer | LLM 改 LLM,模型强效果好 |
| 多 metric 有 trade-off | GepaOptimizer | Pareto front 探索 |
| 只调 temperature / top_p | ParameterOptimizer | Bayesian over numeric space |
| 复杂任务需多步推理优化 | HierarchicalReflectiveOptimizer | 层级反思 |

**Optimizer Chaining 实战推荐**:
1. `EvolutionaryOptimizer` 粗搜大空间
2. `MetaPromptOptimizer` 让 LLM 在 GA 找到的局部最优附近精化
3. `FewShotBayesianOptimizer` 选最佳 few-shot
4. `ParameterOptimizer` 最后调 temperature

### 自动 prompt optimization 的 风险

- **Overfit 单 dataset** — 用 8 个 benchmark cross-validate
- **算法消耗 token 多**:7 算法 × 8 benchmark cross-validation 单次跑可能 $100+,**必须 Counter Tracking + sample budget**
- **不可解释性**:遗传算法跑出的 prompt 可能"看起来奇怪",但 metric 好 — **是否信任无解释的改进?**
- **Distribution shift**:dataset 跟 production traffic 不同分布,optimizer 优化 dataset metric 可能 hurt production
- **Reward hacking**:LLM-as-judge 评 LLM-as-optimizer 改的 prompt,**自评 bias 累加**

## SkillForge 当前选择 + 演进方向

**当前**:`/evolve` meta self-evolution(LLM 提议 → 人审 → 落 `.claude/rules/`)

**演进选项**:
1. **保持现状**(默认):不引入 runtime optimizer,只做 meta self-evolution
2. **引入 1 算法 POC**(推荐):选 `GepaOptimizer`(多目标)或 `FewShotBayesianOptimizer`(数据少),Python SDK 嵌入 SkillForge
3. **完整引入 7 算法**:工程量大,不推荐起步
4. **借鉴 8 内置 benchmark**:SkillForge 应内置 1-2 个业内 benchmark(gsm8k + truthful_qa),用户 zero-config 跑

**关键决策点**:
- **JVM 嵌 Python 跑 optimizer 成本高** — 可考虑 GraalVM 嵌 Python 或独立 Python sandbox service(对照 Opik `apps/opik-sandbox-executor-python/`)
- **Optimizer 跑成本高** — `Counter Tracking` 等价物必加
- **SkillForge 多租户场景** — optimizer 跑长时间任务,需 Langfuse main/secondary 双队列分流避免拖慢其他 project

**判定**:**runtime optimizer 是 SkillForge eval pipeline 的 next horizon**,但优先级低于 1)Score 数据模型(Langfuse 路线)+ 2)main/secondary 双队列 + 3)Pytest CI 集成。**先把 eval foundation 做扎实,再引入 optimizer**。

## 学习视角:为什么 prompt optimization 现在才火

### 历史脉络

- **2020-2022**:Prompt engineering 是手工艺,best practice 是 blog + tweet 流传
- **2022-2023**:LangChain 起,prompt template 工程化但仍手工调
- **2023**:**APE / OPRO / PromptBreeder** 论文出,首批 LLM-as-optimizer 工作
- **2024**:**DSPy** Stanford 发,把 prompt 当 ML 模型可优化
- **2024-2025**:**GEPA / Reflexion / Self-Refine** 学术方向多
- **2025-2026**:**Opik Agent Optimizer 7 算法统一开源 SDK** + LiteLLM 标准化 provider,**工程化第一次完整**

### 为什么 LLM 场景特别需要 prompt optimization

1. **Prompt 是新代码,但没有编译器**:错误只能 eval 发现,人类 review 慢
2. **LLM 模型频繁升级**:GPT-4 → 4o → 5 → o1 → 5.1,**同一 prompt 在新模型可能退化**,需重新优化
3. **多 metric trade-off**:accuracy vs cost vs latency vs safety,人类难手工平衡
4. **Distribution shift**:production traffic 变,prompt 需跟踪
5. **Multi-agent 系统 prompt 互相影响**:单 agent prompt 优化简单,**多 agent 联合优化几乎不可能手工做**

### 跟 fine-tune 的对比

| | Prompt optimization | Fine-tune |
|---|---|---|
| 改什么 | Prompt(LLM 输入) | Weights(LLM 内部) |
| 数据量需求 | 小(10-100 examples) | 大(1000+ examples) |
| 成本 | 低(LLM API 调用) | 高(GPU training) |
| 可逆性 | 完全可逆(改回 prompt) | 不可逆(weight 固化) |
| 部署 | 立即生效 | 需 deploy 新 model |
| 适合 | 任务定义清晰 + 数据少 | 任务复杂 + 数据多 |

**结论**:**Prompt optimization 不替代 fine-tune,而是 fine-tune 之前的低成本路径**。SkillForge 80% 用户场景应优先 prompt optimization,不需要 fine-tune。

## 反模式

- ❌ **没有 dataset 跑 optimizer** — 没有 ground truth,optimizer 算的是 metric 不是真实质量
- ❌ **single metric 单优化** — overfit metric ignore 其他维度,用 `MultiMetricObjective`
- ❌ **跑完不 cross-validate** — overfit dataset,用 8 个 benchmark cross-validate
- ❌ **不限 sample budget** — 算法可能跑爆 token 预算,必须 `Counter Tracking` + sample budget
- ❌ **完全相信 optimizer 输出** — LLM-as-judge 评 LLM-as-optimizer 改的 prompt,self-evaluation bias 累加,**production 上线前必须人审 + production traffic A/B**
- ❌ **不版本化 prompt** — optimizer 跑出 N 个 candidate,不版本化无法回滚
- ❌ **同步 block 跑 optimizer** — 7 算法 × 大 dataset 可能小时级,**必须异步 + sample budget + 提前停**
- ❌ **多 agent 系统单 agent 优化** — agent 间 prompt 互相影响,**应整体 joint optimize**
