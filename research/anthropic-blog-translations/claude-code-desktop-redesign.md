# 重新设计 Claude Code 桌面应用：为并行 Agent 而生

> 原文：[Redesigning Claude Code on desktop for parallel agents](https://claude.com/blog/claude-code-desktop-redesign)
> 发布日期：2026年4月14日
> 翻译日期：2026年4月20日

![桌面应用概览](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/6903d22d0099a66d72e05699_33ddc751e21fb4b116b3f57dd553f0bc55ea09d1-1000x1000.svg)

## 概述

今天我们发布了重新设计的 Claude Code 桌面应用。新应用包括：一个用于管理多个 session 的新侧边栏、用于安排工作空间的拖放布局、集成终端和文件编辑器，以及性能和体验改进。

---

## 为现代 Agent 工作方式而构建

对于许多开发者来说，Agent 工作的形态已经改变了。你不是输入一个 prompt 然后等待。你是在一个 repo 中启动重构，在另一个中修复 bug，在第三个中写测试，检查每个的结果，在偏离时调整方向，并在发布前审查 diffs。

新应用为现在的 Agent 编程感觉构建：**多件事同时进行，你在 orchestrator 座位上**。

![并行 Session 管理](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/6903d22d7d4c10df6024f7bc_ee580919acaba2ddc07425f7a7390c8962cadc94-1000x1000.svg)

---

## 并行运行 Sessions

新侧边栏将每个活跃和最近的 session 放在一个地方。跨多个 repos 启动工作，并在结果到达时在它们之间移动。

- 你可以按状态、项目或环境过滤
- 或按项目分组侧边栏，以便更快找到和恢复 session
- 当 session 的 PR 合并或关闭时，它会自动归档，让侧边栏保持聚焦于 live 的东西

当你需要在任务中间提问时，你可以打开一个 side chat（⌘ + ; 或 Ctrl + ;）来分支对话。Side chats 从主线程拉取上下文，但不添加任何东西回线程，以避免误导你的任务。

![集成工具展示](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/6903d22e13864f88ea55c2d8_b5c98d26c46edc43193e7f7e28a00633a538bb9c-1000x1000.svg)

---

## 审查和发布不用离开应用

重新设计将更多常用工具带入应用，所以你可以在不跳转到编辑器的情况下审查、调整和发布 Claude 的工作：

- **集成终端**：在 session 旁边运行测试或构建
- **应用内文件编辑器**：打开文件，直接进行 spot edits，并保存更改
- **更快的 diff viewer**：为大型变更集重建，性能更好
- **扩展预览**：除了在 preview pane 运行本地 app 服务器，还可以在应用内打开 HTML 文件或 PDF

每个 pane 都是拖放的。把终端、preview、diff viewer 和 chat 排列成适合你工作的任何网格。

---

## 适配你的技术栈

桌面应用现在与 CLI 插件有同等能力。如果你的组织集中管理 Claude Code 插件，或者你已经在本地安装了它们，它们在桌面应用中就像在你的终端中一样工作。

你仍然可以本地或云端运行 session。SSH 支持现在扩展到 Mac 和 Linux，所以你可以在任一平台上将 session 指向远程机器。

---

## 自定义你的工作方式

三种视图模式——Verbose、Normal 和 Summary——让你从完全透明调用工具到只显示结果之间调节界面。

新键盘快捷键覆盖 session 切换、生成和导航；按 ⌘ + /（或 Ctrl + /）查看完整列表。一个新的 usage 按钮可以一目了然地看到你的上下文窗口和 session 使用情况。

---

## 开始使用

重新设计的桌面应用现在对所有 Pro、Max、Team 和 Enterprise 计划的 Claude Code 用户可用，也可通过 Claude API 使用。

下载应用，或者如果你已经安装了就更新并重启。探索文档以了解更多。

---

## 我的总结与理解

### 核心变化

这个重新设计反映了** Agent 工作的新范式**：

1. **并行多个 session**：不再一个一个来，而是同时管理多个
2. **侧边栏统一视图**：所有 session 一目了然
3. **应用内完成更多工作**：不需要跳转到其他工具

### 功能亮点详解

| 功能 | 解决的问题 | 使用场景 |
|------|------------|----------|
| 并行 session 侧边栏 | 多任务切换麻烦 | 同时进行重构、bug 修复、测试 |
| Side chat | 临时提问污染主线程 | 中途确认、追问细节 |
| 拖放布局 | 工作区固定不灵活 | 按个人习惯排列 |
| 集成终端 | 频繁切换窗口 | 跑测试、build |
| 应用内文件编辑 | 跳转编辑器打断流程 | 快速修复 |
| 更快的 diff viewer | 大diff加载慢 | 审查大型重构 |

### 三个视图模式的选择

- **Verbose**：学习阶段，想看 Claude 怎么思考
- **Normal（默认）**：日常使用，平衡信息和效率
- **Summary**：专家模式，只看结果

### 对我们的启发

**UI 设计层面：**
- [ ] 多 session 并行管理的侧边栏
- [ ] 内置 terminal 和文件编辑
- [ ] 拖放布局的灵活性
- [ ] Side chat 机制（临时分支对话）

**交互模式层面：**
- 从"一对一"转向"一对多"的 orchestrator 模式
- 降低上下文切换成本
- 提供渐进式信息密度（Verbose → Summary）

---

**一句话总结**：桌面应用的重新设计反映了 **Agent 工作从"单一交互"到"多任务并行"的范式转变**。
