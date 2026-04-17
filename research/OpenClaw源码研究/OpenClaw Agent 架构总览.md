---
created: '2026-03-30T10:52:23.082127'
tags:
- openclaw
- agent
- architecture
---

# OpenClaw Agent 架构总览

> 源码路径：~/myspace/openclaw
> 分析时间：2026-03-30

## 项目定位

OpenClaw 是一个**多渠道 AI 助手网关**，通过插件化架构桥接各种消息渠道（Telegram、Discord、Signal、WhatsApp 等）与 AI 模型（Claude、GPT、Gemini 等）。

## 核心目录结构

| 目录 | 职责 |
|------|------|
| `src/agents/` | Agent 循环、系统提示构建、Skills 处理、子 Agent 编排 |
| `src/channels/` + `extensions/` | 渠道插件，统一 channel-contract 接口 |
| `src/plugin-sdk/` | 对外暴露的 Plugin SDK 公共接口 |
| `src/config/` + `src/infra/` | 配置管理、路径守卫、重试机制 |
| `src/gateway/` + `src/routing/` | Gateway 守护进程、消息路由 |

## 关键技术选型

- **运行时**：Bun + Node 22+（兼顾速度与生态）
- **语言**：TypeScript ESM，严格类型，避免 any
- **Agent 核心**：`@mariozechner/pi-*`（封装 Claude API 调用循环）
- **测试框架**：Vitest
- **Lint/Format**：Oxlint + Oxfmt（Rust 实现，极速）
- **包管理**：pnpm workspace（Monorepo）

## 请求全生命周期

```
用户消息到达（Telegram/Discord/Signal/…）
    ↓
渠道插件 inbound-envelope 解析 → 路由
    ↓
run-state-machine：debounce → 鉴权 → 会话绑定
    ↓
buildAgentSystemPrompt 组装 system prompt（含 Skills 注入）
    ↓
上下文超窗口？→ Compaction（generateSummary 摘要历史）
    ↓
pi-embedded-runner 调用 Claude API（流式）
    ↓
回复含工具调用？→ 执行工具 → 结果注入 → 继续循环
    ↓
文本分块 → 渠道回复
```

## System Prompt 分层结构（优先级从高到低）

1. **Safety**（最高）：无独立目标，安全边界，不得绕过防护
2. **Skills（mandatory）**：扫描 available_skills → 选择最精确的一个 → 读取并遵循
3. **Tooling + 工具策略**：工具清单（按策略过滤）、调用风格指导
4. **Workspace + 身份 + 时区**：工作目录、授权发送者、时间信息
5. **Context Files（Project Context）**：SOUL.md、AGENTS.md、用户编辑的文件

## Prompt Mode 三级策略

| 模式 | 用途 | 包含内容 |
|------|------|---------|
| `full` | 主 Agent | 所有 Section（Skills/Memory/时区/Messaging/TTS/Docs/心跳/Silent Reply） |
| `minimal` | 子 Agent | 仅 Tooling/Workspace/Runtime，减少 token 开销 |
| `none` | 极简 | 仅一行："You are a personal assistant running inside OpenClaw." |

## Compaction（上下文压缩）

- 文件：`src/agents/compaction.ts`
- 触发：token 超过 `BASE_CHUNK_RATIO(40%)`，加 `SAFETY_MARGIN(1.2)` 安全边距
- 策略：二分裁剪历史 → `generateSummary` 生成摘要
- **标识符保全**：默认 `strict` 模式，强制保留 UUID/Hash/API key/URL，防止 LLM 摘要时重构这些值

## 子 Agent 编排

- **推送模型**：子 Agent 完成后通过 `subagent-announce-queue` 主动推送，主 Agent 无需轮询
- **深度限制**：`subagent-depth.ts` 强制最大嵌套深度，防止无限递归
- **ACP 模式**：当用户要求在 Codex/Claude Code 中执行时，调用 `sessions_spawn` 带 `runtime: "acp"`

## 相关笔记

- [[OpenClaw Skills 机制深度分析]]