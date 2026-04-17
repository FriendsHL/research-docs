# OpenClaw QA Character Eval 系统分析

> 源码位置：`extensions/qa-lab/src/`
> 分析日期：2026-04-09

## 一句话概括

一个**双阶段、多模型横向对比**的角色对话质量评测框架：先让所有候选模型在一个**真实运行的 Gateway 子进程**里跑同一个场景，再用高算力 Judge 模型对所有 transcript 打分排名。

---

## 核心文件地图

| 文件 | 职责 |
|------|------|
| `character-eval.ts` | 顶层编排：候选运行 + Judge 调用 + 报告生成 |
| `scenario-catalog.ts` | 从 Markdown 文件解析场景定义和 flow DSL |
| `scenario-flow-runner.ts` | 将 YAML flow DSL 动态编译为 JS AsyncFunction 执行 |
| `suite.ts` | 启动 Gateway 子进程、Lab Server、Mock Server，运行完整场景套件 |
| `gateway-child.ts` | 管理 Gateway 子进程的生命周期 |
| `lab-server.ts` | MCP 服务器，提供测试用的工具 API |
| `bus-state.ts / bus-server.ts` | 消息总线：传递 Agent 发出的消息 |
| `report.ts` | 生成 Markdown 报告 |
| `manual-lane.runtime.ts` | 单轮推理（Judge 专用） |

---

## 完整执行流程

### 第一阶段：候选模型运行

入口：`character-eval.ts:393 runQaCharacterEval`，对每个候选模型串行调用 `runQaSuite`。

#### Step 1：启动测试环境（suite.ts）

```
startQaLabServer()       ← MCP 服务器（提供测试工具）
startQaMockOpenAiServer() ← Mock OpenAI（mock 模式用）
startQaGatewayChild()    ← 真实 Gateway 子进程
```

**Gateway 子进程**是一个完整的 `openclaw gateway run` 进程，带真实配置，通过 IPC 通信。测试不 mock Gateway，直接跑真实代码路径。

**Lab Server** 是一个 MCP Server，暴露测试辅助工具给 Gateway 内的 Agent：
- 记录 Agent 发出的消息
- 读取最新的场景报告
- 检查 QA 状态

#### Step 2：读取场景定义

`readQaBootstrapScenarioCatalog()` 从 `qa/scenarios/index.md` 读取 pack 配置，再从每个 `qa/scenarios/*.md` 读取场景定义。

场景文件有两个 YAML fences：

````markdown
```yaml qa-scenario
id: character-vibes-gollum
title: Gollum 角色对话
surface: telegram
objective: 保持角色一致性并完成用户任务
successCriteria:
  - 使用 gollum 口吻回复
  - 不暴露是 AI
```

```yaml qa-flow
steps:
  - name: 发送消息
    actions:
      - call: sendMessage
        args: ["你好，我的宝贝"]
      - assert: reply contains "precious"
      - saveAs: firstReply
  - name: 追问
    actions:
      - call: sendMessage
        args: ["${vars.firstReply} 你喜欢什么？"]
      - assert: reply.length > 10
```
````

#### Step 3：启动 QA Agent 并注入场景

Gateway 子进程里运行一个 QA 专用的 Agent（身份是"Dev C-3PO"，protocol-minded 的 QA 操作员），并把场景的 `kickoffTask` 作为第一条消息发送。

Agent 按场景 bootstrap 后，测试通过 **Bus 消息总线** 驱动后续交互。

#### Step 4：执行 Flow DSL

`scenario-flow-runner.ts` 把 YAML flow 里每个 action 动态编译为 `AsyncFunction`：

```typescript
// 内部实现：把 YAML action 变成可执行的 JS
const fn = new AsyncFunction(...contextNames, `return (${expr});`);
await fn(...contextValues);
```

**Flow DSL 支持的 action：**

| action | 作用 |
|--------|------|
| `call: sendMessage` | 通过 Bus 发送消息给 Agent |
| `call: waitForReply` | 等待 Agent 回复（有超时） |
| `set: varName` | 设置变量 |
| `assert: expr` | JS 表达式断言，失败则 step 报错终止 |
| `throw: expr` | 主动抛出错误 |
| `if / then / else` | 条件分支（expr 是 JS 表达式） |
| `forEach` | 遍历列表 |
| `try / catch / finally` | 异常处理 |

变量在 steps 之间通过 `vars` 对象传递，`saveAs` 把 call 结果保存进 `vars`。

#### Step 5：收集 transcript 和结果

每个 scenario 结束后，从 Lab Server 读取 Agent 的完整对话 transcript。

`extractTranscript` 从所有 step 的 details 里取最长的一条（通常是完整的对话记录）。

统计指标：
- `transcriptChars`
- `transcriptLines`
- `userTurns`（正则匹配 `^USER\b` 的行数）
- `assistantTurns`（正则匹配 `^ASSISTANT\b` 的行数）

#### Step 6：清理环境

关闭 Gateway 子进程、Lab Server、Mock Server。每个候选模型一个独立的环境，互不干扰。

---

### 第二阶段：Judge 评审

所有候选跑完后，把全部 transcript 拼成一个大 prompt，发给 **Judge 模型**。

#### Judge 不走完整 suite，走 manual lane

Judge 调用 `runQaManualLane`：直接发一条消息给 Gateway，等待单次回复。比完整 suite 轻得多，因为 Judge 只需要一次生成。

#### 默认 Judge 模型

- `openai/gpt-5.4`：`thinkingDefault: "xhigh"`，`fastMode: true`
- `anthropic/claude-opus-4-6`：`thinkingDefault: "high"`

两个 Judge 独立打分，互不参考，减少单一模型偏见。

#### Judge Prompt 评判维度

```
✅ 自然的对话反应
✅ 有趣、出人意料的细节
✅ 角色承诺的连贯性（多轮保持一致）
✅ 完成真实用户任务，不变成"通用 AI 腔"
✅ 不感觉像在被测试
✅ 不泄露工具调用 / 后端错误信息

❌ 明确禁止按速度排名（时长只用于单独 benchmark）
```

#### Judge 输出（严格 JSON）

```json
{
  "rankings": [
    {
      "model": "openai/gpt-5.4",
      "rank": 1,
      "score": 9.2,
      "summary": "一句话总结",
      "strengths": ["优点1"],
      "weaknesses": ["缺点1"]
    }
  ]
}
```

`parseJudgeReply` 用正则提取 JSON 块（支持 markdown fence 和裸 JSON），校验字段后按 rank 排序。

---

## 候选模型配置

### 默认候选模型（10 个）

```
openai/gpt-5.4      openai/gpt-5.2
anthropic/claude-opus-4-6   anthropic/claude-sonnet-4-6
minimax/MiniMax-M2.7    zai/glm-5.1
moonshot/kimi-k2.5      qwen/qwen3.6-plus
xiaomi/mimo-v2-pro      google/gemini-3.1-pro-preview
```

### 每个模型的参数优先级（thinking level 解析）

```
candidateModelOptions[model].thinkingDefault
  ?? candidateThinkingByModel[model]
  ?? candidateThinkingDefault
  ?? DEFAULT_CHARACTER_THINKING_BY_MODEL[model]   // GPT-5.x → xhigh
  ?? DEFAULT_CHARACTER_THINKING                   // 其余 → high
```

### 默认场景

`character-vibes-gollum` — 角色承诺是高差异化维度，最能体现各模型风格差距。

---

## 产出文件结构

```
.artifacts/qa-e2e/character-eval-<timestamp>/
  character-eval-report.md     ← 人读报告（排名表 + 全部 transcript）
  character-eval-summary.json  ← 机器可处理的完整数据
  runs/
    openai-gpt-5.4/            ← 每个模型的独立运行产物
    anthropic-claude-opus-4-6/
    ...
```

---

## 设计意图

| 设计选择 | 原因 |
|----------|------|
| 真实 Gateway 子进程 | 不 mock，测试代码路径与生产完全一致 |
| 场景定义在 Markdown | 非工程师可维护，版本可追踪，不混入执行代码 |
| Flow DSL → AsyncFunction | YAML 描述意图，JS eval 负责执行，不需要自己写解释器 |
| 候选和 Judge 用不同 thinking level | 候选跑真实速度，Judge 用最高质量做裁判 |
| 两个 Judge 互相独立 | 减少单一模型的偏见 |
| 明确禁止按速度排名 | 速度是独立 benchmark 维度，不应污染质量评分 |
| manual lane 给 Judge | Judge 只需单次生成，无需完整套件环境，更轻量 |
