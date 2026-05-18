---
name: ragflow
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: ragflow
version: 0.25.4
commit: 09d45046e5f46fb2300f8860a3819568d71916d3
license: Apache 2.0
stars: 80.6k
last_ingested_at: 2026-05-17
contributed_to:
  - "[[harness/ragflow]]"
  - "[[concept/rag-design-space]]"
---

# Source — RAGFlow 0.25.4 @ commit 09d45046

> infiniflow/ragflow — **80.6k stars** Apache 2.0,业内 #2 most-starred RAG-specific project(LangChain 125k / **RAGFlow 80.6k** / RAG topic 内最高)。**Tagline**:_"leading open-source RAG engine that fuses cutting-edge RAG with Agent capabilities"_。**Converged context engine + pre-built agent templates**。clone @ `/Users/youren/myspace/ragflow/` 131 MB。

---

## 1. Top-level 仓库结构

```
ragflow/
├── admin/                # 管理面
├── agent/                # Agent canvas (DSL) + 20+ component + gVisor sandbox
├── api/                  # API 层
├── common/               # 工具
├── conf/                 # 配置
├── deepdoc/              # ⭐ 深度文档理解 (16 parser + 10 vision)
├── docker/               # Docker 部署
├── docs/                 # 文档
├── helm/                 # k8s
├── internal/             # Go 组件 (go.mod / go.sum)
├── mcp/                  # MCP client + server
├── memory/               # Agent memory (2025-12-26 ship)
├── rag/                  # ⭐ 核心 RAG 引擎 (app/flow/graphrag/llm/nlp/prompts/raptor/advanced_rag)
├── web/                  # Next.js 前端
├── AGENTS.md + CLAUDE.md # 双语 dogfood (非 symlink)
├── README{,_zh,_tzh,_ja,_ko,_fr,_id,_pt_br,_ar,_tr}.md  # 10+ 语言 README 业内最广
├── go.mod / go.sum       # Go 组件
└── pyproject.toml + uv.lock
```

---

## 2. `deepdoc/` — 深度文档理解(业内独家护城河)[verified]

### 2.1 16 Parser 类型 [code: deepdoc/parser/]

```
docling_parser.py          # IBM Docling 集成
docx_parser.py             # Word
epub_parser.py             # EPUB
excel_parser.py            # Excel
figure_parser.py           # 图片
html_parser.py             # HTML
json_parser.py             # JSON
markdown_parser.py         # Markdown
mineru_parser.py           # ⭐ MinerU 集成(强 PDF parser)
opendataloader_parser.py
paddleocr_parser.py        # PaddleOCR(中文 OCR)
pdf_parser.py              # PDF
ppt_parser.py              # PowerPoint
resume/                    # 简历专用
tcadp_parser.py
txt_parser.py
```

**对照 LlamaIndex `node_parser/file/text/relational/` 3 大类、Dify `extractor/` 通用 extractor**:RAGFlow **按文件格式**显式建模 16 parser(尤其 mineru / paddleocr / docling 这 3 个是 production-grade 集成,业内 PDF/OCR 最强组合)。

### 2.2 10 Vision 模块 [code: deepdoc/vision/]

```
layout_recognizer.py            # 文档版面识别
ocr.py                          # OCR
operators.py                    # 视觉算子
postprocess.py                  # 后处理
recognizer.py                   # 元素识别
seeit.py                        # 可视化工具
t_ocr.py + t_recognizer.py      # 训练用
table_structure_recognizer.py   # ⭐ 表格结构识别
```

**Table structure recognizer 独家** — 把 PDF 里的 table 解析成结构化数据(行/列/合并单元格),而非简单 OCR 文本流。这是 LlamaIndex / Dify / RAG 业内"deep document understanding"的差距 point。

---

## 3. `rag/` — RAG 引擎 14 子模块 [verified]

```
rag/
├── advanced_rag/                                       # 进阶 RAG 算法
│   └── tree_structured_query_decomposition_retrieval.py  # ⭐ 树结构 query 分解检索
├── app/                                                # ⭐ 15 模板 chunking
│   ├── book.py / email.py / laws.py / manual.py /
│   ├── naive.py / one.py / paper.py / picture.py /
│   ├── presentation.py / qa.py / resume.py /
│   ├── table.py / tag.py / audio.py
├── flow/                                               # Orchestrable ingestion pipeline (2025-10-15)
│   ├── base.py / pipeline.py / chunker/ / extractor/ / parser/ / tokenizer/
├── graphrag/                                           # ⭐ 双流派 GraphRAG
│   ├── general/                                          # Microsoft GraphRAG 风
│   ├── light/                                            # LightRAG 风
│   ├── entity_resolution_prompt.py
│   ├── entity_resolution.py
│   ├── ner/
│   ├── phase_markers.py
│   ├── query_analyze_prompt.py
│   ├── search.py
│   └── utils.py
├── llm/                                                # 7 model 类型
│   ├── chat_model.py
│   ├── cv_model.py                                       # CV 模型
│   ├── embedding_model.py
│   ├── ocr_model.py
│   ├── rerank_model.py
│   ├── sequence2txt_model.py                             # seq2txt
│   └── tts_model.py
├── nlp/                                                # ⭐ 中文 NLP 一等公民
│   ├── rag_tokenizer.py
│   ├── surname.py                                        # 中文姓氏识别
│   ├── synonym.py                                        # 同义词
│   ├── term_weight.py                                    # 词权重 (BM25 用)
│   ├── query.py + search.py
├── prompts/                                            # 跨模块 prompt 库
├── raptor.py                                           # ⭐ RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval)
├── res/                                                # 资源
├── svr/                                                # 服务
├── settings.py
└── utils/
```

**4 个高级 RAG 范式内置**:
1. **RAPTOR**(`rag/raptor.py`)— Stanford 论文,递归式摘要构造树形索引
2. **Tree-structured query decomposition retrieval**(`rag/advanced_rag/tree_structured_query_decomposition_retrieval.py`)
3. **Two-style GraphRAG**(`rag/graphrag/general/` + `rag/graphrag/light/`)— Microsoft GraphRAG + HKUDS LightRAG 都内置
4. **Template-based chunking**(`rag/app/` 15 模板,按文档类型选 strategy)

---

## 4. `agent/` — Agent Canvas DSL + gVisor Sandbox [verified]

### 4.1 Graph DSL [code: agent/canvas.py:43+]

```python
class Graph:
    """
    dsl = {
        "components": {
            "begin": {
                "obj": {
                    "component_name": "Begin",
                    "params": {},
                },
                "downstream": ["answer_0"],
                "upstream": [],
            },
            "retrieval_0": {
                "obj": {
                    "component_name": "Retrieval",
                    "params": {}
                },
                "downstream": ["generate_0"],
            ...
```

**显式 DAG DSL**:`components` map + `downstream/upstream` 邻接 — 跟 Dify `node_factory.py` 类似但更声明式(纯 JSON / dict)。

### 4.2 20+ Workflow Components [code: agent/component/]

```
agent_with_tools.py      # Agent (含 tool)
base.py                  # 基类
begin.py                 # 入口
categorize.py            # 分类
data_operations.py       # 数据操作
docs_generator.py        # 文档生成
excel_processor.py       # Excel 处理
exit_loop.py             # 跳出循环
fillup.py                # 填空
invoke.py                # 外部调用
iteration.py + iterationitem.py    # 迭代
list_operations.py
llm.py                   # LLM call
loop.py + loopitem.py    # 循环
message.py
string_transform.py
switch.py                # 分支
variable_aggregator.py
variable_assigner.py
```

**显式控制流 primitive**:Iteration / Loop / Switch / Exit_loop 是 workflow-as-program 风(对照 Dify 7 内置 node + graphon 外部 / LangGraph 状态机)。

### 4.3 gVisor Sandbox [code: agent/sandbox/]

```
sandbox/
├── client.py
├── docker-compose.yml         # 沙箱 Docker 部署
├── executor_manager/          # 执行管理
├── providers/                 # Python / JavaScript 等 runtime
├── result_protocol.py
├── sandbox_base_image/        # Docker 基础镜像
├── sandbox_spec.md            # 协议规范
└── tests/
```

**README:_"gVisor: Required only if you intend to use the code executor (sandbox) feature of RAGFlow."_**

**gVisor is Google's container-aware kernel for isolation** — Codex 用了类似路径,业内 RAG framework 用 gVisor 做 code execution sandbox 是**最高级别安全隔离**(对照 OpenAI Codex 3 平台 sandbox / Dify 无独立 sandbox / LlamaIndex 无 sandbox / mem0 无)。

---

## 5. 关键 verified 锚点

### 5.1 双流派 GraphRAG [code: rag/graphrag/]

`general/`(Microsoft GraphRAG 风,community detection + hierarchical summarization)+ `light/`(HKUDS LightRAG 风,简单快速)— 用户**根据数据规模 / 成本预算**选流派,业内独家提供选择。

### 5.2 中文 NLP 一等公民 [code: rag/nlp/]

```
rag_tokenizer.py    # 中英文 tokenizer
surname.py          # 中文姓氏识别
synonym.py          # 同义词扩展
term_weight.py      # 词权重 (BM25 score 用)
query.py + search.py
```

业内大部分 RAG framework 的 NLP 是英文中心,RAGFlow 中文一等公民(自家在国内市场护城河)。

### 5.3 OpenClaw Ecosystem 集成 [README:91]

> _"2026-03-24 [RAGFlow Skill on OpenClaw](https://clawhub.ai/yingfeng/ragflow-skill) — Provides an official skill for accessing RAGFlow datasets via OpenClaw."_

**RAGFlow 主动跟 OpenClaw 生态打通** — 跟本 wiki 之前 ingest 的 [[harness/openclaw]] 印证。"RAG engine + agent harness ecosystem" 协同的 production 实例。

### 5.4 Memory Module 时间线 [README:92]

> _"2025-12-26 Supports 'Memory' for AI agent."_

**晚于核心 RAG 工程化** — 印证 RAGFlow 路线是"RAG 优先,memory 后补",跟 mem0 / Letta 等 memory-first 路线相反。

### 5.5 RAPTOR [code: rag/raptor.py]

**Recursive Abstractive Processing for Tree-Organized Retrieval** — Stanford 论文级算法内置实现,递归构造文档摘要树,多层粒度检索。**业内除 LlamaIndex(也有 TreeIndex)外少有 production-grade RAPTOR 实现**。

### 5.6 RAG 工程关键时间线(verified from README)

- 2025-08-01 Agentic workflow + MCP
- 2025-08-08 GPT-5 series
- 2025-10-15 **Orchestrable ingestion pipeline**(`rag/flow/`)
- 2025-10-23 **MinerU + Docling** 文档解析
- 2025-11-12 Confluence / S3 / Notion / Discord / Google Drive sync
- 2025-12-26 **Agent memory** ship
- 2026-03-19 Multi-modal model 处理图像
- 2026-03-24 RAGFlow OpenClaw Skill
- 2026-04-24 DeepSeek v4

---

## 6. 关键文件清单(供后续 query)

| 路径 | 主题 |
|---|---|
| `deepdoc/parser/*` | 16 文件格式 parser |
| `deepdoc/vision/{layout,table_structure}_recognizer.py` | 版面 + 表格结构识别 |
| `rag/app/*` | 15 文档类型 chunking 模板 |
| `rag/graphrag/{general,light}/` | 两流派 GraphRAG 实现 |
| `rag/raptor.py` | RAPTOR 算法 |
| `rag/advanced_rag/tree_structured_query_decomposition_retrieval.py` | 树结构 query 分解 |
| `rag/nlp/*` | 中文 NLP 全套 |
| `rag/flow/{pipeline,base}.py + chunker/ + extractor/ + parser/ + tokenizer/` | 可编排 ingestion pipeline |
| `agent/canvas.py` | Graph DSL Class |
| `agent/component/*` | 20+ workflow component |
| `agent/sandbox/{client.py, docker-compose.yml, executor_manager/, sandbox_spec.md}` | **gVisor code execution sandbox** |
| `internal/` (Go) | Go 组件(具体未深) |

---

## 7. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/ragflow/](file:///Users/youren/myspace/ragflow/) 131 MB
- GitHub: `infiniflow/ragflow` 80.6k stars / Apache 2.0
- Cloud:cloud.ragflow.io / DeepWiki: deepwiki.com/infiniflow/ragflow
- 10+ 国语言 README(en/zh-cn/zh-tw/ja/ko/fr/id/pt-br/ar/tr)
- **OpenClaw 生态集成**:clawhub.ai/yingfeng/ragflow-skill
- 衍生 wiki:[[harness/ragflow]] / [[concept/rag-design-space]]
