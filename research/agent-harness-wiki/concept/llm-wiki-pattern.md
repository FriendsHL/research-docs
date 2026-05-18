---
type: concept
name: llm-wiki-pattern
status: draft
verification: source-grounded
updated: 2026-05-16
sources:
  - "[[../source/openclaw]]"
  - "[[../../OpenClaw Memory Wiki 系统分析]]"
  - "https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f (Karpathy llm-wiki.md)"
  - "https://github.com/nashsu/llm_wiki (nashsu/llm_wiki Tauri app)"
related_harnesses:
  - "[[../harness/openclaw]]"
---

# LLM Wiki Pattern(LLM 增量构建知识库模式)

## 定义

**LLM 不在每次查询时从零检索原始文档(RAG 模式),而是增量构建并持续维护一个持久化、结构化、可交叉引用的 markdown wiki。** wiki 在用户和原始资料之间作为"compounding artifact" — 知识被编译一次然后保持最新,而不是每次 query 重新推导。

**对比 RAG**:
- **RAG**:每次 query → 检索 chunk → LLM 综合答 → **下次 query 仍要重做整个流程**,知识不累积
- **LLM Wiki**:source → LLM ingest 写 wiki 页 → query 时 wiki 已是 compounding artifact,**交叉引用、矛盾标记、合成结论都已经在 wiki 里**

**来源**:Andrej Karpathy 在 2026 年发的 [llm-wiki.md gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f),描述这个 pattern,设计为"copy paste 到 LLM agent"(Claude Code / OpenCode / 等)使用。

---

## 设计空间(8 维)

| 维度 | 选项 |
|---|---|
| **架构层** | 三层(raw sources / wiki / schema)/ 双层(无 schema)/ 四层(加 cache/编译产物) |
| **页面类型** | entity / concept / source / synthesis / comparison / report / 自定义 |
| **结构化程度** | 纯 markdown / markdown + frontmatter / markdown + structured claim(confidence/evidence)|
| **新鲜度跟踪** | 无 / page-level updatedAt / claim-level updatedAt + freshness(fresh/aging/stale) |
| **矛盾检测** | 无 / 手动 lint / 自动 cluster detection(同 claimId 多页冲突) |
| **prompt 注入** | 手动 / index 自动生成 / **编译 digest 自动注入 system prompt**(关键差异) |
| **维护方** | LLM only / human only / LLM 写 + human override managed block / 双签 |
| **存储格式** | 纯 markdown / markdown + JSONL 机器格式 / 三层分离(markdown + JSONL + managed block) |

---

## 各实现怎么做

### Karpathy llm-wiki.md(原始 pattern,概念性文档)

- **三层架构**:Raw sources(不可变)/ Wiki(LLM 生成的 markdown)/ Schema(CLAUDE.md / AGENTS.md 等)
- **三个核心操作**:Ingest / Query / Lint
- **两个特殊文件**:`index.md`(内容目录)+ `log.md`(时间线,append-only)
- **Obsidian-compatible** wiki 目录
- **`[[wikilink]]`** + YAML frontmatter
- **"Human curates, LLM maintains"** 角色分工
- **抽象 design pattern**,实现细节交给 agent

### nashsu/llm_wiki(Tauri 桌面应用)

- **两步 Chain-of-Thought ingest**(先分析后生成)
- **4-Signal 关联模型**:Direct link ×3 + Source overlap ×4 + Adamic-Adar ×1.5 + Type affinity ×1.0
- **Louvain community detection** 自动聚类
- **Graph Insights**:surprising connections + knowledge gaps + bridge nodes
- **Vector Semantic Search**(LanceDB)
- **purpose.md** 一等公民("why this wiki exists")
- **Async Review System**

### [[../harness/openclaw]] Memory Wiki(**production 实现,最完整**)

`extensions/memory-wiki/src/` 25 文件:

**Vault 结构**:
```
wiki-vault/
├── entities/          ← 具名实体(人/系统/组织)
├── concepts/          ← 抽象概念
├── syntheses/         ← 跨来源综合分析
├── sources/           ← 摄入的原始材料
├── reports/           ← 自动生成 dashboard
│   ├── open-questions.md
│   ├── contradictions.md
│   └── staleness.md
├── index.md           ← 生成的总索引(managed block)
├── inbox.md           ← 人工暂存区
├── AGENTS.md / WIKI.md
└── .openclaw-wiki/
    ├── cache/
    │   ├── agent-digest.json     ← 编译后 digest(注入 prompt 用)
    │   └── claims.jsonl          ← 机器可读 claim 数据库
    ├── state.json
    └── log.jsonl                  ← 操作审计日志
```

**结构化 claim**(claim-health.ts):
```yaml
claims:
  - id: "user-prefers-dark-mode"
    text: "用户偏好深色模式"
    status: supported    # supported / contested / contradicted / refuted / superseded
    confidence: 0.92
    updatedAt: "2026-03-15"
    evidence:
      - sourceId: "source.chat-2026-03"
        lines: "42-45"
        weight: 0.8
        note: "用户明确说了三次"
        updatedAt: "2026-03-15"
```

**Freshness**:fresh(<30天)/ aging(30-90)/ stale(>90)/ unknown(无 updatedAt)
取 `claim.updatedAt`、`page.updatedAt`、`evidence[].updatedAt` 中**最新的一个**。

**Contradiction cluster**:相同 `claimId` 多页 + text/status 不同 → 进 `reports/contradictions.md` + agent-digest。

**Prompt 注入**(prompt-section.ts):
- 页面排名公式:`矛盾×6 + 未解问题×4 + min(claim数,6)×2 + min(topClaims数,3)`
- 取 Top 4 高分页面,每页 ≤2 claim(confidence↓ → freshness↓)
- 自动注入 system prompt:

```
## Compiled Wiki Snapshot
Compiled wiki currently tracks 23 claims across 4 high-signal pages.
Contradiction clusters: 1.
- 用户偏好研究: entity, 5 claims, 1 open question
  - 用户偏好深色模式 (status supported, confidence 0.92, freshness fresh)
  - 用户不喜欢弹窗通知 (status supported, confidence 0.85, freshness aging)

## Compiled Wiki
Workflow: `wiki_search` first, then `wiki_get` for the exact page you need.
```

**5 Agent 工具**(tool.ts):
- `wiki_status` — vault 健康 + Obsidian CLI 可用性
- `wiki_search` — 按标题/路径/body 搜索,可切换后端和语料库
- `wiki_get` — 按 id 或路径读页面,支持 `fromLine` / `lineCount` 分页
- `wiki_apply` — 窄范围变更:`create_synthesis` / `update_metadata`(附 claims)
- `wiki_lint` — 检查结构、来源缺失、矛盾、未解问题

**Lint 分类**:structure(missing-id / duplicate-id / missing-page-type / missing-title)/ content / health

**三层分离**:
- 人读 = Markdown 页面
- 机器操作 = JSONL / JSON
- 人工备注 = `<!-- openclaw:human:start/end -->` 块,**compile 永远不覆盖**

### [[../harness/openhuman]] Memory Tree(**wiki 内第 2 家 production 实现**)

`src/openhuman/memory/tree/` Rust + SQLite + Obsidian vault 双写,**Karpathy llm-wiki 模式 wiki 内 verified 第 2 家 production**(对照 OpenClaw 第 1 家完整 + 我们 hand-crafted 第 3 家简化):

- **17 个 JSON-RPC controller**(`schemas.rs:27-48`):
  - 写侧 6 个:`ingest / trigger_digest / delete_chunk / flush_now / wipe_all / reset_tree`
  - 读侧 10 个:`list_chunks / get_chunk / list_sources / search / recall / entity_index_for / chunks_for_entity / top_entities / chunk_score / graph_export`
  - 配置 1 个:`get_llm / set_llm`
- **存储**:本地 SQLite(`rusqlite "bundled"` `Cargo.toml:73`)+ Obsidian-compatible vault `.md` 双写 — 用户可直接编辑 markdown,降低 memory 黑盒信任成本
- **chunking**:≤3k token Markdown chunks + 评分 + 折叠成 hierarchical summary trees
- **ingestion 自动化**:118+ OAuth(经 Composio)+ 20min auto-fetch loop 持续填 vault — **Karpathy 概念 + production 自动化补给**
- **来源**:README 明示借鉴 Karpathy obsidian-wiki workflow

跟 OpenClaw 对照:
| 维度 | [[../harness/openclaw]] Memory Wiki | [[../harness/openhuman]] Memory Tree |
|---|---|---|
| 语言 | TypeScript | Rust(`src/openhuman/memory/tree/`) |
| 表面 | 5 Agent tool(status/search/get/apply/lint) | **17 JSON-RPC controller** 更细粒度 |
| Compile digest 注入 | ✅ `agent-digest.json` + 自动 system prompt 段 | ?(`flush_now` 暗示有但未 verify) |
| 双写 markdown | ✅ vault 文件 | ✅ Obsidian-compatible vault |
| Claim 结构化 | ✅ `claims.jsonl` | ?(`chunk_score / graph_export` 暗示但未 verify) |
| 自动 ingest | 人工 + Dreaming | **118+ OAuth auto-fetch 20min** |

### 我们正在做的 agent-harness-wiki(本仓库)

- 在 Karpathy 模式上做的 hand-crafted markdown 实现
- **跟 OpenClaw Memory Wiki 同源**,但简化:
  - 无 claim 结构化(直接在 prose 写)
  - 无 freshness 跟踪
  - 无 contradiction cluster 自动检测
  - 无编译 digest 自动注入(纯人/agent 读)
- **优势**:轻量,无依赖,Claude Code skill 直接驱动
- **劣势**:vs OpenClaw production 实现差距大

---

## Trade-off

### 纯 markdown vs structured claim

| | 纯 markdown(我们)| structured claim(OpenClaw) |
|---|---|---|
| 实现 | 简单 | 复杂(parse / validate / persist) |
| 矛盾检测 | 手动 grep | 自动 cluster |
| Freshness | 不知道 | 自动评级 |
| Prompt 注入 | 全文 | 编译 digest 精选 top-N |
| 数据库友好 | 否 | 是(claims.jsonl) |
| LLM 写入难度 | 低(自由文本) | 高(必须遵守 schema) |

### Compile digest 自动注入 vs 静态 index

| | 静态 index(我们)| 编译 digest 自动注入(OpenClaw) |
|---|---|---|
| 设置 | 简单 | 复杂(compile.ts + prompt-section.ts) |
| Prompt token 用量 | 整个 index | 精选 top-N |
| 实时性 | 手动更新 index | compile 后自动反映 |
| 新页面发现 | 用户/agent 主动查 | system prompt 自带 |

### Managed block 防覆盖

`<!-- openclaw:human:start/end -->` block 是 OpenClaw 的关键边界设计 —— **compile 永远不覆盖**,人工补充安全。
对应 SkillForge 类似:用户手写的 takeaway 不应该被 wiki-ingest 覆盖。

---

## SkillForge 当前选择 + 演进方向

**当前**:我们手做的 agent-harness-wiki **就是 Karpathy 模式的简化实现**(无 claim / 无 compile / 无自动注入)。

**OpenClaw 的演进路径**(SkillForge 知识库可借鉴):

1. **第一步:加 structured claim** —— frontmatter `claims:` 数组 + status/confidence/evidence
2. **第二步:加 freshness 跟踪** —— `updatedAt` + freshness 评级
3. **第三步:contradiction cluster** —— 同 claimId 多页冲突自动 reports/contradictions.md
4. **第四步:compile digest** —— `agent-digest.json` 编译产物 + `claims.jsonl` 机器读
5. **第五步:auto-inject system prompt** —— compile 后自动加 system prompt 段
6. **第六步:managed block 边界** —— 人工 override 永不被覆盖
7. **第七步:5 Agent tool** —— wiki_status / search / get / apply / lint

我们当前在第 0 步(无 claim)。**如果走这条路,意味着 SkillForge 知识库需要 25+ 文件的工程**(对照 OpenClaw memory-wiki extension)。

---

## 学习视角

### 这个 pattern 存在的根本原因(motivation)

**RAG 的根本缺陷**:LLM 在每次 query 时从零重做 retrieve + synthesis,**知识不累积**。
你读完 10 本书,跟 LLM 问问题,它每次都得"假装第一次接触这些书"。

LLM Wiki 模式:**让 LLM 慢慢消化书的内容到 wiki 里**,wiki **是 compounding artifact** —— 下次 query 它读的是已经整理好的 wiki,**不是 raw 书的 chunks**。

**类比**:RAG ≈ 每次考试现场翻书。LLM Wiki ≈ 平时把书做成笔记,考试只翻笔记。

### 哪些 harness 做得好(what_done_well across harnesses)

| 设计点 | 谁做得好 | 为什么 elegant |
|---|---|---|
| **三层分离**(人读/机器/人工 override) | OpenClaw | Don't make me think — 每层职责单一,边界清晰 |
| **Structured claim with evidence array** | OpenClaw | 不只是"这是真的",而是"这是真的因为 X.md:42-45 weight 0.8" —— **可追溯性** |
| **Compile digest auto-inject prompt** | OpenClaw | 知识库从被动等查询升级为主动 push 高信号 —— **paradigm shift** |
| **页面排名 矛盾×6 权重最高** | OpenClaw | 矛盾是"系统失健康"的强信号,优先暴露给 agent |
| **Managed block compile 不覆盖** | OpenClaw `<!-- openclaw:human:start/end -->` | 人工 override 永远赢 —— 边界明确 |
| **Obsidian-compatible vault** | Karpathy / OpenClaw 都做 | 知识库不被 lock-in,markdown + git 是 universal |
| **5 Agent tools 而非单 query** | OpenClaw | 不同操作意图分开 —— status / search / get / apply / lint |

### 学习路径

1. **入门**:读 [Karpathy llm-wiki.md gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) —— 理解 pattern 抽象
2. **进阶**:看 OpenClaw 笔记 `Memory Wiki 系统分析.md` —— 理解 production 实现的具体形态
3. **对比**:看 nashsu/llm_wiki 的 README —— 理解 GUI 实现的另一条路径
4. **深入**:看 OpenClaw `extensions/memory-wiki/src/` 25 文件 —— 理解 vault/ingest/compile/claim-health/prompt-section 各自职责
5. **应用**:对照我们 hand-crafted agent-harness-wiki —— 找差距,决定演进路径

### 反模式

- ❌ **把 wiki 当 RAG chunk 存储** —— 失去 compounding 价值
- ❌ **LLM 写完不 lint** —— 矛盾累积,知识库越来越脏
- ❌ **无 freshness 跟踪** —— stale 知识混在 fresh 知识里
- ❌ **静态 index 不 auto-inject prompt** —— 知识库被动等查询,效率低
- ❌ **无 managed block 区分人/机器写入** —— 人工补充被覆盖,信任崩塌
- ❌ **`MEMORY.md` 当流水账** —— 应该是精选小事实,不是 todo list(Hermes/Claude Code 都明确说不要这样)
