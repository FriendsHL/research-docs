---
name: overview
type: meta
updated: 2026-05-18
status: evolving
---

# Overview — Evolving Thesis

> 这是 wiki 的**全局 thesis**(schema §2 设计的 `overview.md`),把 13 harness + 16 eval + 5 framework + 22 concept + 11 deep ingest 成果汇总成持续演化的核心判断。
>
> 任何后续 ingest 应**回灌这一页**:新发现支持现有判断 → 强化;反证 → 修正;新增维度 → 扩展。

---

## 1. Wiki 状态 snapshot(2026-05-18 第 4 版,Step 4 overview 升级:13 + 16 + 5 + 22 + 11 + 35 = 102 主页)

| 类型 | 数量 | 内容 |
|---|---|---|
| **harness/** | **13** | hermes-agent / claude-code / openclaw / codex / opencode + **agentscope / crewai / dify / langgraph / metagpt / openhuman / ragflow / smolagents**(2026-05-16 后 +8) |
| **eval/(eval-framework)** | **16** | langfuse / opik / swe-bench / tau-bench / dspy / promptfoo / ragas / phoenix / helicone / langchain / mem0 / letta / mempalace / langmem / zep-graphiti + **agentops**(2026-05-17 OTel agent semconv 12 SpanKind)|
| **eval/(`type: framework` ~ 2026-05-18 schema v2)** | **5** | **ag2 / cognee / llama_index / ms-graphrag / pathway**(放在 eval/ 路径但属 framework,详见 [[schema]] §3.2b)|
| **concept/** | **22** | 原 13 + **agent-harness(10 维 cross-harness)/ agent-protocols(5 层协议栈:MCP+A2A+AGNTCY+AG-UI+OTel)/ agent-team-patterns(3 哲学 × 4 组织模式)/ rag-design-space(8 维 / ICLR'26 GraphRAG calibration)/ agent-state-separation(AgentScope StateModule 132 行 PyTorch 风)/ runtime-context-injection(system-reminder 协议)/ sandbox-isolation(6 阶梯 + Firecracker 125ms)/ attribution(5 Layer + 7 步)/ trace-based-eval(4 类 signal)** |
| **deep/** | **11** | claude-code(autodream / compact-tool-result / insights)/ codex(actor-resume-agent / code-mode-v8-isolate)/ openclaw(memory-dreaming-6-dim)/ opencode(lsp-first-class)/ dify(workflow-as-tool)/ ragflow(deepdoc-table-recognizer)/ google-a2a-spec / mcp-spec-deep(2026-05-17 新引入页类型,详见 [[schema]] §3.4b) |
| **source/** | **35** | 13 harness + 16 eval + 5 framework + 1 deep-extra(claude-code-insights)的源文件锚点(全 B 模式 source-grounded)|
| **meta/** | 6 | purpose / schema(v2)/ index / log / backlog / overview(本页) |
| **总行数** | **~50000 行**(预估,翻倍) | 含 frontmatter / 表格 / 代码片段 / 5 层协议栈对比 |
| **跨页一致性** | ✅ 100% | Hermes MAX_DEPTH=1 / Codex DEFAULT_AGENT_MAX_THREADS=6 / Hooks 8/16/27 三家阶梯 / OpenClaw 6 维 dreaming 权重(relevance 0.30 最重)— 跨 source/harness/concept/deep 全部一致 |
| **证据等级** | 主体 verified-from-source | 11 deep 页 100% 含 §WHY + §失败模式 + file:line 锚点 / 3 个 concept(agent-protocols / sandbox-isolation / attribution / trace-based-eval)hybrid 模式(B + WebFetch 外部 spec)|
| **R-XXX takeaway** | **123 项**, **21 家族** | R-RAG / R-MEM / R-HARNESS / R-TEAM / R-OBS / R-COG / R-RF / R-OH / R-INJECT / R-DIFY / R-INSIGHT / R-SMOL / R-AG / R-AS / R-CW / R-GR / R-LG / R-LI / R-MG / R-PW + R-ATTRIB / R-OBS(trace-eval),**SkillForge 决策直接落地清单** |

**调研对象覆盖度(2026-05-18 第 4 版后)**:
- ✅ 自进化 agent(Hermes / OpenClaw)
- ✅ 通用 coding agent(Claude Code 闭源 + opencode 开源 + smolagents code-mode)
- ✅ Cloud + Local 双形态(Codex Rust monorepo)
- ✅ **Karpathy llm-wiki 模式 production 实现**(OpenClaw + **openhuman** — wiki 内第 2 家)
- ✅ **平台型 eval / observability**(Langfuse + Opik + Phoenix + Helicone + LangChain/LangSmith + **AgentOps** OTel agent semconv)
- ✅ **业内 benchmark**(SWE-bench coding + tau-bench multi-turn)
- ✅ **prompt-as-program framework**(DSPy + Opik Agent Optimizer 7 算法)
- ✅ **CLI-first eval + red teaming**(Promptfoo)
- ✅ **RAG eval + RAG framework**(Ragas / **llama_index / ms-graphrag / pathway / cognee / ragflow / dify** 6 路线全覆盖)
- ✅ **Memory framework 7 路线**(mem0 / mempalace / letta / langmem / zep-graphiti + SkillForge + **openhuman Memory Tree**)
- ✅ **Multi-agent framework 3 哲学**(CrewAI Task-driven / AG2 Conversation-driven / MetaGPT Role-play-driven)
- ✅ **Workflow + LangGraph paradigm**(langgraph Pregel-StateGraph 实现 LangChain 之上)
- ✅ **Agent state separation**(AgentScope StateModule 132 行 PyTorch 风)
- ⏳ 待加(P0):**aider / openhands / openai-agents-sdk** — 3 个还没收的关键 harness
- ⏳ 待加(P1):superpowers / Inspect AI / Cursor / Goose / Mastra / Deep Agents

**eval 6 路线分类**(2026-05-18 第 4 版后,扩 framework 类型):
| 层级 | 代表 | 集成方式 |
|---|---|---|
| 1. **Platform**(SDK-first) | Langfuse / Opik / **AgentOps** | Python `@observe()` / JS `@track` SDK |
| 2. **Platform**(OTel-first) | Phoenix(OpenInference)/ AgentOps | OpenInference / OTel GenAI semconv + 12 SpanKind |
| 3. **Platform**(Gateway-first) | Helicone | OpenAI API baseURL 切换,0 SDK |
| 4. **Framework**(Python library) | DSPy / Ragas / LangChain | Python import |
| 5. **CLI-first / Benchmark / Library** | Promptfoo / SWE-bench / tau-bench | YAML config / Docker / Pydantic 数据模型 |
| 6. **Framework(non-eval)** ⭐ 2026-05-18 新 | **ag2 / cognee / llama_index / ms-graphrag / pathway** | 13 维 harness frontmatter,`type: framework`(详见 [[schema]] §3.2b)|

**Memory framework 7 路线分类**(2026-05-18 第 4 版后,openhuman 加为第 7,详见 [[concept/agent-memory]]):
| 路线 | 代表 | 关键特性 |
|---|---|---|
| 1. **LLM-extracted facts + ADD-only 累加** | mem0 v2 | LongMemEval 94.8 + Single-pass + Entity linking + 跟 OpenClaw plugin 联动 |
| 2. **Verbatim + Hierarchical scope** | MemPalace | LongMemEval 96.6 raw / ≥99% rerank(业内最高)+ Wings/Rooms/Drawers + 0 API call |
| 3. **MemGPT Agent OS + function-call memory ops** | Letta | Virtual context + Inner Thoughts + Heartbeat + **git 版本控制 memory block** |
| 4. **LangChain ecosystem memory utilities** | LangMem | LangGraph BaseStore + Background ReflectionExecutor + Prompt opt 一体 |
| 5. **Temporal KG + Episodes** | Zep+Graphiti | Validity windows(facts when true/superseded)+ 4 graph DB driver + Cross encoder rerank |
| 6. **LLM-SYNTHESIS proposal + 人审 gate + dedup** | **SkillForge**(自家)| MEMORY-LLM-SYNTHESIS + DREAM-CONSOLIDATION + ATTRIBUTION-AGENT,**业内独家** |
| 7. **Memory Tree(分层折叠)+ Obsidian vault 双写** ⭐ 2026-05-18 新 | **openhuman**(`memory/tree/schemas.rs:27-48`)| 17 JSON-RPC controller + SQLite(`rusqlite "bundled"`)+ `.md` 双写 + ≤3k token chunks + 评分 + hierarchical 折叠 + 118+ OAuth(经 Composio)+ 20min auto-fetch loop |

**RAG framework 6 路线分类**(2026-05-17~18 第 4 版形成,详见 [[concept/rag-design-space]]):
| 路线 | 代表 | 关键特性 |
|---|---|---|
| 1. **Visual Workflow + DSL** | dify | 用户友好 + workflow_as_tool 抽象(R-DIFY-1/2 P1)|
| 2. **Framework primitive + 13 metric** | llama_index | Query Engine 18+ explicit type + Composable Memory(R-LI-1)|
| 3. **Deep Document Understanding + gVisor** | ragflow | 16 parser + Table Structure Recognizer + 唯一 gVisor sandbox(R-RF-1/2 P1)|
| 4. **KG-as-memory + Community Report** | ms-graphrag | Microsoft GraphRAG 21 workflow pipeline + community-level context |
| 5. **Memory control plane + Decorator API** | cognee | `@decorator` agent memory + 25 modules + memify 7 pipeline(R-COG-1/2 P1 直接打到 SkillForge memory)|
| 6. **Differential Dataflow 增量计算** | pathway | 文档库变化只重算 delta + 内置 MCP server + BSL 1.1 license |

---

## 2. 学习视角:10 个核心判断(differentiator / strengths / what_done_well 跨页提炼,2026-05-18 第 4 版)

> 每条判断都对应 1-N 个 concept / deep 页深入展开。

### 判断 1:Multi-agent 模型已分化为 5 条路线(2026-05-18 +openhuman Fork 第 2 家,详见 [[concept/multi-agent-isolation]])

| 路线 | 代表 | 核心机制 | 跨 agent 状态 |
|---|---|---|---|
| **完全隔离 fire-and-forget** | Hermes / OpenClaw | `MAX_DEPTH=1` + `skip_memory=True` + 无 resume | 不可恢复 |
| **隔离 + Fork 二选** | Claude Code / **openhuman** | SubAgent 隔离 / Fork 继承 parent system prompt + cache(openhuman 为 Fork-style 第 2 家 verified)| Fork 共享 cache |
| **Actor mailbox + control + resume** | Codex | `DEFAULT_AGENT_MAX_THREADS=6` + `SessionSource::SubAgent(ThreadSpawn{depth})` 链 + `resume_agent` | **可中断恢复**(详见 [[deep/codex-actor-resume-agent]] BFS + SpawnReservation RAII)|
| **Tab + 工具触发跨切 + 自动生成** | opencode | 用户主动 Tab 切 build/plan + LLM 调 `PlanExitTool` + `Question.ask` 用户确认 + `Service.generate()` 用 ai SDK 自动生成 agent | plan 文件作为状态载体 |
| **Role-play / Task-driven / Conversation-driven 哲学路线** ⭐ 2026-05-17 新 | MetaGPT / CrewAI / AG2 | 同一问题 3 个不同 metaphor 引出 3 套 abstraction(详见 [[concept/agent-team-patterns]])| Sequential / Hierarchical / GroupChat / Role-play 4 大组织模式 |

**核心 insight**:**没有最好路线,只有最匹配 trust model 跟性能预算的路线**。多租户场景 → 隔离;personal agent → Fork(cache 共享 5-10× 便宜);长任务 → actor + resume;简单交互 → Tab 切换;**业务建模 → 选 metaphor 路线**(MetaGPT Role-play 给业务方友好 / CrewAI Task DAG 给类型安全 / AG2 GroupChat 5+1 给灵活)。SkillForge 应明确 trust model 后选路线。

### 判断 2:Self-evolution 有 4 类形态(详见 [[concept/self-evolution]] + [[concept/prompt-optimization]])

| 类型 | 目标对象 | 产出 | 触发主体 | 案例 |
|---|---|---|---|---|
| **1. Runtime 提炼**(事后)| 终端用户 agent | MEMORY.md / skill / prompt 修订 | LLM session 内或后台 | OpenClaw Memory Dreaming / Hermes commit_memory_session / Claude Code autoDream |
| **2. Meta(dev-facing)** | 开发者 | `.claude/rules/` / agent / command / skill | LLM 提议 → **人审** → 落 dev artifact | Claude Code insight(已知未 verify)/ SkillForge `/evolve` |
| **3. Runtime budget 内嵌**(无 artifact)| 当前 session 模型行为 | 无 artifact,运行时硬约束 + 模板注入 | 框架级 + budget_limited 信号 | **Codex Goals** + token_budget per goal + 5 OTEL metric |
| **4. Algorithm-driven prompt optimization**(runtime 无人审)| 下次 session 用的 prompt | prompt / few-shot / 参数(自动生效) | 算法迭代:遗传 / Bayesian / Meta-prompting / Pareto | **Opik Agent Optimizer** 7 算法(EvolutionaryOptimizer / FewShotBayesianOptimizer / MetaPromptOptimizer / GepaOptimizer / ParameterOptimizer / HierarchicalReflectiveOptimizer / HRPO) |

**核心 insight**:**4 类不互斥,可叠加**。SkillForge `/evolve`(类 2)+ Opik runtime optimizer(类 4)+ Codex Goals 风格 budget(类 3)三者组合是 SkillForge self-evolution 的 next horizon。

### 判断 3:Context Compaction 有 7 维设计空间,各家路线丰富(详见 [[concept/context-compaction]])

7 维:**触发时机** / **压什么** / **压成什么** / **怎么接回** / **边界保护** / **质量保护** / **下游通知**。

各家代表做法:
- **Hermes**:Phase A 廉价预处理(去重 / tool result 改一行)+ Phase B 结构化摘要(Active Task / Completed / Files / Critical Context / Remaining Work 5 段)+ `_sanitize_tool_pairs` 修复孤儿 tool_call/result 配对(`agent/context_compressor.py:1391`)
- **Claude Code**:5 级 compaction pipeline + `AUTOCOMPACT_BUFFER_TOKENS=13_000` + `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3`(BQ data:1,279 sessions × 50+ failures = 250K wasted API calls/day,真实 production 数据驱动)
- **Codex**:**3 套并存**(`compact.rs:585` + `compact_remote.rs:387` + `compact_remote_v2.rs:456`)+ `should_use_remote_compact_task(provider)` provider 路由 + **`InitialContextInjection::{BeforeLastUserMessage, DoNotInject}` 训练对齐注入位置**(把"模型训练时见到的位置"作为代码级硬约束)+ 7 个 telemetry 枚举
- **opencode**:`SUMMARY_TEMPLATE` 7 段强制 Markdown(Goal / Constraints / Progress(Done/InProgress/Blocked)/ Key Decisions / Next Steps / Critical Context / Relevant Files)+ **`PRUNE_PROTECTED_TOOLS=["skill"]`**(skill 调用 first-class memory)+ 4 条规则(含独家 "Do not mention the summary process or that context was compacted")

**核心 insight**:Compaction 不只是"删旧消息",而是**有结构、有边界保护、有持久化协议的工程**。Codex 的训练对齐注入位置是行业最精细做法,opencode 的 SUMMARY_TEMPLATE 强制结构是最易复制的工程模板。

### 判断 4:Hooks 数量阶梯反映工程文化(详见 [[concept/hooks-design]])

```
Codex 8 lifecycle  <  Hermes 16 valid_hooks  <  Claude Code 27 zod-validated
        精简                  中等                          细粒度
        反 over-engineering    Python plugin                Anthropic 演化产物
```

- **Codex 8**(`hooks/src/lib.rs:18-27`):PreToolUse / PermissionRequest / PostToolUse / PreCompact / PostCompact / SessionStart / UserPromptSubmit / Stop。其中 6 个 dispatch matcher,`Stop` / `UserPromptSubmit` 不用(显式区分)。独立 crate + `output_spill.rs` 防输出爆量 + `plugin_hook_declarations` 白名单声明
- **Hermes 16**(`hermes_cli/plugins.py:75-111`):Python 装饰器风格,plugin 自 filter event payload
- **Claude Code 27**(`entrypoints/sdk/coreTypes.ts:25-53`):每个新功能(autoDream / Coordinator / Fork SubAgent / Skills / 5 级 compaction)都对应 1-3 个 hook 事件,演化出 27;`ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']` 只有 2 个默认开,其他 opt-in

**核心 insight**:**hooks 数量上限跟 framework 内部演进自由度成反比**。8 个 hook framework 演进容易,27 个 hook 加新功能必加新 hook。SkillForge 当前 ~6 hook,**应学 Codex 路线起步**(精简 + output_spill + plugin 白名单声明)。

### 判断 5:LLM Observability 行业向 Wide Events 2.0 收敛(详见 [[concept/observability]])

**Wide Events 2.0**(Charity Majors 路线,Langfuse `.agents/ARCHITECTURE_PRINCIPLES.md:13-32` 11 条原则)反对 OTel 1.0 metrics+logs+traces 三分法:
- "Model **observations** as the primary analytical unit. A trace is a correlation handle, **not the only useful entry point**"
- "Prefer **wide, richly attributed events** over fragmented metrics, logs, and trace records"
- "Preserve **high-cardinality context** so users can slice, group, filter, and debug **unknown unknowns**"
- "Favor **immutable or append-oriented event records** for high-volume telemetry"
- "Design storage and query paths around **columnar access patterns**"
- "Real-time / near-real-time debugging workflows. Batch processing **should not make fresh production behavior invisible**"

**3 家路线**:
- **Langfuse**(Node + Postgres + ClickHouse + Redis + S3,Wide Events 2.0 哲学根据地)
- **Opik**(Java + MySQL + ClickHouse + Redis + S3 + 6 独立 app 服务化粒度业内最细 + 40M+ traces/day)
- **Codex**(runtime self-contained:`otel/` crate + `rollout/` JSONL + SQLite,跟中心化 platform 互补)

**ClickHouse 切换阈值**:< 10K traces/day Postgres 够,> 10K 切 ClickHouse(Langfuse production 经验)。

**核心 insight**:LLM 场景 metric 高 cardinality 爆炸,trace+log+metric join 成本高,**Wide Events 2.0 是 LLM observability 事实路线**。SkillForge trace 设计应走 wide events,留 ClickHouse 切换余地。

### 判断 6:Code Mode V8 isolate 是真新范式,但工程门槛高(详见 [[concept/code-mode-orchestration]])

**4 家里 Codex 一家做**:LLM 写 JavaScript 在 V8 isolate 跑(`code-mode/` crate,`rusty_v8` + `deno_core_icudata`),工具暴露成 `tools.<normalized_name>(...)` 全局,`store/load` KV 跨调用,`yield_control()` 流式吐结果。

vs 传统 JSON tool_use(其他 4 家):一次回合一个 tool,LLM 自己跨调用记状态。

**Code Mode 优势**:表达力(JS 完整语言)+ token 节省(一段 JS 几行 vs 多回合 JSON)+ 延迟(并行 + 跨调用 store)+ 熟悉度(JS 是 LLM 训练数据最多语言之一)
**Code Mode 代价**:嵌 V8 ~30MB 原生依赖 + 调试难度 + trace 可观测性差 + 安全表面 + 工具协议异质性(tool_use + Code Mode 双轨制)

**对立面**(opencode 路线,详见 [[concept/code-mode-orchestration]] 更新):**工具级 mode 切换** — framework 提供切换工具(`PlanExitTool`),LLM 调工具触发 framework 切 agent,**把决策上移到工具层**(vs Code Mode 上移到代码层)

**核心 insight**:这是模型生态差异塑造了 harness 设计 — Code Mode 是 OpenAI 独家是因为 OpenAI 模型的 code interpreter 能力强。SkillForge 不必跟,但应了解。

### 判断 7:`.agents/` dogfood 程度跟 LLM agent 友好度强相关(详见 [[concept/agent-runtime-stacks]])

| 项目 | `.agents/` 完整度 | 内容 |
|---|---|---|
| **Opik** | **最完整** | 6 agents + 10 skills + 4 .mdc rules + commands/comet + mcp.json + docs + `.codex` + `.cursor` symlink |
| **opencode** | 第二 | `.opencode/{agent,command,skills,plugins,tool,themes}/` + opencode.jsonc + tui.json |
| **Langfuse** | 第三 | 16 skills + ARCHITECTURE_PRINCIPLES.md(架构哲学 codify 给 LLM 读)|
| **Codex** | 第四 | AGENTS.md 17KB(LLM-readable contributor guide,反 god crate / 模块上限 500 LoC)|
| **Claude Code** | 第五 | `.claude/{agents,commands,rules,skills,settings.json}/` |
| **Hermes / OpenClaw** | 无 | (没有显式 .agents/ 目录)|

**模式**:**项目越成熟 + 越对 LLM agent 工具友好的,`.agents/` dogfood 越深**。Opik / opencode / Langfuse 是行业标杆,Codex 通过 AGENTS.md 17KB 做 LLM-readable contributor guide。

**SkillForge 启示**:仓库应建 `.agents/{agents, skills, rules, commands, mcp.json}/` 目录 + LLM-readable AGENTS.md + `.codex` / `.cursor` symlink to `.agents/`,**多 LLM 工具普适性 + 自己 dogfood Claude Code 模式**。

### 判断 8:Agent execution paradigm 已分化 4 种(2026-05-18 +langgraph 后形成,详见 [[concept/agent-runtime-stacks]] §"Agent Execution Paradigm")

| Paradigm | 代表 | 调度模型 | 状态载体 |
|---|---|---|---|
| **ReAct while-loop** | Claude Code / Codex / opencode / Hermes / OpenClaw | LLM 输出 tool_use → 跑 tool → tool_result → 喂回 LLM,直至 stop_reason=end_turn | message list / agent context |
| **Event-driven**(workflow) | llama_index Workflow / dify | event bus / step decorator,事件触发 step,step 产 event | workflow context / DAG state |
| **Actor model** | Codex / AgentScope | mailbox(mpsc + watch::Sender)+ SubAgent ThreadSpawn + 状态机 transition | actor state(per agent)|
| **Pregel-StateGraph** ⭐ 2026-05-18 新 | **langgraph** | Google Pregel BSP(Bulk Synchronous Parallel)+ 节点 + edge + checkpoint + StateGraph mutation | LangGraph State + Checkpoint 模块化 |

**Cross-cutting paradigm × tech stack 正交矩阵**(详见 concept 页):非互斥,**Codex 同时跑 ReAct + Actor**;**dify 同时跑 Event-driven + workflow_as_tool**;**langgraph 用 Pregel-StateGraph 实现 LangChain 上的 multi-agent**。

**核心 insight**:**Paradigm ≠ tech stack**,4 paradigm 可叠加,选哪种 paradigm 决定**状态可视化 / debug 难度 / 多 agent 同步语义**。SkillForge 默认 ReAct,如果引入 langgraph 风 checkpoint 则建议 Pregel 化(详见 R-LG-1~4)。

### 判断 9:Deep 页 §WHY + §失败模式 + file:line 是 wiki 知识浓度倍增器(2026-05-17 第 4 版新写法)

11 个 deep 页 100% 含 [[schema]] §3.4b 4 必修结构:**WHY**(为什么这么设计)+ **失败模式**(防了哪些 bug)+ **横向校准**(其他 harness 怎么做)+ **学习视角**(可选)。每页 file:line 锚点平均 30-50 处。

**为什么 deep 比 concept 更值钱**:
- **concept** 跨 5+ harness 横向 synthesis(广)
- **deep** 单 harness 单机制 depth-first(深),**直接 grep file:line 可 verify**
- 跟 Google `go/doc` 或 OpenAI `cookbook` 哲学一致:**最贵的不是"是什么",而是"为什么这么做 + 防了哪些 bug"**

**11 deep 页清单**:
- `claude-code-autodream-3-gate` — 4 文件 550 行如何撑 background memory consolidation(lock 文件 mtime 双语义 + 4-phase 强类型 procedural prompt)
- `claude-code-compact-tool-result` — `microCompact.ts` + `compact.ts` 2235 行 cache-editing 防 prompt cache miss + 98% measured 数据
- `claude-code-insights` — MULTIPLE TIMES algorithm + Behavioral Genome 多 layer + 用户行为捕获 + `/insight` 流
- `codex-actor-resume-agent` — 6 文件 6000+ 行 Rust actor + Mailbox 双 channel + SpawnReservation RAII + BFS resume
- `codex-code-mode-v8-isolate` — `code-mode/` crate 9 文件 4251 行 Rust + V8 启动删 4 危险全局 + 8 JS primitive
- `openclaw-memory-dreaming-6-dim` — 6 维加权评分(relevance 0.30 最重)+ 3-phase sleep + 反元评论 prompt
- `opencode-lsp-first-class` — `lsp/` 6 模块 + 9 LSP operations 暴露为 agent tool + 1-based LLM-ergonomic
- `dify-workflow-as-tool` — workflow 作为 tool 暴露的协议层抽象
- `ragflow-deepdoc-table-recognizer` — 16 parser + Table Structure Recognizer 业内最深
- `google-a2a-spec` + `mcp-spec-deep` — 2 个外部 spec hybrid 深挖,跟 [[concept/agent-protocols]] 5 层栈联动

**核心 insight**:SkillForge 内部技术文档应学 deep 页结构,**强制 §WHY + §失败模式 + 显式 file:line**(从根本上防"代码迁移时丢上下文")。

### 判断 10:协议层已收敛为 5 层栈(MCP+A2A+AGNTCY+AG-UI+OTel,详见 [[concept/agent-protocols]])

**5 层协议栈**(全 verified):
- **MCP**(tool 层,18 wiki 集成,**事实标准**)— 6 primitives 对称(Server: Resources/Prompts/Tools + Client: Sampling/Roots/Elicitation)
- **A2A**(agent 层,4 wiki 集成,2025 v1.0)— 4 primitives(Agent Card / Task / Skill / Message)+ _"不共享 internal memory/tools/logic"_ IP 保护
- **AGNTCY**(编排/网格层,Cisco+LF middleware **above** MCP/A2A)— OASF + SLIM + Agent Identity 加密
- **AG-UI**(UI 层,CopilotKit+LG+CrewAI,10+ 1st-party 集成)— **human-in-the-loop interrupts 协议级一等公民**
- **OTel**(telemetry 侧向贯穿,21 wiki 集成,**最广**)— GenAI semconv 含 MCP semantics

**3 个协议联盟健康姿态证据**:**ACP→A2A 合并**(2026-05-17 a2a-protocol.org 明示)/ AGNTCY 明示 _"middleware above"_ 不替代 / OTel GenAI semconv 纳入 MCP semantics。

**核心 insight**:**协议层不会"碎裂成 5 个独立标准"**,而是分层堆叠 + 联盟收敛。SkillForge 建议:**MCP/OTel 🔴 P0 / A2A 🟡 P1 / AG-UI 🟢 P2 / AGNTCY ⚪ P3 不必跟**。

---

## 3. SkillForge 视角:8 个关键决策建议(按优先级排)

> 详细分析见各 harness/eval 页 frontmatter `skillforge_takeaway`,这里只列**核心 8 条决策**。

### 优先级 1:必做(短期,1-3 个月)

#### 决策 1:Score 数据模型起步用 Langfuse 抽象(详见 [[eval/langfuse]])
- **Langfuse `Score = Source(API/EVAL/ANNOTATION) × DataType(NUMERIC/CATEGORICAL/BOOLEAN/CORRECTION/TEXT)` 3×5 矩阵**
- 配上 `PublicApiCreateScoreSourceDomain` 编译期 boundary(外部 API 不能创建 EVAL source,防伪造内部 evaluator output)+ `ANNOTATION` 必须带 configId + `TEXT_SCORE_MAX_LENGTH=500` 硬上限
- **不要走 Opik OOP 多态**(Java 子类爆炸风险),起步数据抽象更安全,后续按需 evolve
- 实现:Spring `@Entity Score` + zod 等价物(Spring `@Validated`)+ 公开 REST API enum 限制

#### 决策 2:`output_spill` 防 hook 输出爆量 +  8 lifecycle hook 精简版
- **学 Codex 8 hook 路线**(`hooks/src/lib.rs:18-27`):PreToolUse / PermissionRequest / PostToolUse / PreCompact / PostCompact / SessionStart / UserPromptSubmit / Stop
- **不要走 Claude Code 27 hook 路线**(细粒度难学难维护)
- 加 `output_spill` 等价物(防 hook 输出爆量)+ plugin 显式声明监听哪些事件(白名单注册)

#### 决策 3:Main + Secondary 双队列分流多租户(详见 [[eval/langfuse]])
- **Langfuse `IngestionQueue` + `IngestionSecondaryQueue` 模式**(`queues.ts:333-334`)
- 注释明确 "Separates high priority + high throughput projects from other projects"
- SkillForge 多租户场景必备,**避免高吞吐 project 拖慢其他**
- 实现:Spring `@Async` + 多 ExecutorService + project priority routing

### 优先级 2:重要(中期,3-6 个月)

#### 决策 4:LSP first-class 集成(详见 [[concept/lsp-integration]])
- **opencode `lsp/` 6 模块开源参考**(`client.ts` + `server.ts` + `diagnostic.ts` + `language.ts` + `launch.ts` + `lsp.ts` + `tool/lsp.ts`)
- Agent 可看 diagnostic / hover / definition / symbol — IDE 级语义理解
- JVM 嵌 LSP client 没问题(JSON-RPC over stdio)
- 起步:**TS LSP**(用户最广)POC,后续多语言抽象

#### 决策 5:Goals + token_budget per goal(对照 Codex)
- **Codex `core/src/goals.rs`**:`SetGoalRequest { objective, status, token_budget }` + 5 个 OTEL metric(`codex.goal.{created,completed,duration_seconds,token_count,budget_limited}`)+ 3 prompt template
- SkillForge eval pipeline 受益(每个 goal 内嵌 budget,trace 直接关联 goal lifecycle)
- 实现:Spring `@Entity Goal` + `MeterRegistry`(Micrometer / OTel)+ goal continuation prompt

#### 决策 6:Production loop 单线打通(对照 Langfuse)
- **`addDatasetRunItemsToEvalQueue.ts` 模式**:production trace → dataset → eval → score 全自动
- 配 EvalTemplate 版本控制(eval 跟 prompt 一样版本化,改 prompt 才能 reproducible 重跑)
- SkillForge "Prompt A/B + 自动晋升" 的开源参考实现

### 优先级 3:演进(长期,6-12 个月)

#### 决策 7:`/evolve` 现有 + Opik runtime Optimizer 互补(详见 [[concept/prompt-optimization]])
- SkillForge `/evolve` meta self-evolution(类 2)= 已有
- 加 Opik runtime optimizer 等价物(类 4):**先 POC 1-2 算法**(`GepaOptimizer` 多目标 + `FewShotBayesianOptimizer` 数据少)
- 不要早期复刻全 7 算法(工程量大)
- 借鉴 8 内置 benchmark dataset(gsm8k / truthful_qa / halu_eval / ...)用户 zero-config 跑

#### 决策 8:`.agents/` dogfood + LLM-readable AGENTS.md
- 仓库建 `.agents/{agents, skills, rules, commands, mcp.json, docs}/` 目录
- 写 SkillForge AGENTS.md(对照 Codex 17KB / opencode 4KB / Langfuse `ARCHITECTURE_PRINCIPLES.md` 11 原则)
- `.codex` / `.cursor` symlink to `.agents/` 多 LLM 工具普适性
- 内置 builtin agent(planner / build-fixer / code-reviewer / meta-auditor / test-runner / architect 风格,对照 Opik 6 agents)

### 不要走的路(避坑 5 条)

- ❌ **每个 provider 一个 crate / package**(Codex 路线,SkillForge SPI 抽象更轻量)
- ❌ **Opik OOP 多态评分模型 Java 子类越加越多**(类爆炸风险,起步用 Langfuse 数据抽象)
- ❌ **完全照搬 Opik 7 algorithm 工程量**(早期 POC 1-2 算法即可)
- ❌ **Effect-TS 风格代码移植到 Java**(语言无法移植,只能学 functional 哲学:避 try/catch / 早返回 / inline 变量 / Schema-first IO)
- ❌ **Code Mode V8 嵌入 JVM**(GraalVM JS 替代不现实,投入产出比低)

### SkillForge 已领先的(3 条)

- ✅ **Eval Pipeline + Prompt A/B + 自动晋升**(opencode / Codex / Hermes / OpenClaw 都没有)
- ✅ **`/evolve` meta self-evolution**(类 2,Opik runtime optimizer 是类 4 互补)
- ✅ **统一 Provider/Channel SPI 抽象**(比 Opik 每 provider 一 crate / Langfuse 23+ 集成 callback 更轻量)

---

### 优先级 0/1 R-XXX 必做清单(2026-05-18 第 4 版,15 条 from `/wiki-lint` 报告)

> 8 条决策(上文)= overview 级综合判断;15 条 R-XXX(下表)= 来源各 page `skillforge_takeaway` 编号,**每条都可 grep 回原页 verify**。详见 [[lint-report-2026-05-18]]。

| ID | 描述 | 来源 | 工程量 / 备注 |
|---|---|---|---|
| **R-AG-1** | 5+1 speaker selection 模式(auto/manual/random/round_robin/Callable) | [[eval/ag2]] | 架构启示 |
| **R-AS-1** | StateModule 抽象统一持久化(业内最干净) | [[harness/agentscope]] + [[deep/...]] | 132 行撬动 5 个 capability |
| **R-AS-2** | Interrupt fake-tool-result 补全 | [[harness/agentscope]] | 直接打到生产 |
| **R-COG-1** | Agent Trace + Feedback 自动入 KG | [[eval/cognee]] | 架构启示,**直接打到 SkillForge memory** |
| **R-COG-2** | Decorator-based Agent Memory API | [[eval/cognee]] | 集成抽象 |
| **R-CW-1** | Process enum 显式分支(sequential/hierarchical/consensual) | [[harness/crewai]] | 架构 |
| **R-DIFY-1** | Visual Workflow + DSL 双轨 | [[harness/dify]] | P1 product 决策 |
| **R-DIFY-2** | workflow_as_tool 抽象 | [[harness/dify]] | P1 architecture |
| **R-LI-1** | Query Engine 18+ 种 explicit type 抽象 | [[eval/llama_index]] | 架构启示 |
| **R-MG-1** | `investment + cost_manager.max_budget + NoMoneyException` 三件套 | [[harness/metagpt]] | production safety 业内独家 |
| **R-MG-2** | `Action` 抽象不同于 `Tool` | [[harness/metagpt]] | 架构启示 |
| **R-RF-1** | Deep Document Understanding 16 parser + Table Structure Recognizer | [[harness/ragflow]] | 架构启示 |
| **R-RF-2** | gVisor Sandbox 集成 | [[harness/ragflow]] | P1 safety |
| **R-SMOL-1** | `executor_type: Literal[...]` 抽象给 sandbox 选型 | [[harness/smolagents]] | 1 周工程量 |
| **R-HARNESS-1 / R-HARNESS-6** | LSP first-class + PreToolUse/PreCompact hooks | [[concept/agent-harness]] | 各 1-3 周 |

**总体**:8 条 overview 决策 + 15 条 R-XXX P0/P1 = **23 条最高优先级**。其余 P2 应做 45 条 + P3 远期 63 条 详见 [[lint-report-2026-05-18]]。

---

## 4. Wiki 维护原则(从 6 ingest 实践提炼)

### 原则 1:B 模式必须(`/wiki-ingest` skill 已 codify)

A 模式 hallucinate 风险已被 Hermes ingest 实践验证:
- A 模式 wiki 自编 `MAX_DEPTH=2`(实际 1)
- A 模式 wiki 自编 `approvals.mode = ["on","auto","off"]`(实际含 `smart`)
- A 模式 wiki 错 7 vs 6 terminal backends(笔记错,B 模式补)

**B 模式机制**:每个 verified 事实带 `[code: file:line]` 锚点,**lint 时可 grep 验证**。

### 原则 2:双视角 takeaway 不可偏废(purpose.md §1+§2)

每个 harness/eval 页必须填:
- `skillforge_takeaway`:工程视角(可借鉴 / 业内验证 / SkillForge 领先 / 待决策 / 风险)
- `learning_takeaway`:学习视角(differentiator / strengths / what_done_well)

**两者互补**:学习视角让 SkillForge 看到"为什么这家这么设计"(motivation),工程视角让学习有 actionable 落点。

### 原则 3:版本时效 + 4 级证据等级让 wiki 自带"知道自己 stale"能力

- `last_ingested_version` + `last_ingested_at` + `community_known_newer` + `known_post_ingest_features` 4 字段
- 4 级证据:`[verified-from-source]`(B 模式 grep)/ `[doc-only]`(README / 官方 docs)/ `[notes-only]`(用户笔记)/ `[community-knowledge]`(已知未 verify)
- **lint 规则**:`[doc-only]/[notes-only] > 3 = warning` / `[community-knowledge] > 2 = warning` / `last_ingested_version > 6 months = info`

### 原则 4:跨页一致性靠 grep + 关键常量

每次 ingest 后跑 `/wiki-lint` 等价 grep 验证:
- 关键常量(MAX_DEPTH / DEFAULT_AGENT_MAX_THREADS / HOOK_EVENT_NAMES 数量)跨 source/harness/concept 一致
- 矛盾时 source 页是真理(B 模式 verify),其他页跟 source 同步

### 原则 5:Iron Law 写入边界(schema §1)

**LLM 只能在 `agent-harness-wiki/` 内写文件**。
其他目录(`research/Hermes源码研究/` / `research/claude code 源码/` / `research/OpenClaw源码研究/` / `research/anthropic-blog-translations/`)**只读**,通过 wikilink `[[../X源码研究/xxx]]` 引用。
例外:用户授权的 note sync(标 `verified_by:` frontmatter)。

### 原则 6:Two-Step Chain-of-Thought ingest

**Step 1 Analysis**(7 段):
1. 这是什么 / 2. 关键实体与概念 / 3. 13 维度初判 / 4. 与现有 wiki 的关系(印证/矛盾/新增)/ 5. SkillForge 启示初判 / 6. 建议产出的页面 / 7. 触发问题(写入 log.md)

**Step 1 输出后等用户确认**,然后 Step 2 生成文件。**不可省 Step 1**(防止生成无人审的内容)。

---

## 5. Open Questions / 待 ingest 候选(2026-05-18 第 4 版,清空已完成项)

### 紧急 - 需 verify 的现有 community-knowledge

- **Claude Code `insight` 实现细节** — ✅ 2026-05-17 完成(详见 [[deep/claude-code-insights]])
- **Claude Code `goal`** — ⏳ 待 verify,对照 Codex Goals
- **Claude Code `agent-view` / `agent-manager`** — ⏳ UI 跟 lifecycle 增强细节

### 用户后续可加的 harness 候选(按 wiki 当前缺口,P0/P1)

| 候选 | 类型 | 状态 | 原因 |
|---|---|---|---|
| **aider** | harness | ⏳ P0 待 ingest | 业内 most-popular 开源 coding agent |
| **openhands** | harness | ⏳ P0 待 ingest | All Hands AI 出品,多 agent 平台 |
| **openai-agents-sdk** | harness | ⏳ P0 待 ingest | OpenAI 官方 agent SDK |
| **superpowers** | harness | ⏳ P1 | Anthropic 出品,跟 Claude Code 关系待 verify |
| **Inspect AI** | eval | ⏳ P1 | UK AISI 出品的 LLM eval framework |
| **Cursor / Windsurf 源码** | harness | ⏳ P2 | (闭源,可能只能 doc-only) |
| **Gemini CLI / Goose** | harness | ⏳ P2 | Google / Block 的 coding agent |
| **Mastra / Deep Agents** | harness | ⏳ P2 | TS multi-agent framework |

**已完成不再列**:DSPy(2026-05-16)/ SWE-bench / tau-bench(2026-05-16)/ Phoenix / Helicone / LangChain(2026-05-16)/ CrewAI / AG2 / MetaGPT / AgentScope / Dify / RAGFlow / LlamaIndex / MS-GraphRAG / Cognee / Pathway / LangGraph / smolagents / openhuman(2026-05-17~18)。

### Wiki 当前未覆盖维度(可能新建 concept)

- **Skills 系统**:Claude Code Skills 6 LoadedFrom / Codex `skills` crate / opencode `tool/skill.ts` — 三家 skill 体系横向对比未做
- **MCP 集成深度**:5+ harness 都集成 MCP 但深度差异大,未单独成 concept(已有 [[deep/mcp-spec-deep]] spec 视角,缺 harness 集成横向视角)
- **Permission 模型(细)**:Codex 3 平台沙箱 + opencode `Permission.Ruleset` + Claude Code 5 permission modes — 横向 concept 未做(已有 [[concept/sandbox-isolation]] 6 阶梯隔离,但 permission ≠ sandbox)
- **Auth / Identity**:Codex `agent-identity/` crate + Opik `apps/opik-backend/.../identity/` + opencode `packages/identity/` — 多租户 identity 模型未做
- **Plugin / Extension SDK**:opencode 4 类(shell/tool/tui/plugin)+ Codex 27+ extensions + Hermes plugin 装饰器 + OpenClaw 30+ extensions — 横向对比未做
- **Cancellation / Cleanup**:Codex `AgentControl` 状态机 + opencode `Question.RejectedError` + Hermes 等 — 横向 cancel 模型未做

---

## 6. Wiki 演化的元判断(meta-thesis,2026-05-18 第 4 版)

随着 ingest 推进,wiki 本身**就是一个学习 + SkillForge 决策的活产物**。它不是静态文档,而是:

1. **Schema 持续演化**(2026-05-16 v1 13 维 → 2026-05-18 v2 加 `type: framework`)— **wiki 自身的设计也是 evolving thesis 的一部分**
2. **跨页交叉引用密集**(平均每 concept 入站 3-5 次,每 harness 入站 4-14 次;openhuman/langgraph 反向链 2026-05-18 清空积压)— **wiki 是图,不是树**,通过 wikilink 网络支持任意角度查询
3. **新增页类型 deep/**(2026-05-17 第 4 版引入)— 解决 concept synthesis 跨 5+ harness 太广、单机制深挖跑不动的问题,11 个 deep 页 §WHY + §失败模式 + file:line 是 wiki 知识浓度倍增器(详见判断 9)
4. **24 小时 14+ ingest wiki 已 reshape concept 多次**:
   - multi-agent-isolation:2 路线 → 4 路线 → **5 路线**(openhuman Fork 第 2 家 + role-play/task-driven/conversation-driven 哲学路线)
   - self-evolution:1 类 → **4 类**(加 Opik runtime optimizer 类 4)
   - agent-eval:8 维补全 → 加 Opik OOP 多态 + AgentOps OTel 12 SpanKind
   - context-compaction:7 维补全 → 加 Codex 训练对齐 + opencode SUMMARY_TEMPLATE
   - agent-memory:5 路线 → **7 路线**(加 SkillForge LLM-SYNTHESIS + openhuman Memory Tree)
   - agent-runtime-stacks:5 编程哲学 → 加 **Agent Execution Paradigm**(ReAct / Event / Actor / **Pregel-StateGraph**)
   - 新增 concept(2026-05-17~18):agent-harness / agent-protocols / agent-team-patterns / rag-design-space / agent-state-separation / runtime-context-injection / sandbox-isolation / attribution / trace-based-eval = **9 个新 concept**
5. **B 模式 source-grounded 是 wiki 信任基础**,未来任何"AI 加速 wiki"的尝试不能放弃 file:line 锚点。deep 页把这个标准推到极致(平均每页 30-50 处 file:line)
6. **123 个 R-XXX takeaway action items 是 SkillForge 决策直接落地清单**,本身就是 wiki 价值的具象化

**核心元判断**:**Wiki 价值 = 跨 harness 横向连接的密度 × 每个事实的 verified 程度 × 双视角(学习 + SkillForge)的覆盖完整度 × deep 页深挖密度**。

任何后续 ingest 都应**强化这 4 个维度**(2026-05-18 加第 4 维 deep 页深挖密度)。
