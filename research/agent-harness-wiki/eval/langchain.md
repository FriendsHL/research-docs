---
type: eval-framework
name: langchain
status: active
language: python + typescript

# === 版本时效 ===
last_ingested_version: "langchain v1.3.1 @ commit 2259d29 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - LangSmith SaaS 闭源,具体 eval/observability/归因 feature 走 community-knowledge
  - LangGraph 独立 repo(`langchain-ai/langgraph`)未 ingest
  - Deep Agents 独立 repo(`langchain-ai/deepagents`)未 ingest

sources:
  - "[[../source/langchain]]"

# === 9 维度 ===
target: |
  **业内规模最大 LLM framework + agent ecosystem**(MIT framework + LangSmith 闭源 SaaS)。slogan **"The agent engineering platform"**。

  **本 ingest 主要覆盖 LangChain framework**(`libs/langchain_v1/` + `libs/core/`),**LangSmith eval/observability 闭源**只能 [doc-only]。

  跟其他 platform/framework 对照:
  - [[../eval/langfuse]] / [[../eval/opik]] = OSS platform(LangChain 通过 CallbackHandler 集成)
  - [[phoenix]] = OSS observability(LangChain 通过 OpenInference callback 集成)
  - [[dspy]] = framework(竞争对手,prompt-as-program)
  - **LangChain v1 = framework + agent orchestration + 闭源 SaaS 生态**(规模最大)

  跟 SkillForge 关系:**SkillForge 是 server-side platform,LangChain 是 Python framework**,**层不同**。SkillForge 用户可能在自己 agent code 里用 LangChain,SkillForge 通过 OTel / callback 收集这些 trace。
  [verified-from-source(framework 部分)+ doc-only(LangSmith 部分)]

dataset: |
  - **`langchain_core` Document 抽象**(`langchain_core/documents/`)
  - **No platform-level Dataset entity**(LangSmith 提供,但闭源)
  - 主要走 HuggingFace `datasets` 集成
  - **Standard Tests**(`libs/standard-tests/`)— 业内独家强制 partner package 标准测试
  [verified-from-source]

scorer: |
  - **`langchain_core/callbacks/`** Callback system — 所有 LLM observability 集成的 base layer
  - **`langchain_core/tracers/`** Tracer 抽象 — LangSmith trace 接入点
  - **没有内置 metric**(对照 [[ragas]] 27 / [[promptfoo]] 50 / [[phoenix]] 11)
  - LangSmith 闭源 SaaS 提供 evaluators / dataset / experiments,但**SDK 闭源 detail 未深读**

  ### LangSmith 集成方式(业内已知)
  - `LangSmith` Python / JS SDK(独立 repo `langchain-ai/langsmith-sdk`)
  - 通过 `LangChainTracer` callback 自动上报 trace
  - LangSmith dashboard 看 trace / dataset / experiment
  - **LangSmith 是 LangChain ecosystem 的 production 闭环**
  [doc-only(LangSmith 部分)]

reproducibility: |
  - **`Standard Tests`**(`libs/standard-tests/`)— 业内独家:所有 partner package 必须通过同一套 standard test suite
  - **v0 vs v1 双轨**(`langchain_classic/` + `langchain_v1/`)— deprecation 模式
  - **`model-profiles/`**(`libs/model-profiles/`)模型 profile 集中(context window / cost / capability)
  - **`partners/` 16+ 独立 PyPI package** — 按需 install,版本隔离
  [verified-from-source]

coverage: |
  ### 5 大支柱(README "LangChain ecosystem")
  1. **LangChain framework**(本仓库,OSS)
  2. **Deep Agents** — Planning / Subagents / File System 高层包(独立 repo)
  3. **LangGraph** — 低层 agent orchestration(独立 repo)
  4. **LangSmith** — Agent evals / observability / debugging(**闭源 SaaS**)
  5. **LangSmith Deployment** — 长任务 stateful agent 部署

  ### 核心抽象(`langchain_core/`)
  - **Runnable Protocol**(`runnables/`)— 业内标志性 `prompt | model | output_parser` pipe 组合
  - **Callback System**(`callbacks/`)— 业内事实标准 observability 集成
  - **Document / Embedding / VectorStore / Retriever**(RAG 核心抽象)
  - **Tool 抽象**(`tools/`)
  - **Message types**(`messages/`)
  - **Output Parsers**(`output_parsers/`)
  - **Prompt Template**(`prompts/` + `prompt_values.py`)

  ### v1 主体(`libs/langchain_v1/`)
  - **`init_chat_model("openai:gpt-5.4")`** — `provider:model` 字符串风格(业内独家)
  - **`langchain/agents/`** — 新 Agent module
  - **`langchain/chat_models/` + `embeddings/` + `tools/`**

  ### 不覆盖(本 ingest)
  - **LangSmith eval / dataset / experiment 闭源细节**
  - **LangGraph orchestration**(独立 repo,未 ingest)
  - **Deep Agents planning / subagents 实现**(独立 repo,未 ingest)
  - **LangChain.js**(独立 repo)
  [verified-from-source(framework)+ doc-only(SaaS)]

production_loop: |
  **本仓库 framework 无 production loop** — LangChain framework 是 dev-time library。

  **LangSmith ecosystem 提供 production loop**(闭源 SaaS):
  - LangSmith dashboard 看 production trace
  - Datasets + Experiments
  - Evaluators 自定 metric
  - **LangSmith Deployment** 长任务 stateful agent

  跟 [[../eval/langfuse]] `addDatasetRunItemsToEvalQueue` production loop 对照:**Langfuse 完全 OSS,LangSmith 闭源 SaaS**。
  [doc-only]

automation: |
  - **Callback / Tracer** 集成业内事实标准 — Langfuse / Opik / Phoenix / Helicone 都通过 callback handler 接 LangChain
  - **`init_chat_model("provider:model")`** — 1 行代码切 provider
  - **`partners/` 16 PyPI package** — `pip install langchain-anthropic` 按需
  - **CI/CD via standard-tests**(`libs/standard-tests/`)— partner package 必通过
  - LangChain Academy(免费课程)+ Chat LangChain(`chat.langchain.com`)— dev experience
  [verified-from-source]

cost: |
  - **`model-profiles/`** 集中维护模型 cost / context / capability — 对照 Langfuse `default-model-prices.json` / Helicone `buildPricesOpenRouter.py`
  - LangSmith dashboard 自带 cost tracking(闭源 SaaS)
  - **`rate_limiters.py`**(`langchain_core/`)— 一等公民
  [verified-from-source + doc-only]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:LangChain 是**业内事实标准 framework**,SkillForge 用户可能在 agent code 里用它

  SkillForge 当前:
  - **server-side platform**(Java/Spring + ChannelAdapter SPI)
  - 用户在 channel 跟 SkillForge 交互,SkillForge 后端编排 agent
  - 用户可能在自己 agent / skill 里用 LangChain Python(SkillForge 不限制)

  ## 可借鉴(给 SkillForge 工程细节)

  ### 高优(1-2 周)
  1. **`provider:model` 字符串风格**(`init_chat_model("openai:gpt-5.4")`)— SkillForge ChannelAdapter / provider config 可考虑这种 string-based routing(对照当前可能用 enum + provider name 分开)
  2. **OpenInference / OTel callback ingestion** — SkillForge 通过 OTel 接受 LangChain trace(对照 R14 OTel ingestion 需求)+ Phoenix / Langfuse / Opik 互通
  3. **`langchain_core/callbacks/` Callback System 模板** — SkillForge 自家 LifecycleHooks(N3 已交付)可对照 LangChain callback 模式确保业内兼容

  ### 中优(1 个月)
  4. **`Runnable | pipe`** 组合抽象 — SkillForge skill 编排是否提供 pipe 风格 DSL(对照当前可能是 YAML/Spring 配置)
  5. **`Standard Tests`**(`libs/standard-tests/`)— SkillForge skill marketplace 可强制 standard test 通过(对照 SKILL-IMPORT / SKILL-IMPORT-BATCH 已交付)
  6. **`model-profiles/`** 集中模型 profile — SkillForge 模型配置可拆独立 profile module

  ### 低优 / 不必跟
  - **`Runnable Protocol` 全栈抽象**(SkillForge Java 生态不需要重做,已有自家抽象)
  - **`langchain_v1/` vs `langchain_classic/` 双轨**(SkillForge 单一 API)
  - **`Deep Agents`**(SkillForge SubAgent + Team 已实现等价能力)
  - **LangSmith SaaS 路线**(SkillForge 不该绑死闭源 SaaS,走 OSS Langfuse / Opik 集成)

  ## SkillForge 已领先 LangChain ecosystem
  - **完整 production loop + 飞轮 V1+V2+V3+V4**(LangChain ecosystem 靠 LangSmith 闭源)
  - **Skill 系统 + marketplace 一等公民**(LangChain Deep Agents 才推 subagents)
  - **多 channel + ChannelAdapter SPI**(LangChain 没 channel 抽象)
  - **`/evolve` meta self-evolution + ATTRIBUTION-AGENT**(LangChain ecosystem 没有等价)
  - **MULTIMODAL-MVP + MCP-CLIENT-MVP + PROMPT-CACHE-MVP**(LangChain 主要靠 partner package + LangSmith 闭源)

  ## 关键决策建议

  **SkillForge 跟 LangChain 关系定位**:
  - **不绑死 LangChain**:SkillForge 自家 SPI 抽象,用户 agent code 可用 LangChain 也可用任何其他
  - **通过 OTel + Callback 收集 LangChain trace**:用户在 SkillForge agent 用 LangChain 时,trace 自动进 SkillForge observability
  - **借鉴 Callback System + Runnable pipe 工程哲学**(不重做,但理解业内事实标准)
  - **不学 LangSmith 闭源 SaaS 路线**:SkillForge 完全 OSS 路线更适合

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **业内规模最大 LLM framework ecosystem**(5 大产品:LangChain / Deep Agents / LangGraph / LangSmith / LangSmith Deployment)
  2. **2026 v1 重写**(v1.3.1),完整 API 重做
  3. **`Runnable Protocol` + `|` pipe 组合**(`langchain_core/runnables/`)— 业内标志性抽象,启发 DSPy / Promptfoo 等
  4. **`Callback System`**(`langchain_core/callbacks/`)— 业内事实标准 observability 集成 base layer
  5. **`init_chat_model("provider:model")`** 字符串风格 — 业内独家
  6. **`Standard Tests`** 业内独家:partner package 强制通过标准测试
  7. **`partners/` 16+ 独立 PyPI package** — 每 provider 1 package
  8. **5 layer 抽象**(low Runnable / mid Chain / high Agent / higher LangGraph / highest Deep Agents)— 学习曲线陡但功能完整
  9. **v0 (`langchain_classic`)+ v1 (`langchain_v1`)+ core 三 monorepo 拆分** — 兼容 + 演进
  10. **`langchain.com` + `docs.langchain.com` + `chat.langchain.com` + `forum.langchain.com` + `academy.langchain.com`** 5 站完整生态

  ## 优势 (strengths)
  1. **业内 community 最大** — community-contributed components / templates / integrations 最多
  2. **`Runnable` invoke / batch / stream / ainvoke 统一接口** — 工程化
  3. **`langchain_core` lightweight + `partners/` 按需安装** — 用户 pip install 控制大小
  4. **`model-profiles/`** 模型 profile 集中(context / cost / capability)
  5. **`text-splitters/`** 独立模块 — RAG 必备
  6. **`tracers/` 抽象 + LangSmith / Langfuse / Phoenix 多 backend** — 用户选 observability backend
  7. **`Document / Embedding / VectorStore / Retriever`** 抽象齐全 — RAG full stack
  8. **`Output Parsers`** 模块化(JSON / Pydantic / XML / Markdown 各 parser)
  9. **`example_selectors/`** few-shot 选择(KNN / 长度 / 语义)— 业内独家把 few-shot 选择抽象成 first-class
  10. **`indexing/`** 独立模块 — RAG indexing 工程化
  11. **`load/` 序列化** — 完整 module state 持久化
  12. **`forum.langchain.com`** 替代 GitHub Issues 做 community discussion

  ## 哪些点做得好 (what_done_well)
  1. **`Runnable | pipe`** 操作符重载 — Pythonic + Unix philosophy(对照 DSPy `Module(Signature)` 风格)
  2. **`init_chat_model("openai:gpt-5.4")`** 字符串风格切 provider — dev 体验极致(对照 LiteLLM `completion(model="claude-3-5-sonnet-20241022")`)
  3. **`langchain_classic/` 命名**(`libs/langchain/langchain_classic/`)— 比 `langchain_v0` 更 dignified,**保留 legacy 体面 deprecate**
  4. **`Standard Tests`**(`libs/standard-tests/`)强制 partner 一致性 — 业内独家
  5. **`Callback` + `Tracer` 双轨抽象** — Callback 给 live observability,Tracer 给 persistent trace,**concern 分离**
  6. **`model-profiles/`** 集中模型 profile — 减少散落配置
  7. **`partners/anthropic/` / `partners/openai/` / 等 16 个独立 PyPI** — `pip install langchain-anthropic` 按需,**对照 [[../harness/codex]] 100+ crate 路线**
  8. **`forum.langchain.com` + `chat.langchain.com` + `academy.langchain.com`** 3 站 dev 资源
  9. **`extended_testing_deps.txt`**(`libs/core/`)+ `tests/` — 测试依赖跟主依赖分离
  10. **`README.md` "TIP" + "WARNING" 标签丰富** — dev 友好 onboarding
  11. **`init_chat_model` lazy import provider package** — 用户用 `"anthropic:..."` 时才 import `langchain-anthropic`
  12. **`langchain-ai/langsmith-sdk` SDK MIT + LangSmith SaaS 闭源** — 商业模式漂亮(SDK 开源给社区,SaaS 收费)
---

# LangChain(LangChain Inc, MIT, v1.3.1, 2026 重写)

## 一句话定位

**业内规模最大 LLM framework + 5 大产品生态**(LangChain framework / Deep Agents / LangGraph / **LangSmith 闭源 SaaS** / LangSmith Deployment)。slogan "The agent engineering platform"。**本 ingest 主要覆盖 LangChain framework**,LangSmith eval/observability 闭源走 doc-only。

## v1 重大变化(2026)

- 主入口:`from langchain.chat_models import init_chat_model`
- 字符串切 provider:`init_chat_model("openai:gpt-5.4")`
- 重写完整 API,**与 v0 不兼容**(`libs/langchain/langchain_classic/` 保留兼容层)
- **Deep Agents** 高层包(planning / subagents / file system)

## 5 大产品(README "LangChain ecosystem")

```
LangChain framework(本仓库,OSS)
       ↓ 高层包
Deep Agents(独立 repo)
       ↓ 低层
LangGraph(独立 repo)— Agent orchestration
       ↓ production
LangSmith(闭源 SaaS)— Evals / Observability / Debugging
       ↓ deployment
LangSmith Deployment — 长任务 stateful agent
```

**完整 ecosystem 跨 5+ repos**。

## SkillForge 校准

详见 frontmatter `skillforge_takeaway`。最核心 3 条:

1. **SkillForge 不绑死 LangChain**(自家 SPI 抽象,用户 agent code 可用 LangChain 也可用 raw OpenAI SDK)
2. **通过 OTel / Callback 收集 LangChain trace**(用户在 agent 用 LangChain 时,SkillForge observability 自动接到)
3. **不学 LangSmith 闭源 SaaS 路线**(SkillForge 完全 OSS 更适合,集成 Langfuse / Opik / Phoenix 等)

**SkillForge 已超过 LangChain ecosystem 几个维度**:Production loop 飞轮 V1-V4 + Multi-surface 优化 + ATTRIBUTION-AGENT + MULTIMODAL-MVP + MCP-CLIENT-MVP + PROMPT-CACHE-MVP + Skill marketplace 一等公民。

## 待回答(LangSmith 闭源不能 verify)
- LangSmith Evaluators schema(对照 Langfuse Score 3×5 / Opik Assertion+Feedback / Phoenix Score)
- LangSmith Datasets schema
- LangSmith Experiments 实现
- LangGraph state machine + checkpointing 细节(独立 repo)
- Deep Agents planning / subagents 实现(独立 repo)
- LangSmith Deployment 长任务 stateful agent 怎么做
