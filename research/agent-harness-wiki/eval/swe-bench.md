---
type: eval-framework
name: swe-bench
status: active
language: python

# === 版本时效 ===
last_ingested_version: "PyPI `swebench` @ commit f7bbbb2 (2026-03-18)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/swe-bench]]"

# === 9 维度 ===
target: |
  **业内 coding benchmark 标杆**(ICLR 2024 Oral + ICLR 2025)。Given codebase + issue → LLM 生成 patch 解决。**纯 benchmark + Docker 评测 harness**(不是 platform / framework)。
  覆盖 **8 语言**(`harness/constants/__init__.py:136-144`):C / Go / Java / JS / PHP / Python(主)/ Ruby / Rust
  [verified-from-source]

dataset: |
  **4 个 HuggingFace 公开 dataset**:
  - `princeton-nlp/SWE-bench`(主 test split)
  - `princeton-nlp/SWE-bench_Lite`(轻量)
  - **`princeton-nlp/SWE-bench_Verified`**(500 题 OpenAI Preparedness 人工验证,2024-08)— **production 优先用**
  - `princeton-nlp/SWE-bench_Multimodal`(ICLR 2025,图像 + 代码)

  **自家 collect 流水线**(`swebench/collect/`):`get_top_pypi.py` → `print_pulls.py`(过滤 PR + issue + 测试)→ `get_tasks_pipeline.py` → `build_dataset.py`
  [verified-from-source]

scorer: |
  **Regression-aware 4 test category**(`constants/__init__.py:40-43`):
  - `FAIL_TO_PASS`(主 metric:原失败 test 应通过)
  - `FAIL_TO_FAIL`(原失败 test 不应 success-by-accident)
  - `PASS_TO_PASS`(防 regression:原通过 test 应保持通过)
  - `PASS_TO_FAIL`(regression detection:原通过 test 不应失败)

  **精确率**(`grading.py:198-212`):
  - `F2P score = FAIL_TO_PASS success / total`
  - `P2P score = PASS_TO_PASS success / total`

  跟 Langfuse Score 数据抽象 / Opik OOP 多态都不同 — 这是 **regression-aware code patch eval** 专属设计
  [verified-from-source]

reproducibility: |
  - **Docker 全容器化**(2024-06 OpenAI Preparedness 支持)— DockerHub images(`hub.docker.com/u/swebench`)
  - **状态 marker**(`constants:80-86`):`APPLY_PATCH_FAIL/PASS` / `INSTALL_FAIL/PASS/TIMEOUT` / `RESET_FAILED` / `TESTS_ERROR`
  - **`DOCKER_PATCH="/tmp/patch.diff"` / `DOCKER_USER="root"` / `DOCKER_WORKDIR="/testbed"`**(`constants:71-73`)
  - 资源要求:**120GB 磁盘 + 16GB RAM + 8 CPU**,ARM64 实验性需 `--namespace ''`
  [verified-from-source]

coverage: |
  - 覆盖:8 语言 real-world GitHub issue patch eval
  - **配套 SWE-agent**(2024-04,SOTA on full SWE-bench)
  - **3 种 BM25 retrieval**(13K / 27K / 40K / 50K Llama tokens)+ Oracle 理想检索基线
  - **SWE-Llama 7b/13b + PEFT**(princeton-nlp HuggingFace)训练模型

  **不覆盖**:
  - 不是 multi-turn / conversation(对照 [[tau-bench]])
  - 不是 agentic tool use(对照 [[../eval/opik]] 8 内置 benchmark 偏 QA / reasoning)
  - 不是 RAG(对照 [[ragas]])
  [verified-from-source]

production_loop: |
  **One-shot benchmark**,跑完得 score 结束。
  没有"自动晋升"或 production loop(对照 [[../eval/langfuse]] `addDatasetRunItemsToEvalQueue`)。
  Leaderboard 在 `swebench.com`,论文式公开排名。
  [verified-from-source]

automation: |
  - **CLI 主接口**:`python -m swebench.harness.run_evaluation --predictions_path X --dataset_name Y --run_id Z`
  - **Cloud 评测 2 路**:
    - **`sb-cli`**(`github.com/swe-bench/sb-cli`)— Princeton 自家 AWS-based
    - **Modal**(`modal.com`)— Serverless Python
  - **`--max_workers`** 并行 `< min(0.75 * os.cpu_count(), 24)`
  - 输出:`logs/build_images` + `logs/run_evaluation` + `evaluation_results/`
  [verified-from-source]

cost: |
  - Per-eval 主要成本:**Docker image build + LLM inference**
  - SWE-bench Verified 500 题在 GPT-4o 上跑约 $50-100,Claude 3.5 Sonnet 类似
  - **没有 SDK 跟 cost tracker**(对照 [[../eval/langfuse]] / [[../eval/opik]] 内建 cost)
  - 用户自管成本
  [verified-from-source]

skillforge_takeaway: |
  ## 可借鉴(2 条)
  1. **Regression-aware 4 test category 评分模型**(F2P / FAIL_TO_FAIL / PASS_TO_PASS / PASS_TO_FAIL)— SkillForge **不能完全跟**(coding benchmark 专属),但 **"防 regression"思路**值得借鉴给 SkillForge 的 prompt A/B 自动晋升 + canary 灰度回流(SKILL-CANARY-ROLLOUT V2 已交付)
  2. **Docker 全容器化评测 reproducibility** + 状态 marker(`APPLY_PATCH_FAIL/PASS` 等)— 工程模板可学,SkillForge eval 也可用容器化保证可复现

  ## 业内验证(3 条)
  - **HuggingFace dataset 一等公民** 印证业内 benchmark 选型
  - **SWE-bench Verified 500 题 OpenAI 人工验证** 印证 **production eval 必须 trustworthy dataset**
  - **SWE-agent 是配套 reference agent** 印证 benchmark + 参考 agent 模式

  ## SkillForge 不必跟(3 条)
  - **120GB 磁盘 + 16GB RAM 资源要求** — SkillForge eval 不该走这么重的路线
  - **One-shot benchmark 模式** — SkillForge 已是 production loop 派(飞轮 V1-V4),不需要 SWE-bench 风格
  - **coding patch 范式** — SkillForge 不是 pure coding agent,定位更广

  ## 关键 insight
  **SWE-bench 不是 SkillForge 该集成的 dataset**(8 个内置 benchmark 应该选 QA/reasoning 类如 gsm8k / truthful_qa,不是 coding patch),但**"防 regression"思路 + Docker reproducibility** 工程模板可借鉴。

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **业内 coding benchmark 标杆**(ICLR 2024 Oral)— Langfuse / Opik / Phoenix 是 platform,SWE-bench 是 dataset+harness,**完全不同 layer**
  2. **Regression-aware 4 test category** — F2P 主 metric + P2P 防 regression + 2 个 edge case(FAIL_TO_FAIL / PASS_TO_FAIL),业内独家
  3. **8 语言原生支持**(C/Go/Java/JS/PHP/Python/Ruby/Rust)— `MAP_REPO_VERSION_TO_SPECS_*` 各语言独立 specs
  4. **3 个变体 + Multimodal**:Lite / Verified / Multimodal 满足不同场景
  5. **OpenAI Preparedness 团队 2x 介入**:2024-06 Docker harness + 2024-08 Verified 500 题人工验证

  ## 优势 (strengths)
  1. **Verified 500 题 trustworthy** — 真人验证可解,排除原始 dataset noise
  2. **配套 SWE-agent + SWE-Llama + BM25 retrieval** 完整 ecosystem
  3. **Cloud 评测 2 路**(sb-cli AWS + Modal)降低用户门槛
  4. **状态 marker 字符串** 清晰区分 patch apply / install / reset / tests 4 阶段失败

  ## 哪些点做得好 (what_done_well)
  1. **4 test category enum** 直接当 dict key 用(`constants:40-43`)— 简单工程实现 regression detection
  2. **`MAP_REPO_VERSION_TO_SPECS_*` 8 个 dict 合并** 用 `{**a, **b}` 风格(`constants:137-144`)— Python idiomatic
  3. **`DOCKER_PATCH="/tmp/patch.diff"` 硬编码约定** — 让 patch 工具(`apply_patch` 跟 Codex / opencode 同款)有共同地址
---

# SWE-bench(Princeton NLP, MIT)

## 一句话定位
业内 coding benchmark 标杆,**Given codebase + issue → LLM 生成 patch 解决**。8 语言 Docker 评测 harness + 4 个 HF dataset 变体(Lite/Verified/Multimodal)+ SWE-agent reference。**纯 benchmark,不是 platform**。

## 起源
- 2024-01 ICLR 2024 Oral(Princeton NLP)
- 2024-04 SWE-agent 配套(SOTA)
- **2024-06 Docker harness 全容器化**(OpenAI Preparedness)
- **2024-08 Verified 500 题**(OpenAI Preparedness Part 2)
- 2025-01 Multimodal + sb-cli + Modal

## 工程关键
详见 frontmatter。**唯一对 SkillForge actionable 借鉴**:Regression-aware 评分模型 + Docker reproducibility 模板。

## 跟 wiki 现有

详见 [[../concept/benchmark-datasets]](待 ingest 中)— **eval 三层分类**:platform / framework / benchmark,SWE-bench 是 benchmark 层。SkillForge 已是 platform 层(超过 SWE-bench layer)。

## 待回答
- SWE-bench Verified 是否真覆盖 SkillForge 关心的"通用 agent 失败 pattern"?推断:**不太相关**(SkillForge 不是 pure coding agent)
- 是否值得 SkillForge 跑 SWE-bench Verified 一次?**仅作 baseline reference**,实际项目无需
