---
type: source
name: langchain
origin: "/Users/youren/myspace/langchain/ (LangChain Inc, MIT)"
ingested_at: 2026-05-16
ingested_commit: 2259d29
ingested_commit_date: 2026-05-15
ingested_version_label: "langchain v1.3.1(`libs/langchain_v1/pyproject.toml`),langchain_core 独立 semver,monorepo"
contributed_to:
  - eval/langchain.md
  - concept/agent-eval.md
---

# Source — LangChain

> **业内规模最大 LLM framework + agent ecosystem**(MIT,LangChain Inc)。**v1.3.1**(2026 v1 重写),slogan "**The agent engineering platform**"。完整生态:LangChain framework + LangGraph + LangSmith(闭源 SaaS)+ Deep Agents + 100+ integrations。**LangSmith 是 eval/observability 闭源平台**(本仓库主要是 framework,LangSmith 商业 SaaS 不开源)。

## 仓库结构概览(`libs/` 8 大模块 monorepo)

```
langchain/
├── libs/
│   ├── core/                         # ⭐ langchain_core(基础抽象)
│   │   └── langchain_core/
│   │       ├── agents.py             # Agent 抽象
│   │       ├── callbacks/            # Callback system(observability hook)
│   │       ├── chat_history.py       # 对话历史
│   │       ├── document_loaders/     # 文档加载器
│   │       ├── documents/            # Document 实体
│   │       ├── embeddings/           # Embedding 抽象
│   │       ├── example_selectors/    # Few-shot example 选择
│   │       ├── indexing/             # 索引
│   │       ├── language_models/      # LLM / Chat model 抽象
│   │       ├── load/                 # 序列化
│   │       ├── messages/             # Message 类型
│   │       ├── output_parsers/       # 输出解析
│   │       ├── outputs/              # 输出类型
│   │       ├── prompts/              # ⭐ Prompt template
│   │       ├── prompt_values.py
│   │       ├── rate_limiters.py
│   │       ├── retrievers.py         # Retriever 抽象
│   │       ├── runnables/            # ⭐ Runnable interface(LangChain v0.1+ 核心抽象)
│   │       ├── stores.py
│   │       ├── structured_query.py
│   │       ├── tools/                # Tool 抽象
│   │       ├── tracers/              # Trace 抽象(LangSmith 集成层)
│   │       ├── vectorstores/         # Vector store 抽象
│   │       └── _security/            # 安全
│   ├── langchain/                    # ⭐ langchain_classic(v0.x 兼容层)
│   │   └── langchain_classic/        # 旧 API 兼容
│   ├── langchain_v1/                 # ⭐⭐ v1.3.1(2026 重写,新版主体)
│   │   └── langchain/                # 主模块
│   │       ├── agents/               # ⭐ Agent
│   │       ├── chat_models/          # Chat model
│   │       ├── embeddings/
│   │       ├── messages/
│   │       ├── rate_limiters/
│   │       └── tools/
│   ├── partners/                     # ⭐ 16+ provider 集成
│   │   ├── anthropic/
│   │   ├── chroma/                   # Chroma vector store
│   │   ├── deepseek/
│   │   ├── exa/                      # Exa search
│   │   ├── fireworks/
│   │   ├── groq/
│   │   ├── huggingface/
│   │   ├── mistralai/
│   │   ├── nomic/                    # Nomic embeddings
│   │   ├── ollama/
│   │   ├── openai/
│   │   ├── openrouter/
│   │   ├── perplexity/
│   │   ├── qdrant/                   # Qdrant vector store
│   │   └── xai/                      # xAI Grok
│   ├── text-splitters/               # Text splitting strategies
│   ├── model-profiles/               # 模型配置 profile
│   └── standard-tests/               # 标准测试套
├── AGENTS.md + CLAUDE.md             # LLM agent guide
├── CITATION.cff
└── README.md (英文)
```

**注意**:本仓库**不含 LangSmith / LangGraph 源码**(它们在 `langchain-ai/langsmith-sdk` + `langchain-ai/langgraph` 独立 repo,且 **LangSmith 是闭源 SaaS**,只 SDK 开源)。

## 关键 file:line 锚点

### v1.3.1(2026 v1 重写,主体)

**`libs/langchain_v1/pyproject.toml:version = "1.3.1"`**

LangChain v1 是 2026 重写,**重大变化**(README + Tips):
- 新主 entry:**`from langchain.chat_models import init_chat_model`**(`init_chat_model("openai:gpt-5.4")`)
- 用 `provider:model` 字符串风格,**业内独家**
- `langchain.invoke(...)` 风格
- **`Deep Agents`**(http://docs.langchain.com/oss/python/deepagents/)— 新 high-level package,内置 planning / subagents / file system 等 patterns
- **`LangGraph`**(`docs.langchain.com/oss/python/langgraph/overview`)— 低层 agent orchestration framework

### `langchain_core/runnables/` Runnable Interface(核心抽象)

LangChain v0.1+ 引入的 **Runnable Protocol** — 所有组件(model / prompt / parser / retriever)都实现 `Runnable` 接口,可用 `|` 操作符组合(Unix pipe 风格):

```python
chain = prompt | model | output_parser
result = chain.invoke({"question": "..."})
```

**业内标志性抽象**,被 LangChain v1 保留为核心。

### `langchain_core/callbacks/` Callback System

LangChain 业内最早把 callback 做成 framework 一等公民:
- 所有 LLM / chain / agent / tool 调用走 callback
- LangSmith / Langfuse / Phoenix 都通过 callback handler 接 LangChain
- 跟 [[../eval/langfuse]] 集成方式: `LangfuseCallbackHandler`
- 跟 [[../eval/opik]] 集成方式: callback registration
- 跟 [[phoenix]] 集成方式: OpenInference callback

### `langchain_core/tracers/` Tracer

Tracer 抽象 — LangSmith trace 接入点。OpenInference / OTel ingestion 通常通过 callback + tracer 双轨。

### `partners/` 16+ Provider 集成(`libs/partners/`)

```
anthropic / chroma / deepseek / exa / fireworks /
groq / huggingface / mistralai / nomic / ollama /
openai / openrouter / perplexity / qdrant / xai
```

**每个 partner 是独立 PyPI package**(`langchain-anthropic` / `langchain-openai` / 等),用户按需 pip install。

跟 [[../harness/codex]] 100+ crate / [[../eval/opik]] 6 app 拆分粒度对照,**LangChain 是"每个 partner 1 package"路线**。

### LangChain Ecosystem 5 大产品(README "LangChain ecosystem")

1. **LangChain framework**(本仓库)— 标准接口 + integrations
2. **Deep Agents** — Build agents with planning / subagents / file systems(高层包)
3. **LangGraph** — Low-level agent orchestration(独立 repo)
4. **LangSmith** — Agent evals / observability / debugging(**闭源 SaaS**)
5. **LangSmith Deployment** — Deploy 长任务 stateful agent

### v0 vs v1 双轨

- **`libs/langchain/langchain_classic/`** — v0.x 兼容
- **`libs/langchain_v1/`** — v1.x 主体

LangChain 工程文化:**`_classic` 后缀给旧 API**(对照 [[../harness/codex]] `execpolicy-legacy/`,业内 deprecate 模式)。

### Standard Tests(`libs/standard-tests/`)

所有 partner package 必须通过的 standard test suite — **业内独家强制集成测试**(对照 SkillForge SCENARIO eval)。

### Text Splitters(`libs/text-splitters/`)

文本切分策略集 — RAG 场景必备(对照 [[ragas]] RAG 三角)。

### Model Profiles(`libs/model-profiles/`)

模型配置 profile(context window / cost / capability)— 类 [[../eval/langfuse]] `default-model-prices.json` 集中维护。

### `Deep Agents`(高层包,2026 推)

README "TIP":
> "Just getting started? Check out **Deep Agents** — a higher-level package built on LangChain for agents that have built-in capabilites for common usage patterns such as planning, subagents, file system usage, and more."

跟 [[../harness/opencode]] / [[../harness/codex]] / [[../harness/claude-code]] 同方向(planning / subagents / file system 是 coding agent 标配),**LangChain 用 Deep Agents 给业内标杆能力**(类 SkillForge 自家 SubAgent + Team 路线)。

## 与 wiki 现有

### 印证
- **Callback / Tracer 抽象** 是业内事实标准 — 所有 LLM observability(Langfuse / Opik / Phoenix / Helicone)都通过 LangChain callback 集成
- **Runnable + `|` pipe 操作符** 跟 DSPy `Module` / Promptfoo `assertion` 都是 composable abstraction
- **Partner package 拆分** 跟 Codex 100+ crate / Opik 6 app 同思路(每个 provider 独立 package)
- **`provider:model` 字符串**(`init_chat_model("openai:gpt-5.4")`)跟 [[../harness/opencode]] / [[../eval/opik]] LiteLLM 字符串风格一致

### 矛盾
- **LangChain v1 vs v0 兼容性问题** — 业内 community 集成 LangChain v0 API 多年,v1 重写带来兼容性挑战
- **LangSmith 闭源 SaaS** vs Langfuse / Opik MIT/Apache 完全 OSS — license 矛盾,SkillForge 不该绑死 LangSmith
- **LangChain 抽象层级偏多**(Runnable / Chain / Agent / LangGraph / Deep Agents)— 用户学习曲线陡峭

### 新增
- **`eval/langchain.md`**(新建)— 跟 LangSmith 一起作为生态对照
- **扩 [[../concept/agent-eval]]**:加 LangChain Callback / Tracer 抽象 — 业内事实标准

## 事实漂移 / 注意点

- **v1.3.1 是 2026 重写**,跟 v0.x API 不兼容,生态适配中
- **`langchain_v1/`** 是新主体,**`langchain_classic/`** 是 v0 兼容层
- **LangSmith 不在本仓库** — `langchain-ai/langsmith-sdk` 独立(SDK MIT),但**主 SaaS 闭源**
- **LangGraph 不在本仓库** — `langchain-ai/langgraph` 独立(OSS)
- **Deep Agents 不在本仓库** — `langchain-ai/deepagents` 独立(OSS)
- **完整 ecosystem 跨 5+ repos**:langchain / langgraph / langsmith-sdk / deepagents / langchainjs(JS 版)+ 多个 partner packages
- **`init_chat_model("provider:model")` 字符串解析** 是 v1 主入口(README Quick Start),业内独家这种 string-based provider 路由
- **`Standard Tests`**(`libs/standard-tests/`)是业内独家做 partner 包标准测试 enforcement
