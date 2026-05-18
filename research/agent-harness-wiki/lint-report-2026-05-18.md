---
name: lint-report-2026-05-18
type: meta
generated_by: /wiki-lint
generated_at: 2026-05-18
scope: full
---

# Wiki Lint Report — 2026-05-18

> **不写 log.md**(lint 不算内容改动)。本报告只读产出,任何修复由用户决定。

## 扫描范围

| 目录 | 实际页数 | index.md 收录数 | 差异 |
|---|---|---|---|
| harness/ | 13 | 13 | ✅ 一致 |
| eval/ | 21 | 21 | ✅ 一致 |
| concept/ | 22 | 20 | ❌ **2 个未收录** |
| deep/ | 11 | 11 | ✅ 一致 |
| comparison/ | 0 | 0 | ✅ 空,索引说明清晰 |
| source/ | 35 | 35 | ✅ 一致 |
| **总计** | **102** | **100** | |

---

## ❌ Errors(必修,4 条)

### E1. `index.md` 未收录 2 个 concept 页

| 文件 | 状态 |
|---|---|
| `concept/attribution.md` | ❌ 存在但 index 未收录 |
| `concept/trace-based-eval.md` | ❌ 存在但 index 未收录 |

**修复**:在 `index.md` §Concept 加 2 行索引。

### E2. 5 个 eval 页用错 frontmatter schema

| 文件 | `type:` 实际值 | 实际 frontmatter | 应该是 |
|---|---|---|---|
| `eval/ag2.md` | `eval` | 13-dim harness frontmatter | `type: eval-framework` + 8 dims;或挪到 `harness/` |
| `eval/cognee.md` | `eval` | 13-dim harness frontmatter | 同上 |
| `eval/llama_index.md` | `eval` | 13-dim harness frontmatter | 同上 |
| `eval/ms-graphrag.md` | `eval` | 13-dim harness frontmatter | 同上 |
| `eval/pathway.md` | `eval` | 13-dim harness frontmatter | 同上 |

**根因**:这 5 个其实是 framework(multi-agent / memory / RAG / streaming),不是 eval framework。当初放 `eval/` 是因为它们关联到 memory/RAG 调研路线,但 schema §3.2 要求 eval 页 `type: eval-framework` + 8 维(`target/dataset/scorer/reproducibility/coverage/production_loop/automation/cost/skillforge_takeaway`)。

**修复建议**(用户决定):
- 选 A:挪到 `harness/`(语义对,工程量大,要改 30+ 反向链)
- 选 B:扩 schema 加新 type `framework`(支持 harness-like 内容但定位上跟 eval/memory/RAG 相关)— 推荐
- 选 C:维持现状,改 lint 规则放宽对 `eval/*` 强制 8 维

**✅ 决策(2026-05-18)**:**选 B — 扩 schema 加 `type: framework`**(下次执行,本次 commit 暂不动)。

具体步骤(留待下次 session):
1. 在 `schema.md` §3 加新章节 `3.2b framework/*.md`(或允许 `eval/*` 子目录用 `type: framework`)
2. 13 维 harness frontmatter 复用 + 强调 takeaway 必填
3. lint §8 规则更新:`type: framework` 不强制 8 维 eval,但强制 13 维 harness + takeaway
4. 5 个文件改 `type: eval` → `type: framework`(可保留在 `eval/` 路径不动,避免反向链翻新)

记入下次 session 议程,跟 Step 4(overview 升级)一起做。

### E3. `harness/agentscope.md` safety 字段整体未 verify

```yaml
safety: |
  ? 未 verify. 需 grep "safety|guardrail|prompt_injection|sandbox".
  跟 Codex 3 平台 sandbox / Claude Code 5 permission mode + YOLO ML 对比未完成.
```

**修复**:grep agentscope 源码 `guardrail/safeguard/sandbox/prompt_injection`,补全 6 个 sub-field。Schema §8 明示 `safety 未填 = error`。

### E4. `harness/metagpt.md` safety 只填了 cost,缺其他防御

```yaml
safety: |
  ⭐ investment + NoMoneyException — cost-aware multi-agent (业内独家)
```

只覆盖 budget enforce,缺 sandbox / tool_acl / bash_blocklist / path_traversal / prompt_injection / anti_runaway 等 6 个 safety 子项。

**修复**:补全标 N/A 也行(schema §5 兜底)。

---

## ⚠️ Warnings(建议修,11 条)

### W1. 反向加链积压(✅ 本会话 Step 3 已修)

| harness | 应反向加链到 | 修复后状态 |
|---|---|---|
| `openhuman` | `concept/llm-wiki-pattern` | ✅ 已加 "wiki 内第 2 家 production 实现" section + 跟 OpenClaw 对照表 |
| `openhuman` | `concept/agent-memory` | ✅ 已加 第 7 路线 框图 + 7 路线小结对照 |
| `openhuman` | `concept/multi-agent-isolation` | ✅ 已加 Fork-style 第 2 家 verified section + 5 路线汇总表更新 |
| `langgraph` | `concept/agent-runtime-stacks` | ✅ 已加 §"Agent Execution Paradigm" 4 paradigm + paradigm × 技术栈正交矩阵 |
| `smolagents` | `concept/code-mode-orchestration` | ✅ 已有 1 引用(上次 ingest 时顺手做) |
| `smolagents` | `concept/sandbox-isolation` | ✅ 已有 3 引用(上次 ingest 时顺手做) |

### W2. 7 个维度内容含 `?` 待 verify

| 文件 | 维度 | 内容 |
|---|---|---|
| `harness/crewai.md` | cancellation | `? 未深 verify` |
| `harness/dify.md` | self_evolution | `? 未深入 verify` |
| `harness/langgraph.md` | safety.sandbox + path_traversal | `?` × 2 |
| `harness/metagpt.md` | cancellation | `? 未深 verify` |
| `harness/openhuman.md` | safety(3 子项) | bash_blocklist + path_traversal + prompt_injection 各 `?` |
| `harness/opencode.md` | safety.path_traversal | `?` |
| `harness/ragflow.md` | observability | `? 未深 verify` |
| `harness/smolagents.md` | eval_loop + safety.secret_scan | `?` × 2 |

**修复**:下次 ingest 各自源码时补;或主动标 `N/A` 如确实无此机制。

### W3. 3 个 takeaway 含 `?` 自我标注待 verify

| 文件 | takeaway 项 |
|---|---|
| `harness/langgraph.md` | Checkpoint as independent module — `? 未 verify SkillForge persistence 是否抽出 SPI` |
| `harness/openhuman.md` | Compaction library 抽出 / Subagent loading 路线 / Memory 表面 各 `?` |
| `harness/smolagents.md` | Hooks 哲学 / Sandbox 抽象 各 `?` |

**根因**:这些是 _"对 SkillForge 是否已实现 X"_ 的 verify 问题。需要去读 SkillForge 源码 verify。属于 takeaway action item 子类型。

### W4. log.md 1 个 entry 缺"触发问题"栏

| 行 | entry |
|---|---|
| `## [2026-05-16] meta` | `wiki 状态 snapshot(3 harness + 5 concept)` |

**判断**:这是 meta snapshot 而非 ingest,缺触发问题可接受。**建议无需修**(schema §6 应允许 meta 类 entry 例外)。

---

## ℹ️ Info(参考,3 条)

### I1. 无传统意义"语义 orphan"

每个非 meta 页都至少有 1 个其他 page(非 index/log/backlog/overview/purpose/schema)引用。

但 W1 标的 `openhuman` 和 `langgraph` semantic 引用极少(仅 1-2 个 cross-link),反向加链做完会显著改善。

### I2. 13 维度未发现 `eval_loop` ↔ `self_evolution` 内容相似

逐对 grep 0 命中 — schema §8 关切的"两件事混写"问题不存在。

### I3. 跨页矛盾扫描:无明显数值矛盾

未发现同一事实(如 context window / token 数 / cmd count)在多页给出不同数值。

(限于 token 预算,本次只做关键字段一致性核对,未做全文 cross-validation。)

---

## 🎯 核心产出:未消化的 SkillForge Takeaway 清单

> 这是 wiki 存在的最高价值产物。**123 个 R-XXX action items 全部待消化**(SkillForge 集成是未来工作)。

### 总览(按家族 / 优先级聚合)

**R-XXX 共 123 项**,跨 21 个 R 家族:

| 家族(数量) | 来源 | 主要议题 |
|---|---|---|
| **R-RAG (12)** | `concept/rag-design-space` | RAG 设计空间 8 维 |
| **R-MEM (10)** | `concept/agent-memory` | Memory 5+1 路线 + Graphiti episode provenance |
| **R-HARNESS (10)** | `concept/agent-harness` | 10 维 cross-harness 设计 |
| **R-TEAM (8)** | `concept/agent-team-patterns` | Multi-agent 3 哲学路线 |
| **R-OBS (8)** | `concept/observability` | Wide Events 2.0 vs OTel |
| **R-COG (7)** | `eval/cognee` | Agent trace + feedback 入 KG |
| **R-RF (6)** | `harness/ragflow` | Deep Doc Understanding + gVisor |
| **R-OH (6)** | `harness/openhuman` | Memory Tree + TokenJuice + Composio |
| **R-INJECT (6)** | `concept/runtime-context-injection` | system-reminder 协议 |
| **R-DIFY (6)** | `harness/dify` | workflow_as_tool + moderation |
| **R-INSIGHT (5)** | `deep/claude-code-insights` | MULTIPLE TIMES algorithm |
| **R-SMOL (5)** | `harness/smolagents` | executor_type Literal + code-mode |
| 其他 9 家族(各 3-5 项) | | |

### 🔴 P0/P1 必做清单(15 条,需用户决策优先级)

| ID | 描述 | 来源 | 备注 |
|---|---|---|---|
| **R-AG-1** | 5+1 speaker selection 模式(auto/manual/random/round_robin/Callable) | `eval/ag2` | 架构启示 |
| **R-AS-1** | StateModule 抽象统一持久化(业内最干净) | `harness/agentscope` / `deep/...` | 132 行撬动 5 个 capability |
| **R-AS-2** | Interrupt fake-tool-result 补全 | `harness/agentscope` | 直接打到生产 |
| **R-COG-1** | Agent Trace + Feedback 自动入 KG | `eval/cognee` | 架构启示,**直接打到 SkillForge memory** |
| **R-COG-2** | Decorator-based Agent Memory API | `eval/cognee` | 集成抽象 |
| **R-CW-1** | Process enum 显式分支(sequential/hierarchical/consensual) | `harness/crewai` | 架构 |
| **R-DIFY-1** | Visual Workflow + DSL 双轨 | `harness/dify` | P1 product 决策 |
| **R-DIFY-2** | workflow_as_tool 抽象 | `harness/dify` | P1 architecture |
| **R-LI-1** | Query Engine 18+ 种 explicit type 抽象 | `eval/llama_index` | 架构启示 |
| **R-MG-1** | `investment + cost_manager.max_budget + NoMoneyException` 三件套 | `harness/metagpt` | production safety 业内独家 |
| **R-MG-2** | `Action` 抽象不同于 `Tool` | `harness/metagpt` | 架构启示 |
| **R-RF-1** | Deep Document Understanding 16 parser + Table Structure Recognizer | `harness/ragflow` | 架构启示 |
| **R-RF-2** | gVisor Sandbox 集成 | `harness/ragflow` | P1 safety |
| **R-SMOL-1** | `executor_type: Literal[...]` 抽象给 sandbox 选型 | `harness/smolagents` | 1 周工程量 |
| **R-HARNESS-1 / R-HARNESS-6** | LSP first-class + PreToolUse/PreCompact hooks | `concept/agent-harness` | 各 1-3 周 |

### 🟡 P2 应做清单(45 条)/ 🟢 P3 远期(63 条)

数量太多,详见各 page `skillforge_takeaway` 字段 + R-XXX 编号。

---

## 📋 建议下一步(按用户原定 5 步计划)

### 立即(本会话)

- [x] Step 1:在 research-docs 启动 Claude Code(已完成,自动加载 project-level skills)
- [x] Step 2:`/wiki-lint` 审计(本报告)
- [x] **Step 3:反向加链批量(✅ 完成,4 处全清)**:
  - ✅ `concept/llm-wiki-pattern.md` 加 openhuman "wiki 内第 2 家 production 实现" + 对照表
  - ✅ `concept/agent-memory.md` 加 openhuman 第 7 路线框图 + 7 路线小结
  - ✅ `concept/multi-agent-isolation.md` 加 openhuman Fork 第 2 家 + 5 路线表更新
  - ✅ `concept/agent-runtime-stacks.md` 加 §"Agent Execution Paradigm" 4 paradigm 含 langgraph
- [ ] commit + push(本 session 末尾)
- [ ] 留待下次 session:E1(index 加 2 行)+ E2 选 B 扩 schema + E3/E4 safety verify

### 中期(下次 session)

- [ ] Step 4:升级 `[[overview]]`(1 小时)— 当前 2026-05-16 首版基于 5 harness + 2 eval + 12 concept,需更新到 13+16+22+11 = 62 主页 + 35 source
- [ ] Step 5:Step 5:下一个 P0 harness(aider / openhands / openai-agents-sdk 三选一)

### 长期 health

- 6 个月后(2026-11-18)对 `openhuman` 标 [project-in-flux] 做 re-ingest(QuickJS skills runtime rebuild 演化看)
- 跑一次 `/wiki-compare safety` 横向看 13 harness safety 维度(尤其 metagpt/agentscope/ragflow 补完后)
- 跟 SkillForge backlog 对账消化 15 条 P0/P1 R-XXX,验证哪些已实现 / 哪些待决策 / 哪些不抄

---

## 健康度总评

| 维度 | 评分 | 备注 |
|---|---|---|
| 索引完整性 | 🟡 98%(100/102 收录) | 2 个 concept 漏录(留下次清) |
| frontmatter 完整度 | 🟡 ~85% | 7 个 `?` 待 verify + 2 个 safety 字段缺失(留下次清) |
| 双向 wikilink | ✅ **本 session Step 3 已清 4 处积压** | openhuman/langgraph 反向链全部完成 |
| takeaway 可执行性 | ✅ 优秀 | 123 个 R-XXX 编号 + 优先级齐 |
| deep 页结构合规 | ✅ 100% | 11 个 deep 全含 WHY + 失败模式 |
| log.md 格式合规 | ✅ 97%(38/39) | 1 个 meta entry 例外可接受 |
| stale 标记 | ✅ 无 | 最老 ingest 2026-05-15(3 天) |

**整体**:🟡 健康度良好,2 处 E + 11 处 W 可在 30 分钟内全清。**反向加链积压**是核心积欠,建议 Step 3 优先做。
