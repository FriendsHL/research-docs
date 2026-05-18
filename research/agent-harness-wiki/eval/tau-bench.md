---
type: eval-framework
name: tau-bench
status: active (但已被 τ³-bench 替代)
language: python

# === 版本时效 ===
last_ingested_version: "tau-bench (v1) @ commit 59a200c (2026-03-18)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - τ²-bench → τ³-bench(新 banking domain + voice modality + airline/retail 修复)在 sierra-research/tau2-bench

sources:
  - "[[../source/tau-bench]]"

# === 9 维度 ===
target: |
  **Tool-Agent-User Interaction benchmark**(Sierra Research, MIT)。Multi-turn 评测:**LLM-simulated user × Agent × domain-specific API tools × policy guidelines**。
  跟 [[swe-bench]] 完全不同(SWE-bench one-shot patch,tau-bench multi-turn conversation)。
  ⚠️ **已被 τ³-bench 替代**(新增 banking domain + voice evaluation modality + airline/retail task 修复)。
  [verified-from-source]

dataset: |
  **2 个 domain(本仓库)**(`tau_bench/envs/`):
  - **`airline/`**:航空场景(改签 / 退款 / 行李规则)— `tasks.py` + `tasks_test.py`
  - **`retail/`**:零售场景(订单 / 退货 / 物流)— `tasks.py` + `tasks_test.py` + **`tasks_train.py`** + `tasks_dev.py`(train/dev/test 三档)

  **τ³-bench 扩展**(community-knowledge):新增 `banking` domain + `voice` evaluation modality

  **`few_shot_data/`** 提供 few-shot 示例,**`historical_trajectories/`** 存历史 trajectory 作为 eval input
  [verified-from-source]

scorer: |
  **Pass^k metric**(README leaderboard):
  - **Pass^1**(单次跑通过)
  - **Pass^2 / Pass^3 / Pass^4**(重复跑 k 次都通过的比率)
  - **多次 consistency 是真挑战**:Claude 3.5 Sonnet 在 retail Pass^1 = 0.692 → Pass^4 = 0.462,下降 33%

  **2 种 GradingStrategy**(`auto_error_identification.py:69-71`):
  ```python
  class GradingStrategy(Enum):
      ACTIONS = "actions"      # 比对 action sequence 跟 ground truth
      OUTPUTS = "outputs"      # 比对 output keys
  ```

  **`Action` 数据模型**(`types.py:10-12`):
  ```python
  class Action(BaseModel):
      name: str                # tool name 或 RESPOND_ACTION_NAME="respond"
      kwargs: Dict[str, Any]   # tool 参数 或 {"content": ...}
  ```

  **RewardResult 双路**(`types.py:17-31`):`RewardOutputInfo` / `RewardActionInfo`
  - Output-based:`r_outputs` + `outputs: Dict[str, bool]`
  - Action-based:`r_actions` + `gt_data_hash`(SHA256 of ground-truth data)
  [verified-from-source]

reproducibility: |
  - **`consistent_hash`**(`envs/base.py:39`)用 SHA256 of sorted tuple-ified value 生成 hash — 跨平台一致
  - **`gt_data_hash`** 在 `RewardActionInfo` 锁定 ground-truth data 版本
  - **Multi-provider 支持**:OpenAI / Anthropic / Google / Mistral / AnyScale(env var)
  - **`task_index`** 可指定具体 task 或 random
  - 没有 Docker 容器化(对照 [[swe-bench]])
  [verified-from-source]

coverage: |
  **覆盖**:
  - Multi-turn agent + user 对话
  - **LLM-simulated user** + Human user 双模式(`LLMUserSimulationEnv` + `HumanUserSimulationEnv`)
  - 4 种 agent 策略(`tau_bench/agents/`):TC / ReAct / Act / Few-shot
  - Tool 调用 + policy guideline 约束(`wiki.md` + `rules.py`)
  - **自动错误归因**(`auto_error_identification.py`)

  **不覆盖**:
  - Coding agent / patch(对照 SWE-bench)
  - Long-horizon / multi-day session
  - Multi-agent 协作
  - RAG(对照 [[ragas]])
  [verified-from-source]

production_loop: |
  **One-shot benchmark**,跑完 Pass^k 出 leaderboard 分数结束。

  **跟 SkillForge ATTRIBUTION-AGENT 关键交集**:
  - tau-bench **`auto_error_identification.py`** 是离线归因(对单 trajectory 做事后分析)
  - SkillForge ATTRIBUTION-AGENT 是 production 在线归因(跨 session 聚类 + 决策"该改什么 surface")
  - **同类思路,不同 layer**

  没有 production trace 自动归集 / EvalTemplate 版本(对照 [[../eval/langfuse]])
  [verified-from-source]

automation: |
  - **CLI**:`python run.py --env retail --agent tool_calling --model gpt-4o ...`
  - **`auto_error_identification.py`**:跑完 task 后自动归因失败(LLM 驱动 + ThreadPoolExecutor 并发)
  - **`--max-concurrency`** 控制并发 API 调用
  - **No CI/CD integration**(对照 [[promptfoo]] CLI-first CI 友好)
  [verified-from-source]

cost: |
  - Per-eval 成本:**user simulation LLM × agent LLM × N turns × task 数**(典型 200 task × 10 turns × 2 LLM ≈ $50-150/run)
  - **`SolveResult.total_cost`** 字段(`types.py:36`)+ **`LLMUserSimulationEnv.total_cost`** 跟踪
  - litellm `response_cost`(via `_hidden_params`)拿 per-call cost
  - 没有 cost 聚合 dashboard(对照 [[../eval/langfuse]] / [[../eval/opik]])
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键发现:**tau-bench 跟 SkillForge ATTRIBUTION-AGENT 是同方向工作**

  tau-bench `auto_error_identification.py:31-66` 暴露完整归因 schema:
  ```python
  class FaultAuthor(Enum):
      USER = "user"
      AGENT = "agent"
      ENVIRONMENT = "environment"

  class FaultType(Enum):
      CALLED_WRONG_TOOL = "called_wrong_tool"
      USED_WRONG_TOOL_ARGUMENT = "used_wrong_tool_argument"
      GOAL_PARTIALLY_COMPLETED = "goal_partially_completed"
      OTHER = "other"

  class FaultAssignmentResult(BaseModel):
      task_id: int
      author: FaultAuthor                  # 3 种归因主体
      description: str                     # LLM 写的归因说明
  ```

  **跟 SkillForge ATTRIBUTION-AGENT 校准**:
  - SkillForge 飞轮 V3 ATTRIBUTION-AGENT 决策"该改 skill / prompt / behavior_rule / hook 哪个 surface"
  - tau-bench `FaultAuthor` 决策"是 user / agent / env 谁的错"+ `FaultType` 4 类
  - **SkillForge 是 surface-level optimization 归因**,tau-bench 是 **fault-level 单次失败归因**
  - 两个 schema 都 LLM 驱动,Pydantic typed

  ## 可借鉴(给 SkillForge ATTRIBUTION-AGENT)

  1. **`FaultAuthor` 3 类(USER/AGENT/ENVIRONMENT)+ `FaultType` 4 类 Pydantic enum schema** — SkillForge AttributionAgent 输出可借鉴这种 **typed enum**(对照当前用 LLM free-form description)
  2. **`GradingStrategy` enum (ACTIONS / OUTPUTS)** 区分两种 grading — SkillForge EvalTask 4 维分数(quality/efficiency/latency/cost)可借鉴 ACTIONS 维度(对照 OUTPUTS 维度)
  3. **`consistent_hash` SHA256 of sorted tuple-ified value** 跨平台一致 hash — SkillForge cache 或 dedup 可用
  4. **`SolveResult.total_cost` 字段一等公民** — SkillForge EvalTask 应有同字段(可能已有)
  5. **LLM-simulated user**(`LLMUserSimulationEnv`)+ Human user 双模式 — **SkillForge EVAL-DYNAMIC-USER-SIM**(backlog Phase 2/3)的开源参考实现!直接对照
  6. **`historical_trajectories/`** 把历史 trajectory 当 eval input — SkillForge 已做(production trace → dataset),但 `RewardActionInfo.gt_data_hash` 锁定 ground-truth data 版本是工程细节

  ## SkillForge 已领先(对照 tau-bench)
  - **Production 在线归因**(SkillForge ATTRIBUTION-AGENT 飞轮 V3)vs tau-bench 离线 benchmark 归因 — **SkillForge 走得更远**
  - **多 surface 优化**(skill/prompt/behavior_rule/hook)vs tau-bench 单 fault 归因 — **SkillForge 框架更广**
  - **production loop + canary 灰度回流**(V1+V2)— tau-bench 完全没有

  ## 不必跟
  - tau-bench Pass^k metric — SkillForge 走 4 维 dimensionStatus,**不需要 Pass^k 风格**
  - airline/retail 2 domain — SkillForge 走自家 Scenario,**不需要 hardcoded domain**
  - 已被 τ³-bench 替代,**没必要 fork 旧版**

  ## 关键决策建议

  **如果 SkillForge EVAL-DYNAMIC-USER-SIM(backlog Phase 2/3)继续推进**:
  - 直接参考 tau-bench `LLMUserSimulationEnv.generate_next_message`(`envs/user.py:46-54`)— litellm-based 用户模拟
  - 参考 `auto_error_identification.py:71-100` context_description 给 LLM 的归因提示模板
  - 参考 τ³-bench `voice modality` 探索

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **Multi-turn Tool-Agent-User benchmark**(业内首批,Sierra Research)— 跟 SWE-bench one-shot patch / Opik 8 内置 benchmark QA 形态完全不同
  2. **LLM-simulated user**(`LLMUserSimulationEnv`)+ Human user 双模式 — 业内独家
  3. **自动错误归因**(`auto_error_identification.py`)— **业内 benchmark 里独家把"归因"做成 first-class**
  4. **Pass^k consistency metric**(k=1,2,3,4)— 跑多次取交集,**抓 LLM 输出 stochasticity**
  5. **Wiki + Rules 注入 domain knowledge** — agent prompt 跟 domain 解耦
  6. **2 种 RewardResult**(ActionInfo / OutputInfo)+ `gt_data_hash` 锁定 ground truth

  ## 优势 (strengths)
  1. **`FaultAuthor` 3 类 + `FaultType` 4 类 Pydantic enum schema** — 归因 LLM output typed
  2. **`consistent_hash` 跨平台 SHA256** — 工程细节漂亮
  3. **4 种 agent 策略**(TC / ReAct / Act / Few-shot)一个 codebase 跑 — leaderboard 公平对比
  4. **litellm 多 provider abstraction**(`envs/user.py:5`)— OpenAI / Anthropic / Google / Mistral / AnyScale 统一
  5. **`ThreadPoolExecutor` 并发**(`auto_error_identification.py:12`)— `--max-concurrency` 控制

  ## 哪些点做得好 (what_done_well)
  1. **`RESPOND_ACTION_NAME = "respond"` + `RESPOND_ACTION_FIELD_NAME = "content"`** 把 respond 当成 Action 子类型 — **统一 schema**(不需要区分 tool call vs message)
  2. **`Env` 把 `wiki + rules` 作为构造参数注入** — domain knowledge 显式分离,可热替换
  3. **`tasks_train.py + tasks_dev.py + tasks_test.py`**(retail)— 完整 train/dev/test 三档,防 overfit
  4. **README 顶部明示 outdated + 推 τ³-bench** — Sierra 工程文化,**主动 deprecate 旧版**
  5. **`auto_error_identification.py:74-100` `context_description(grading_strategy)`** 把 ACTIONS vs OUTPUTS 两种 grading 提示模板拆开 — 提示工程
  6. **`display_traj` + `display_actions` + `display_context`** 3 个 helper 把 trajectory → prompt 渲染拆 3 段 — 工程清晰
---

# τ-bench(Sierra Research, MIT)

## 一句话定位
Multi-turn Tool-Agent-User Interaction benchmark,**LLM-simulated user × Agent × domain tools × policy**。**业内 benchmark 里独家"自动错误归因"**(对照 SkillForge ATTRIBUTION-AGENT 同方向)。⚠️ 已被 τ³-bench 替代。

## 起源
- Sierra Research(Bret Taylor / Clay Bavor 公司,2024 推出 OS 级 AI agent platform)
- 2024 paper + leaderboard 出
- 2025 τ²-bench → **τ³-bench**(banking + voice + 修复)

## 5 大工程亮点

### 1. Multi-turn User Simulation
`LLMUserSimulationEnv` 用 litellm 跑另一个 LLM 模拟 user(`envs/user.py:36-54`):
- `system_prompt` 含 instruction
- `generate_next_message` 拿 user 下一句
- `total_cost` 累计成本
- 也支持 `HumanUserSimulationEnv` 真人 input

### 2. 自动错误归因(`auto_error_identification.py`)

最重要的工程产物 — **业内首批把"benchmark 失败归因"做成 first-class**:

```python
class FaultAuthor(Enum):       # 3 类归因主体
    USER, AGENT, ENVIRONMENT

class FaultType(Enum):         # 4 类错误
    CALLED_WRONG_TOOL,
    USED_WRONG_TOOL_ARGUMENT,
    GOAL_PARTIALLY_COMPLETED,
    OTHER
```

LLM 驱动:把 user instruction + ground truth + trajectory → LLM → `FaultAssignmentResult { task_id, author, description }`。

**对照 SkillForge ATTRIBUTION-AGENT**:
| 维度 | tau-bench auto_error_identification | SkillForge ATTRIBUTION-AGENT |
|---|---|---|
| Layer | 离线 benchmark 单 trajectory | Production 跨 session 聚类 |
| 归因 schema | `FaultAuthor × FaultType` 3×4 enum | 改 `skill/prompt/behavior_rule/hook` surface |
| 触发 | 用户手动跑 CLI | 自动 cron(SessionAnnotationService + PatternClusteringService) |
| 输出 | `FaultAssignmentResult` Pydantic | Optimization Event 因果链 |
| 后续 action | 用户读 description 改 prompt | 自动入 candidate 生成 + A/B + canary |

**SkillForge 已在 tau-bench 之上一层**,但 **`FaultAuthor + FaultType` enum schema 工程细节可借鉴**(给 ATTRIBUTION-AGENT 输出加 typed enum,而非 free-form description)。

### 3. Pass^k Metric(consistency)

抓 LLM 输出 stochasticity:Pass^k = 重复跑 k 次都通过的比率。

```
Claude 3.5 Sonnet (20241022) retail:
  Pass^1 = 0.692 → Pass^4 = 0.462  (33% drop)
```

**关键 insight**:**single-pass benchmark 高估 production 稳定性**,Pass^4 更接近 production。

### 4. 2 种 GradingStrategy

```python
class GradingStrategy(Enum):
    ACTIONS = "actions"    # 比对 action sequence(tool 调用序列)
    OUTPUTS = "outputs"    # 比对 output keys(语义结果)
```

`RewardActionInfo` 锁 `gt_data_hash`(SHA256 of ground-truth)防数据集 drift。

### 5. 4 种 Agent 策略一个 codebase

`tau_bench/agents/{tool_calling, chat_react, few_shot, base}_agent.py` — 同一 Env 跑 4 种策略对比 leaderboard。

## SkillForge 校准

详见 frontmatter `skillforge_takeaway`。最关键 3 条:

1. **`FaultAuthor + FaultType` typed enum schema** 可加给 SkillForge ATTRIBUTION-AGENT 输出
2. **`LLMUserSimulationEnv` 是 EVAL-DYNAMIC-USER-SIM(backlog Phase 2/3)的开源参考实现** — 直接对照
3. **`consistent_hash` SHA256 + `gt_data_hash`** 工程细节给 dataset versioning 借鉴

## 待回答
- τ³-bench `banking domain` 实现细节(needs separate ingest)
- `voice evaluation modality` 怎么做 — 真音频还是 transcription
- Sierra Research 完整 platform 跟 SkillForge platform 定位差异
