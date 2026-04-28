# 多 Agent 协调模式：如何选择最适合你的模式

> 原文：[Multi-agent coordination patterns](https://claude.com/blog/multi-agent-coordination-patterns)
> 发布日期：2026年4月10日
> 翻译日期：2026年4月20日

![多 Agent 协调模式总览](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d901978c5cc197e3f1c1c7_1b56c9bc.png)

## 概述

在之前的文章中，我们探讨了多 Agent 系统何时提供价值，单一 Agent 何时是更好的选择。这篇文章是给已经做出判断的团队看的——他们需要决定哪种协调模式适合他们的问题。

我们看到团队根据"听起来高大上"来选择模式，而不是根据实际问题。我们建议从**最简单的模式**开始，观察它在哪里挣扎，然后从那里演进。

本文探讨五种模式的机制和局限性：

1. **Generator-Verifier**：质量关键输出，有明确的评估标准
2. **Orchestrator-Subagent**：清晰的任务分解，子任务有边界
3. **Agent Teams**：并行、独立、长时间运行的子任务
4. **Message Bus**：事件驱动的管道，Agent 生态系统不断增长
5. **Shared State**：协作工作，Agent 相互构建彼此的发现

---

## 模式 1：Generator-Verifier

这是最简单的多 Agent 模式，也是部署最广泛的。我们之前把它作为验证 subagent 模式介绍，这里使用更广泛的 generator-verifier 框架，因为 generator 不一定是 orchestrator。

### 工作原理

Generator 接收任务并产生初始输出，然后传递给 verifier 进行评估。Verifier 检查输出是否满足要求的标准，要么接受它作为完成，要么拒绝并提供反馈。如果被拒绝，反馈会路由回 generator，generator 用它来产生修订版本。这个循环继续，直到 verifier 接受输出或达到最大迭代次数。

### 适用场景

考虑一个生成客户工单邮件回复的支持系统。generator 使用产品文档和工单上下文生成初始回复。Verifier 根据知识库检查准确性，根据品牌指南评估语气，并确认回复解决了提出的每个问题。失败的检查带着反馈返回给 generator，反馈指出具体问题，例如功能错误归属定价层或工单问题未回复。

**使用这个模式当：**
- 输出质量很关键
- 评估标准可以明确
- 适用于代码生成（一个 agent 写代码，另一个写和运行测试）、事实核查、基于规则的评分、合规验证，以及错误输出成本高于额外生成周期的任何领域

### 局限性

Verifier 只能和它的标准一样好。一个只被告知检查输出是否好而没有进一步标准的 verifier，会在 generator 的输出上盖橡皮图章。团队最常失败的地方是实现了循环但没有定义验证意味着什么，创造了一种质量控制的假象而没有实质内容。

该模式还假设生成和验证是可分离的技能。如果评估创意方法像生成一样困难，verifier 可能无法可靠地发现问题。

最后，迭代循环可能停滞。如果 generator 无法解决 verifier 的反馈，系统会振荡而不收敛。最大迭代限制加回退策略（升级给人类、返回最佳尝试并附上警告）防止这变成无限循环。

---

## 模式 2：Orchestrator-Subagent

层级定义了这个模式。一个 agent 充当团队领导，计划工作、分配任务、综合结果。Subagents 处理特定职责并报告回来。

### 工作原理

领导 agent 接收任务并决定如何处理。它可能直接处理一些子任务，同时将其他任务分配给 subagents。Subagents 完成工作并返回结果，orchestrator 将其综合成最终输出。

Claude Code 使用这个模式。主 agent 自己编写代码、编辑文件、运行命令，当需要搜索大型代码库或调查独立问题时，在后台分派 subagents，这样工作继续的同时结果流回。每个 subagent 在自己的上下文窗口中操作，并返回精简的发现。这让 orchestrator 的上下文聚焦于主要任务，而探索并行发生。

### 适用场景

考虑一个自动化代码审查系统。当拉取请求到达时，系统需要检查安全漏洞、验证测试覆盖率、评估代码风格、评估架构一致性。每个检查都不同，需要不同的上下文，并产生清晰的输出。Orchestrator 将每个检查分派给专门的 subagent，收集结果并综合成统一审查。

**使用这个模式当：**
- 任务分解清晰
- 子任务相互依赖最小
- Orchestrator 保持对整体目标的一致视图，而 subagents 专注于特定职责

### 局限性

Orchestrator 成为信息瓶颈。当一个 subagent 发现与另一个 subagent 工作相关的信息时，信息必须通过 orchestrator 路由。如果安全 subagent 发现影响架构 subagent 分析的认证缺陷，orchestrator 必须识别这种依赖并适当路由信息。经过几次这样的交接，关键细节经常丢失或被总结掉。

顺序执行也限制吞吐量。除非明确并行化，subagents 一个接一个地运行，意味着系统产生多 agent token 成本而没有速度收益。

---

## 模式 3：Agent Teams

当工作分解为可以独立长时间运行的并行子任务时，orchestrator-subagent 可能变得不必要地限制。

### 工作原理

Coordinator 将多个 worker agents 作为独立进程生成。队友从共享队列认领任务，自主跨多步处理，并发出完成信号。

与 orchestrator-subagent 的区别在于 worker 的持久性。Orchestrator 为一个有限的子任务生成一个 subagent，subagent 在返回结果后终止。队友在多项任务中保持存活，积累上下文和领域专业化，随时间提高性能。Coordinator 分配工作并收集结果，但不在任务之间重置 workers。

### 适用场景

考虑将大型代码库从一个框架迁移到另一个框架。队友可以独立迁移每个服务，有自己的依赖、测试套件和部署配置。Coordinator 将每个服务分配给一个队友，每个队友自主完成迁移：依赖更新、代码更改、测试修复、验证。Coordinator 收集完成的迁移并在全系统运行集成测试。

**使用这个模式当：**
- 子任务独立
- 受益于持续的多步工作
- 每个队友积累其领域的上下文，而不是每次分派从头开始

### 局限性

独立性是关键要求。与 orchestrator-subagent 不同，orchestrator 可以在 subagents 之间调解并路由信息，队友自主操作，不能轻易共享中间发现。如果一个队友的工作影响另一个，两者都不知道，他们的输出可能冲突。

完成检测也更难。由于队友自主工作可变时长，coordinator 必须处理部分完成，其中一个队友两分钟完成，另一个需要二十分钟。

共享资源加剧这两个问题。当多个队友操作同一代码库、数据库或文件系统时，两个队友可能编辑同一文件或做出不兼容的更改。该模式需要仔细的任务分区和冲突解决机制。

---

## 模式 4：Message Bus

随着 Agent 数量增加和交互模式变得复杂，直接协调变得难以管理。Message bus 引入共享通信层，agents 在其中发布和订阅事件。

### 工作原理

Agents 通过两个原语交互：publish 和 subscribe。Agents 订阅它们关心的主题，路由器传递匹配的消息。具有新能力的新 agents 可以开始接收相关工作，而无需重新接线现有连接。

### 适用场景

安全运营自动化系统展示了这个模式的优势。警报来自多个来源，分类 agent 按严重程度和类型对每个进行分类，将高严重性网络警报路由到网络调查 agent，将凭证相关警报路由到身份分析 agent。每个调查 agent 可能发布丰富请求，上下文收集 agent 满足这些请求。发现流向响应协调 agent，决定适当的行动。

这个管道适合 message bus，因为事件从一个阶段流向下一个，团队可以随着威胁类别演进添加新的 agent 类型，团队可以独立开发和部署 agents。

**使用这个模式当：**
- 事件驱动的管道
- 工作流从事件中出现而不是预定序列
- Agent 生态系统可能增长

### 局限性

事件驱动的灵活性使追踪更难。当警报触发五个 agents 的事件级联，理解发生了什么需要仔细的日志和关联。调试比跟踪 orchestrator 的顺序决策更难。

路由准确性也至关重要。如果路由器错误分类或丢弃事件，系统静默失败，处理 nothing 但从不崩溃。基于 LLM 的路由器提供语义灵活性，但引入自己的失败模式。

---

## 模式 5：Shared State

Orchestrators、团队领导和之前模式中的消息路由器都集中管理信息流。Shared State 通过让 agents 通过持久存储直接协调来移除中间人。

### 工作原理

Agents 自主操作，从和写入共享数据库、文件系统或文档。没有中央协调器。Agents 检查存储中的相关信息，对发现采取行动，并将发现写回。工作时通常由初始化步骤向存储中植入问题或数据集，当满足终止条件时结束：时间限制、收敛阈值，或指定 agent 决定存储包含足够答案。

### 适用场景

考虑一个研究综合系统，多个 agents 调查复杂问题的不同方面。一个探索学术文献，另一个分析行业报告，第三个检查专利，第四个监控新闻报道。每个 agent 的发现可能通知其他人的调查。学术文献 agent 可能发现其公司应该被行业 agent 更仔细检查的关键研究者。

通过 shared state，发现直接进入存储。行业 agent 可以立即看到学术 agent 的发现，而不必等待协调器路由信息。Agents 建立在彼此的工作上，共享存储成为不断发展的知识库。

Shared State 还移除了协调器作为单点故障。如果任何一个 agent 停止，其他继续读写。在 orchestrator 和 message-bus 系统中，协调器或路由器故障停止一切。

### 局限性

没有明确协调，agents 可能重复工作或追求矛盾的方法。两个 agents 可能独立调查同一线索。Agent 交互产生系统行为而不是自上而下设计，使结果不那么可预测。

更难失败的模式是反应循环。例如，Agent A 写发现，Agent B 读取并写后续，A 看到后续并回应。系统继续燃烧 token 在不收敛的工作上。重复工作和并发写入有已知的工程修复（锁定、版本控制、分区）。反应循环是行为问题，需要第一类终止条件：时间预算、收敛阈值（ N 周期无新发现），或指定 agent 决定存储包含足够答案。系统把终止当马后炮 tend to cycle indefinitely or stop arbitrarily when one agent's context fills.

---

## 选择和演进模式

正确的模式取决于关于系统的几个结构性问题。我们之前主张以上下文为中心的分解，根据每个 agent 需要什么上下文而不是它做什么类型的工作来划分工作。这个原则在这里也适用。这些模式在它们如何管理上下文边界和信息流方面不同。

### Orchestrator-Subagent vs. Agent Teams

都涉及 coordinator 向其他 agents 分派工作。问题是 workers 需要保持上下文多长时间。

- 选择 **Orchestrator-Subagent** 当子任务短、聚焦、产生清晰输出
- 选择 **Agent Teams** 当子任务受益于持续的多步工作

当 subagents 需要在调用之间保留状态时，agent teams 是更好的选择。

### Orchestrator-Subagent vs. Message Bus

两者都可以处理多步工作流。问题是工作流结构有多可预测。

- 选择 **Orchestrator-Subagent** 当步骤顺序已知
- 选择 **Message Bus** 当工作流从事件中出现并可能根据发现而变化

当条件逻辑在 orchestrator 中积累以处理扩展的情况，message bus 使路由明确和可扩展。

### Agent Teams vs. Shared State

都涉及 agents 自主工作。问题是 agents 是否需要彼此的发现。

- 选择 **Agent Teams** 当 agents 处理不交互的独立分区
- 选择 **Shared State** 当 agents 的工作是协作的，发现应该实时流动

一旦队友需要相互通信而不仅仅是分享最终结果，shared state 使这更自然。

### Message Bus vs. Shared State

都支持复杂的多 agent 协调。问题是工作作为离散事件流动还是积累到共享知识库。

- 选择 **Message Bus** 当 agents 对管道中的事件做出反应
- 选择 **Shared State** 当 agents 随着时间在积累的发现上构建

Message bus 仍有路由器，这意味着中央组件决定事件去向。Shared state 是去中心化的。如果消除单点故障是优先项，shared state 更完全地提供这一点。

---

## 总结表

| 情况 | 模式 |
|------|------|
| 质量关键输出，明确评估标准 | Generator-Verifier |
| 清晰任务分解，有界子任务 | Orchestrator-Subagent |
| 并行工作负载，独立长时间子任务 | Agent Teams |
| 事件驱动管道，增长的 Agent 生态 | Message Bus |
| 协作研究，Agents 共享发现 | Shared State |
| 无需单点故障 | Shared State |

---

## 图片补充

![模式对比表](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d901978c5cc197e3f1c1c4_76abfe32.png)

![模式选择决策树](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69d901978c5cc197e3f1c1cd_4282798b.png)

---

## 我的总结与理解

### 核心洞见

这篇文章系统性地梳理了 5 种多 Agent 协调模式，关键洞察：

1. **从简单开始**：先选最简单能工作的模式，再根据实际痛点演进
2. **没有银弹**：每种模式都有明确的适用场景和局限性
3. **模式可以组合**：生产系统经常混合使用
4. **上下文是核心**：选择模式时考虑每个 Agent 需要什么上下文

### 各模式的深度理解

| 模式 | 协调方式 | 最佳场景 | 致命弱点 |
|------|----------|----------|----------|
| Generator-Verifier | 循环迭代 | 质量重要、有明确标准 | Verifier 标准不够精确会变成橡皮图章 |
| Orchestrator-Subagent | 中央调度 | 任务可分解、步骤固定 | Orchestrator 成为信息瓶颈 |
| Agent Teams | 自主协作 | 并行任务、独立工作 | 无法处理相互依赖的任务 |
| Message Bus | 事件驱动 | 动态工作流、管道处理 | 路由错误会静默失败 |
| Shared State | 去中心化 | 协作研究、实时共享 | 可能产生反应循环（无限震荡） |

### 选择模式的决策框架

**第一步：明确任务特征**
- 任务可分解吗？步骤固定吗？→ Orchestrator-Subagent
- 输出质量有明确标准吗？→ Generator-Verifier
- 子任务独立且长时间运行吗？→ Agent Teams
- 工作流由事件驱动吗？→ Message Bus
- Agent 需要实时协作吗？→ Shared State

**第二步：考虑非功能需求**
- 需要消除单点故障？→ Shared State
- 生态系统会增长？→ Message Bus
- 需要快速并行？→ Agent Teams

**第三步：从最小可行模式开始**
- 先用最简单的模式
- 观察它在哪里挣扎
- 按需演进到更复杂的模式

### 对 OpenClaw 的启发

我们的 subagent 模式类似于 Orchestrator-Subagent，但有更多可以探索的空间：

1. **质量敏感任务**：可以引入 Generator-Verifier，比如代码审查
2. **并行任务**：考虑 Agent Teams 模式，支持更长的生命周期
3. **协作场景**：探索 Shared State 来共享上下文
4. **事件驱动**：当有 webhook 触发时，Message Bus 是好选择

### 可操作的建议

- [ ] 审计现有的多 Agent 实现，识别当前使用的模式
- [ ] 标记每个模式的痛点，观察不是假设
- [ ] 准备一个"模式演进地图"，知道何时升级
- [ ] 对于新功能，默认用 Orchestrator-Subagent，只在必要时演进

---

**一句话总结**：多 Agent 协调没有最优解，**从简单开始、观察痛点、逐步演进**才是真谛。
