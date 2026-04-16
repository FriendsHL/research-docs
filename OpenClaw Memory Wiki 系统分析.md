# OpenClaw Memory Wiki 系统分析

> 源码位置：`extensions/memory-wiki/src/`
> 分析日期：2026-04-09

## 一句话概括

一个**结构化知识管理插件**：把 Agent 对话产生的知识沉淀为带结构化 claim、可溯源、可自动 lint 的 Markdown wiki，vault 本身就是一个 Obsidian-compatible 目录，同时通过编译产物自动注入 Agent 的系统 prompt。

---

## 核心文件地图

| 文件 | 职责 |
|------|------|
| `vault.ts` | 初始化 vault 目录结构 |
| `ingest.ts` | 摄入本地文件为 source 页面 |
| `bridge.ts` | 从 memory-core 自动同步 artifacts（日记/梦境报告/事件日志）|
| `compile.ts` | 编译索引、dashboard 页面、agent-digest.json |
| `claim-health.ts` | claim 新鲜度评估 + 矛盾 cluster 检测 |
| `prompt-section.ts` | 从 digest 生成系统 prompt 注入片段 |
| `tool.ts` | 暴露给 Agent 的 5 个工具 |
| `lint.ts` | vault 结构健康检查 |
| `obsidian.ts` | Obsidian CLI 集成 |
| `query.ts / apply.ts` | 查询 & 变更操作 |

---

## Vault 目录结构

```
wiki-vault/
  entities/          ← 具名实体（人、系统、组织）
  concepts/          ← 抽象概念
  syntheses/         ← 跨来源综合分析
  sources/           ← 摄入的原始材料
  reports/           ← 自动生成的仪表盘页面
    open-questions.md
    contradictions.md
    staleness.md
  index.md           ← 生成的总索引（managed block）
  inbox.md           ← 人工暂存区
  AGENTS.md / WIKI.md
  .openclaw-wiki/
    cache/
      agent-digest.json   ← 编译后 digest（注入 prompt 用）
      claims.jsonl        ← 机器可读 claim 数据库
    state.json
    log.jsonl             ← 操作审计日志
```

---

## 知识流向（三层架构）

```
原始材料（sources/）
      ↓  ingest / bridge 同步
结构化页面（entities / concepts / syntheses）
      ↓  compile 编译
agent-digest.json + claims.jsonl
      ↓  prompt-section 读取
系统 Prompt（自动注入 Agent 上下文）
```

**三层分离原则**：
- 人读：Markdown 页面
- 机器操作：JSONL / JSON
- 人工备注：`<!-- openclaw:human:start/end -->` 块，compile 永远不覆盖

---

## 结构化 Claim（claim-health.ts）

每个 wiki 页面可以在 frontmatter 里声明结构化 claim：

```yaml
claims:
  - id: "user-prefers-dark-mode"
    text: "用户偏好深色模式"
    status: supported        # supported / contested / contradicted / refuted / superseded
    confidence: 0.92
    updatedAt: "2026-03-15"
    evidence:
      - sourceId: "source.chat-2026-03"
        lines: "42-45"
        weight: 0.8
        note: "用户明确说了三次"
        updatedAt: "2026-03-15"
```

### 新鲜度评级

| 级别 | 条件 |
|------|------|
| `fresh` | 距上次修改 < 30 天 |
| `aging` | 30 ~ 90 天 |
| `stale` | > 90 天 |
| `unknown` | 无 `updatedAt` |

新鲜度取 claim.updatedAt、page.updatedAt、evidence[].updatedAt 中最新的一个。

### 矛盾检测逻辑

相同 `claimId` 在多个页面出现，且文本或 status 不同 → 产生矛盾 cluster。
矛盾 cluster 会出现在 `reports/contradictions.md` 和 agent-digest 里。

---

## 系统 Prompt 注入（prompt-section.ts）

### 页面排名算法

```
score = 矛盾数 × 6 + 未解问题数 × 4 + min(claim数, 6) × 2 + min(topClaims数, 3)
```

最多取 4 个高分页面，每页最多展示 2 条 claim（按 confidence 降序 → freshness 降序）。

### 注入的 Prompt 示例（简化）

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

---

## 5 个 Agent 工具（tool.ts）

| 工具 | 作用 |
|------|------|
| `wiki_status` | vault 健康状态 + Obsidian CLI 可用性 |
| `wiki_search` | 按标题/路径/body 搜索，可切换后端和语料库 |
| `wiki_get` | 按 id 或路径读页面，支持 `fromLine` / `lineCount` 分页 |
| `wiki_apply` | 窄范围变更：`create_synthesis` / `update_metadata`（附 claims） |
| `wiki_lint` | 检查结构、来源缺失、矛盾、未解问题 |

Lint 分类（`lint.ts`）：

| 类别 | 检查项 |
|------|--------|
| structure | missing-id, duplicate-id, missing-page-type, missing-title |
| provenance | missing-source-ids, missing-import-provenance |
| links | broken-wikilink |
| contradictions | contradiction-present, claim-conflict |
| quality | open-question, low-confidence, claim-missing-evidence, stale-page, stale-claim |

---

## Bridge 模式（bridge.ts）

`vaultMode: "bridge"` 时，wiki 自动从 memory-core 同步：

| artifact 类型 | 说明 |
|--------------|------|
| `memory-root` | 记忆根文件 |
| `daily-note` | 每日注记 |
| `dream-report` | Dreaming 产出 |
| `event-log` | 事件日志（存为 JSON fence） |

**变更检测**：SHA-1 fingerprint（基于文件路径），跳过未变更文件，删除已消失 artifact 对应的 source 页面。

---

## Obsidian 集成（obsidian.ts）

直接调用 PATH 上的 `obsidian` CLI：

```
obsidian search query=<q>       ← 搜索
obsidian open path=<p>          ← 打开文件
obsidian command id=<cmd-id>    ← 触发命令
obsidian daily                  ← 打开每日笔记
```

支持多 vault（通过 `vaultName` 配置前缀 `vault=<name>`）。

---

## 设计意图

| 设计选择 | 原因 |
|----------|------|
| Vault = Obsidian 目录 | 人和 Agent 共用同一份数据，人可以直接在 Obsidian 里编辑 |
| managed block 标记 | plugin 更新生成内容，人工备注在标记外，两者互不干扰 |
| 结构化 claim + evidence | 知识可溯源、可审计、可检测矛盾，不只是堆 prose |
| compile 触发时机 | 每次 ingest / apply 后立即 compile，保证 digest 始终最新 |
| bridge 模式 | 自动把 dreaming / daily-note 的产出沉淀进 wiki，知识无需手动搬运 |
| lint 作为 first-class 工具 | Agent 在重度使用 wiki 后可以自检，主动发现知识质量问题 |
| prompt 注入按信号强度排名 | 矛盾和未解问题权重最高，让 Agent 优先关注需要解决的问题，而不是确定性知识 |
