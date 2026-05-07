# Claude Managed Agents 新功能：Dreaming、Outcomes 与多 Agent 编排

> 原文：[New in Claude Managed Agents: dreaming, outcomes, and multiagent orchestration](https://claude.com/blog/new-in-claude-managed-agents)
> 发布日期：2026年5月6日
> 翻译日期：2026年5月7日

## 概述

Anthropic 为 Claude Managed Agents 推出三大重要功能：**Dreaming（研究预览）、Outcomes、多 Agent 编排**，以及 webhook。这些能力让 Agent 能够自我提升、验证自己的工作，并通过并行化处理复杂任务。

---

## 用 Dreaming 构建自我提升的 Agent

**它做什么：** Dreaming 是一个**计划性后台过程**，会回顾 Agent 的 session 和记忆库，提取模式并整理记忆，实现持续改进。

**核心能力：**

- 识别**反复出现的错误**与工作流模式
- 浮现跨多个 Agent 的团队偏好
- 重组记忆以保持信号质量
- 可以**全自动运行，也可以让人在变更前审阅**

![Claude Managed Agents Dreaming](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69f8e9ad765c7eed52dcf468_Claude-Managed-Agents-Blog-Followup-Dreaming.png)

系统将 **实时记忆**（Agent 在工作中即时捕获学习）与 **session 间的精炼**（dreaming 跨 Agent 抽取共享学习）结合在一起。

**可用性：** Claude 平台上的研究预览，开发者可以申请试用。

---

## 交付更好的 outcome

**它做什么：** Outcomes 让开发者**定义成功的 rubric**，Agent 朝着这个目标工作，并由一个独立的 grader 根据指定标准评估输出。

**工作机制：**

1. Agent 接收一份描述"什么算成功"的 rubric
2. 一个独立的 grader 在**隔离的上下文窗口**中评估输出
3. grader 指出需要的修正
4. Agent 反复修订，直到达到质量阈值

**内部测试效果数据：**

> - 与标准 prompting 相比，任务成功率提升 **最多 10 个百分点**
> - **docx 文件生成**任务成功率 +8.4%
> - **pptx 文件生成**任务成功率 +10.1%

**最适合的场景：**

- 需要详尽覆盖的细节型任务
- 主观质量（品牌语气、设计规范）

**附加能力：** Agent 完成工作时通过 webhook 通知。

---

## 用多 Agent 处理复杂任务

**多 Agent 编排支持：**

- **Lead Agent** 把工作拆成专门的部分
- **Specialist subagent** 拥有各自的模型、prompt 和工具
- 在共享文件系统上**并行执行**
- Agent 之间的事件历史**持久化**
- 在 Claude Console 中获得完整的**执行追踪**

**示例工作流：** 一个调查型 lead Agent 把任务分发给 subagent，让它们**同时分析**部署历史、错误日志、指标和支持工单。

![多 Agent 编排 UI](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69f8ea208aefcf18345ee3ef_Claude-Managed-Agents-Blog-Followup-Sessions-UI.png)

---

## 团队们正在构建什么

### Harvey（法律科技）

- 用 Managed Agents 处理复杂的法律工作（起草、文档生成）
- Dreaming 让 Agent 能跨 session 保留文件类型 workaround 和工具使用模式
- 成果："**完成率在他们的测试中提升了约 6 倍**"

### Netflix 平台团队

- 构建了一个分析 Agent，处理**数百个构建产生的日志**
- 多 Agent 编排并行分析多个批次
- 重点：识别数千个应用中**反复出现的问题**

### Spiral by Every（写作平台）

- 用多 Agent 编排和 Outcomes 构建 API/CLI 写作 Agent
- **架构**：lead Agent（Haiku）做请求路由；subagent（Opus）负责起草
- **并行执行**多个起草请求
- **Outcomes 强制**：草稿根据编辑原则和记忆中存储的用户语气打分
- 只有质量过关的草稿才会返回

### Wisedocs（文档核验）

- 文档质量检查 Agent 用 Outcomes 确保符合规范
- 成果："**审查速度提升 50%**，同时仍与团队标准保持一致"

---

## 上手

**当前状态：**

- **Dreaming**：研究预览（需申请）
- **Outcomes**：公开 beta
- **多 Agent 编排**：公开 beta
- **Memory**：公开 beta

**资源：**

- 申请 Dreaming：claude.com/form/claude-managed-agents
- 文档：platform.claude.com/docs/en/managed-agents/overview
- 部署入口：platform.claude.com 的 Claude Console

---

## 相关阅读

- "Claude Security is now in public beta"（2026年4月30日）
- "New connectors in Claude for everyday life"（2026年4月23日）
- "Bringing memory to Claude"（2025年9月11日）
- "Built-in memory for Claude Managed Agents"（2026年4月23日）
