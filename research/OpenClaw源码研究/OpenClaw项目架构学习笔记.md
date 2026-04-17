---
created: '2026-03-27T11:02:55.954890'
tags:
- openclaw
- architecture
- study
---

## 项目概览

OpenClaw 是一个**开源的个人 AI 助手**，核心理念是"AI that actually does things"——运行在你自己的设备上，支持你常用的通讯频道，遵守你自定义的规则。

项目曾经历多次更名：Warelay → Clawdbot → Moltbot → OpenClaw

- **官网**：https://openclaw.ai
- **文档**：https://docs.openclaw.ai
- **技术栈**：TypeScript (ESM)，Node 22+/Bun，pnpm monorepo
- **License**：MIT

---

## 整体架构

### 顶层目录结构

| 目录/文件 | 说明 |
|---|---|
| src/ | 核心源码 |
| extensions/ | 插件（内部目录名，对外叫 plugins） |
| apps/ | 端应用（iOS、Android、macOS、shared） |
| packages/ | 共享包（clawdbot、moltbot 等历史遗留） |
| ui/ | Web 前端 |
| docs/ | 文档（Mintlify 托管） |
| skills/ | 内置 skill 集合 |
| scripts/ | 构建/发布脚本 |
| dist/ | 构建产物 |

### src/ 核心模块划分

| 模块 | 路径 | 职责 |
|---|---|---|
| CLI 入口 | src/cli/ | 命令行指令注册与解析 |
| 命令实现 | src/commands/ | 具体命令逻辑 |
| Gateway | src/gateway/ | 网关服务，消息路由控制面 |
| Agents | src/agents/ | AI Agent 编排，系统提示词，skill 管理 |
| Auto-reply | src/auto-reply/ | 消息自动回复逻辑 |
| Channels | src/channels/ | 消息频道抽象层 |
| Routing | src/routing/ | 消息路由 |
| Plugins | src/plugins/ | 插件安装/更新/ClawHub 对接 |
| Plugin SDK | src/plugin-sdk/ | 对外暴露的插件开发 SDK |
| Config | src/config/ | 配置管理（Zod schema） |
| Infra | src/infra/ | 基础设施（ClawHub 客户端等） |
| Media | src/media/ | 媒体处理管道 |
| ACP | src/acp/ | Agent Communication Protocol |
| Canvas Host | src/canvas-host/ | Canvas 渲染宿主 |
| Context Engine | src/context-engine/ | 上下文管理 |
| Memory | src/memory/ | 记忆系统 |
| Sessions | src/sessions/ | 会话管理 |
| Hooks | src/hooks/ | 生命周期钩子 |
| Cron | src/cron/ | 定时任务 |
| TTS | src/tts/ | 文字转语音 |

---

## 支持的通讯频道

**内置频道（src/ 下）：**
Telegram、Discord、Slack、Signal、iMessage、WhatsApp (web)

**扩展频道（extensions/ 下）：**
Microsoft Teams、Matrix、Zalo、Zalo Personal、Voice Call、Google Chat、Feishu、LINE、IRC、Mattermost、Nextcloud Talk、Nostr、Synology Chat、Tlon、Twitch、BlueBubbles

---

## 支持的 AI 模型提供商（extensions/）

Anthropic、OpenAI、Google（Gemini）、Mistral、Groq、DeepSeek、Ollama、Qwen、Moonshot（Kimi）、xAI、Amazon Bedrock、Azure（Microsoft）、Cloudflare AI Gateway、Vercel AI Gateway、HuggingFace、Together、OpenRouter、NVIDIA、vLLM、sGLang、BytePlus、VolcEngine、Minimax、Chutes、Venice、Github Copilot 等 40+ 个

---

## 插件（Plugin）体系

- 插件目录：extensions/*（workspace packages）
- 插件注册表：ClawHub（https://clawhub.ai）
- 插件 SDK 公共接口：openclaw/plugin-sdk/*
- 安装方式：npm install --omit=dev（生产依赖）
- 命名规范：包名以 @openclaw/<id> 为主

**核心原则：**
- 可选能力优先走插件，不内置到 core
- 插件不可直接 import core src/**，只能用 SDK 公共接口
- Memory 插件同一时间只能激活一个

---

## Skills 体系

- 内置 skill 存放于 skills/ 目录
- 社区 skill 发布到 ClawHub（clawhub.ai）
- 本地 skill 安装目录：~/.claude/skills/（Cursor）
- ClawHub CLI 管理：clawhub install/update/publish/search

---

## 技术规范

- **语言**：TypeScript ESM，严格类型，禁止 any
- **格式/Lint**：Oxfmt + Oxlint
- **测试**：Vitest，V8 coverage，阈值 70%
- **构建**：pnpm build（tsdown）
- **包管理**：pnpm（同时支持 bun）
- **运行时**：Node 22+，Bun（开发/测试优先）

### 重要命令

```
pnpm install        # 安装依赖
pnpm build          # 类型检查 + 构建
pnpm test           # 运行测试
pnpm check          # lint + format 检查
pnpm format:fix     # 格式化
pnpm openclaw ...   # 开发模式运行 CLI
```

---

## 端应用（apps/）

| 目录 | 说明 |
|---|---|
| apps/ios/ | iOS App |
| apps/android/ | Android App |
| apps/macos/ | macOS 菜单栏 App（SwiftUI，Observation 框架） |
| apps/shared/ | 跨端共享代码 |

---

## MCP 支持

通过 mcporter（https://github.com/steipete/mcporter）集成 MCP，解耦于 core runtime，支持动态增删 MCP server 无需重启 gateway。

---

## 当前重点方向

**优先级高：**
1. 安全与安全默认值
2. Bug 修复与稳定性
3. 安装可靠性与首次使用体验

**次优先级：**
- 支持更多模型提供商
- 完善主流消息频道支持
- 性能与测试基础设施
- 更好的 computer-use 与 agent 能力
- macOS/iOS/Android/Windows/Linux 伴侣应用