---
name: crewai
type: harness
status: active
language: python
last_ingested_version: 1.14.5a6
last_ingested_at: 2026-05-17
last_commit: a95d26763f4766b1a4f7c19c039133d1202dbdaa
community_known_newer: false
known_post_ingest_features:
  - "Process.consensual (TODO 已标在 process.py:11,未实现)"

sources:
  - "[[source/crewai]]"
  - "/Users/youren/myspace/crewai/"

agent_loop: |
  Crew.kickoff() 触发. Process enum 决定循环模式:
    sequential: Task 按顺序串行, 前 task output 喂给后 task
    hierarchical: Manager agent (manager_llm / manager_agent) 分配 task 给 worker agents, 可并行
    consensual: TODO 未实现 (consensus 协议)
  Agent loop 在 agents/crew_agent_executor.py + step_executor.py
  step_callback + task_callback + before/after_kickoff_callbacks 4 个 hook 粒度

tool_protocol: |
  BaseTool + CrewAI 自家 tool framework + MCP 集成 (`mcp/` 子目录).
  task.tools 可 task-level 限制 (manager 不能让 worker 越权调任意 tool).
  function_calling_llm 配置单独 LLM 跑 tool call (vs main LLM 跑 reasoning).
  used_tools / tools_errors / delegations 跟踪 metric.

context_mgmt: |
  通过 task.context: list[Task] 显式传 task 间 context. 不做 in-context compaction.
  knowledge/ 模块提供 RAG-as-context.

memory: |
  ⭐ `memory/` 8 模块: unified_memory + encoding_flow + recall_flow + memory_scope + storage + analyze
  3 维 memory scope: short_term / long_term / entity
  embedder 跟 memory 强耦合, memory enable 时必须配 EmbedderConfig
  storage 支持多 backend (Chroma/Qdrant/Postgres/...)

multi_agent: |
  ⭐ **核心定位**. Crew = team of agents.
  Process.hierarchical 模式: manager_agent 分配 task delegate 给 worker (跟踪 delegations: int)
  Process.sequential: agent 按 task list 序串行
  Process.consensual: TODO (consensus 模式未实现)
  crews/ 子目录: 多 crew 协调 (跨 crew)

eval_loop: |
  Crew 不内置 eval framework. 通过 share_crew=True 可上报 crewai SaaS (CrewAI Plus) 跑分析.
  usage_metrics 跟踪 token 用量.
  step_callback + task_callback 可 custom eval logic.

self_evolution: |
  planning 模式 (`planner_observer.py`) 让 crew 在执行前先生成 plan, 提升复杂 task 成功率.
  experimental/ 子目录含实验功能 (train_crew 等).
  share_crew 路径: 上报 crew config + execution → CrewAI SaaS 训练改进 (用户可选).

observability: |
  events/ 模块 event system + usage_metrics + UsageMetrics.
  跟外部 (Langfuse/Phoenix/Helicone) 集成可通过 hooks.

hooks: |
  hooks/ 子目录 + 4 个 lifecycle callback:
    step_callback (每 step 后)
    task_callback (每 task 后)
    before_kickoff_callbacks (crew kickoff 前)
    after_kickoff_callbacks (crew kickoff 后)

safety: |
  security/ 模块 + security_config (SecurityConfig) 含 fingerprinting (agent identity 防伪造).
  task.allow_crewai_trigger_context 三态控制 trigger payload injection.
  task-level tools 限制 防 worker 越权.

provider: |
  llm.py + llms/ 子目录, 支持多 LLM provider (通过 LangChain LLM 抽象 + 自家 wrapper).
  manager_llm / function_calling_llm / chat_llm 可分别配不同 model.
  100+ provider 通过 LangChain 集成 (LangChain 是 transitive dep).

channels: |
  Python library (主) + CrewAI Plus SaaS (plus_api.py) + cli 工具 (crewai-cli).
  flow.py 含 human_feedback.py / async_feedback/ 支持 human-in-the-loop.

persistence: |
  config 可 JSON. memory storage 通过 storage/ 多 backend.
  crews/ 多 crew 编排 + project/ scaffolding (类 cookiecutter).

cancellation: |
  ? 未深 verify.
  max_rpm 限流 (防 LLM API 限流而非 cancel).

skillforge_takeaway: |
  CrewAI 是业内 production agent team 事实标准, 跟 SkillForge skill+agent 路线有 4 个借鉴点.

  🔴 **R-CW-1 — Process enum 显式分支 (sequential/hierarchical/consensual)**(架构)
     SkillForge 当前 SubAgent 是单一 spawn pattern, 没有显式 "team 协作模式" 抽象.
     CrewAI Process enum 是 _"team 协作模式 = first-class type"_ 的简洁设计.
     工作量: 远期参考 (SkillForge SubAgent 之上加 CrewConfig 抽象).

  🟡 **R-CW-2 — Task DAG via `context: list[Task]`**(类型安全 task 编排)
     SkillForge 当前 task 间传 context 路径未明确, CrewAI 用 task 实例引用是干净抽象.
     Task DAG = compile-time 验证依赖关系, 避免 runtime missing context bug.
     工作量: < 1 周 (加 task.context 字段 + dependency resolver).

  🟡 **R-CW-3 — 3 维 memory scope (short_term/long_term/entity)**(memory 架构)
     SkillForge memory_kind 三态 (observation/reflection/optimized) 是按 lifecycle 分,
     CrewAI scope 是按 _"who's memory"_ 分 (短期 = 当前 session / 长期 = 跨 session / entity = 特定实体).
     两者正交可叠加. 工作量: < 1 周.

  🟢 **R-CW-4 — task.allow_crewai_trigger_context 三态控制**(细粒度 context injection)
     SkillForge runtime context injection 现是隐式(每 task 都注),
     CrewAI 3 态 None/True/False 给 task 显式控制. 学 [[concept/runtime-context-injection]] 同思路.
     工作量: 配置项加一条.

  **❌ 不该跟**:
  - LangChain transitive dep (CrewAI 通过 LangChain 接 LLM, SkillForge 自家 OpenAi/Claude provider)
  - share_crew 上报 SaaS (SkillForge 自家平台, 不上报)
  - crews/ 多 crew 编排 (SkillForge SubAgent 已 cover)
---

# Harness — CrewAI v1.14.5a6

> CrewAI 是 **业内 production agent team 事实标准**(30k+ stars / MIT)。**Crew = team of agents + Tasks + Process**(sequential / hierarchical / consensual TODO)三件套。13 维详见 frontmatter。本页 narrative 重点:**Crew/Task/Process 抽象 + Manager-Worker hierarchical + 3 维 memory scope** 是 production agent team 必修课。

---

## 1. 项目定位(一句话)

**Production-ready agent team framework**(MIT,Python,30k+ stars)。**Crew + Tasks + Process** 是业内 agent team 的事实抽象。跟 SkillForge SubAgent 路线**互补**(SkillForge 是 spawn primitive,CrewAI 是 team organization)。

---

## 2. 核心抽象 — Crew/Task/Process 三件套

```
        ┌──── Crew ────┐
        │   (entity)    │
        │   agents[]   │
        │   tasks[]    │
        │   process    │ ───────── Process enum
        │   manager_*  │           ├─ sequential   (默认串行)
        │   memory     │           ├─ hierarchical (manager 分配)
        │   callbacks  │           └─ consensual   (TODO 未实现)
        └──────────────┘
                │
                │ kickoff()
                ▼
        ┌── Execution ──────┐
        │   For each task:   │
        │   - Sequential:    │── prev task output → next task context
        │   - Hierarchical:  │── manager_agent.delegate(task, worker)
        │     - track        │   ├─ delegations += 1
        │       delegations  │   └─ workers run in parallel
        │   - Tool calls     │── used_tools / tools_errors track
        │   - Callbacks      │── step / task / before / after kickoff
        └────────────────────┘
                │
                ▼
        ┌── Output ─────────┐
        │   TaskOutput      │── task.output_json/pydantic/response_model
        │   - file output   │── output_file + create_directory
        │   - structured    │── 3 种 structured output 方式
        └───────────────────┘
```

---

## 3. 三大独家技术点

### 3.1 Process enum — Team 协作模式 first-class type

```python
class Process(str, Enum):
    sequential = "sequential"    # Task → Task → Task 串行
    hierarchical = "hierarchical" # Manager 分配 task,worker 可并行
    # TODO: consensual = 'consensual'  # consensus 协议未实现
```

**关键对比**:
- 业内大部分 framework(LangGraph / AG2)需要用户**编写 graph / state machine** 描述协作
- CrewAI **3 enum 一键切换**,大部分用例够用
- TODO 标 `consensual` 说明 CrewAI 在演化(consensus 协议是论文级方向)

**Hierarchical 模式细节**:
- `manager_llm` 简单场景:给个 LLM 实例,自动生成 manager agent
- `manager_agent` 复杂场景:自定义 manager(可有 own tools / memory / role)
- `task.delegations: int` 跟踪委派次数(audit)

### 3.2 Task DAG via `context: list[Task]`(类型安全)

```python
research_task = Task(
    description="Research topic X",
    expected_output="Markdown report",
    agent=researcher,
)

write_task = Task(
    description="Write article based on research",
    expected_output="2000 word article",
    agent=writer,
    context=[research_task],   # ⭐ research_task 的 output 作为 write_task 的 context
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
)
```

**3 个优势 vs string template 风**(LangGraph / AG2 部分用法):
- **编译时验证**:`context=[research_task]` 是 Python 引用,IDE 跳转 + linter check
- **没有 string template 错乱**(`{{prev_output}}` 这种)
- **task 重命名安全**(IDE refactor 跟着改)

### 3.3 3 维 memory scope(`memory/memory_scope.py`)

```
MemoryScope:
  ├─ short_term  # 当前 session / kickoff 内
  ├─ long_term   # 跨 session 持久
  └─ entity      # 关联到特定实体(agent / user / project)
```

**业内对照**:
- [[eval/mem0]] 单一 memory(LLM-extract ADD-only)
- [[eval/letta]] Memory block(read-only / writable 区分,但 scope 不分)
- [[eval/mempalace]] Wings/Rooms/Drawers 层次化(类 entity scope)
- **CrewAI 3 维 scope 显式分** — 跟 SkillForge `memory_kind` 三态(observation/reflection/optimized)正交(SkillForge 按 lifecycle,CrewAI 按 _"who's memory"_)

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[concept/multi-agent-isolation]] | 扩展 | CrewAI Crew 是 _"team-as-entity"_ 抽象,跟 Claude Code SubAgent(spawn primitive)/ Codex actor(message primitive)正交 |
| [[concept/agent-memory]] | 加 7th 路线 | CrewAI _"3 维 scope(short/long/entity)"_ 跟 5 路线 + Dify pure-RAG + Cognee memify 7 共存,**第 7 路线 "scope-based"** |
| [[harness/dify]] | 互补 | Dify workflow_as_tool 是 _"workflow nest"_,CrewAI Crew 是 _"team organize"_,两层不同 |
| [[harness/agentscope]] | A2A 对照 | 都集成 Google A2A protocol(`a2a/` 子目录)|
| [[deep/codex-actor-resume-agent]] | 互补 | Codex 是 actor primitive,CrewAI 是 team abstraction,两层互补 |
| [[concept/agent-team-patterns]] | **核心贡献者** | CrewAI Process enum 是 4 大模式之一(Sequential + Hierarchical) |

---

## 5. R-CW 借鉴清单

详见 frontmatter `skillforge_takeaway`。总结:

| ID | 项 | 工作量 | 优先级 |
|---|---|---|---|
| **R-CW-1** | Process enum 显式分支(sequential / hierarchical)| 远期 | 🔴 P1 架构 |
| **R-CW-2** | Task DAG via `context: list[Task]` | < 1 周 | 🟡 P2 |
| **R-CW-3** | 3 维 memory scope(short/long/entity)| < 1 周 | 🟡 P2 |
| **R-CW-4** | `task.allow_crewai_trigger_context` 三态 | 配置项 | 🟢 P3 |

---

## 6. 已知 post-ingest 风险

- `crew.py` 2305 行只读了 init / fields(140 行),`kickoff()` / `_run_hierarchical` 等核心方法未深
- `task.py` 1468 行只读了 113-220 init / fields
- `flow/` 子目录(event-driven workflow,跟 Crew 并行抽象)未深
- `crews/` 多 crew 编排未读
- `experimental/` 实验功能未读
- `plus_api.py`(CrewAI Plus SaaS)未读

---

## 7. 引用

- [[source/crewai]] 完整源码锚点
- GitHub: `joaomdmoura/crewAI` 30k+ stars / MIT
- 文档:docs.crewai.com
- 跨页对照:[[harness/agentscope]] / [[harness/dify]] / [[deep/codex-actor-resume-agent]] / [[concept/agent-team-patterns]] / [[concept/agent-memory]] / [[concept/multi-agent-isolation]]
