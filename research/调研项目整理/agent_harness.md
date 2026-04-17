# AI Agent Harness 调研报告

> 调研时间：2026-03-19
> 调研范围：AI Agent 领域的 Harness 概念、框架、实践

---

## 一、核心概念

### 1.1 Test Harness 在软件工程中的定义

**传统定义：**
Test Harness（测试线束/测试框架）是指用于自动化测试的软件工具和脚本的集合，包括：
- 测试执行引擎
- 测试数据管理
- 结果收集和报告
- Mock/Stub 组件

**核心作用：**
- 隔离被测系统（SUT）
- 提供可控的测试环境
- 自动化测试执行
- 标准化结果收集

### 1.2 Evaluation Harness 在 AI/ML 领域的含义

**ML 领域的 Evaluation Harness：**
一套标准化的评估框架，用于：
- 统一的评测流程
- 标准化的数据集和指标
- 可复现的实验结果
- 模型间的公平比较

**典型示例：**
- **lm-evaluation-harness** (EleutherAI): 大语言模型评估的事实标准
- **HELM** (Stanford): 全方位的语言模型评估
- **BIG-bench** (Google): 超越现有基准的评估框架

### 1.3 Agent Harness 的定义

**Agent Harness 是什么：**

Agent Harness 是专门针对 AI Agent 系统设计的测试和评估框架，需要处理 Agent 的独特特性：

```
传统软件测试 → 固定输入 → 固定输出
传统 ML 评估   → 固定样本 → 预测标签
Agent Harness  → 开放环境 → 多步交互 → 不确定性结果
```

**核心组件：**
1. **Environment Harness**: 模拟 Agent 运行环境（工具、API、沙箱）
2. **Task Harness**: 定义任务、目标、成功标准
3. **Interaction Harness**: 管理多轮对话/交互
4. **Evaluation Harness**: 评估 Agent 行为和结果
5. **Observability Harness**: 监控、日志、追踪

---

## 二、主流框架/工具

### 2.1 学术/研究机构框架

#### Stanford HELM (Holistic Evaluation of Language Models)
- **机构**: Stanford CRFM
- **特点**: 多维度评估（准确率、鲁棒性、公平性、效率等）
- **覆盖**: 50+ 场景，数百个指标
- **链接**: https://crfm.stanford.edu/helm/lite/

#### EleutherAI lm-evaluation-harness
- **机构**: EleutherAI
- **特点**: 开源社区标准，支持数百个任务
- **GitHub**: https://github.com/EleutherAI/lm-evaluation-harness
- **使用**: Hugging Face、NVIDIA、MosaicML 等都在用

#### BIG-bench
- **机构**: Google Research + 合作机构
- **特点**: 超过 200 个任务，关注"超越现有模型能力"
- **GitHub**: https://github.com/google/BIG-bench

#### AgentBench
- **机构**: 清华大学等
- **特点**: 专门针对 LLM-as-Agent 的多维度评测
- **评估维度**: 多轮对话、工具使用、推理、规划
- **GitHub**: https://github.com/THUDM/AgentBench

#### AgentEval (Microsoft)
- **机构**: Microsoft Research
- **特点**: 评估 Agent 在复杂任务中的表现
- **关注点**: 任务完成率、效率、安全性

### 2.2 开源项目

#### LangChain Evaluation
- **仓库**: https://github.com/langchain-ai/langchain
- **工具**: 
  - `LangSmith`: 完整的调试和评估平台
  - `LangGraph`: 可观测性 + 测试
- **评估方式**: 
  - 基于参考答案的评估
  - 基于规则的评估
  - LLM-as-judge

#### TruLens
- **仓库**: https://github.com/truera/trulens
- **特点**: LLM 应用的可观测性和评估
- **功能**:
  - RAG 评估（上下文相关性、答案正确性）
  - Chain 追踪
  - 反馈函数库

#### Ragas
- **仓库**: https://github.com/explodinggradients/ragas
- **特点**: RAG 系统评估
- **指标**:
  - Faithfulness（忠实度）
  - Answer Relevance（答案相关性）
  - Context Precision（上下文精确度）
  - Context Recall（上下文召回率）

#### Promptfoo
- **仓库**: https://github.com/promptfoo/promptfoo
- **特点**: Prompt 和 Agent 的红队测试
- **功能**:
  - 批量测试 prompt 变体
  - 安全评估
  - 回归测试

#### Agent-Eval
- **仓库**: 多个独立项目
- **主要方向**:
  - 工具调用正确性
  - 多步规划能力
  - 错误恢复能力

### 2.3 商业平台

#### LangSmith (LangChain)
- **定位**: 端到端的 LLM 应用开发平台
- **功能**: 追踪、测试、评估、监控
- **特点**: 与 LangChain 生态深度集成

#### Weights & Biases (W&B)
- **定位**: ML 实验追踪平台
- **Agent 支持**: Prompt 追踪、评估、对比
- **特点**: 强大的可视化

#### Arize AI
- **定位**: ML 可观测性平台
- **功能**: 模型监控、漂移检测、性能分析

#### Humanloop
- **定位**: LLM 迭代平台
- **功能**: Prompt 测试、版本控制、评估

### 2.4 Agent 特定框架

#### WebArena
- **机构**: CMU 等多校联合
- **特点**: 真实 Web 环境中的 Agent 评估
- **任务**: 网页浏览、表单填写、电商操作等
- **GitHub**: https://github.com/web-arena-x/webarena

#### InterCode
- **机构**: Stanford
- **特点**: 代码执行环境中的 Agent 评估
- **功能**: Shell、Python、SQL 环境

#### ToolBench
- **机构**: 清华等
- **特点**: 工具学习评估
- **GitHub**: https://github.com/openbmb/toolbench

#### GentBench
- **机构**: 微软等
- **特点**: 通用 Agent 任务基准
- **关注**: 工具使用、推理、规划

---

## 三、关键问题

### 3.1 Agent 评估的独特挑战

#### 3.1.1 非确定性
**问题描述:**
- 相同输入可能产生不同输出（LLM 的随机性）
- 多次执行可能有不同路径
- 结果难以确定性验证

**解决方案:**
```
1. 多次采样 + 统计显著性
   - 同一任务运行 N 次
   - 报告成功率分布（而非单次结果）
   
2. 种子固定
   - 固定随机种子
   - 可复现但可能掩盖问题

3. 确定性输出模式
   - 强制使用 JSON/XML 输出
   - 约束解码（constrained decoding）
```

#### 3.1.2 多轮交互
**问题描述:**
- 评估依赖交互历史
- 错误会传播和放大
- 难以定义"正确"的对话路径

**解决方案:**
```
1. 模拟用户 (User Simulator)
   - 规则驱动
   - LLM 扮演
   - 预定义对话树

2. 检查点评估
   - 关键节点评估
   - 最终目标评估
   - 过程质量评估

3. 状态机验证
   - 定义合法状态转移
   - 检测非法状态
```

#### 3.1.3 工具调用
**问题描述:**
- 外部工具可能失败
- 工具状态管理复杂
- 副作用难以回滚

**解决方案:**
```
1. Mock 工具
   - 本地模拟工具行为
   - 预定义响应
   - 状态隔离

2. 沙箱环境
   - Docker 容器隔离
   - 可快照/回滚
   - 资源限制

3. 工具追踪
   - 记录所有工具调用
   - 验证调用序列
   - 检查参数正确性
```

#### 3.1.4 长时任务
**问题描述:**
- 任务可能需要很多步骤
- 评估成本高
- 中间失败难以调试

**解决方案:**
```
1. 任务分解
   - 将长任务拆分为子任务
   - 独立评估每个子任务
   - 组合评估整体

2. 检查点恢复
   - 保存执行状态
   - 从失败点重试
   - 减少重复执行

3. 早停机制
   - 检测明显失败
   - 跳过后续步骤
```

### 3.2 主流评估指标和方法

#### 3.2.1 任务完成指标
| 指标 | 描述 | 适用场景 |
|------|------|----------|
| Success Rate | 任务成功完成的比例 | 最终目标评估 |
| Task Completion | 是否达到目标状态 | 确定性任务 |
| Partial Credit | 部分完成的得分 | 复杂多步任务 |
| Steps to Success | 完成任务所需步数 | 效率评估 |

#### 3.2.2 行为质量指标
| 指标 | 描述 | 评估方式 |
|------|------|----------|
| Tool Call Accuracy | 工具调用正确率 | 规则/LLM Judge |
| Response Quality | 回复质量 | LLM-as-Judge |
| Reasoning Coherence | 推理连贯性 | LLM评估/人工 |
| Safety Compliance | 安全合规性 | 规则检查 |

#### 3.2.3 效率指标
| 指标 | 描述 | 评估方式 |
|------|------|----------|
| Token Usage | Token 消耗 | 直接统计 |
| Latency | 响应延迟 | 时间测量 |
| Cost | API 调用成本 | 价格计算 |
| Iteration Count | 迭代次数 | 直接统计 |

#### 3.2.4 评估方法

**1. 规则基础评估**
```python
def evaluate_tool_call(call, expected):
    """检查工具调用是否符合预期"""
    return (
        call.name == expected.name and
        set(call.args.keys()) == set(expected.args.keys()) and
        all(call.args[k] == expected.args[k] for k in expected.args)
    )
```

**2. LLM-as-Judge**
```python
PROMPT = """
你是一个评估专家。请评估以下 Agent 回复的质量。

任务: {task}
Agent 回复: {response}
参考答案: {reference}

请从以下维度评分 (1-5):
1. 正确性: 是否正确完成任务
2. 完整性: 是否完整回答
3. 有用性: 对用户是否有帮助
"""
```

**3. 嵌入相似度**
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')
similarity = cosine_similarity(
    model.encode([agent_output]),
    model.encode([reference_output])
)
```

**4. 模拟用户评估**
```python
# 用户模拟器
user_simulator = LLM(system_prompt="你是一个需要帮助的用户...")

# 对话评估
while not done:
    user_msg = user_simulator.get_response(agent_msg)
    agent_msg = agent.get_response(user_msg)
    evaluate_turn(user_msg, agent_msg)
```

### 3.3 与传统 ML 评估的区别

| 维度 | 传统 ML 评估 | Agent Harness |
|------|-------------|---------------|
| **输入** | 固定样本 | 开放环境/用户意图 |
| **输出** | 预测标签 | 行为序列/最终状态 |
| **评估** | 准确率/F1 等 | 任务成功率/行为质量 |
| **环境** | 无状态 | 有状态/工具/环境 |
| **交互** | 单次推理 | 多轮交互 |
| **确定性** | 高 | 低（LLM 随机性） |
| **可复现性** | 高 | 需要额外设计 |
| **成本** | 低 | 高（多步推理） |

---

## 四、实践案例

### 4.1 构建一个简单的 Agent Test Harness

#### 架构设计
```
┌─────────────────────────────────────────────────────┐
│                  Test Harness                        │
├─────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Task Runner  │  │ Environment  │  │  Evaluator │ │
│  │              │  │   Manager    │  │            │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│         │                 │                 │         │
│         ▼                 ▼                 ▼         │
│  ┌──────────────────────────────────────────────┐   │
│  │                Agent Under Test              │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

#### 代码示例

**1. 基础框架**
```python
from dataclasses import dataclass
from typing import Any, Callable, Optional
from enum import Enum
import json

class TaskStatus(Enum):
    SUCCESS = "success"
    FAILURE = "failure"
    PARTIAL = "partial"
    ERROR = "error"

@dataclass
class Task:
    """定义一个测试任务"""
    id: str
    description: str
    initial_state: dict
    goal_state: dict
    tools_available: list[str]
    max_steps: int = 10
    timeout: int = 300  # seconds

@dataclass
class TaskResult:
    """任务执行结果"""
    task_id: str
    status: TaskStatus
    steps: list[dict]
    final_state: dict
    metrics: dict
    error: Optional[str] = None

class AgentHarness:
    """Agent 测试框架"""
    
    def __init__(self, agent, environment):
        self.agent = agent
        self.environment = environment
        self.evaluators = []
        self.mocks = {}
    
    def register_evaluator(self, name: str, evaluator: Callable):
        """注册评估器"""
        self.evaluators.append((name, evaluator))
    
    def register_mock(self, tool_name: str, mock_func: Callable):
        """注册工具 Mock"""
        self.mocks[tool_name] = mock_func
    
    def run_task(self, task: Task) -> TaskResult:
        """执行单个任务"""
        # 初始化环境
        self.environment.reset(task.initial_state)
        
        # 运行 Agent
        steps = []
        for i in range(task.max_steps):
            # 获取当前状态
            current_state = self.environment.get_state()
            
            # Agent 决策
            action = self.agent.decide(current_state)
            
            # 执行动作（可能使用 mock）
            if action.tool in self.mocks:
                result = self.mocks[action.tool](action.args)
            else:
                result = self.environment.execute(action)
            
            steps.append({
                "step": i,
                "action": action.to_dict(),
                "result": result
            })
            
            # 检查是否完成
            if self._check_goal(current_state, task.goal_state):
                return TaskResult(
                    task_id=task.id,
                    status=TaskStatus.SUCCESS,
                    steps=steps,
                    final_state=current_state,
                    metrics=self._compute_metrics(steps)
                )
        
        # 超过最大步数
        return TaskResult(
            task_id=task.id,
            status=TaskStatus.FAILURE,
            steps=steps,
            final_state=self.environment.get_state(),
            metrics=self._compute_metrics(steps)
        )
    
    def run_suite(self, tasks: list[Task]) -> list[TaskResult]:
        """运行测试套件"""
        results = []
        for task in tasks:
            result = self.run_task(task)
            results.append(result)
        return results
    
    def evaluate(self, result: TaskResult) -> dict:
        """运行所有评估器"""
        eval_results = {}
        for name, evaluator in self.evaluators:
            eval_results[name] = evaluator(result)
        return eval_results
```

**2. 环境管理**
```python
class EnvironmentManager:
    """管理 Agent 运行环境"""
    
    def __init__(self):
        self.state = {}
        self.tools = {}
        self.history = []
    
    def reset(self, initial_state: dict):
        """重置环境"""
        self.state = initial_state.copy()
        self.history = []
    
    def register_tool(self, name: str, func: Callable):
        """注册可用工具"""
        self.tools[name] = func
    
    def execute(self, action) -> Any:
        """执行动作"""
        if action.tool not in self.tools:
            raise ValueError(f"Unknown tool: {action.tool}")
        
        result = self.tools[action.tool](**action.args)
        self.history.append({
            "action": action,
            "result": result
        })
        return result
    
    def get_state(self) -> dict:
        """获取当前状态"""
        return self.state.copy()
```

**3. 评估器实现**
```python
class Evaluators:
    """常用评估器集合"""
    
    @staticmethod
    def success_rate(results: list[TaskResult]) -> float:
        """成功率"""
        success = sum(1 for r in results if r.status == TaskStatus.SUCCESS)
        return success / len(results)
    
    @staticmethod
    def avg_steps(results: list[TaskResult]) -> float:
        """平均步数"""
        return sum(len(r.steps) for r in results) / len(results)
    
    @staticmethod
    def tool_correctness(expected_calls: dict) -> Callable:
        """工具调用正确性"""
        def evaluator(result: TaskResult) -> float:
            correct = 0
            total = 0
            for step in result.steps:
                tool = step["action"]["tool"]
                if tool in expected_calls:
                    total += 1
                    if step["result"]["success"]:
                        correct += 1
            return correct / total if total > 0 else 1.0
        return evaluator
    
    @staticmethod
    def llm_judge(prompt_template: str, model: str = "gpt-4") -> Callable:
        """LLM-as-Judge 评估器"""
        def evaluator(result: TaskResult) -> dict:
            from openai import OpenAI
            client = OpenAI()
            
            prompt = prompt_template.format(
                task=result.task_id,
                steps=json.dumps(result.steps),
                final_state=json.dumps(result.final_state)
            )
            
            response = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}]
            )
            
            return {"judge_response": response.choices[0].message.content}
        return evaluator
```

**4. 完整使用示例**
```python
# 创建 Agent
agent = MyAgent(model="gpt-4")

# 创建环境
env = EnvironmentManager()
env.register_tool("search", mock_search)
env.register_tool("calculator", mock_calculator)

# 创建 Harness
harness = AgentHarness(agent, env)

# 注册评估器
harness.register_evaluator("tool_correctness", 
    Evaluators.tool_correctness({"search": {}, "calculator": {}}))
harness.register_evaluator("llm_quality",
    Evaluators.llm_judge(JUDGE_PROMPT))

# 定义任务
tasks = [
    Task(
        id="math_search_001",
        description="计算 2024年世界杯冠军所在国家的人口",
        initial_state={},
        goal_state={"answer": "约2.1亿"},
        tools_available=["search", "calculator"]
    ),
    # ... 更多任务
]

# 运行测试
results = harness.run_suite(tasks)

# 评估结果
for result in results:
    eval_result = harness.evaluate(result)
    print(f"Task {result.task_id}: {result.status}")
    print(f"Evaluations: {eval_result}")
```

### 4.2 真实场景最佳实践

#### 4.2.1 LangSmith 实践
```yaml
# langsmith_config.yaml
evaluation:
  evaluators:
    - name: correctness
      type: llm
      prompt: |
        Evaluate if the agent's response correctly answers the question.
        Response: {response}
        Question: {question}
        Expected: {reference}
    - name: tool_usage
      type: custom
      function: check_tool_sequence
```

#### 4.2.2 自动化测试流水线
```yaml
# .github/workflows/agent_test.yml
name: Agent Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up environment
        run: |
          docker-compose -f docker-compose.test.yml up -d
      
      - name: Run agent tests
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          python -m pytest tests/agents/ --harness
      
      - name: Generate report
        run: |
          python scripts/generate_report.py
      
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: agent-test-results
          path: results/
```

#### 4.2.3 持续评估
```python
class ContinuousEvaluation:
    """持续评估系统"""
    
    def __init__(self, harness, baseline_results):
        self.harness = harness
        self.baseline = baseline_results
        self.alert_thresholds = {
            "success_rate": 0.9,  # 不低于基线的 90%
            "avg_steps": 1.2,     # 不超过基线的 120%
        }
    
    def evaluate_changes(self, new_agent):
        """评估 Agent 变更"""
        new_results = self.harness.run_suite(self.tasks)
        
        comparison = {
            "success_rate": {
                "baseline": self.baseline["success_rate"],
                "current": self.compute_success_rate(new_results),
            },
            "avg_steps": {
                "baseline": self.baseline["avg_steps"],
                "current": self.compute_avg_steps(new_results),
            }
        }
        
        # 检测回归
        alerts = self._check_regressions(comparison)
        return {"comparison": comparison, "alerts": alerts}
    
    def _check_regressions(self, comparison):
        alerts = []
        
        # 成功率下降检查
        if comparison["success_rate"]["current"] < \
           self.baseline["success_rate"] * self.alert_thresholds["success_rate"]:
            alerts.append({
                "type": "regression",
                "metric": "success_rate",
                "message": "Success rate dropped significantly"
            })
        
        return alerts
```

---

## 五、总结与建议

### 5.1 关键要点

1. **Agent Harness ≠ 传统测试框架**
   - 需要处理非确定性
   - 需要管理复杂环境
   - 需要多维度评估

2. **核心挑战**
   - 非确定性输出
   - 多轮交互评估
   - 工具调用验证
   - 长时任务效率

3. **主流方案**
   - 多次采样 + 统计
   - Mock/Sandbox 环境
   - LLM-as-Judge
   - 检查点评估

### 5.2 选型建议

| 场景 | 推荐工具 |
|------|----------|
| 快速原型验证 | LangSmith + 自建测试 |
| 学术研究 | AgentBench, HELM |
| 生产环境 | LangSmith/TruLens + CI/CD |
| RAG 系统 | Ragas + LangSmith |
| 安全评估 | Promptfoo + 自建红队 |

### 5.3 发展趋势

1. **标准化**: Agent Benchmarks 正在标准化（如 AgentBench）
2. **自动化**: 自动生成测试用例
3. **可观测性**: 更好的追踪和调试能力
4. **安全评估**: 红队测试和对抗性评估
5. **人机协同**: 人在回路（HITL）的评估方法

---

## 六、参考资源

### 论文
1. "AgentBench: Evaluating LLMs as Agents" (2023)
2. "WebArena: A Realistic Web Environment for Building Autonomous Agents" (2023)
3. "ToolBench: An Open Platform for Large Language Model Tool Learning" (2023)
4. "Evaluating Language Model Agents on Realistic Tasks" (2024)

### 开源项目
- lm-evaluation-harness: https://github.com/EleutherAI/lm-evaluation-harness
- HELM: https://github.com/stanford-crfm/helm
- LangSmith: https://www.langchain.com/langsmith
- TruLens: https://github.com/truera/trulens
- Ragas: https://github.com/explodinggradients/ragas
- AgentBench: https://github.com/THUDM/AgentBench

### 博客文章
- LangChain Blog: "Evaluating LLM Applications"
- Anthropic Blog: "Building reliable agents"
- OpenAI Cookbook: "How to evaluate LLM applications"

---

*报告完成于 2026-03-19*