---
name: agent-state-separation
type: concept
status: B-mode source-grounded (depth-first)
updated: 2026-05-17
sources:
  - "[[source/agentscope]]"
  - "[[harness/agentscope]]"
  - "[[harness/codex]]"
  - "[[harness/claude-code]]"
---

# Agent State Separation — 一个 PyTorch 抽象,如何变成 agent runtime 的关键设计选择

> **要点**:AgentScope 的 `StateModule`(`module/_state_module.py:20-152`)是 **PyTorch `nn.Module` 移植到 agent state**,只有 132 行代码,却**撬动 5 个 agent runtime 关键能力**:跨环境迁移 active agent / A/B 测试不同 behavior / 时间旅行 debug / production trace → training dataset / interrupt-then-resume 不破 history。
> 
> 我第一次看到 AgentScope README 写 _"separates an agent's creation from its state"_ 时以为是营销话术。读完 `_state_module.py` 才意识到:**这真的是 nn.Module + custom JSON converter,而且整个框架 26 个模块都自觉地 inherit StateModule**。这种"框架级 self-discipline" 是为什么这个 132 行 class 能撬动 5 个 capability 的原因。

---

## 1. 为什么这个抽象重要 — 5 种你以为是独立 feature 的能力,其实都靠它

普通用 agent 框架的人,大概会把下面 5 件事当成**独立 feature**:

| capability | 你以为的实现 |
|---|---|
| A. 跨进程 / 跨主机迁移 active agent | "复杂的 distributed system" |
| B. A/B 测试不同 agent behavior(同 user / 同 session,跑两套)| "复制一份 agent" |
| C. 时间旅行 debug(回到 session 第 7 轮看当时 agent 内部状态)| "存所有快照" |
| D. Production trace → training dataset(为 RL fine-tune 收集数据)| "log everything" |
| E. Interrupt-then-resume(中断 → 关机 → 重启 → 继续)| "事务 + 检查点" |

**真相**:这 5 件事其实**只需要 1 个能力** — _"把 agent 的所有 stateful 内部对象一次性、忠实、可还原地 dump 成 JSON"_。剩下的就是 "存哪里 / 何时 dump / 何时 load" 的工程细节。

AgentScope 把 _"把 agent state 一次性 dump"_ 抽成 `StateModule`,这 5 个 capability 自动 fall out:
- A → load_state_dict 到另一台机器
- B → 同一个 state_dict 喂给两个 agent class
- C → 每轮 dump 一份 state_dict,debug 时按 turn idx 还原
- D → state_dict + tool_use trace 就是 RL training input
- E → 用 4 个 session backend 之一(JSON/Redis/Tablestore)存 state_dict

**这就是为什么 AgentScope 的 README 一句 _"separates creation from state"_ 撬动了 25k stars** — 这不是营销话术,是一个**真正的抽象层**。

---

## 2. 机制 — 132 行做对了什么

完整源码 `src/agentscope/module/_state_module.py:20-152`。三个设计选择:

### 2.1 `__setattr__` 钩子自动追踪 sub-module

```python
[code: _state_module.py:29-39]
def __setattr__(self, key: str, value: Any) -> None:
    """Set attributes and record state modules."""
    if isinstance(value, StateModule):
        if not hasattr(self, "_module_dict"):
            raise AttributeError(
                f"Call the super().__init__() method within the constructor "
                f"of {self.__class__.__name__} before setting any attributes.",
            )
        self._module_dict[key] = value
    super().__setattr__(key, value)
```

**用户什么都不做**,只要 `self.memory = some_memory`,如果 `some_memory` 是 StateModule 子类,就自动注册进 `_module_dict`。

**对照 PyTorch**:`torch.nn.Module.__setattr__` 完全相同模式 — `Module` / `Parameter` / `Buffer` 三类自动追踪。AgentScope 简化成只追踪 `StateModule`(因为 agent state 不区分 trainable vs frozen)。

**为什么用 `OrderedDict` 而非普通 dict**(`:26`):序列化顺序稳定 → JSON 输出 byte-identical 跨多次 save → **prompt cache 边界稳定**(详见 [[concept/runtime-context-injection]] §9 cache-aware design)+ **diff 工具友好**(state 演化 git-like)。

### 2.2 递归 state_dict / load_state_dict

```python
[code: _state_module.py:49-72]
def state_dict(self) -> dict:
    state = {}
    for key in self._module_dict:
        attr = getattr(self, key, None)
        if isinstance(attr, StateModule):
            state[key] = attr.state_dict()   # ← 递归

    for key in self._attribute_dict:
        attr = getattr(self, key)
        to_json_function = self._attribute_dict[key].to_json
        if to_json_function is not None:
            state[key] = to_json_function(attr)
        else:
            state[key] = attr
    return state
```

**两类**:`_module_dict` 是 nested StateModule(递归)+ `_attribute_dict` 是 atom 字段(per-attribute serializer)。

**关键的 Python 巧妙之处**:`load_state_dict` 用 `strict=True` 模式(`:74-99`)抛 `KeyError` 如果 state_dict 少 key,但 `strict=False` 可跳过 — **支持 schema 演化**(你给 StateModule 加新字段后,老 state_dict 缺这个 key 不报错)。

```python
[code: _state_module.py:84-92]
for key in self._module_dict:
    if key not in state_dict:
        if strict:
            raise KeyError(...)
        continue
    self._module_dict[key].load_state_dict(state_dict[key])
```

### 2.3 `register_state` — per-attribute 自定义 JSON 转换

```python
[code: _state_module.py:108-151]
def register_state(
    self,
    attr_name: str,
    custom_to_json: Callable[[Any], JSONSerializableObject] | None = None,
    custom_from_json: Callable[[JSONSerializableObject], Any] | None = None,
) -> None:
    attr = getattr(self, attr_name)
    if custom_to_json is None:
        # Make sure the attribute is JSON serializable natively
        try:
            json.dumps(attr)
        except Exception as e:
            raise TypeError(...) from e
    if attr_name in self._module_dict:
        raise ValueError(...)
    self._attribute_dict[attr_name] = _JSONSerializeFunction(
        to_json=custom_to_json,
        load_json=custom_from_json,
    )
```

**两条契约**:
- 如果你不给 `custom_to_json`,框架**立刻测**(`json.dumps(attr)` 试一次)— 失败立即 raise,而不是延迟到 save 时才炸
- `attr_name` 不能同时在 `_module_dict` 和 `_attribute_dict` — 同名 sub-module 和 atom 字段冲突会抛 ValueError

**这是 production-grade 防御**:让出错点尽量早 + 让契约违反不能静默。对照 PyTorch `register_buffer` 不做 type check 但也没 JSON 序列化需求 — AgentScope 的扩展是 reasonable trade-off。

---

## 3. 为什么 132 行能 work — 隐藏的 framework-wide self-discipline

`StateModule` 本身 132 行没什么 — 关键是 **agentscope 整个框架的 26 个模块,凡是 stateful 的都自觉 inherit StateModule**:

| 模块 | StateModule 子类 | 序列化什么 |
|---|---|---|
| `agent/_agent_base.py` | `AgentBase` | 模型 ID / sys_prompt / 当前角色信息 |
| `memory/InMemoryMemory` | `MemoryBase` | 完整对话 history Msg 列表 |
| `memory/_long_term_memory/` | `LongTermMemoryBase` | 长期 memory entries(经过 ReMe / 自家 reflector) |
| `plan/_plan_notebook.py` | `PlanNotebook(StateModule)` (`:172`) | 当前 plan + subtask state + plan history |
| `tool/_toolkit.py` | `Toolkit` | 注册的 tool function 元数据(不存 callable 本身) |
| `rag/_knowledge_base.py` | `KnowledgeBase` | retriever config + 索引元数据(不存 doc body) |

**对照其他 agent 框架** — 这些模块在 LangChain / AutoGen / CrewAI 里**各自定义自己的序列化方法**(dict_dump / model_dump / to_dict),**没有递归的统一抽象**。

**结果**:AgentScope 写一个 ReActAgent,继承 StateModule,**它的 memory / plan / toolkit / rag 全是 sub-StateModule**,一次 `agent.state_dict()` 拿到完整树形 JSON。Session backend 拿到这个 JSON 存 Tablestore / Redis / 文件,完全无知 agent 内部结构。

**这是 framework discipline > abstraction power 的好例子** — `StateModule` 本身简单,但因为所有 stateful 类都遵守它,整个框架的可序列化性 / 可迁移性 / 可观测性都 emergent 出来。

---

## 4. 跟 Codex `resume_agent` 校准 — 完全不同的 cancellation/resume 模型

Codex `core/src/agent/registry.rs` + `actor.rs` 也支持 pause / resume,但**机制完全不同**(详见 [[harness/codex]] §multi_agent_isolation):

| | AgentScope | Codex |
|---|---|---|
| **抽象层** | StateModule 树(per-attribute JSON 序列化) | Actor mailbox(Rust message passing) |
| **resume 单位** | 整个 agent state 树 | 单个 actor 的 mailbox 消息流 |
| **资源边界** | 进程内 / 跨进程(A2A + Nacos)/ 跨主机 | 同一进程内不同 actor(用 channel)+ `resume_agent` 跨 actor 切换 |
| **持久化** | 4 backend(JSON/Redis/Tablestore/abstract)| rollout JSONL + SQLite |
| **interrupt 信号** | `asyncio.CancelledError` Python 风 | `CancellationToken` Rust 风 + actor `select!` |
| **fake-result 补全** | ✅ in-flight tool_call 自动注入 fake `ToolResultBlock`(`_react_agent.py:639-653`) | ⚠️ 未在源码中明确验证(可能在 actor message handler 内) |

**核心区别**:AgentScope 的 resume 是 **跨进程 / 跨时间维度**(把整个 agent 状态搬到另一个 runtime),Codex 的 resume 是 **同进程 actor 之间切换**。两者解决的问题不同 — AgentScope 服务于 _"production agent live migration"_,Codex 服务于 _"multi-agent cooperation"_。

**深入一点的对照**:
- AgentScope 是 _"agent 可以被冻成 JSON 移动到任意地方再化冰复活"_
- Codex 是 _"多个 agent 像 Erlang actor 一样在同一个 BEAM-like runtime 里 message-passing,可以让其中一个 sleep 让另一个跑"_

如果你要做**"用户登录后,无论从手机切到电脑都接续上之前的 agent 状态"** — AgentScope 路线
如果你要做**"主 agent 暂停让 sub-agent 跑完任务再回来"** — Codex 路线

---

## 5. 跟 Claude Code 校准 — Claude Code 完全没有 state 抽象

我特意 grep 了 Claude Code v2.1.88 看是否有 StateModule 类抽象:

```
grep -rE "class StateModule|state_dict|load_state_dict" /Users/youren/myspace/claw-cli-claude-code-source-code-v2.1.88/src/ --include="*.ts" --include="*.tsx" 2>/dev/null
→ 业务代码 0 命中
```

Claude Code 的 session 持久化是 `Session` 类内部的 `serialize() / deserialize()`,但 **memory / plan / hooks 各自的状态没有统一抽象**。这意味着:
- Claude Code session resume 是 **session-level**(整个 session 序列化为 JSON)
- 没有 _"只迁移 memory 不迁移 toolkit"_ 这种 fine-grained 能力
- A/B 测试不同 hook configuration 但保留同 memory 需要手动 surgery

**这不是 Claude Code 弱**,而是设计选择不同:Claude Code 是 IDE-friendly CLI / 终端工具,session 是原子单位;AgentScope 是**框架库**,需要支撑各种 deployment 场景所以做了更细的抽象。

---

## 6. PyTorch `nn.Module` 影响痕迹 — 几乎每一行都是

```
PyTorch nn.Module → AgentScope StateModule:
─────────────────────────────────────────────────
nn.Module._modules: OrderedDict          → StateModule._module_dict: OrderedDict
nn.Module._parameters: OrderedDict       → StateModule._attribute_dict: OrderedDict
nn.Module.__setattr__ 自动跟踪 sub-module → StateModule.__setattr__ 同款
nn.Module.state_dict() 递归             → StateModule.state_dict() 同款
nn.Module.load_state_dict(strict=True)   → StateModule.load_state_dict(strict=True) 同款
nn.Module.register_parameter(name, value)→ StateModule.register_state(attr_name, ...)
                                            (扩展:custom_to_json / custom_from_json)
```

**唯一不同**:PyTorch 默认所有 `nn.Parameter` 都已经是 tensor(序列化用 pickle),AgentScope 默认要求 JSON 兼容(序列化跨进程 / 跨语言),所以加了 per-attribute 的 `custom_to_json / custom_from_json` hook。

**所以**:理解 AgentScope StateModule 的最快路径是 _"如果你写过 PyTorch,把 Module 想成 agent state container 而非 neural network — 一切都 carry over"_。

---

## 7. 一个未解的工程问题 — `_attribute_dict` 何时初始化?

读源码时我卡了 5 分钟:

```python
[code: _state_module.py:24-27]
def __init__(self) -> None:
    """Initialize the state module."""
    self._module_dict = OrderedDict()
    self._attribute_dict = OrderedDict()
```

但 `__setattr__`(`:29-39`)只追踪 `StateModule` 实例进 `_module_dict`,**没看到自动把 atom 字段放进 `_attribute_dict`**。

答案在 `register_state`(`:108-151`):**atom 字段必须用户显式 `self.register_state("my_attr")` 才会被追踪**。

**对照 PyTorch**:`nn.Parameter` 是 _"任何 tensor wrap 一下就自动注册"_,AgentScope `register_state` 是 _"任何 atom 字段必须显式注册"_。

**这是 trade-off**:
- 显式 better — 用户清楚哪些 atom 字段会被持久化(`temperature` 不会被存,但 `temperature` 是 attribute)
- 隐式 better — 减少模板代码

AgentScope 选显式 — 跟 PyTorch `register_buffer` 也是显式同款。**Pythonic 的"explicit > implicit"。**

---

## 8. SkillForge 借鉴清单 — R-AS-1 详细化

SkillForge 当前状态(verified from [[concept/agent-memory]] + [[concept/agent-harness]]):

```
SkillForge 持久化层 [verified-from-source]:
─────────────────────────────────────────────
SessionEntity (Spring @Entity)        ← session 元数据
MessageEntity                          ← 对话历史
ToolUseRecordEntity                    ← tool 调用
MemoryEntity (observation/reflection/  ← 三态 memory
              optimized)
SubAgentRunEntity                      ← SubAgent 运行
SkillEntity + SkillVersionEntity       ← skill 注册
LlmTraceEntity + LlmSpanEntity         ← 可观测
AgentDefinitionEntity                  ← agent 配置

→ 各自 Spring JPA, 各自序列化 (无统一抽象)
```

**对照 AgentScope StateModule 路线,SkillForge 的 R-AS-1 实施路径**:

| 步骤 | 描述 | 工作量 |
|---|---|---|
| 1 | 定义 `SkillForgeStateModule<T>` 接口,带 `toState(): JsonNode` / `fromState(JsonNode): T` / `mergeNestedModules(): Map<String, StateModule>` | < 半周 |
| 2 | MemoryEntity / SkillEntity / SubAgentRunEntity 各加 toState / fromState 静态方法(Jackson `@JsonSerialize` 用) | 1 周 |
| 3 | 新建 `AgentStateAggregator` service 把 N 个 StateModule 收集成单一 nested JSON | < 半周 |
| 4 | SessionBase 同款抽象 `saveAgentState(sessionId, stateModules: Map<String, StateModule>)` 接 Spring `PersistenceContext` | 1 周 |
| 5 | per-attribute custom JSON converter 用 Jackson `JsonSerializer<T> / JsonDeserializer<T>`(Java 原生) | 配套 |
| 6 | strict mode 用 Bean Validation `@NotNull`(Java 已有) | 配套 |

**总工作量**:2-4 周(单工程师全职)。**ROI**:撬动跨环境迁移 / A/B / 时间旅行 debug / RL training dataset 5 个 capability。

---

## 9. 学习视角 — 这页让我自己学到什么

1. **Framework discipline > abstraction power**:`StateModule` 132 行不复杂,但**所有 26 个模块自觉遵守它**比抽象本身重要 10 倍。任何 SkillForge 推广 StateModule 风格抽象的提议,**第一关不是设计而是 codebase-wide self-discipline**。
2. **PyTorch 是 agent runtime 设计的金矿**:`nn.Module` / `optim.Optimizer` / `Dataset` / `DataLoader` 这些抽象都是十年磨出来的 production-grade 设计。AgentScope 的 StateModule 只是第一波 — 未来可能还会看到 agent runtime 借鉴 `Optimizer.step()` / `LRScheduler` / `EMA(exponential moving average)` 用于 agent self-improvement。
3. **README 营销话术不一定是 BS**:第一次看到 _"separates an agent's creation from its state"_ 时我以为是营销。读完 132 行 + 看 framework-wide 实践,确认这是真东西。但很多框架的 README 类似话术**就是 BS**(没有底层抽象支撑)。**判断标准**:有没有 1 个 dedicated 抽象类 + 几乎所有 stateful 模块都 inherit。

---

## 引用链路

- 源码:`/Users/youren/myspace/agentscope/src/agentscope/module/_state_module.py:1-152`(完整 132 行)
- 配套:`session/_session_base.py:7-48`(消费侧)
- ReActAgent 中应用:`agent/_react_agent.py:575-655`(interrupt + state preservation)
- 跨页对照:[[concept/multi-agent-isolation]] / [[concept/agent-memory]] / [[concept/self-evolution]] / [[harness/codex]](resume_agent)
- 学术 backbone:arxiv 2402.14034
- PyTorch 原型:[pytorch.org docs nn.Module](https://docs.pytorch.org/docs/stable/generated/torch.nn.Module.html)
