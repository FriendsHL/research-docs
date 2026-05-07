# 构建 Claude Code 的经验：Prompt Cache 决定一切

> 原文：[Lessons from building Claude Code: Prompt caching is everything](https://claude.com/blog/lessons-from-building-claude-code-prompt-caching-is-everything)
> 发布日期：2026年4月30日
> 翻译日期：2026年5月7日
> 作者：Thariq Shihipar，Claude Code 团队技术成员

## 概述

prompt cache 是 Claude Code 架构的根基——它对长时间运行的 agentic 产品在**降低成本和改善延迟**方面起着决定性作用。

### 核心原则

> **"Cache rules everything around me."**（Cache 主宰一切）

对 Agent 而言，prompt cache 不只是优化项，**它是可行性和性能的前提**。

---

## 为 Cache 友好性设计你的 prompt 布局

prompt cache 通过**前缀匹配**工作。最优结构是：

1. **静态 system prompt 与 Tools**（全局缓存）
2. **CLAUDE.md**（项目内缓存）
3. **Session 上下文**（session 内缓存）
4. **会话消息**

这种顺序"**出乎意料地脆弱**"——任何时间戳的注入、不确定的工具排序、或工具参数变化都可能让它崩掉。

---

## 用 messages 来传递更新

当某些信息过期时（时间变了、文件更新了），**不要去修改 prompt**——而是在后续轮次里通过 messages 传递更新信息，使用诸如 `<system-reminder>` 这样的标签。这样能保持 cache 完整性。

---

## 不要在 session 中切换模型

prompt cache 是**模型特定**的。在一次对话中从 Opus 切换到 Haiku，会让整个 cache 重建——这比一直用 Opus 还贵。**模型切换最好通过 subagent 来处理。**

---

## 不要在 session 中增删工具

更改工具集合会**让整个对话的 cache 前缀失效**。工具应该在整个 session 中保持不变。

---

## Plan Mode 与 Cache 设计

Plan Mode 演示了"cache 友好"的功能设计：

- 进入 plan mode 时**不交换工具集**——所有工具仍然在场
- Agent 收到一条 system message，解释当前的限制
- `EnterPlanMode` 和 `ExitPlanMode` **本身就是可调用的工具**

这样 Agent 可以**自主切换模式而不破坏 cache**。

---

## Tool Search 与延迟加载

对于含有大量 MCP 工具的项目，`defer_loading` 提供轻量级的工具桩（仅含名字）。模型在需要时通过工具搜索发现它们，**只有在被选中时才加载完整 schema**——保持 cache 前缀的稳定性。

---

## 在不破坏 Cache 的前提下做 Compaction

当上下文塞满需要做总结（compaction）时，需要 cache 安全的处理方式：

### 问题

独立的 API 调用使用不同的 system prompt 且没有工具——它**从第一个 token 开始就偏离了父对话的 cache 前缀**，所有缓存收益都丢失了。

### 解决方案

- 使用与父对话**完全相同**的 system prompt、user 上下文、system 上下文和工具定义
- 把 compaction prompt 作为**一条新的 user message** 追加进去
- 这样可以复用 cache

实现上需要维护一个 "**compaction buffer**" 来存放总结输出。

> **API 现在已经内置了支持这些模式的 compaction 能力。**

---

## 经验总结

1. **prompt cache 通过前缀匹配工作**——任何前缀的改动都会让下游 cache 失效
2. **用 messages 而非 system prompt 修改来传递更新**
3. **不要在对话中切换工具或模型**；通过工具来切换模型状态
4. **像监控 uptime 一样监控 cache 命中率**；把 cache 失效当成事故对待
5. **fork 类操作（compaction、summarization）必须共享父前缀**才能命中 cache

---

*开始使用 Claude Code 吧。*
