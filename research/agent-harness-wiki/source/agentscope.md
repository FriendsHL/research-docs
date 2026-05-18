---
name: agentscope
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: agentscope
version: 1.0.20
commit: b840fa927ad409aeddb9df359631ed586f2ae830
license: Apache-2.0
stars: 25.2k
last_ingested_at: 2026-05-17
contributed_to:
  - "[[harness/agentscope]]"
  - "[[concept/agent-state-separation]]"
---

# Source — AgentScope 1.0.20

> Alibaba 通义实验室生产级 Python agent 框架。clone @ `/Users/youren/myspace/agentscope/`,commit b840fa9 `chore(version): bumping verstion to 1.0.20`。Apache 2.0 / Python 3.10+。**arxiv 2402.14034**(学术 backbone)。

---

## 1. Top-level 仓库结构

```
agentscope/
├── .gemini/                          # Google Gemini CLI 集成配置
├── docs/                             # 含 roadmap.md
├── examples/                         # 9 大类 example
├── src/agentscope/                   # 26 模块 (见 §2)
├── tests/                            # 64 test 文件
├── pyproject.toml                    # Python 3.10+ / uv 一等
└── README.md + README_zh.md          # 双语
```

---

## 2. `src/agentscope/` 26 模块全清单 [verified]

按职责分组:

| 类 | 模块 | 关键文件 |
|---|---|---|
| **核心抽象** | `module/` | `_state_module.py:20-152` `StateModule` 类 — **PyTorch nn.Module 风格用于 agent state**(`state_dict / load_state_dict / _module_dict OrderedDict / register_state` + `custom_to_json / custom_from_json` per-attr 序列化函数) |
| **Agent** | `agent/` | `_agent_base.py` 774 行 / `_react_agent_base.py` 116 行 / `_react_agent.py` 1137 行 / `_a2a_agent.py` / `_realtime_agent.py` / `_user_agent.py` |
| **Model** | `model/` | 6 provider:`_anthropic_model.py` / `_dashscope_model.py`(Qwen)/ `_gemini_model.py` / `_ollama_model.py` / `_openai_model.py` / **`_trinity_model.py`**(RL training) |
| **Formatter** | `formatter/` | 7 provider-specific:`_a2a_formatter.py` / `_anthropic_formatter.py` / `_dashscope_formatter.py` / `_deepseek_formatter.py` / `_gemini_formatter.py` / `_ollama_formatter.py` / `_openai_formatter.py` + `_truncated_formatter_base.py`(基类含截断) |
| **Memory** | `memory/` | **双层**:`_long_term_memory/` + `_working_memory/`(业内罕见显式分离)+ `InMemoryMemory` |
| **RAG** | `rag/` | `_knowledge_base.py:13-130` `KnowledgeBase.retrieve / add_documents / retrieve_knowledge` + `_document.py` + `_reader/` + `_simple_knowledge.py` + `_store/` |
| **Plan** | `plan/` | **`_plan_notebook.py` 904 行** — 一等公民 plan(create / revise / update_subtask / finish_subtask / view / recover_historical_plan + `register_plan_change_hook`)+ `_plan_model.py` + `_in_memory_storage.py` + `_storage_base.py` |
| **Tool** | `tool/` | `_toolkit.py` + `_async_wrapper.py` + `_coding/` + `_multi_modality/` + `_text_file/` + `_response.py` + `_types.py` |
| **MCP** | `mcp/` | **`_mcp_function.py`** — 把 MCP tool 抽出为本地 callable Python 函数(独家); `_http_stateful_client.py` / `_http_stateless_client.py` / `_stdio_stateful_client.py` + `_stateful_client_base.py` + `_client_base.py` |
| **A2A** | `a2a/` | `_base.py` 25 行 + `_file_resolver.py` + **`_nacos_resolver.py`**(Alibaba Nacos 服务发现 — A2A 节点发现独家)+ `_well_known_resolver.py` |
| **Pipeline** | `pipeline/` | `_msghub.py` 多 agent 消息总线 + `_chat_room.py` + `_sequential.py` (in `_functional.py`) + `_class.py` |
| **Session** | `session/` | **4 backend**:`_json_session.py` / `_redis_session.py` / **`_tablestore_session.py`**(Alibaba Cloud Tablestore)/ `_session_base.py:7-48`(`save_session_state` / `load_session_state` 抽象) |
| **Tracing** | `tracing/` | OTel 集成 / 6 decorator:`trace / trace_llm / trace_reply / trace_format / trace_toolkit / trace_embedding` |
| **Realtime** | `realtime/` | Realtime voice agent 模块 |
| **TTS** | `tts/` | TTSModelBase 抽象 |
| **Evaluate** | `evaluate/` | `_benchmark_base.py` / `_evaluator/` / `_evaluator_storage/` / `_metric_base.py` / `_solution.py` / `_task.py` / **`_ace_benchmark/`** 内置 |
| **Tuner** | `tuner/` | **RL training pipeline 内置**:`_algorithm.py` / `_config.py` / `_dataset.py` / `_judge.py`(LLM-as-judge)/ `_model.py` / `_tune.py` / `_workflow.py` + `model_selection/` + `prompt_tune/` 子目录 |
| **Tune** | `tune/` | 单文件 stub(`__init__.py`),实际逻辑在 `tuner/` |
| **Hooks** | `hooks/` | **极简 1 文件**:`_studio_hooks.py` — vs Claude Code 27 / opencode 27,显著少 |
| **Embedding** | `embedding/` | embedding 抽象 |
| **Token** | `token/` | `TokenCounterBase` 抽象 |
| **Message** | `message/` | `Msg` / `ToolUseBlock` / `ToolResultBlock` / `TextBlock` / `AudioBlock`(多模态 first-class) |
| **Exception** | `exception/` | 异常类 |
| **Types** | `types/` | `JSONSerializableObject` 等 |
| **Misc** | `_logging.py` / `_run_config.py` / `_utils/` / `_version.py` | 顶层 |

---

## 3. 关键 verified 锚点

### 3.1 `StateModule` 核心 [code: src/agentscope/module/_state_module.py:20-152]

```python
class StateModule:
    """The state module class in agentscope to support nested state
    serialization and deserialization."""

    def __init__(self) -> None:
        self._module_dict = OrderedDict()
        self._attribute_dict = OrderedDict()

    def __setattr__(self, key, value):
        """Set attributes and record state modules."""
        if isinstance(value, StateModule):
            ...
            self._module_dict[key] = value
        super().__setattr__(key, value)

    def state_dict(self) -> dict: ...         # 递归
    def load_state_dict(self, state_dict, strict=True) -> None: ...   # 递归

    def register_state(
        self, attr_name,
        custom_to_json=None,
        custom_from_json=None,
    ) -> None: ...
```

**业内独家干净抽象**:PyTorch `nn.Module` 移植到 agent state。详见 [[concept/agent-state-separation]] §2-3 深度分析。

### 3.2 Interrupt 协议 [code: src/agentscope/agent/_react_agent.py:575-655]

```python
interrupted_by_user = False
try:
    async with tts_context:
        msg = Msg(name=self.name, content=[], role="assistant")
        if self.model.stream:
            async for content_chunk in res:
                ...
except asyncio.CancelledError as e:
    interrupted_by_user = True
    raise e from None

finally:
    await self.memory.add(msg)

    # Post-process for user interruption
    if interrupted_by_user and msg:
        # Fake tool results
        tool_use_blocks = msg.get_content_blocks("tool_use")
        for tool_call in tool_use_blocks:
            msg_res = Msg("system", [
                ToolResultBlock(
                    type="tool_result",
                    id=tool_call["id"],
                    name=tool_call["name"],
                    output="The tool call has been interrupted by the user.",
                ),
            ], "system")
            await self.memory.add(msg_res)
```

**关键工程细节**:in-flight tool_call 被中断后**合成 fake ToolResultBlock 补全 memory**,保证 resume 时 history 无孤儿 tool_use(无 result 的 tool_use 会让 OpenAI/Anthropic API 拒绝)。同样的关注点 Hermes `_sanitize_tool_pairs` / Claude Code `annotateBoundaryWithPreservedSegment` 也有但路径不同 — AgentScope 是 **interrupt-time 合成**,前两者是 **compact-time 修复**。

### 3.3 Chunk-level interrupt [code: _react_agent.py:687-699]

```python
async for chunk in tool_res:
    tool_res_msg.content[0]["output"] = chunk.content
    await self.print(tool_res_msg, chunk.is_last)

    # Raise the CancelledError to handle the interruption in the
    # handle_interrupt function
    if chunk.is_interrupted:
        raise asyncio.CancelledError()
```

Tool 执行流式输出时 chunk 自带 `is_interrupted` flag,**模型 generation 和 tool execution 都可被中断**。

### 3.4 SummarySchema — 强类型 memory 压缩 [code: _react_agent.py:43-87+]

```python
class SummarySchema(BaseModel):
    task_overview: str = Field(max_length=300, description=...)
    current_state: str = Field(max_length=300, ...)
    important_discoveries: str = Field(max_length=300, ...)
    next_steps: str = Field(max_length=200, ...)
    context_to_preserve: str = Field(max_length=300, ...)
```

**5 字段 pydantic 强类型**,每个字段含 max_length + description(LLM 训练 prompt)— vs Claude Code 自由文本 summary,**强类型防 hallucination + 防过长**。

### 3.5 SessionBase [code: src/agentscope/session/_session_base.py:7-48]

```python
class SessionBase:
    @abstractmethod
    async def save_session_state(
        self, session_id, user_id="",
        **state_modules_mapping: StateModule,
    ) -> None: ...

    @abstractmethod
    async def load_session_state(
        self, session_id, user_id="",
        allow_not_exist=True,
        **state_modules_mapping: StateModule,
    ) -> None: ...
```

**用 `**state_modules_mapping: StateModule` kwarg 收集任意命名的 StateModule**,实现统一序列化任意 agent 状态组合 — 不约束你必须存 memory + plan + toolkit,你存 N 个 StateModule 都行。

### 3.6 PlanNotebook 公共 API [code: src/agentscope/plan/_plan_notebook.py:172+]

`PlanNotebook(StateModule)` 9 个 async method:
- `create_plan(:232)` / `revise_current_plan(:303)` / `update_subtask_state(:433)` / `finish_subtask(:548)` / `view_subtasks(:653)` / `finish_plan(:694)` / `view_historical_plans(:739)` / `recover_historical_plan(:762)` / `get_current_hint(:845)`
- 2 hook 注册:`register_plan_change_hook(:866)` / `remove_plan_change_hook(:885)`
- **plan 是可编辑文档 + 有 history + 有 hooks** — 比 Claude Code 的 TodoWrite / Codex 的 goals.rs 都更结构化

---

## 4. Trinity-RFT 训练成绩(README example,verified 数字)

| Example | Model | 训练前 → 训练后 |
|---|---|---|
| Math Agent | Qwen3-0.6B | 75% → **85%** |
| Frozen Lake | Qwen2.5-3B | 15% → **86%** |
| Learn to Ask | Qwen2.5-7B | 47% → **92%** |
| Email Search | Qwen3-4B | 60% |
| Werewolf | Qwen2.5-7B | win rate 50% → **80%** |
| Data Augment AIME-24 | Qwen3-0.6B | 20% → **60%** |

**6 个 real production RL benchmark**,Frozen Lake +71pp / AIME +40pp 是显著提升。

---

## 5. 关键 2025-2026 演化

按 `README.md News` 倒序:
- **2026-04** Roadmap to AgentScope 2.0 (Discussion #1441)
- **2026-02** Realtime Voice Agent + Multi-Agent Realtime
- **2026-01** Database support + memory compression
- **2025-12** A2A protocol + TTS
- **2025-11** Trinity-RFT 集成(`tuner/`)+ **Anthropic Agent Skill 支持** + ReMe 长期记忆 + Alias-Agent + Data-Juicer Agent samples

---

## 6. 关键文件清单(供后续 query)

| 路径 | 行数 | 主题 |
|---|---|---|
| `src/agentscope/module/_state_module.py` | 152 | StateModule 抽象核心 |
| `src/agentscope/agent/_react_agent.py` | 1137 | ReActAgent 主类含 interrupt 协议 |
| `src/agentscope/agent/_agent_base.py` | 774 | Agent 基类 |
| `src/agentscope/plan/_plan_notebook.py` | 904 | PlanNotebook(plan 一等公民) |
| `src/agentscope/rag/_knowledge_base.py` | 130 | RAG KnowledgeBase |
| `src/agentscope/session/_session_base.py` | 48 | SessionBase(state_modules_mapping) |
| `src/agentscope/mcp/_mcp_function.py` | (small) | MCP-as-local-function 转换 |
| `src/agentscope/tuner/*` | 10 files | Trinity-RFT pipeline |

---

## 7. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/agentscope/](file:///Users/youren/myspace/agentscope/)
- arxiv:[2402.14034](https://arxiv.org/abs/2402.14034)
- 衍生 wiki 页:[[harness/agentscope]] / [[concept/agent-state-separation]]
- 跨页对照:[[harness/codex]](resume_agent / actor 模型)/ [[concept/multi-agent-isolation]] / [[concept/agent-memory]]
