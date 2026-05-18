---
name: agent-team-patterns
type: concept
status: B-mode source-grounded (depth-first synthesis)
updated: 2026-05-17
sources:
  - "[[harness/crewai]]"
  - "[[eval/ag2]]"
  - "[[harness/metagpt]]"
  - "[[harness/claude-code]]"
  - "[[harness/codex]]"
  - "[[harness/agentscope]]"
  - "[[harness/dify]]"
  - "[[deep/codex-actor-resume-agent]]"
---

# Agent Team Patterns — 3 哲学 × 4 组织模式 × 5 production engineering wisdom

> **要点**:Multi-agent 不是 _"有多个 agent 就行"_。这页 synthesis 跨 3 个新 ingest(CrewAI / AG2 / MetaGPT)+ 已 ingest 的 Claude Code SubAgent / Codex actor / AgentScope MsgHub / Dify workflow_as_tool,提炼出 **3 条 multi-agent 哲学路线**(Task-driven / Conversation-driven / Role-play-driven)+ **4 大组织模式**(Sequential / Hierarchical / GroupChat / Role-play)+ **5 个 production engineering wisdom**(预算 enforce / speaker selection / state machine transition / SOP / cost-aware)。
>
> 一个反直觉发现:**multi-agent 框架的根本差异不在 "有没有 agent class",而在 "用什么 metaphor 组织 agent"**(Task / Conversation / Role-play 是 3 个不同 mental model)。

---

## 1. 写这页之前需要校准的 3 件事

### 1.1 _"Multi-agent"_ 是个 overloaded 词

业内说 _"multi-agent"_ 可能指 3 种完全不同的事:

| 层级 | 含义 | wiki 已 ingest 项目 |
|---|---|---|
| **Primitive 层** | _"我能 spawn 一个 sub-agent"_(spawn / mailbox / fork / actor)| Claude Code SubAgent / Codex actor (`[[deep/codex-actor-resume-agent]]`)/ AgentScope MsgHub |
| **Composition 层** | _"我能把 workflow / skill 包成 callable agent / tool"_ | Dify workflow_as_tool (`[[deep/dify-workflow-as-tool]]`)/ AgentScope StateModule (`[[deep/...]]`) |
| **Team / Organization 层** | _"我有一队 agent 协作完成 task"_ | **CrewAI / AG2 / MetaGPT(本次新 ingest)** |

**本页只讨论 Team 层** — Primitive 和 Composition 在其他 concept page 覆盖。

### 1.2 3 个新 ingest 项目代表 3 条 multi-agent 哲学路线

```
                  ┌─── CrewAI ─── Task-driven       ─────┐
                  │              (任务驱动)              │
                  │                                       │
Multi-Agent ─────┼─── AG2 ────── Conversation-driven ────┼──── 三条哲学
                  │              (对话驱动)              │      正交不冲突
                  │                                       │
                  └─── MetaGPT ── Role-play-driven   ────┘
                                  (角色扮演驱动)
```

**3 条哲学不冲突,可叠加**:你可以在 MetaGPT 的 Role-play 框架里用 CrewAI 的 Task DAG,或在 CrewAI 的 Crew 内用 AG2 的 GroupChat。

### 1.3 SkillForge 当前 SubAgent 是 Primitive 层,不是 Team 层

[verified-from-source] from [[concept/agent-harness]] R-HARNESS verify:
- SkillForge `SubAgentRunEntity` + `SubAgentRegistry` + `SubAgentStartupRecovery` = **Primitive 层**(spawn / 异步 result / crash recovery)
- **没有 Team / Crew / Manager 抽象**(没有 _"一组 agent + 协作模式"_ 的 first-class type)

所以 R-CW-1 / R-AG-1 / R-MG-1 都是 _"如果 SkillForge 要做 Team 层"_ 的远期参考,不是当下缺口。

---

## 2. 3 条 multi-agent 哲学路线深度对照

### 2.1 CrewAI — Task-driven

**核心 mental model**:_"team 是一组 agent 跑一系列 task"_。

```python
crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[
        Task(description="research X", agent=researcher),
        Task(description="write article", agent=writer, context=[task1]),
        Task(description="review article", agent=reviewer, context=[task2]),
    ],
    process=Process.sequential,  # 或 hierarchical
)
crew.kickoff()
```

**强项**:
- ✅ **业务方友好**(_"我有任务 1 / 2 / 3,让 X / Y / Z agent 做"_ 直观)
- ✅ **Task DAG 显式编排**(`context: list[Task]` 类型安全)
- ✅ **Production-ready**(30k+ stars,SaaS Plus)

**弱项**:
- ❌ 不适合_"开放讨论 / 不确定流程"_(必须预定义 task)
- ❌ 只 2 种 process(sequential / hierarchical),consensual TODO 未实现

### 2.2 AG2 (AutoGen) — Conversation-driven

**核心 mental model**:_"team 是一组 agent 你来我往说话"_。

```python
manager = GroupChatManager(groupchat=GroupChat(
    agents=[user_proxy, assistant1, assistant2, critic],
    speaker_selection_method="auto",  # 或 manual/random/round_robin/Callable
    allowed_or_disallowed_speaker_transitions={
        user_proxy: [assistant1],
        assistant1: [assistant2, critic],
        critic: [user_proxy, assistant1],
    },
    speaker_transitions_type="allowed",
    max_round=10,
))
user_proxy.initiate_chat(manager, message="Help me with X")
```

**强项**:
- ✅ **5+1 speaker selection 模式**(auto / manual / random / round_robin / Callable + eligibility filter)
- ✅ **状态机式 transition 控制**(显式 allowed transitions)
- ✅ **开放讨论 / consensus 友好**(不预定义 task,LLM 自己 reason)
- ✅ **20+ advanced pattern 内置**(Society of Mind / Swarm / CaptainAgent)

**弱项**:
- ❌ **庞大基类**(`ConversableAgent` 4577 行)
- ❌ **学习曲线陡**(_"什么时候用哪个 selection method"_ 不直观)
- ❌ Production engineering 没 CrewAI 友好

### 2.3 MetaGPT — Role-play-driven

**核心 mental model**:_"team 是一家公司,每个 agent 扮一个 role,按 SOP 工作"_。

```python
company = Team()
company.hire([
    TeamLeader(),
    ProductManager(),
    Architect(),
    Engineer2(),
    QAEngineer(),
])
company.invest(investment=10.0)  # ⭐ 预算 enforce
company.run_project("Build a 2048 game")
# Roles 自动按 SOP 协作:
#   PM 写 PRD → Architect 设计 → Engineer 写 code → QA 测试
```

**强项**:
- ✅ **强 metaphor**(_"软件公司"_ 直观)
- ✅ **16+ 预定义 role**(Architect / Engineer / PM / QA / etc 即取即用)
- ✅ **`investment` + `NoMoneyException` 预算 enforce**(业内独家 cost-aware)
- ✅ **学术深度**(AFlow ICLR 2025 oral + Stanford Smallville + Werewolf benchmark)
- ✅ **Action vs Tool 双层抽象**

**弱项**:
- ❌ **预定义 role 特化**(SDLC 之外的领域要自家写 role)
- ❌ **MGX SaaS 商业化压力**(OSS 维护可能受影响)

---

## 3. 4 大组织模式 × 项目分布

```
                      Sequential
                          │
                          │  CrewAI sequential / MetaGPT BY_ORDER /
                          │  AG2 sequential chat / LangGraph linear
                          │
       Round-robin ◄──── Multi-Agent ──── ► Hierarchical
       GroupChat            模式分布           Manager-Worker
            │                                       │
            │ AG2 GroupChat                          │ CrewAI hierarchical /
            │ (auto/random/round_robin)              │ MetaGPT Role-play /
            │ MetaGPT debate                         │ AG2 manager_llm
            │                                       │
            └──────────────────┬────────────────────┘
                               │
                          Role-play /
                          Adversarial
                          │
                          │ MetaGPT 16+ role / AG2 Society of Mind /
                          │ AG2 Swarm / ChatDev / AG2 critic-vs-proposer
```

### 模式 1 — Sequential / Pipeline

```
A → B → C → D
```

**触发场景**:Task 间有明确依赖关系(_"先 research 才能 write,先 write 才能 review"_)。

**业内实现**:
| 项目 | 实现 |
|---|---|
| CrewAI | `process=Process.sequential` |
| MetaGPT | `RoleReactMode.BY_ORDER` |
| AG2 | Sequential chat helper |
| LangGraph | Linear StateGraph |
| Dify | Workflow sequential nodes |

### 模式 2 — Hierarchical / Manager-Worker

```
        Manager
       /   |   \
      W1   W2   W3
```

**触发场景**:**任务需要分解 + 并行**;Manager 决定 _"谁做什么"_。

**业内实现**:
| 项目 | 实现 | 备注 |
|---|---|---|
| CrewAI | `process=Process.hierarchical` + `manager_llm` / `manager_agent` | 简洁 |
| MetaGPT | `TeamLeader` role 协调其他 role | Role-play 风 |
| AG2 | `GroupChatManager + manager_llm` | dataclass |
| LangGraph | Supervisor pattern | state machine |

### 模式 3 — Round-robin / GroupChat

```
A ↔ B ↔ C ↔ D (按规则发言)
```

**触发场景**:**开放讨论 / consensus formation**;每个 agent 都有发言权。

**业内实现**:
| 项目 | 实现 | 独家 |
|---|---|---|
| **AG2** | ⭐ `GroupChat + speaker_selection_method` | **5+1 模式 + state machine transitions** |
| MetaGPT | publish/subscribe via Environment | 不显式 group chat |
| CrewAI | 无显式 round-robin |
| LangGraph | 用 state machine 模拟 |

### 模式 4 — Role-play / Adversarial

```
Proposer ←→ Critic (反复 critique)
   │           │
   └─→ Decision ─┘
```

**触发场景**:**需要多视角验证 / debate / self-correction**。

**业内实现**:
| 项目 | 实现 | 备注 |
|---|---|---|
| **MetaGPT** | ⭐ 16+ 预定义 role 含 _"对抗"_(QA vs Engineer)| Role-play 是核心 |
| AG2 contrib | ⭐ `society_of_mind_agent.py` + `swarm_agent.py` + `captainagent.py` | 3 个学术级 pattern |
| ChatDev | 学术级 communicative agents for SDE | 论文级 |
| CAMEL-AI | Role-playing 学术起点 | _"千人千 agent"_ 范式 |

---

## 4. 5 个 production engineering wisdom(从 3 项目提炼)

### 4.1 预算 enforcement(MetaGPT 独家)

```python
[code: metagpt/team.py:92-100]
def invest(self, investment: float):
    self.investment = investment
    self.cost_manager.max_budget = investment

def _check_balance(self):
    if self.cost_manager.total_cost >= self.cost_manager.max_budget:
        raise NoMoneyException(...)
```

**为什么是 wisdom**:Multi-agent 跑起来 token 用量是单 agent 10-100x。_"agent team 跑出几千刀账单"_ 是 production 真问题。MetaGPT _"软件公司"_ metaphor 自然引出 _"投资 / 预算 / 破产"_ 抽象 — **强制 explicit budget**。

**业内对照**:CrewAI `max_rpm` 是 rate limit,AG2 `MAX_CONSECUTIVE_AUTO_REPLY` 是 turn limit,都不是 cost budget。

### 4.2 Speaker selection 多模式(AG2 独家)

```python
[code: ag2/autogen/agentchat/groupchat.py:147]
speaker_selection_method: Literal["auto", "manual", "random", "round_robin"] | Callable = "auto"
```

**为什么是 wisdom**:不同场景需要不同 selection — auto 适合大部分,round_robin 适合 fair debate,Callable 适合复杂业务规则,random 适合 chaos engineering / 实验。

**额外**:`select_speaker_auto_multiple_template` + `select_speaker_auto_none_template` 4 个 prompt template 处理 LLM 返多个名字 / 返没名字 / 返非法名字的边界 case — **production-tested**。

### 4.3 State machine transition 控制(AG2 独家)

```python
allowed_or_disallowed_speaker_transitions: dict = None
speaker_transitions_type: Literal["allowed", "disallowed", None] = None
```

**为什么是 wisdom**:_"agent A 之后可以是 B 或 C,但不能是 D"_ 显式定义 → 可静态分析 + 可 debug + 跟 LangGraph state machine 同思路。

### 4.4 Task DAG via 实例引用(CrewAI 独家)

```python
[code: crewai/task.py:160-163]
context: list[Task] | None = Field(
    description="Other tasks that will have their output used as context for this task.",
    default=NOT_SPECIFIED,
)
```

**为什么是 wisdom**:_类型安全_ 的 task 依赖:`context=[research_task]` 是 Python 引用,IDE 跳转 + linter check。**比 string template `{{prev_output}}` 强一个数量级**。

### 4.5 SOP(Standard Operating Procedures)预定义 role(MetaGPT 独家)

```python
[code: metagpt/software_company.py:18+]
company.hire([
    TeamLeader(), ProductManager(), Architect(), Engineer2(), DataAnalyst(),
])
```

**为什么是 wisdom**:**预定义 role + 预定义 Action 让 _"agent team 怎么协作"_ 变成 zero-config**。用户不用想 _"我该雇什么 role / 让 PM 做什么 action"_ — 跟着 SDLC 经典流程走。**业务方友好的极致**。

---

## 5. 项目 × 模式对照矩阵(完整)

| | CrewAI | AG2 | MetaGPT | Claude Code | Codex | AgentScope | Dify | LangGraph |
|---|---|---|---|---|---|---|---|---|
| **Sequential** | ✅(default) | ✅ | ✅(BY_ORDER) | ❌(单 loop) | ✅(actor) | ✅(pipeline) | ✅(workflow) | ✅(linear) |
| **Hierarchical** | ✅(hierarchical) | ✅(manager_llm) | ✅(TeamLeader) | ✅(SubAgent) | ✅(resume_agent) | ✅(MsgHub) | ✅(workflow_as_tool) | ✅(supervisor) |
| **GroupChat** | ❌ | ⭐ **核心** | partial(Environment bus) | ❌(显式 Coordinator) | ✅(actor mailbox) | ✅(MsgHub) | ❌ | partial |
| **Role-play** | ❌ | partial(contrib) | ⭐ **核心**(16+ role) | ❌ | ❌ | ❌ | ❌ | ❌ |
| **State machine transitions** | ❌ | ⭐ allowed_speaker_transitions | ❌ | ❌ | ❌ | ❌ | ✅(node graph) | ⭐ first-class |
| **Cost budget enforce** | ❌(max_rpm) | ❌(turn limit) | ⭐ NoMoneyException | ❌ | ❌ | ❌ | partial(token tracking) | ❌ |
| **Speaker auto-selection** | ❌ | ⭐ 5+1 模式 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Task DAG 类型安全** | ⭐ context: list[Task] | ❌ | partial | ❌ | ❌ | ❌ | ✅(node graph) | ✅ |
| **预定义 role marketplace** | ❌ | partial(contrib) | ⭐ 16+ role | ❌(SubAgent 通用) | ❌ | ❌ | partial(tools) | ❌ |

**🟢 = 该项目最强维度**。

**核心观察**:**每个项目都有 2-3 个独家维度**,SkillForge 借鉴时**必须 cherry-pick**,不要全套抄。

---

## 6. SkillForge R-TEAM 借鉴清单(综合 3 ingest)

汇总:**[[harness/crewai]] R-CW-1~4** + **[[eval/ag2]] R-AG-1~5** + **[[harness/metagpt]] R-MG-1~5** = **R-TEAM 综合清单**。

### 🔴 P1 必做(2 条,直接打 SkillForge multi-agent 短板)

| ID | 项 | 借鉴源 | 工作量 |
|---|---|---|---|
| **R-TEAM-1** | **`investment` + `NoMoneyException` 预算 enforce** | MetaGPT (R-MG-1) | 1-2 周 |
| **R-TEAM-2** | **Task DAG via `context: list[Task]`(类型安全)** | CrewAI (R-CW-2) | < 1 周 |

### 🟡 P2 应做(3 条,远期 multi-agent 路径触发)

| ID | 项 | 借鉴源 | 工作量 |
|---|---|---|---|
| **R-TEAM-3** | Process / RoleReactMode enum(团队协作模式 first-class type) | CrewAI (R-CW-1) + MetaGPT (R-MG-3) | 远期 |
| **R-TEAM-4** | Speaker selection 5+1 模式(auto/manual/random/round_robin/Callable) | AG2 (R-AG-1) | 远期 |
| **R-TEAM-5** | Auto selection 4 prompt template(处理 LLM 边界 case) | AG2 (R-AG-3) | < 半周 |

### 🟢 P3 远期参考(3 条)

| ID | 项 | 借鉴源 |
|---|---|---|
| **R-TEAM-6** | Action vs Tool 双层抽象 | MetaGPT (R-MG-2) |
| **R-TEAM-7** | 状态机式 transition 控制 | AG2 (R-AG-2) |
| **R-TEAM-8** | AFlow 自动 workflow 生成 / Werewolf benchmark | MetaGPT (R-MG-4, R-MG-5) |

### ⚠️ 不该跟(从 3 项目共同提炼)

- **庞大基类 inheritance**(AG2 ConversableAgent 4577 行)— 违反 composition over inheritance
- **20+ 特化 role / 20+ contrib agent**(MetaGPT / AG2)— 过度 niche,SkillForge 走通用 agent 路线
- **商业 SaaS 上报**(CrewAI Plus / MGX)— SkillForge 自家平台,不上报第三方
- **预定义 SDLC role marketplace**(MetaGPT)— 太特化,通用 SkillForge 不需要

---

## 7. 跟现有 wiki 关系

| 当前页 | 关系 |
|---|---|
| [[concept/multi-agent-isolation]] | 扩展第 6 维 _"team organization 模式"_(原 5 维 trade-off 是 spawn / mailbox / state 等 primitive 层)|
| [[concept/agent-memory]] | CrewAI 3 维 memory scope 是第 7 路线("scope-based") |
| [[deep/codex-actor-resume-agent]] | 互补 — Codex 是 actor primitive,本页是 team 抽象 |
| [[deep/dify-workflow-as-tool]] | 互补 — Dify 是 workflow nest,本页是 team-as-entity |
| [[concept/agent-harness]] R-HARNESS | R-TEAM 是 R-HARNESS 的 multi-agent 子集补充 |

---

## 8. 学习视角 — 写这页让我学到的 3 件事

1. **`investment + NoMoneyException` 是 multi-agent 必修课**:`MetaGPT.team.invest($X)` + cost_manager 自动 enforce 是业内唯一 cost-aware multi-agent。**任何 production agent team 都该有显式预算 + raise 机制**,因为 multi-agent 跑起来 token 用量是单 agent 10-100x,容易出 _"几千刀账单事故"_。
2. **不同 metaphor 引出不同设计**:CrewAI(Task)/ AG2(Conversation)/ MetaGPT(Role-play)三个 metaphor 不只是 naming 差异,**它们引出完全不同的 abstractions + APIs + ergonomics**。**任何 framework 设计第一步是选 metaphor**,后面 API 都跟着 metaphor 自然涌现。
3. **AG2 `auto speaker selection 4 prompt template` 是 _"production-tested LLM-decided 流程"_ 的标准答案**:_"LLM 返多个名字"_ + _"LLM 没返名字"_ + _"LLM 返非法名字"_ 三类失败 explicit retry prompt — **这种 _"反复 retry with refined prompt"_ pattern 应该是任何 _"让 LLM 做决策"_ 流程的默认选项**。

---

## 引用链路

- 3 个新 ingest:[[harness/crewai]] / [[eval/ag2]] / [[harness/metagpt]] + 对应 source page
- 已 ingest 涉及 multi-agent:[[harness/claude-code]] / [[harness/codex]] / [[harness/agentscope]] / [[harness/dify]] / [[harness/openclaw]]
- 相关 deep page:[[deep/codex-actor-resume-agent]] / [[deep/dify-workflow-as-tool]]
- 相关 concept:[[concept/multi-agent-isolation]] / [[concept/agent-memory]] / [[concept/agent-harness]] / [[concept/agent-state-separation]]
- 学术 backbone:MetaGPT arxiv 2308.00352 / AutoGen arxiv 2308.08155 / AFlow ICLR 2025 oral / Stanford Generative Agents (Park et al 2023)
