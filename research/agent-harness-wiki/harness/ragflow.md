---
name: ragflow
type: harness
status: active
language: python+go+typescript
last_ingested_version: 0.25.4
last_ingested_at: 2026-05-17
last_commit: 09d45046e5f46fb2300f8860a3819568d71916d3
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[source/ragflow]]"
  - "/Users/youren/myspace/ragflow/"

agent_loop: |
  Graph DSL (`agent/canvas.py:43+`) — 显式 DAG: components map + downstream/upstream 邻接.
  Agent 是 workflow 节点的一种 (`agent/component/agent_with_tools.py`),
  不是顶层 agent loop. 跟 Dify workflow 同思路, 但更声明式 (纯 JSON dict).
  Iteration / Loop / Switch / Exit_loop 显式控制流 primitive.

tool_protocol: |
  Component-based: 20+ 内置 component 各自定义 input/output schema.
  支持 Python / JavaScript code executor (2025-05-23 ship), 跑在 gVisor sandbox 内.
  MCP 集成 (2025-08-01).

context_mgmt: |
  RAG-as-memory 路线: 长上下文走 retrieval, 不做 in-context 压缩.
  Agent memory 模块 (`memory/`) 2025-12-26 ship — 晚于核心 RAG, 印证 "RAG 优先, memory 后补".

memory: |
  `memory/services/` + `memory/utils/`. Module 较新 (2025-12-26).
  本次 ingest 未深 verify 内部 (后续可补).
  对照 [[concept/agent-memory]] 5 路线, RAGFlow memory 是 RAG-engine-extended-memory.

multi_agent: |
  workflow 内 Agent 节点 + AgentMesh 子 agent.
  没有 SubAgent / Coordinator / Fork 精细抽象.
  Iteration / Loop component 提供 workflow-level 并发.

eval_loop: |
  无独立 eval framework. 依赖外部 (Ragas / Promptfoo / 等).

self_evolution: |
  不做.

observability: |
  ? 未深 verify. 跟外部 (Langfuse / Phoenix) 集成可能.

hooks: |
  agent canvas 内 component lifecycle, 但不是 Claude Code 风 lifecycle hook.

safety: |
  ⭐ **gVisor sandbox** (`agent/sandbox/`) — 业内最高级别 code execution 隔离.
  README explicit: "gVisor: Required only if you intend to use the code executor (sandbox) feature of RAGFlow."
  跟 Codex 3 平台 sandbox / Dify 无独立 sandbox 形成阶梯.
  详见 [[../concept/sandbox-isolation]] (阶梯 4 userspace kernel,业内 RAG framework 唯一).

provider: |
  7 LLM model 类型 (chat/cv/embedding/ocr/rerank/sequence2txt/tts) + 多 provider 配置.
  跟 LlamaIndex 104 / Dify 100+ provider 量级差距, 但够用.

channels: |
  Web UI / Self-hosting / RAGFlow Cloud SaaS / Docker / k8s (helm/).
  10+ 国语言 README — 国际化最广.

persistence: |
  PostgreSQL + ES / OpenSearch (RAG store) + MinIO / S3 (文件) + Redis (broker).
  Docker Compose 全栈.

cancellation: |
  `api/db/services/task_service.has_canceled` — task-level cancel.
  agent canvas `TaskCanceledException` exception 链.

skillforge_takeaway: |
  RAGFlow 跟 SkillForge 都是 production AI 平台, 但 RAGFlow 走 RAG-first 路线
  (deep document understanding + template chunking + gVisor sandbox), SkillForge 走 skill-first 路线.

  🔴 **R-RF-1 — Deep Document Understanding 16 parser + Table Structure Recognizer**(架构启示)
     RAGFlow `deepdoc/parser/*` 16 文件格式 parser + `deepdoc/vision/table_structure_recognizer.py`.
     业内最完整 PDF/OCR/table 解析栈 (mineru + paddleocr + docling 三家集成).
     SkillForge 当前 skill marketplace 含 doc, 但 doc parsing 是上传时简单文本提取, 未做深度结构化.
     远期 SkillForge memory v3 + skill content 升级时, 这套 parser 集成模板可参考.
     工作量: 远期, 不直接打到生产.

  🔴 **R-RF-2 — gVisor Sandbox 集成**(P1 safety)
     `agent/sandbox/` Docker Compose + executor_manager + sandbox_base_image + sandbox_spec.md.
     SkillForge `Code Sandbox` 当前实现路径未 verify (内部?外部?).
     gVisor 是业内最高级别 isolation, RAGFlow 模板让 RAGFlow 用户能跑 Python/JavaScript 代码不爆安全.
     SkillForge 如果允许 skill 跑用户代码 (生成数据 / 执行业务逻辑), gVisor 是必备选项.
     工作量: 1-2 周 集成 + 1 周 SkillForge skill engine 适配.

  🟡 **R-RF-3 — Template-based Chunking 15 文档类型**(参考)
     `rag/app/{book,email,laws,manual,naive,one,paper,picture,presentation,qa,resume,table,tag,audio}.py`
     按文档类型选 chunking strategy (法律文档 ≠ 论文 ≠ 简历).
     SkillForge 当前无显式 chunking 概念 (memory 不是 doc-based), 但 skill marketplace 上的文档 + skill content
     可参考此模板做 type-aware chunking.
     工作量: 远期参考.

  🟡 **R-RF-4 — RAPTOR + Tree Query Decomposition**(进阶 retrieval pattern)
     `rag/raptor.py` Stanford 算法 + `rag/advanced_rag/tree_structured_query_decomposition_retrieval.py`.
     SkillForge MemoryService 单层 hybrid 检索, 进阶到 multi-hop / tree-organized 检索时,
     RAPTOR 是 production-grade reference.
     工作量: 远期, 看 SkillForge memory query 复杂度.

  🟢 **R-RF-5 — 双流派 GraphRAG (general + light)**(参考)
     `rag/graphrag/general/` (Microsoft 风) + `rag/graphrag/light/` (LightRAG 风) 都内置, 让用户选.
     **ICLR'26 GraphRAG-Benchmark 论文警示 KG 未必更好** — SkillForge 不必盲跟,
     但如果走 KG memory v3, RAGFlow 双流派给 reference.

  🟢 **R-RF-6 — OpenClaw 生态集成模式**(参考)
     2026-03-24 RAGFlow Skill on OpenClaw (clawhub.ai/yingfeng/ragflow-skill).
     SkillForge skill marketplace 如果对外开放第三方 RAG engine (RAGFlow/LlamaIndex/...),
     可参考这种集成 pattern (skill 包装 RAG engine API + 跑在用户 OpenClaw 内).

  **❌ 不该跟**:
  - 中文 NLP 一等公民 — SkillForge 走多语言路线, 不需要中文专用 tokenizer
  - 10+ 国语言 README — 商业化阶段决定, 不是技术问题
  - Go 组件混入 — SkillForge Java 栈, 不混语言
---

# Harness — RAGFlow 0.25.4

> RAGFlow 是业内 #2 most-starred RAG 项目(80.6k stars,仅次于 LangChain 125k),**deep document understanding** 是核心护城河。13 维详见 frontmatter。本页 narrative 重点:**RAGFlow 跟其他 RAG framework 的根本差异是 _"承认文档解析是 RAG 的第一性原理"_** —— 16 parser + table structure recognizer + gVisor sandbox 是 production engineering 答案。

---

## 1. 项目定位(一句话)

**Production-grade RAG engine + agent platform**(80.6k stars / Apache 2.0 / InfiniFlow / 2024 起步)。Tagline:_"leading open-source RAG engine that fuses cutting-edge RAG with Agent capabilities"_。

---

## 2. 为什么"deep document understanding"是 RAG framework 的胜负手

业内 RAG framework 默认假设是:_"给我 plain text,我做 chunking + embedding + retrieval"_。但 production 用户面对的真实数据是:

| 数据类型 | LangChain / LlamaIndex 默认 | RAGFlow 路径 |
|---|---|---|
| 扫描 PDF | 第三方 OCR 用户自配 | **PaddleOCR + MinerU 内置** |
| PDF 里的表格 | 单纯 text 流(行列丢失)| **`table_structure_recognizer.py` 行列识别 + 合并单元格** |
| 多列排版 PDF | text 顺序乱 | **`layout_recognizer.py` 版面识别先理顺** |
| PDF 里的图片 / 图表 | 跳过 / 用户自处理 | **`figure_parser.py` + CV model + 多模态 LLM** |
| 简历 / 法律文档 | 通用 chunking | **`rag/app/resume.py + laws.py` 专用模板** |
| Word / PPT / Excel | python-docx / openpyxl 简单提取 | **专用 parser + 保结构** |

**这就是 RAGFlow 80.6k stars 的真实原因** — 不是 algorithm novelty,是**承认"production 用户的文档是垃圾,RAG 入口必须先做深度结构化"** 的工程决心。

**对 SkillForge 启示**:SkillForge 当前 skill marketplace 含 doc(skill content 是 Markdown),没有 PDF / Word / Excel / 扫描件等垃圾数据来源。所以 RAGFlow deepdoc/ 不直接打到生产。**但**:如果 SkillForge 商业化阶段要做"客户上传业务文档"功能,RAGFlow deepdoc 是模板。

---

## 3. 四大独家技术点

### 3.1 16 文件格式 Parser + 10 Vision 模块(`deepdoc/`)

- 16 parser 含 mineru / paddleocr / docling / 4 类 office / 多媒体(audio / figure)
- 10 vision 含 layout_recognizer / table_structure_recognizer / OCR / postprocess

**业内独家完整深度文档理解栈**。LlamaIndex 159 reader 但 reader 是"调外部 API 接数据",RAGFlow deepdoc 是"在 framework 内做深度结构化解析"。

### 3.2 gVisor Sandbox(`agent/sandbox/`)

**业内 RAG framework 唯一显式 gVisor sandbox**:Docker Compose + executor_manager + base image + protocol spec。让 user-uploaded Python / JavaScript 代码 component 能在 production 安全跑。

**对 SkillForge skill marketplace 是直接打到生产的 P1**:如果 skill 允许用户在 skill 内嵌代码 / 业务逻辑,gVisor 是默认选项。详见 R-RF-2。

### 3.3 双流派 GraphRAG(`rag/graphrag/general/` + `rag/graphrag/light/`)

业内 RAG framework 一般选一个 GraphRAG 流派(LightRAG 或 Microsoft GraphRAG),RAGFlow **两个都内置**,让用户按数据规模 / 成本选。

⚠️ **ICLR'26 GraphRAG-Benchmark calibration**:_"GraphRAG frequently underperforms vanilla RAG on many real-world tasks."_ — RAGFlow 给选择,但不强推。

### 3.4 RAPTOR + Tree Query Decomposition(`rag/raptor.py` + `rag/advanced_rag/`)

业内除 LlamaIndex 外少有 production-grade RAPTOR 实现。RAGFlow 内置 Stanford 论文级算法 — 这是 framework "上行 academic-to-prod" 的诚意指标。

---

## 4. 跟 Dify 对照(同 production AI platform 赛道)

| 维度 | Dify | RAGFlow |
|---|---|---|
| 主战场 | 全栈 AI 应用平台(RAG + Agent + Workflow + Model Mgmt) | RAG engine + Agent canvas |
| Stars | 114k | 80.6k |
| 文档解析 | 通用 extractor(浅) | **16 parser + vision(深)** |
| Chunking | 2 splitter 文件 | **15 模板按文档类型** |
| GraphRAG | 不内置 | **双流派内置** |
| 代码沙箱 | Plugin 隔离(无 gVisor) | **gVisor production sandbox** |
| 多语言 README | 几个 | **10+ 国** |
| Stack | Flask + Celery + Next.js | Python + **Go 组件** + Next.js |
| OpenClaw 集成 | 无 | **2026-03 RAGFlow Skill on OpenClaw** |

**结论**:Dify 是"广度"(全栈 AI 平台),RAGFlow 是"深度"(RAG-specific 极致工程化)。**SkillForge 如果要选 production 路线参考,Dify 是产品定位对标,RAGFlow 是 RAG 引擎技术深度对标**。

---

## 5. 跟现有 wiki 关系

| 当前页 | 关系 | 新增点 |
|---|---|---|
| [[harness/dify]] | 同赛道对照 | 见 §4 表 |
| [[eval/llama_index]] | 互补 | LlamaIndex 抽象,RAGFlow 工程化 |
| [[harness/openclaw]] | **直接关联** | RAGFlow 主动给 OpenClaw 提供 official skill(2026-03-24)— 印证 OpenClaw ecosystem 协同 |
| [[concept/agent-memory]] | 不强相关 | RAGFlow memory module 后补(2025-12),路线偏 RAG-extended |
| [[concept/rag-design-space]] | **核心贡献者** | 16 parser + 15 chunking template + 双流派 GraphRAG + RAPTOR 是 RAG 工程化最广维度 |

---

## 6. R-RF 借鉴清单

详见 frontmatter `skillforge_takeaway`。**最高 ROI 是 R-RF-2 gVisor sandbox**(SkillForge skill engine 安全直接相关)。

| ID | 项 | 工作量 | ROI | 优先级 |
|---|---|---|---|---|
| **R-RF-1** | Deep document understanding 16 parser + vision | 远期(skill marketplace 升级时)| 中 — 商业化用 | 🟡 P2 |
| **R-RF-2** | **gVisor Sandbox 集成** | 2-3 周 | 高 — skill safety 直接打 | 🔴 P1 |
| **R-RF-3** | Template-based chunking 15 文档类型 | 远期 | 中 | 🟡 P2 |
| **R-RF-4** | RAPTOR + Tree Query Decomposition | 远期 | 中 — 高级 retrieval | 🟡 P2 |
| **R-RF-5** | 双流派 GraphRAG(general + light) | 远期 | 低 — KG 路线警示 | 🟢 P3 |
| **R-RF-6** | OpenClaw 生态集成模式 | 商业化决策 | 中 | 🟢 P3 |

---

## 7. 已知 post-ingest 风险

- `internal/`(Go 组件)未深 verify
- `memory/` 模块新(2025-12-26),内部细节未读
- `agent/canvas.py` Graph DSL 只读了 60 行,完整 schema 未深
- 大部分 README claim(MinerU/Docling 集成质量、table recognition 准确率)未 benchmark verify

---

## 8. 引用

- [[source/ragflow]] 完整源码锚点
- GitHub: `infiniflow/ragflow` 80.6k stars / Apache 2.0
- Cloud:cloud.ragflow.io / DeepWiki: deepwiki.com/infiniflow/ragflow
- **OpenClaw 生态**:clawhub.ai/yingfeng/ragflow-skill(2026-03)
- 跨页对照:[[harness/dify]] / [[eval/llama_index]] / [[harness/openclaw]] / [[concept/rag-design-space]]
