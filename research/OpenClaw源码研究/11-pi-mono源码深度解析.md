Fetching document 2752538360......
# 【11】pi-mono 源码深度解析：OpenClaw 的 Agent 核心底座


> 调研日期：2026-03-23 / 2026-03-24<br>
> 作者：huanglin12
>
>


---

## 一、pi-mono 是什么？

pi-mono 是 OpenClaw 的核心底层框架，仓库地址：`github.com/badlogic/pi-mono`。

**OpenClaw 的核心 Agent 能力全部建在 pi-mono 之上**，采用嵌入式集成方式（非子进程调用）。pi-mono 由 4 个核心包组成，形成清晰的三层架构：

```Plain Text
// 代码块
pi-coding-agent   ← 顶层：Session 管理、UI、RPC、Skills、Auto-Compaction
      ↑
pi-agent-core     ← 中层：有状态 Agent 运行时（Loop + 工具执行 + 事件流）
      ↑
pi-ai             ← 底层：统一 LLM API 抽象层
      ↑
pi-tui            ← TUI 工具库（终端 UI，独立）

```


---

## 二、pi-ai：统一 LLM 抽象层

### 核心定位

pi-ai 是整个底座的最底层，职责是**封装所有 LLM Provider 的差异**，对上层提供统一的 API。

### 支持的 Provider（15+）


| 类别 | Provider |
| --- | --- |
| 国际主流 | OpenAI、Anthropic（Claude）、Google Gemini、AWS Bedrock、Mistral |
| 国内 | 可通过 OpenAI 兼容接口接入 |
| 订阅复用 | Claude Code 订阅、GitHub Copilot、Gemini CLI、OpenAI Codex |

**关键依赖**：各家官方 SDK（`@anthropic-ai/sdk`、`openai`、`@google/genai`、`@aws-sdk/client-bedrock-runtime`、`@mistralai/mistralai`）

### 核心特性


- **统一 Streaming 事件流**：无论哪家 Provider，上层代码拿到的都是标准事件
- **Thinking/Reasoning 标准化**：统一处理各家的思维链输出
- **多模态 Content Block**：图片、文件、工具调用统一格式
- **跨 Provider 上下文序列化**：可以在不同模型间切换而不丢失上下文

### 工程细节


- `partial-json` 库：用于解析 Streaming 中**未完成的工具调用 JSON**（流式输出时 JSON 可能截断）
- **OAuth 实现**：纯原生（`node:http` + Web Crypto API），不依赖第三方 OAuth 库


---

## 三、OAuth 机制详解：复用 Claude Code 订阅

这是 pi-mono 最有意思的技术细节之一。

### 核心原理


| 字段 | 值 |
| --- | --- |
| 协议 | OAuth 2.0 + PKCE（S256，Web Crypto API） |
| Client ID | `9d1c250a-e61b-44d9-88ed-5944d1962f5e`（Claude Code 官方 client_id，硬编码） |
| 授权地址 | `https://claude.ai/oauth/authorize` |
| 本地回调端口 | 53692（pi-mono）/ 54545（CLIProxyAPI，Go 实现） |

**关键洞察**：pi-mono 通过硬编码 Claude Code 的官方 client_id，**冒充官方 CLI** 完成 OAuth 授权，从而复用用户已有的 Claude 订阅账号，无需额外付费。

### 支持的订阅来源


- Anthropic Claude（Claude Code 订阅）
- Google Gemini CLI
- OpenAI Codex
- GitHub Copilot


> 注：Cursor 订阅**不支持**（Cursor 走私有协议，不开放标准 OAuth）
>
>

### CLIProxyAPI 项目（相关生态）

仓库：`github.com/router-for-me/CLIProxyAPI`

**作用**：把各家 CLI 工具的订阅账号包装成标准 OpenAI 兼容 API，支持多账号 round-robin 负载均衡。

特别之处：使用 `utls` 库**伪装 Firefox TLS 指纹**，绕过 Cloudflare 的机器人检测。

用途：可作为 OpenClaw 的 API Provider，直接把 Claude Code 订阅用于 OpenClaw。


---

## 四、pi-agent-core：有状态 Agent 运行时

### 核心定位

pi-agent-core 是中间层，负责**维护 Agent 的完整生命周期**，包括对话循环、工具执行、事件流。

### Agent Loop 详解

```Plain Text
// 代码块
prompt
   ↓
transformContext（上下文裁剪/压缩）
   ↓
convertToLlm（AgentMessage[] → LLM Message[]，过滤 UI-only 消息）
   ↓
调用 LLM
   ↓
解析工具调用
   ↓
执行工具（parallel 或 sequential）
   ↓
回到循环，直到 LLM 不再调用工具

```

### 核心 Hook 点（OpenClaw 扩展的关键）


| Hook | 用途 | OpenClaw 在这里做什么 |
| --- | --- | --- |
| `beforeToolCall` | 工具执行前拦截 | 沙箱路径检查、权限控制 |
| `afterToolCall` | 工具结果后处理 | 结果过滤、审计日志 |
| `transformContext` | 上下文裁剪/压缩 | **Compaction 压缩** |
| `convertToLlm` | 消息格式转换 | 过滤 UI-only 消息 |

### 工具执行模式


- **parallel**（默认）：多工具并行执行，提升效率
- **sequential**：顺序执行，用于有依赖关系的工具

### 高级能力


| 能力 | 说明 |
| --- | --- |
| **Steering** | 运行中打断 Agent，注入新指令 |
| **FollowUp** | 排队任务，Agent 完成当前 turn 后自动继续 |

### 事件系统（完整生命周期）

```Plain Text
// 代码块
agent_start / agent_end
turn_start / turn_end
message_start / message_update / message_end
tool_execution_start / tool_execution_update / tool_execution_end

```


---

## 五、pi-coding-agent：顶层高级 SDK

### 核心定位

pi-coding-agent 是最顶层的封装，在 pi-agent-core 之上提供：


- Session 管理（会话树、分叉、导航）
- 多种运行模式（交互/打印/RPC）
- Auto-Compaction（自动上下文压缩）
- Auto-Retry（自动重试）
- Skills 系统

### AgentSession 核心方法


| 方法 | 作用 |
| --- | --- |
| `prompt()` | 发送 prompt，触发新一轮对话 |
| `steer()` | 中途打断（当前工具执行完后插入） |
| `followUp()` | 排队（当前 turn 全部完成后执行） |
| `abort()` | 立刻中止当前执行 |
| `compact()` | 手动触发上下文压缩 |
| `fork()` | 从某条消息分叉，创建新 session 文件 |
| `navigateTree()` | 在 session 树中导航（不新建文件） |

### Session 文件格式（JSONL 树结构）

Session 文件是标准 JSONL 格式，每条 Entry 包含 `parentId` 字段，形成树状结构：

```Plain Text
// 代码块
Entry 类型：
- message        → 普通对话消息
- compaction     → 压缩记录
- branch_summary → 分支摘要
- model_change   → 模型切换记录
- custom         → 自定义事件
- label          → 标签（书签）

```

这种树形结构支持 `fork`（创建分支）和 `navigateTree`（在分支间切换）。

### 三种运行模式


| 模式 | 用途 | 特点 |
| --- | --- | --- |
| **interactive** | 开发调试 | 完整 TUI（Claude Code 风格终端界面） |
| **print** | CI/CD | 纯文本输出，无交互 |
| **rpc** | 程序集成 | stdin/stdout JSON-Lines RPC → **这是 acpx 的接入点** |

### RPC 模式与 acpx 集成链路

```Plain Text
// 代码块
大象消息
   ↓
OpenClaw Auto-Reply
   ↓
sessions_spawn(acp)
   ↓
ACP control-plane
   ↓
acpx（ACP 执行器）
   ↓
pi-coding-agent（rpc 模式，stdin/stdout）
   ↓
文件系统

```

**关键**：acpx 通过 stdin 发送 `RpcCommand` JSON 控制 pi-coding-agent，从 stdout 读取 `AgentSessionEvent` 事件流，再翻译成 ACP 标准事件推给 OpenClaw。

### Auto-Compaction & Auto-Retry 机制


| 触发条件 | 处理方式 |
| --- | --- |
| 超过 token 阈值 | 自动触发 compact（压缩上下文） |
| context overflow | 自动触发 compact（不 retry） |
| LLM overload / rate_limit / 5xx | 指数退避重试 |


---

## 六、整体架构总结

```Plain Text
// 代码块
┌─────────────────────────────────────────┐
│              OpenClaw                   │
│  (Gateway / Channels / Hooks / Cron)    │
├─────────────────────────────────────────┤
│           pi-coding-agent               │
│  Session管理 | RPC模式 | Auto-Compaction │
├─────────────────────────────────────────┤
│            pi-agent-core                │
│  Agent Loop | 工具执行 | 事件流 | Hooks  │
├─────────────────────────────────────────┤
│               pi-ai                     │
│  统一LLM抽象 | 15+ Provider | OAuth     │
└─────────────────────────────────────────┘

```

**OpenClaw 不是把 pi-mono 当子进程调用，而是把它作为库直接嵌入进程**，在关键 Hook 点（`beforeToolCall`、`transformContext`、`convertToLlm`）插入自己的逻辑，实现：


- 沙箱安全检查
- 上下文压缩（Compaction）
- 消息格式转换
- 权限控制

这种嵌入式架构比子进程调用有更低的延迟和更强的集成能力。


---

## 七、MCP Server 三级分层（工程最佳实践）

pi-coding-agent 的 Skills 系统对 MCP Server 的管理采用三级分层，这也是 OpenClaw Skills 的设计基础：


| 级别 | 策略 | 适用场景 | OpenClaw 对应 |
| --- | --- | --- | --- |
| Level 1 | 常驻注入 | 高频轻量工具，每次启动就注入 | 核心内置工具 |
| Level 2 | 按需加载 | 描述符常驻，触发才注入完整定义 | **Skills（主要模式）** |
| Level 3 | 动态发现 | 低频高成本，用完即卸载 | 临时工具 |

**为什么需要分级？**

5 个 MCP Server 约有 55,000 tokens 的工具定义开销。如果每次都全量加载，不仅浪费 token，还会稀释模型对相关工具的注意力。Level 2 的"描述符常驻+按需激活"是最优平衡点。


> 这也是 OpenClaw Skills 的工作原理：skill 描述符始终在系统提示中，但 (http://SKILL.md "SKILL.md") 内容只在命中时才加载。
>
>


---

## 八、关键洞察


1. **OAuth 复用**是 pi-mono 最核心的工程创新——通过冒充官方 CLI，让所有 Claude 订阅用户都能直接用 OpenClaw，无需额外付费
2. **Hook 系统**是 OpenClaw 实现可扩展性的关键——`beforeToolCall`、`transformContext` 等钩子让 OpenClaw 能在不修改 pi-mono 的情况下插入自己的逻辑
3. **RPC 模式**是 ACP 集成的核心——acpx 本质上是 pi-coding-agent 的 RPC 客户端
4. **Session JSONL 树**是上下文管理的基础数据结构——压缩、分叉、导航都基于这棵树
5. **三级分层**解决了工具数量增多时的 token 膨胀问题——OpenClaw Skills 直接复用了这套机制

Done

