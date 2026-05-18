---
type: source
name: promptfoo
origin: "/Users/youren/myspace/promptfoo/ (promptfoo Inc, MIT)"
ingested_at: 2026-05-16
ingested_commit: 7279113
ingested_commit_date: 2026-05-15
ingested_version_label: "0.121.11(`package.json:version`)"
contributed_to:
  - eval/promptfoo.md
  - concept/agent-eval.md
---

# Source — Promptfoo

> **MIT 开源 CLI-first LLM eval + red teaming framework**(promptfoo.dev,Powers LLM apps serving 10M+ users in production)。**50+ 内置 assertion + 25+ red-team 攻击策略 + Drizzle DB migrations + React 19/Vite/MUI v7 Web UI + GitHub Action code-scan**。

## 仓库结构概览(modular AGENTS.md 业内最完整)

```
promptfoo/
├── src/                            # Core library (TypeScript)
│   ├── app/                        # React 19/Vite/MUI v7 Web UI
│   ├── assertions/                 # 50+ assertion handlers(见下)
│   ├── codeScan/                   # Code scan(检 PR LLM 安全合规)
│   ├── commands/                   # CLI commands
│   ├── evaluate.ts / evaluator.ts  # 评测核心
│   ├── matchers/                   # Assertion matcher helpers
│   ├── providers/                  # LLM providers(50+)
│   ├── redteam/                    # 安全测试 / 25+ 攻击策略
│   ├── server/                     # Backend server
│   ├── database/                   # Drizzle ORM
│   ├── feedback.ts / guardrails.ts # 用户反馈 / 护栏
│   └── golang/                     # Go binding
├── docs/agents/                    # Reusable coding-agent docs
├── code-scan-action/               # GitHub Action wrapper
├── plugins/                        # Agent plugin bundles
├── test/                           # Tests (Vitest)
├── site/                           # Docs site (Docusaurus)
├── examples/                       # Example configs
├── drizzle/                        # DB migrations
├── architecture/                   # 架构文档
├── helm/                           # Helm chart
├── biome.jsonc                     # Biome linter/formatter
├── conductor.json + conductor-setup.sh    # Conductor 部署
├── AGENTS.md (顶层)
├── CLAUDE.md
└── + 每个 src/* + docs/agents/ + .github/ + ... 都有自己 AGENTS.md
```

## 关键 file:line 锚点

### 50+ 内置 Assertion(`src/assertions/`)

按 grading 模式分:

| 类别 | Assertion |
|---|---|
| **精确匹配** | `equals` / `contains` / `startsWith` / `regex` / `matches_regex` / `wordCount` |
| **结构** | `json` / `xml` / `html` / `sql` |
| **多语言判定** | `javascript` / `python` / `ruby` / `webhook` |
| **NLP metric** | `bleu` / `rouge` / `meteor` / `gleu` / `ngrams` / `levenshtein` / `perplexity` |
| **LLM-as-judge** | `llmRubric` / `modelGradedClosedQa` / `factuality` / `g-eval`(`geval.ts`)/ `searchRubric` |
| **Semantic** | `similar`(embedding 距离) |
| **RAG** | `answerRelevance` / `contextFaithfulness` / `contextRecall` / `contextRelevance` |
| **Cost / Latency** | `cost` / `latency` / `finishReason` |
| **Tool** | `toolCallF1` / `functionToolCall` |
| **Trace 归因** | **`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration`** |
| **Trajectory** | `trajectory`(多轮 trajectory 校验) |
| **Safety** | `moderation` / `refusal` / `redteam` / `guardrails` |
| **Classifier** | `classifier`(机器学习分类器) |
| **Skill** | `skill`(skill 调用校验) |
| **Pi** | `pi`(pi.ai 集成) |
| **Synthesis** | `synthesis`(synthetic eval) |
| **OpenAI 专属** | `openai`(structured output 等) |
| **Citation** | `citation`(RAG 引用校验) |

**Trace 归因 3 个 assertion**(`traceErrorSpans` / `traceSpanCount` / `traceSpanDuration`)— **业内独家把 trace span 直接当 assertion 输入**。

### 25+ Red-team 攻击策略(`src/redteam/strategies/`)

| 策略 | 描述 |
|---|---|
| `base64` / `hex` / `rot13` / `leetspeak` / `homoglyph` / `otherEncodings` | 编码绕过 |
| `multilingual` | 多语言绕过 |
| `mathPrompt` | 数学题包装 |
| `crescendo` | 渐进诱导(`arxiv.org/abs/2404.01833`) |
| `goat` | GOAT 攻击 |
| `hydra` | Hydra 多 turn |
| `iterative` | 迭代攻击 |
| `simba` | SIMBA 启发式 |
| `gcg` | GCG(Greedy Coordinate Gradient) |
| `bestOfN` | Best-of-N 攻击 |
| `citation` | Citation 攻击 |
| `custom` | 用户自定 |
| `authoritativeMarkupInjection` | 权威标记注入 |
| `mischievousUser` | 调皮 user 模拟 |
| `indirectWebPwn` | 间接 web 攻击 |
| `promptInjections/` | Prompt injection 子目录 |
| `likert` | Likert 评分操纵 |
| `layer` | 多层 wrapping |
| `retry` | Retry 攻击 |

**业内最完整开源 LLM red-team strategy 集**。

### Code Scan GitHub Action(`code-scan-action/`)

把 Promptfoo 包成 GitHub Action,**自动 review PR 的 LLM 相关安全合规问题**(`README.md:55` "**Review pull requests** for LLM-related security and compliance issues with **code scanning**")。

### Modular AGENTS.md 工程文化

顶层 AGENTS.md(`promptfoo/AGENTS.md:7`)明确:
> "Read the relevant AGENTS.md when working in that directory."

每个子目录都有自己的 AGENTS.md:
- `.agents/AGENTS.md` — Codex metadata + repo skills
- `.github/AGENTS.md` — GitHub Actions
- `code-scan-action/AGENTS.md`
- `docs/agents/AGENTS.md` — Reusable coding-agent docs
- `plugins/AGENTS.md`
- `src/app/AGENTS.md` + `src/assertions/AGENTS.md` + `src/codeScan/AGENTS.md` + `src/commands/AGENTS.md` + `src/matchers/AGENTS.md` + `src/providers/AGENTS.md` + `src/redteam/AGENTS.md` + `src/server/AGENTS.md`
- `test/AGENTS.md` + `site/AGENTS.md` + `examples/AGENTS.md` + `drizzle/AGENTS.md`

**业内 AGENTS.md modular 文化最深**,跟 [[../harness/codex]] 17KB 顶层 / [[../harness/opencode]] 4KB 顶层不同 — Promptfoo 是 **顶层 router + 模块级 detail**,跟 [[../eval/langfuse]] `.agents/` 多 skill 模式互补。

### Drizzle ORM + Web UI

- **`drizzle/`** + `drizzle.config.ts` — DB migrations(跟 [[../harness/opencode]] 同选)
- **`src/app/` React 19/Vite/MUI v7** — Web UI
- **`src/server/`** — Node backend
- **`biome.jsonc`** — Biome linter/formatter(替代 ESLint+Prettier)
- **`knip.json`** — Knip(unused exports/files detection)

### Provider 50+(`src/providers/`)

OpenAI / Anthropic / Azure / Bedrock / Ollama / Cohere / Google / Mistral / Replicate / HuggingFace / vLLM / **+ 40+ 其他**(README "Compare models side-by-side")

### Citation & 主页 stats

- README "**Powers LLM apps serving 10M+ users in production**"
- README "**Battle-tested**"
- 主页:https://www.promptfoo.dev/
- Discord 社区:`discord.gg/promptfoo`

## 与 wiki 现有

### 印证
- **Drizzle ORM + SQLite** 跟 [[../harness/opencode]] 同选,印证 TS 生态 ORM 主流
- **Biome linter** 是新一代 ESLint+Prettier 替代,跟 opencode 一致
- **多 provider abstraction** 跟其他所有 eval 一致
- **RAG metric**(answerRelevance / contextFaithfulness / contextRecall / contextRelevance)印证 [[ragas]] 是 RAG eval 标杆

### 矛盾
- Promptfoo `assertion` 概念跟 [[../eval/opik]] `Assertion`(`AssertionResultDAO.java` Pytest 风格)**名字相同但 layer 不同**:
  - Promptfoo `assertion` = 50+ 评测规则(grading function)
  - Opik `Assertion` = Pytest 风格 pass/fail + reason
  - **两者不互斥**,Promptfoo 50+ assertion 可作为 Opik Assertion 的具体实现

### 新增
- **`eval/promptfoo.md`**(新建)
- **扩 [[../concept/agent-eval]]**:加 Promptfoo "**CLI-first eval + red teaming + GitHub Action code-scan**"路线(对照 platform / framework / benchmark 3 层之外的第 4 层 "CLI eval")
- **(可选)新建 `concept/red-teaming.md`**:Promptfoo 25+ 策略 + 业内 LLM safety eval

## 事实漂移 / 注意点

- 0.121.11 表示**未发 1.0**(虽然 10M+ users in production)— 表示 API 可能仍在演化
- **MIT license** + **完全本地跑**(README "LLM evals run **100% locally** - your prompts never leave your machine")— 跟 [[../eval/opik]] / [[../eval/langfuse]] cloud-first 路线区分
- **`.agents/AGENTS.md` 跟 Codex 兼容**(`promptfoo/AGENTS.md:11` "Codex metadata and repo skills")
- 没有自家 platform UI 收费版(对照 Comet Opik / Langfuse Cloud)
- **`Pi` provider**(`assertions/pi.ts`)是 pi.ai(Inflection 的 LLM)集成,小众
