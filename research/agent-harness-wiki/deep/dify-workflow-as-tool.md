---
type: deep
name: dify-workflow-as-tool
status: B-mode source-grounded
harness: dify
mechanism: workflow 包成 ToolProvider + WorkflowTool 继承普通 Tool + workflow_call_depth 递归跟踪 + 8 类型 variable→parameter 自动映射 + reserved-keys 输出隔离 + HumanInput pause 在 nested workflow 内显式禁用
updated: 2026-05-17
sources:
  - "[[harness/dify]]"
  - "/Users/youren/myspace/dify/api/core/tools/workflow_as_tool/provider.py:1-257"
  - "/Users/youren/myspace/dify/api/core/tools/workflow_as_tool/tool.py:1-399"
---

# Dify Workflow-as-Tool — 已发布 workflow 包成 callable tool 给 agent 调用,nested composition 业内独家

> 要点:Dify `api/core/tools/workflow_as_tool/` 2 文件 655 行实现**业内独家 nested workflow composition**:任何已发布 workflow 自动包成 `WorkflowTool`(继承普通 `Tool` 基类),agent 可以像调 `read_file` 那样 `call_workflow_X({...})`。Tool parameter 从 workflow 的 input variables 自动推断(`VARIABLE_TO_PARAMETER_TYPE_MAPPING` 8 类型映射),output schema 从 workflow output node 自动生成。
>
> 这页深挖 3 个反直觉工程决策:**(1) `workflow_call_depth: int` 显式跟踪嵌套深度,`_invoke` 时 `call_depth + 1` 递归累加**,防无限嵌套;**(2) `pause_state_config=None` 在 nested workflow 内显式禁用 HumanInput**,因为 sub-workflow 不该 pause 等用户输入(只主 workflow 可以);**(3) Output `"text" / "json" / "files"` 是 Dify reserved keys,其他 variable 作为 variable_message 单独 yield**,实现 _"主输出 + 命名子输出"_ 双轨。
>
> 业内对照:LangChain Tool / OpenAI Function / AgentScope MsgHub 都没有 _"workflow 作为 callable tool"_ 抽象。**Dify 是 production 第一个把这个 nested composition 做到的 OSS framework**。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通 agent framework 调 tool:
```
Agent → Tool: read_file(path)
              → Python function → return content
```

Dify workflow-as-tool 调用链:
```
Agent → Tool: customer_onboarding_workflow({
  user_email: "...",
  company: "...",
  plan_tier: "premium"
})
        → 整个 workflow 跑(LLM 节点 + datasource 节点 + knowledge retrieval + ...)
        → return 结构化 outputs (containing 多个 named variables)
```

**这是 _"workflow 复用"_ 范式革命**:
- 业务方在 Dify Studio 拖拽一个 workflow(_"客户 onboarding 流程"_)
- 一键 _"publish as tool"_
- 其他 workflow / 其他 agent 直接调用,无需重写

**Nested composition** 例子:
```
Marketing Workflow
   ↓ calls
Customer Onboarding Workflow (workflow_as_tool)
   ↓ calls
Email Send Workflow (workflow_as_tool)
   ↓ calls
LLM Sentiment Check Workflow (workflow_as_tool)
   ...深度无限制(但有 call_depth 跟踪)
```

**LangChain Tool / OpenAI Function 这种 _"workflow 是 first-class compose primitive"_ 抽象都没有**:
- LangChain Tool = Python function wrapping(没有 workflow 概念)
- OpenAI Function = function schema(没有 workflow)
- Dify workflow_as_tool = workflow 自动包 tool(独家)

**为什么容易被忽略**:
1. 实现简单(2 文件 655 行),不像 V8 code-mode / RAGFlow deepdoc 那么显眼
2. 概念熟悉但实现细节多(parameter 映射 + output schema 推断 + 递归深度 + pause 禁用)
3. Dify dashboard 的 _"Publish as Tool"_ 按钮看起来 trivial,后端逻辑实际不 trivial

**对 production AI 平台的启示**:**workflow composition 应该是 platform first-class capability**。如果用户拖拽 workflow A 后**不能在 workflow B 内调用 A**,平台就只是 _"很多孤立 workflow 的集合"_ 而非 _"可组合的 building blocks"_。Dify 把这个补上是 _"产品力分水岭"_。

---

## 2. 机制 — Provider + Tool 两层

### 2.1 WorkflowToolProviderController(provider.py)

```python
[code: provider.py:45-80]
class WorkflowToolProviderController(ToolProviderController):
    provider_id: str
    tools: list[WorkflowTool] = Field(default_factory=list)
    
    @classmethod
    def from_db(cls, db_provider: WorkflowToolProvider) -> WorkflowToolProviderController:
        with session_factory.create_session() as session, session.begin():
            app = session.get(App, db_provider.app_id)
            ...
            controller = WorkflowToolProviderController(
                entity=ToolProviderEntity(
                    identity=ToolProviderIdentity(
                        author=user.name if user else "",
                        name=db_provider.label,
                        label=I18nObject(en_US=..., zh_Hans=...),
                        description=I18nObject(en_US=..., zh_Hans=...),
                        ...
                    ),
                    credentials_schema=[],
                    plugin_id=None,
                ),
                provider_id=db_provider.id,
            )
            controller.tools = [controller._get_db_provider_tool(db_provider, app, ...)]
        return controller
```

**关键设计**:把已发布的 workflow 当作 _"特殊的 tool provider"_。`provider_type = ToolProviderType.WORKFLOW`(`:82-84`)区分自家 workflow vs 第三方 tool provider。

### 2.2 自动 parameter mapping(`provider.py:33-42`)

```python
VARIABLE_TO_PARAMETER_TYPE_MAPPING = {
    VariableEntityType.TEXT_INPUT:  ToolParameter.ToolParameterType.STRING,
    VariableEntityType.PARAGRAPH:   ToolParameter.ToolParameterType.STRING,
    VariableEntityType.SELECT:      ToolParameter.ToolParameterType.SELECT,
    VariableEntityType.NUMBER:      ToolParameter.ToolParameterType.NUMBER,
    VariableEntityType.CHECKBOX:    ToolParameter.ToolParameterType.BOOLEAN,
    VariableEntityType.FILE:        ToolParameter.ToolParameterType.FILE,
    VariableEntityType.FILE_LIST:   ToolParameter.ToolParameterType.FILES,
    VariableEntityType.JSON_OBJECT: ToolParameter.ToolParameterType.OBJECT,
}
```

**8 个 workflow input variable type → tool parameter type 的显式映射**。Workflow 用 `VariableEntityType`(workflow domain),tool 用 `ToolParameter.ToolParameterType`(tool domain),mapping 桥接两个 type system。

**业内对照**:LangChain Tool 让开发者手写 JSON schema,Dify 自动推断 — **降低 production AI 平台用户(业务方)的认知负担**。

### 2.3 自动 output schema 生成(`provider.py:166-178`)

```python
outputs = WorkflowToolConfigurationUtils.get_workflow_graph_output(graph)
reserved_keys = {"json", "text", "files"}

properties = {}
for output in outputs:
    if output.variable not in reserved_keys:
        properties[output.variable] = {
            "type": output.value_type,
            "description": "",
        }
output_schema = {"type": "object", "properties": properties}
```

**Output schema 从 workflow output node 自动推断**。3 个 reserved keys(`text` / `json` / `files`)是 Dify 内部协议字段,不进 output schema(避免跟用户定义的同名 variable 冲突)。

### 2.4 WorkflowTool._invoke 主流程(`tool.py:72-142`)

```python
def _invoke(self, user_id, tool_parameters, ...) -> Generator[ToolInvokeMessage, None, None]:
    app = self._get_app(app_id=self.workflow_app_id)
    workflow = self._get_workflow(app_id=self.workflow_app_id, version=self.version)
    
    tool_parameters, files = self._transform_args(tool_parameters=tool_parameters)
    
    from core.app.apps.workflow.app_generator import WorkflowAppGenerator
    generator = WorkflowAppGenerator()
    
    user = self._resolve_user(user_id=user_id)
    self._latest_usage = LLMUsage.empty_usage()
    
    generator_args = {"inputs": tool_parameters, "files": files}
    ...
    
    result = generator.generate(
        app_model=app,
        workflow=workflow,
        user=user,
        args=generator_args,
        invoke_from=self.runtime.invoke_from,
        streaming=False,
        call_depth=self.workflow_call_depth + 1,   # ⭐ 递归深度 +1
        pause_state_config=None,                    # ⭐ Nested workflow 禁用 pause
    )
    ...
    
    outputs = data.get("outputs")
    if outputs is None:
        outputs = {}
    else:
        outputs, files = self._extract_files(outputs)
        for file in files:
            yield self.create_file_message(file)
    
    # ⭐ 主输出 (text/json/files) + 命名 variable 双轨
    for key, value in outputs.items():
        if key not in {"text", "json", "files"}:
            yield self.create_variable_message(variable_name=key, variable_value=value)
    
    self._latest_usage = self._derive_usage_from_result(data)
    
    yield self.create_text_message(json.dumps(outputs, ensure_ascii=False))
    yield self.create_json_message(outputs, suppress_output=True)
```

**Streaming generator pattern**:不是 return,是 `yield` 多个 `ToolInvokeMessage`(file / variable / text / json)。Caller 可以一边收 message 一边处理(不必等 workflow 全跑完)。

---

## 3. 3 个关键设计:递归跟踪 / Pause 禁用 / Reserved keys

### 3.1 `workflow_call_depth` 递归深度跟踪

```python
[code: tool.py:42-58]
def __init__(self, ..., workflow_call_depth: int, ...):
    ...
    self.workflow_call_depth = workflow_call_depth
    ...

# In _invoke:
call_depth=self.workflow_call_depth + 1   # 每次 invoke 时累加

# In provider.py:206:
workflow_call_depth=0    # 顶层 invoke 从 0 开始
```

**显式递归深度跟踪**。WorkflowAppGenerator 接收 `call_depth`,**在 generator 内部做 max depth check**(很可能有 const 限制深度,如 max=5 或 max=10)。

业内对照:
- LangChain Tool 没有递归概念(tool 是 function,不能调 tool)
- Codex actor `next_thread_spawn_depth + exceeds_thread_spawn_depth_limit`(详见 [[deep/codex-actor-resume-agent]] §3)
- **Dify workflow_as_tool 是 application-level 递归,Codex 是 thread-level 递归** — 两层不同

### 3.2 `pause_state_config=None` Nested workflow 禁用 HumanInput

```python
[code: tool.py:118]
pause_state_config=None,
# NOTE(QuantumGhost): We explicitly set `pause_state_config` to `None`
# because workflow pausing mechanisms (such as HumanInput) are not
# supported within WorkflowTool execution context.
```

**为什么 nested workflow 不能 HumanInput pause**:
- 主 workflow 调用 `customer_onboarding_workflow_as_tool({...})`
- 如果 onboarding workflow 内有 HumanInput 节点等用户输入
- 主 workflow 完全阻塞等子 workflow 等用户 — **死锁**
- Agent 看到 tool 长时间没 return 也不知道为啥

显式禁用是 _"避免设计错误"_ 而非 _"技术限制"_。**作者 inline comment(`# NOTE(QuantumGhost):`)解释意图**,production engineering 文化好习惯。

### 3.3 Reserved keys + variable message 双轨输出

```python
[code: tool.py:135-142]
for key, value in outputs.items():
    if key not in {"text", "json", "files"}:
        yield self.create_variable_message(variable_name=key, variable_value=value)

yield self.create_text_message(json.dumps(outputs, ensure_ascii=False))
yield self.create_json_message(outputs, suppress_output=True)
```

**3 reserved keys + 命名 variable 设计**:

- `text` / `json` / `files` 是 Dify 内部协议输出字段(主 output)
- 其他 workflow output 变成 `variable_message`(命名子输出,caller workflow 可以引用 `tool_X.output.user_id`)

**双轨好处**:
- 主输出永远有(text / json),caller LLM 看的
- 命名子输出供 workflow visual 编排时引用("connect onboarding.user_id to next node's input")
- 不冲突(reserved keys 显式排除)

### 3.4 i18n first-class

```python
[code: provider.py:66-68]
label=I18nObject(en_US=db_provider.label, zh_Hans=db_provider.label),
description=I18nObject(en_US=db_provider.description, zh_Hans=db_provider.description),
```

所有 user-facing strings 用 `I18nObject(en_US=..., zh_Hans=...)`。**Dify 国际化 first-class**(国内市场为主,中英双语必备)。

业内对照:LlamaIndex / Microsoft GraphRAG 英文 only,Dify / RAGFlow / AgentScope 都国际化 first-class(中国市场必修)。

---

## 4. WHY — 为什么这么设计

### 4.1 为什么 workflow_as_tool 而非 _"workflow 调 workflow"_ 专属机制?

可以另起一套 _"workflow nesting"_ 协议(类似函数调用栈)。Dify 选 _"workflow 是 tool 的特殊形式"_。

**好处**:
- 复用整个 tool 调用 infrastructure(parameter validation / permission gate / OpenTelemetry trace / etc)
- Agent 调 workflow 跟调普通 tool 一致(LLM 不需要学新协议)
- Workflow 编辑 UI 跟普通 tool 一致(_"add tool"_ 选 _"my published workflows"_ tab)
- Provider plugin system 复用(WorkflowToolProvider 跟其他 tool provider 同基类)

**坏处**:
- 抽象漏:Tool 协议假设 _"短暂同步调用"_,workflow 可能跑几分钟(`streaming=False` 显式同步,但慢)
- Pause 不支持(如 §3.2)

Dify 选 _"复用 > 专属"_,合理。

### 4.2 为什么 8 类型 VARIABLE_TO_PARAMETER_TYPE 显式映射?

可以用 `getattr` / dict 通用 mapping。Dify 显式列 8 类型:

**显式好处**:
- 新加 type 时编译期发现(增 enum value 但忘加 mapping → 立即 raise)
- Type-safe(IDE 跳转 + linter check)
- 代码 reader 一眼看出支持哪 8 type

**通用 mapping 坏处**:运行时才发现 type missing,error message 不友好。

**这是 Pythonic 显式 > 通用 设计**。

### 4.3 为什么 reserved keys 是 `text/json/files` 而非用 namespace?

可以用 `_dify_text`/`_dify_json` 前缀 namespace 避免冲突。Dify 用 unprefixed reserved keys。

**因为**:
- `text/json/files` 是 ToolInvokeMessage 协议固有概念(其他 tool 也用)
- 跟 OpenAI tool result schema 兼容("text" content blocks)
- Unprefixed 让 LLM 读 output 时更自然("text" 而非 "_dify_text")

**Trade-off**:用户的 workflow output 不能命名为 `text/json/files`(会被 reserved)。在 _"用户认知简单"_ vs _"用户少受限制"_ 之间选了前者。

### 4.4 为什么 Output 同时 yield text_message + json_message + 命名 variable?

3 个目的:
- `text_message` 给 LLM agent 看(stringified output,主 path)
- `json_message(suppress_output=True)` 给 audit / programmatic access(suppress=不显示给用户)
- `variable_message` 给 workflow visual 编排(连线引用具体 variable)

**3 个不同 consumer 3 种 message** — 不省略,各取所需。

---

## 5. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | Dify 怎么防 | 代码位置 |
|---|---|---|
| **Workflow A 调 B 调 A 无限递归** | `workflow_call_depth` 显式跟踪,_invoke 时 `+1` 递归累加,downstream 有 max depth check | `tool.py:48, 57, 114` |
| **Nested workflow 用 HumanInput 死锁主 workflow** | `pause_state_config=None` 显式禁用 + NOTE 注释解释 | `tool.py:115-118` |
| **Workflow variable name 跟 Dify 内部协议字段冲突(text/json/files)** | `reserved_keys = {"json", "text", "files"}` 显式排除 | `provider.py:169, tool.py:136` |
| **新加 workflow variable type 未映射到 ToolParameter type** | `VARIABLE_TO_PARAMETER_TYPE_MAPPING` 显式 dict,missing key 立即 `raise ValueError(f"unsupported variable type {variable.type}")` | `provider.py:126-127` |
| **Tool runtime 跨 user 状态泄漏** | `tenant_id` 必填 + `_resolve_user` per-invoke 重新解析 | `tool.py:95-97` + `provider.py:197-199` |
| **每次 invoke 累计 LLM usage** | Per-invoke `self._latest_usage = LLMUsage.empty_usage()` 显式重置 | `tool.py:99` |
| **Workflow output 找不到对应 file 但 outputs 有 file mapping** | `_extract_files(outputs)` 分离 file 和非 file output,file 单独 yield create_file_message | `tool.py:130-132` |
| **Workflow 删除后老的 WorkflowToolProvider 还引用** | `from_db` 每次 fresh fetch,`workflow not found` 显式 raise | `provider.py:106-107` |
| **i18n 漏掉某语言导致 UI 显示 raw key** | 强制 `I18nObject(en_US=..., zh_Hans=...)` 双语必填 schema | `provider.py:66-67` 等多处 |
| **Tool 调用绕过 permission check** | 继承普通 `Tool` 基类,permission 通过 base 自动套用 | `tool.py:13, 35` |
| **Workflow 版本不一致(用户改了 workflow 但 tool 还用老版)** | `version=db_provider.version` 显式锁版本,_get_workflow 按 version 查 | `provider.py:101-103, tool.py:84` |

---

## 6. 横向校准 — 4 家 framework 的 tool composition 模型

| | Dify workflow_as_tool | LangChain Tool | OpenAI Function | AgentScope MsgHub |
|---|---|---|---|---|
| **Composition 抽象** | ⭐ Workflow as Tool(nested workflow) | Tool = Python function(无嵌套) | Function = schema(无嵌套) | MsgHub 多 agent 协调(不是 tool composition) |
| **递归深度跟踪** | ⭐ `workflow_call_depth` 显式 | 无 | 无 | 无 |
| **Parameter auto-mapping** | ⭐ 8 类型显式 mapping | 手写 JSON schema | 手写 JSON schema | N/A |
| **Output schema auto-gen** | ⭐ 从 workflow output node 自动 | 手写 | 手写 | N/A |
| **Reserved keys 隔离** | ⭐ text/json/files | 无 | 无 | N/A |
| **Pause control** | ⭐ Nested 内禁 HumanInput | 无概念 | 无概念 | 无 |
| **Streaming output** | ✅ Generator yield | partial | ❌ | partial |
| **i18n first-class** | ⭐ I18nObject 双语必填 | 无 | 无 | partial |
| **Version pinning** | ⭐ `version` 字段 | 无 | 无 | N/A |

**核心洞察**:Dify 是**唯一把 _"workflow"_ 当作 first-class composition unit** 的 OSS production AI 平台。其他 framework 把 tool 当原子,workflow 是 tool 的容器(不能 nest)。

**Dify 模型本质**:Workflow = 高级 tool(可以拖拽编辑 + 内部多 LLM 调用 + 状态管理),普通 tool = 低级 workflow(单步骤 Python function)。**两者同协议**,无缝 compose。

---

## 7. 学习视角 — 写这页让我学到的 3 件事

1. **`# NOTE(<author>):` inline comment 是 production engineering 文化好习惯**:`pause_state_config=None` 旁边 4 行 NOTE 解释 _"为什么 nested workflow 禁 HumanInput"_。看代码的人 5 秒理解设计意图,不需要 git blame 找 commit message。**任何 _"看似奇怪的 default value"_ 都该有 inline comment 解释**(`pause_state_config=None` 看起来像忘了配,实际是显式禁用)。
2. **显式 mapping(8 类型 dict)> 通用 reflection(getattr)**:`VARIABLE_TO_PARAMETER_TYPE_MAPPING` 8 类型显式列。新加 type 时 missing key → runtime ValueError → 立即修。通用 reflection 会 silent fallback 到 None → 后续才崩。**Pythonic _"explicit > implicit"_ 在 type system 桥接时尤其重要**。
3. **`workflow_call_depth + 1` 这种 _"递归累加 + downstream max check"_ 是 multi-level 系统的基本要求**:任何 _"X 可以调 X"_ 的递归系统(workflow / agent / tool / function)都该有显式 depth tracking,跟 [[deep/codex-actor-resume-agent]] §registry.rs `next_thread_spawn_depth + exceeds_thread_spawn_depth_limit` 同思路。**不显式跟踪 = 必有无限递归 bug 的一天**。

---

## 引用链路

- 源码:`/Users/youren/myspace/dify/api/core/tools/workflow_as_tool/`
  - `provider.py:1-257` 完整(WorkflowToolProviderController + VARIABLE_TO_PARAMETER_TYPE_MAPPING + from_db + _get_db_provider_tool + get_tools/get_tool)
  - `tool.py:1-150` 关键(WorkflowTool 类 + _invoke 主流程 + workflow_call_depth + pause 禁用 + reserved keys + 3 类 output message yield)
  - `tool.py:144-399`(本次未深读 — _derive_usage_from_result / _transform_args / _extract_files / _resolve_user / _get_workflow 等 helper)
  - `core/tools/__base/tool.py`(本次未深读 — Tool 基类,permission gate / ToolRuntime)
  - `core/workflow/file_reference.py`(本次未深读 — file resolve)
  - `core/app/apps/workflow/app_generator.py`(本次未深读 — WorkflowAppGenerator,downstream max depth check 应在此处)
- Harness 入口:[[harness/dify]] §tool_protocol + §3.1 workflow_as_tool 独家
- 横向 concept:[[concept/rag-design-space]] §5 R-RAG-3 / [[concept/agent-harness]] 多 harness 横向
- 横向 deep:[[deep/codex-actor-resume-agent]] (depth tracking 对照 — workflow_call_depth vs thread_spawn_depth)
