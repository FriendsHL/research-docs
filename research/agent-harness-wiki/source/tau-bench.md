---
type: source
name: tau-bench
origin: "/Users/youren/myspace/tau-bench/ (Sierra Research, MIT)"
ingested_at: 2026-05-16
ingested_commit: 59a200c
ingested_commit_date: 2026-03-18
ingested_version_label: "(无 PyPI 包,以 commit 为准。⚠️ 已被 τ²-bench → τ³-bench 替代,新 banking domain + voice modality)"
contributed_to:
  - eval/tau-bench.md
  - concept/benchmark-datasets.md
---

# Source — τ-bench

> **Sierra Research 出品**(MIT,2024)— **Tool-Agent-User Interaction benchmark**。User 由 LLM 模拟,Agent 在 airline / retail domain 完成 task,有 policy guideline + tool。**已被 τ³-bench 替代**(新增 banking domain + voice modality)。

## 仓库结构概览

```
tau-bench/
├── tau_bench/
│   ├── types.py                    # Pydantic types(Action / Task / RewardResult / SolveResult)
│   ├── run.py                      # 主入口
│   ├── envs/                       # 2 个 domain
│   │   ├── airline/                # 航空(env.py / tasks.py / rules.py / tools/ / wiki.md)
│   │   ├── retail/                 # 零售(同上 + tasks_train.py / tasks_dev.py / tasks_test.py)
│   │   ├── base.py                 # Env 基类
│   │   ├── user.py                 # **LLM-simulated user**
│   │   └── tool.py                 # Tool 基类
│   ├── agents/                     # 4 种 agent 策略
│   │   ├── tool_calling_agent.py   # TC(主 leaderboard 用)
│   │   ├── chat_react_agent.py     # ReAct
│   │   ├── few_shot_agent.py       # Few-shot
│   │   └── base.py
│   └── model_utils/                # Model wrapper(args.py)
├── auto_error_identification.py    # 自动错误归因脚本
├── few_shot_data/                  # Few-shot 示例
├── historical_trajectories/        # 历史 trajectory(eval input)
├── run.py                          # 顶层 entry
└── setup.py                        # `tau_bench` PyPI 包
```

## 关键 file:line 锚点

### 核心数据模型(`tau_bench/types.py`)

```python
# types.py:7
RESPOND_ACTION_NAME = "respond"
RESPOND_ACTION_FIELD_NAME = "content"

class Action(BaseModel):     # tool 调用或 respond
    name: str
    kwargs: Dict[str, Any]

class Task(BaseModel):       # 任务定义
    user_id: str
    actions: List[Action]    # gold trajectory
    instruction: str         # user instruction
    outputs: List[str]       # expected output keys

class RewardOutputInfo(BaseModel):    # output-based reward
    r_outputs: float
    outputs: Dict[str, bool]

class RewardActionInfo(BaseModel):    # action-based reward
    r_actions: float
    gt_data_hash: str        # ground-truth data hash

class RewardResult(BaseModel):        # 综合
    reward: float
    info: Union[RewardOutputInfo, RewardActionInfo]
    actions: List[Action]

class SolveResult(BaseModel):
    reward: float
    messages: List[Dict[str, Any]]
    info: Dict[str, Any]
    total_cost: Optional[float] = None
```

### Env 设计(`tau_bench/envs/base.py`)

```python
# base.py:39 (consistent_hash with SHA256)
def consistent_hash(value: Hashable) -> str:
    return sha256(str(value).encode("utf-8")).hexdigest()

# base.py:46 Env class
class Env(object):
    def __init__(
        self,
        data_load_func,      # 加载 domain 数据(airline / retail)
        tools,               # 可用 tool 列表
        tasks,               # task 库
        wiki: str,           # domain knowledge(航空公司政策 / 退货规则等)
        rules: List[str],    # policy rules
        user_strategy,       # user 模拟策略
        user_model,          # 模拟 user 的 LLM
        user_provider,       # provider
        task_index,
    ):
        ...
```

**关键设计**(`base.py:46-90`):
- **LLM-simulated user**:user 不是真人,是另一个 LLM 模拟(`user_model` + `user_strategy`)
- **Wiki + Rules** 作为 domain knowledge 注入 agent
- **Tools 作为 typed API**(航空查航班 / 改签 / 退款等)
- **Task = (instruction, gold actions, expected outputs)**

### 4 种 Agent 策略(`tau_bench/agents/`)

- **`tool_calling_agent.py`** — Tool-Calling(主 leaderboard,Claude 3.5 Sonnet 取得 best)
- **`chat_react_agent.py`** — ReAct(`arxiv.org/abs/2210.03629`)
- **`few_shot_agent.py`** — Few-shot
- **`base.py`** — Agent 基类

### Leaderboard 评分(README)

**Pass^k**(`k=1,2,3,4`)— 重复跑 k 次都通过的比率。

#### Airline(2 domain 之一)
| Strategy | Pass^1 | Pass^4 |
|---|---|---|
| **TC Claude 3.5 Sonnet (20241022)** | **0.460** | **0.225** |
| TC GPT-4o | 0.420 | 0.200 |
| TC Claude 3.5 Sonnet (20240620) | 0.360 | 0.139 |
| TC GPT-4o-mini | 0.225 | 0.100 |
| Act GPT-4o | 0.365 | 0.140 |
| ReAct GPT-4o | 0.325 | 0.160 |

#### Retail
| Strategy | Pass^1 | Pass^4 |
|---|---|---|
| **TC Claude 3.5 Sonnet (20241022)** | **0.692** | **0.462** |
| TC GPT-4o | 0.604 | 0.383 |
| TC Claude 3.5 Sonnet (20240620) | 0.626 | 0.387 |

**关键 insight**:Pass^k 下降快(0.460 → 0.225 over 4 runs)— **multi-run 一致性是真挑战**。

### 自动错误归因(`auto_error_identification.py`)

**业内独家**:跑完 task 后,自动分析失败原因(tool 调用错 / user simulation 偏差 / policy 违反等)。**跟 SkillForge ATTRIBUTION-AGENT 同方向**!

### 多 provider 支持(README)
- OpenAI / Anthropic / Google / Mistral / AnyScale(env vars 设置 API key)

### 重要警告(README 顶部)
> **⚠️ This repository contains outdated versions of the airline and retail tasks. Please use τ³-bench(`sierra-research/tau2-bench`)**
- τ³-bench 修了 airline / retail task bug
- 加 banking domain + voice modality

## 与 wiki 现有

### 印证
- **跟 SkillForge ATTRIBUTION-AGENT 同方向**(`auto_error_identification.py` 自动错误归因)— **业内 2 家做归因**:Sierra τ-bench + SkillForge
- **Pydantic types 一等公民** 印证 [[../eval/langfuse]] / [[../eval/opik]] schema-first IO

### 新增
- **`eval/tau-bench.md`**(新建)— **Multi-turn user-simulated agent benchmark**,跟 SWE-bench 互补(SWE-bench 是 one-shot patch)
- 加 **[[../concept/benchmark-datasets]]** 三层分类:platform / framework / benchmark

## 事实漂移

- ⚠️ **τ-bench 已被 τ³-bench 替代**,本仓库 outdated(用户应迁 `sierra-research/tau2-bench`)
- **没有 web UI / dashboard** — 纯 Python CLI benchmark
- **2 个 domain**(airline / retail)实际可扩,τ³-bench 加 banking
- **LLM-simulated user 是关键创新**:真实用户对话 expensive,LLM 模拟降低成本但带来 user simulation bias 问题
