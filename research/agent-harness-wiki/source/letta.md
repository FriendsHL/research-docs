---
type: source
name: letta
origin: "/Users/youren/myspace/letta/ (Letta AI, formerly MemGPT, Apache 2.0)"
ingested_at: 2026-05-16
ingested_commit: 1131535
ingested_commit_date: 2026-05-14
ingested_version_label: "letta 0.16.8(`pyproject.toml`,Python 3.12)"
contributed_to:
  - eval/letta.md
  - concept/agent-memory.md
---

# Source — Letta(formerly MemGPT)

> **Berkeley + Letta AI 出品**(Apache 2.0)— "**Build AI with advanced memory that can learn and self-improve over time**"。**MemGPT paper 学术出身**(arxiv.org/abs/2310.08560)+ **Agent OS + virtual context + recall management**。
>
> Letta Code(CLI,`npm install -g @letta-ai/letta-code`)+ Letta API(SaaS)双产品 + Python/TypeScript SDK + 完整 self-host(`compose.yaml` + alembic + nginx + sandbox + clickhouse + otel)。

## 仓库结构概览

```
letta/
├── letta/                            # ⭐ Python 主包
│   ├── agent.py                      # Agent 基类(`from typing ABC, abstractmethod`)
│   ├── agents/                       # Agent 实现
│   ├── adapters/                     # Adapter pattern
│   ├── interfaces/                   # Interface
│   ├── personas/                     # ⭐ MemGPT 论文 persona 系统
│   ├── humans/                       # ⭐ User profile / Human personas
│   ├── prompts/                      # System prompt 模板
│   ├── plugins/                      # Plugin 系统
│   ├── functions/                    # Function calling + Composio 集成
│   ├── llm_api/                      # LLM API client
│   ├── local_llm/                    # 本地 LLM(Inner Thoughts KWARG)
│   ├── data_sources/                 # RAG data sources
│   ├── groups/                       # ⭐ Multi-agent group
│   ├── schemas/ + serialize_schemas/ # Pydantic schemas
│   ├── services/                     # ⭐ 后端 service(archive / block / conversation / file_processor / lettuce / ...)
│   ├── orm/                          # ORM(SQLAlchemy?)
│   ├── server/                       # FastAPI server
│   ├── client/                       # Client SDK
│   ├── cli/                          # CLI
│   ├── jobs/                         # Background jobs
│   ├── monitoring/                   # Monitoring
│   ├── otel/                         # OpenTelemetry
│   ├── model_aliases.py
│   ├── model_specs/
│   ├── helpers/
│   ├── exceptions/ + errors.py
│   ├── streaming_interface.py + streaming_utils.py
│   ├── interface.py
│   ├── log.py + log_context.py
│   ├── config.py + config_file.py + settings.py
│   ├── constants.py
│   ├── templates/
│   ├── types/
│   ├── utils.py + validators.py
│   ├── system.py + main.py
│   └── openai_backcompat/            # OpenAI backward compat
├── alembic/ + alembic.ini             # ⭐ DB migrations
├── db/                                # DB schema
├── init.sql
├── compose.yaml + dev-compose.yaml + development.compose.yml + docker-compose-vllm.yaml
├── Dockerfile + nginx.conf            # 完整 self-host stack
├── certs/                             # TLS certs
├── sandbox/                           # ⭐ Sandbox(对照 Codex sandboxing)
├── otel/                              # OpenTelemetry config
├── fern/                              # ⭐ Fern API SDK 生成(对照 Langfuse `fern/`)
├── examples/
├── scripts/
├── assets/
├── AI_POLICY.md + PRIVACY.md + TERMS.md + SECURITY.md
├── CITATION.cff                       # 学术引用
├── README.md
├── pyproject.toml(version = "0.16.8", py 3.12)
└── package-lock.json + project.json
```

## 关键 file:line 锚点

### MemGPT 论文核心概念(README + agent.py imports)

**论文**:Packer et al., 2023, "MemGPT: Towards LLMs as Operating Systems"(`arxiv.org/abs/2310.08560`)

**核心 5 概念**:
1. **Virtual context**:context window 不够时,把不常用 memory swap 到 archival storage(类 OS virtual memory)
2. **Memory hierarchy**:核心 prompt(in-context)+ working context + recall storage + archival storage
3. **Inner Thoughts / Inner Monologue**(`agent.py:32 INNER_THOUGHTS_KWARG`)— LLM 内部思考分开存
4. **Function calling 主动 memory ops**:`core_memory_append` / `core_memory_replace` / `conversation_search` / `archival_memory_insert` / `archival_memory_search`
5. **Heartbeat**(`agent.py:18 FIRST_MESSAGE_ATTEMPTS, FUNC_FAILED_HEARTBEAT_MESSAGE, REQ_HEARTBEAT_MESSAGE`)— Agent 主动心跳触发后台思考

### Agent.py 关键 import(`letta/agent.py:1-40`)

```python
from letta.constants import (
    CLI_WARNING_PREFIX,
    COMPOSIO_ENTITY_ENV_VAR_KEY,
    ERROR_MESSAGE_PREFIX,
    FIRST_MESSAGE_ATTEMPTS,
    FUNC_FAILED_HEARTBEAT_MESSAGE,
    LETTA_CORE_TOOL_MODULE_NAME,
    LETTA_MULTI_AGENT_TOOL_MODULE_NAME,        # ⭐ Multi-agent tool 一等公民
    LLM_MAX_TOKENS,
    READ_ONLY_BLOCK_EDIT_ERROR,                # ⭐ Read-only block 概念
    REQ_HEARTBEAT_MESSAGE,                     # ⭐ Heartbeat 一等公民
    SEND_MESSAGE_TOOL_NAME,
)
from letta.errors import ContextWindowExceededError
from letta.helpers import ToolRulesSolver       # ⭐ Tool rules solver
from letta.helpers.composio_helpers import (
    execute_composio_action,                   # ⭐ Composio 集成(business tool ecosystem)
    generate_composio_action_from_func_name
)
from letta.interface import AgentInterface
from letta.llm_api.helpers import (
    calculate_summarizer_cutoff,                # ⭐ Summarizer cutoff(虚拟 context 切割)
    get_token_counts_for_messages,
    is_context_overflow_error                   # ⭐ Context overflow 探测
)
from letta.local_llm.constants import INNER_THOUGHTS_KWARG  # ⭐ MemGPT Inner Thoughts
```

### Services(`letta/services/`,后端核心)

**18+ service module**:
- **`archive_manager.py`** — ⭐ Archive(MemGPT 论文核心:archival_storage)
- **`block_manager.py` + `block_manager_git.py`** — ⭐ Memory block 管理(`git` 版本控制 block!)
- **`context_window_calculator/`** — Context window 计算(MemGPT virtual context 决策)
- **`conversation_manager.py`** — 对话管理
- **`agent_manager.py` + `agent_generate_completion_manager.py` + `agent_serialization_manager.py`**
- **`file_manager.py` + `file_processor/`** — File 处理(`files_agents_manager.py`)
- **`group_manager.py`** — ⭐ Multi-agent group(对照 SkillForge Team)
- **`identity_manager.py`** — Identity(对照 [[../harness/codex]] `agent-identity/` crate)
- **`job_manager.py`** — Background job
- **`credit_verification_service.py`** — 信用验证(用量计费)
- **`lettuce/`** — Lettuce 子系统(待 verify,可能是 LLM utilities)
- **`clickhouse_otel_traces.py` + `clickhouse_provider_traces.py`** — ⭐ ClickHouse OTEL trace(对照 Langfuse / Opik / Phoenix)

### Block Manager 用 Git 版本控制(`services/block_manager_git.py`)

**业内独家**:**Memory block 用 git 版本控制**!
- `block_manager.py` 主类
- `block_manager_git.py` git 版本管理
- 跟 SkillForge MEMORY-DREAM-CONSOLIDATION `archived_reason` 不同 — Letta **存所有版本(git history)**,**SkillForge 显式 archive**

### Read-only block(`constants:READ_ONLY_BLOCK_EDIT_ERROR`)

**业内独家**:某些 memory block 是**只读 reference**,不允许 agent 修改(如 system instructions / 用户 hardcoded preferences)。

### Heartbeat 机制(`constants:REQ_HEARTBEAT_MESSAGE`)

**MemGPT 论文核心**:Agent 完成 tool call 后**主动心跳**触发下一步,让 agent 可在用户没说话时自主继续(对照传统 agent 等用户输入)。

`FUNC_FAILED_HEARTBEAT_MESSAGE` — tool 失败时的 heartbeat message。

### Multi-agent tool(`constants:LETTA_MULTI_AGENT_TOOL_MODULE_NAME`)

**Multi-agent 一等公民**(对照 [[../harness/codex]] `multi_agents/` v1+v2 / [[../harness/claude-code]] AgentTool / [[../harness/opencode]] 2 builtin + `@general` subagent / [[../harness/hermes-agent]] delegate)— Letta 也是 multi-agent capable。

`letta/groups/` 子模块 + `services/group_manager.py` 实现。

### `lettuce/` 子系统

待 verify(从命名推测可能是 letta + utility 双关名,内部 utility / LLM helper)。

### Persona / Human(`letta/personas/` + `letta/humans/`)

MemGPT 论文核心:Agent 跟 user 都有 **persona profile**,作为 system instruction 的可学 block。

### Composio 集成(`functions/composio_helpers.py`)

Composio(business tool marketplace)集成 — Letta agent 可调用 Composio 的 100+ 业务 tool(Slack / Gmail / GitHub / Linear / 等)。

### Fern SDK 生成(`fern/`)

OpenAPI → Python/TS SDK 自动生成(对照 [[../eval/langfuse]] `fern/`,业内同模式)。

### Sandbox(`sandbox/`)

Letta 自家 sandbox 实现(对照 [[../harness/codex]] `sandboxing/` + [[../eval/opik]] `apps/opik-sandbox-executor-python/`)。

### ClickHouse + OpenTelemetry

`services/clickhouse_otel_traces.py` + `services/clickhouse_provider_traces.py` — Letta 自家用 ClickHouse 存 OTEL trace(对照 [[../eval/langfuse]] / [[../eval/opik]] / [[../eval/phoenix]] 都用 ClickHouse)。

### `letta-code` CLI(独立 npm package)

`@letta-ai/letta-code` — CLI 工具,运行本地 agent + 终端交互(对照 [[../harness/claude-code]] / [[../harness/opencode]])。**Letta 同时是 memory framework 跟 coding agent**(meta!)。

## 与 wiki 现有

### 印证
- **ClickHouse OTEL trace** 跟 Langfuse / Opik / Phoenix 同选
- **Fern SDK 生成** 跟 Langfuse 同模式
- **Composio 集成** 业内主流 business tool
- **Sandbox 一等公民** 跟 Codex / Opik 一致

### 矛盾
- **Letta 是 memory framework + coding agent + Agent OS 三合一**,跟 wiki 当前分类**不太适应**:
  - mem0 = pure memory framework(只 memory layer)
  - SkillForge = platform(eval + memory + skill)
  - Letta = framework + agent + CLI(`letta-code`)三层
  - 这意味着 Letta 不该归 `eval/` 而是 `harness/` + `memory/`?**待用户决策**

### 新增
- **`eval/letta.md`**(暂归 eval/,后续可能重分类)
- **新建 `concept/agent-memory.md`**:加 Letta 的 "**Agent OS + virtual context + memory hierarchy**" 路线 vs mem0 "**Single-pass ADD-only**" vs SkillForge "**MEMORY-LLM-SYNTHESIS + DREAM-CONSOLIDATION**"

## 事实漂移 / 注意点

- **letta 0.16.8** 跟 mem0 2.0.2 同档(都迭代快)
- **Python 3.12 强制**(`pyproject.toml:target-version = "py312"`)— 新 Python 版本
- **`letta-code` CLI 是独立 npm package**(`letta-ai/letta-code`),本仓库不含
- **`@letta-ai/letta-client` npm + `letta-client` pip** — SDK 双 package
- **`letta-code` 支持 skills + subagents + 内置 pre-built skills/subagents**(README)— 跟 [[../harness/claude-code]] skill 体系类似
- **`block_manager_git.py` git-based memory block versioning** — 业内独家
- **READ_ONLY_BLOCK + heartbeat + inner thoughts** — MemGPT 论文 3 大原创概念
- **`lettuce/` 子系统命名俏皮**(letta + lettuce / utility 双关)
- **`monitoring/` + `otel/` 独立子模块** — 自家 observability
- **`credit_verification_service.py`** — 用量计费(Letta API SaaS 商业模式)
- **`compose.yaml + dev-compose.yaml + development.compose.yml + docker-compose-vllm.yaml`** 4 个 compose 配置 — 多场景部署
