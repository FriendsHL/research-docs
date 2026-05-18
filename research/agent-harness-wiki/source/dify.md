---
name: dify
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: dify
version: latest @ commit 9d0906c684
commit: 9d0906c684c4ccbb9ea665fb6aa3fcf562aabe57
license: Dify Open Source License (Apache 2.0 + 商业限制)
stars: 114k+
last_ingested_at: 2026-05-17
contributed_to:
  - "[[harness/dify]]"
  - "[[concept/rag-design-space]]"
---

# Source — Dify @ commit 9d0906c

> langgenius/dify — 业内最大的 production AI 应用平台之一(114k stars,仅次于 LangChain 125k)。LF AI & Data Foundation 旗下,**Production AI Application Platform** 定位:RAG + Agent + Workflow + Model Management 全栈 + Visual UI。clone @ `/Users/youren/myspace/dify/` 156 MB。

---

## 1. Top-level 仓库结构

```
dify/
├── api/                      # Flask + DDD 后端 (560 Python files)
├── web/                      # Next.js + TS 前端
├── dify-agent/               # Agent 后端服务
├── docker/                   # Docker 部署
├── dev/                      # 开发工具
├── docs/                     # 文档
├── e2e/                      # E2E 测试
├── AGENTS.md                 # → 双语 dogfood (CLAUDE.md symlink)
├── CLAUDE.md → AGENTS.md     # 业内独家 symlink 复用
├── .codex                    # Codex dogfood (空文件 marker)
├── .gemini/                  # Gemini CLI dogfood
├── .claude/skills/           # Claude Code 6 skills
├── .agents/skills/           # 7 generic agent skills
├── .vite-hooks/              # Vite hooks
└── pyproject.toml            # uv-based Python
```

**多 harness dogfood 业内罕见**:Claude Code + Codex + Gemini + 自家 .agents/ 同时存在,印证 Dify 团队**实际用多家 agent 工具开发**,不是单一 vendor lock-in。

---

## 2. `api/core/` — 31 模块 [verified]

按职责分组:

| 类 | 模块 | 内容 |
|---|---|---|
| **App** | `app/` | `apps/` 多种 app 类型 + `task_pipeline/` + `app_config/` + `features/` + `file_access/` + `layers/` + `llm/` + `workflow/` 子 |
| **Agent** | `agent/` | **5 个 runner**:`base_agent_runner.py` / `cot_agent_runner.py`(CoT 基类)/ `cot_chat_agent_runner.py`(chat 变体)/ `cot_completion_agent_runner.py`(completion 变体)/ `fc_agent_runner.py`(function calling)+ `strategy/` + `output_parser/` + `prompt/` + `plugin_entities.py` |
| **Workflow** | `workflow/` | `node_factory.py` 188 行(`NodeType` × `node_version` 双键 lazy registry,`_LazyNodeTypeClassesMapping:172-213` 实现 cache snapshot + override)+ `nodes/` 内置 7 类(agent/datasource/knowledge_index/knowledge_retrieval/trigger_plugin/trigger_schedule/trigger_webhook)+ **`graphon.nodes` 外部包**(`node_factory.py:107`)+ `human_input_*.py` 3 文件 human-in-the-loop first-class |
| **RAG** | `rag/` | **14 子模块**(详见 §3)+ 完整工程化 pipeline |
| **Tools** | `tools/` | **6 tool 类型**:`builtin_tool/` / `custom_tool/` / `mcp_tool/` / `plugin_tool/` / **`workflow_as_tool/`**(workflow 作为 tool 独家)+ `__base/` + `tool_manager.py` + `tool_label_manager.py` + `signature.py` |
| **MCP** | `mcp/` | MCP 客户端 |
| **Memory** | `memory/` | **极简 1 文件** `token_buffer_memory.py` — Dify 不做长期 memory,长期 context 都走 RAG knowledge base |
| **Datasource** | `datasource/` | **5 类**:`local_file/` / `online_document/` / `online_drive/` / `website_crawl/` + `__base/` + `datasource_file_manager.py` + `datasource_manager.py` |
| **Model** | `model_manager.py` / `provider_manager.py` | **100+ LLM provider** 管理(LangChain 之外业内最广) |
| **Plugin** | `plugin/` | `backwards_invocation/` / `endpoint/` / `impl/` — 插件 host runtime |
| **Ops / Telemetry** | `ops/` / `telemetry/` / `logging/` | Production observability |
| **Moderation** | `moderation/` | 安全审核 |
| **Indexing** | `indexing_runner.py` | RAG indexing 入口 |
| **LLM generator** | `llm_generator/` | LLM 调用统一封装 |
| **Repositories** | `repositories/` | DDD Repository 层 |
| **Schemas / Entities / Errors / Context / Helper / Callback_handler / Extension / External_data_tool / Trigger / Db / Hosting_configuration.py / Base** | 配套基础设施 | DDD 架构清晰 |

**`api/services/rag_pipeline/`** 另起 service 层(`api/core` 之外):
- `pipeline_generate_service.py` / `pipeline_template/` / `rag_pipeline_dsl_service.py` / `rag_pipeline_manage_service.py` / `rag_pipeline_task_proxy.py` / `rag_pipeline_transform_service.py` / `rag_pipeline.py` / `transform/` + `entity/`
- **RAG pipeline 有专门 DSL 服务层** + transform pipeline

---

## 3. `api/core/rag/` — 14 子模块详解 [verified]

```
api/core/rag/
├── cleaner/                # 输入清洗
├── data_post_processor/    # 检索结果后处理
├── datasource/             # 数据源接入
├── docstore/               # 文档存储
├── embedding/              # 嵌入计算
│   ├── cached_embedding.py # 嵌入缓存
│   ├── embedding_base.py   # 基类
│   └── retrieval.py        # 嵌入检索
├── entities/               # DDD 实体
├── extractor/              # 各文件格式抽取
├── index_processor/        # 索引处理器
│   ├── index_processor_base.py
│   ├── index_processor_factory.py
│   ├── index_processor.py
│   ├── constant/
│   └── processor/
├── models/                 # 数据模型
├── pipeline/               # 仅 queue.py (Pipeline 服务在 api/services/rag_pipeline/)
├── rerank/                 # 重排序
│   ├── rerank_base.py
│   ├── rerank_factory.py
│   ├── rerank_model.py     # 模型重排
│   ├── rerank_type.py      # RerankMode 2 枚举 (reranking_model / weighted_score)
│   ├── weight_rerank.py    # 加权重排
│   └── entity/
├── retrieval/              # 检索方法
│   ├── dataset_retrieval.py
│   ├── retrieval_methods.py  # RetrievalMethod 4 枚举
│   ├── output_parser/
│   ├── router/
│   └── template_prompts.py
├── splitter/               # 文本切分
│   ├── fixed_text_splitter.py
│   └── text_splitter.py
└── summary_index/          # 摘要索引(独家)
```

---

## 4. 关键 verified 锚点

### 4.1 RetrievalMethod 4 枚举 [code: api/core/rag/retrieval/retrieval_methods.py:1-20]

```python
from enum import StrEnum

class RetrievalMethod(StrEnum):
    SEMANTIC_SEARCH = "semantic_search"
    FULL_TEXT_SEARCH = "full_text_search"
    HYBRID_SEARCH = "hybrid_search"
    KEYWORD_SEARCH = "keyword_search"

    @staticmethod
    def is_support_semantic_search(retrieval_method: str) -> bool:
        return retrieval_method in {
            RetrievalMethod.SEMANTIC_SEARCH,
            RetrievalMethod.HYBRID_SEARCH,
        }
```

**4 路径 + 静态 helper 显式判断什么 method 支持什么 search**(防 retrieval method 跟下游能力错配)。

### 4.2 RerankMode 2 枚举 [code: api/core/rag/rerank/rerank_type.py]

```python
class RerankMode(StrEnum):
    RERANKING_MODEL = "reranking_model"  # 用 reranker model (Cohere 等)
    WEIGHTED_SCORE = "weighted_score"     # 加权融合多路 retrieval score
```

**对照** SkillForge `MemoryService.java:853-908` rrfMerge 是 RRF(Reciprocal Rank Fusion),Dify weighted_score 是另一种融合策略。

### 4.3 Workflow Node 注册 [code: api/core/workflow/node_factory.py:104-220]

```python
@lru_cache(maxsize=1)
def register_nodes() -> None:
    """Import production node modules so they self-register with ``Node``."""
    _import_node_package("graphon.nodes")           # 外部包
    _import_node_package("core.workflow.nodes")     # 内置

def resolve_workflow_node_class(*, node_type: NodeType, node_version: str) -> type[Node]:
    node_mapping = get_node_type_classes_mapping().get(node_type)
    if not node_mapping:
        raise ValueError(f"No class mapping found for node type: {node_type}")
    latest_node_class = node_mapping.get(LATEST_VERSION)
    matched_node_class = node_mapping.get(node_version)
    node_class = matched_node_class or latest_node_class
```

**双维度索引** `NodeType × node_version` → `Node class`。**`_LazyNodeTypeClassesMapping`(`:172-213`)** 是 `MutableMapping` 实现:cache snapshot + delete tracking + override — workflow 节点可在 runtime 动态替换 / 隐藏。

### 4.4 Workflow_as_tool [code: api/core/tools/workflow_as_tool/]

**Dify 独家**:把已编辑的 workflow 当作 tool 注册给 agent 调用 — 实现 nested workflow composition。其他 RAG framework 没有这个 abstraction。

### 4.5 Human-in-the-loop 3 文件 [code: api/core/workflow/]

- `human_input_adapter.py` — 用户输入适配器
- `human_input_forms.py` — 表单 schema
- `human_input_policy.py` — 输入策略(何时打断 / 何时只 hint)

**workflow 内 human-in-the-loop 一等公民**,跟 SkillForge `AskUserQuestion` 类似但是是 workflow-level 抽象.

### 4.6 5 Agent Runner [code: api/core/agent/]

| Runner | 路径 | 角色 |
|---|---|---|
| `base_agent_runner.py` | 基类 | Common abstractions |
| `cot_agent_runner.py` | CoT 基类 | Chain-of-Thought 基础 |
| `cot_chat_agent_runner.py` | CoT chat | Chat-style CoT |
| `cot_completion_agent_runner.py` | CoT completion | Completion-style CoT |
| `fc_agent_runner.py` | Function calling | OpenAI-style function calling |

**显式分离 CoT vs Function Calling 路线** — 大部分 framework 现在偏向 function calling,Dify 保留 CoT 路线给 prompt-only model(老模型 / 国产模型部分)。

---

## 5. 多 harness dogfood 证据

`.claude/skills/` 6 个 Claude Code skills:
- backend-code-review / component-refactoring / e2e-cucumber-playwright / frontend-code-review / frontend-query-mutation / frontend-testing

`.agents/skills/` 7 个 generic agent skills:
- backend-code-review / component-refactoring / e2e-cucumber-playwright / frontend-code-review / frontend-testing / how-to-write-component / tailwind-css-rules

`.codex`(空 marker)/ `.gemini/`(配置)/ `.cursor/`(隐含)/ `CLAUDE.md -> AGENTS.md` symlink

**用 7 个标准化 skill 覆盖 backend review / component refactor / E2E test / frontend review / 写组件 / Tailwind** — 这套 dogfood 自己开发自己,印证 Dify 团队 production 流程跟它给客户的产品是**同套**。

---

## 6. 跟其他 wiki 已 ingest 项目对比

| | Dify | Langfuse | Opik | RAGFlow(待 ingest) |
|---|---|---|---|---|
| 定位 | Production AI App Platform | Observability + Eval | Observability + Eval + Optimization | RAG Engine |
| RAG 内置 | ✅ 14 子模块 | ❌(eval-side 不做 RAG)| ❌(同前)| ✅(更专注 chunking) |
| Workflow 内置 | ✅ 完整(NodeType × version)| ❌ | ❌ | ❌ |
| Agent 内置 | ✅ 5 runner + plugin | ❌ | ❌ | ✅(部分) |
| Tool 类型 | 6(workflow_as_tool 独家)| - | - | - |
| Stack | Flask + Celery + Redis + DDD + Next.js | TS pnpm Web+Worker | Java Spring + ClickHouse + React | Python + JS |

---

## 7. 关键文件清单(供后续 query)

| 路径 | 主题 |
|---|---|
| `api/core/workflow/node_factory.py` 188+ 行 | NodeType × version 双键 lazy registry |
| `api/core/rag/retrieval/retrieval_methods.py` | 4 retrieval method |
| `api/core/rag/rerank/rerank_type.py` | 2 rerank mode |
| `api/core/rag/splitter/` | text_splitter + fixed_text_splitter |
| `api/core/rag/index_processor/` | factory + base + processor/ |
| `api/services/rag_pipeline/` | DSL service + transform + manage |
| `api/core/agent/cot_*.py + fc_agent_runner.py` | 5 agent runner |
| `api/core/tools/workflow_as_tool/` | workflow 作为 tool 独家 |
| `api/core/workflow/human_input_*.py` | human-in-the-loop 3 文件 |
| `api/core/memory/token_buffer_memory.py` | 唯一 memory 文件(极简) |
| `api/core/datasource/{local_file,online_document,online_drive,website_crawl}/` | 5 datasource type |
| `.claude/skills/` + `.agents/skills/` | 6+7 dogfood skills |
| `AGENTS.md` + `CLAUDE.md` symlink | 多 harness dogfood marker |

---

## 8. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/dify/](file:///Users/youren/myspace/dify/) 156 MB
- LF 项目:`langgenius-dify` health score badge
- 衍生 wiki:[[harness/dify]](13 维 narrative)/ [[concept/rag-design-space]] 后续合并
