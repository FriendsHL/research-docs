---
name: schema
type: meta
updated: 2026-05-15
---

# Schema — Agent Harness & Eval Wiki

这是 LLM 维护本 wiki 的**操作规范**。任何 ingest / query / compare / lint 都必须遵守这里的约束。

---

## 1. 写入边界 (Iron Law)

**LLM 只能在 `research/agent-harness-wiki/` 目录内创建和修改文件。**

- ✅ 可写:`research/agent-harness-wiki/**`
- ❌ 不可写:`research/agent-harness-wiki/` 以外的任何路径(包括 `research/Hermes源码研究/`、`research/claude code 源码/`、`research/OpenClaw源码研究/`、`research/anthropic-blog-translations/` 等)
- ✅ 可读:`/Users/youren/myspace/**` 下的任何 raw source(含 `skillforge/`、`hermes-agent/`、`openclaw/`、`codex/`、`opencode/`、`claw-cli-claude-code-source-code-v2.1.88/`、`langfuse/`、`opik/`、`EvolvingAgent/`、`superpowers/` 等)

外部材料通过 **wikilink 引用** (`[[../Hermes源码研究/xxx]]`) 或 **frontmatter `sources:` 字段** 关联,不复制内容进 wiki。

---

## 2. 目录结构

```
agent-harness-wiki/
├── purpose.md          # 为何存在 (LLM 每次都读)
├── schema.md           # 本文件
├── index.md            # 目录 (LLM 查询的入口)
├── log.md              # 时间线,每条带"触发问题"
├── overview.md         # 全局 thesis (lint 时维护)
├── harness/            # 单个 agent harness 研究 (wide-first quick reference, 13 维 frontmatter)
├── eval/               # 单个评测框架研究 (同上)
├── concept/            # 横切概念 (Context Compaction, Tool Use, SubAgent...)
│                       # 两种模式: wide synthesis (多 harness 横向) 或 depth (单一概念深挖)
├── deep/               # ⭐ 2026-05-17 新增 — 单一机制 depth-first 深挖
│                       # 跟 concept/ 的差异: deep/ 聚焦"单 harness 内的 1 个机制"
│                       # concept/ 聚焦"跨多 harness 的 1 个抽象 / N 维设计空间"
├── comparison/         # 跨 harness 维度对比 (/wiki-compare 产物)
└── source/             # 原始资料摘要 (博客/论文/源码笔记)
```

---

## 3. 页面类型与 frontmatter

每个页面必须有 YAML frontmatter,第一行 `type:` 决定页面种类。

### 3.1 harness/*.md — Agent Harness 页

```yaml
---
type: harness
name: <name>                    # 如 claude-code, aider, hermes-agent
status: <active|archived>
language: <java|python|ts|rust|go>

# === 版本时效(B 模式必填) ===
last_ingested_version: <version>          # 如 v2.1.88
last_ingested_at: <YYYY-MM-DD>
community_known_newer: <true|false>       # 是否知道有新版但未 ingest
known_post_ingest_features:               # 已知新特性但 unverified(对应 [community-knowledge])
  - <feature1>
  - <feature2>

sources:                        # 原始资料引用
  - "[[../Hermes源码研究/xxx]]"
  - "/Users/youren/myspace/hermes-agent/README.md"

# === 13 个能力维度 (对齐 SkillForge 模块) ===

agent_loop: |
  while tool_use; streaming; cancel via signal; anti-runaway by step cap=50

tool_protocol: |
  Anthropic tool_use schema; dynamic registry; MCP-compatible

context_mgmt: |
  Light compact (index-aligned) + Full compact (summary);
  boundary preserved as USER role;
  persistence-shape invariant 维护

memory: |
  pgvector; session-scoped + user-scoped; 主动 + 被动召回

multi_agent: |
  SubAgent (异步派发 + 自动回推, 不轮询) +
  Team (网状, TeamCreate/Send/Kill)

eval_loop: |
  离线 eval + Prompt A/B + 自动晋升; 数据集来源 production trace

self_evolution: |
  trigger: 手动 /evolve
  source: session trace + lint
  output: 新 rule / 新 agent / 新 command 建议
  review: LLM 提议 → 人审 → 落 .claude/rules/
  rollback: 版本化, git history

observability: |
  Langfuse 风格 trace; SSE 流式记录; session replay

hooks: |
  Lifecycle: PreToolUse, PostToolUse, Stop, SafetyHook, AskUser

safety:
  sandbox: process
  tool_acl: user-confirm + denylist
  bash_blocklist: ["rm -rf /", "curl | sh", "sudo *"]
  path_traversal: chroot to working dir
  anti_runaway: "step_cap=50, token_cap=200k, time_cap=10min"
  prompt_injection: "tool_result quarantined in <tool_result> blocks; user input never inlined into system prompt"
  secret_scan: 日志脱敏 + env 隔离
  auth: session 隔离, RBAC

provider: |
  Multi-provider 抽象;
  Claude, OpenAI-compatible (DeepSeek/DashScope/vLLM/Ollama)

channels: |
  REST + WebSocket; ChannelAdapter SPI;
  Web/CLI/Feishu/Telegram

persistence: |
  PostgreSQL (embedded dev / external prod) + pgvector; Flyway migrations

cancellation: |
  soft cancel via interrupt flag; hard kill via thread interrupt;
  状态机: idle → running → cancelled → idle

# === 服务 SkillForge 决策的关键栏 ===
skillforge_takeaway: |
  - 可借鉴: X 的做法 (理由)
  - SkillForge 现状: Y
  - 待决策: Z (链接到具体 prd/tech-design)
  - 风险: ...

# === 服务个人学习 / 理解 harness 设计空间的关键栏 ===
learning_takeaway: |
  ## 跟其他 harness 不同的是(differentiator)
  - 核心定位 / 设计选择上的独特位置
  - 跟同类 harness 在哪几个维度上明显不一样

  ## 优势(strengths)
  - 横向对比同类做对的地方(带具体 file:line)
  - 其他 harness 没做到的工程细节

  ## 哪些点做得好(what_done_well)
  - elegant 的设计点,值得学的解决问题方式
  - 一个简单约束/抽象解决多个问题的例子
  - 时序/边界/取舍上的精彩处理
---

# <Harness Name>

## 一句话定位
<10-30 字概括>

## 起源与定位
<这个 harness 谁做的, 为什么做, 解决什么问题>

## 核心架构
<架构图 / 模块图>

## 13 维度详解
<每个 frontmatter 字段展开>

## 与 SkillForge 的对照
<最重要的一节: 哪些可借鉴, 哪些可避坑>

## 待回答的问题
<lint 会扫这一节>
```

### 3.2 eval/*.md — 评测框架页

```yaml
---
type: eval-framework
name: <name>                    # swe-bench, tau-bench, agentbench, inspect-ai, langfuse...
status: <active|deprecated>
sources: [...]

# === 评测框架专属维度 ===
target: |
  评测对象: agent / LLM / tool-use 单点 / 多 agent 协作

dataset: |
  来源 (人工 / production trace / synthetic), 规模, 更新频率, 是否泄露

scorer: |
  rule-based / execution-based / LLM-as-judge / 人工

reproducibility: |
  seed / temperature / 模型版本锁定; docker / VM; 离线 vs 联网

coverage: |
  覆盖什么能力, 不覆盖什么; 已知 gaming pattern

production_loop: |
  评测结果如何反哺 production; 是否支持 prompt A/B + 自动晋升

automation: |
  CI/CD 集成度; trigger (PR / 定时 / 手动)

cost: |
  per-run token / 时间; 是否支持采样

skillforge_takeaway: |
  - 可借鉴: ...
  - SkillForge eval pipeline 当前覆盖: ...
  - 待对齐: ...
---
```

### 3.2b eval/*.md(`type: framework`)— 框架页(2026-05-18 新增)

放在 `eval/` 路径下,但实际是 memory / RAG / multi-agent / streaming framework,**不是 eval framework**。这类页面用 **13 维 harness frontmatter**(复用 §3.1),只是 `type` 字段标 `framework` 而非 `eval-framework`。

```yaml
---
type: framework                    # ⚠️ 区别于 eval-framework
name: <name>                       # ag2, cognee, llama_index, ms-graphrag, pathway, ...
status: <active|deprecated>
sources: [...]

# === 13 维 harness frontmatter(复用 §3.1)===
agent_loop: |
tool_protocol: |
context_mgmt: |
memory: |
multi_agent: |
eval_loop: |
self_evolution: |
observability: |
hooks: |
safety: |
provider: |
channels: |
persistence: |
cancellation: |

skillforge_takeaway: |
  - 可借鉴: ...
  - 风险 / 待决策 / SkillForge 已领先 ...

learning_takeaway: |
  - differentiator / strengths / what_done_well
---
```

**为什么不挪到 `harness/`**:
1. 这些 framework 是 memory/RAG/streaming 配套设施,非 agent 主驱动 runtime
2. 调研路线上跟 eval framework 一起 ingest(ag2 / cognee / llama_index 跟 mem0 / letta 同批)
3. 挪到 harness/ 会引发 30+ 反向链翻新

**lint 规则放宽**:`type: framework` 不强制 8 维 eval-specific(target/dataset/scorer/...),但**强制 13 维 harness + safety + takeaway**(沿用 §3.1 规则)。

**现有 5 个**:`ag2 / cognee / llama_index / ms-graphrag / pathway`(2026-05-18 决策 + 改 type 后)。

---

### 3.3 concept/*.md — 横切概念页

横切概念 (Context Compaction / Tool Use 协议 / SubAgent 模式 / Memory / Self-Evolution / Safety 等):

```yaml
---
type: concept
name: <name>
status: <draft|stable>
sources: [...]
related_harnesses: [claude-code, hermes-agent, ...]  # 哪些 harness 涉及这个概念
---

# <Concept Name>

## 定义
## 设计空间 (有哪些选项)
## 各 harness 怎么实现 (用 [[harness/xxx]] 链)
## Trade-off
## SkillForge 当前选择 + 演进方向
```

### 3.4 comparison/*.md — 维度对比页 (/wiki-compare 产物)

```yaml
---
type: comparison
dimensions: [context_mgmt, memory]   # 横向比较的维度
harnesses: [claude-code, aider, hermes-agent, openclaw]
generated_by: /wiki-compare
generated_at: 2026-05-15
---

# Context 管理 + Memory 对比

| Harness | context_mgmt | memory | 备注 |
|---|---|---|---|
| ... | ... | ... | ... |

## 关键差异
## SkillForge 启示
```

### 3.4b deep/*.md — 单机制 depth-first 深挖页(2026-05-17 新增)

**用途**:聚焦**单 harness 内的 1 个机制**做深挖,讲清楚"机制 + WHY + 失败模式防 + 横向校准 + 学习视角"。
跟 concept/ 区别:concept/ 是跨多 harness 的抽象或 N 维设计空间(synthesis),deep/ 是单点深挖(focus)。

```yaml
---
type: deep
name: <harness-mechanism>      # 如 claude-code-autodream-3-gate, codex-actor-resume-agent
status: B-mode source-grounded
harness: <claude-code | codex | opencode | ...>  # 源 harness
mechanism: <一句话标识机制>
updated: <YYYY-MM-DD>
sources:
  - "[[harness/X]]"            # 必须双向 wikilink 连回 harness 页
  - "/Users/youren/myspace/X/path/to/source.rs:line-line"  # file:line 锚点
---

# 标题:<harness> 的 <mechanism> — 一句话洞察

> 要点:1-3 句话讲清这个机制为什么重要 + 你写这页学到什么(builder 视角)

## 1. 为什么这个机制重要(也容易被忽略)
## 2. 机制 — N 行做对了什么
   (源码 + verified file:line 锚点逐段讲)
## 3. WHY — 为什么这么设计 (工程动机)
## 4. 失败模式 — 这个机制防了哪些 bug
## 5. 横向校准 — 其他 harness 怎么处理同问题
   (跟 [[harness/...]] / [[concept/...]] 对照)
## 6. (可选) 学习视角 — 写这页让我学到什么
## 引用链路
```

**Lint 规则**:
- 必须含 §"WHY" 和 §"失败模式"(否则降级为 catalog 不是深挖)
- 必须双向 wikilink 连回 `[[harness/X]]`
- 源码引用必须 file:line 锚点(B 模式)
- 建议 200-400 行(太短失去深度价值,太长应拆 2 篇)

### 3.5 source/*.md — 原始资料摘要

```yaml
---
type: source
name: <name>
origin: <url 或本地路径>
ingested_at: 2026-05-15
contributed_to: [harness/hermes-agent.md, concept/context-compaction.md]
---

# <Source Title>

## 核心论点
## 关键事实
## 与现有 wiki 的关系 (印证 / 矛盾 / 新增)
```

---

## 4. 命名约定

- 文件名:小写 + 连字符,如 `claude-code.md`, `context-compaction.md`, `swe-bench.md`
- wikilink:用相对路径,如 `[[harness/claude-code]]`, `[[concept/context-compaction]]`
- 跨目录引用现有研究材料:`[[../Hermes源码研究/xxx]]`(注意:这只是引用,不修改对方)

---

## 5. 维度兜底值

如果某个维度调研后**确实没有**,写 `N/A` 不要留空(留空会被 lint 标缺失)。
如果某个维度**没调研过**,写 `?` (lint 会提醒去调研)。

---

## 6. log.md 格式

每条 entry 必须三段式:

```markdown
## [2026-05-15] ingest | <source name>

**触发问题**: 为什么要 ingest 这个? 想回答什么? (对 SkillForge 哪个决策有用?)

**改动**:
- 新建 [[harness/xxx]]
- 更新 [[concept/yyy]]
- 添加 [[../Hermes源码研究/xxx]] 引用

**takeaway**: 一句话总结对 SkillForge 的启示
```

`grep "^## \[" log.md` 应该能拿到完整时间线。

---

## 7. index.md 维护

- index.md **仅索引 `agent-harness-wiki/` 内的页面**,不索引 `research/` 其他目录
- 按 type 分组:harness / eval / concept / comparison / source
- 每行格式:`- [[type/name]] — 一句话摘要`
- 每次 ingest 后由 LLM 自动更新

---

## 8. Lint 规则 (`/wiki-lint` 检查)

| 规则 | 严重度 |
|---|---|
| harness/eval/framework 页 frontmatter 缺维度 (留空) | error |
| `safety` 维度未填 (任何 harness 页或 `type: framework` 页) | error |
| `type: eval-framework` 缺 8 维 (target/dataset/scorer/reproducibility/coverage/production_loop/automation/cost) | error |
| `type: framework` 沿用 §3.1 13 维,**不强制** 8 维 eval-specific | (规则放宽,2026-05-18) |
| `skillforge_takeaway` 为空或只有 "?" | warning |
| `eval_loop` 和 `self_evolution` 写成相同内容 | warning(两件事,别混) |
| 孤儿页 (无入站 wikilink) | warning |
| index.md 未收录的页面 | error |
| log.md 缺"触发问题"栏 | warning |
| 跨页矛盾(同一事实不同结论) | info, 让 LLM 高亮 |
| stale 标记(超 6 个月未更新且有新源涉及) | info |

---

## 9. 演进

schema 本身会演化。每次重大调整在文件底部追加:

### Changelog
- 2026-05-15: v1 — 初始 13 维度,对齐 SkillForge 模块
- 2026-05-18: v2 — 加 §3.2b `type: framework`(放宽 eval/*.md 强制 8 维要求);§8 lint 规则同步更新。修复 5 个 eval(ag2/cognee/llama_index/ms-graphrag/pathway)实际是 RAG/memory/multi-agent framework 而非 eval framework 的语义错配
