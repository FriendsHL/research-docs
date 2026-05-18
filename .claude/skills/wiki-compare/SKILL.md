---
name: wiki-compare
description: Use when the user invokes /wiki-compare, asks to compare harnesses on a specific dimension, or says things like "对比一下 X 和 Y 的 context 管理" / "横向比较 memory" / "harness 在 safety 上有什么差异". The skill scans frontmatter of all harness/*.md (or eval/*.md) pages in ${CLAUDE_PROJECT_DIR}/research/agent-harness-wiki/, produces a comparison matrix on requested dimensions, writes it to comparison/<name>.md, and updates index.md + log.md.
---

# wiki-compare — 跨 harness 维度对比

## 用途

横向比较多个 harness(或 eval framework)在某些维度上的差异,输出**对比表 + 关键差异 + SkillForge 启示**,并落盘到 `comparison/`。

## 输入

调用形式:
- `/wiki-compare <dim1>[,<dim2>...]` — 对所有 harness 比较若干维度
- `/wiki-compare <dim> for <harness1>, <harness2>` — 限定到几个 harness
- `/wiki-compare` — 不带参数:展示可用维度,让用户选

## 可用维度

参考 `research/agent-harness-wiki/schema.md` §3.1 的 13 维 + skillforge_takeaway:
- `agent_loop` / `tool_protocol` / `context_mgmt` / `memory` / `multi_agent` / `eval_loop` / `self_evolution` / `observability` / `hooks` / `safety` / `provider` / `channels` / `persistence` / `cancellation` / `skillforge_takeaway`

eval 页用 schema §3.2 的维度:
- `target` / `dataset` / `scorer` / `reproducibility` / `coverage` / `production_loop` / `automation` / `cost` / `skillforge_takeaway`

## 流程

### 1. 读 Context

- `research/agent-harness-wiki/schema.md` — 维度定义
- `research/agent-harness-wiki/index.md` — 拿到所有 harness/eval 页清单
- `research/agent-harness-wiki/purpose.md` — SkillForge 视角

### 2. 扫 frontmatter

用 Glob + Read,扫 `harness/*.md`(或按 type 过滤):
```bash
ls research/agent-harness-wiki/harness/*.md
```
逐个 Read,只读 frontmatter(YAML 块)和 H1 标题。

### 3. 维度合规性检查

对每个 harness 页:
- 若用户请求维度 = `?` → 标记 "未调研",在表里渲染 ❓
- 若 = `N/A` → 渲染 "—"
- 若空 → 标 "(缺失)" 并在 lint 提示里报

### 4. 生成对比表

```yaml
---
type: comparison
dimensions: [<dim1>, <dim2>, ...]
harnesses: [name1, name2, ...]
generated_by: /wiki-compare
generated_at: <YYYY-MM-DD>
sources: [harness/name1.md, harness/name2.md, ...]
---

# <dim1> + <dim2> ... 对比

## 对比矩阵

| Harness | <dim1> | <dim2> | ... |
|---|---|---|---|
| [[harness/name1]] | <摘要,30 字内> | ... | ... |
| [[harness/name2]] | ... | ... | ... |
```

**单元格规则**:
- 长度限制(≤30 字),完整内容用 wikilink 跳到 harness 页
- 用 emoji/符号编码典型类别(可选):✅ 有 / ❌ 无 / ⚠️ 部分 / ❓ 未调研

### 5. 关键差异

```
## 关键差异
1. **<差异 1>**: <harness X> vs <harness Y> — <一句话>
2. ...
```

### 6. SkillForge 启示

```
## SkillForge 启示
- 在 <dim1> 上, SkillForge 当前是 <现状>, 业界主流是 <模式>, 差距/优势 = <...>
- 推荐 next step: ...
```

### 7. 写入

- 文件名:`comparison/<dim1>-<dim2>-...-YYYYMMDD.md`(或语义命名)
- 更新 `index.md` 的 Comparison 段
- 追加 `log.md` entry(type: compare)

## log.md 格式

```
## [YYYY-MM-DD] compare | <dim list>

**触发问题**: <用户问什么 / 为什么要这个对比>

**改动**: 新建 [[comparison/xxx]]

**takeaway**: <最大的差异点 + SkillForge 启示一句话>
```

## 重要约束

1. **不修改 harness 页**。compare 只读 frontmatter + 写 comparison/。
2. **缺失维度要标记**,顺便提醒"建议补 ingest"。
3. **SkillForge 启示不能空**。如果用户对比的维度跟 SkillForge 完全无关,在 takeaway 里如实说"对 SkillForge 暂无直接启示"。
4. **embed wikilink**。表格里每个 harness 名要 wikilink,方便从 comparison 跳回 harness 页。
