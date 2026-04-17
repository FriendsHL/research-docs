---
created: '2026-04-10T11:17:50.058597'
tags:
- claude-code
- 源码
- subagent
- token-economics
- pipeline
- prompt-cache
---

# 06 Subagent Token 经济学与 Pipeline 优化

> 基于 SkillForge 项目开发过程中的实战经验，梳理 Claude Code 子 Agent 的 prompt cache 命中机制、多 Agent Pipeline（Solo/Light/Full）的 token 成本分析、以及优化策略。
> 
> 前置阅读：[[05 Subagent 与 AgentTool]]、[[01 Agent 主循环]]

---

## 1. Fork Subagent 的 Cache 机制

### 1.1 什么被 fork（命中 cache）/ 什么没被 fork

当 Claude Code 通过 \`Agent()\` 工具启动一个子 agent 时，harness 层做了**部分 fork**：

| 层级 | 内容 | 是否 fork / cache |
|---|---|---|
| **System Prompt** | 基础指令、CLAUDE.md 内容、工具定义、env info、git status | ✅ 共享，prompt cache 命中 |
| **MessageList**（对话历史） | 用户消息、assistant 回复、工具调用 + 结果 | ❌ 不共享，子 agent 从空白开始 |

```
父对话:
┌─────────────────────────────────────────┐
│ System prompt (~15K tokens)             │ ← 子 agent 继承，cache 命中
│   - 基础指令                             │
│   - CLAUDE.md 内容                       │
│   - 工具定义（所有可用工具的 schema）      │
│   - git status / env                    │
│ ─────────────────────────────────────── │
│ MessageList (~80K tokens)               │ ← 子 agent 看不到
│   - 用户的所有消息                       │
│   - Claude 的所有回复                    │
│   - 工具调用 + 结果（Read、Bash 等）     │
└─────────────────────────────────────────┘

子 agent 拿到的:
┌─────────────────────────────────────────┐
│ System prompt (~15K)  ← 和父一样, cache ✅ │
│ ─────────────────────────────────────── │
│ 父给的 prompt (~3-8K) ← 全新内容, 不 cache │
│ (没有父的 MessageList)                    │
└─────────────────────────────────────────┘
```

### 1.2 为什么"~10% 增量"只对小任务成立

Claude Code 内部用的子 agent（搜索、context 压缩、memory 检索等）通常是**小任务**：

```
小任务子 agent（grep 一下、读一个文件）:
  System prompt (cached):   15K   ← 免费
  新 prompt "搜索 TODO":     0.5K  ← 新
  Grep 工具调用 + 结果:      1K   ← 新
  回复:                     0.5K  ← 新
  ────────────────────
  总 input: 17K
  其中 cached: 15K (88%)
  增量: 2K (12%) ≈ ~10% ✓
```

但大任务子 agent（比如我们的 reviewer 读 10+ 文件写报告）：

```
大任务子 agent:
  System prompt (cached):   15K   ← 免费
  新 prompt (brief):         5K   ← 新
  Read 10 个文件:           30K   ← 新
  分析 + 写 report:         5K   ← 新
  ────────────────────
  总 input: 55K
  其中 cached: 15K (27%)
  增量: 40K (73%) ← 远不是 10%
```

**结论：cache 占比取决于 system prompt 在总 token 中的比重。小任务 system prompt 占大头 → cache 高 → ~10%；大任务文件读取占大头 → cache 低 → ~70-85%。**

---

## 2. 三种操作的 Cache 对比

| 操作 | System prompt | MessageList | 真实增量 |
|---|---|---|---|
| **Agent() 新建** | ✅ cache（~15K 免费） | ❌ 不共享（从零开始） | 取决于新任务大小 |
| **SendMessage 继续** | ✅ cache | ✅ 全部 cache（同一个对话） | **~5-10%** |
| **Fork（不存在的理想）** | ✅ cache | ✅ cache（复制父的 prefix） | **~5-10%** |

### 关键区别：Agent() vs SendMessage vs Fork

- **Agent()**：开一间**新空办公室**，新人从零开始
- **SendMessage**：走进**已有的办公室**递一张新纸条，同一个人继续干
- **Fork（不存在）**：把已有办公室**复制两间**，两个人并行干

SendMessage 是"穷人版 fork"——拿到了 cache 的好处（同一对话 prefix 全命中），但**串行**（同一个 agent 不能同时跑两个任务）且**自审有偏见**（同一个人审自己写的代码）。

---

## 3. 内部 Context 压缩的 Token 经济学

Claude Code 的上下文 full 压缩也走 fork subagent，同样只有 system prompt cache 命中。但压缩的 ROI 和 review 不一样：

```
压缩 sub-agent:
  一次性成本: ~43K tokens（发送 50 条消息让 LLM 总结）
  之后每轮节省: ~50K tokens（80K→30K）
  10 轮后 ROI: 500K 节省 / 43K 成本 = 11.6x 回报 ← 超值

Review sub-agent:
  一次性成本: ~43K tokens（读文件写报告）
  收益: 抓几个 bug ← 有价值但不省后续 token
  ROI: 质量收益 / 43K 成本 ← 看任务
```

**压缩是"花一次省后面每一轮"，review 是"花一次买一次质量"。**

理论上压缩还有一种更高效的实现方式——不 fork 子 agent，而是在主对话内联做：

```
方案 A（fork 子 agent）:
  子 agent: [system(cached) + 50条消息(新 40K)] → 输出 summary
  cache 命中率: ~27%

方案 B（同对话内联）:
  主对话追加: "请压缩前50条消息"(新 0.5K)
  ↑ 整个 messageList 全 cache 命中 ↑
  cache 命中率: ~99%
```

方案 B 更省但需要 harness 事后清理临时消息。不确定 Claude Code 内部用的是 A 还是 B。

---

## 4. 多 Agent Pipeline 的 Token 成本分析

### 4.1 实测数据（SkillForge Phase 1 CLI 模块）

| 角色 | tokens | 模型 | 文件读取 |
|---|---|---|---|
| Dev round 1 | ~111K | Opus | 自己读 ~20 文件 |
| Reviewer A（correctness） | ~49K | Opus（应该用 Sonnet） | **又读了同样的文件** |
| Reviewer B（craft） | ~55K | Opus（应该用 Sonnet） | **又读了同样的文件** |
| Judge | ~25K | Opus（应该用 Sonnet） | 不读文件，只读 report |
| Dev fix loop (SendMessage) | ~50K | Opus | **cache 命中**，增量小 |
| Round 2 reviewer | ~42K | Opus（应该用 Sonnet） | 又读文件 |
| **总计** | **~332K** | | |
| 如果只 Solo dev | ~111K | | |
| **倍率** | **~3x** | | |

### 4.2 为什么是 3x 而不是 1.1x

```
3x 的构成拆解:
  1.0x = dev（基线）
  0.5x = reviewer A 重新读文件（完全不共享 dev 的 messageList）
  0.5x = reviewer B 同上
  0.2x = judge（较轻，只读 report）
  0.5x = fix loop（SendMessage 复用 dev 上下文 → 实际增量小）
  0.3x = round 2 reviewer
  ────
  3.0x 总计
```

**主要浪费在 reviewer 重新读 dev 已经读过的文件。** 这是 Agent() 不 fork messageList 的直接后果。

---

## 5. 四层优化策略

### 5.1 模型分级（节省 ~40% 成本，不省 token 数）

| 角色 | 推荐模型 | Agent 参数 |
|---|---|---|
| Dev | Opus | 不传 model（继承父级） |
| Reviewer | Sonnet | \`model: "sonnet"\` |
| Judge | Sonnet | \`model: "sonnet"\` |

Sonnet 单价约为 Opus 的 1/3~1/5，review/judge 的结构化分析任务 Sonnet 完全胜任。

### 5.2 diff-in-prompt（节省 ~30-40% input tokens）

```
❌ 之前：让 reviewer 自己 Read 文件
   reviewer input: system(15K) + brief(5K) + 10 个文件(30K) = 50K

✅ 之后：我先跑 git diff，把结果塞进 reviewer prompt
   reviewer input: system(15K) + brief(5K) + diff(10K) = 30K
   节省: ~40%
```

### 5.3 SendMessage 串行复用（节省 ~60%，降到 ~1.2x）

```
Step 1: Agent(dev) → 写代码 → 完成 (1.0x)
Step 2: SendMessage(dev, "切换角色：Reviewer A") → cache 命中 (0.05x)
Step 3: SendMessage(dev, "切换角色：Reviewer B") → cache 命中 (0.05x)
Step 4: 自己 self-judge (0x)
Step 5: SendMessage(dev, "fix N 个 blocker") → cache 命中 (0.05x)
总计: ~1.15x
```

代价：串行（不能并行跑两个 reviewer）+ 自审偏见（同一个 agent 审自己代码）。

### 5.4 效率对比总表

| 方案组合 | 倍率 | 并行 | 独立视角 | 质量 |
|---|---|---|---|---|
| 全 Opus + reviewer 自读文件 | ~3.0x | ✅ | ✅ | 最高 |
| + Sonnet for review/judge | ~2.0x（成本） | ✅ | ✅ | 同上 |
| + diff-in-prompt | ~1.5x | ✅ | ✅ | 同上 |
| + 1 reviewer 替代 2 | ~1.3x | N/A | ✅ | 略降 |
| SendMessage 串行复用 | ~1.15x | ❌ 串行 | ❌ 自审 | 中（用 checklist 缓解） |
| 理论 fork（API 不支持） | ~1.1x | ✅ | ✅ | 最高 |

---

## 6. 决策框架：什么时候值得花 3x

### Solo / Light / Full 三档

| 模式 | 流程 | 倍率 | 适用 |
|---|---|---|---|
| 🟢 Solo | 1 dev → 浏览器测 → commit | 1x | 单文件改 / docs / rename |
| 🟡 Light | 1 dev → SendMessage self-review → commit | ~1.2x | 中等任务 |
| 🔴 Full | 1 dev → 2 reviewer → judge → fix → commit | ~1.5-3x | 核心引擎 / 并发 / 跨模块 |

### 开工前 3 问自检

1. 会不会触碰项目的**核心文件清单**？→ Full
2. 动没动 entity/schema / serializer / build 配置？→ 至少 Light
3. Brief 超过 800 字？→ Full

详细的决策红黄灯规则见 \`~/.claude/CLAUDE.md\` 的"开发协作模式"段落。

---

## 7. 实战案例

### SkillForge Phase 2 Auto-compact（Full Pipeline）

- Dev 产出 34 个测试，但有 14 个 blocker（B2 gating 反向、tool pair 配对 stale index、@Transactional 缺失等）
- 2 个 reviewer 并行 → judge → fix loop → round 2 → approve
- **pipeline 额外 ~560K token，但防了 5 个浏览器测不出的生产 bug** → ROI 超值

### SkillForge Phase 1 CLI 模块（Full Pipeline）

- Dev 产出 78 个测试，但 ChatCommand 的 \`c.toString()\` 在 List 上会输出垃圾
- Reviewer A 精准定位 3 个 correctness blocker
- **每次 chat 命令都会触发的输出 bug，solo 模式几乎肯定会 ship**

---

## 8. 关键结论

1. **Claude Code 的子 agent 确实走 fork 路径，但只 fork system prompt，不 fork messageList**
2. **~10% 增量只对小任务成立**（system prompt 占大头时）；大任务 reviewer 读文件占主导 → ~70-85% 增量
3. **SendMessage 是唯一能让 messageList 也命中 cache 的操作**（但串行 + 自审偏见）
4. **Agent 工具 API 不暴露 fork_from 参数**——如果暴露了，pipeline 从 ~3x 降到 ~1.1x 且保留并行+独立视角
5. **最实际的优化组合**：Sonnet for review + diff-in-prompt → 从 3x 降到 ~1.5x，质量不降