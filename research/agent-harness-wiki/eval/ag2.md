---
name: ag2
type: eval
status: active
language: python
last_ingested_version: 0.13.0
last_ingested_at: 2026-05-17
last_commit: ce95893889127cec9b11f81b200de7360ad5a36c
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/ag2]]"
  - "/Users/youren/myspace/ag2/"

agent_loop: |
  ConversableAgent.initiate_chat() 主入口 + generate_reply 异步循环.
  4577 行基类含多个 reply function chain (register_reply 可扩展).
  GroupChatManager 跑 group chat loop: send → select_speaker → broadcast → reply → loop until max_round / TERMINATE.
  human_input_mode: "ALWAYS" / "NEVER" / "TERMINATE" 三态 controls human-in-the-loop.

tool_protocol: |
  register_function 注册 function calling. function_map per-agent.
  tools/ + function_utils.py + interop/ 工具集成.
  mcp/ 子目录 MCP 支持.
  group_tool_executor.py 给 group chat tool execution.

context_mgmt: |
  message history per-agent (messages dict keyed by sender Agent).
  send/receive 消息 reset_history / clear_history (manual control).
  enable_clear_history 让 user 用 "clear history" phrase trigger clear.

memory: |
  AG2 不内置 persistent memory framework. 主要 in-conversation message history.
  contrib/rag/ + qdrant_retrieve_user_proxy_agent.py + retrieve_assistant_agent.py 集成 RAG 作 memory.

multi_agent: |
  ⭐ **核心定位**. 5 个 multi-agent 模式:
  
  1. Two-agent chat (default): AssistantAgent + UserProxyAgent
  2. GroupChat + GroupChatManager (n-agent, 5+1 speaker selection)
  3. Nested chat (chat-within-chat)
  4. Sequential chat (a → b → c chain)
  5. Society of Mind / Swarm / Captain (contrib/ 高级模式)
  
  GroupChat 5+1 speaker selection: auto / manual / random / round_robin / Callable + eligibility filter
  allowed_or_disallowed_speaker_transitions 状态机式 transition 控制.
  
  New group abstraction (agentchat/group/) 是 declarative state machine 风:
  - patterns/ / targets/ / handoffs.py / on_condition.py / llm_condition.py / guardrails.py

eval_loop: |
  contrib/agent_eval/ 子目录含 multi-agent eval framework (自家).
  contrib/agent_optimizer.py 含 prompt optimizer.

self_evolution: |
  agent_optimizer.py 可以根据 trial 历史调整 agent 配置.
  Society of Mind 范式让 agent 内部多个 sub-agent voting 提升 robustness.

observability: |
  ⭐ opentelemetry/ 顶层子目录 OTel 内置 (40k+ stars 项目里独家).
  runtime_logging.py + events/ + logger/.

hooks: |
  register_hook (lifecycle hook)
  register_reply (reply function chain)
  ConversableAgent 4577 行基类含大量 hook point.

safety: |
  MAX_CONSECUTIVE_AUTO_REPLY 防 runaway (default per-class).
  agentchat/group/guardrails.py + safeguards/ 防护层 (新版).
  human_input_mode TERMINATE 让用户中断.

provider: |
  oai/ (OpenAI 兼容层) + llm_clients/ + llm_config/ 多 provider 支持.
  支持 OpenAI / Azure / Anthropic / Gemini / 等通过 OpenAI 兼容 API + 自家 client.

channels: |
  Python library (主) + cli/ + notebook 友好(notebook/ examples)
  realtime/ + realtime_agent/ 支持 realtime voice.
  agentchat/remote/ remote agent.

persistence: |
  cache/ + message history 序列化.
  ag_ui/ 协议(AG UI)给 frontend 用.

cancellation: |
  KeyBoardInterrupt → admin agent takeover (GroupChat).
  human_input_mode TERMINATE.

skillforge_takeaway: |
  AG2 (formerly AutoGen) 是 multi-agent conversation 范式标杆. 跟 SkillForge SubAgent 路线**强对照**.

  🔴 **R-AG-1 — 5+1 speaker selection 模式 (auto/manual/random/round_robin/Callable)**(架构启示)
     SkillForge SubAgent 当前是 parent.spawn(child) 单一模式.
     AG2 5 个 selection method 是 "n-agent 协作时谁说话" 的设计空间完整覆盖.
     SkillForge 远期如果走 multi-agent 协作可借鉴此抽象.
     工作量: 远期,SubAgent 之上加 SpeakerSelection enum.

  🟡 **R-AG-2 — 状态机式 transition 控制 (allowed_or_disallowed_speaker_transitions)**(架构)
     显式定义 "agent A 可 transit 到 B/C 但不能到 D" — 状态机 first-class.
     比 callable selection 更可观察 / 可 debug / 可静态分析.
     工作量: 远期,multi-agent 复杂场景才需要.

  🟡 **R-AG-3 — Auto speaker selection 4 prompt template 处理 LLM 边界 case**(production engineering)
     auto mode 时 LLM 返多个名字 / 返没名字 / 返非法名字, AG2 用 retry prompt 处理.
     SkillForge 任何 _"让 LLM 做选择"_ 的逻辑都该考虑这种 retry pattern.
     工作量: < 半周 (在 LLM-decided 流程里加 retry prompt).

  🟢 **R-AG-4 — Society of Mind / Swarm / Captain 3 个高级模式**(参考)
     contrib/ 含 3 个 advanced multi-agent pattern.
     Society of Mind 是 Minsky 1986 经典神经科学, agent 内部多 sub-agent voting 提升 robustness.
     远期参考, 不直接打到生产.

  🟢 **R-AG-5 — opentelemetry/ 内置**(observability)
     AG2 在 40k+ stars 项目里 OTel 内置独家. SkillForge OBS-2 OTel 协议化时可参考 AG2 instrument 路径.
     工作量: 远期, 跟 [[concept/observability]] R-OBS 系列对照.

  **❌ 不该跟**:
  - 4577 行 ConversableAgent 庞大基类 (过度 inheritance,违反 composition over inheritance)
  - 20+ contrib agent (太多 niche pattern, SkillForge 不需要)
  - human_input_mode ALWAYS default (UserProxyAgent 默认每次问 user,production agent 不合适)
---

# Eval — AG2 (formerly AutoGen) v0.13.0

> AG2 是 **学术起点级 multi-agent framework**(40k+ stars / Apache 2.0,formerly Microsoft AutoGen,2026 分叉到 AG2ai)。**ConversableAgent + GroupChat + 5+1 speaker selection** 是业内 _"multi-agent conversation"_ 范式标杆。

---

## 1. 项目定位(一句话)

**学术起点级 multi-agent conversation framework**。Microsoft Research 2023 原创,2026 社区分叉到 AG2ai。**所有 agent 继承 ConversableAgent**(统一 4577 行基类)+ **GroupChat 5+1 speaker selection 模式** 是其最大特征。

---

## 2. 跟 CrewAI 的本质区别

```
CrewAI:
  Crew = team of agents
  Process enum: sequential / hierarchical (TODO consensual)
  Task DAG: 显式 task.context = [prev_task]
  ─────────────────────────────────────────
  Strengths: production-ready, 简洁, 业务方友好
  Weaknesses: 协作模式只有 2 种

AG2:
  ConversableAgent (4577 行基类)
  GroupChat: dataclass + 5+1 speaker selection
  Conversation-driven (message history per pair)
  ─────────────────────────────────────────
  Strengths: 学术深度, 5 selection + state machine, 20+ advanced pattern
  Weaknesses: 庞大基类, 学习曲线陡, less production-friendly
```

**核心抽象差异**:
- CrewAI = **Task-driven**(task list 定 protocol)
- AG2 = **Conversation-driven**(message exchange 定 protocol)

业内**两条不同 multi-agent 哲学路线**。

---

## 3. 三大独家技术点

### 3.1 5+1 Speaker Selection 模式

```python
speaker_selection_method: Literal["auto", "manual", "random", "round_robin"] | Callable = "auto"
```

| 模式 | 适用 |
|---|---|
| `auto` | LLM 自动选(用 select_speaker_message_template 提示)|
| `manual` | 用户输入选 |
| `random` | 随机选(实验 / chaos engineering)|
| `round_robin` | 轮转(每个 agent 都说一次,公平 / debate)|
| Callable | 自定义 fn (last_speaker, groupchat) → Agent / str / None |

**+1**:`eligibility_policies: list[AgentEligibilityPolicy]` 给 selection 加 filter(AND semantics,所有 policy 都返 True 才 eligible)。

业内对照:
- **CrewAI** 只 sequential / hierarchical 两种
- **MetaGPT** role-based 固定流程
- **AG2** 5+1 模式覆盖全设计空间

### 3.2 Auto Selection 边界 case 4 prompt template

```python
select_speaker_message_template      # 主提示("从 X/Y/Z 里选")
select_speaker_prompt_template       # 末尾追问
select_speaker_auto_multiple_template # LLM 返多个名字时 retry prompt
select_speaker_auto_none_template     # LLM 没返名字时 retry prompt
```

**production engineering wisdom**:LLM 做 _"选哪个"_ 类型决策时,常见失败:
- 返了多个名字("X or maybe Y")— `multiple_template` retry "只返一个"
- 没返名字(扯到别的)— `none_template` retry 给出 list

AG2 把这 2 类失败 explicit prompt 处理,**比 _"LLM 一次返不对就 fallback"_ 更鲁棒**。

### 3.3 状态机式 Transition 控制

```python
allowed_or_disallowed_speaker_transitions: dict[str, Any] | None = None
speaker_transitions_type: Literal["allowed", "disallowed", None] = None
```

显式声明 _"agent A 之后可以是 B 或 C,但不能是 D"_。

**好处**:
- 可静态分析(transition graph 可视化)
- 可 debug(看 transition 路径)
- 跟 LangGraph 状态机思路同源,但更轻量

vs `Callable` 自定义 selection — Callable 灵活但黑盒,transitions dict 可观察。AG2 两者都提供。

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 |
|---|---|
| [[harness/crewai]] | **同赛道对照** — CrewAI Task-driven vs AG2 Conversation-driven 两条 multi-agent 哲学 |
| [[concept/multi-agent-isolation]] | 扩展 5+1 speaker selection 维度到现有 5 维 trade-off |
| [[concept/agent-team-patterns]] | **核心贡献者** — GroupChat 是 _"Round-robin / GroupChat"_ 模式的标杆 |
| [[deep/codex-actor-resume-agent]] | 互补 — Codex actor 是 in-process primitive,AG2 GroupChat 是 conversation layer |
| [[harness/agentscope]] | A2A protocol 对照 |

---

## 5. R-AG 借鉴清单

详见 frontmatter `skillforge_takeaway`。总结:

| ID | 项 | 优先级 | 工作量 |
|---|---|---|---|
| **R-AG-1** | 5+1 speaker selection 模式 | 🔴 P1 架构 | 远期 |
| **R-AG-2** | 状态机式 transition 控制 | 🟡 P2 | 远期(复杂 multi-agent 需求触发)|
| **R-AG-3** | Auto selection 4 prompt template 处理边界 case | 🟡 P2 | < 半周 |
| **R-AG-4** | Society of Mind / Swarm / Captain 3 advanced pattern | 🟢 P3 远期 | — |
| **R-AG-5** | opentelemetry/ 内置 | 🟢 P3 跟 R-OBS 配套 | — |

---

## 6. 已知 post-ingest 风险

- ConversableAgent 4577 行基类只看了 init/fields 一小部分,reply chain / hook 机制未深
- GroupChat speaker selection auto 模式具体 prompt sequence 未跑通完整 example
- `agentchat/group/` 新版 abstraction(condition-based)未深读
- `contrib/` 20+ agent 只看名字,实际 implementation 未读
- AG2 vs 老 AutoGen 分叉细节 + 2026 路线图未查

---

## 7. 引用

- [[source/ag2]] 完整源码锚点
- GitHub: `ag2ai/ag2` 40k+ stars / Apache 2.0
- PyPI: `pip install ag2`(老名 `pyautogen` 仍可用)
- 论文:_"AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation"_(arxiv 2308.08155)
- 跨页对照:[[harness/crewai]] / [[concept/agent-team-patterns]] / [[deep/codex-actor-resume-agent]] / [[harness/agentscope]]
