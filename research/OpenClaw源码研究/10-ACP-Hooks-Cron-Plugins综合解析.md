Fetching document 2752573635......
# OpenClaw 源码深度解析：ACP / Hooks / Cron / Plugins（2026-03-24）


> 本文是 OpenClaw 架构学习系列第二篇，记录 2026-03-24 的源码调研内容，涵盖 ACP 协议、Hooks 系统、Cron 定时任务、Heartbeat 机制、Plugins 与 Channel SDK 五大模块。
>
>


---

## 一、ACP 模块（Agent Communication Protocol）

### 1.1 核心定位

ACP 是 OpenClaw 与外部 Coding Agent（如 acpx/Claude Code）之间的**结构化通信协议**，源码路径：`/app/src/acp/`。

**ACP vs tmux 的本质区别：**


| ACP | tmux |
| --- | --- |
| 面向对象 | 程序（结构化 JSON 事件） | 人类（终端字符输出） |
| 通信方式 | 事件流（start/output/end） | 文本字符流 |
| 可解析性 | 完全可解析 | 需要解析 ANSI 转义码 |
| 取消支持 | 优雅取消（cancel 事件） | 发 SIGINT 信号 |

**一句话总结：tmux 是给人看的终端工具，ACP 是给程序解析的结构化协议。**

### 1.2 核心文件


- `types.ts`：协议类型定义（ACPSession、ACPEvent、ACPMessage）
- `server.ts`：ACP HTTP Server，接受外部 coding agent 连接
- `translator.ts`：pi-agent-core 事件 → ACP 标准事件转换
- `session.ts`：管理"入站"ACP 客户端连接（谁连进来了）
- `session-mapper.ts`：ACP session ↔ OpenClaw session 映射
- `control-plane/`：管理"出站"派发给 coding agent 的 session
- `runtime/`：ACP 运行时，注册后端实现（acpx）

### 1.3 ACP vs acpx

```Plain
// 代码块
ACP（接口/协议）               acpx（具体实现）
/app/src/acp/              /app/extensions/acpx/src/
─────────────              ──────────────────────────
定义通信格式                   实现实际的 coding agent 行为
相当于"插座"                  相当于"插头"
通过 registerAcpRuntimeBackend("acpx", ...) 注册

```

### 1.4 ACP vs Subagent


| ACP Session | Subagent |
| --- | --- |
| 执行方式 | 独立进程（外部 coding agent） | 进程内函数调用 |
| 会话持久性 | 持久（可恢复） | 一次性 |
| 取消能力 | 优雅取消 | 无（或强制中断） |
| 事件流 | 结构化事件流 | 无 |
| 定位 | "一等公民 coding agent" | 轻量任务派发 |


---

## 二、Hooks 系统

### 2.1 架构概览

Hooks 是 OpenClaw 的**事件驱动扩展系统**，源码路径：`/app/src/hooks/`。

每个 Hook = `HOOK.md`（frontmatter 声明监听的 events）+ `handler.ts`（具体实现逻辑）。

**事件类型（粗粒度 → 细粒度）：**

```Plain
// 代码块
command          → command:new、command:reset
session          → （session 生命周期）
agent            → agent:bootstrap
gateway          → gateway:startup
message          → message:received、message:sent

```

### 2.2 Hook 触发位置


| 触发文件 | 触发的事件 |
| --- | --- |
| `gateway/server-startup.ts` | `gateway:startup` |
| `infra/outbound/deliver.ts` | `message:sent` |
| `auto-reply/reply/dispatch-from-config.ts` | `message:received` |
| `auto-reply/reply/commands-core.ts` | `command:new`、`command:reset` |
| `agents/bootstrap-hooks.ts` | `agent:bootstrap` |

### 2.3 内置 Hook 说明


| Hook | 功能 |
| --- | --- |
| `session-memory` | `/new` 时将对话摘要写入 (http://MEMORY.md "MEMORY.md") |
| `command-logger` | 记录所有斜杠命令日志 |
| `boot-md` | Agent 启动时将 (http://BOOT.md "BOOT.md") 注入上下文 |
| `bootstrap-extra-files` | `agent:bootstrap` 时注入额外文件（(http://AGENTS.md "AGENTS.md")、(http://TOOLS.md) 等），适合 monorepo 场景 |

### 2.4 Gmail Watcher

Gmail Watcher 是 Hooks 系统中的独立子系统：


- 启动 `gog gmail watch serve` 子进程
- 接收 Gmail Pub/Sub 推送
- 将新邮件事件转换为 OpenClaw 消息，触发 agent 处理

### 2.5 `command` 事件含义

`command` 事件对应用户发送的**斜杠命令**（`/new`、`/reset`、`/status` 等），在 `commands-core.ts` 中触发，不是编程意义上的"命令"。


---

## 三、Cron 定时任务系统

### 3.1 三种调度模式

```Plain
// 代码块
at      → 一次性执行（指定时间点）
every   → 固定间隔循环（every: "1h"）
cron    → 标准 cron 表达式（"0 9 * * *"）

```

### 3.2 两种执行目标（关键区别）


| main（systemEvent） | isolated（agentTurn） |
| --- | --- |
| 执行方式 | 注入主 session 消息队列，等 heartbeat 唤醒 | 独立 session，立即执行 |
| 上下文 | **共享主 session 历史** | 完全隔离 |
| 适合场景 | 需要上下文感知的任务 | 纯自动化任务（定时报告、检查脚本） |
| Session 复用 | 无需配置 | freshness 机制（daily/idle 模式） |

### 3.3 两种 Payload


- `systemEvent`：系统事件注入（主 session 感知到）
- `agentTurn`：完整 agent 执行（isolated 模式专用）

### 3.4 Delivery 模式

```Plain
// 代码块
none      → 静默执行，结果不推送
announce  → 执行完毕后推送结果到指定 channel
webhook   → 推送到外部 webhook URL

```

### 3.5 可靠性保障


- **错误退避**：30s → 1min → 5min → 15min → 60min
- **重启补偿**：Gateway 重启后检查 missed jobs，自动补跑错过的任务


---

## 四、Heartbeat 机制

### 4.1 (http://HEARTBEAT.md "HEARTBEAT.md") 的真实作用

**常见误解**：(http://HEARTBEAT.md) 在 agent 启动时加载到上下文中。

**正确理解**：(http://HEARTBEAT.md) **不是**上下文文件，而是每次 heartbeat 触发时读取的**任务清单**，读完后作为消息体发给 agent，agent 自行判断是否需要执行某项任务。

```Plain
// 代码块
heartbeat 触发
    ↓
heartbeat-runner 读 HEARTBEAT.md
    ↓
把内容作为消息体发给 agent（Provider = "heartbeat"）
    ↓
Agent 读到清单 → 判断当前时间/状态 → 执行或不执行
    ↓
无事可做 → 回复 HEARTBEAT_OK → OpenClaw 静默处理
有任务完成 → 回复结果 → 推送到大象

```

### 4.2 五种触发来源


| 来源 | reason 标记 | 典型场景 |
| --- | --- | --- |
| 定时器（默认30min） | `interval` | 无事发生也定期检查 |
| Cron main job 完成 | `cron:<jobId>` | 定时任务需要 agent 处理 |
| exec/bash 脚本退出 | `exec:<id>:exit` | raptor_monitor.sh 执行完毕 |
| 外部进程事件 | `exec-event` | 系统 cron 有输出 |
| Hook 主动唤醒 | `hook:<jobId>` | Hook 处理完毕需要 agent |


---

## 五、Plugins / Channel SDK

### 5.1 两套系统的关系

```Plain
// 代码块
OpenClaw 扩展体系
├── Plugins（通用插件）         → /app/src/plugins/
│     提供：工具、HTTP 路由、CLI 命令、Provider、Service、Hook
│
└── Channel Plugin（渠道插件）  → /app/src/channels/plugins/
      是 Plugins 的特化子集，专门实现 IM 接入

```

### 5.2 Plugin 完整能力

一个 Plugin（npm 包）可通过 `OpenClawPluginApi` 注册以下能力：


| 注册方法 | 说明 |
| --- | --- |
| `registerTool()` | 给 agent 新增工具（LLM 可调用） |
| `registerHook()` | 监听系统事件（Internal Hook） |
| `registerHttpHandler()` | 注册 HTTP 接口 |
| `registerChannel()` | 接入新 IM 渠道 |
| `registerGatewayMethod()` | 注册 Gateway JSON-RPC 方法 |
| `registerCli()` | 扩展 CLI 命令 |
| `registerService()` | 注册后台常驻服务（随 Gateway 启停） |
| `registerProvider()` | 接入新的 AI 模型 Provider |
| `registerCommand()` | 注册斜杠命令（绕过 LLM 直接处理） |
| `on(hookName, handler)` | 注册 Plugin Hook（拦截 LLM 流程） |

### 5.3 Plugin Hooks vs Internal Hooks


| Internal Hooks | Plugin Hooks |
| --- | --- |
| 定义方式 | `HOOK.md` + `handler.ts`（文件系统） | `api.on("hook_name", handler)`（代码） |
| 粒度 | 粗（message/session/command） | **细**（before_tool_call、llm_input…） |
| 拦截能力 | 只能监听，不能修改 | **可以修改/拒绝**（before_tool_call 可以拒绝工具调用） |
| 作者 | 用户/工作区开发者 | 插件开发者（npm 包） |

**Plugin Hooks 完整事件列表：**

```Plain
// 代码块
before_model_resolve   → 可修改使用哪个 model
before_prompt_build    → 可修改 system prompt
before_agent_start     → agent 启动前
llm_input              → 发给 LLM 前（可看到完整消息列表）
llm_output             → LLM 返回后
before_tool_call       → 工具调用前（可拒绝/修改参数）
after_tool_call        → 工具调用后
tool_result_persist    → 工具结果写入 transcript 前
before_message_write   → 消息写入 transcript 前
before/after_compaction → 上下文压缩
session_start/end      → session 生命周期
message_received/sending/sent → 消息收发
subagent_spawning/spawned/ended → subagent 生命周期
gateway_start/stop     → Gateway 生命周期

```

### 5.4 Channel Plugin 的核心工作：双向格式转换

**Channel Plugin = 格式转换器 + 能力声明器 + 协议适配器**

发消息方向（OpenClaw → Channel）：

```Plain
// 代码块
OpenClaw 内部 Markdown
    ↓  chunker（分块 + 格式转换）
各 IM 原生格式
    ↓  deliveryMode（direct / gateway / hybrid）
IM API 发送

```

各渠道格式差异：


| Telegram | Discord | 大象(Daxiang) |
| --- | --- | --- |
| 文本格式 | HTML 子集 | Markdown | 自定义富文本 |
| 最大长度 | 4096 字 | 2000 字 | - |
| 收消息方式 | Webhook / Long Poll | Gateway Bot | 长连接推送 |
| 插件形式 | 内置 | 内置 | **外部 npm 包** |

**Channel Plugin 让 OpenClaw 核心层完全不感知底层 IM 差异，暴露统一接口。**

### 5.5 Plugin vs Tool


| Tool | Plugin |
| --- | --- |
| 类比 | 一把螺丝刀 | 一个工具箱 |
| 包含关系 | 不能包含 Plugin | **可包含多个 Tool** |
| 生命周期 | 无（纯函数） | 有（start/stop） |
| 配置 schema | 无 | 有（`openclaw.plugin.json`） |
| 分发方式 | 内联注册 | npm 包独立发布安装 |

### 5.6 扩展层次总结（从轻到重）

```Plain
// 代码块
HEARTBEAT.md          → 告诉 agent 醒来后做什么（最轻量）
Internal Hooks         → 监听系统事件，触发自定义逻辑
Plugin Hooks（api.on） → 深度拦截 LLM 流程（改 prompt/拦截 tool）
Channel Plugin         → 接入全新 IM 渠道
Provider Plugin        → 接入全新 AI 模型提供商

```


---

## 六、总结

今日学习完成了 OpenClaw 中层架构的全部核心模块。至此，整体架构图已较完整：

```Plain
// 代码块
用户 IM（大象/Telegram/Discord）
    ↓  Channel Plugin（格式转换 + 协议适配）
Gateway（HTTP + WebSocket，~100 RPC 方法）
    ↓
Auto-Reply（防抖 → 去重 → 分发 → agent 执行）
    ↓
pi-agent-core（Agent Loop：prompt → LLM → 工具调用 → 循环）
    ↓
pi-ai（统一 LLM 抽象层，15+ Provider）

横切关注点：
├── Memory（SQLite + 向量检索 + FTS5 混合搜索）
├── Session & Routing（SessionKey + Binding + dmScope）
├── Hooks（Internal + Plugin，事件驱动扩展）
├── Cron（定时任务，main/isolated 两种模式）
├── Heartbeat（定期唤醒 agent，HEARTBEAT.md 作为任务清单）
├── ACP（结构化协议，连接外部 coding agent）
└── Plugins（通用扩展插槽：Tool/Service/Provider/Channel）

```

**下一步**：pi-coding-agent 高层 SDK 学习。

Done

