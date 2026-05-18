---
name: glossary
type: meta
updated: 2026-05-18
status: stable
---

# Glossary — 概念对齐(101 术语 × 11 区)

> **存在意义**:不对齐会让每次对话浪费 5-10 分钟重新定义,而且容易聊串。这里固化我跟用户的术语共识。
>
> **格式**:每个术语 4 段 — _称呼 / 定义 / vs 相邻 / wiki 出现位置_。
> **高频混淆配对**(agent vs harness vs framework / dataset 四件套 / system prompt vs rules / metric vs dimension / score vs label / attribution vs label vs score / eval vs benchmark)做扩展对照表。
>
> **更新原则**:遇到新术语先来这里查 → 没有就来这里加 → 再用。不要在对话里临时定义。

## 11 区目录

| 区 | 内容 | 数量 |
|---|---|---|
| [A. Agent 运行抽象](#a-agent-运行抽象12) | agent / harness / framework / runtime / SDK / provider / channel / session / turn / step / loop / agentic | 12 |
| [B. Tool 调用与协议](#b-tool-调用与协议9) | tool / tool_use / tool_result / MCP / A2A / ACP / AG-UI / AGNTCY / OTel | 9 |
| [C. Skill 加载体系](#c-skill-加载体系6) | skill / extension(plugin)/ command / hook / lifecycle event / recipe | 6 |
| [D. Context & 提示](#d-context--提示11) | context window / system prompt / system-reminder / rules / instructions / prompt / template / few-shot / prompt cache / compaction / summary | 11 |
| [E. Memory](#e-memory6) | memory / short-term / long-term / KG-memory / memory block / episode | 6 |
| [F. Eval / Benchmark](#f-eval--benchmark21) | eval / evaluator / benchmark / dataset / training set / dev set / test set / eval set / hold-out / ground truth / label / score / metric / dimension / rubric / assertion / judge / pass@k / golden / A/B test / champion-challenger | 21 |
| [G. Multi-agent](#g-multi-agent8) | multi-agent / SubAgent / fork / handoff / role / team / orchestrator / swarm | 8 |
| [H. RAG & 检索](#h-rag--检索7) | RAG / retriever / embedding / vector store / chunk(chunking)/ reranker / KG | 7 |
| [I. Safety & Attribution](#i-safety--attribution10) | safety / permission / sandbox / guardrail / moderation / prompt injection / egress / red team / attribution / provenance | 10 |
| [J. Observability](#j-observability5) | trace / span / log / Wide Events 2.0 / OTel semconv | 5 |
| [K. Self-evolution](#k-self-evolution6) | self-evolution / fine-tune / prompt optimization / reflection / RL(RLHF/RLAIF)/ distillation | 6 |
| **总计** | | **101** |

---

## A. Agent 运行抽象(12)

### agent
**中文**:智能体 | **English**:agent | **我固定叫**:agent | **缩写**:无

**定义**:_可以自主选 tool、执行、循环、调整_ 的 LLM 驱动执行体。区别于一次性 LLM call:有 loop + 状态 + 工具使用。

**vs 相邻**:
- vs **harness**:agent = _执行体本身_(LLM + loop + 决策);harness = _承载 agent 运行的整套基础设施_(provider、context、hooks、permission)
- vs **framework**:framework = 编写 agent 用的库(代码层);agent = 跑起来的实体
- vs **skill** / **tool**:agent 主动 invoke skill/tool;skill/tool 是被 invoke 的被动能力

**wiki**:每个 [[harness/*]] frontmatter `agent_loop` / [[concept/agent-harness]] 10 维 cross-harness 设计

### agentic
**中文**:智能体式的(形容词)| **English**:agentic | **我固定叫**:agentic | **缩写**:无

**定义**:形容词,描述 _具备 agent 特性_ 的系统/行为(自主、循环、工具使用、目标导向)。常用搭配:agentic AI / agentic loop / agentic workflow / agentic RL。

**vs 相邻**:agent 是名词(具体实体);agentic 是形容词(强调 _性质_)。"agentic workflow" ≠ "agent workflow",前者强调 _有 agent 特质_(可循环 / 工具使用)而非简单线性。

**wiki**:[[harness/agentscope]] Trinity-RFT _"agentic RL training"_(业内独家)/ [[backlog]] §3 _"agentic 评测集合"_

### harness
**中文**:运行壳 / 框架运行时 | **English**:harness | **我固定叫**:harness(不翻译)| **缩写**:无

**定义**:**承载 agent 运行的整套基础设施** — 包含 LLM provider、context_mgmt、permission、hooks、persistence 等。是 wiki 命名的核心词。

**vs 相邻**(高频混淆,扩展对照):

| | **agent** | **harness** | **framework** |
|---|---|---|---|
| 是什么 | 跑起来的执行体 | 承载 agent 的基础设施 | 写代码用的库 |
| 例子 | "Claude Code 的 build agent" | "Claude Code 这个 harness" | "LangChain 这个 framework" |
| wiki 路径 | (实例,不单独成页) | `harness/*.md` 13+ 个 | `eval/*.md type:framework` 5 个 + `harness/*.md` 部分 |
| 我的用法 | 偶尔(实例化时) | **最常用**(讨论运行时设计) | 提到代码层 / Python 包 时 |

**wiki**:**wiki 本名 `agent-harness-wiki`** / `harness/` 子目录 14 页

### framework
**中文**:框架 | **English**:framework | **我固定叫**:framework | **缩写**:无

**定义**:**写 agent / RAG / memory 用的代码库** — 通常是 PyPI / npm package,用户 import 后写自己的 agent。强调 _代码层_。

**vs 相邻**:
- vs **harness**:harness = 跑 agent 的 _基础设施_;framework = 写 agent 的 _代码库_(虽然两者经常重叠,如 LangGraph 既是 framework 又能当 harness)
- 在 wiki 内 [[schema]] §3.2b 加了 `type: framework` 专指 _eval/* 内 5 个 memory/RAG/multi-agent 库_(ag2 / cognee / llama_index / ms-graphrag / pathway)

**wiki**:[[schema]] §3.2b / [[overview]] §1 eval 6 路线分类第 6 行

### runtime
**中文**:运行时 | **English**:runtime | **我固定叫**:runtime | **缩写**:无

**定义**:**实际跑 agent / framework / 代码的执行环境**。可指 _进程级_(Python interpreter / V8 isolate / Rust binary)或 _agent 级_(Claude Code runtime / Codex runtime)。

**vs 相邻**:
- vs **harness**:harness 包含 runtime,但还加 context_mgmt / permission / hooks;runtime 更窄,只指执行机制
- vs **framework**:framework 是写的代码,runtime 是跑的环境

**wiki**:[[concept/agent-runtime-stacks]] 5 编程哲学 + 4 paradigm / [[harness/codex]] `otel/` crate _"runtime self-contained"_

### SDK
**中文**:软件开发工具包 | **English**:SDK (Software Development Kit) | **我固定叫**:SDK | **缩写**:**SDK**

**定义**:**框架对外暴露的程序化集成层**(Python `import xxx` / TS `import xxx` / Rust crate)。用户用 SDK 跟 framework 交互,而不是直接读源码。

**vs 相邻**:
- vs **framework**:framework 是 _整体_,SDK 是 framework 暴露的 _接口集_
- vs **API**:API 通常指 _远程网络接口_(REST / gRPC);SDK 是 _本地代码层_。SDK 内部通常 wrap API

**wiki**:[[eval/opik]] 3 SDK(Python/TS/Ruby)/ [[harness/langgraph]] sdk-py + sdk-js 独立 package / [[harness/goose]] goose-sdk crate

### provider
**中文**:提供方 / 模型提供方 | **English**:provider | **我固定叫**:provider | **缩写**:无

**定义**:**提供 LLM 服务的外部 backend**(Anthropic / OpenAI / Google / Azure / Ollama / ...)。在 harness 内通过 `Provider` trait / interface 抽象。

**vs 相邻**:
- vs **channel**:provider = _后端模型方_;channel = _用户接入前端方式_(CLI / Web / Slack)
- vs **model**:provider 是公司,model 是具体模型(`anthropic` provider 下有 `claude-opus-4-7` 等多个 model)

**wiki**:每个 [[harness/*]] `provider` 维度 / [[harness/goose]] 40+ provider 业内最广(含 5 ACP + 5 跨家 coding agent provider)/ [[overview]] §3 决策 1(SkillForge SPI 抽象)

### channel
**中文**:通道 / 接入方式 | **English**:channel | **我固定叫**:channel | **缩写**:无

**定义**:**用户跟 agent 交互的前端方式**(CLI / Web / Mobile / Slack / Discord / Email / Voice / IDE / API)。harness 通常支持多 channel。

**vs 相邻**:
- vs **provider**:见上
- vs **interface**:channel 偏 _业务侧前端_;interface 更技术(API interface)

**wiki**:每个 [[harness/*]] `channels` 维度 / [[harness/openclaw]] 24+ channels + 4 native apps / [[harness/hermes-agent]] 18+ channels

### session
**中文**:会话 | **English**:session | **我固定叫**:session | **缩写**:无

**定义**:**一次 agent 跟用户的完整交互单位**。通常对应一个 conversation,有起止时间 / message history / 可恢复(若 framework 支持)。

**vs 相邻**:
- vs **turn**:turn = session 内一轮(user message + agent reply);session 含多 turn
- vs **conversation**:基本同义,session 更偏 framework 术语,conversation 更偏 LLM 协议术语

**wiki**:[[harness/codex]] `SessionSource::SubAgent(ThreadSpawn{depth})` / [[concept/agent-interop]] external-agent-sessions

### turn
**中文**:回合 | **English**:turn | **我固定叫**:turn | **缩写**:无

**定义**:**session 内一轮交互**(user 一条 message + agent 一段 reply,reply 可能含多 step / tool_use)。

**vs 相邻**:
- vs **step**:turn 含多 step(LLM call + tool 调用)
- vs **round**(GroupChat 概念):AG2/CrewAI 内 round = group chat 内一轮选 speaker + 该 speaker 回复

**wiki**:[[eval/tau-bench]] _"multi-turn benchmark"_ / [[concept/agent-eval]]

### step
**中文**:步 | **English**:step | **我固定叫**:step | **缩写**:无

**定义**:**agent loop 内一次推进**(可能是 LLM call / tool 调用 / state mutation)。一个 turn 内可有多 step。

**vs 相邻**:
- vs **turn**:见上
- vs **loop iteration**:step ≈ loop 单次迭代

**wiki**:[[harness/smolagents]] 5 Memory Step Types(SystemPromptStep / TaskStep / **ActionStep** / PlanningStep / FinalAnswerStep)/ [[harness/crewai]] `step_callback` 4 lifecycle 之一

### loop
**中文**:循环 | **English**:loop | **我固定叫**:loop / agent loop | **缩写**:无

**定义**:**agent 反复 _LLM call → 解析 → tool 执行 → 喂回 LLM_ 的循环结构**。直到 stop condition(end_turn / max_steps / error)。是 agent 区别于 1-shot LLM call 的核心。

**vs 相邻**:
- vs **ReAct loop**:ReAct 是 loop 的一种范式(Reason → Act → Observe 循环)
- vs **workflow**:workflow 是 _预定义 DAG_;loop 是 _自适应循环_

**wiki**:每个 [[harness/*]] `agent_loop` 维度 / [[concept/agent-runtime-stacks]] §Agent Execution Paradigm 4 种(ReAct loop / Event-driven / Actor / Pregel-StateGraph)

---

## B. Tool 调用与协议(9)

### tool
**中文**:工具 | **English**:tool | **我固定叫**:tool | **缩写**:无

**定义**:**LLM 通过 tool_use(function calling)协议调用的单次函数**。输入 schema 化(JSON Schema),输出回 tool_result。是 agent 跟外界交互的 _原子动作_。

**vs 相邻**:
- vs **skill**:tool = _单次函数调用_,无状态,LLM 主动 invoke;skill = _多步任务包_ + 文档,framework 加载到上下文
- vs **action**(MetaGPT):MetaGPT `Action` ≠ `Tool` — Action = role 职责级,Tool = 底层执行单元
- vs **MCP tool**:同语义,但 MCP tool 通过 MCP 协议暴露(跨 framework 兼容)

**wiki**:每个 [[harness/*]] `tool_protocol` 维度 / [[concept/code-mode-orchestration]](Code Mode vs JSON tool_use 对照)

### tool_use
**中文**:工具调用(协议词)| **English**:tool_use / function calling | **我固定叫**:tool_use(协议词)/ function calling(俗称)| **缩写**:无

**定义**:**LLM 输出的特殊消息块,表示 _要调用某个 tool_**。Anthropic API 用 `tool_use` block(`{"type":"tool_use","name":"X","input":{...}}`),OpenAI 用 `function_call` 字段(已 deprecated)→ `tool_calls`(新)。

**vs 相邻**:跟 **tool** 是 _协议触发器 vs 工具本身_;跟 **tool_result** 是 _请求 vs 响应_。

**wiki**:[[concept/code-mode-orchestration]] / 每个 [[harness/*]] `tool_protocol` 维度

### tool_result
**中文**:工具结果 | **English**:tool_result | **我固定叫**:tool_result | **缩写**:无

**定义**:**framework 把 tool 执行结果喂回 LLM 的消息块**。含 `tool_use_id`(对应哪次调用)+ `content`(结果文本/JSON/图像)+ 可选 `is_error`。

**vs 相邻**:跟 **tool_use** 是 _响应 vs 请求_;**孤儿 tool_result**(无对应 tool_use)会引发 LLM 报错,compaction 时需特别处理(详见 [[harness/hermes-agent]] `_sanitize_tool_pairs`)

**wiki**:[[deep/claude-code-compact-tool-result]] / [[concept/context-compaction]] _"边界保护"_ 维度

### MCP
**中文**:模型上下文协议 | **English**:Model Context Protocol | **我固定叫**:MCP | **缩写**:**MCP**(固定,不展开)

**定义**:**Anthropic 2024 提出的 tool / resource / prompt 跨 framework 调用标准协议**。已成事实标准(18+ wiki 集成)。6 primitives 对称:Server 3(Resources/Prompts/Tools)+ Client 3(Sampling/Roots/Elicitation)。

**vs 相邻**:
- vs **A2A**:MCP = _tool 层_(LLM 调工具);A2A = _agent 层_(agent 调 agent)
- vs **OpenAPI / gRPC**:MCP 是 _LLM-friendly_ schema(自带 description / 自适配 prompt);OpenAPI 偏 REST

**wiki**:[[concept/agent-protocols]] §2 / [[deep/mcp-spec-deep]] / [[harness/goose]] 5 内置 MCP server / [[harness/ragflow]] / [[harness/dify]]

### A2A
**中文**:Agent2Agent 协议 | **English**:Agent2Agent Protocol | **我固定叫**:A2A | **缩写**:**A2A**

**定义**:**Google 2025 提出的 agent 之间互通协议**。Linux Foundation 托管。4 primitives:Agent Card / Task / Skill / Message。强调 _"不共享 internal memory/tools/logic"_ 的 IP 保护。

**vs 相邻**:
- vs **MCP**:见上(layer 不同)
- vs **ACP**:ACP 已 merge 进 A2A(2026-05-17 a2a-protocol.org 明示)
- vs **AGNTCY**:AGNTCY 是 _A2A/MCP 之上的 mesh middleware_(详见 [[concept/agent-protocols]] §4)

**wiki**:[[concept/agent-protocols]] §3 / [[deep/google-a2a-spec]] / [[harness/agentscope]] `a2a/` 5 文件 / [[harness/crewai]] A2A 集成

### ACP
**中文**:Agent Client 协议 | **English**:Agent Client Protocol | **我固定叫**:ACP | **缩写**:**ACP**(已 merged,但术语保留)

**定义**:**IBM BeeAI 出品的 agent protocol,已 merge 进 A2A under Linux Foundation**(2026-05-17)。原 REST-based(跟 A2A JSON-RPC 不同),MIME type + 同步/异步/streaming + 嵌入式 metadata offline discovery。

**vs 相邻**:跟 **A2A** 是 _前身 vs 现状_。**[[harness/goose]] 是业内 ACP 实现最完整**(`goose-acp-macros` 295 行 proc-macro + 5 ACP provider 实例),即使合并后仍是参考样本。

**wiki**:[[concept/agent-protocols]] §3.4 + §3.5(goose 实现示范)/ [[harness/goose]] `Cargo.toml:27-28`

### AG-UI
**中文**:Agent-User Interaction 协议 | **English**:Agent-User Interaction Protocol | **我固定叫**:AG-UI | **缩写**:**AG-UI**

**定义**:**CopilotKit 等推的 agent ↔ UI 互通协议**(10+ 1st-party 集成)。核心是 _human-in-the-loop interrupts 协议级一等公民_(LangGraph / CrewAI 内置)。

**vs 相邻**:跟 **MCP** / **A2A** 是 _UI 层 vs tool 层 vs agent 层_,5 层栈中 UI 层。

**wiki**:[[concept/agent-protocols]] §5 / [[harness/langgraph]] `prebuilt/interrupt.py` first-class HITL

### AGNTCY
**中文**:Internet of Agents(编排/网格层)| **English**:AGNTCY | **我固定叫**:AGNTCY | **缩写**:**AGNTCY**

**定义**:**Cisco Outshift + Linux Foundation 主导的 _跨协议 mesh middleware_**。**不替代** MCP/A2A,而是 _之上_ 的编排/网格层。5 大组件:OASF(Agent Discovery)+ Agent Identity 加密 + SLIM(messaging)+ Observability + Lifecycle。

**vs 相邻**:跟 **MCP / A2A** 是 _网格层 vs 协议层_ — _"middleware above"_ 是 AGNTCY doc 明示。

**wiki**:[[concept/agent-protocols]] §4 / [[overview]] §2 判断 10(SkillForge 协议优先级 P3 不必跟)

### OTel
**中文**:OpenTelemetry | **English**:OpenTelemetry | **我固定叫**:OTel | **缩写**:**OTel**

**定义**:**CNCF 出品的可观测协议标准**(trace / metric / log 三件套)。LLM 场景有 GenAI semconv 子规范(含 MCP semantics)。**21+ wiki 集成,最广**。

**vs 相邻**:
- vs **Wide Events 2.0**:OTel 1.0 是 _三分法_;Wide Events 2.0 是 _挑战者_(详见 [[concept/observability]])
- vs **OpenInference**:Phoenix 主推的 LLM trace semconv,跟 OTel GenAI semconv 并行

**wiki**:[[concept/agent-protocols]] §6 / [[concept/observability]] / [[eval/agentops]] 12 SpanKind + 3 Legacy sub-kind

---

## C. Skill 加载体系(6)

### skill
**中文**:技能包 | **English**:skill / Skill | **我固定叫**:skill(Anthropic Skill 标准)| **缩写**:无

**定义**:**LLM 可加载的可复用任务包**(prompt + 关联文件 + metadata),framework 根据 trigger 自动加载到上下文。Anthropic Skills SDK 标准的产物(`.claude/skills/<name>/SKILL.md`)。

**vs 相邻**(高频混淆):
- vs **tool**:tool = 单次函数调用,LLM 主动 invoke;skill = 多步任务包,framework 把 SKILL.md 加载进 prompt
- vs **command**:command = slash(`/xxx`)用户主动触发;skill = LLM 根据描述自动加载
- vs **agent**:agent = 主动 loop 执行体;skill = 被动加载的能力包(无 loop)
- vs **plugin / extension**:plugin 是上一代术语,Skills 是 Anthropic 2025 重新设计的 _首选加载机制_

**wiki**:[[harness/claude-code]] / [[harness/codex]] `skills` crate / [[harness/opencode]] `tool/skill.ts` / wiki-* 4 个 skill 在 `.claude/skills/`(commit `7ca9a65`)

### extension (plugin)
**中文**:扩展 / 插件 | **English**:extension / plugin | **我固定叫**:extension(新词)/ plugin(老词)| **缩写**:无

**定义**:**外挂到 framework 的功能模块**,通常含一组 hook + 一组 tool + manifest。比 skill 更"重",通常需要安装/启用。

**vs 相邻**:
- vs **skill**:skill 是 Anthropic 主推的 _轻量_ 加载机制(2025+);extension/plugin 是 _重量_ 模块(包含 hook / tool / 资源)
- 命名差异:VS Code _"extension"_、Hermes _"plugin"_、Codex _"plugin"_、OpenClaw _"extension"_

**wiki**:[[harness/codex]] 27+ extensions / [[harness/openclaw]] 30+ extensions / [[harness/hermes-agent]] plugin 装饰器

### command
**中文**:命令 / 斜杠命令 | **English**:command / slash command | **我固定叫**:command / slash command | **缩写**:无

**定义**:**用户在 chat 内输入 `/xxx` 触发的入口**。框架解析为 _action / skill 调用 / 系统操作_。是用户跟 framework 主动交互的快捷方式。

**vs 相邻**:
- vs **skill**:command 是 _用户主动触发的入口_;skill 是 _LLM 自动加载的能力包_。command 内部可以 invoke skill
- vs **tool**:command 是 _用户层交互_(UI 侧);tool 是 _LLM 层调用_(LLM 侧)

**wiki**:[[harness/claude-code]] `.claude/commands/` / [[harness/codex]] `slash_commands.rs` / [[harness/goose]] `slash_commands.rs`

### hook
**中文**:钩子 | **English**:hook / lifecycle event | **我固定叫**:hook | **缩写**:无

**定义**:**framework 在生命周期关键点暴露给用户/plugin 的回调插槽**(PreToolUse / PostToolUse / SessionStart / ...)。让用户在不改 framework 代码的前提下 _介入_ 流程。

**vs 相邻**:
- vs **callback**:callback 是程序术语(任意位置回调);hook 是 framework 设计模式(预定义点位)
- vs **plugin / extension**:plugin = 一组 hook + 代码;hook = plugin 的接入点
- vs **lifecycle event**:基本同义,lifecycle event 更偏 _事件本身_,hook 更偏 _事件的回调机制_

**wiki**:[[concept/hooks-design]] 阶梯(Codex 8 < goose 11 < Hermes 16 < Claude Code 27)

### lifecycle event
**中文**:生命周期事件 | **English**:lifecycle event | **我固定叫**:lifecycle event(完整)/ event(简称)| **缩写**:无

**定义**:**framework 运行过程中预定义的关键节点**(start / pre-tool / post-tool / pre-compact / post-compact / stop ...)。hook 订阅 lifecycle event。

**vs 相邻**:跟 **hook** 是 _事件本身 vs 订阅机制_。在 Open Plugins 标准(open-plugins.com)内,event 是 _协议命名_,hook 是 _实现方式_。

**wiki**:[[concept/hooks-design]] / [[harness/goose]] 11 HookEvent enum

### recipe
**中文**:配方 / 工作流谱 | **English**:recipe | **我固定叫**:recipe | **缩写**:无

**定义**:**Codex / goose 引入的 _声明式 workflow spec_**(YAML / JSON),framework 按 recipe 跑一组 agent / tool / 步骤。比 imperative 代码更易复用 / 测试 / 跨 channel 触发。

**vs 相邻**:
- vs **workflow**:workflow 是更通用术语(LangGraph / dify 等都有);recipe 强调 _声明式 + 易分享_
- vs **skill**:skill 是 _能力包_(LLM 加载);recipe 是 _执行流程_(framework 跑)

**wiki**:[[harness/codex]] `recipe` 模块 / [[harness/goose]] `recipe/` + `recipe_deeplink.rs` + `goose-self-test.yaml` recipe-as-test

---

## D. Context & 提示(11)

### context window
**中文**:上下文窗口 | **English**:context window | **我固定叫**:context window | **缩写**:**CW**(偶尔)

**定义**:**LLM 单次推理可接受的最大 token 数**(Claude Opus 4.7 = 1M / GPT-5 = ? / Gemini 2.5 = 2M)。决定 agent _一次能塞多少历史 + 工具 + 指令_。

**vs 相邻**:
- vs **compaction threshold**:阈值百分比(如 80%)触发压缩,详见 **compaction**
- vs **prompt cache**:cache 是 _跨调用复用 context 内部前缀_,context window 是 _单次输入上限_

**wiki**:[[harness/goose]] `DEFAULT_COMPACTION_THRESHOLD=0.8` / [[harness/claude-code]] `AUTOCOMPACT_BUFFER_TOKENS=13_000`

### system prompt
**中文**:系统提示词 | **English**:system prompt / system message | **我固定叫**:system prompt | **缩写**:**SP**(偶尔)

**定义**:**LLM API 调用时传的 `system` role 消息**,设定 agent 的 _身份、能力、风格、约束_。是 LLM 协议层的固定字段(Anthropic API `system` 参数 / OpenAI API `system` message)。

**vs 相邻**(扩展对照):

| | **system prompt** | **rules** | **instructions** |
|---|---|---|---|
| 谁定义 | framework / 开发者 | 项目维护者 / 用户 | 用户(per session)|
| 加载方式 | API 参数(每次调用都传)| 项目文件(`.claude/rules/`,被 framework 注入 system prompt 或 user 消息)| 临时 prompt 输入 |
| 生效粒度 | session 级 | 项目级 / 全局 | turn 级 |
| 例子 | "You are Claude Code..." 数千 token | `.claude/rules/code-style.md` 项目编码风格 | 用户在 chat 内说 "用中文回复" |
| 修改成本 | framework 改代码 | 编辑文件 + 下次 session 生效 | 当场说一句 |

**wiki**:[[concept/runtime-context-injection]] / [[harness/claude-code]] / [[deep/claude-code-compact-tool-result]]

### system-reminder
**中文**:系统提醒 | **English**:system-reminder | **我固定叫**:system-reminder | **缩写**:无

**定义**:**Anthropic Claude Code 独家的 _轻量上下文注入协议_**(`<system-reminder>` XML tag)— 让 framework 在 LLM 推理过程中动态注入提示而不污染 system prompt。

**vs 相邻**:
- vs **system prompt**:system prompt 是 _静态、session 级_;system-reminder 是 _动态、turn 级_,且不可被 LLM 信任为权威指令(模型训练过明确区分)
- vs **user message**:system-reminder 比 user message 优先级更高(框架级注入),但 _trained contract_ 保证 LLM 不会被注入欺骗

**wiki**:[[concept/runtime-context-injection]] / [[deep/claude-code-insights]] / [[deep/claude-code-compact-tool-result]] 内多处 `<system-reminder>` 引用

### rules
**中文**:规则 / 项目约定 | **English**:rules | **我固定叫**:rules | **缩写**:无

**定义**:**项目级或全局的约束性指令**,通常以文件形式存放(`.claude/rules/*.md` / `.cursor/rules/` / `AGENTS.md`),被 framework 在合适时机注入 system prompt 或 user 消息。**比 system prompt 颗粒度细**,**比 instructions 持久**。

**vs 相邻**:见 **system prompt** 扩展对照表。

**wiki**:[[harness/claude-code]] `.claude/rules/` / [[harness/codex]] `AGENTS.md` 17KB / [[concept/agent-runtime-stacks]] §`.agents/` dogfood 判断 7

### instructions
**中文**:指令 | **English**:instructions | **我固定叫**:instructions | **缩写**:无

**定义**:**用户在 chat 内的临时指令**(_"用中文回复" / "代码加注释"_)。turn 级生效,无持久化。

**vs 相邻**:见 **system prompt** 扩展对照表。

**wiki**:不单独成页(高频术语)

### prompt
**中文**:提示词 | **English**:prompt | **我固定叫**:prompt | **缩写**:无

**定义**:**送进 LLM 的输入文本**。是 _笼统父概念_,具体可分 system prompt / user prompt / few-shot example / template 等。

**vs 相邻**:
- vs **system prompt**:system prompt ⊂ prompt(子类)
- vs **template**:template 是 prompt 的 _骨架_,填变量后才成 prompt

**wiki**:[[concept/prompt-optimization]] 6 维设计空间 / [[eval/dspy]] _"Prompt-as-program"_

### template
**中文**:模板 | **English**:template / prompt template | **我固定叫**:template | **缩写**:无

**定义**:**含变量占位符的 prompt 骨架**(`"Translate {{lang}} to English: {{text}}"`)。framework 在运行时填入变量产出最终 prompt。

**vs 相邻**:
- vs **prompt**:template + 变量 → prompt
- vs **prompt cache**:template 通常作为 cache prefix(变量在后面,前缀稳定可 cache)

**wiki**:[[harness/codex]] InitialContextInjection / [[harness/opencode]] `SUMMARY_TEMPLATE` 7 段强制结构

### few-shot
**中文**:少样本(示例)| **English**:few-shot (examples) | **我固定叫**:few-shot | **缩写**:无

**定义**:**在 prompt 内嵌 N 个示例(input → expected output 对)**,让 LLM 通过 in-context learning 学会任务,无需训练。N 通常 1-10,大于 10 叫 many-shot。

**vs 相邻**:
- vs **zero-shot**:0 示例(直接指令 + 任务)
- vs **fine-tune**:few-shot 是 prompt 内嵌(轻量),fine-tune 是模型参数更新(重)

**wiki**:[[eval/dspy]] FewShotBayesianOptimizer / [[eval/opik]] `FewShotBayesianOptimizer` algorithm

### prompt cache
**中文**:提示缓存 | **English**:prompt cache | **我固定叫**:prompt cache | **缩写**:无

**定义**:**LLM provider 把 prompt 前缀缓存,后续相同前缀的调用复用,降本提速**。Anthropic 5-min TTL 默认,跨调用 5-10× 便宜。

**vs 相邻**:
- vs **memory**:cache 是 _传输层优化_(provider 侧);memory 是 _应用层状态_(agent 侧)
- vs **context window**:cache 在 context window 内 _子段优化_,不扩窗

**wiki**:[[deep/claude-code-compact-tool-result]] _"cache-editing 不删本地消息只发 cache_edits 给 server"_ 98% cache miss 实测

### compaction
**中文**:压缩 | **English**:compaction / context compaction | **我固定叫**:compaction | **缩写**:无

**定义**:**当 context 接近 window 上限时,framework 把旧消息压缩成摘要塞回**。是 long-running agent 的必需机制。

**vs 相邻**:
- vs **summary**:summary 是 compaction 的 _产物_;compaction 是 _动作_
- vs **memory**:compaction 是 _session 内压缩_;memory 是 _跨 session 持久化_

**wiki**:[[concept/context-compaction]] 7 维设计空间 / [[deep/claude-code-compact-tool-result]] / [[harness/goose]] 80% threshold

### summary
**中文**:摘要 | **English**:summary | **我固定叫**:summary | **缩写**:无

**定义**:**compaction 产出的浓缩 context 段落**。通常有强制结构(opencode `SUMMARY_TEMPLATE` 7 段:Goal / Constraints / Progress(Done/InProgress/Blocked)/ Key Decisions / Next Steps / Critical Context / Relevant Files)。

**vs 相邻**:跟 **compaction** 是 _产物 vs 动作_。跟 **memory** 不同,summary 是 session 内 inline 注入,不跨 session。

**wiki**:[[harness/opencode]] `SUMMARY_TEMPLATE` / [[harness/hermes-agent]] Phase B 5 段结构 / [[concept/context-compaction]]

---

## E. Memory(6)

### memory
**中文**:记忆 | **English**:memory | **我固定叫**:memory | **缩写**:无

**定义**:**agent 跨 session 保存的状态/事实/经验**。区别于 _session 内 context_。形式多样:文件(MEMORY.md)/ KG / vector store / SQLite。

**vs 相邻**:
- vs **context**:context 是 _session 内_(turn 历史);memory 是 _跨 session_
- vs **knowledge base**:KB 通常 _静态 + 全局_(文档库);memory _动态 + agent 特定_

**wiki**:[[concept/agent-memory]] 7 路线 / 每个 [[harness/*]] `memory` 维度

### short-term memory
**中文**:短期记忆 | **English**:short-term memory | **我固定叫**:short-term memory | **缩写**:**STM**(偶尔)

**定义**:**session 内 active 的快速 memory**,通常是 _压缩后塞回 context_ 的近期事实。容量小,易遗忘。

**vs 相邻**:
- vs **long-term**:阈值后晋升 long-term(详见 [[deep/openclaw-memory-dreaming-6-dim]] 6 维评分晋升)
- vs **context**:short-term memory 通常驻留 context,但有自己的容量边界

**wiki**:[[harness/crewai]] 3 维 memory scope(short_term / long_term / entity)/ [[harness/hermes-agent]] 3 层 memory

### long-term memory
**中文**:长期记忆 | **English**:long-term memory | **我固定叫**:long-term memory | **缩写**:**LTM**(偶尔)

**定义**:**跨 session 持久化的稳固 memory**。通常存 KG / vector store / SQLite,通过 retrieval 拉回。

**vs 相邻**:见 **short-term**。

**wiki**:[[harness/agentscope]] long_term + working 双层 / [[harness/crewai]] long_term scope

### KG-memory (Knowledge Graph memory)
**中文**:知识图谱记忆 | **English**:KG memory | **我固定叫**:KG-memory | **缩写**:**KG-mem**

**定义**:**把 memory 存成 entity + relationship 图**,支持图谱遍历检索。Graphiti / Zep / Cognee / ms-graphrag 都走这路线。

**vs 相邻**:
- vs **vector memory**:vector 适合语义相似;KG 适合多跳推理 + 时序事实
- vs **episode**:episode 是 KG 的 _时序原子单位_(Graphiti 概念)

**wiki**:[[eval/zep-graphiti]] / [[eval/cognee]] / [[concept/agent-memory]] 第 5 路线(Temporal KG)

### memory block
**中文**:记忆块 | **English**:memory block | **我固定叫**:memory block | **缩写**:无

**定义**:**Letta(MemGPT)引入的 memory 单位**,可读写、可 git 版本控制(`block_manager_git.py`)、可 read-only。是 _agent 主动管理_ 的 memory 抽象。

**vs 相邻**:
- vs **chunk**:chunk 是 _RAG 文档切片_(被动);block 是 _agent 主动写_(主动)

**wiki**:[[eval/letta]] `block_manager_git.py` 业内独家

### episode
**中文**:情节 / 事件单元 | **English**:episode | **我固定叫**:episode | **缩写**:无

**定义**:**Graphiti(Zep)引入的 KG 时序原子单位**。每个 fact 都追溯到 source episode(provenance),支持 _事实何时为真 / 何时被推翻_(validity windows)。

**vs 相邻**:
- vs **fact**:fact 是 _三元组_(主-谓-宾);episode 是 fact 的 _来源容器_
- vs **provenance**:episode 是 _provenance 实例_(在 KG 上下文)

**wiki**:[[eval/zep-graphiti]] Episode provenance / [[concept/agent-memory]] 第 5 路线

---

## F. Eval / Benchmark(21)

### eval / evaluation
**中文**:评测 | **English**:eval / evaluation | **我固定叫**:eval | **缩写**:无

**定义**:**给 agent / prompt / model 打分的过程**。可以是离线 benchmark,也可以是 production trace 上的持续评估。是 _动词_(跑 eval)和 _名词_(eval framework)双用。

**vs 相邻**:
- vs **benchmark**:benchmark = _固定 test set + 标准跑法_;eval = _更笼统_,含 benchmark + production eval + LLM-as-judge

**wiki**:[[concept/agent-eval]] 8 维 / `eval/` 16 + `framework/` 5 子目录

### evaluator
**中文**:评估器 | **English**:evaluator | **我固定叫**:evaluator | **缩写**:无

**定义**:**实际执行 eval 的组件 / 类**(可以是 LLM judge / rule-based / human)。在框架内通常是抽象接口(Phoenix `Evaluator` / Ragas `Metric` / Promptfoo `Assertion`)。

**vs 相邻**:
- vs **judge**:judge 是 evaluator 的 _LLM 实现子类_
- vs **metric**:evaluator 是 _执行单元_;metric 是 _量化函数_。evaluator 跑出 score,score 算 metric

**wiki**:[[eval/phoenix]] `ClassificationEvaluator` / [[eval/ragas]] `MetricWithLLM` 6 抽象基类

### benchmark
**中文**:基准测试 | **English**:benchmark | **我固定叫**:benchmark | **缩写**:无

**定义**:**业内公认的固定 test set + 标准跑法 + 公开 leaderboard**(SWE-bench / tau-bench / LongMemEval / MMLU)。是 eval 的子集,但有 _跨家可比_ 的特殊地位。

**vs 相邻**:
- vs **eval**:benchmark ⊂ eval(benchmark 是 _最严的 eval 形式_)
- vs **eval set**:benchmark 是 _一次性 test set_,eval set 是 _持续演化_

**wiki**:[[eval/swe-bench]] / [[eval/tau-bench]] / [[harness/metagpt]] AFlow ICLR 2025 / [[eval/opik]] 8 内置 benchmark dataset

### dataset
**中文**:数据集 | **English**:dataset | **我固定叫**:dataset | **缩写**:无

**定义**:**一组样本的集合**,每个样本含 input + 期望输出(ground truth)。是 _笼统父概念_,具体分 training / dev / test / eval 4 个子集。

**vs 相邻**(扩展对照,最高频混淆):

| | **training set** | **dev set / validation set** | **test set** | **eval set** |
|---|---|---|---|---|
| 用途 | 训模型用(梯度更新)| 调超参 / 选模型 | 最后报告性能 | **生产中持续评测**(deployment monitoring)|
| 数据可见性 | 模型训练时看过 | 调参时反复看 | **训练时严禁看**(否则数据泄露)| production trace 子集 |
| 何时用 | 训练阶段 | 训练后 / 部署前 | 部署前 release gate | **持续(production loop)** |
| 在 LLM agent 语境 | 基模型训练用,跟我们无关 | SkillForge prompt A/B 选 prompt | benchmark 跑 SWE-bench 用 | Langfuse `addDatasetRunItemsToEvalQueue` |
| 我的用法 | 几乎不用 | 偶尔(prompt 选优时)| 偶尔(跑 benchmark 时)| **最常用**(SkillForge 默认指这个)|

**wiki**:[[eval/swe-bench]](test set 典型)/ [[eval/langfuse]] `addDatasetRunItemsToEvalQueue` production loop(eval set 典型)

### training set
**中文**:训练集 | **English**:training set | **我固定叫**:training set | **缩写**:**train**(偶尔)

**定义**:见 **dataset** 表。在 LLM agent 语境通常不用(基模型由 Anthropic / OpenAI 训,我们不动)。

**vs 相邻**:见 **dataset** 扩展对照表。

**wiki**:几乎不用,偶尔在 [[harness/agentscope]] Trinity-RFT(agentic RL training)

### dev set / validation set
**中文**:验证集 / 开发集 | **English**:dev set / validation set | **我固定叫**:dev set | **缩写**:无

**定义**:**训练时反复看的小集合,用于调超参 / 选模型(model selection)**。LLM agent 场景常对应 _prompt A/B 时选优用的小集_。

**vs 相邻**:见 **dataset** 扩展对照表。

**wiki**:[[eval/dspy]] BootstrapFewShot 内部 dev set 用法 / [[overview]] §3 决策 6 Prompt A/B 自动晋升

### test set
**中文**:测试集 | **English**:test set | **我固定叫**:test set | **缩写**:无

**定义**:见 **dataset** 表 — _训练前留出 / 训练时不看 / 最后报告性能_ 的子集。在我们语境 ≈ benchmark suite。

**vs 相邻**:见 **dataset** 扩展对照表。**关键纪律**:test set 不应被反复刷分调整(否则等于变 dev set)。

**wiki**:[[eval/swe-bench]] 4 变体 / [[eval/tau-bench]]

### eval set
**中文**:评测集 | **English**:eval set / evaluation set | **我固定叫**:eval set | **缩写**:无

**定义**:**生产中持续评测 agent 行为用的样本集**。通常是 production trace 的人工或自动 curation 子集,**持续追加** + **定期回归测试**。**是 LLM agent 语境最常说的"评测数据"**。

**vs 相邻**:见 **dataset** 扩展对照表。**关键区别**:test set 一次性,eval set 持续演化。

**wiki**:[[eval/langfuse]] `addDatasetRunItemsToEvalQueue` production loop / [[concept/agent-eval]] / [[concept/trace-based-eval]]

### hold-out
**中文**:留出集 | **English**:hold-out (set) | **我固定叫**:hold-out | **缩写**:无

**定义**:**从总数据集切出来 _不参与训练_ 的子集**(test set 是最典型 hold-out)。强调 _切出动作_ + _隔离纪律_。

**vs 相邻**:
- vs **test set**:hold-out ⊆ test set(test set 是 hold-out 的一种)
- vs **k-fold cross-validation**:hold-out 是 _一次切_;k-fold 是 _N 次切轮换_

**wiki**:偶尔出现,不单独成章节

### ground truth
**中文**:真实标签 / 真值 | **English**:ground truth | **我固定叫**:ground truth / GT | **缩写**:**GT**

**定义**:**样本对应的 _正确答案_**,是 _算 metric 的基准_。在 LLM agent 场景常以多种形式存在:执行结果(test 通过/失败)/ 人工标注 / 强模型(GPT-5)输出参考。

**vs 相邻**:
- vs **label**:label 是 _标的过程产物_(可能是 GT,也可能是 annotation 的中间产物);GT 强调 _"被视为正确"_
- vs **prediction**:prediction = LLM 输出,跟 GT 算 metric

**wiki**:[[concept/agent-eval]] / [[eval/swe-bench]](测试用例执行结果作 GT)

### label / annotation
**中文**:标注 / 打标 | **English**:label / annotation | **我固定叫**:label(单数动词)/ annotation(过程或产物)| **缩写**:无

**定义**:**给样本 _人工或 LLM 加标签_ 的过程或产物**。typically discrete categories(spam/not-spam / hallucinated/factual / level-1/2/3)。

**vs 相邻**:
- vs **score**:见 **score** 对照(label 离散,score 数值)
- vs **ground truth**:label 通常 _被采用为 GT_,但 GT 强调 _"正确性被认可"_,label 强调 _"标注动作"_
- vs **annotation**(细分):多人标 + 一致性检验(Cohen's κ)后才升 GT;单人标的叫 annotation 更准

**wiki**:[[concept/agent-eval]] / [[eval/langfuse]] Score `ANNOTATION` source 三态之一

### score
**中文**:分数 / 打分 | **English**:score / scoring | **我固定叫**:score(名词)/ scoring(动词)| **缩写**:无

**定义**:**metric 在某个样本上的输出值**(数值或分类)。score 通常带 source(API / EVAL / ANNOTATION)+ dataType(NUMERIC / CATEGORICAL / BOOLEAN / TEXT)。

**vs 相邻**(高频混淆 score vs label):
- vs **label**:**score 强调 _数值化_**(0.87 / pass);**label 强调 _离散分类_**(positive / hallucinated)。但 Langfuse Score 把 5 种 dataType 都叫 score(NUMERIC / CATEGORICAL / BOOLEAN / **CORRECTION** / **TEXT**)
- vs **metric**:score = metric 函数的 _输出_;metric = score 的 _算法_

**wiki**:[[eval/langfuse]] Score 3×5 矩阵 / [[eval/phoenix]] `Score.explanation` 业内最规范

### metric
**中文**:指标 | **English**:metric | **我固定叫**:metric | **缩写**:无

**定义**:**衡量 agent / model 性能的 _量化函数_**(accuracy / faithfulness / latency / cost / token)。每个 metric 输出一个数值。

**vs 相邻**(扩展对照,metric vs dimension 高频混淆):

| | **metric** | **dimension** |
|---|---|---|
| 是什么 | 一个 _量化函数_(输出数值) | 一个 _评估角度_(下面挂多个 metric)|
| 例子 | `accuracy = 0.87` / `latency = 250ms` | "performance" 维度 / "cost" 维度 / "safety" 维度 |
| 关系 | metric ⊂ dimension(_一个 dimension 下有多个 metric_)| dimension 是 metric 的 _组织框架_ |
| wiki 例 | Ragas 27 metric | [[concept/agent-eval]] 8 维 / [[harness/*]] 13 维 frontmatter |
| 我的用法 | 算数值时 | 设计评估框架时 |

**wiki**:[[eval/ragas]] 27 metric / [[eval/llama_index]] 13 metric / [[concept/agent-eval]]

### dimension
**中文**:维度 | **English**:dimension | **我固定叫**:dimension / 维 | **缩写**:无

**定义**:**评估 / 设计 / 分析的角度**,是 metric / 子项的 _组织框架_。注意 wiki 还有第二种用法:_13 维 harness frontmatter_(设计维度,非评估维度,但同概念)。

**vs 相邻**:见 **metric** 扩展对照表。**警告**:wiki 内 "13 维度" 既指 _harness 设计角度_(agent_loop / tool_protocol / ... 13 个 frontmatter 字段)又借用了 dimension 一词,容易混。

**wiki**:[[concept/agent-eval]] 8 维 / [[schema]] §3.1 13 维 frontmatter

### rubric
**中文**:评分细则 | **English**:rubric | **我固定叫**:rubric | **缩写**:无

**定义**:**告诉 evaluator(尤其 LLM-as-judge)_怎么打分_ 的细则文本**。通常含 _好的特征 / 坏的特征 / 边界 case / 打分 scale_。

**vs 相邻**:
- vs **metric**:metric 是 _算法_;rubric 是 _算法的输入说明_(给 LLM judge 看)
- vs **prompt**:rubric 是 _特殊用途 prompt_(专为 evaluator)

**wiki**:[[eval/promptfoo]] `llmRubric` assertion / [[eval/ragas]] AspectCritic 内部 rubric

### assertion
**中文**:断言 | **English**:assertion | **我固定叫**:assertion | **缩写**:无

**定义**:**对 agent 输出的 _断言式 eval_**(`response_contains: "thanks"` / `latency < 1s` / `cost < $0.01`)。布尔 pass/fail,适合 CI 集成。

**vs 相邻**:
- vs **metric**:assertion 是 _布尔型 metric_(pass/fail);metric 通常 _数值_
- vs **scorer**:scorer 偏 _算 score_;assertion 偏 _算 pass/fail_

**wiki**:[[eval/promptfoo]] 50+ 内置 assertion / [[eval/opik]] Assertion + FeedbackDefinitionModel

### judge / LLM-as-judge
**中文**:评判器 / LLM 风评判 | **English**:judge / LLM-as-judge | **我固定叫**:LLM-as-judge(完整)/ judge(简称)| **缩写**:无

**定义**:**用 LLM 给 agent 输出打分 / 标注的方法**。比人工标快/便宜,比 rule-based 灵活,但有 bias 和 hallucination 风险。配 rubric 使用效果好。

**vs 相邻**:
- vs **human annotation**:LLM-as-judge = 自动;human = 慢贵但权威。production 通常混合
- vs **rule-based scoring**:rule-based(regex / exact match)适合 deterministic;LLM-as-judge 适合开放性输出

**wiki**:[[eval/ragas]] AspectCritic / [[eval/phoenix]] `ClassificationEvaluator include_explanation=True` / [[eval/promptfoo]] `llmRubric` / [[concept/attribution]] Layer B

### pass@k
**中文**:通过率(尝试 k 次)| **English**:pass@k | **我固定叫**:pass@k | **缩写**:**pass@1 / pass@k**

**定义**:**给 agent / model k 次尝试,有 1 次通过就算成功的指标**。k=1 严格;k=5 / k=10 衡量 _有多大概率潜在能解_。Codex / SWE-bench 标杆指标之一。

**vs 相邻**:
- vs **accuracy**:accuracy = 1 次必对的占比;pass@k = k 次内对的占比(k>1 总是 ≥ accuracy)
- vs **Pass^k**(tau-bench):**Pass^k 是 _consistency_** — k 次都对才算,跟 pass@k(1 次对即可)相反。**两者完全相反方向**

**wiki**:[[eval/swe-bench]] / [[eval/tau-bench]] _"Pass^k consistency"_(Claude 3.5 Sonnet retail Pass^1=0.692 → Pass^4=0.462)

### golden
**中文**:黄金样本 | **English**:golden / golden set | **我固定叫**:golden(集)/ golden sample(单)| **缩写**:无

**定义**:**精挑细选的 _权威评测样本_**,通常人工标 + 多人审 + 长期稳定。是 _最高质量的 eval 数据_。

**vs 相邻**:
- vs **ground truth**:golden 是 _高质量 GT 集_;GT 强调 _正确性_,golden 强调 _权威 + 稳定_
- vs **eval set**:golden ⊂ eval set,但要求更严

**wiki**:不单独成页(高频术语)

### A/B test
**中文**:A/B 测试 | **English**:A/B test / A/B testing | **我固定叫**:A/B test | **缩写**:**A/B**

**定义**:**同时跑 _对照组(A,baseline)_ 跟 _实验组(B,候选)_,根据 metric 决定胜出**。是 _部署决策方法_,跟 eval 配合使用。

**vs 相邻**:
- vs **eval**:eval 算 score;A/B test 用 eval 结果 _做决策_
- vs **champion-challenger**:见下,A/B 是同义但更普遍

**wiki**:[[overview]] §3 SkillForge 已领先的(_Eval Pipeline + Prompt A/B + 自动晋升_)/ [[eval/langfuse]] EvalTemplate 版本控制

### champion-challenger
**中文**:在位 vs 挑战者 | **English**:champion-challenger | **我固定叫**:champion-challenger | **缩写**:**C-C**(罕用)

**定义**:**部署模式:champion = 当前胜出方,challenger = 候选;持续 A/B,赢的就晋升 champion**。强调 _持续迭代_。

**vs 相邻**(部署对照模式三件套):
- vs **A/B test**:champion-challenger 是 _持续 A/B_;一次性叫 A/B
- vs **shadow**:shadow = challenger _不影响用户_(只录响应),A 全流量,B 仅录 metric
- vs **canary**:canary = challenger _放小流量_(1% / 5%)逐步放量

**wiki**:[[overview]] §3 决策 6 Production loop(对照 Langfuse `addDatasetRunItemsToEvalQueue`)

---

## G. Multi-agent(8)

### multi-agent
**中文**:多智能体(系统)| **English**:multi-agent / multi-agent system | **我固定叫**:multi-agent | **缩写**:**MA**(罕用)

**定义**:**多个 agent 协作完成任务的系统**。可分:完全隔离 fire-and-forget / Fork 继承 / Actor mailbox / Tab 切换 / Role-play 哲学(5 路线,详见 [[concept/multi-agent-isolation]])。

**vs 相邻**:
- vs **agent**:单 agent vs 多 agent 系统
- vs **swarm**:swarm 是 _扁平多 agent_(无 hierarchy);multi-agent 笼统

**wiki**:[[concept/multi-agent-isolation]] 5 路线 / [[concept/agent-team-patterns]] 3 哲学 × 4 组织模式

### SubAgent
**中文**:子智能体 | **English**:SubAgent | **我固定叫**:SubAgent(Anthropic 命名)| **缩写**:无

**定义**:**主 agent 派生的子 agent,跑独立 task 后返结果**。Claude Code / Codex 经典模式。通常隔离 context(防污染主 agent)。

**vs 相邻**:
- vs **fork**:SubAgent 是 _隔离派生_(新 context);fork 是 _继承派生_(共享 parent prompt + cache)
- vs **agent**(顶层):SubAgent 是 _主从关系下的从_

**wiki**:[[harness/claude-code]] SubAgent / [[harness/codex]] `agent/subagent_*` / [[deep/codex-actor-resume-agent]]

### fork
**中文**:派生 / 分叉 | **English**:fork / fork SubAgent | **我固定叫**:fork | **缩写**:无

**定义**:**主 agent _共享 context + prompt cache_ 派生子 agent**。比 SubAgent 隔离更轻量(cache 共享 5-10× 便宜),但 context 互相影响。

**vs 相邻**:
- vs **SubAgent**:见上
- vs **branch**:fork 强调 _派生子任务_;branch 强调 _状态分叉_

**wiki**:[[harness/claude-code]] forkSubagent / [[harness/openhuman]] `harness/fork_context.rs` Fork-style 第 2 家 verified

### handoff
**中文**:交接 | **English**:handoff | **我固定叫**:handoff | **缩写**:无

**定义**:**OpenAI Swarm / Agents SDK 引入的概念:agent A 把 context 完整交给 agent B 继续执行**。强调 _任务移交_,不再回头。

**vs 相邻**:
- vs **delegation**:delegation 是 _派出去等结果_;handoff 是 _交出去不回头_
- vs **SubAgent**:SubAgent 返结果给主 agent;handoff 不返(执行权完全转移)

**wiki**:[[harness/crewai]] `delegations: int` 跟踪 / [[harness/ag2]] handoffs.py / [[backlog]] §1 P0 `openai-agents-sdk` 候选

### role
**中文**:角色 | **English**:role | **我固定叫**:role | **缩写**:无

**定义**:**multi-agent 系统内 _预定义的 agent 类型_**(Architect / Engineer / QAEngineer / ProductManager)。MetaGPT 的核心抽象。

**vs 相邻**:
- vs **agent**:role 是 _agent 的类型_;agent 是 _role 的实例_
- vs **persona**:persona 偏 _风格_;role 偏 _职责_

**wiki**:[[harness/metagpt]] 16+ 预定义 role / [[concept/agent-team-patterns]] Role-play-driven 哲学

### team
**中文**:团队 | **English**:team / crew | **我固定叫**:team | **缩写**:无

**定义**:**一组 agent 协作单元**。MetaGPT `Team` / CrewAI `Crew` 是核心抽象。通常含 _agents + process + memory_。

**vs 相邻**:
- vs **multi-agent**:team 是 _multi-agent 的具体抽象_
- vs **crew**(CrewAI 词):同义

**wiki**:[[harness/metagpt]] `team.py` Team class / [[harness/crewai]] Crew class / [[concept/agent-team-patterns]]

### orchestrator (coordinator)
**中文**:编排器 / 协调器 | **English**:orchestrator / coordinator | **我固定叫**:orchestrator | **缩写**:无

**定义**:**multi-agent 系统的 _中心调度 agent_**,决定调用哪个 agent / 顺序 / 终止条件。CrewAI hierarchical 模式的 `manager_agent` / Claude Code Coordinator 同义。

**vs 相邻**:
- vs **role**(MetaGPT TeamLeader):TeamLeader 是 _role 角色_,orchestrator 是 _调度模式_。可重合
- vs **manager**(CrewAI):同义,CrewAI 用 `manager_agent` / `manager_llm`

**wiki**:[[harness/claude-code]] Coordinator mode / [[harness/crewai]] manager_agent hierarchical / [[harness/openhuman]] orchestrator agent archetype

### swarm
**中文**:群 | **English**:swarm | **我固定叫**:swarm | **缩写**:无

**定义**:**扁平 multi-agent 系统**(无 hierarchy,agent 互相 handoff)。OpenAI Swarm(已演化成 Agents SDK)+ AG2 contrib SwarmAgent 都走这路线。

**vs 相邻**:
- vs **multi-agent**:swarm 是 _扁平多 agent_;multi-agent 笼统(可含 hierarchy)
- vs **team**:team 通常有 _structured roles_;swarm 强调 _emergent collaboration_

**wiki**:[[harness/ag2]] `contrib/swarm_agent` / [[backlog]] §1 P0 `openai-agents-sdk` _"原 Swarm"_

---

## H. RAG & 检索(7)

### RAG
**中文**:检索增强生成 | **English**:Retrieval-Augmented Generation | **我固定叫**:RAG | **缩写**:**RAG**

**定义**:**LLM 推理前 _先检索相关文档_ 加进 context** 的范式。解决 LLM 知识 stale + 私有数据问题。是 _推理时增强_,不改模型。

**vs 相邻**:
- vs **fine-tune**:RAG 知识在 _外部 / 动态_;fine-tune 知识在 _模型参数 / 静态_
- vs **memory**:memory 偏 _agent 状态_(主动写);RAG 偏 _文档检索_(被动查)

**wiki**:[[concept/rag-design-space]] 8 维 / 6 个 RAG framework(dify / llama_index / ragflow / ms-graphrag / cognee / pathway)

### retriever
**中文**:检索器 | **English**:retriever | **我固定叫**:retriever | **缩写**:无

**定义**:**RAG 系统内 _负责检索文档_ 的组件**。常见类型:vector search / BM25 / hybrid / KG traversal / multi-step decomposition。

**vs 相邻**:
- vs **embedder**:embedder 算 vector,retriever 用 vector 查
- vs **reranker**:retriever 召回 top-K,reranker 重排

**wiki**:[[eval/llama_index]] 18+ Query Engine 含 retriever / [[eval/cognee]] 22 Retriever / [[harness/ragflow]] retrieval

### embedding
**中文**:嵌入向量 | **English**:embedding (vector) | **我固定叫**:embedding | **缩写**:无

**定义**:**把文本(/ 图像 / 音频)映射成 _固定维度向量_** 的过程或产物。是 vector search 的基础。常见 dim:384(MiniLM)/ 768(BERT)/ 1536(OpenAI ada-002)/ 3072(text-embedding-3-large)。

**vs 相邻**:
- vs **vector**:embedding 通常是 vector,但 vector 还可来自其他(Hash-based, ...)
- vs **encoding**:encoding 是通用过程;embedding 是 _semantic-preserving 向量化_

**wiki**:[[eval/llama_index]] 66 embeddings / [[concept/rag-design-space]] 8 维之一

### vector store
**中文**:向量存储 | **English**:vector store / vector database | **我固定叫**:vector store | **缩写**:**VDB**(偶尔)

**定义**:**专门存 embedding + 高效相似度搜索的数据库**(Pinecone / Weaviate / Qdrant / Chroma / pgvector / Milvus / Lance / FAISS)。

**vs 相邻**:
- vs **KG store**:VDB 适合语义相似;KG 适合多跳推理
- vs **traditional DB**:VDB 优化 ANN(approximate nearest neighbor)search

**wiki**:[[eval/llama_index]] 78 vector_stores / [[overview]] §3 决策 1 SkillForge pgvector 路线

### chunk (chunking)
**中文**:切片(切分)| **English**:chunk / chunking | **我固定叫**:chunk(单)/ chunking(过程)| **缩写**:无

**定义**:**RAG 把文档切成 _可被 embedding + retrieve 的小片段_**。切法影响 retrieval 质量。常见策略:fixed-size / sentence / paragraph / structural(按 markdown 标题)/ semantic(按内容含义)。

**vs 相邻**:
- vs **document**:document 是 _完整文档_,chunk 是 _切片_
- vs **memory block**:chunk 被动切;block 主动写

**wiki**:[[harness/ragflow]] 15 模板按文档类型 chunking / [[harness/openhuman]] Memory Tree ≤3k token chunks

### reranker (reranking)
**中文**:重排器(重排)| **English**:reranker / reranking | **我固定叫**:reranker | **缩写**:无

**定义**:**retriever 召回 top-N 后,reranker 用 _更精的模型_ 重新排序**(通常 cross-encoder 风,比 retriever 慢但更准)。可大幅提升 retrieval 质量。

**vs 相邻**:
- vs **retriever**:retriever 召回快,reranker 重排精
- vs **LLM-as-judge**:LLM 也可做 rerank,但比 cross-encoder 慢贵

**wiki**:[[eval/zep-graphiti]] Cross Encoder reranking 业内独家 / [[harness/ragflow]] rerank 模块 / [[harness/dify]] rerank(model + weighted_score)

### KG
**中文**:知识图谱 | **English**:Knowledge Graph | **我固定叫**:KG | **缩写**:**KG**

**定义**:**用 _entity + relationship_ 三元组组织的结构化知识**。LLM 时代复兴(GraphRAG / Cognee / Graphiti)— 因 LLM 能自动抽 entity + relationship。

**vs 相邻**:
- vs **vector store**:KG 强结构(多跳推理);vector 弱结构(语义相似)
- vs **KG-memory**:KG-memory 是 _KG 用作 memory_;KG 笼统(可作知识库)

**wiki**:[[concept/rag-design-space]] / [[eval/ms-graphrag]] / [[eval/zep-graphiti]] / [[eval/cognee]]

---

## I. Safety & Attribution(10)

### safety
**中文**:安全 | **English**:safety | **我固定叫**:safety | **缩写**:无

**定义**:**防止 agent 造成有害行为的总称**(数据泄露 / 误删 / 调用危险工具 / 被 prompt injection)。在 wiki 是 harness 13 维必填字段,含多 sub-field。

**vs 相邻**:
- vs **security**:safety 偏 _行为后果_(误操作 / 数据泄露);security 偏 _攻击防御_(prompt injection / 入侵)。常混用
- vs **guardrail**:guardrail 是 _safety 的具体实现机制_

**wiki**:每个 [[harness/*]] `safety` 维度(强制)/ [[harness/goose]] security 6 文件双向防御

### permission
**中文**:权限 | **English**:permission | **我固定叫**:permission | **缩写**:无

**定义**:**agent 调用 tool / 访问资源时的授权机制**。常见模式:always-allow / always-ask / allow-once / deny。Claude Code 5 permission modes + YOLO ML 路线。

**vs 相邻**:
- vs **sandbox**:permission 是 _应用层授权_(framework 决定);sandbox 是 _OS 层隔离_(进程 / namespace / VM)
- vs **acl**:acl 偏 _静态规则_;permission 偏 _runtime 决策_

**wiki**:[[harness/claude-code]] 5 permission modes / [[harness/goose]] permission 5 文件 / [[concept/sandbox-isolation]]

### sandbox
**中文**:沙箱 | **English**:sandbox | **我固定叫**:sandbox | **缩写**:无

**定义**:**把 agent 工具执行隔离在受限环境** 防止误伤主系统。详见 [[concept/sandbox-isolation]] 6 阶梯:无隔离 / namespaces(bwrap+landlock)/ Seatbelt+AppContainer / OCI 容器 / gVisor / microVM Firecracker。

**vs 相邻**:
- vs **permission**:见上
- vs **container**:container 是 sandbox 实现的一种;sandbox 笼统

**wiki**:[[concept/sandbox-isolation]] 6 阶梯 + 8 维 / [[harness/ragflow]] gVisor 业内独家 / [[harness/codex]] 3 平台沙箱

### guardrail
**中文**:护栏 | **English**:guardrail | **我固定叫**:guardrail | **缩写**:无

**定义**:**runtime 拦截 _agent 不该做的输入/输出_ 的机制**。可在 LLM input(prompt injection 检测)/ output(harmful content)/ tool call(危险参数)三处拦截。

**vs 相邻**:
- vs **safety**:guardrail 是 _safety 的具体实现_
- vs **moderation**:moderation 偏 _内容审查_(主要文本);guardrail 偏 _行为拦截_(含 tool call)

**wiki**:[[eval/opik]] guardrails-backend 独立 service / [[eval/agentops]] 12 SpanKind 含 **GUARDRAIL** / [[harness/ag2]] `agentchat/group/guardrails.py`

### moderation
**中文**:内容审查 | **English**:moderation | **我固定叫**:moderation | **缩写**:无

**定义**:**对 agent 输入/输出做 _敏感内容审查_**(暴力 / 色情 / 政治 / PII)。通常用 classifier 或 LLM 判定。

**vs 相邻**:
- vs **guardrail**:见上(moderation ⊂ guardrail,聚焦内容)
- vs **filter**:filter 偏 _规则_;moderation 偏 _模型判定_

**wiki**:[[harness/dify]] `core/moderation/` 独立模块业内罕见

### prompt injection
**中文**:提示注入 | **English**:prompt injection | **我固定叫**:prompt injection | **缩写**:**PI**(偶尔)

**定义**:**攻击者通过 _输入文本_ 操纵 LLM 行为**(_"Ignore previous instructions and..."_)。是 LLM 应用最常见安全威胁。防御方法:input scanner / system-reminder 协议层不可信 / classification 等。

**vs 相邻**:
- vs **jailbreak**:jailbreak 偏 _绕过 model alignment_;prompt injection 偏 _操纵 framework / agent 行为_
- vs **adversary attack**:adversary attack 笼统(含 PI / DoS / data poisoning);PI 是 _最常见子类_

**wiki**:[[harness/goose]] `security/adversary_inspector.rs` + `scanner.rs` PromptInjectionScanner / [[eval/dspy]] `UNSAFE_LM_STATE_KEYS` 防御

### egress
**中文**:出向 / 数据出境 | **English**:egress | **我固定叫**:egress | **缩写**:无

**定义**:**agent 把数据 _发往外部_ 的动作**(curl / wget POST / 发邮件 / push 到云)。**egress 防御**是检测 + 拦截可疑出向(防 exfiltration)。

**vs 相邻**:
- vs **ingress**:ingress = 入向(数据进来);egress = 出向(数据出去)。安全双向防御要双管
- vs **DLP (Data Loss Prevention)**:DLP 笼统(含 egress + 内部传播);egress 偏 _跨边界出向_

**wiki**:[[harness/goose]] `security/egress_inspector.rs` 业内罕见(regex domain + EgressDestination struct)

### red team / red teaming
**中文**:红队 / 红队测试 | **English**:red team / red teaming | **我固定叫**:red team | **缩写**:无

**定义**:**专门 _对抗性测试 agent / LLM_** 找漏洞的过程或团队。LLM 场景常用 25+ 攻击策略(crescendo / GCG / multilingual / mathPrompt / promptInjections)。

**vs 相邻**:
- vs **eval**:eval 看 _正常输入_ 性能;red team 看 _恶意输入_ 抵抗
- vs **fuzz testing**:fuzz 偏 _随机生成_;red team 偏 _策略性攻击_

**wiki**:[[eval/promptfoo]] 25+ red-team 策略业内最广 / [[backlog]] §2 P0 候选

### attribution
**中文**:归因 | **English**:attribution | **我固定叫**:attribution | **缩写**:无

**定义**:**把 _某个观察到的现象 / 分数_ 归因到 _某个原因 / 行为 / 个体_ 的过程**。LLM agent 场景至少 3 种细分:trace attribution(行为追溯)/ fault attribution(失败归因到 USER/AGENT/ENV)/ score attribution(score 归因到具体 reasoning step)。

**vs 相邻**(扩展对照 — attribution / label / score / provenance 易混):

| | **attribution** | **label** | **score** | **provenance** |
|---|---|---|---|---|
| 回答的问题 | _为什么_ 这样 / _谁_ 造成的 | _这个是什么类_ | _这个有多好_ | _数据从哪来_ |
| 时间方向 | 反向追溯 | 当前分类 | 当前打分 | 反向追溯(数据流)|
| 例子 | "失败是因为 ENV bug,不是 agent" | "这个 trace 标 hallucinated" | "answer score 0.87" | "这个 fact 来源 source X line 42" |
| wiki 出现 | [[concept/attribution]] | 同上 Layer A | 同上 Layer B | [[concept/agent-memory]] Graphiti episode |

**wiki**:[[concept/attribution]] 5 Layer + 7 步 / [[eval/tau-bench]] `auto_error_identification.py` `FaultAuthor × FaultType`

### provenance
**中文**:溯源 / 数据来源 | **English**:provenance | **我固定叫**:provenance | **缩写**:无

**定义**:**追溯数据 _从哪来 / 经过哪些处理_** 的元数据链。在 KG/memory 场景至关重要(每个 fact 都能溯源到 source episode / 文档 line)。

**vs 相邻**:见 **attribution** 扩展对照表。**关键差异**:provenance 追的是 _数据流_;attribution 追的是 _因果 / 归责_。

**wiki**:[[eval/zep-graphiti]] Episode provenance / [[concept/agent-memory]] / [[concept/attribution]]

---

## J. Observability(5)

### trace
**中文**:链路追踪 | **English**:trace | **我固定叫**:trace | **缩写**:无

**定义**:**一次完整调用的 _跨服务追踪记录_**(含 N 个 span,有 root + 父子关系)。LLM 场景 trace = 一个完整 user→agent→tool→...→reply 链。

**vs 相邻**:
- vs **span**:trace 含 N span(span 是 trace 的原子单位)
- vs **log**:log 是 _离散事件_(无关联);trace 有 _完整因果链_

**wiki**:[[concept/trace-based-eval]] 4 类 trace-based signal / [[concept/observability]] 三家路线 / 多个 [[eval/*]] OTel 集成

### span
**中文**:跨度 / 段 | **English**:span | **我固定叫**:span | **缩写**:无

**定义**:**trace 内一段 _有起止时间的操作单元_**(LLM call / tool call / DB query / RPC)。含 attributes / events / status。

**vs 相邻**:
- vs **trace**:见上
- vs **SpanKind**:SpanKind 是 _span 类型_(AgentOps 12 种含 WORKFLOW/SESSION/TASK/...)

**wiki**:[[eval/agentops]] 12 SpanKind / [[concept/agent-protocols]] §6 OTel GenAI semconv / [[concept/trace-based-eval]]

### log
**中文**:日志 | **English**:log | **我固定叫**:log | **缩写**:无

**定义**:**离散事件记录**(时间戳 + level + message + 可选 attributes)。OTel 1.0 三件套之一(trace / metric / log)。

**vs 相邻**:
- vs **trace**:log 离散,trace 有因果链
- vs **event**(Wide Events 2.0):event 是 _带丰富 attribute 的高维事件_,接近 log 但语义更宽

**wiki**:[[harness/codex]] `logging.rs` / [[concept/observability]] Wide Events 2.0 vs OTel 1.0 对照

### Wide Events 2.0
**中文**:宽事件 2.0 | **English**:Wide Events 2.0 | **我固定叫**:Wide Events 2.0(完整)/ wide events(简称)| **缩写**:无

**定义**:**Charity Majors / Burmistrov 路线 — 反对 OTel 1.0 三分法,推 _wide, richly attributed event_ 为主分析单位**。强调 high-cardinality / columnar / immutable append-only。

**vs 相邻**:
- vs **OTel 1.0**:OTel 1.0 是 trace+metric+log 三分;Wide Events 2.0 是 _以 event 为主_,trace 只是 correlation handle
- vs **log**:log 通常 thin(几个字段);wide events 通常 rich(数十字段)

**wiki**:[[concept/observability]] 11 原则(Langfuse `.agents/ARCHITECTURE_PRINCIPLES.md:13-32`)/ [[overview]] §2 判断 5

### OTel semconv
**中文**:OTel 语义约定 | **English**:OTel semantic conventions | **我固定叫**:OTel semconv | **缩写**:**semconv**

**定义**:**OpenTelemetry 定义的 _attribute 命名标准_**(`http.method` / `db.system` / `gen_ai.system` / ...)。让不同 vendor 的 trace / metric / log _跨家可比_。

**vs 相邻**:
- vs **OTel SDK**:SDK 是实现,semconv 是命名规则
- vs **OpenInference**:Phoenix 主推的 LLM 专用 semconv,跟 OTel `gen_ai.*` semconv 并行

**wiki**:[[concept/agent-protocols]] §6 / [[eval/agentops]] OTel semconv 子模块 12 SpanKind + Legacy 3 sub-kind

---

## K. Self-evolution(6)

### self-evolution
**中文**:自进化 | **English**:self-evolution | **我固定叫**:self-evolution | **缩写**:无

**定义**:**agent / framework 在运行中 _自我改进 prompt / skill / 行为_**。详见 [[concept/self-evolution]] 4 类:Runtime 提炼 / Meta(dev-facing)/ Runtime budget 内嵌 / Algorithm-driven prompt optimization。

**vs 相邻**:
- vs **fine-tune**:fine-tune 改模型参数;self-evolution 通常改 _prompt / skill / memory_(轻量)
- vs **prompt optimization**:prompt opt ⊂ self-evolution(第 4 类)

**wiki**:[[concept/self-evolution]] 4 类 / [[deep/claude-code-autodream-3-gate]] / [[deep/openclaw-memory-dreaming-6-dim]] / [[overview]] §2 判断 2

### fine-tune
**中文**:微调 | **English**:fine-tune / fine-tuning | **我固定叫**:fine-tune | **缩写**:**FT**(偶尔)

**定义**:**用领域数据 _更新模型参数_** 让模型适应特定任务/风格。比 RAG / few-shot 重,但效果更深。LoRA / QLoRA 是参数高效 FT 方法。

**vs 相邻**:
- vs **RAG**:见 RAG 节
- vs **prompt optimization**:prompt opt 改 _输入_;fine-tune 改 _参数_

**wiki**:不单独成页,在 [[concept/prompt-optimization]] / [[concept/self-evolution]] 提及

### prompt optimization
**中文**:提示优化 | **English**:prompt optimization | **我固定叫**:prompt optimization | **缩写**:**PO**(偶尔)

**定义**:**算法自动 _找最优 prompt_** 的过程(遗传 / Bayesian / Meta-prompting / Pareto / 7+ 算法)。是 self-evolution 第 4 类。

**vs 相邻**:
- vs **fine-tune**:见上
- vs **self-evolution**:prompt opt ⊂ self-evolution

**wiki**:[[concept/prompt-optimization]] 6 维 / [[eval/opik]] 7 算法 / [[eval/dspy]] 15 optimizer / [[overview]] §3 决策 7

### reflection
**中文**:反思 | **English**:reflection | **我固定叫**:reflection | **缩写**:无

**定义**:**agent 跑完任务后 _回顾自己的输出_ 找改进点**。可在 turn 内(self-reflection)或后台(background reflection 写进 memory)。

**vs 相邻**:
- vs **summary**:summary 是 _浓缩_;reflection 是 _评价 + 改进_
- vs **judge**:judge 评第三方;reflection 评自己

**wiki**:[[harness/openhuman]] `learning/reflection.rs` / [[eval/langmem]] `ReflectionExecutor` Local+Remote 双执行器

### RL / RLHF / RLAIF
**中文**:强化学习(各家)| **English**:RL / RLHF / RLAIF | **我固定叫**:RL(笼统)/ RLHF(human feedback)/ RLAIF(AI feedback)| **缩写**:**RL** / **RLHF** / **RLAIF**

**定义**:**强化学习训练范式**:
- **RL**:agent 跟环境互动,奖励信号优化策略
- **RLHF**:人工对模型输出排序作奖励信号(ChatGPT-style 训练)
- **RLAIF**:LLM 自己当 judge 替代人工(Anthropic Constitutional AI 路线)

**vs 相邻**:
- vs **fine-tune**:RL 是 _带奖励信号的 fine-tune_
- vs **self-evolution**:RL 改参数,self-evolution 通常改 prompt

**wiki**:[[harness/agentscope]] Trinity-RFT agentic RL training 业内独家(`tuner/` 10 文件)

### distillation
**中文**:蒸馏 | **English**:knowledge distillation | **我固定叫**:distillation | **缩写**:无

**定义**:**用大模型(teacher)输出训练小模型(student)** 让小模型学到大模型的知识。LLM 场景常用 GPT-5/Claude Opus 蒸馏到 Llama 3 70B / Mistral 7B。

**vs 相邻**:
- vs **fine-tune**:fine-tune 用 _人工标_;distillation 用 _大模型标_
- vs **RLAIF**:RLAIF 是 _LLM 当 judge_;distillation 是 _LLM 当 teacher 出 ground truth_

**wiki**:不单独成页,偶尔在 [[concept/agent-eval]] 提及 tau-bench / SWE-bench dataset 来源

---

## 变更日志

- **2026-05-18 v1**:首版创建 — 23 术语 × 7 区,基于用户列的 15 个核心 + 8 个高频相邻
- **2026-05-18 v2**:扩到 **101 术语 × 11 区**(覆盖整个 wiki 高频术语) — 加 9 区 / 78 新术语:agent 抽象补 9(agentic / runtime / SDK / provider / channel / session / turn / step / loop)+ tool 协议补 7(tool_use / tool_result / A2A / ACP / AG-UI / AGNTCY / OTel)+ skill 补 4(extension / command / lifecycle event / recipe)+ context 补 9(context window / system-reminder / instructions / prompt / template / few-shot / prompt cache / compaction / summary)+ memory 6 / multi-agent 8 / RAG 7 / safety 9(safety / permission / sandbox / guardrail / moderation / prompt injection / egress / red team / provenance)+ observability 5 / self-evolution 6;原 G "高阶" 拆分:attribution 移入 I (Safety),A/B test 移入 F (Eval)
