---
name: metagpt
type: harness
status: active
language: python
last_ingested_version: latest @ 11cdf466
last_ingested_at: 2026-05-17
last_commit: 11cdf466d042aece04fc6cfd13b28e1a70341b1f
community_known_newer: false
known_post_ingest_features:
  - "MGX (mgx.dev) 商业产品 2025-02 launch — closed-source SaaS"

sources:
  - "[[source/metagpt]]"
  - "/Users/youren/myspace/MetaGPT/"

agent_loop: |
  Role.react() 是核心 loop, 含 3 个 RoleReactMode:
    REACT: 经典 observe → think → act → loop
    BY_ORDER: 按预设 action list 顺序跑 (pipeline)
    PLAN_AND_ACT: 先 plan 再 act (类 hierarchical)
  Role._observe / _think / _act 是核心 3 method.

tool_protocol: |
  Action 抽象不同于 Tool. Action = "role 在做什么" (高级行为), Tool = 底层执行.
  25+ 预定义 Action (design_api / fix_bug / project_management / 等).
  Action 内部可调多个 Tool. tools/ 子目录 + provider integration.

context_mgmt: |
  Role 含 RoleContext + msg_buffer + memory.
  Environment 是 instant messaging bus, 所有 message publish/subscribe 流过 env.
  publish_message → env → 按 subscribed_tags route 给 role 的 msg_buffer.
  没有显式 compaction (in-conversation, 短任务为主).

memory: |
  memory/ 模块 + Role.memory: Memory 实例 (per-role).
  跨 Role 通过 Environment 消息传递, 不共享 memory state.
  message 含 cause_by / sent_from / send_to 路由元数据.

multi_agent: |
  ⭐ **核心定位**. Team + Role + Environment + SOP (Standard Operating Procedures) 4 件套.
  16+ 预定义 role (Architect / Engineer / ProductManager / QAEngineer / etc).
  Environment 是 message bus, role 之间通过 publish/subscribe message 协作.
  7 种 environment (software/android/mgx/minecraft/stanford_town/werewolf/api) — 同 Team 抽象跨多领域.
  Team.hire(roles) → Team.run_project(idea) → publish requirement → role 自动协作.

eval_loop: |
  ext/aflow/ — AFlow paper (ICLR 2025 oral) 含 agentic workflow eval + auto-generation.
  ext/werewolf/ — Werewolf game 是经典 multi-agent benchmark.

self_evolution: |
  ⭐ AFlow 内置 (ext/aflow/) — "Automating Agentic Workflow Generation" 自动生成 + 优化 agent workflow.
  SPO paper (2025-02 arxiv 2502.06855) + AOT paper (2502.12018) 在 examples/ 含 code.

observability: |
  logs.py + runtime logging. 跟外部 (Langfuse/OTel) 集成需用户自配.

hooks: |
  Action lifecycle + Role lifecycle + Environment lifecycle 多层 hook.
  subscription.py 给 message 订阅模式.

safety: |
  ⭐ **`investment` + `NoMoneyException`** — cost-aware multi-agent (业内独家).
  team.invest(investment) → cost_manager.max_budget enforce.
  超预算自动 raise NoMoneyException 终止.

provider: |
  provider/ 子目录, 多 LLM provider 支持. HumanProvider 含 human-in-the-loop.

channels: |
  CLI (software_company.py + startup.py) + Python library + 商业 MGX (mgx.dev SaaS).
  4 国 README (En/Cn/Fr/Ja).

persistence: |
  Team.serialize / deserialize → SERDESER_PATH/team/team.json.
  Role 也实现 SerializationMixin.
  exp_pool/ experiment pool 持久化实验.

cancellation: |
  ? 未深 verify.
  NoMoneyException 是 budget-based 终止.

skillforge_takeaway: |
  MetaGPT 是 "软件公司 as multi-agent system" 范式起点. 3 条强相关启示 + 2 个学术参考.

  🔴 **R-MG-1 — `investment` + `cost_manager.max_budget` + `NoMoneyException` 三件套**(production safety)
     MetaGPT 是业内唯一 cost-aware multi-agent — team 跑过预算自动终止.
     SkillForge SubAgent 当前无预算 enforcement, 多 agent 协作时容易 _"LLM API 账单失控"_.
     工作量: 1-2 周 (加 SubAgentBudget enforce + NoMoneyException 类异常).

  🔴 **R-MG-2 — `Action` 抽象不同于 `Tool`**(架构启示)
     MetaGPT 哲学: Action = "role 在做什么" (高级行为), Tool = 底层执行.
     SkillForge 当前 Skill ≈ Tool, 没有 _"Action 作为 role 职责"_ 的中间抽象.
     远期参考 — 如果走 multi-role 路线, Action 抽象给 _"role 职责 first-class"_ 的 reference.

  🟡 **R-MG-3 — 3 RoleReactMode (REACT/BY_ORDER/PLAN_AND_ACT)**(架构)
     SkillForge agent loop 当前主 ReAct, 缺 BY_ORDER (pipeline) / PLAN_AND_ACT (先 plan 再 act) 两个 mode.
     远期参考, 跟 [[harness/crewai]] Process enum 同设计空间.

  🟢 **R-MG-4 — AFlow 自动生成 agent workflow**(学术参考)
     ICLR 2025 oral paper, 自动生成 + 优化 agent workflow.
     SkillForge SELF-EVOLUTION 远期路线可参考. 工作量: 复杂, 远期.

  🟢 **R-MG-5 — Generative Agents Smallville + Werewolf benchmark**(学术参考)
     ext/stanford_town + ext/werewolf 是经典 multi-agent benchmark 实现.
     SkillForge multi-agent eval 远期可用 werewolf benchmark.

  **❌ 不该跟**:
  - 16+ 预定义 role (太特化, SkillForge 走通用 agent 路线)
  - MGX 商业 SaaS (跟 SkillForge 平台定位重合, 不参考)
  - 4 国 README 国际化 (商业化阶段决定)
---

# Harness — MetaGPT @ commit 11cdf466

> MetaGPT 是 **_"软件公司 as multi-agent system"_ 范式起点**(45k+ stars / MIT / arxiv 2308.00352)。**Team + Role + Environment + SOP** 四件套 + **16+ 预定义 role**(Architect / Engineer / ProductManager / QAEngineer / TeamLeader)+ **25+ 预定义 Action** + **NoMoneyException 预算 enforce**(业内独家 cost-aware multi-agent)。

---

## 1. 项目定位(一句话)

**_"软件公司 as multi-agent system"_ 范式起点**(arxiv 2308.00352)+ AFlow ICLR 2025 oral + 商业 MGX(mgx.dev)。**MetaGPT 哲学**:_"Assign different roles to GPTs to form a collaborative entity for complex tasks"_。

---

## 2. 跟 CrewAI / AG2 的根本差异

```
CrewAI:
  Crew = team of agents
  Process: sequential / hierarchical
  Task DAG via context: list[Task]
  ─────────────────────────────────────────
  哲学: Task-driven (任务驱动)

AG2:
  ConversableAgent + GroupChat
  5+1 speaker selection
  Conversation-driven
  ─────────────────────────────────────────
  哲学: Conversation-driven (对话驱动)

MetaGPT:
  Team + Role + Environment + SOP
  16+ 预定义 role + 25+ Action
  Environment 是 message bus
  ─────────────────────────────────────────
  哲学: Role-play / SOP-driven (角色扮演 + 标准流程)
```

**三条不同 multi-agent 哲学路线**:Task / Conversation / Role-play。

**MetaGPT 独特**:
- **预定义 role 是 first-class**(不是 agent template,是真 Python class 继承 Role)
- **Action 抽象不同于 Tool**(Action = role 行为,Tool = 底层执行)
- **Environment 多类型**(software / android / mgx / minecraft / stanford_town / werewolf)
- **SOP**(Standard Operating Procedures)— 显式标准流程模拟公司组织

---

## 3. 三大独家技术点

### 3.1 `investment` + `NoMoneyException` 预算 enforcement

```python
[code: team.py:92-100]
def invest(self, investment: float):
    """Invest company. raise NoMoneyException when exceed max_budget."""
    self.investment = investment
    self.cost_manager.max_budget = investment
    logger.info(f"Investment: ${investment}.")

def _check_balance(self):
    if self.cost_manager.total_cost >= self.cost_manager.max_budget:
        raise NoMoneyException(self.cost_manager.total_cost, f"Insufficient funds: {self.cost_manager.max_budget}")
```

**业内独家 cost-aware multi-agent**:

| Framework | 预算控制 |
|---|---|
| **MetaGPT** | ⭐ `team.invest($X)` + `NoMoneyException` 超预算自动 raise |
| CrewAI | `max_rpm` 是 rate limit 不是 budget |
| AG2 | `MAX_CONSECUTIVE_AUTO_REPLY` 是 turn limit 不是 budget |
| Claude Code / Codex | UI 提示 token usage 但不强制终止 |

**为什么 MetaGPT 有这个**:因为它模拟 _"软件公司"_,_"投资 / 预算 / 破产"_ 是 _自然_ 抽象。**business metaphor 引出真实 production engineering 需求** — agent team 可能跑出几千刀 LLM 账单。

### 3.2 Action vs Tool 双层抽象

**MetaGPT 哲学**:
```
Role (一个角色)
   ↓ has many
Action (一项职责,如 "设计 API")
   ↓ uses
Tool (底层执行,如 "调 OpenAI API" / "写文件")
```

例子:`Engineer` role 的 actions = `[WriteCode, FixBug, RebuildClassView]`。`WriteCode` action 内部 call `OpenAI API` + `WriteFile tool`。

**对照**:
- LangChain Tool / OpenAI Function = MetaGPT 的 Tool 层
- CrewAI Task = MetaGPT 的 Action 层(但 CrewAI Task 更动态)
- AG2 register_function = MetaGPT 的 Tool 层(无 Action 中间层)

**Action 抽象的好处**:
- Role 的职责显式建模(_"Engineer 做什么"_ = list[Action])
- Action 可以预定义 + 复用(_"WriteCode"_ 是通用 action)
- 25+ 预定义 Action 给软件开发场景足够覆盖

### 3.3 16+ 预定义 Role + 7 Environment + 7 ext 领域

```
Roles (16+):
  Architect / Engineer / Engineer2 / ProductManager / ProjectManager /
  QAEngineer / Researcher / CustomerService / Sales / Searcher /
  Teacher / TutorialAssistant / Assistant / InvoiceOCRAssistant /
  DataAnalyst / TeamLeader

Environments (7):
  software / android / mgx / minecraft / stanford_town / werewolf / api

Extensions (7 学术领域):
  aflow (ICLR 2025 oral) / android_assistant / cr / sela / spo / 
  stanford_town (Generative Agents) / werewolf (multi-agent benchmark)
```

**业内 multi-agent benchmark 集大成**:
- **Werewolf**(`ext/werewolf/`)— 经典 deception + alliance + consensus benchmark
- **Stanford Smallville**(`ext/stanford_town/`)— 25 agent 模拟小镇社交(Park et al 2023 论文)
- **AFlow**(`ext/aflow/`)— ICLR 2025 oral,自动生成 + 优化 agent workflow

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 |
|---|---|
| [[harness/crewai]] | **同赛道对照** — CrewAI Task-driven vs MetaGPT Role-play vs AG2 Conversation-driven 3 哲学 |
| [[eval/ag2]] | 同上 |
| [[concept/multi-agent-isolation]] | 加 _Role-play / SOP-driven_ 维度 |
| [[concept/agent-team-patterns]] | **核心贡献者** — Role-play / SOP 模式标杆 |
| [[harness/openclaw]] | OpenClaw 也用 _"role"_ 概念(memory 路径)但不深 |
| [[deep/openclaw-memory-dreaming-6-dim]] | 互补 — OpenClaw 用 algorithm,MetaGPT 用 role-play |

---

## 5. R-MG 借鉴清单

详见 frontmatter `skillforge_takeaway`。**最高 ROI 是 R-MG-1 cost-aware multi-agent**:

| ID | 项 | 优先级 | 工作量 |
|---|---|---|---|
| **R-MG-1** | `investment` + `NoMoneyException` 预算 enforce | 🔴 P1 | 1-2 周(production multi-agent 必备)|
| **R-MG-2** | `Action` 抽象不同于 `Tool` | 🟡 P2 远期 | 架构启示 |
| **R-MG-3** | 3 RoleReactMode(REACT/BY_ORDER/PLAN_AND_ACT) | 🟡 P2 远期 | 跟 CrewAI Process 同设计空间 |
| **R-MG-4** | AFlow 自动 workflow 生成 | 🟢 P3 学术 | 复杂 |
| **R-MG-5** | Werewolf / Smallville benchmark | 🟢 P3 学术 | multi-agent eval |

---

## 6. 已知 post-ingest 风险

- `Role` 592 行只看了 init / fields(60 行),`react()` / `_observe` / `_think` / `_act` 内部未深
- 25+ Action 只看名字,实际 implementation(`action.py` + `action_node.py` + `action_graph.py`)未深
- `ext/aflow/` AFlow 实现细节未读
- `MGXEnv`(商业版默认 env)实现未读
- SOP(Standard Operating Procedures)代码层具体表达未找到 — 可能是隐式在 Role + Action 组合里

---

## 7. 引用

- [[source/metagpt]] 完整源码锚点
- GitHub: `geekan/MetaGPT` 45k+ stars / MIT
- Arxiv:[2308.00352](https://arxiv.org/abs/2308.00352) MetaGPT 原论文
- AFlow ICLR 2025 oral / SPO arxiv 2502.06855 / AOT 2502.12018
- 商业产品:**MGX**(`mgx.dev`)— _"world's first AI agent development team"_(2025-02 launch)
- 跨页对照:[[harness/crewai]] / [[eval/ag2]] / [[concept/agent-team-patterns]] / [[concept/multi-agent-isolation]] / [[harness/openclaw]]
