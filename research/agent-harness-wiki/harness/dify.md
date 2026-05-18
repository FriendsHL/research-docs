---
name: dify
type: harness
status: active
language: python+typescript
last_ingested_version: "latest main @ 9d0906c"
last_ingested_at: 2026-05-17
last_commit: 9d0906c684c4ccbb9ea665fb6aa3fcf562aabe57
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/dify]]"
  - "/Users/youren/myspace/dify/"

agent_loop: |
  5 个 runner (`api/core/agent/`):
    base_agent_runner.py / cot_agent_runner.py / cot_chat_agent_runner.py /
    cot_completion_agent_runner.py / fc_agent_runner.py
  显式区分 CoT(老模型/国产模型 prompt-only)vs Function Calling(OpenAI/Claude 风) 两条路线 —
  跟绝大部分 framework 押 function calling 一条路不同, Dify 保留 CoT 是 production 必要.
  Workflow 模式下走 workflow node engine,不走 ReAct.

tool_protocol: |
  **6 tool 类型** (`api/core/tools/`):
    builtin_tool / custom_tool / mcp_tool / plugin_tool / **workflow_as_tool**(独家) / __base
  workflow_as_tool 业内独家把已编辑 workflow 包成 callable tool 给 agent 调用 — nested workflow composition.
  ToolEngine + ToolManager + ToolLabelManager + signature.py 完整工程化.

context_mgmt: |
  **极简**:`api/core/memory/token_buffer_memory.py` 1 文件 —
  Dify 不做长期 memory,长期 context 全走 RAG knowledge base.
  这是设计选择,不是缺失 — workflow + RAG knowledge base 取代了"agent 内 memory"角色.

memory: |
  同 context_mgmt — Dify "memory" = RAG knowledge base + dataset retrieval.
  没有 mem0/Letta/MemPalace 那种 entity/reflection/summarization 路线.
  对照 [[concept/agent-memory]] 5 路线, **Dify 是第 6 路线: pure-RAG-as-memory**.

multi_agent: |
  Workflow 内 agent 节点可调用其他 workflow_as_tool, 形成 nested 协调.
  没有 SubAgent / Coordinator / Fork 之类细粒度 multi-agent 抽象 —
  Dify 倾向用 workflow + agent node 解决, 而非显式 multi-agent.

eval_loop: |
  `api/core/ops/` 含 production observability + 应用层 LLMOps,
  没有独立 eval framework (vs Langfuse/Opik).
  跟 Langfuse / Opik 集成走 ops/.

self_evolution: |
  ? 未深入 verify.
  Workflow DSL 可以版本化 (NodeType × node_version 双键), 但 agent / skill 演化机制未明确.

observability: |
  `api/core/ops/` + `telemetry/` + `logging/` 三层.
  没看到自家 trace storage, 依赖外部 (Langfuse / Opik / Helicone 等) 集成.

hooks: |
  Workflow-level 触发器: trigger_plugin / trigger_schedule / trigger_webhook 3 类.
  不是 Claude Code 风 lifecycle hook, 是 workflow 入口 hook (cron / event / API).
  Hook 数量比 lifecycle 系角度不可比.

safety: |
  `api/core/moderation/` 安全审核模块独立存在 — 业内罕见的"内容审核作为 first-class 模块".
  没有 sandbox 概念(workflow 节点不跑用户代码), 但 plugin 系统有 plugin_tool 隔离.
  详见 [[../concept/sandbox-isolation]] (阶梯 0 — _"承认不做"_ 用架构约束规避 sandbox 复杂度,合法工程决策).

provider: |
  **100+ LLM provider** 通过 `provider_manager.py` + `model_manager.py` 管理.
  业内仅次于 LangChain. 这是 Dify 商业平台的护城河之一 — 任何用户都能用任何 model 跑.

channels: |
  Web (Next.js) / Self-hosting / Dify Cloud SaaS / Docker / k8s.
  Visual drag-and-drop workflow builder 是核心 channel(非开发者也能用).

persistence: |
  PostgreSQL (主) + Redis (broker) + S3 兼容 (文件) + 向量 DB 多种.
  Celery 异步任务. DDD repositories 层封装.

cancellation: |
  Workflow node 级 cancel (workflow_run state 管理).
  Agent runner 级 cancel 未深 verify.

skillforge_takeaway: |
  Dify 跟 SkillForge **产品定位最近**(都是"给企业 / 业务方建 agent + skill + RAG 的平台"),
  比纯 framework (LangChain/LlamaIndex/LangGraph) 更直接对照.

  🔴 **R-DIFY-1 — Visual Workflow + DSL 双轨**(P1 product 决策)
     Dify `services/rag_pipeline/rag_pipeline_dsl_service.py` 把 workflow DSL 跟 visual UI 互转,
     用户可在 UI 拖拽 或 直接编辑 YAML — 双向 sync. SkillForge skill YAML + Dashboard 可参考此模式.
     工作量: SkillForge 当前 skill YAML 已是 source of truth, 加 Dashboard 编辑器 ROI 高.

  🔴 **R-DIFY-2 — workflow_as_tool 抽象**(P1 architecture)
     `api/core/tools/workflow_as_tool/` 把已发布 workflow 包成 callable tool 给 agent 调用 —
     nested composition. SkillForge 当前 skill 也是 tool 级,但 **skill 调 skill** 没有显式抽象.
     这条直接对应 SkillForge SubAgent 模式, 但更轻量(不开 child session,只 tool 调用).
     工作量: 1-2 周.

  🟡 **R-DIFY-3 — 100+ provider 统一 manager**(P2 ecosystem)
     `provider_manager.py` + `model_manager.py` 处理 100+ provider 接入,
     model selection 路由策略 + 配置 hot-reload + 限流分账.
     SkillForge 当前 2 provider (OpenAI + Claude), 商业化阶段需要扩.

  🟡 **R-DIFY-4 — moderation first-class 模块**(P2 safety)
     `api/core/moderation/` 把内容审核独立成 module,不是埋在某个 layer 里.
     SkillForge safety hook 当前散在 PreToolUse / behavior_rule, 可参考独立化.
     工作量: < 1 周.

  🟢 **R-DIFY-5 — RAG 4 retrieval method × 2 rerank mode 明确枚举**(参考)
     `RetrievalMethod` 4 枚举 + `RerankMode` 2 枚举 + 静态 helper 显式判断 method-feature 兼容.
     SkillForge MemoryService 当前 hybrid FTS+Vector+RRF 单一路径, 远期可参考 4-method 抽象.

  🟢 **R-DIFY-6 — 多 harness dogfood 套路**(参考)
     `.claude/skills/` + `.agents/skills/` + `.codex` + `.gemini/` + `AGENTS.md ↔ CLAUDE.md symlink`.
     SkillForge 自己开发不一定用 7 套, 但 **AGENTS.md ↔ CLAUDE.md symlink** 这种细节
     标准化协议(同一份文档对多家 agent 工具)很值得抄.

  **❌ 不该跟**:
  - Visual builder UI 主线 — SkillForge 是 Dashboard 路线, 不做拖拽
  - 100+ provider 自己接 — 先用 OpenAI + Claude + 国产 3 家足够
  - Workflow engine 100% 抄 — SkillForge skill+agent 已经是更细粒度抽象
---

# Harness — Dify @ commit 9d0906c

> Dify 是业内最大的 **Production AI Application Platform**,跟 SkillForge 产品定位最近。13 维详见 frontmatter。本页 narrative 重点:**workflow_as_tool 抽象**、**RAG 工程化深度**、**100+ provider 护城河**、**多 harness dogfood 自证** 4 个独家角度。

---

## 1. 项目定位(一句话)

**LF AI & Data Foundation 旗下,业内 #2 RAG/agent framework**(114k stars,仅次于 LangChain 125k)。产品形态 = **production AI 应用平台**:RAG + Agent + Workflow + Model Management + Visual UI 全栈一体,Self-hosting / Dify Cloud SaaS 双轨。

---

## 2. 为什么对 SkillForge 重要 — 产品定位对照

Dify 跟 SkillForge 是 wiki 已 ingest 项目里**最接近的产品定位**:

| | Dify | SkillForge | 5 个 dev tool harness(Hermes/CC/OpenClaw/Codex/opencode) |
|---|---|---|---|
| 用户 | 业务方 / 非开发者 / 企业 | 业务方 / 非开发者 / 企业 | 开发者本人 |
| 入口 | Visual workflow + chatbot | Skill + Agent + Dashboard | CLI + IDE |
| 持久化目标 | 业务知识库 + 工作流 | Skill marketplace + Memory + Agent定义 | 单用户 session |
| 商业模型 | SaaS(Cloud)+ Self-hosting | 同 | OSS 工具 |
| 多租户 | ✅ | ✅(规划)| ❌(单机) |

**所以**:Dify 提供 SkillForge **最直接的产品形态对照**。其他 dev tool harness 提供**技术深度**对照(memory / hooks / observability),但产品 / 商业 / 多租户 / 业务方友好度对照空缺,Dify 填上。

---

## 3. 四大独家技术点

### 3.1 `workflow_as_tool` 抽象(`api/core/tools/workflow_as_tool/`)

**业内独家**:已发布 workflow 可以被 agent 当 tool 调用,实现 **nested workflow composition**。

```
Agent A 调用 Tool X
  X 是 Workflow B(被 wrap 成 tool)
    Workflow B 节点 N 是 Agent C
      Agent C 又调用 Tool Y
        Y 是 Workflow D
          ...
```

LangChain Tool / OpenAI Function 都没有这个抽象 — 它们的 Tool 是"Python 函数"或"API 端点",**不是"另一个 workflow"**。

**对 SkillForge 启示** [verified-from-source]:
- SkillForge 当前 `SubAgentRunEntity` 是 session-level spawn(完整 child session)
- SkillForge skill 当前是 prompt + tool list,**没有"skill 调 skill" 显式抽象**
- 加 `R-DIFY-2`:把 skill A 调 skill B 升级为 tool 调用而非 sub-session — 轻量化 nested compose

### 3.2 RAG 14 子模块完整工程化(`api/core/rag/`)

对照 5 路 RAG framework(详见 [[concept/rag-design-space]] 后续):

| 子模块 | Dify | LlamaIndex | LangChain | RAGFlow | SkillForge |
|---|---|---|---|---|---|
| **cleaner** | ✅ 独立模块 | partial | partial | ✅ | ❌ |
| **data_post_processor** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **extractor** | ✅ | ✅ | ✅ | **强**(table/figure) | ❌ |
| **splitter** | ✅(2 文件) | ✅(多种 strategy) | ✅(各种 splitter) | ✅ | ❌(memory_kind 三态非 chunking) |
| **embedding** | ✅ + cached_embedding | ✅ | ✅ | ✅ | ✅(`MemoryService.java`) |
| **rerank** | ✅(model + weighted)| ✅(多种)| ✅ | ✅(自家)| ✅(RRF) |
| **retrieval methods** | 4(semantic/fulltext/hybrid/keyword)| ✅(多种)| ✅ | ✅ | ✅(hybrid:FTS+Vector+RRF) |
| **summary_index** | ✅(独家) | ✅(类似) | partial | ? | ❌ |

**SkillForge RAG 工程化广度对比 Dify 显著欠缺**(因为 SkillForge memory 是 entity-based 而非 doc-based)。但 SkillForge 把 RAG-ish 能力都包在 MemoryService 里,**逻辑等价**。Dify 工程化抽象给 SkillForge memory v3 重构提供模板。

### 3.3 100+ LLM Provider 统一(`api/core/provider_manager.py`)

业内仅次于 LangChain。具体细节看 source page,但产品意义:**任何用户带任何 model 都能跑**,这是 Dify 商业护城河。

**对 SkillForge**:商业化前 OpenAI + Claude + 国产 3 家(豆包/通义/Kimi/DeepSeek)足够。Dify 100+ 是 platform aspiration,SkillForge 是垂直 SaaS,**不需要照抄**。

### 3.4 多 harness Dogfood(`.claude/` + `.agents/` + `.codex` + `.gemini/`)

`.claude/skills/` 6 + `.agents/skills/` 7 + `.codex` marker + `.gemini/` + `CLAUDE.md → AGENTS.md symlink`。

**意义**:Dify 团队用多家 agent 工具开发自己,且**用 7 个标准化 skill 覆盖 backend review / component refactor / E2E test / frontend review / 写组件 / Tailwind**。这套是 "production AI 平台团队 dogfood 流程" 模板。

**对 SkillForge 启示**:
- SkillForge 自家 dogfood 流程未 verify
- 可参考 Dify 7 skill 模板组建 SkillForge 自家开发 SOP

---

## 4. 跟现有 wiki 关系

| 当前页 | 关系 |
|---|---|
| [[concept/agent-runtime-stacks]] | 加 6th stack:**Flask DDD + Celery + Redis + Next.js**(Python production platform 路线,vs SkillForge Java Spring) |
| [[concept/agent-memory]] | 加 6th 路线:**Pure-RAG-as-memory**(Dify 不做 entity/reflection,长期 context 全 RAG)— 跟 mem0/Letta/MemPalace/LangMem/Zep+Graphiti 5 路线正交 |
| [[concept/rag-design-space]] | 第一个 contributor — 14 子模块工程化 + 4 retrieval method + 2 rerank mode |
| [[concept/lsp-integration]] | 无关 |
| [[concept/agent-harness]] | R-DIFY-1~R-DIFY-6 加入 borrow clipboard |

---

## 5. SkillForge R-DIFY 借鉴清单

详见 frontmatter `skillforge_takeaway`. 总结:

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-DIFY-1** | Visual Workflow + DSL 双轨(YAML ↔ UI 双向)| 中期 | 高 — 产品差异化 | 🔴 P1 |
| **R-DIFY-2** | workflow_as_tool / skill_as_tool 轻量 nested | 1-2 周 | 高 — 架构清晰 | 🔴 P1 |
| **R-DIFY-3** | 100+ provider 统一 manager(收敛到 3-5 家)| 1-2 周 | 中 — 商业化 | 🟡 P2 |
| **R-DIFY-4** | Moderation first-class 模块 | < 1 周 | 中 — safety | 🟡 P2 |
| **R-DIFY-5** | RAG 4 retrieval method × 2 rerank mode 抽象 | 中期 | 远期参考 | 🟢 P3 |
| **R-DIFY-6** | 多 harness dogfood 套路(7 skill 模板)| < 半周 | 中 — 团队效率 | 🟢 P3 |

---

## 6. 已知 post-ingest 风险

- Workflow node engine 内部细节未深(只看到 factory + lazy registry)
- `graphon.nodes` 外部包未深入(可能是 production 节点实现源)
- RAG pipeline DSL service 9 文件未 verify(DSL 协议 / transform 协议待挖)
- Plugin runtime 沙箱机制未 verify(只看到 plugin/ 5 子目录)

---

## 7. 引用

- [[source/dify]] 完整源码锚点
- LF AI & Data Foundation health badge
- GitHub: `langgenius/dify` 114k+ stars
- 跨页对照:[[harness/agentscope]](Alibaba 路线产品 vs Dify production platform)/ [[concept/agent-runtime-stacks]] / [[concept/agent-memory]] / [[concept/rag-design-space]]
