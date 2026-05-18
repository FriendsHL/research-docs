---
type: source
name: smolagents
origin: /Users/youren/myspace/smolagents (HuggingFace/smolagents @ commit 025b6ad, 2026-05-14)
version: 1.26.0.dev0
ingested_at: 2026-05-18
contributed_to:
  - "[[../harness/smolagents]]"
  - "[[../concept/code-mode-orchestration]]"      # 第 2 家 verified anchor
  - "[[../concept/sandbox-isolation]]"            # E2B/Docker/Modal 首家 verified 集成
read_files:
  - "README.md (1-120)"
  - "AGENTS.md (full, 4 lines)"
  - "src/smolagents/agents.py (1-200, 1505-1640) + grep class boundaries"
  - "src/smolagents/local_python_executor.py (1-150, 1583-1700) + grep class/constants"
  - "src/smolagents/remote_executors.py (56-156) + grep class boundaries"
  - "src/smolagents/mcp_client.py (full, 171 lines)"
  - "src/smolagents/memory.py (214-300) + grep class boundaries"
  - "pyproject.toml (version line)"
---

# smolagents 源码摘要

## 核心论点

**HuggingFace smolagents 是 code-mode 路线 _"理念旗手"_** — 业内最早(2024-12 launch blog post)公开倡导 _"agents that think in code"_ 范式,**Python AST interpreter 进程内 sandbox** 是业内罕见路线(对照 [[../harness/codex]] V8 isolate JavaScript / E2B Firecracker / Cloudflare container 都不同)。19 文件 ~13K LOC 极致 minimalism。

## 关键事实

### 项目结构(已 verified)
- **License**:Apache 2.0
- **版本**:1.26.0.dev0 @ commit `025b6ad`(2026-05-14)
- **代码量**:`src/smolagents/` 19 Python 文件 ~13K LOC
- **大文件**:`models.py` 2102 / `agents.py` 1814 / `local_python_executor.py` 1768 / `remote_executors.py` 1428 / `tools.py` 1422
- **小但关键**:`mcp_client.py` 171 / `memory.py` 316

### 3 主类继承(已 verified)
- `MultiStepAgent` ABC (`agents.py:268`)
- `ToolCallingAgent` (`:1215`) — JSON tool_use 路线
- `CodeAgent` (`:1505`) — **code-mode 主角**,**executor_type Literal 6 选项**

### 6 sandbox executor 选项(已 verified `agents.py:1535`)
- `local` — `LocalPythonExecutor` AST interpreter(默认,进程内 0 cold start)
- `blaxel` — `BlaxelExecutor`
- `e2b` — `E2BExecutor`(Firecracker microVM)
- `modal` — `ModalExecutor`
- `docker` — `DockerExecutor`
- `wasm` — `WasmExecutor`(Pyodide + Deno WebAssembly)

### AST interpreter 多道防御(已 verified `local_python_executor.py`)
- **白名单 builtins**:`BASE_PYTHON_TOOLS` 40+ (`:74-127`)
- **黑名单 modules 9 个**:builtins/io/multiprocessing/os/pathlib/pty/shutil/socket/subprocess/sys (`:130`)
- **黑名单 functions**:compile/eval/exec/globals/locals/__import__/os.popen/... (`:143`)
- **反射防御**:`nodunder_getattr` 拒绝 `__*__` (`:68`)
- **4 硬限额**:`DEFAULT_MAX_LEN_OUTPUT=50000 / MAX_OPERATIONS=10M / MAX_WHILE_ITERATIONS=1M / MAX_EXECUTION_TIME_SECONDS=30`
- **`FinalAnswerException(BaseException)` (`:1572`)** elegant 跳出机制 + `RemotePythonExecutor.FINAL_ANSWER_EXCEPTION` 常量跨边界一致

### 5 Memory Step Types + CallbackRegistry
- `SystemPromptStep / TaskStep / ActionStep / PlanningStep / FinalAnswerStep` (`memory.py:51-210`)
- `AgentMemory` (`:214`) + `return_full_code()` (`:273`) 拼接 code_action 成单 script
- **`CallbackRegistry` (`:280`) hooks-by-step-type 业内独特**

### MCP 集成(已 verified `mcp_client.py` 171 行完整)
- wrap `mcpadapt.MCPAdapt + SmolAgentsAdapter`
- 支持 stdio + streamable-http + sse
- **`structured_output: bool` FutureWarning v1.25 默认改 True** — 协议演化对接

## 与现有 wiki 的关系

### 印证
- **[[../concept/code-mode-orchestration]]** 第 2 家 verified anchor(原本只 Codex 1 家)
- **[[../concept/sandbox-isolation]]** E2B/Docker/Modal 首家 verified 集成(原本 doc-only)
- **[[../concept/agent-protocols]]** §2 MCP `structured_output` v1.25 协议演化对接证据

### 新增
- **业内罕见 AST interpreter Python sandbox** 路线(白名单+黑名单+反射防御+4 硬限额)
- **`executor_type: Literal[...]` 比 string SPI 强类型化** (vs [[../harness/hermes-agent]] 7 backends)
- **`CallbackRegistry` hooks-by-step-type** 第 3 种 hooks 哲学(vs CC event-based / Hermes lifecycle-based)
- **HF Hub 持久化路线** (`agent.push_to_hub` / `from_hub`)
- **`FinalAnswerException` 跨 local/remote 一致 control flow**(协议常量级抽象)

### 矛盾 / 校准
- README 自夸 `agents.py "~1000 lines"` 实际 1814 行(允许的营销夸张)
- `managed_agents + remote executor` **互斥**(`agents.py:1608` raise) — multi-agent + 远程 sandbox 是 smolagents 已知缺口

## 待回答(下次 re-ingest 优先)

- `eval_loop` 维度 — 读 `examples/` + `tests/`
- `cancellation` 维度 — grep interrupt / cancel pattern
- `secret_scan` 维度 — 未深查
- `managed_agents + ManagedAgentPromptTemplate` 完整 hierarchical multi-agent 实现
- v1.26 MCP `structured_output` 默认是否真改 True(git log 看 changelog)
