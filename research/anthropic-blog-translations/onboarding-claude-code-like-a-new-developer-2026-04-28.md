# 把 Claude Code 当成新员工来 onboarding：来自 17 年开发经验的教训

> 原文：[Onboarding Claude Code like a new developer: Lessons from 17 years of development](https://claude.com/blog/onboarding-claude-code-like-a-new-developer-lessons-from-17-years-of-development)
> 发布日期：2026年4月28日
> 翻译日期：2026年5月7日

## 概述

Brendan MacLean 是华盛顿大学 MacCoss Lab 的 Claude 开发者大使。他把成熟的开发者 onboarding 流程应用到 Claude Code 上。该实验室维护着 Skyline，一款 2008 年起开发的开源蛋白质分析软件，包含 **70 多万行 C# 代码**。

---

## 挑战

MacCoss Lab 庞大的代码库让 AI 工具难以发挥作用。最初在浏览器中使用 Claude.ai 的尝试需要把整个文件复制过去，并且只能局限在自包含的小问题上，无法获得任何项目上下文。**每次 session 都像从零开始。**

> 关键挑战：每次和 Claude.ai 的 session 都像是从零开始——它完全不理解 Skyline 是什么、组件之间的关系如何，以及 17 年开发积累下来的约定。

---

## 解决方案：结构化的上下文管理

MacLean 实施了三大核心策略：

### 1. 专门的 AI 上下文仓库

他创建了一个独立的 GitHub 仓库 `pwiz-ai`，把所有 AI 相关的上下文放进去，与主代码库分离。这种做法让上下文可以跨所有分支和开发阶段复用。

仓库根目录的 `CLAUDE.md` 提供环境配置和文档索引——奠定基础理解，而不是把所有专业知识都嵌入进来。

### 2. 用 Skill 编码领域知识

Skill 用来编码 Claude 可以自动或手动加载的特定能力。MacLean 关键的几个 skill 包括：

- **`debugging` skill**：阻止"猜测式试错"模式，要求先定位根因再修复
- **`skyline-development` skill**：让 Claude 熟悉项目结构和文档体系
- **`version-control` skill**：编码项目特有的 commit 与 PR 约定

Skill 遵循"**引用而非内嵌**"的原则——指向中央文档，而不是复制内容。

### 3. 用 MCP 集成接入实时数据

Model Context Protocol（MCP）集成让 Claude 实时访问：

- 测试结果与异常报告
- Skyline 基础设施中的支持线程
- GitHub 上的 release tag
- LabKey Server 的数据

---

## 可衡量的成果

### Files View 面板项目

一个 UI 项目在原开发者离开后停滞了一整年。MacLean 用 Claude Code 在 **两周内** 完成了它，最终 commit 由 Claude 共同署名。

### 自动化的基础设施

- **2,000+ 张教程截图复现**：完全自动化，几乎 100% 可重现
- **每日总结邮件**：Claude 从夜间测试基础设施、测试失败、异常和支持线程中拉取数据，生成报告
- **MCP Server**：Claude 自己写了 C# 和 Python 两个版本来做数据集成

### 开发者生产力的转变

实验室的开发者现在花更多时间 **指挥 Claude Code**，而不是自己直接写代码。一位原本怀疑 AI 的开发者，借助 Claude Code 构建了一个用于可视化离子迁移率数据的 mobilogram 面板。

> 文化层面的影响：我看到几乎每个人都开始尝试一些有趣的新功能——这些功能他们之前可能因为被其他工作压得喘不过气而不敢去碰。

---

## 处理遗留代码库的关键建议

### 上下文是地基

To-do list 和 plan 不能跨 session 持久化——只有维护好的上下文可以。MacLean 强调：

> 不要指望奇迹。投入时间去建立和维护你的上下文层。把它当成一个项目工件来对待：版本化它、扩展它、维护它。

### 构建 Skill 库

把领域知识编码成可复用的 skill。skill 用轻量的引用维护，而不是把中央文档的内容复制一份。

### 在数据访问重要时使用 MCP

为需要真实数据的场景构建集成：测试结果、异常报告、支持线程。**这一点对开源项目尤其重要**——因为开源项目的"机构知识"必须能跨越个人贡献者持续存在。

---

## 为什么这种方式有效

传统的开发者 onboarding 也是这样：你不会把一份 70 万行的代码库塞给新人，期待他们第一天就出活。你会先给他们一个有边界的小项目，带着他们走一遍，然后随着理解的加深逐步扩大范围。

用 Claude Code 构建的上下文运作方式完全相同。一旦对一个代码库足够了解，**人类工程师和 Claude 都能跨分支、跨时间点高效工作。**

---

## 对开源项目的影响

对开源项目而言，构建上下文层有特殊意义。开源项目没有 onboarding 预算，也无法保证贡献者的稳定性，但**有据可查的上下文可以贯穿项目的整个生命周期**——这是人类的机构知识做不到的。`pwiz-ai` 仓库正是这一思路的体现：上下文属于项目本身，不属于任何个人贡献者。

---

## 历史小注

Anthropic 的联合创始人 Dario Amodei 曾经是 MacCoss Lab 的成员。
