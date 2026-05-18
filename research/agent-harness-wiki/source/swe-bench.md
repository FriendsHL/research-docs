---
type: source
name: swe-bench
origin: "/Users/youren/myspace/SWE-bench/ (Princeton NLP, MIT)"
ingested_at: 2026-05-16
ingested_commit: f7bbbb2
ingested_commit_date: 2026-03-18
ingested_version_label: "(pip package `swebench`, monorepo 无统一 semver,以 commit 为准)"
contributed_to:
  - eval/swe-bench.md
  - concept/benchmark-datasets.md
  - concept/agent-eval.md
---

# Source — SWE-bench

> **Princeton NLP 出品的业内 coding benchmark 标杆**(MIT,ICLR 2024 Oral + ICLR 2025)。"Given a codebase + issue → LLM 生成 patch 解决"范式。**纯 dataset + Docker 评测 harness**,不是 platform / framework。

## 仓库结构概览

```
SWE-bench/
├── swebench/
│   ├── collect/                    # 从 PR 抓 issue+patch 构建 dataset
│   │   ├── get_top_pypi.py         # 拉 top PyPI 包
│   │   ├── print_pulls.py          # 列 PR
│   │   ├── get_tasks_pipeline.py   # 完整 task 抓取流水线
│   │   ├── build_dataset.py        # 构建 dataset
│   │   └── build_dataset_ft.py     # 构建 fine-tune dataset
│   ├── harness/                    # 评测核心(Docker + grading)
│   │   ├── constants/__init__.py   # MAP_REPO_VERSION_TO_SPECS_* 8 语言 specs
│   │   ├── run_evaluation.py       # 主入口:python -m swebench.harness.run_evaluation
│   │   ├── grading.py              # F2P / P2P 评分逻辑
│   │   ├── docker_build.py         # Docker image 构建
│   │   ├── docker_utils.py         # Docker 工具
│   │   ├── dockerfiles/            # Dockerfile 模板
│   │   ├── log_parsers/            # 测试日志解析(各语言不同)
│   │   ├── modal_eval/             # Modal cloud 评测
│   │   ├── prepare_images.py       # Image 预热
│   │   ├── remove_containers.py    # 清理 container
│   │   ├── reporting.py            # 报告生成
│   │   ├── test_spec/              # Test spec(repo + version + spec)
│   │   └── utils.py                # Harness 工具
│   ├── inference/                  # Model 推理
│   │   ├── run_api.py              # API 模型(OpenAI/Anthropic)
│   │   ├── run_llama.py            # 本地 Llama
│   │   ├── run_live.py             # 实时推理
│   │   ├── llamao/                 # Llama 工具
│   │   ├── make_datasets/          # 训练 dataset 构造
│   │   └── codellama_device_maps.json
│   ├── versioning/                 # 8 语言 repo 版本管理
│   │   ├── get_versions.py
│   │   ├── constants.py
│   │   └── extract_web/            # 抓 web 版本号(pvlib/sqlfluff/astropy/matplotlib)
│   └── resources/
├── docs/                           # 文档站(MkDocs)
├── tests/                          # 测试
├── pyproject.toml                  # PyPI 包定义
├── README.md (含日文 / 简中 / 繁中)
└── LICENSE (MIT)
```

## 关键 file:line 锚点

### 评分核心(grading.py + constants)
- `swebench/harness/constants/__init__.py:40-43` — **4 个 test category**:
  - `FAIL_TO_PASS = "FAIL_TO_PASS"`(主 metric:patch 应让原来失败的 test 通过)
  - `FAIL_TO_FAIL = "FAIL_TO_FAIL"`(patch 不应让失败 test 变 success-by-accident)
  - `PASS_TO_PASS = "PASS_TO_PASS"`(防 regression:patch 应保持通过 test)
  - `PASS_TO_FAIL = "PASS_TO_FAIL"`(regression detection:patch 不应让通过 test 变失败)
- `swebench/harness/grading.py:198-201` — `F2P score = FAIL_TO_PASS success / total`
- `swebench/harness/grading.py:208-212` — `P2P score = PASS_TO_PASS success / total`
- `swebench/harness/grading.py:246-279` — `get_eval_report` 输出 `{FAIL_TO_PASS, PASS_TO_PASS}` × `{success, failure}` 2×2 矩阵

### 8 语言支持(`harness/constants/__init__.py:136-144`)
```python
MAP_REPO_VERSION_TO_SPECS = {
    **MAP_REPO_VERSION_TO_SPECS_C,        # C 语言
    **MAP_REPO_VERSION_TO_SPECS_GO,       # Go
    **MAP_REPO_VERSION_TO_SPECS_JAVA,     # Java
    **MAP_REPO_VERSION_TO_SPECS_JS,       # JS
    **MAP_REPO_VERSION_TO_SPECS_PHP,      # PHP
    **MAP_REPO_VERSION_TO_SPECS_PY,       # Python(主,数量最多)
    **MAP_REPO_VERSION_TO_SPECS_RUBY,     # Ruby
    **MAP_REPO_VERSION_TO_SPECS_RUST,     # Rust
}
```

### Docker 评测约定(constants)
- `constants/__init__.py:71-73` — Docker:`DOCKER_PATCH="/tmp/patch.diff"` / `DOCKER_USER="root"` / `DOCKER_WORKDIR="/testbed"`
- `constants/__init__.py:74-76` — 输出文件:`LOG_REPORT="report.json"` / `LOG_INSTANCE="run_instance.log"` / `LOG_TEST_OUTPUT="test_output.txt"`
- `constants/__init__.py:80-86` — 状态 marker:`APPLY_PATCH_FAIL = ">>>>> Patch Apply Failed"` / `APPLY_PATCH_PASS = ">>>>> Applied Patch"` / `INSTALL_FAIL/PASS/TIMEOUT` / `RESET_FAILED` / `TESTS_ERROR`

### 关键 dataset(HuggingFace)
- `princeton-nlp/SWE-bench`(test split)— 主 dataset
- `princeton-nlp/SWE-bench_Lite` — 轻量版
- `princeton-nlp/SWE-bench_Verified` — **OpenAI Preparedness 团队人工验证 500 题**(2024-08)
- `princeton-nlp/SWE-bench_Multimodal` — ICLR 2025 多模态(图像 + 代码)

### 4 个变体演化历史(README News)
- 2024-01:SWE-bench 论文 ICLR 2024 Oral(`openreview.net/forum?id=VTF8yNQM66`)
- 2024-04:**SWE-agent** 配套 reference agent 发布(SOTA on full SWE-bench)
- 2024-06:Docker 全容器化评测(OpenAI Preparedness 支持)
- 2024-08:**SWE-bench Verified** 500 题人工验证(OpenAI Preparedness Part 2)
- 2025-01:SWE-bench Multimodal + sb-cli cloud 评测 + Modal 集成

### Inference 模块(3 种推理路径)
- `inference/run_api.py` — API 模型(OpenAI / Anthropic / Cohere 等,走 API)
- `inference/run_llama.py` — 本地 Llama(用 `codellama_device_maps.json` 多 GPU 分配)
- `inference/run_live.py` — 实时推理(开发调试用)
- `inference/make_datasets/` — 训练 dataset 构造(SWE-Llama 训练用)

### Collect 模块(从 PR 抓 issue+patch)
- `collect/get_top_pypi.py` — 拉 top PyPI 包(选 candidate repo)
- `collect/print_pulls.py` — 列 PR(过滤 issue + patch)
- `collect/get_tasks_pipeline.py` — **完整 task 抓取流水线**:repo → PR → issue + 测试 → instance
- `collect/build_dataset.py` — 把 task 构造成 SWE-bench instance(`instance_id` + `model_patch` 等字段)

### 资源要求(README)
- `x86_64` 机器,至少 120GB 磁盘 + 16GB RAM + 8 CPU
- `max_workers < min(0.75 * os.cpu_count(), 24)`
- ARM64(M-series Mac)支持实验性,需 `--namespace ''` 强制本地构建
- Docker Desktop 需 ~120GB 虚拟磁盘

### Cloud 评测
- **`sb-cli`** (`github.com/swe-bench/sb-cli`)— Princeton 自家 AWS-based cloud 评测工具
- **Modal**(`modal.com`)— Serverless Python cloud,Modal 跟 SWE-bench 团队合作

### 配套模型 + Retrieval
- **SWE-Llama**:7b / 13b + PEFT 变体(princeton-nlp HuggingFace)
- **3 种 BM25 retrieval**:13K / 27K / 40K / 50K(Llama tokens)
- **Oracle retrieval**:理想检索基线
- 用法:用 retrieval 提取相关代码 → 喂 LLM → 生成 patch

## 与 wiki 现有记录的关系

### 印证
- **Docker reproducibility** 印证 [[../eval/langfuse]] + [[../eval/opik]] 自托管 Docker / Helm 部署模式 — Docker 是 LLM eval 行业 reproducibility 事实标准
- **HuggingFace dataset 一等公民** 印证 [[../concept/prompt-optimization]] Opik 8 内置 benchmark dataset(gsm8k / truthful_qa / halu_eval 等)走 HuggingFace 模式

### 矛盾(无,SWE-bench 是 benchmark 不是 platform/framework,跟现有页面定位互补不冲突)

### 新增
- **eval/swe-bench.md**(新建,4 家 eval 里首个**纯 benchmark** 类别,跟 Langfuse/Opik platform 不同定位)
- **concept/benchmark-datasets.md**(新建)— **三层 eval 分类**:platform([[eval/langfuse]] / [[eval/opik]])/ framework([[harness/openclaw]] QA Character Eval)/ **benchmark**(SWE-bench / τ-bench / Opik 8 内置 / DSPy 内置)
- **[[concept/agent-eval]] 加 benchmark 层**:补 8 维设计空间里 dataset 来源维度
- **4-Test-Category 评分模型**(F2P / FAIL_TO_FAIL / PASS_TO_PASS / PASS_TO_FAIL):跟 Langfuse Score 数据抽象 / Opik OOP 多态都不同,是 **regression-aware 评分**(防 patch 引入 regression)

## 事实漂移 / 注意点

- **SWE-bench 是 dataset + harness 组合**,不是 platform。**不能跟 Langfuse / Opik 直接对比**,是不同 layer。
- **8 语言但 Python 主导**(数量最多,其他 7 语言是后续扩展):评估时 `MAP_REPO_VERSION_TO_SPECS_PY` 是主体
- **Verified 500 题是 OpenAI Preparedness 验证**(2024-08),业内争议较小,**生产环境 eval 应优先用 Verified**(原始 SWE-bench 含部分 unsolvable 题)
- `swebench` 是 PyPI 包名(pip install swebench),仓库名是 `SWE-bench`,paper 也用 SWE-bench
- **资源开销大**:120GB + 16GB + 8 CPU,**不是 CI/CD friendly**(对照 Promptfoo / Ragas 轻量 CI 集成)
- **没有 SDK**(对照 Langfuse Python/JS SDK,Opik Python/TS/Ruby SDK):SWE-bench 通过 CLI(`python -m swebench.harness.run_evaluation`)使用,通过 HuggingFace `load_dataset` 读 dataset
- 没有"自动晋升"或 production loop(对照 Langfuse `addDatasetRunItemsToEvalQueue`):SWE-bench 是 one-shot benchmark,跑完得 score 就结束
