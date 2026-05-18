---
type: harness
name: smolagents
status: active
language: python

last_ingested_version: 1.26.0.dev0
last_ingested_commit: 025b6ad
last_ingested_at: 2026-05-18
community_known_newer: false
known_post_ingest_features: []

sources:
  - "/Users/youren/myspace/smolagents/src/smolagents/agents.py"
  - "/Users/youren/myspace/smolagents/src/smolagents/local_python_executor.py"
  - "/Users/youren/myspace/smolagents/src/smolagents/remote_executors.py"
  - "/Users/youren/myspace/smolagents/src/smolagents/mcp_client.py"
  - "/Users/youren/myspace/smolagents/src/smolagents/memory.py"
  - "/Users/youren/myspace/smolagents/README.md"
  - "[[../source/smolagents]]"

agent_loop: |
  [verified-from-source]
  `MultiStepAgent` ABC (`agents.py:268`) + 2 子类:**`ToolCallingAgent`** (`:1215`,JSON tool_use 传统路线) + **`CodeAgent`** (`:1505`,code-mode 主角)。
  循环结构:while not final_answer and steps < max_steps → planning step(每 `planning_interval` 周期)→ action step → callback
  `RunResult` dataclass (`:196`) 含 output / state / steps / token_usage / timing / messages
  **`stream_outputs: bool = False`** 流式 + **`use_structured_outputs_internally`**(v1.17.0+)双 yaml prompt template 切换(`code_agent.yaml` / `structured_code_agent.yaml`)

tool_protocol: |
  [verified-from-source]
  `BaseTool / Tool` 抽象 (`tools.py` 1422 行) + `validate_tool_arguments` (`tools.py`) 入参校验
  **CodeAgent 不走传统 JSON tool_use** — LLM 写 Python 代码 `result = tool_x(...)` 直接调用,**业内 code-mode 路线**
  **3 个 code_block_tags 选项** (`agents.py:1556-1564`):`("<code>","</code>")` default / `("```python","```")` markdown / 自定义 tuple
  **4 种 tool 适配器** (README):MCP server / LangChain / HF Hub Space / Python function
  **MCP 集成**:`mcp_client.py` 171 行 wrap `mcpadapt.MCPAdapt + SmolAgentsAdapter`,支持 stdio + streamable-http + sse 全 transport
  **`structured_output: bool` FutureWarning**(`mcp_client.py:91-100`):_"v1.25 默认改 True"_ — **显式追 MCP spec 2025-06-18 outputSchema** 协议演化对齐

context_mgmt: |
  [verified-from-source]
  **极简 — 跟 5 step types 直接绑定**,无 compaction 机制
  `AgentMemory.get_succinct_steps()` (`memory.py:236`,去掉 model_input_messages)/ `get_full_steps()` 两档输出
  对照 [[claude-code]] 5 级 compaction / [[codex]] 3 套 compact + post-compact InitialContextInjection — smolagents 偏简 minimalism

memory: |
  [verified-from-source]
  **5 step types**:`SystemPromptStep / TaskStep / ActionStep / PlanningStep / FinalAnswerStep`(`memory.py:51-210`)+ base `MemoryStep` (`:42`)+ `ToolCall` (`:25`)
  `AgentMemory` (`:214`):`system_prompt + steps[]`,`reset()` / `replay(logger, detailed)` 完整轨迹回放
  **`return_full_code()` (`:273`) 拼接所有 code_action 成单 Python script** — elegant replay/debug 设计,业内独家(对照 CC autoDream 是 memory 持久化路线,smolagents 是 single-script reconstruction)
  **无 long-term / cross-session memory** — 跟 [[../eval/mem0]] / [[../eval/letta]] 等专门 memory framework 对比偏弱

multi_agent: |
  [verified-from-source]
  **`managed_agents` hierarchical 模式**(`agents.py:1608`)— manager agent 调用 sub-agent 走 `ManagedAgentPromptTemplate`(task + report 两段 prompt)
  **已知缺口**:_"Managed agents are not yet supported with remote code execution"_(`agents.py:1608` raise Exception)— **multi-agent + remote sandbox 互斥**

eval_loop: |
  ? 未深查 examples/ + tests/

self_evolution: |
  N/A — 无 self-evolution / 自动 prompt optimization 机制

observability: |
  [verified-from-source]
  `Monitor + AgentLogger + TokenUsage` (`monitoring.py` 273 行)
  Rich console 渲染(Group/Live/Markdown/Panel/Rule/Text 全 6 个 rich 模块,`agents.py:36-41`)
  **无 OTel / OpenInference 标准 trace export**(对照 wiki 21 家用 OTel)— 这是 smolagents 简化哲学的 trade-off

hooks: |
  [verified-from-source]
  **`CallbackRegistry` hooks-by-step-type 业内独特模式**(`memory.py:280`):`register(step_cls, callback)` — 按 step 类型注册,不是事件命名(CC 27 PreToolUse/PostToolUse)也不是 lifecycle(Hermes 16)
  跟 5 step types 1:1 绑定(SystemPromptStep / TaskStep / ActionStep / PlanningStep / FinalAnswerStep)— **新加 step type 自动有 hook 槽位**,设计 elegant

safety:
  sandbox: "[verified-from-source] **6 executor 选项**(`agents.py:1535` Literal type):**`local`**(LocalPythonExecutor AST interpreter,进程内 0 cold start)/ **`blaxel`** / **`e2b`**(Firecracker microVM)/ **`modal`** / **`docker`** / **`wasm`**(Pyodide+Deno WebAssembly)。详见 [[../concept/sandbox-isolation]] — smolagents 是 wiki 内首家 verified 直接用 E2B/Docker/Modal 的 harness"
  tool_acl: "[verified-from-source] **AST 解释器层多道防御**(`local_python_executor.py`):`BASE_PYTHON_TOOLS` 40+ 白名单 builtins(`:74-127`)+ `DANGEROUS_MODULES` 9 个黑名单(`:130` builtins/io/multiprocessing/os/pathlib/pty/shutil/socket/subprocess/sys)+ `DANGEROUS_FUNCTIONS` 黑名单(`:143` compile/eval/exec/globals/locals/__import__/...)+ `nodunder_getattr` 防 `__*__` 反射逃逸(`:68`)"
  bash_blocklist: "N/A — smolagents CodeAgent 跑 Python 代码非 shell,本身不暴露 bash"
  path_traversal: "[verified-from-source] 通过 `pathlib` 在 DANGEROUS_MODULES 黑名单 + LocalPythonExecutor 进程内执行(无新进程)+ 用户应叠加远程 executor(docker/e2b)做 OS-level 隔离"
  anti_runaway: "[verified-from-source] **4 硬限额**(`local_python_executor.py:57-60`):`DEFAULT_MAX_LEN_OUTPUT=50000` / `MAX_OPERATIONS=10_000_000` / `MAX_WHILE_ITERATIONS=1_000_000` / **`MAX_EXECUTION_TIME_SECONDS=30`**(timeout 强制中断)+ agent loop `max_steps` 控制"
  prompt_injection: "[verified-from-source] AST 解释器对 LLM-generated code 本身就是 _'untrusted-by-default'_ 处理 — 白名单 builtins + 黑名单 modules/functions + 反射防御。**但 tool description 走 LLM 没有 MCP-style untrusted annotation**"
  secret_scan: "? 未深查"
  auth: "N/A(本地框架,无 multi-tenant auth);**HF Hub `push_to_hub` / `from_hub` 走 HF token**"

provider: |
  [verified-from-source]
  `MODEL_REGISTRY` (`models.py` 2102 行) + **多 provider 类**:`InferenceClientModel`(HF Inference Providers 网关)+ `LiteLLMModel`(LiteLLM 接 100+ LLM,`README:99-103`)+ `OpenAIModel`(OpenAI-compatible)
  **支持 local transformers / ollama / vLLM** + 远程 HF Hub / OpenAI / Anthropic / Together / OpenRouter / DeepSeek 等
  `CODEAGENT_RESPONSE_FORMAT` 常量 — code agent 专用 response 格式
  Model-agnostic 是 README slogan

channels: |
  [verified-from-source]
  **`cli.py`** (294 行)— CLI 入口
  **`gradio_ui.py`** (464 行)— Gradio Web UI **first-class channel**(对照其他 harness gradio 是可选)
  **`vision_web_browser.py`** (247 行)— vision web browse 一等 channel
  **HF Hub `agent.push_to_hub` / `from_hub`**(README:71-76)— **agent 分享渠道**借力 HF ecosystem

persistence: |
  [verified-from-source]
  **HF Hub agent 持久化路线**(`agent.push_to_hub("user/agent_name")` + `from_hub`)— **业内独家借力 HF model registry**
  **无本地 session 持久化**(对照 [[codex]] rollout JSONL+SQLite / [[claude-code]] persistence)— smolagents 是 _"无状态短任务"_ 假设

cancellation: |
  ? 推测靠 `MAX_EXECUTION_TIME_SECONDS=30` timeout decorator(`local_python_executor.py:60, 1665`)+ agent loop step 级中断未深查

skillforge_takeaway: |
  ## 可借鉴

  ### 🔴 R-SMOL-1 P1 — `executor_type: Literal[...]` 抽象给 SkillForge sandbox 选型(1 周)
  smolagents `agents.py:1535` `executor_type: Literal["local","blaxel","e2b","modal","docker","wasm"]` 6 选项 + `create_python_executor()` factory 模式是 production sandbox 抽象模板。**比 [[hermes-agent]] `tools/environments/` 7 backends string SPI 更强类型化**。
  SkillForge 应该:`executor_type: Literal["jvm-native","docker","e2b","gvisor","modal"]`(对应 [[../concept/sandbox-isolation]] §10 5 场景),跟 R-RF-2 gVisor 借鉴 + R-PROTO-* MCP 联动。

  ### 🟡 R-SMOL-2 P2 — Code Mode Python 路线 POC 参考(2-4 周技术评估)
  smolagents 是 wiki 内 verified 第 2 家 code-mode(对照 [[codex]] V8 isolate JS 路线)。SkillForge JVM-based 做 _"LLM 写 Python"_ 不现实(需嵌 Python runtime 大工程),**真正方向是 _"LLM 写 Java/Kotlin 经 GraalVM polyglot"_**。但 smolagents `LocalPythonExecutor` AST interpreter 思路(白名单 builtin + 黑名单 module + 反射防御 + 4 硬限额)**可以直接搬到 JVM 上写 Java AST interpreter**。Code Mode 是 P2 决策,不必跟。

  ### 🟡 R-SMOL-3 P2 — `CallbackRegistry` hooks-by-step-type 第 3 种 hooks 哲学(2 周)
  smolagents `memory.py:280` 按 step 类型注册 callback,不是事件命名(CC 27 PreToolUse/PostToolUse)也不是 lifecycle(Hermes 16)。**第 3 种 hooks 设计哲学**。
  优势:**新加 step type 自动有 hook 槽位**,减少 hook 命名爆炸。
  SkillForge 现状若是 lifecycle-based,可考虑混合模式 — lifecycle hooks(PreToolUse 等)+ step-type hooks(OnActionStep / OnPlanningStep)双 surface。

  ### 🟡 R-SMOL-4 P2 — `FinalAnswerException` 跨 local/remote 一致 control flow(0.5-1 周)
  `local_python_executor.py:1572 FinalAnswerException(BaseException)` + `remote_executors.py:71 FINAL_ANSWER_EXCEPTION = "FinalAnswerException"` 常量 — **Python exception 跨进程边界保持 control flow 一致**。
  SkillForge 多 sandbox 路线下,**统一 final_answer 信号机制**(exception / sentinel / RPC code)是 elegant 抽象。

  ### 🟢 R-SMOL-5 P3 — `AgentMemory.return_full_code()` replay 思路(可选)
  `memory.py:273` 拼接所有 code_action 成单 Python script。**只对 code-mode 有用**(JSON tool_use 拼不出来),SkillForge 当前非 code-mode 不必。

  ## SkillForge 现状对照

  - **Hooks 哲学**:? 未 verify SkillForge hooks 是 lifecycle / event / step-type
  - **Sandbox 抽象**:? 未 verify executor_type Literal 类型化程度
  - **MCP `structured_output`**:? 待 verify SkillForge MCP client 是否走 outputSchema(对齐 smolagents v1.25 跟随姿态)

  ## 风险 / 不抄

  - **AST interpreter 路线 JVM 不可行** — 嵌 Python 跨 JVM 大工程,**不抄 LocalPythonExecutor**,如果做 Code Mode 走 V8(参考 Codex)或 GraalVM polyglot Java AST
  - **`managed_agents` + remote executor 互斥** 是 smolagents 已知缺口(`agents.py:1608` raise) — SkillForge 多 agent + 远程 sandbox 必须设计兼容,**这是 smolagents 反例,SkillForge 不能犯**
  - **Context compaction 缺位** — smolagents 极简到没 compaction 适合短任务,SkillForge 长任务必须有(借鉴 CC 5 级 compaction / Codex 3 套)

learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)
  - **Code-mode 路线 _"理念旗手"_** — smolagents slogan _"agents that think in code"_,**业内最早公开倡导 code-mode 范式**(2024-12 launch blog post)。**对照 [[codex]] Code Mode 是 _"工程实现旗手"_**(V8 + 100+ crates),smolagents 是 _"概念推广 + Python AST interpreter 极简实现"_
  - **业内罕见 AST interpreter 进程内 Python sandbox** — 不是 subprocess + venv + restrict,而是 `ast.parse` 后 recursive `evaluate_ast` 节点级解释。**0 cold start + 强 syscall-level 防御**(白名单 + 黑名单 + 反射防御)
  - **6 executor `executor_type: Literal[...]` unified sandbox 抽象** — production 灵活度业内最高之一(Hermes 7 backends 同思路但 string SPI 弱类型)
  - **HF ecosystem 一等公民** — `push_to_hub / from_hub / from_space` 是 channel + persistence 双重独家路线,借力 HF model registry
  - **极致 minimalism** — 19 文件 ~13K LOC,核心 `agents.py` 自称 _"~1000 lines"_(实际 1814),抽象保留 minimalism

  ## 优势(strengths,横向比谁做得对)
  - **`FinalAnswerException` 跨 local/remote 一致 control flow**(`local_python_executor.py:1572` + `remote_executors.py:71`)— Python exception 跨进程边界保持 control flow,**elegant 跨边界抽象**
  - **AST interpreter 多道防御工程化诚实**:进程内本身不安全,**承认并叠加 4 道防御**(白名单 builtins + 黑名单 modules + 黑名单 functions + 反射防御)+ 4 硬限额 — 比 _"假装进程内 = 安全"_ 路线诚实
  - **MCP `structured_output` v1.25 默认 True 主动跟随姿态**(`mcp_client.py:91-100`)— **协议演化对接证据**,跟 MCP spec 2025-06-18 outputSchema 显式对齐,不靠 backward compat 拖延
  - **`additional_authorized_imports = ["*"]` warning**(`agents.py:1578-1582`)— 用户主动放权时 logger.log 提醒,**安全姿态诚实**

  ## 哪些点做得好(what_done_well,elegant 设计)
  - **`CallbackRegistry` hooks-by-step-type 第 3 种 hooks 哲学** — 跟 CC event-based / Hermes lifecycle-based 都不同,**新加 step type 自动有 hook 槽位**,设计 elegant
  - **`AgentMemory.return_full_code()` 拼接所有 code_action 成单 script**(`memory.py:273`)— code-mode 独有 replay/debug 技巧,**JSON tool_use harness 拼不出来**
  - **`code_block_tags` 3 选项**(`("<code>","</code>")` default + `("```python","```")` markdown + 自定义 tuple)— 适配不同 LLM 风格,体现 _"协议级灵活性"_
  - **`use_structured_outputs_internally` v1.17.0+ 双 yaml prompt template 动态切换**(`code_agent.yaml` / `structured_code_agent.yaml`)— 同 agent 两种 prompt mode 切换
  - **承认 `managed_agents + remote executor` 互斥**(`agents.py:1608` raise Exception)— **工程诚实**,不偷偷凑合,显式 raise 让用户知道边界
---

# smolagents

## 一句话定位

HuggingFace 出品 _"agents that think in code"_ Python framework,**业内 code-mode 路线理念旗手**(对照 [[codex]] 工程实现旗手),核心 `CodeAgent` 让 LLM 写 Python 代码而非 JSON tool_use,**6 executor 选项**(local AST interpreter + Blaxel/E2B/Modal/Docker/WASM)+ MCP/LangChain/HF Hub Space tool 适配器,极致 minimalism(19 文件 ~13K LOC)。

## 起源与定位

**HuggingFace** 2024-12 发布 launch blog post([huggingface.co/blog/smolagents](https://huggingface.co/blog/smolagents))。围绕 _"agents that think in code"_ 概念推广 code-mode 范式,**业内最早公开倡导**。

跟竞品对照:
- vs **[[codex]] Code Mode**:smolagents Python AST interpreter 进程内 / Codex V8 isolate JavaScript 进程内 — **同范式不同语言不同 sandbox 路线**
- vs **[[crewai]] / [[ag2]] / [[metagpt]]**:这些是 multi-agent framework,smolagents 是 single-agent code-mode 主打
- vs **[[opencode]] / [[claude-code]]**:这些是 coding agent harness(用 agent 写代码),smolagents 是 _"agent 思考时用代码"_(用代码作为 reasoning + tool calling 媒介)

## 核心架构

```
        ┌────────── MultiStepAgent (ABC, agents.py:268) ──────────┐
        │                                                          │
        ▼                                                          ▼
  ToolCallingAgent (:1215)                            CodeAgent (:1505) ⭐
    JSON tool_use 路线                                  code-mode 路线
                                                              │
                                              ┌───────────────┴───────────────┐
                                              │ executor_type: Literal[       │
                                              │   "local"  → LocalPythonExecutor (AST 解释器) │
                                              │   "blaxel" → BlaxelExecutor                  │
                                              │   "e2b"    → E2BExecutor (Firecracker)       │
                                              │   "modal"  → ModalExecutor                   │
                                              │   "docker" → DockerExecutor                  │
                                              │   "wasm"   → WasmExecutor (Pyodide+Deno)     │
                                              │ ]                              │
                                              └────────────────────────────────┘
                                                              │
                                                              ▼
                                              FinalAnswerException 跨 local/remote
                                              一致 control flow (协议常量)

  Memory:    5 step types + AgentMemory + CallbackRegistry hooks-by-step-type
  Provider:  InferenceClient / LiteLLM / OpenAI + transformers/ollama/vLLM local
  Channels:  CLI + Gradio UI + vision_web_browser + HF Hub push/from_hub
  MCP:       mcp_client.py wrap mcpadapt (stdio/streamable-http/sse + structured_output v1.25)
```

## 13 维度详解

frontmatter 已 verified 详细,精华:
- **agent_loop**:`MultiStepAgent` ABC + 2 子类(ToolCallingAgent JSON / CodeAgent code-mode)
- **safety.sandbox**:**6 executor 选项**(`executor_type: Literal` 强类型化)
- **AST interpreter 多道防御**:白名单 + 黑名单 + 反射防御 + 4 硬限额(MAX_OPERATIONS=10M / WHILE=1M / TIME=30s / OUTPUT=50K)
- **MCP 集成**:`structured_output FutureWarning v1.25` 显式追 spec 2025-06-18

## 与 SkillForge 的对照

详见 frontmatter `skillforge_takeaway` 5 条 R-SMOL-N。最高 ROI 是 **R-SMOL-1 `executor_type: Literal[...]` 抽象**(1 周),跟 [[../concept/sandbox-isolation]] §10 SkillForge 5 场景 sandbox 选型直接呼应。

## 待回答的问题

- [ ] `eval_loop` 维度 — 读 `examples/` + `tests/` 补
- [ ] `cancellation` 维度 — grep interrupt / cancel
- [ ] `secret_scan` 维度 — 未深查
- [ ] `managed_agents` 完整实现细节(hierarchical multi-agent + ManagedAgentPromptTemplate task/report 双 prompt 段)
- [ ] **MCP `structured_output` 默认是否真在 v1.26 改 True**(commit 025b6ad @ 2026-05-14,值得 git log 看 changelog)
- [ ] 跟 [[concept/code-mode-orchestration]] 第 2 家 verified anchor 联动后,**code-mode synthesis 升级是否需要单独 deep 页?**(`deep/smolagents-ast-interpreter` 候选)

## 引用链路

### 父概念
- [[../concept/code-mode-orchestration]] — smolagents 是 wiki 内第 2 家 verified code-mode anchor(对照 [[codex]] V8 isolate)
- [[../concept/sandbox-isolation]] — smolagents `executor_type` 6 选项是 wiki 内第一家 verified 用 E2B/Docker/Modal 的 harness

### 相关 concept
- [[../concept/agent-protocols]] §2 MCP — smolagents `structured_output` v1.25 是协议演化对接证据
- [[../concept/agent-harness]] — 13 维 cross-harness 对照
- [[../concept/hooks-design]] — smolagents `CallbackRegistry` 是第 3 种 hooks 哲学(对照 CC event / Hermes lifecycle)

### 对照 harness
- [[codex]] — Code Mode V8 isolate JS / smolagents Python AST 双路线对照
- [[hermes-agent]] — 7 backends sandbox SPI / smolagents 6 executor Literal 强类型对照
- [[claude-code]] — 27 hooks event-based / smolagents 5 hooks step-type 哲学对照
