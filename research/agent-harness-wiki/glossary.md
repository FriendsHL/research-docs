---
name: glossary
type: meta
updated: 2026-05-18
status: stable
---

# Glossary — 概念对齐(23 术语)

> **存在意义**:不对齐会让每次对话浪费 5-10 分钟重新定义,而且容易聊串。这里固化我跟用户的术语共识。
>
> **格式**:每个术语 4 段 — _称呼 / 定义 / vs 相邻 / wiki 出现位置_。
> **高频混淆配对**(agent vs harness vs framework / dataset 四件套 / system prompt vs rules / metric vs dimension / score vs label / attribution vs label vs score / eval vs benchmark)做扩展对照。
>
> **更新原则**:遇到新术语先来这里查 → 没有就来这里加 → 再用。不要在对话里临时定义。

---

## A. Agent 运行抽象(4)

### agent
**中文**:智能体 | **English**:agent / agentic | **我固定叫**:agent | **缩写**:无

**定义**:_可以自主选 tool、执行、循环、调整_ 的 LLM 驱动执行体。区别于一次性 LLM call:有 loop + 状态 + 工具使用。

**vs 相邻概念**:
- vs **harness**:agent = _执行体本身_(LLM + loop + 决策);harness = _承载 agent 运行的整套基础设施_(provider、context、hooks、permission)
- vs **framework**:framework = 编写 agent 用的库(代码层);agent = 跑起来的实体
- vs **skill** / **tool**:agent 主动 invoke skill/tool;skill/tool 是被 invoke 的被动能力

**wiki 出现**:每个 [[harness/*]] frontmatter `agent_loop` / [[concept/agent-harness]] 10 维 cross-harness 设计

### harness
**中文**:运行壳 / 框架运行时 | **English**:harness | **我固定叫**:harness(不翻译)| **缩写**:无

**定义**:**承载 agent 运行的整套基础设施** — 包含 LLM provider、context_mgmt、permission、hooks、persistence 等。是 wiki 命名的核心词。

**vs 相邻概念**(高频混淆,扩展对照):

| | **agent** | **harness** | **framework** |
|---|---|---|---|
| 是什么 | 跑起来的执行体 | 承载 agent 的基础设施 | 写代码用的库 |
| 例子 | "Claude Code 的 build agent" | "Claude Code 这个 harness" | "LangChain 这个 framework" |
| wiki 路径 | (实例,不单独成页) | `harness/*.md` 13 个 | `eval/*.md type:framework` 5 个 + `harness/*.md` 部分 |
| 我的用法 | 偶尔(实例化时) | **最常用**(讨论运行时设计) | 提到代码层 / Python 包 时 |

**wiki 出现**:**wiki 本名 `agent-harness-wiki`** / `harness/` 子目录 13 页

### framework
**中文**:框架 | **English**:framework | **我固定叫**:framework | **缩写**:无

**定义**:**写 agent / RAG / memory 用的代码库** — 通常是 PyPI / npm package,用户 import 后写自己的 agent。强调 _代码层_。

**vs 相邻概念**:
- vs **harness**:harness = 跑 agent 的 _基础设施_;framework = 写 agent 的 _代码库_(虽然两者经常重叠,如 LangGraph 既是 framework 又能当 harness)
- 在 wiki 内 [[schema]] §3.2b 加了 `type: framework` 专指 _eval/* 内 5 个 memory/RAG/multi-agent 库_(ag2 / cognee / llama_index / ms-graphrag / pathway)

**wiki 出现**:[[schema]] §3.2b / [[overview]] §1 eval 6 路线分类第 6 行

### hook
**中文**:钩子 | **English**:hook / lifecycle event | **我固定叫**:hook | **缩写**:无

**定义**:**framework 在生命周期关键点暴露给用户/plugin 的回调插槽**(PreToolUse / PostToolUse / SessionStart / ...)。让用户在不改 framework 代码的前提下 _介入_ 流程。

**vs 相邻概念**:
- vs **callback**:callback 是程序术语(任意位置回调);hook 是 framework 设计模式(预定义点位)
- vs **plugin** / **extension**:plugin = 一组 hook + 代码;hook = plugin 的接入点

**wiki 出现**:[[concept/hooks-design]](阶梯 Codex 8 < goose 11 < Hermes 16 < Claude Code 27)

---

## B. Tool 调用与协议(2)

### tool
**中文**:工具 | **English**:tool | **我固定叫**:tool | **缩写**:无

**定义**:**LLM 通过 tool_use(function calling)协议调用的单次函数**。输入 schema 化(JSON Schema),输出回 tool_result。是 agent 跟外界交互的 _原子动作_。

**vs 相邻概念**:
- vs **skill**:tool = _单次函数调用_,无状态,LLM 主动 invoke;skill = _多步任务包_ + 文档,framework 加载到上下文
- vs **action**(MetaGPT 概念):MetaGPT `Action` ≠ `Tool` — Action = role 职责级抽象,Tool = 底层执行单元
- vs **MCP tool**:同样语义,但 MCP tool 通过 MCP 协议暴露(跨 framework 兼容)

**wiki 出现**:每个 [[harness/*]] `tool_protocol` 维度 / [[concept/code-mode-orchestration]](Code Mode vs JSON tool_use 对照)

### MCP
**中文**:模型上下文协议 | **English**:Model Context Protocol | **我固定叫**:MCP | **缩写**:**MCP**(已固定术语,不展开)

**定义**:**Anthropic 2024 提出的 tool / resource / prompt 跨 framework 调用标准协议**。已成事实标准(18+ wiki 集成)。6 primitives 对称:Server 3(Resources/Prompts/Tools)+ Client 3(Sampling/Roots/Elicitation)。

**vs 相邻概念**:
- vs **A2A**:MCP = _tool 层_(LLM 调工具);A2A = _agent 层_(agent 调 agent)
- vs **OpenAPI / gRPC**:MCP 是 _LLM-friendly_ schema(自带 description / 自适配 prompt);OpenAPI 偏 REST

**wiki 出现**:[[concept/agent-protocols]] §2 / [[deep/mcp-spec-deep]] / 多个 harness 内置 MCP server(如 [[harness/goose]] 5 内置 + [[harness/ragflow]] / [[harness/dify]])

---

## C. Skill 加载体系(1)

### skill
**中文**:技能包 | **English**:skill / Skill | **我固定叫**:skill(项目内或 Anthropic Skill 标准)| **缩写**:无

**定义**:**LLM 可加载的可复用任务包**(prompt + 关联文件 + metadata),framework 根据 trigger 自动加载到上下文。Anthropic Skills SDK 标准的产物(`.claude/skills/<name>/SKILL.md`)。

**vs 相邻概念**(高频混淆):
- vs **tool**:tool = 单次函数调用,LLM 主动 invoke;skill = 多步任务包,framework 把 SKILL.md 加载进 prompt 给 LLM
- vs **command**:command = slash(`/xxx`)触发的入口;command 内部可以 invoke skill。command 是 _用户主动调_,skill 是 _LLM 根据描述自动加载_
- vs **agent**:agent = 主动 loop 执行体;skill = 被动加载的能力包(无 loop)
- vs **plugin / extension**:plugin 是上一代术语,Skills 是 Anthropic 2025 重新设计的 _首选加载机制_

**wiki 出现**:[[harness/claude-code]] / [[harness/codex]] `skills` crate / [[harness/opencode]] `tool/skill.ts` / [[harness/goose]] `skills/`(待 verify)/ wiki-* 4 个 skill 在 `.claude/skills/`(2026-05-17 commit `7ca9a65`)

---

## D. Prompt & Rules(2,高频混淆)

### system prompt
**中文**:系统提示词 | **English**:system prompt / system message | **我固定叫**:system prompt | **缩写**:**SP**(偶尔)

**定义**:**LLM API 调用时传的 `system` role 消息**,设定 agent 的 _身份、能力、风格、约束_。是 LLM 协议层的固定字段(Anthropic API `system` 参数 / OpenAI API `system` message)。

**vs 相邻概念**(扩展对照):

| | **system prompt** | **rules** | **instructions** |
|---|---|---|---|
| 谁定义 | framework / 开发者 | 项目维护者 / 用户 | 用户(per session)|
| 加载方式 | API 参数(每次调用都传)| 项目文件(`.claude/rules/`,被 framework 注入 system prompt 或 user 消息)| 临时 prompt 输入 |
| 生效粒度 | session 级 | 项目级 / 全局 | turn 级 |
| 例子 | "You are Claude Code..." 数千 token | `.claude/rules/code-style.md` 项目编码风格 | 用户在 chat 内说 "用中文回复" |
| 修改成本 | framework 改代码 | 编辑文件 + 下次 session 生效 | 当场说一句 |

**wiki 出现**:[[concept/runtime-context-injection]](system-reminder 协议)/ [[harness/claude-code]] / [[deep/claude-code-compact-tool-result]]

### rules
**中文**:规则 / 项目约定 | **English**:rules | **我固定叫**:rules | **缩写**:无

**定义**:**项目级或全局的约束性指令**,通常以文件形式存放(`.claude/rules/*.md` / `.cursor/rules/` / `AGENTS.md`),被 framework 在合适时机注入 system prompt 或 user 消息。**比 system prompt 颗粒度细**,**比 instructions 持久**。

**vs 相邻概念**:
- vs **system prompt**:见上表
- vs **AGENTS.md / CLAUDE.md**:这两个是 _顶层 rules 文档_ 的具体实例(被 framework 自动加载)
- vs **skill**:rules 是 _被动约束_(always-on);skill 是 _按需加载_(trigger-based)

**wiki 出现**:[[harness/claude-code]] `.claude/rules/` / [[harness/codex]] `AGENTS.md` 17KB / [[concept/agent-runtime-stacks]] §`.agents/` dogfood 判断 7

---

## E. Eval 数据(5,高频混淆)

### dataset
**中文**:数据集 | **English**:dataset | **我固定叫**:dataset | **缩写**:无

**定义**:**一组样本的集合**,每个样本含 input + 期望输出(ground truth)。是 _笼统父概念_,具体分 training / dev / test / eval 4 个子集。

**vs 相邻概念**(扩展对照,最高频混淆):

| | **training set** | **dev set / validation set** | **test set** | **eval set** |
|---|---|---|---|---|
| 用途 | 训模型用(梯度更新)| 调超参 / 选模型(model selection)| 最后报告性能(model evaluation)| **生产中持续评测 agent 行为**(deployment monitoring)|
| 数据可见性 | 模型训练时看过 | 调参时反复看 | **训练时严禁看**(否则数据泄露)| 通常是 production trace 子集 |
| 何时用 | 训练阶段 | 训练后 / 部署前 | 部署前 release gate | **持续(production loop)** |
| 在 LLM agent 语境 | _基模型训练用,跟我们无关_ | _SkillForge prompt A/B 选 prompt_ | _benchmark 跑 SWE-bench 用_ | _Langfuse `addDatasetRunItemsToEvalQueue` production loop_ |
| 我的用法 | 几乎不用 | 偶尔(prompt 选优时)| 偶尔(跑 benchmark 时)| **最常用**(SkillForge 场景默认指这个) |

**wiki 出现**:[[eval/swe-bench]](test set 典型)/ [[eval/langfuse]] `addDatasetRunItemsToEvalQueue` production loop(eval set 典型)

### training set
**中文**:训练集 | **English**:training set | **我固定叫**:training set | **缩写**:**train**(偶尔)

**定义**:见上表。在 LLM agent 语境通常不用(基模型由 Anthropic / OpenAI 训,我们不动)。

**vs 相邻概念**:见 [[#dataset]] 扩展对照表

**wiki 出现**:几乎不用,偶尔在 [[harness/agentscope]] Trinity-RFT(agentic RL training)讨论训练 dataset 时出现

### test set
**中文**:测试集 | **English**:test set | **我固定叫**:test set | **缩写**:无

**定义**:见 [[#dataset]] 表 — _训练前留出 / 训练时不看 / 最后报告性能_ 的子集。在我们语境 ≈ benchmark suite(SWE-bench 500 题 / tau-bench retail+airline)。

**vs 相邻概念**:见 [[#dataset]] 扩展对照表。**关键纪律**:test set 不应被反复刷分调整(否则等于变 dev set,失去测试意义)

**wiki 出现**:[[eval/swe-bench]] 4 变体 / [[eval/tau-bench]]

### eval set
**中文**:评测集 | **English**:eval set / evaluation set | **我固定叫**:eval set | **缩写**:无

**定义**:**生产中持续评测 agent 行为用的样本集**。通常是 production trace 的人工或自动 curation 子集,**持续追加** + **定期回归测试**。是 _LLM agent 语境最常说的"评测数据"_。

**vs 相邻概念**:见 [[#dataset]] 扩展对照表。**关键区别**:test set 一次性,eval set 持续演化

**wiki 出现**:[[eval/langfuse]] `addDatasetRunItemsToEvalQueue` production loop / [[concept/agent-eval]] / [[concept/trace-based-eval]]

### ground truth
**中文**:真实标签 / 真值 | **English**:ground truth | **我固定叫**:ground truth / GT | **缩写**:**GT**

**定义**:**样本对应的 _正确答案_**,是 _算 metric 的基准_。在 LLM agent 场景常以多种形式存在:执行结果(test 通过/失败)/ 人工标注 / 强模型(GPT-5)输出参考。

**vs 相邻概念**:
- vs **label**:label 是 _标的过程产物_(可能是 GT,也可能是 annotation 的中间产物);GT 强调 _"被视为正确"_
- vs **prediction**:prediction = LLM 输出,跟 GT 算 metric

**wiki 出现**:[[concept/agent-eval]] / [[eval/swe-bench]](测试用例执行结果作 GT)

---

## F. Eval 打分(7)

### eval / evaluation
**中文**:评测 | **English**:eval / evaluation | **我固定叫**:eval | **缩写**:无

**定义**:**给 agent / prompt / model 打分的过程**。可以是离线 benchmark,也可以是 production trace 上的持续评估。是 _动词_(跑 eval)和 _名词_(eval framework)双用。

**vs 相邻概念**:
- vs **benchmark**:benchmark = _固定 test set + 标准跑法_(SWE-bench);eval = _更笼统_,含 benchmark + production eval + LLM-as-judge

**wiki 出现**:[[concept/agent-eval]] 8 维设计空间 / `eval/` 16 + `framework/` 5 子目录

### benchmark
**中文**:基准测试 | **English**:benchmark | **我固定叫**:benchmark | **缩写**:无

**定义**:**业内公认的固定 test set + 标准跑法 + 公开 leaderboard**(SWE-bench / tau-bench / LongMemEval / MMLU)。是 eval 的子集,但有 _跨家可比_ 的特殊地位。

**vs 相邻概念**:
- vs **eval**:benchmark ⊂ eval(benchmark 是 _最严的 eval 形式_)
- vs **eval set**:benchmark 是 _一次性 test set_,eval set 是 _持续演化_;benchmark 用来跨家比,eval set 用来 production 监控

**wiki 出现**:[[eval/swe-bench]] / [[eval/tau-bench]] / [[harness/metagpt]] AFlow ICLR 2025 / [[eval/opik]] 8 内置 benchmark dataset

### metric
**中文**:指标 | **English**:metric | **我固定叫**:metric | **缩写**:无

**定义**:**衡量 agent / model 性能的 _量化函数_**(accuracy / faithfulness / latency / cost / token)。每个 metric 输出一个数值。

**vs 相邻概念**(扩展对照,metric vs dimension 高频混淆):

| | **metric** | **dimension** |
|---|---|---|
| 是什么 | 一个 _量化函数_(输出数值) | 一个 _评估角度_(下面挂多个 metric)|
| 例子 | `accuracy = 0.87` / `latency = 250ms` | "performance" 维度 / "cost" 维度 / "safety" 维度 |
| 关系 | metric ⊂ dimension(_一个 dimension 下有多个 metric_)| dimension 是 metric 的 _组织框架_ |
| wiki 例 | Ragas 27 metric | [[concept/agent-eval]] 8 维 / [[harness/*]] 13 维 frontmatter |
| 我的用法 | 算数值时 | 设计评估框架时 |

**wiki 出现**:[[eval/ragas]] 27 metric / [[eval/llama_index]] 13 metric / [[concept/agent-eval]]

### dimension
**中文**:维度 | **English**:dimension | **我固定叫**:dimension / 维 | **缩写**:无

**定义**:**评估 / 设计 / 分析的角度**,是 metric / 子项的 _组织框架_。注意 wiki 还有第二种用法:_13 维 harness frontmatter_(设计维度,非评估维度,但同概念)。

**vs 相邻概念**:见 [[#metric]] 扩展对照表。**警告**:wiki 内 "13 维度" 既指 _harness 设计角度_(agent_loop / tool_protocol / ... 13 个 frontmatter 字段)又借用了 dimension 一词,容易混。在 _评估语境_ 才指 evaluation dimension。

**wiki 出现**:[[concept/agent-eval]] 8 维 / [[schema]] §3.1 13 维 frontmatter

### score
**中文**:分数 / 打分 | **English**:score / scoring | **我固定叫**:score(名词)/ scoring(动词)| **缩写**:无

**定义**:**metric 在某个样本上的输出值**(数值或分类)。score 通常带 source(API / EVAL / ANNOTATION)+ dataType(NUMERIC / CATEGORICAL / BOOLEAN / TEXT)。

**vs 相邻概念**(高频混淆 score vs label):
- vs **label**:**score 强调 _数值化_**(0.87 / pass);**label 强调 _离散分类标签_**(positive / negative / hallucinated)。但很多场景两者重合 — Langfuse Score 数据模型把 5 种 dataType 都叫 score(NUMERIC / CATEGORICAL / BOOLEAN / **CORRECTION** / **TEXT**)
- vs **metric**:score = metric 函数的 _输出_;metric = score 的 _算法_

**wiki 出现**:[[eval/langfuse]] Score 3×5 矩阵 / [[eval/phoenix]] `Score.explanation` 业内最规范

### label / annotation
**中文**:标注 / 打标 | **English**:label / annotation | **我固定叫**:label(单数动词)/ annotation(过程或产物)| **缩写**:无

**定义**:**给样本 _人工或 LLM 加标签_ 的过程或产物**。typically discrete categories(spam/not-spam / hallucinated/factual / level-1/2/3)。是 eval 数据准备的关键步骤。

**vs 相邻概念**:
- vs **score**:见 [[#score]] 对照
- vs **ground truth**:label 通常 _被采用为 GT_,但 GT 强调 _"正确性被认可"_,label 强调 _"标注动作"_
- vs **annotation**(细分):多人标 + 一致性检验(Cohen's κ)后才升 GT;单人标的叫 annotation 更准

**wiki 出现**:[[concept/agent-eval]] / [[eval/langfuse]] Score `ANNOTATION` source 三态之一

### judge / LLM-as-judge
**中文**:评判器 / LLM 风评判 | **English**:judge / LLM-as-judge | **我固定叫**:LLM-as-judge(完整)/ judge(简称)| **缩写**:无

**定义**:**用 LLM 给 agent 输出打分 / 标注的方法**。比人工标快/便宜,比 rule-based 灵活,但有 bias 和 hallucination 风险。配 rubric(评分细则)使用效果好。

**vs 相邻概念**:
- vs **human annotation**:LLM-as-judge = 自动,human = 慢但贵且权威;production 通常混合(LLM 先打分,人审 disagreement case)
- vs **rule-based scoring**:rule-based(regex / exact match)适合 deterministic;LLM-as-judge 适合开放性输出
- vs **evaluator**:evaluator 是 _框架抽象_(可以是 LLM judge / rule / human);LLM-as-judge 是 _具体方法_

**wiki 出现**:[[eval/ragas]] AspectCritic / [[eval/phoenix]] `ClassificationEvaluator include_explanation=True` / [[eval/promptfoo]] `llmRubric` / [[concept/attribution]] Layer B

---

## G. 高阶概念(2)

### attribution
**中文**:归因 | **English**:attribution | **我固定叫**:attribution | **缩写**:无

**定义**:**把 _某个观察到的现象 / 分数_ 归因到 _某个原因 / 行为 / 个体_ 的过程**。LLM agent 场景至少 3 种细分:trace attribution(行为追溯)/ fault attribution(失败归因到 USER/AGENT/ENV)/ score attribution(score 归因到具体 reasoning step)。

**vs 相邻概念**(扩展对照 — attribution / label / score / provenance 易混):

| | **attribution** | **label** | **score** | **provenance** |
|---|---|---|---|---|
| 回答的问题 | _为什么_ 这样 / _谁_ 造成的 | _这个是什么类_ | _这个有多好_ | _数据从哪来_ |
| 时间方向 | 反向追溯 | 当前分类 | 当前打分 | 反向追溯(数据流)|
| 例子 | "失败是因为 ENV bug,不是 agent" | "这个 trace 标 hallucinated" | "answer score 0.87" | "这个 fact 来源 source X line 42" |
| wiki 出现 | [[concept/attribution]] 5 Layer + 7 步 | 同上 Layer A | 同上 Layer B + [[eval/langfuse]] | [[concept/agent-memory]] Graphiti episode |

**wiki 出现**:[[concept/attribution]] 专题(5 Layer + 7 步)/ [[eval/tau-bench]] `auto_error_identification.py` `FaultAuthor × FaultType`(业内 benchmark 首批自动归因)

### A/B test
**中文**:A/B 测试 | **English**:A/B test / A/B testing | **我固定叫**:A/B test | **缩写**:**A/B**

**定义**:**同时跑 _对照组(A,baseline)_ 跟 _实验组(B,候选)_,根据 metric 决定胜出**。是 _部署决策方法_,跟 eval 配合使用。

**vs 相邻概念**:
- vs **eval**:eval 算 score;A/B test 用 eval 结果 _做决策_(选 A 还是 B 上线)
- vs **champion-challenger**:同义(champion = 当前胜出方 / challenger = 候选);A/B 更普遍 / champion-challenger 更强调持续迭代
- vs **shadow / canary**:都是 _部署对照模式_,但 shadow = 候选不影响用户(只录响应)/ canary = 候选放小流量 / A/B = 50-50 切流量,各有适用场景

**wiki 出现**:[[overview]] §3 SkillForge 已领先的(_Eval Pipeline + Prompt A/B + 自动晋升_)/ [[eval/langfuse]] EvalTemplate 版本控制

---

## 不在本表的相邻术语(指针)

如果遇到下面这些常出现的术语,**先去其专题概念页查**,不在 glossary 重复:

- **MCP 6 primitives / A2A / AG-UI / OTel** → [[concept/agent-protocols]] 5 层栈
- **SubAgent / fork / handoff / orchestrator** → [[concept/multi-agent-isolation]] 5 路线
- **memory / KG-memory / memory block / episode** → [[concept/agent-memory]] 7 路线
- **RAG / retriever / embedding / chunk / reranker / KG** → [[concept/rag-design-space]] 8 维 / [[concept/agent-memory]]
- **sandbox / permission / guardrail / prompt injection / egress** → [[concept/sandbox-isolation]] 6 阶梯 / [[harness/goose]] security 双向防御
- **trace / span / log / Wide Events 2.0** → [[concept/observability]] / [[concept/trace-based-eval]]
- **self-evolution / prompt optimization / reflection** → [[concept/self-evolution]] 4 类 / [[concept/prompt-optimization]] 6 维
- **agent execution paradigm(ReAct / Event-driven / Actor / Pregel)** → [[concept/agent-runtime-stacks]] §Agent Execution Paradigm

---

## 变更日志

- **2026-05-18**:v1 创建 — 23 术语 × 7 区,基于用户列的 15 个核心 + 8 个高频相邻(agent / harness / framework / hook / benchmark / dimension / judge / system prompt extension)
