---
name: wiki-ingest
description: Use when the user invokes /wiki-ingest, asks to "ingest <source>" into the agent-harness-wiki, or wants to add a new harness/eval framework / concept page to the wiki at ${CLAUDE_PROJECT_DIR}/research/agent-harness-wiki/. The skill ingests a raw source (local code repo, markdown notes, blog post, paper) into the wiki using a Two-Step Chain-of-Thought process with B-mode source-grounded verification, and updates index.md + log.md.
---

# wiki-ingest — 把一个 raw source 纳入 agent-harness-wiki

## 用途

将一个原始资料(本地源码、已有研究笔记、博客、论文)摄入 wiki。**严格 Two-Step Chain-of-Thought**:先分析、再生成,中间和用户对齐。**B 模式默认开启**:源码项目必须读 `.py`/`.java`/`.ts` 等核心入口模块,任何精确数字/常量/函数名必须带 `[code: file:line]` 锚点。

## 边界 (Iron Law)

读 `${CLAUDE_PROJECT_DIR}/research/agent-harness-wiki/schema.md` 的 §1 写入边界:**只能在 `research/agent-harness-wiki/**` 内写入文件**。其他目录(`research/Hermes源码研究/`、`research/claude code 源码/` 等)**只读**,通过 wikilink 引用即可。

## 输入

用户调用方式:
- `/wiki-ingest <path-or-url>` — 单一资料
- `/wiki-ingest <path>` — 本地目录(递归)
- `/wiki-ingest` — 不带参数:让用户从 [[index]] 的"候选 raw source"列表挑

## Pre-Step:Context 准备

1. **必读**(每次):
   - `research/agent-harness-wiki/purpose.md` — 双重目的(学习 + SkillForge)
   - `research/agent-harness-wiki/schema.md` — frontmatter 维度规范
   - `research/agent-harness-wiki/index.md` — 现有页面
   - `research/agent-harness-wiki/log.md` — 最近 5 条 entry,看上下文
2. **可选**:`research/agent-harness-wiki/overview.md`(如存在)

---

## B 模式:源码项目额外步骤(Mandatory)

如果 ingest 对象是**源码项目**(本地有 `.py`/`.java`/`.ts`/`.go`/`.rs` 等),必须:

### B.1 项目结构勘察
1. `ls <project>/` + Glob 入口文件(README/AGENTS.md/CLAUDE.md/SECURITY.md/pyproject.toml/package.json/pom.xml/Cargo.toml)
2. 识别主语言、入口模块、目录布局
3. 输出"项目结构勘察"小节(放进 Step 1 报告)

### B.2 必读核心入口模块(至少 5-8 个)
按 13 维度分布**有针对性挑选**核心模块:

| 维度 | 典型入口模块(以 Python agent 为例) |
|---|---|
| `agent_loop` | main agent class file(如 `run_agent.py::AIAgent`) |
| `tool_protocol` | `tools/registry.py`、`model_tools.py`、`toolsets.py` |
| `context_mgmt` | `agent/context_compressor.py` |
| `memory` | `hermes_state.py`、`tools/memory_tool.py`、`agent/memory_manager.py` |
| `multi_agent` | `tools/delegate_tool.py` |
| `eval_loop` | `environments/`、`batch_runner.py` |
| `self_evolution` | `tools/skills_tool.py`、`tools/skill_manager_tool.py`、`tools/skill_usage.py` |
| `hooks` | `hermes_cli/plugins.py` |
| `safety` | `tools/approval.py`、`agent/redact.py`、`tools/skills_guard.py` |
| `provider` | `agent/transports/*`、`agent/anthropic_adapter.py` |
| `channels` | `gateway/run.py`、`gateway/platforms/base.py` |
| `persistence` | `hermes_state.py`、`hermes_constants.py` |
| `cancellation` | 在 `run_agent.py` 内 grep `_interrupt_requested` |

**读取策略**(token 控制):
1. **先 grep 找行号**:`grep -nE "def AIAgent|MAX_DEPTH|max_iterations|_PARALLEL_SAFE_TOOLS|VALID_HOOKS|..." <file>`
2. **再 Read with offset/limit**:只读关键段(50-300 行/段),避免大文件全读
3. **大文件提醒**:`run_agent.py` 12k LOC、`cli.py` 11k LOC 不能全读,必须 selective
4. **Token 软上限**:input 累计 150K tokens 告警,200K 强制停下问用户

### B.3 笔记交叉验证
如果用户已有相关研究笔记(`research/<project>源码研究/`),读取笔记**作为 hypothesis**,但**所有精确事实必须以源码为准**。笔记跟源码冲突时:
- 标记"⚠️ 笔记声明 X,源码实际 Y(`file:line`)"
- 在 Step 1 报告里专门列"笔记/源码差异清单"
- 这是 wiki 价值之一 — 发现笔记漂移

### B.4 证据等级标签(三档)
每个 frontmatter 维度的内容必须以**首字段**标证据等级:

```yaml
agent_loop: |
  [verified-from-source]
  同步 while loop in run_conversation() (run_agent.py:1234-1456);
  max_iterations=90 (run_agent.py:87);
  ...
```

**4 档**:
- `[verified-from-source]` — 直接读源码确认,带 `file:line` 锚点
- `[doc-only]` — 仅 README/AGENTS.md/SECURITY.md 等官方文档支持,无源码验证
- `[notes-only]` — 仅依赖用户已有研究笔记,源码未读
- `[community-knowledge]` — 社区资讯 / 用户口头告知 / 推断,**无源码 verify**。必须标信息来源 + 时间。常用于"新版本特性已知但源码未访问"场景。

**版本时效字段**(B 模式必填 harness 页 frontmatter):
- `last_ingested_version`:如 v2.1.88
- `last_ingested_at`:ingest 日期
- `community_known_newer: true/false`:是否有新版未 ingest
- `known_post_ingest_features`: 数组,列已知新特性(对应 `[community-knowledge]` section)

**lint 规则补充**:
- `[doc-only]` 和 `[notes-only]` 超过 3 条 = warning
- `[community-knowledge]` 超过 2 条 = warning(建议 re-ingest 新版)
- `last_ingested_version` 超 6 个月不更新 = info
- 有 `known_post_ingest_features` 非空 = 提醒 re-ingest

**「已知近期演进」section** 模板(当 harness 有新版未 ingest 时,写在 13 维主体外):
```markdown
## 已知近期演进(post <version>,unverified)

> ⚠️ 信息来源:<who told / when>,**未源码 verify**。
> Verify 路径:<official changelog / product 体验 / re-ingest>。

### <feature name>
**`[community-knowledge]`** <一句话描述>
- **机制**:<推断>
- **关联现有维度**:<哪个 13 维需要扩展>
- **对 SkillForge**:<决策意义>
- **Verify 优先级**:高/中/低 + 理由
```

---

## Step 1 — Analysis(不写文件,只输出分析报告给用户)

读取 source(Pre-Step + B 模式),然后**结构化输出**:

```markdown
## Analysis: <source name>

### 0. 项目结构勘察 (B 模式)
- 主语言: ...
- 入口文件: README / AGENTS.md / ...
- 目录布局: ...

### 0.1 已读源码清单 (B 模式,精确到文件)
- `<file>` (rows N-M) — 用于回答哪些维度
- ...

### 0.2 笔记/源码差异 (如有)
| 维度 | 笔记声明 | 源码实际 | 行号 |
|---|---|---|---|
| ... | ... | ... | ... |

### 0.3 Token 用量 (B 模式)
- 已用 input: ~N tokens / 150K 软上限
- 状态: OK / 接近上限 / 已超

### 1. 这是什么
<一句话定位>

### 2. 关键实体与概念
- Entity / Concept: <name> — <一句话>
- ...

### 3. 13 个维度的初判
对每个 schema 维度给:
- 证据等级 [verified-from-source]/[doc-only]/[notes-only]
- 1-3 句话内容
- 精确数字 / 常量 / 函数名必带 [code: file:line]
- 没调研写 ?

- agent_loop: [verified-from-source] ... (`run_agent.py:1234`)
- tool_protocol: [verified-from-source] ... (`tools/registry.py:45`)
- ...

### 3.1 证据不足维度清单
- 维度 X: 只有 [doc-only],原因 ...,建议是否补读 <file>
- 维度 Y: 标 ?,需追加读 <file>

### 4. 与现有 wiki 的关系
- 印证 / 矛盾 / 新增

### 5. SkillForge 启示初判 (skillforge_takeaway 视角)
<3-5 条 bullet, 每条带 SkillForge 的对应位置 + 行号锚点>

### 5.1 学习视角初判 (learning_takeaway 三段)

**核心三问,每个 harness/concept 页都要回答:**

#### A. 跟其他 harness 不同的是(differentiator)
- 核心定位 / 设计选择上的独特位置
- 跟同类 harness 在哪几个维度上明显不一样

#### B. 优势(strengths)
- 横向对比同类做对的地方(带 file:line)
- 其他 harness 没做到的工程细节

#### C. 哪些点做得好(what_done_well)
- elegant 的设计点,值得学的解决问题方式
- 一个简单约束/抽象解决多个问题的例子
- 时序/边界/取舍上的精彩处理

### 6. 建议产出的页面
- [ ] harness/<name>.md
- [ ] concept/<name>.md (新建 / 更新)
- [ ] source/<name>.md (源摘要)
- [ ] index.md (更新)
- [ ] log.md (新 entry)

### 7. 触发问题 (写入 log.md 这一栏)
<这次 ingest 是为了回答 SkillForge 的什么问题?>
```

**输出后停下,等用户确认或修改**。如果用户说"继续",进 Step 2。

## Step 2 — Generation

按 Step 1 用户确认的清单生成文件:

### 2.1 写 source/*.md
按 schema §3.5 模板。`contributed_to` 字段先填空,产出 harness/concept 页后回填。
**B 模式增强**:`origin` 字段除了项目路径,**列出本次 ingest 实际读过的源码文件清单**(便于未来 verify)。

### 2.2 写 harness/*.md 或 eval/*.md 或 concept/*.md
**严格按 schema §3 模板**:
- harness 页必须填全 13 维 + skillforge_takeaway
- **B 模式增强**:每个维度内容首字段必须是 `[verified-from-source]` / `[doc-only]` / `[notes-only]` 之一
- **B 模式增强**:精确数字 / 常量 / 函数名 / 文件路径必须带 `[code: file:line]` 锚点
- 维度调研后没有 → `N/A`;没调研过 → `?` (lint 会标)
- `safety` 必须填(schema §8 lint 规则:safety 为空报 error)
- `skillforge_takeaway` 不能空或单 `?`(报 warning)
- **`learning_takeaway` 不能空,必须含 3 段(differentiator/strengths/what_done_well)**
  - 每段至少 2-3 条 bullet
  - 视角:站在"理解 agent 设计空间"的学习者,不是"决定 SkillForge 抄哪条"的工程师
  - 跟 `skillforge_takeaway` 重叠可以,但视角必须不同
- wikilink 用相对路径:`[[harness/xxx]]`、`[[concept/yyy]]`、`[[../Hermes源码研究/xxx]]`

### 2.3 更新 index.md
把新页加到对应分组(Harness / Eval / Concept / Source),格式:
```
- [[type/name]] — 一句话摘要
```

### 2.4 追加 log.md entry
严格按 schema §6 三段式:
```
## [YYYY-MM-DD] ingest | <source name>

**触发问题**: <Step 1 第 7 条>

**改动**:
- 新建 [[harness/xxx]]
- 更新 [[concept/yyy]]
- 引用 [[../Hermes源码研究/xxx]]

**模式**: B (source-grounded verification)
**已读源码**: N 个文件,~M tokens

**takeaway**: <一句话总结对 SkillForge 启示>
```

### 2.5 回填 source 页的 contributed_to

### 2.6 最终汇报
输出:
- 新建/更新的文件列表(带行数)
- **B 模式增强**:13 维度证据等级分布([verified-from-source] / [doc-only] / [notes-only] / ? 各几个)
- 13 维度填充率(几个 N/A、几个 `?`)
- skillforge_takeaway 摘要
- 建议的后续动作

## 重要约束

1. **不复制原始内容进 wiki**。摘要 + 提炼 + wikilink + `[code: file:line]` 引用,而非粘贴大段代码 / 文档。
2. **不修改 raw source**。`research/Hermes源码研究/`、`/Users/youren/myspace/<project>/` 等都是只读的。
3. **不创建未经用户 Step 1 确认的页面**。
4. **Two-Step 不可省**。即使用户说"快速 ingest",Step 1 的分析必须先输出。
5. **SkillForge 启示和学习视角都是一等公民**。
   - `skillforge_takeaway` — 工程视角("可借鉴/印证/更前/待决策/风险")
   - `learning_takeaway` — 学习视角(differentiator / strengths / what_done_well)
   - **两者不可偏废**,wiki 的双重目的(学习 + 服务 SkillForge)就在这两个字段
   - 任一为空或单 `?` 都报 warning
6. **concept 页加"学习视角"段**(在文末):motivation / 各 harness 谁做得好 / 学习路径 / 反模式。
6. **B 模式默认开启**(源码项目)。精确数字 / 常量 / 函数名 / 文件路径必须带 `[code: file:line]` 锚点;每维度首字段证据等级标签。
7. **Token 软上限 150K input**。Step 1 末尾必须报告 token 用量。接近上限主动停下问用户是否继续。
8. **笔记 ≠ 源码**。如果用户已有研究笔记,读笔记作为 hypothesis,但所有精确事实以源码为准;差异列入 Step 1 报告。

## 失败回滚

如果 Step 2 写到一半失败:
- 用 git status 看动了哪些文件
- 跟用户报告,询问 `git checkout -- <files>` 还是手动修
- **不要主动 git checkout**(schema §1 写入边界,但 checkout 也是写操作)

## 跳过 B 模式的场景

只有以下场景可跳过 B 模式,改读文档/笔记:
- 非源码项目(纯论文 / 博客文章 / 视频转录)
- 源码不可访问(远程仓库未克隆 / 闭源 SaaS)
- 用户显式说"快速 ingest,不读源码"(此时 Step 1 必须警告:产出全部标 `[doc-only]`,后续如要做决策需 verify)

任何跳过 B 模式的 ingest,产出页 frontmatter 必须有 `verification: doc-only` 顶层字段,lint 会高亮。
