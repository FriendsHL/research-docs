# Advisor 策略：为 Agent 带来智能提升

> 原文：[The advisor strategy: Give agents an intelligence boost](https://claude.com/blog/the-advisor-strategy)
> 发布日期：2026年4月9日
> 翻译日期：2026年4月20日

## 概述

**将 Opus 作为 advisor 与 Sonnet 或 Haiku 作为 executor 配对**，以极低的成本获得接近 Opus 级别的智能。

开发者们已经广泛采用了一种叫做"**advisor 策略**"的方法：将 Opus 作为 advisor（顾问），Sonnet 或 Haiku 作为 executor（执行者）。这使得 Agent 能够获得接近 Opus 级别的智能，同时保持接近 Sonnet 的成本。

今天，我们正式推出 **advisor tool**，让 advisor 策略成为 API 调用中的一行代码变更。

---

## 使用 advisor 策略构建更具成本效益的 Agent

![Advisor 策略架构图](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d7a8216b96ea826922fcf4_e9f8286d.png)

### 工作原理

使用 advisor 策略时，**Sonnet 或 Haiku 作为 executor 端到端运行任务**，包括调用工具、读取结果、迭代解决方案。当 executor 遇到无法合理解决的问题时，它会**咨询 Opus 作为 advisor**。Opus 访问共享上下文并返回计划、纠正或停止信号，然后 executor 继续执行。

**advisor 不会调用工具或产生面向用户的输出**，它只向 executor 提供指导。

### 与传统模式的对比

这颠覆了常见的 sub-agent 模式（通常是大模型作为 orchestrator 分解工作并委托给更小的 worker 模型）。在 advisor 策略中，**更小、更具成本效益的模型驱动执行并升级，无需工作分解、工作池或编排逻辑**。前沿级别的推理只会在 executor 需要时使用，剩余运行保持在 executor 级别的成本。

### 性能提升

在我们的评估中，**Sonnet + Opus advisor 在 SWE-bench Multilingual 上比单独的 Sonnet 提高了 2.7 个百分点**，同时将每个 Agent 任务的成本**降低了 11.9%**。

---

## Advisor Tool

我们正在将 advisor 策略引入 API，推出了 **advisor tool**——这是一个服务端工具，Sonnet 和 Haiku 在需要指导或特定任务帮助时可以调用。

![Advisor Tool 效果对比](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d7a8216b96ea826922fcfa_880b9e59.png)

### 基准测试结果

在我们的评估中，**Sonnet + Opus advisor 在 BrowseComp 和 Terminal-Bench 2.0 基准测试中分数有所提升**，同时每个任务的成本**低于单独的 Sonnet**。

### Haiku 也能作为 Executor

advisor 策略也可以使用 **Haiku 作为 executor**：

- 在 BrowseComp 上，**Haiku + Opus advisor 得分 41.2%**，是其单独得分 19.7% 的**两倍多**
- Haiku + Opus advisor 的得分比单独的 Sonnet **低 29%**，但每个任务成本**低 85%**
- 虽然 advisor 相对于 Haiku 单独使用会增加成本，但组合价格仍然只是 Sonnet 成本的一小部分，使其成为需要平衡智能和成本的高容量任务的绝佳选择

![Haiku + Advisor 效果](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d7a8216b96ea826922fcfd_ca657f5f.png)

---

## 使用方法

在 Messages API 请求中声明 `advisor_20260301`，模型切换在单个 `/v1/messages` 请求内完成——无需额外的往返或上下文管理。**executor 模型决定何时调用它**。当它调用时，我们会将精选的上下文路由到 advisor 模型，返回计划，然后 executor 在同一请求内继续执行。

### 代码示例

```python
response = client.messages.create(
    model="sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a function to calculate fibonacci"}],
    tools=[...],
    # 添加 advisor tool
    tools_on_same_platform=[{
        "name": "advisor_20260301",
        "description": "Consult Opus for guidance on complex decisions",
        "type": "custom"
    }]
)
```

### 定价

- **Advisor tokens** 按 advisor 模型的费率计费
- **Executor tokens** 按 executor 模型的费率计费
- 由于 advisor 只生成简短的计划（通常 **400-700 个文本 token**），而 executor 以其较低费率处理完整输出，总体成本**远低于**端到端运行 advisor 模型

### 内置成本控制

- 设置 **max_uses** 来限制每个请求的 advisor 调用次数
- Advisor tokens 在使用量报告中**单独报告**，方便按层级跟踪支出

### 与现有工具配合使用

advisor tool 只是 Messages API 请求中的另一个条目。你的 Agent 可以在同一循环中**搜索网页、执行代码和咨询 Opus**。

---

## 开始使用

advisor tool 现已在 Claude Platform 上以 beta 形式原生可用。

### 步骤

1. 添加 beta 功能头：`anthropic-beta: advisor-tool-2026-03-01`
2. 将 `advisor_20260301` 添加到你的 Messages API 请求中
3. 根据你的用例修改 system prompt

我们建议针对以下三个版本运行你现有的评估套件：
- Sonnet 单独运行
- Sonnet executor + Opus advisor
- Opus 单独运行

查看[文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)了解更多详情。

---

## 参考资料

- [Advisor Tool 官方文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)
- [SWE-bench Multilingual](https://www.swebench.com/multilingual.html)
- [BrowseComp](https://github.com/chanch/llm-browsing-benchmark)
- [Terminal-Bench 2.0](https://github.com/terminal-bench/terminal_bench)

---

## 我的总结与理解

### 核心价值

这个策略的本质是**「按需调用高端智能」**，而非全程使用高端模型。把 Opus 当作「军师」，Sonnet/Haiku 当作「执行者」——遇到复杂决策时请教 Opus，平时自己干活。

### 关键洞察

1. **成本与智能的平衡艺术**
   - 传统做法：要么全用 Sonnet（贵），要么全用 Haiku（傻）
   - Advisor 策略：大部分时间用 Sonnet/Haiku，只有关键时刻才请 Opus 出山
   - 结果：成本接近 Sonnet，智能接近 Opus

2. **2.7 个百分点意味着什么？**
   - SWE-bench Multilingual 是代码任务 benchmark，提升 2.7 个点不算小数
   - 更重要的是：**成本还降了 11.9%**——这是罕见的「又强又便宜」

3. **Haiku + Opus 的场景价值**
   - 得分是 Haiku 单独的两倍多（19.7% → 41.2%）
   - 成本只有 Sonnet 的 15%
   - 适合：对智能有一定要求、但量大的场景（比如批量处理、筛选过滤）

### 对我们的启发

在 OpenClaw 里其实已经有类似的实践：
- **主 agent 用 Sonnet** 做规划和决策
- **子 agent 用 Haiku** 执行简单任务
- 复杂任务再升级给更强的模型

这跟 Claude 官方推荐的思路是一致的。不同在于官方把它做成了**原生 API 能力**，我们是通过 prompt 和模型选择手动实现的。

### 值得关注的点

- **max_uses 参数**：可以限制 advisor 调用次数，避免在简单任务上过度调用
- **400-700 token 的 plan**：说明 Opus 返回的不是完整答案，而是「思路」和「方向」，executor 仍然自己执行
- **与现有工具共存**：可以在同一循环里 search web + execute code + consult Opus，灵活性很高

---

**一句话总结**：这不是什么新技术，而是一种**更聪明的模型使用策略**——让合适的模型干合适的活，用最低成本撬动最高智能。
