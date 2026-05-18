---
name: wiki-query
description: Use when the user invokes /wiki-query, asks a question that should be answered using the agent-harness-wiki at ${CLAUDE_PROJECT_DIR}/research/agent-harness-wiki/, or says things like "查一下 wiki" / "wiki 怎么说" / "wiki 里 X 是怎么做的". The skill reads index.md, finds relevant pages, does 2-hop graph expansion via wikilinks, synthesizes an answer with citations, and offers to file the answer back as a new wiki page.
---

# wiki-query — 基于 wiki 综合回答问题

## 用途

把用户问题作为输入,在 agent-harness-wiki 内做检索 + 综合,**带引用**地回答。好的回答可以**回写**到 `wiki/queries/` 或直接成为 `concept/` / `comparison/` 页。

## 输入

`/wiki-query <question>` 或自然语言提问后由 description 触发。

## 流程

### 1. 读 Context (必做)

- `research/agent-harness-wiki/purpose.md` — 让回答带 SkillForge 视角
- `research/agent-harness-wiki/schema.md` §3 — 知道页面类型和 frontmatter 字段
- `research/agent-harness-wiki/index.md` — **检索的主入口**(列了所有页 + 摘要)

### 2. 命中候选页 (Phase 1)

从 index.md 选相关页:
- 关键词匹配(中英文都考虑)
- 同义词扩展(如 "上下文压缩" = context_mgmt = compaction)
- 标题 + 摘要双重匹配
- 输出 Top 5-8 候选

### 3. 读取候选页并打分

读每个候选页全文,按以下信号排:
- **直接相关**:frontmatter 的相关维度有内容 ×3
- **wikilink 邻居**:从命中页 wikilink 出去的二跳页加进来 ×1.5
- **同 source 邻居**:`sources:` 重叠的页 ×2
- **type 加权**:concept 页一般更适合解释性问题,harness 页更适合"X 怎么做"问题

### 4. Graph Expansion (Phase 2,二跳)

把 Top 3 命中页的 wikilink 邻居纳入,但仅当邻居的 frontmatter / 摘要跟问题相关时保留。最多扩展 5 个二跳页。

### 5. Budget Control

如果总内容超过单次合理上下文(粗估 100KB 文本),按相关度排序裁:
- 每页保留 frontmatter + H2 章节标题 + 命中维度段落
- 不相关章节标"〔略〕"

### 6. 综合回答

输出格式(强约束):

```markdown
## <Question>

### 答案
<3-10 段, 直接回答>

### 引用
- [1] [[harness/xxx]] §<section> — <一句话引用什么>
- [2] [[concept/yyy]] — ...
- [3] [[../Hermes源码研究/xxx]] — (跨目录引用)
- ...

### SkillForge 角度
<这个答案对 SkillForge 决策意味着什么? 1-3 句>

### 知识缺口 (如有)
- wiki 里没有覆盖的方面: ...
- 建议 ingest: ...

### 是否回写
建议:
- [ ] 不回写(单次问答,价值有限)
- [ ] 回写到 wiki/queries/<slug>.md(有价值的临时分析)
- [ ] 回写到 concept/<name>.md(发现了新概念页)
- [ ] 回写到 comparison/<name>.md(其实是个对比问题)

等用户选择。
```

### 7. 回写(若用户确认)

按 schema 模板生成对应页,加 frontmatter,**更新 index.md + log.md**(log 里"触发问题"就是用户的原问题)。

## 重要约束

1. **必须引用**。任何事实声明都要 `[N]` 引脚 + 对应 wiki 页。无引用的猜测明确标 ⚠️ 推测。
2. **不杜撰**。wiki 里没有的内容,标"知识缺口",而不是凭训练数据答(除非用户明确说"凭你自己知识答")。
3. **SkillForge 视角**永远要给。即使用户没问,也加一段(可能只有一行)。
4. **不直接修改 wiki**(除非走第 7 步用户确认的回写)。
