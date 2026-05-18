---
name: ag2
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: ag2 (formerly autogen)
version: 0.13.0
commit: ce95893889127cec9b11f81b200de7360ad5a36c
license: Apache 2.0
stars: 40k+
last_ingested_at: 2026-05-17
contributed_to:
  - "[[eval/ag2]]"
  - "[[concept/agent-team-patterns]]"
---

# Source — AG2 (AutoGen) v0.13.0 @ commit ce95893

> ag2ai/ag2(formerly Microsoft AutoGen,2026 分叉到 AG2ai)— **学术起点级 multi-agent framework**(40k+ stars,Apache 2.0)。**GroupChat + ConversableAgent + UserProxyAgent** 三件套是业内 _"multi-agent conversation"_ 范式标杆。clone @ `/Users/youren/myspace/ag2/` 37 MB。

---

## 1. 仓库结构 — 36+ 子模块

```
ag2/autogen/
├── a2a/                # Google A2A protocol
├── ag_ui/              # AG UI 协议
├── agentchat/          # ⭐ 核心 multi-agent (本次 ingest 重点)
├── agents/             # agent 实现
├── beta/               # beta features
├── browser_utils.py    # browser tooling
├── cache/
├── code_utils.py       # 代码执行 helper
├── coding/             # code execution sandbox
├── doc_utils.py
├── environments/
├── events/             # event system
├── extensions/
├── fast_depends/
├── formatting_utils.py
├── function_utils.py
├── graph_utils.py
├── import_utils.py
├── interop/
├── io/
├── json_utils.py
├── llm_clients/        # LLM provider 抽象
├── llm_config/
├── logger/
├── math_utils.py
├── mcp/                # MCP 支持
├── messages/           # message 抽象
├── oai/                # OpenAI 兼容层
├── opentelemetry/      # ⭐ OTel 内置
├── retrieve_utils.py
├── runtime_logging.py
├── testing/
├── token_count_utils.py
├── tools/
├── types.py
├── version.py
└── py.typed
```

**OTel first-class**(`opentelemetry/` 顶层子目录),vs CrewAI / MetaGPT 都依赖外部 obs 工具。

---

## 2. `autogen/agentchat/` — 核心 multi-agent

```
agentchat/
├── agent.py                     # Agent 基类
├── assistant_agent.py           # ⭐ AssistantAgent (默认配 assistant prompt)
├── chat.py                      # chat 基础设施
├── conversable_agent.py         # ⭐ ConversableAgent (基类, 4577 行)
├── user_proxy_agent.py          # ⭐ UserProxyAgent (human_input_mode=ALWAYS)
├── groupchat.py                 # ⭐ GroupChat (1184 行+)
├── eligibility_policy.py        # speaker 资格 filter
├── group/                       # ⭐ 新版 group abstraction
├── realtime/ + realtime_agent/  # realtime voice
├── remote/                      # remote agent
├── utils.py
└── contrib/                     # ⭐ advanced patterns (20+ 模式)
```

---

## 3. 核心抽象 — 3 Agent 类型 + GroupChat

### 3.1 ConversableAgent [code: conversable_agent.py:136-]

```python
class ConversableAgent(LLMAgent):
    """(In preview) A class for generic conversable agents which can be 
    configured as assistant or user proxy."""
    
    # 4577 行的庞大基类, 是所有 agent 的祖先
    # 核心能力:
    #   - send / receive / initiate_chat / generate_reply
    #   - register_reply (extensible reply function chain)
    #   - register_hook (lifecycle hook)
    #   - register_function (function calling)
    #   - human_input_mode: "ALWAYS" / "NEVER" / "TERMINATE"
    #   - MAX_CONSECUTIVE_AUTO_REPLY (anti-runaway)
```

**AG2 哲学**:_"Everything is a ConversableAgent"_。AssistantAgent / UserProxyAgent / GroupChatManager / 各种 contrib agent **全都继承 ConversableAgent**,统一 API。

### 3.2 GroupChat dataclass [code: groupchat.py:48-180+]

```python
@dataclass
class GroupChat:
    """(In preview) A group chat class"""
    
    agents: list[Agent]
    messages: list[dict] = []
    max_round: int = 10
    admin_name: str = "Admin"
    func_call_filter: bool = True
    
    # ⭐ 5+1 speaker selection 模式
    speaker_selection_method: Literal["auto", "manual", "random", "round_robin"] | Callable = "auto"
    max_retries_for_selecting_speaker: int = 2
    
    # ⭐ 状态机式 transition 控制
    allow_repeat_speaker: bool | list[Agent] | None = None
    allowed_or_disallowed_speaker_transitions: dict[str, Any] | None = None
    speaker_transitions_type: Literal["allowed", "disallowed", None] = None
    
    # User control
    enable_clear_history: bool = False
    send_introductions: bool = False
    
    # ⭐ Speaker auto-selection 4 prompt template (处理 LLM 边界 case)
    select_speaker_message_template: str = "..."           # 系统级提示
    select_speaker_prompt_template: str = "..."             # 让 LLM 选
    select_speaker_auto_multiple_template: str = "..."     # LLM 返多个名字时 retry
    select_speaker_auto_none_template: str = "..."          # LLM 没返名字时 retry
    
    # Eligibility filter
    eligibility_policies: list[AgentEligibilityPolicy] = []
```

**关键设计**:

1. **5+1 speaker selection 模式**(`auto/manual/random/round_robin/Callable`)
2. **状态机式 transition 控制**(`allowed_or_disallowed_speaker_transitions` + `speaker_transitions_type` 显式定义 agent A 可以 transit 到 B/C 但不能到 D)
3. **Auto selection 4 个 prompt template** 处理 LLM 边界 case:
   - `select_speaker_message_template` — 主提示("从 X / Y / Z 里选")
   - `select_speaker_prompt_template` — 末尾追问
   - `select_speaker_auto_multiple_template` — LLM 返了多个名字时 retry prompt
   - `select_speaker_auto_none_template` — LLM 没返名字时 retry prompt
4. **`func_call_filter`** — function call 触发时,按 function name 自动选有这个 function 的 agent

**Speaker selection 6 选项**:
```python
# auto: 默认,LLM 选
# manual: user input 选
# random: 随机
# round_robin: 按 agents 顺序轮转
# Callable: 自定义函数(传 last_speaker + groupchat,返 Agent / str / None)
# (eligibility filter 作为额外约束层)
```

### 3.3 GroupChatManager [code: groupchat.py:1184-]

```python
class GroupChatManager(ConversableAgent):
    """管理 GroupChat 的 manager agent. 继承 ConversableAgent."""
    # 跑 group chat loop:
    #   1. 收到 message
    #   2. select_speaker (按 speaker_selection_method)
    #   3. broadcast 给 speaker
    #   4. speaker reply
    #   5. clear_agents_history (如果触发 "clear history" phrase)
    #   6. loop until max_round / TERMINATE
```

**Manager 也是 ConversableAgent**,这是 AG2 的统一性 — manager 可以再被嵌入到更大的 GroupChat 里(meta-groupchat)。

### 3.4 AssistantAgent + UserProxyAgent [code: assistant_agent.py:17, user_proxy_agent.py:17]

```python
class AssistantAgent(ConversableAgent):
    """Default config:
    - system_message: 通用 assistant prompt
    - human_input_mode: NEVER (LLM 全自动)
    - code_execution_config: False (默认不执行 code)
    """

class UserProxyAgent(ConversableAgent):
    """Default config:
    - human_input_mode: ALWAYS (每次 reply 都要 user 输入)
    - code_execution_config: 通常配置为可执行 code
    """
```

**AG2 默认 pattern**:`AssistantAgent + UserProxyAgent` 二元 conversation。User 输入 → UserProxyAgent → AssistantAgent → tool call(via UserProxy code execution)→ AssistantAgent → reply。

---

## 4. `agentchat/group/` — 新版 multi-agent abstraction

```
group/
├── patterns/                    # ⭐ 各种 group pattern (round_robin / hand_off / etc)
├── targets/                     # ⭐ TransitionTarget (where to go next)
├── events/
├── handoffs.py                  # ⭐ agent 间 handoff
├── guardrails.py / safeguards/  # ⭐ 防护层
├── on_condition.py              # ⭐ condition-based transition
├── on_context_condition.py
├── available_condition.py / context_condition.py / context_expression.py / context_str.py
├── context_variables.py
├── group_tool_executor.py
├── group_utils.py
├── llm_condition.py
├── multi_agent_chat.py
├── reply_result.py
└── speaker_selection_result.py
```

**新版 group abstraction 是 declarative state machine 风**(condition-based transitions),vs 旧版 GroupChat 是 dataclass + Callable 风。

---

## 5. `agentchat/contrib/` — 20+ advanced multi-agent pattern

```
contrib/
├── agent_eval/                          # agent eval framework
├── agent_optimizer.py
├── capabilities/                         # agent capability mixin
├── captainagent.py                       # ⭐ CaptainAgent (复杂编排)
├── gpt_assistant_agent.py                # OpenAI Assistants API
├── graph_rag/
├── llamaindex_conversable_agent.py       # LlamaIndex 集成
├── llava_agent.py                        # 多模态 (LLaVA)
├── math_user_proxy_agent.py              # 数学专用
├── multimodal_conversable_agent.py       # 通用多模态
├── nlip_agent.py
├── qdrant_retrieve_user_proxy_agent.py
├── rag/
├── retrieve_assistant_agent.py / retrieve_user_proxy_agent.py
├── society_of_mind_agent.py              # ⭐ Society of Mind 范式
├── swarm_agent.py                        # ⭐ Swarm pattern
├── text_analyzer_agent.py
├── vectordb/
└── web_surfer.py                         # browser agent
```

**3 个学术级独家 pattern**:
- **Society of Mind**(`society_of_mind_agent.py`)— Minsky 1986 神经科学经典,一个 agent 内部含多个 sub-agent voting
- **Swarm**(`swarm_agent.py`)— Swarm 协调模式
- **CaptainAgent**(`captainagent.py`)— 复杂编排 captain pattern

---

## 6. 关键 verified 锚点

| 路径 | 主题 |
|---|---|
| `agentchat/conversable_agent.py:136-` | ConversableAgent 4577 行基类 |
| `agentchat/groupchat.py:48-180+` | GroupChat dataclass + 5 speaker selection + transition state machine + 4 prompt template |
| `agentchat/groupchat.py:1184-` | GroupChatManager 类 |
| `agentchat/assistant_agent.py:17` / `user_proxy_agent.py:17` | AssistantAgent + UserProxyAgent |
| `agentchat/group/patterns/ + targets/ + handoffs.py` | 新版 declarative group abstraction |
| `agentchat/group/on_condition.py + llm_condition.py + context_condition.py` | condition-based transitions |
| `agentchat/group/guardrails.py + safeguards/` | 防护层 |
| `agentchat/contrib/society_of_mind_agent.py + swarm_agent.py + captainagent.py` | 3 个 advanced multi-agent pattern |
| `agentchat/contrib/{multimodal,llava,math,retrieve}_*_agent.py` | 多模态 + 数学 + RAG agent |
| `agentchat/eligibility_policy.py` | speaker 资格 filter |
| `autogen/a2a/` | Google A2A protocol |
| `autogen/opentelemetry/` | OTel 内置 |
| `autogen/mcp/` | MCP 支持 |
| `autogen/realtime/ + realtime_agent/` | Realtime voice |

---

## 7. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/ag2/](file:///Users/youren/myspace/ag2/) 37 MB
- GitHub: `ag2ai/ag2` 40k+ stars / Apache 2.0(formerly `microsoft/autogen`)
- PyPI: `pip install ag2`(老名 `pyautogen` 仍可用)
- 衍生 wiki:[[eval/ag2]] / [[concept/agent-team-patterns]]
