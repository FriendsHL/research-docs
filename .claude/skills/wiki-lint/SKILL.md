---
name: wiki-lint
description: Use when the user invokes /wiki-lint, asks to health-check / audit the agent-harness-wiki at ${CLAUDE_PROJECT_DIR}/research/agent-harness-wiki/, or says things like "查一下 wiki 哪里有问题" / "wiki 健康检查" / "wiki 哪些维度没填". The skill scans all pages, reports schema violations (missing dimensions, missing safety, etc.), orphan pages, stale claims, contradictions, and surfaces unactioned skillforge_takeaway items.
---

# wiki-lint — wiki 健康检查

## 用途

定期(或用户主动)对整个 wiki 做体检。不直接修改文件,**只产出报告**;修复建议交给用户决定(用户可手动改 / 或下一步走 `/wiki-ingest` / `/wiki-compare` 自动补)。

## 输入

`/wiki-lint` — 全量扫描
`/wiki-lint <subdir>` — 只扫某目录(如 `harness/`)

## 流程

### 1. 读 Context

- `research/agent-harness-wiki/schema.md` §8 — lint 规则表(本 skill 的真实规则源)
- `research/agent-harness-wiki/index.md` — 索引(对比实际文件 vs 索引)
- `research/agent-harness-wiki/log.md` — 看 stale 标记

### 2. 扫所有页

用 Glob:
```
research/agent-harness-wiki/{harness,eval,concept,comparison,source}/*.md
```

逐个 Read。

### 3. 规则检查(对照 schema §8)

#### A. Frontmatter 完整性 (severity: error)
- 每个 harness 页必须 13 维全填(N/A 也算填,空才报)
- eval 页必须 8 维(`target` / `dataset` / `scorer` / `reproducibility` / `coverage` / `production_loop` / `automation` / `cost` + `skillforge_takeaway`)
- `safety` 字段任何 harness 页都必须填(SkillForge 强制项)

#### B. SkillForge takeaway (severity: warning)
- `skillforge_takeaway` 不能空、不能只有 "?"、不能是"暂无"无内容
- 至少包含一条"可借鉴 X / 风险 Y / 待决策 Z"形式

#### C. 维度混淆 (severity: warning)
- `eval_loop` 和 `self_evolution` 文本相似度过高(≥80%)报警 — 两件事,别混

#### D. 孤儿页 (severity: warning)
- 该页未被任何其他页 wikilink 引用,且未在 index.md 列出 → 报
- 用 grep 检查反向引用

#### E. index.md 同步 (severity: error)
- 文件存在但 index 没收录 → 报
- index 引用了不存在的文件 → 报

#### F. log.md 格式 (severity: warning)
- 每条 entry 缺"触发问题"栏 → 报

#### G. 跨页矛盾 (severity: info)
- 同一事实(如"Claude Code 的 context window")在多页给出不同数值 → 高亮让 LLM 判断

#### H. Stale (severity: info)
- 页面 frontmatter `updated:` 字段超 6 个月,且本地对应 source 在此期间有改动 → 提醒

#### I. 跨目录引用有效性 (severity: warning)
- `[[../xxx]]` 链接对应文件不存在 → 报(注意 Obsidian 的引用规则)

### 4. 未消化的 takeaway 审计 (重点!)

扫所有 `skillforge_takeaway` 字段,提取里面的 **action items**(关键词:"可借鉴"/"待决策"/"建议"/"应该"):

输出一张"未消化清单":
```
## 未消化的 SkillForge takeaway

| 来源 page | takeaway action | 状态 |
|---|---|---|
| [[harness/claude-code]] | 可借鉴 hooks 命名空间设计 | ⚪ 未确认 |
| [[harness/hermes-agent]] | 待决策 memory pgvector vs lance | ⚪ 未确认 |
```

这是 wiki 存在的核心价值 — **让 SkillForge 决策有迹可循**。

### 5. 报告输出

```markdown
# Wiki Lint Report — <YYYY-MM-DD>

## 扫描范围
- harness/: N 页
- eval/: N 页
- concept/: N 页
- comparison/: N 页
- source/: N 页

## Errors (必修)
- [[harness/xxx]]: `safety` 字段为空
- [[index]]: 引用了不存在的 [[concept/yyy]]
- ...

## Warnings (建议修)
- [[harness/zzz]]: skillforge_takeaway 内容为 "?"
- [[harness/aaa]] 与 [[harness/bbb]] 在 context_window 数值不一致
- ...

## Info
- [[concept/memory]]: 最近 7 个月未更新, opik 在 5 月有大改
- ...

## 未消化的 SkillForge Takeaway
<上面第 4 步的清单>

## 建议下一步
- [ ] 跑 /wiki-ingest <source> 补 N 个 missing dimensions
- [ ] 跑 /wiki-compare safety 横向看 safety 维度
- [ ] 人工审 N 条未消化 takeaway, 决定哪些进 SkillForge backlog
```

### 6. 不写 log.md

lint 不算"内容改动",**不写 log.md**(避免污染时间线)。

## 重要约束

1. **lint 只读不写**。任何修复都让用户决定。
2. **报告要可执行**,不要泛泛"建议改进"。每个 finding 都要带具体页面 + 具体字段 + 修复路径。
3. **未消化 takeaway 是核心产出**。其他规则是卫生,这个是价值。
