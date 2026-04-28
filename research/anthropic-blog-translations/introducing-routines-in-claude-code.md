# 介绍 Claude Code 中的 Routines：自动化工作流

> 原文：[Introducing routines in Claude Code](https://claude.com/blog/introducing-routines-in-claude-code)
> 发布日期：2026年4月14日
> 翻译日期：2026年4月20日

![Routines 架构图](https://cdn.prod.website-files.com/68a44d4040f98a4adf2207b6/69de678887f94fb639698fa7_dd878b86.png)

## 概述

今天，我们在 Claude Code 中推出 **Routines**（研究预览版）。Routine 是一个你配置一次（包含 prompt、repo 和 connectors）的 Claude Code 自动化，然后按计划、从 API 调用或响应事件运行。Routines 在 Claude Code 的 Web 基础设施上运行，所以不依赖你的笔记本是否打开。

开发者已经使用 Claude Code 自动化软件开发周期，但直到现在，他们还需要自己管理 cron jobs、基础设施和 MCP 服务器等工具。Routines 可以访问你的 repos 和 connectors，所以你可以打包自动化并设置按计划或触发运行。

---

## 工作原理

### 定时 Routines

给 Claude Code 一个 prompt 和一个节奏（每小时、每晚或每周），它就会按计划运行：

- **每晚 2 点**：从 Linear 拉取最重要的 bug，尝试修复，并打开一个草稿 PR

如果你在 CLI 中使用 `/schedule`，那些任务现在就是 scheduled routines。

### API Routines

你也可以将 routine 配置为由 API 调用触发。每个 routine 有自己的端点和 auth token。POST 一条消息，获取回一个 session URL。将 Claude Code 接入你的 alerts、deploy hooks、内部工具——任何你能发出 HTTP 请求的地方：

- **读取 alert payload**，找到拥有服务，并在 #oncall 发布一个分类摘要和建议的第一步

### Webhook Routines（从 GitHub 开始）

将 routine 订阅为自动响应 GitHub 仓库事件而启动。Claude 会为每个符合你过滤条件的 PR 创建一个新 session 并运行你的 routine：

- **请标记触及 /auth-provider 模块的任何 PR**。此模块的任何更改都需要总结并发布到 #auth-changes。

Claude 为每个 PR 打开一个 session，并继续向该 session 提供该 PR 的更新，所以它可以处理后续如评论和 CI 失败。

---

## 团队正在构建什么

早期用户创建 routines 的一些常见模式：

### 定时 Routines

- **待办管理**：每晚分类新 issues、标记、分配，并在 Slack 发布摘要
- **文档漂移**：每周扫描合并的 PR，标记引用已更改 API 的文档，并打开更新 PR

### API Routines

- **部署验证**：你的 CD pipeline 在每次部署后发布，Claude 对新构建运行冒烟测试，扫描错误日志查找回归，并在 release channel 发布 go/no-go
- **Alert 分诊**：将 Datadog 指向 routine 的端点，Claude 拉取 trace，与最近部署关联，在值班人员打开页面之前准备好草稿修复
- **反馈解决**：文档反馈小部件或内部 dashboard 发布报告，Claude 在有问题的 repo 中打开 session，并起草更改

### GitHub Routines

- **库移植**：每个合并到 Python SDK 的 PR 都会触发一个 routine，将更改移植到并行的 Go SDK，并打开匹配的 PR
- **定制代码审查**：PR 打开时，在安全和性能上运行你团队自己的检查清单，在人工审查者查看之前留下内联评论

---

## 开始使用

Routines 现在对 Claude Code Pro、Max、Team 和 Enterprise 用户可用（需要启用 Web 上的 Claude Code）。前往 claude.ai/code 创建你的第一个 routine，或在 CLI 中输入 `/schedule`。

Routines 以与交互式 session 相同的方式消耗订阅使用限制。此外，routines 有每日限制：
- Pro 用户每天最多运行 5 个 routine
- Max 用户每天最多运行 15 个 routine
- Team 和 Enterprise 用户每天最多运行 25 个 routine

你可以通过额外使用运行超过这些限制的 routine。

---

## 我的总结与理解

### 核心价值

Routines 把 Claude Code 从"交互式工具"变成了"自动化平台"：

1. **定时执行**：不再需要自己管理 cron jobs
2. **API 触发**：可以集成到现有工作流中
3. **Webhooks**：直接响应 GitHub 事件
4. **云端运行**：不依赖本地笔记本

### 三种触发方式对比

| 触发方式 | 适用场景 | 延迟 | 复杂度 |
|----------|----------|------|--------|
| 定时 | 周期性任务（待办整理、文档同步） | 按计划 | 低 |
| API | CI/CD 集成、alert 响应 | 实时 | 中 |
| Webhook | GitHub 事件响应 | 事件驱动 | 中 |

### 典型场景深度解读

1. **定时 Routines**
   - 每晚 2 点：从 Linear 拉取 top bug，尝试修复，开 PR
   - 每周：扫描合并的 PR，标记需要更新的文档

2. **API Routines**
   - 部署后：自动跑冒烟测试，扫描错误日志
   - Alert 触发：拉取 trace，关联部署，拟合修复

3. **Webhook Routines**
   - PR 打开时：运行团队的安全/性能检查清单
   - 库移植：Python SDK PR 合并 → 自动移植到 Go SDK

### 对 OpenClaw 的启发

**我们可以借鉴的思路：**

1. **定时任务不需要自己搭服务器**
   - 直接使用 Claude Code Routines
   - 或类似地使用云端执行的 Agent

2. **通过 webhook 响应外部事件**
   - GitHub PR/Issue 事件
   - 内部系统的业务事件

3. **API 方式触发适合 CI/CD 集成**
   - 部署流水线中调用
   - 代码审查流程中触发

**具体可落地的场景：**
- [ ] 定时扫描开源依赖漏洞
- [ ] PR 自动触发代码审查
- [ ] 部署后自动验证
- [ ] 定时清理 stale branches

---

**一句话总结**：Routines 让 Claude Code 变成了**可编程的自动化平台**，而不只是交互工具。
