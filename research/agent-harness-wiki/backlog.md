---
name: backlog
type: meta
updated: 2026-05-17
---

# Backlog — 待 ingest 模块清单

> 本文件是 wiki 的 **ingest 排期清单**。每次 `/wiki-ingest` 之前从这里取下一个目标,完成后把 `status` 改成 `done` 并在 [[log]] 留时间戳。
>
> **跟 [[log]] 的区别**:log 是已完成时间线(向后看),backlog 是未完成计划(向前看)。
> **跟 [[index]] 的区别**:index 索引已 ingest 页面,backlog 列尚未存在的页面。

## 状态约定

| status | 含义 |
|---|---|
| `planned` | 已识别 + 优先级已定,等待 ingest |
| `in-progress` | 已开始 ingest 但未发布 |
| `done` | 已 ingest,index 已收录 |
| `parked` | 已识别但主动暂缓(写明原因) |

## 优先级约定

- **P0** — agentic 强相关 + 业内必看 + 跟现有 wiki 有直接对照,**下一批要做的**
- **P1** — 重要但跟现有页面对照点较少 / 闭源 doc-only
- **P2** — 完整性收尾,先记录不急做

---

## 0. 用户钦点的两个 P0 (2026-05-17)

### 0.1 `concept/sandbox-isolation` — Sandbox / 执行隔离横切页

现状:`sandbox` 在 6 个 harness 页里被引用但没有自己的概念页 — codex(bwrap+landlock+seatbelt+windows)/ ragflow(gVisor)/ hermes(7 backends:local/docker/ssh/modal/daytona/singularity/vercel)/ opencode(shell+pty)/ openclaw(Dockerfile.sandbox)/ dify(plugin 隔离)— 跨页引用但**没有横向对照的 anchor**。

要建的页面:`concept/sandbox-isolation.md`,跨 harness 把 sandbox 设计空间梳理清楚 — 隔离粒度(进程/容器/microVM/userspace kernel)× 执行对象(shell/code/tool)× 部署形态(本地/远程/edge)。

**对照对象**(已 verify in wiki):
- Codex `sandboxing/` + `linux-sandbox/` + `bwrap/` + `windows-sandbox-rs/` 3 平台 +1
- RAGFlow `agent/sandbox/` gVisor Docker Compose
- Hermes `tools/environments/` 7 backends
- opencode `packages/opencode/src/shell/` + `pty/`
- OpenClaw `Dockerfile.sandbox-{base,browser,common}`
- Dify plugin_tool 隔离

**新增对照对象**(外部 raw source,需 ingest):
- **E2B** — cloud code sandbox SDK,几乎所有 cloud agent 在用,Firecracker microVM 路线
- **Daytona** — dev environment as a service,workspace 形态
- **Modal** — Python-first cloud compute,被 Hermes 直接用
- **Vercel Sandbox** — 被 Hermes 引用,Vercel infra 路线
- **gVisor** — Google userspace kernel,RAGFlow 唯一直接用
- **Firecracker** — AWS microVM,E2B 底层
- **Cloudflare Workers / Sandbox** — V8 isolate edge 路线,可跟 Codex Code Mode 对照
- **StackBlitz WebContainers** — browser-side Node runtime,完全 client-side
- **bubblewrap / landlock / seatbelt** — Linux/macOS 系统调用层(Codex 已 verify)

**预期产物**:
- `concept/sandbox-isolation.md` — N 维设计空间 + 9 实现对照表 + SkillForge 选型
- 可选 `deep/codex-sandbox-3-platform.md` 或 `deep/ragflow-gvisor.md` 后续做单点深挖

| status | priority |
|---|---|
| **done(2026-05-17)** → [[concept/sandbox-isolation]] | P0 |

### 0.2 `concept/a2a-protocol` + `deep/google-a2a-spec` — Agent2Agent 协议

现状:A2A 在 [[harness/agentscope]] / [[harness/crewai]] 两页都集成但**没有协议本身的 anchor 页**。`concept/agent-interop` 一句话带过(_"MCP 是统一 tool 接口"_ 跟 A2A 区分),但 A2A 自己缺独立页面。

要建的页面:
- `concept/a2a-protocol.md` — A2A spec 概念页 + 跟 MCP/ACP/AGNTCY 横向定位
- (可选)`deep/google-a2a-spec.md` — A2A v0.x spec 逐节深挖 → **done(2026-05-18)** [[deep/google-a2a-spec]]

**对照对象**:
- Google A2A spec(`a2a.dev` / `agent2agent-protocol` repo)
- AgentScope `a2a/` 5 文件 + `_nacos_resolver.py`(已 verify in [[harness/agentscope]])
- CrewAI `a2a/` 子目录(已 verify in [[harness/crewai]])
- ACP (Agent Communication Protocol, Linux Foundation)
- AGNTCY (Cisco + LF)
- AG-UI (UI 侧 agent 协议)

**横向问题**:agent 互通至少 3 层 — tool 层(MCP)/ agent 层(A2A/ACP)/ UI 层(AG-UI),wiki 现在只有 MCP 在 [[harness/claude-code]] 和 [[concept/agent-interop]] 里提到,A2A 完全没单独 anchor。

**实际产出**:合并到 [[concept/agent-protocols]] 一页(2026-05-17),扩为 **5 层栈**(加 OTel 侧向贯穿层)。ACP 已 merge 进 A2A(a2a-protocol.org 2026-05-17 明示),作 historical note。

| status | priority |
|---|---|
| **done(2026-05-17)** → [[concept/agent-protocols]](合并 §5 同条目) | P0 |

---

## 1. Harness — Agent 运行时框架

> index.md 顶部已列出但未 ingest:Aider / Cline / EvolvingAgent / superpowers / SkillForge

### P0(下一批做)

| name | type | 一句话定位 | raw source | status |
|---|---|---|---|---|
| `aider` | harness | paul-gauthier/aider,coding agent 老牌 CLI,Python,git-aware diff-edit 路线 | github.com/Aider-AI/aider | planned |
| `openhands` | harness | All-Hands-AI/OpenHands(原 OpenDevin),开源 Devin 路线最完整,Python+TS | github.com/All-Hands-AI/OpenHands | planned |
| `smolagents` | harness | HuggingFace,**code-mode 鼻祖之一**(LLM 写 Python),跟 [[deep/codex-code-mode-v8-isolate]] 直接对照 | github.com/huggingface/smolagents | **done(2026-05-18)** → [[harness/smolagents]] |
| `langgraph` | harness | LangChain 出品的 stateful agent graph,跟 [[eval/langchain]] 分离独立做 harness 视角 | github.com/langchain-ai/langgraph | **done(2026-05-18)** → [[harness/langgraph]] |
| `openai-agents-sdk` | harness | OpenAI Agents SDK(原 Swarm),Python+TS,handoff 抽象 | github.com/openai/openai-agents-python | planned |

### P1

| name | type | 一句话定位 | raw source | status |
|---|---|---|---|---|
| `cline` | harness | cline/cline,VS Code agent,TypeScript,plan/act 双 mode | github.com/cline/cline | planned |
| `roo-code` | harness | Roo-Code(原 Roo Cline),Cline fork,multi-mode | github.com/RooCodeInc/Roo-Code | planned |
| `goose` | harness | Block/Square 出品,MCP 主推 + 多 provider,Rust | github.com/block/goose | **done(2026-05-18)** → [[harness/goose]] |
| `continue` | harness | continuedev/continue,IDE-first 开源 Copilot 替代,TypeScript | github.com/continuedev/continue | planned |
| `crush` | harness | charmbracelet/crush,TUI 路线,Go,跟 [[harness/opencode]] 同类 | github.com/charmbracelet/crush | planned |
| `swe-agent` | harness | Princeton,跟 [[eval/swe-bench]] 配套但单独的 agent loop,Python | github.com/SWE-agent/SWE-agent | planned |
| `pydantic-ai` | harness | Pydantic AI,Python 类型安全 agent 框架,structured output 一等 | github.com/pydantic/pydantic-ai | planned |
| `autogen-v04` | harness | Microsoft AutoGen v0.4 重写(跟 [[eval/ag2]] fork 分流),actor + event-driven | github.com/microsoft/autogen | planned |
| `google-adk` | harness | Google Agent Development Kit,跟 Gemini + A2A 绑定 | github.com/google/adk-python | planned |
| `mastra` | harness | Mastra,TypeScript agent framework + workflow + RAG | github.com/mastra-ai/mastra | planned |

### P2(完整性)

| name | type | 一句话定位 | raw source | status |
|---|---|---|---|---|
| `agno` | harness | Agno(原 Phidata),Python,multi-modal + reasoning | github.com/agno-agi/agno | planned |
| `bee-agent` | harness | IBM Bee Agent Framework,TypeScript,Granite 主推 | github.com/i-am-bee/beeai-framework | planned |
| `strands-agents` | harness | AWS Strands Agents,Python | github.com/strands-agents/sdk-python | planned |
| `taskweaver` | harness | Microsoft TaskWeaver,code-as-action,Python | github.com/microsoft/TaskWeaver | planned |
| `genkit` | harness | Google Genkit,跟 ADK 互补,TS+Go+Python | github.com/firebase/genkit | parked — 跟 ADK 重叠,先做 ADK |
| `voltagent` | harness | VoltAgent,TypeScript,observability-first | github.com/VoltAgent/voltagent | planned |
| `langroid` | harness | langroid/langroid,Python,multi-agent,academic 风 | github.com/langroid/langroid | planned |
| `plandex` | harness | plandex-ai/plandex,Go,plan 一等公民 | github.com/plandex-ai/plandex | planned |
| `tabby` | harness | TabbyML/tabby,self-hosted Copilot,Rust | github.com/TabbyML/tabby | planned |
| `gpt-pilot` | harness | Pythagora-io/gpt-pilot,old-school code generator | github.com/Pythagora-io/gpt-pilot | parked — 已 legacy,只做 doc 摘要 |

### 闭源 / doc-only(参考对照)

| name | type | 一句话定位 | status |
|---|---|---|---|
| `cursor` | harness | Anysphere Cursor,闭源 IDE,基于公开 blog/docs/逆向 | planned — doc-only |
| `windsurf` | harness | Codeium Windsurf,闭源 IDE | planned — doc-only |
| `devin` | harness | Cognition Devin,闭源 cloud agent | planned — doc-only |
| `replit-agent` | harness | Replit Agent + Bounties,闭源 | planned — doc-only |
| `factory-droids` | harness | Factory.ai droids,闭源 cloud agent | planned — doc-only |
| `jules` | harness | Google Jules,闭源 background agent | planned — doc-only |
| `lovable-bolt-v0` | harness | Lovable / Bolt.new / v0,web app builder 三家(合并一页) | planned — doc-only |
| `sourcegraph-amp` | harness | Sourcegraph Amp,闭源 coding agent | planned — doc-only |

---

## 2. Eval / Observability — 评测 + 可观测

### P0

| name | 一句话定位 | raw source | status |
|---|---|---|---|
| `braintrust` | YC 出品,eval-first observability platform,跟 [[eval/langfuse]] 对照 | github.com/braintrustdata + braintrust.dev docs | planned |
| `inspect-ai` | UK AISI 出品,**agent safety eval 标杆**,wiki 现在 safety eval 维度空白 | github.com/UKGovernmentBEIS/inspect_ai | planned |
| `weave` | W&B Weave,实验跟踪 + LLM tracing | github.com/wandb/weave | planned |

### P1

| name | 一句话定位 | raw source | status |
|---|---|---|---|
| `logfire` | Pydantic Logfire,OTel-native,跟 [[harness/]] pydantic-ai 配套 | github.com/pydantic/logfire | planned |
| `trulens` | TruEra TruLens,RAG triad + feedback function 路线 | github.com/truera/trulens | planned |
| `deepeval` | Confident AI,pytest 风 LLM eval | github.com/confident-ai/deepeval | planned |
| `lunary` | lunary-ai/lunary,open-source LLM observability | github.com/lunary-ai/lunary | planned |
| `baml` | boundary BAML,DSL-first prompt+function,跟 [[concept/prompt-optimization]] 互补 | github.com/BoundaryML/baml | planned |

### P2

| name | 一句话定位 | status |
|---|---|---|
| `galileo` | Galileo,闭源 LLM observability | planned — doc-only |
| `honeyhive` | HoneyHive | planned — doc-only |
| `promptlayer` | PromptLayer | planned — doc-only |
| `vellum` | Vellum | planned — doc-only |
| `langsmith` | LangSmith(LangChain 闭源 SaaS)— [[eval/langchain]] 提到但没单独页 | planned — doc-only |

---

## 3. Benchmark — agentic 评测集合

> wiki 现有 [[eval/swe-bench]] + [[eval/tau-bench]],agentic benchmark 维度大量空白。

### P0

| name | 一句话定位 | raw source | status |
|---|---|---|---|
| `agentbench` | THUDM,LLM-as-Agent 多维度评测起点 | github.com/THUDM/AgentBench | planned |
| `gaia` | HuggingFace GAIA,general assistant benchmark | huggingface.co/datasets/gaia-benchmark/GAIA | planned |
| `osworld` | OSWorld,真实 OS 桌面 agent,跟 computer-use 配套 | github.com/xlang-ai/OSWorld | planned |
| `webarena` | WebArena + VisualWebArena,web agent 标杆 | github.com/web-arena-x/webarena | planned |
| `agentdojo` | ETH AgentDojo,**agent prompt injection 安全 benchmark 标杆** | github.com/ethz-spylab/agentdojo | planned |

### P1

| name | 一句话定位 | status |
|---|---|---|
| `terminalbench` | TerminalBench,terminal agent | planned |
| `mle-bench` | OpenAI MLE-Bench,ML engineering | planned |
| `assistantbench` | AssistantBench,real-world web tasks | planned |
| `appworld` | AppWorld,multi-app agent | planned |
| `mind2web` | Mind2Web | planned |
| `webvoyager` | WebVoyager | planned |
| `browsecomp` | OpenAI BrowseComp,浏览 benchmark | planned |
| `toolbench` | ToolBench / ToolLLM | planned |
| `cybench` | Cybench,security CTF agent | planned |
| `multi-swe-bench` | Multi-SWE-Bench,SWE-bench 多语言扩展 | planned |
| `livecodebench` | LiveCodeBench | planned |
| `bigcodebench` | BigCodeBench | planned |
| `aider-polyglot` | Aider Polyglot,跟 aider 配套 | planned |
| `inspect-evals` | Inspect Evals(配套 inspect-ai) | planned |

---

## 4. RAG / 知识

> wiki 现有 [[harness/ragflow]] + [[eval/llama_index]] + [[eval/pathway]] + [[eval/ms-graphrag]] + [[deep/ragflow-deepdoc-table-recognizer]]。

| name | 一句话定位 | raw source | priority | status |
|---|---|---|---|---|
| `haystack` | deepset Haystack,跟 RAGFlow 同代际独立流派 | github.com/deepset-ai/haystack | P1 | planned |
| `r2r` | SciPhi R2R,RAG+ingestion+API 全栈 | github.com/SciPhi-AI/R2R | P1 | planned |
| `verba` | Weaviate Verba,RAG application | github.com/weaviate/Verba | P2 | planned |
| `autorag` | Marker-Inc AutoRAG | github.com/Marker-Inc-Korea/AutoRAG | P2 | planned |
| `lightrag` | HKUDS LightRAG,RAGFlow `graphrag/light/` 引用 | github.com/HKUDS/LightRAG | P1 | planned |
| `nano-graphrag` | gusye1234/nano-graphrag,精简 GraphRAG | github.com/gusye1234/nano-graphrag | P2 | planned |
| `hipporag` | HippoRAG,海马体启发 memory+RAG | github.com/OSU-NLP-Group/HippoRAG | P1 | planned |

---

## 5. 协议 / 互通(concept + deep)

> A2A 已经在 §0.2 列 P0。

| name | type | 一句话定位 | priority | status |
|---|---|---|---|---|
| `deep/mcp-spec-deep` | deep | MCP spec 逐节深挖,wiki 多页引但没单点深挖 | P0 | **done(2026-05-18)** → [[deep/mcp-spec-deep]] |
| `concept/agent-protocols` | concept | MCP / A2A / AGNTCY / AG-UI + OTel **5 层栈**横向对照(ACP 已 merge 进 A2A) | P0 | **done(2026-05-17)** → [[concept/agent-protocols]] |
| `concept/acp` | concept | Linux Foundation ACP | P1 | planned |
| `concept/agntcy` | concept | Cisco + LF AGNTCY | P2 | planned |
| `concept/ag-ui` | concept | AG-UI protocol(UI 侧) | P2 | planned |
| `concept/computer-use` | concept | Anthropic Computer Use + OpenAI CUA + browser-use 横向对照 | P1 | planned |

---

## 6. Memory(补)

> wiki 现有 [[eval/mem0]] + [[eval/mempalace]] + [[eval/langmem]] + [[eval/letta]] + [[eval/zep-graphiti]] + [[eval/cognee]]。

| name | 一句话定位 | priority | status |
|---|---|---|---|
| `memgpt-paper-deep` | MemGPT 原论文(letta 前身),concept/agent-memory 已引但没源页 | P2 | planned |
| `a-mem` | A-MEM,Zettelkasten-style agent memory | P2 | planned |
| `memorybank` | MemoryBank,Ebbinghaus 遗忘曲线 | P2 | planned |
| `generative-agents-stanford` | Stanford Smallville generative agents | P2 | planned — MetaGPT 已 ext 提及 |

---

## 7. 已有素材升级(repo 内已经攒了但没进 wiki)

| 素材 | 当前位置 | 该去哪 | priority | status |
|---|---|---|---|---|
| Hermes 12 章源码笔记 | `research/Hermes源码研究/00..11.md` | `source/hermes-agent` 升级 + 跟 [[harness/hermes-agent]] 双向引用 | P1 | planned |
| OpenAI Workspace Agents 分析 | `research/OpenAI Workspace Agents 完整分析.md` | 新建 `harness/openai-workspace-agents`(闭源 doc-only) | P2 | planned |
| Multi-agent Swarm 调研 | `research/调研项目整理/multi-agent-swarm-research.md` | 拆成 source/swarm + 合并到 [[harness/openai-agents-sdk]](Swarm 继任) | P1 | planned |
| OpenClaw 5 篇分析 | `research/OpenClaw*.md` | 已部分进 [[harness/openclaw]],其余可作 `source/openclaw-extra` | P2 | planned |
| Claude Code 源码笔记 | `research/claude code 源码/` | 已大量进 [[harness/claude-code]] + 4 deep 页,review 是否还有未覆盖维度 | P2 | parked |

---

## 8. Concept 缺口(横切)

> 现有 19 个 concept 页 + 9 个 deep 页,新缺口:

| name | 一句话定位 | priority | status |
|---|---|---|---|
| `concept/sandbox-isolation` | §0.1 钦点 | P0 | **done(2026-05-17)** → [[concept/sandbox-isolation]] |
| `concept/a2a-protocol` | §0.2 钦点(已合并到 agent-protocols) | P0 | **done(2026-05-17)** → [[concept/agent-protocols]] |
| `concept/agent-protocols` | §5 互通栈(扩为 5 层栈含 OTel) | P0 | **done(2026-05-17)** → [[concept/agent-protocols]] |
| `concept/computer-use` | §5 desktop/browser agent | P1 | planned |
| `concept/code-execution-tool` | code interpreter / E2B / e.g. tool 形态横向 | P1 | planned |
| `concept/agent-economics` | cost/budget/quota,**MetaGPT NoMoneyException 独家**,值得抽出来 | P2 | planned |
| `concept/structured-output` | json/pydantic/native response,跟 [[harness/crewai]] 20+ output 类型对照 | P2 | planned |

---

## 工作流约定(LLM 自动化)

1. 取下一个 `P0 + planned` 的条目
2. 跑 `/wiki-ingest <source path/url>` — skill 自动 B 模式 source-grounded
3. 完成后:
   - 把 `status: planned` 改成 `status: done` + 加 ingest 日期
   - 把行末附上 `→ [[harness/xxx]]` 实际 wikilink
   - 在 [[log]] 加时间线 entry(三段式:触发问题 / 改动 / takeaway)
   - 在 [[index]] 加索引
4. 如果 ingest 过程发现条目应该 split(比如 A2A 拆 concept + deep),回来更新 backlog

## 用户钦点新增条目

| name | type | 一句话定位 | priority | status |
|---|---|---|---|---|
| `openhuman` | harness | **tinyhumansai 2026-05-13 newly trending _"Personal AI superintelligence"_** Tauri 2.x 桌面 app — Memory Tree + Obsidian Wiki + TokenJuice + Composio 桥接 118+ OAuth + 桌面 mascot Google Meet | P0 用户钦点 | **done(2026-05-18)** → [[harness/openhuman]] |

## Changelog

- 2026-05-17 — v1 初版。基于 _"看下我们当前还有哪些项目没有囊括在内,尤其是 agentic 相关的"_ 触发问题。Sandbox + A2A 由用户钦点 P0。共识别 ~70 条 backlog 条目。
- 2026-05-18 — 用户钦点新增 `harness/openhuman`(2026-05-13 newly trending 项目)— wiki 内首家 personal AI niche harness。
