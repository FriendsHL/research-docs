---
type: eval-framework
name: promptfoo
status: active
language: typescript

# === 版本时效 ===
last_ingested_version: "0.121.11 @ commit 7279113 (2026-05-15)"
last_ingested_at: 2026-05-16
community_known_newer: false
known_post_ingest_features: []

sources:
  - "[[../source/promptfoo]]"

# === 9 维度 ===
target: |
  **CLI-first LLM eval + red teaming framework**(MIT,promptfoo.dev,**Powers LLM apps serving 10M+ users in production**)。
  独家定位:**100% 本地跑**("LLM evals run 100% locally - your prompts never leave your machine")+ **CI/CD-first**(GitHub Action `code-scan-action/` + `npm test` 风格 fail PR)。
  跟 4 家 eval 路线对比:
  - [[../eval/langfuse]] / [[../eval/opik]] = **platform**(SaaS UI)
  - [[swe-bench]] / [[tau-bench]] = **benchmark**(dataset + harness)
  - [[dspy]] = **framework**(prompt-as-program)
  - **Promptfoo = CLI eval(第 4 层)** + red teaming
  [verified-from-source]

dataset: |
  - 用户在 `promptfooconfig.yaml` 定义 `tests:` 数组(每个 test 含 `vars` 跟 `assert`)
  - 支持 CSV / JSON / 远程 URL 导入
  - **Drizzle ORM + SQLite**(`drizzle/` migrations)本地持久化
  - 没有平台级 dataset 实体(对照 [[../eval/langfuse]] Dataset)
  - 跟 HuggingFace `datasets` 集成可通过 Python provider
  [verified-from-source]

scorer: |
  **50+ 内置 assertion 类型**(`src/assertions/`):

  ### 按类别(`source/promptfoo.md` 完整列出)
  - **精确**:equals / contains / startsWith / regex / wordCount
  - **结构**:json / xml / html / sql
  - **多语言判定**:javascript / python / ruby / webhook
  - **NLP metric**:bleu / rouge / meteor / gleu / ngrams / levenshtein / perplexity
  - **LLM-as-judge**:llmRubric / modelGradedClosedQa / factuality / `g-eval`(`geval.ts`)/ searchRubric
  - **Semantic**:similar(embedding 距离)
  - **RAG**:answerRelevance / contextFaithfulness / contextRecall / contextRelevance(对照 [[ragas]])
  - **Cost / Latency**:cost / latency / finishReason
  - **Tool**:toolCallF1 / functionToolCall
  - **Trace 归因**:**`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration`**(独家:trace span 直接当 assertion 输入)
  - **Trajectory**:trajectory / trajectoryUtils(多轮校验)
  - **Safety**:moderation / refusal / redteam / guardrails
  - **Classifier**:classifier(ML 分类器)
  - **专门**:skill / synthesis / citation / openai / pi

  ### 模式
  - **Assertion = grading function**(yaml 配置 + TS 实现)
  - 用户在 `promptfooconfig.yaml` 写:
    ```yaml
    tests:
      - vars:
          question: "What is 2+2?"
        assert:
          - type: contains
            value: "4"
          - type: llmRubric
            value: "Answer should be concise"
    ```
  [verified-from-source]

reproducibility: |
  - **本地优先 + cache**(README "Fast, with features like **live reload and caching**")
  - **Drizzle ORM SQLite** 本地持久化历史 eval results
  - **Biome formatter**(替 Prettier+ESLint)— 代码格式一致
  - **`knip.json`** — Unused exports 检测,防 dead code 飘移
  - **Vitest** 测试
  - GitHub Action 在 CI 跑(`code-scan-action/`)
  [verified-from-source]

coverage: |
  ## 5 大支柱(README "What can you do with Promptfoo?")
  1. **Test prompts and models** — automated evaluations
  2. **Secure LLM apps** — red teaming + vulnerability scanning(25+ 攻击策略)
  3. **Compare models side-by-side**(OpenAI / Anthropic / Azure / Bedrock / Ollama / 50+ provider)
  4. **Automate checks** in CI/CD
  5. **Review pull requests** for LLM-related security/compliance issues — code scanning GitHub Action

  ## Red Teaming 25+ 攻击策略(`src/redteam/strategies/`)
  详见 [[../source/promptfoo]] "25+ Red-team 攻击策略"完整表格

  ## 不覆盖
  - 没有 production loop(对照 [[../eval/langfuse]])
  - 没有 multi-tenant 隔离
  - 没有 Optimizer(对照 [[dspy]] / [[../eval/opik]])
  - 没有完整 trace UI(对照 [[phoenix]])
  [verified-from-source]

production_loop: |
  **本地 + CI/CD-first**,不是 production trace 闭环。

  用户工作流:
  1. 写 `promptfooconfig.yaml`
  2. `npx promptfoo eval` 跑评测
  3. 看 Web UI 或 JSON 结果
  4. CI(`code-scan-action/`)在 PR 跑 → fail PR 如不达标
  5. Production trace 通常出到 Langfuse / Opik / Phoenix(promptfoo 不做 production observability)

  **重点**:promptfoo 是 **pre-production gate**,不是 production observability(明确分工)
  [verified-from-source]

automation: |
  - **CLI 主接口**:`npx promptfoo eval` / `npx promptfoo view`
  - **`code-scan-action/`** GitHub Action 包装 — **业内独家把 LLM eval 包成 GH Action**
  - **`pre-commit-config.yaml`** + `.husky/` Git hooks
  - **Drizzle migrations** 自动 schema 演进
  - **`commands/` CLI 模块化**(每个命令一个文件)
  [verified-from-source]

cost: |
  - **100% 本地** — 不上传 prompt / 数据到任何服务,**无 cloud cost**
  - Per-eval 主要成本:LLM API 调用 + LLM-as-judge 调用
  - **`cost` assertion 内置** — 直接 assert 单 eval 成本上限
  - **`latency` assertion 内置** — 直接 assert latency 上限
  - **provider-level token tracking**(LiteLLM-like)
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Promptfoo 是 SkillForge **eval assertion 库的"open source reference"**

  SkillForge 自家路线:
  - **EVAL-V2** 4 维分数(quality / efficiency / latency / cost)+ dimensionStatus
  - **Scenario / EvalRun / EvalTask** 三层实体
  - **PROD-LABEL-CLUSTER** + **SKILL-CANARY-ROLLOUT** + **ATTRIBUTION-AGENT** 飞轮闭环

  Promptfoo 走的是 **CLI-first + 50 assertion + red teaming**,跟 SkillForge platform 路线**互补不冲突**。

  ## 可借鉴(给 SkillForge EvalRun 补 assertion 库)

  1. **50+ 内置 assertion 类型清单** — SkillForge 当前 4 维分数维度数较少,可借鉴 Promptfoo 的 NLP metric / RAG metric / Tool / Safety 分类,扩 SkillForge eval assertion 库
  2. **`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration`**(`src/assertions/`)— **业内独家把 trace span 当 assertion 输入**。SkillForge OBS-2 + OBS-4 trace 已就绪,**可加 trace-level assertion**(eval 是否经过特定 span 路径 / span 耗时上限 / error span 数 ≤ X)
  3. **25+ Red-team 攻击策略** — SkillForge `SKILL-SECURITY-SCAN`(2026-05-13 已交付,skill 安装时静态扫描)已有 security scan,**可借鉴 Promptfoo crescendo / iterative / promptInjections 跟 SkillForge agent 评测对接**(adversarial eval)
  4. **`code-scan-action` GitHub Action 包装** — SkillForge eval 可包成 GitHub Action 让用户 CI 自动跑(SkillForge dashboard 是 web,但 CI 集成是另一条路)
  5. **Modular AGENTS.md**(每个 src/* 子目录都有自己 AGENTS.md)— SkillForge 仓库已有 `.claude/` + `.codex/`,可学 Promptfoo modular 模式(每个模块 AGENTS.md)
  6. **Drizzle ORM + SQLite local-first**(对照 [[../harness/opencode]] 同选)— SkillForge 当前 Postgres,但 **CLI eval 本地模式可考虑 SQLite**(给离线场景)
  7. **`cost` + `latency` assertion 一等公民**(`assertions/cost.ts` / `latency.ts`)— SkillForge 4 维分数已含 cost + latency,但作为 assertion 独立 type 是工程清晰

  ## SkillForge 已领先
  - **Production trace + 飞轮闭环**(Promptfoo 完全没有)
  - **Scenario / EvalRun 平台化**(Promptfoo 是 CLI)
  - **多 channel + Provider SPI 抽象**(Promptfoo 50 provider 但走 LiteLLM-like 拼接)

  ## 不必跟
  - **CLI-first 走法**(SkillForge 已 web platform)
  - **100% 本地不传任何数据**(SkillForge 是 server-side platform,定位不同)
  - **完整复制 50 assertion**(SkillForge 起步选 20 个核心:NLP metric 5 + RAG 4 + Tool 2 + Safety 4 + Trace 3 + LLM-judge 2)
  - **25+ red-team 完整集成**(SkillForge 早期选 5-7 个:promptInjections + crescendo + iterative + multilingual + base64)

  ## 关键决策建议

  **如果 SkillForge eval assertion 库扩展**:
  - **优先级 1**:`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration` 3 个 trace-level assertion(对照 SkillForge OBS-2 / OBS-4 现有 trace 模型)
  - **优先级 2**:RAG 4 metric(answerRelevance / contextFaithfulness / contextRecall / contextRelevance)— **跟 [[ragas]] 一致**
  - **优先级 3**:`trajectory` assertion(多轮校验,对照 SKILL-AB-MULTITURN-FIX 已交付)
  - **优先级 4**:5 个 red-team 策略 POC

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **CLI-first + 100% 本地 + CI/CD friendly**(`code-scan-action/` GitHub Action)— 业内 LLM eval 里最 dev-friendly
  2. **50+ 内置 assertion 类型**(业内最广)— 涵盖精确 / NLP / RAG / Tool / Safety / Trace / Trajectory / LLM-as-judge 8 大类
  3. **25+ red-team 攻击策略**(业内最完整)— crescendo / GCG / GOAT / Hydra / leetspeak / multilingual / mathPrompt / promptInjections
  4. **Trace-level assertion** 业内独家(`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration`)
  5. **Modular AGENTS.md**(每个 src/* 都有自己 AGENTS.md + Codex / Cursor 兼容)— 业内最深 modular dogfood
  6. **100% 本地 + Drizzle ORM SQLite + Biome formatter** — TS 生态新一代 stack
  7. **`Pi` provider**(`assertions/pi.ts`)— pi.ai 集成,业内少见
  8. **`code-scan-action/`** 把 eval 包成 GitHub Action — **业内独家工程产物**

  ## 优势 (strengths)
  1. **`promptfooconfig.yaml` 配置驱动** — YAML 写 tests + assertions,不写代码,**low entry**
  2. **`live reload + caching`**(README)— dev 体验快
  3. **50 provider + Compare side-by-side**(README)— matrix view 直观对比
  4. **`evaluator.ts` + `evaluate.ts`** 主入口拆 2 文件 — concern 分离
  5. **`drizzle.config.ts`** + Drizzle migrations — schema 演进可控
  6. **`feedback.ts` + `guardrails.ts`** top-level 单文件 — 简单功能不过度抽象

  ## 哪些点做得好 (what_done_well)
  1. **`src/assertions/traceErrorSpans.ts` / `traceSpanCount.ts` / `traceSpanDuration.ts`** 3 个 trace-level assertion 拆独立文件 — 工程清晰
  2. **`src/redteam/strategies/` 25+ 文件每个一个攻击** — 不堆 if-else
  3. **Modular AGENTS.md 顶层 router + 子目录 detail** — Codex 17KB 顶层 vs Promptfoo 顶层短 + 每子目录 detail,**两种 modular 风格,Promptfoo 更适合大 codebase**
  4. **`100% locally`** + `prompts never leave your machine` — 隐私一等公民,**业内独家明示**
  5. **`code-scan-action/AGENTS.md`** + GitHub Action 包装 — **eval 跟 dev workflow 集成的工程典范**
  6. **`commands/AGENTS.md`** + CLI 命令模块化 — 每个 CLI 命令一个文件,新加命令成本低
  7. **`knip.json`** unused exports 检测 — 防 dead code,业内少见集成
  8. **`biome.jsonc`** 替代 ESLint+Prettier — 工程化新选型
---

# Promptfoo(promptfoo Inc, MIT, 0.121.11)

## 一句话定位

**CLI-first LLM eval + red teaming framework**(MIT,promptfoo.dev,10M+ production users)。**50+ 内置 assertion + 25+ red-team 攻击策略 + GitHub Action `code-scan-action` + Modular AGENTS.md 业内最深**。100% 本地跑,prompts 不离开机器。

## 4 路线对照

```
Platform(Langfuse / Opik)
       ↑ 自动 production loop + SaaS UI
       |
Framework(DSPy)
       ↑ Python library + 15 optimizer
       |
Benchmark(SWE-bench / tau-bench)
       ↑ dataset + harness + leaderboard
       |
CLI eval(Promptfoo)←—— 你在这一层
       ↑ YAML config + 50 assertion + CI/CD
```

## 3 大工程亮点

### 1. 50+ 内置 Assertion(`src/assertions/`)

业内最广的 assertion 库,详见 frontmatter `scorer`。**关键独家:trace-level assertion**:
- `traceErrorSpans` — trace 中 error span 数 ≤ X
- `traceSpanCount` — trace 总 span 数在区间
- `traceSpanDuration` — span 总耗时上限

**SkillForge OBS-2 + OBS-4 trace 模型已就绪**,可直接加这 3 个 trace-level assertion。

### 2. 25+ Red-team 攻击策略(`src/redteam/strategies/`)

业内最完整开源 LLM red-team 集。详见 [[../source/promptfoo]] 完整表格。

**SkillForge SKILL-SECURITY-SCAN(2026-05-13 已交付)**是 skill 静态扫描,**Promptfoo 是 agent runtime adversarial eval**,**两者互补**。

### 3. Code Scan GitHub Action(`code-scan-action/`)

把 Promptfoo eval 包成 GitHub Action,**自动 review PR 的 LLM 相关安全合规问题** — 业内独家工程产物。

## SkillForge 校准

详见 frontmatter `skillforge_takeaway`。最核心 4 条:

1. **加 3 个 trace-level assertion**(对照 SkillForge OBS-2/OBS-4)— 短期(1 周)
2. **加 RAG 4 metric**(answerRelevance / contextFaithfulness / contextRecall / contextRelevance)— 对照 [[ragas]]
3. **加 trajectory assertion**(多轮校验)— 对照 SKILL-AB-MULTITURN-FIX
4. **POC 5 个 red-team 策略**(promptInjections / crescendo / iterative / multilingual / base64)

## 待回答
- `g-eval` 跟 LLM rubric 区别(都是 LLM-as-judge,实现细节差?)
- `Pi` provider 跟其他 provider 集成方式
- `code-scan-action/` 具体 LLM 安全规则集
- `trajectory.ts` 多轮校验跟 tau-bench Pass^k 对照
