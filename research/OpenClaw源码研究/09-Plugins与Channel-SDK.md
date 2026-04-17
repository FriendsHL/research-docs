Fetching document 2752833374......
# OpenClaw 源码精读：Plugins 与 Channel SDK


> 源码路径：`/app/src/plugins/` 和 `/app/src/channels/plugins/`
>
>


---

## 一、两套扩展体系的关系

```Plain
// 代码块
OpenClaw 扩展体系
├── Plugin（通用插件）          /app/src/plugins/
│   能注册：Tool / Hook / HTTP路由 / GatewayMethod / CLI / Service / Provider / Command
│   + 深度拦截 LLM 流程的 Plugin Hooks
│
└── Channel Plugin（渠道插件）  /app/src/channels/plugins/
    是 Plugin 的特化子集，专门用于接入新的 IM 渠道
    Daxiang 就是一个 Channel Plugin（外部 npm 包形式）

```


---

## 二、Plugin：通用插件系统

### 2.1 Plugin 的两种定义形式

**函数式（简洁）**：

```TypeScript
// 代码块
// 直接导出函数
export default function(api: OpenClawPluginApi) {
  api.registerTool(myTool);
  api.on("before_tool_call", handler);
}

```

**对象式（功能更丰富）**：

```TypeScript
// 代码块
export default {
  id: "my-plugin",
  name: "My Plugin",
  version: "1.0.0",
  configSchema: zodSchema,  // 配置验证 schema
  register(api) {
    api.registerTool(tool1);
    api.registerService(backgroundService);
  },
  activate(api) {
    // 配置加载完成后调用（比 register 更晚）
    // 适合需要读配置才能初始化的逻辑
  },
} satisfies OpenClawPluginDefinition;

```

`register` vs `activate` 的区别：`register` 在插件加载时立即调用（早），`activate` 在配置完全加载后调用（晚）。需要用户配置值的逻辑放 `activate`，纯注册操作放 `register`。

### 2.2 OpenClawPluginApi：完整注册接口

```TypeScript
// 代码块
type OpenClawPluginApi = {
  // 基础信息
  id: string;
  name: string;
  version?: string;
  config: OpenClawConfig;           // 当前 openclaw 配置
  pluginConfig?: Record<string, unknown>;  // 该插件自己的配置
  runtime: PluginRuntime;           // 运行时环境（sandbox/host/…）
  logger: PluginLogger;             // 结构化日志
  
  // 注册能力
  registerTool(tool, opts?)              // 注册 LLM 可调用的工具
  registerHook(events, handler, opts?)   // 注册 Internal Hook 处理器
  registerHttpHandler(handler)           // 注册 HTTP 拦截器（所有请求）
  registerHttpRoute({ path, handler })   // 注册特定路径的 HTTP 接口
  registerChannel(plugin)                // 接入新 IM 渠道
  registerGatewayMethod(method, handler) // 注册 Gateway JSON-RPC 方法
  registerCli(registrar, opts?)          // 扩展 `openclaw` CLI 命令
  registerService(service)               // 注册后台常驻服务
  registerProvider(provider)             // 接入新 AI 模型提供商
  registerCommand(command)               // 注册斜杠命令（绕过 LLM 直接处理）
  
  on(hookName, handler, { priority? })   // 注册 Plugin Hook（拦截 LLM 流程）
  resolvePath(input)                     // 解析相对路径（以 workspaceDir 为基准）
};

```

### 2.3 各注册方法详解

`**registerTool**`：

```TypeScript
// 代码块
// 静态 tool
api.registerTool({
  name: "query_internal_api",
  description: "查询内部 API",
  inputSchema: z.object({ endpoint: z.string() }),
  execute: async ({ endpoint }) => { ... }
});

// 动态 tool factory（根据配置/上下文决定注册哪些 tool）
api.registerTool((ctx: OpenClawPluginToolContext) => {
  if (!ctx.config.features?.myFeature) return null;
  return myTool;
}, { optional: true });

```

`OpenClawPluginToolContext` 包含：agentId、sessionKey、workspaceDir、messageChannel、sandboxed 等——让 tool 知道自己在哪个 session 里被调用。

`**registerService**`：

```TypeScript
// 代码块
api.registerService({
  id: "my-background-worker",
  async start(ctx) {
    // ctx.stateDir → 持久化状态存储目录
    // ctx.config   → openclaw 配置
    this.timer = setInterval(() => checkSomething(), 60000);
  },
  async stop(ctx) {
    clearInterval(this.timer);
  }
});
// 随 Gateway 启动而 start()，随 Gateway 停止而 stop()

```

`**registerCommand**`（重要：绕过 LLM）：

```TypeScript
// 代码块
api.registerCommand({
  name: "status",           // 用户发 /status 触发
  description: "查看当前状态",
  acceptsArgs: true,
  requireAuth: true,        // 默认 true，只有白名单用户可用
  handler: async (ctx) => {
    // ctx.senderId, ctx.channel, ctx.args, ctx.isAuthorizedSender
    return "当前状态：正常运行中";
    // 返回字符串直接发给用户，不经过 LLM
  }
});

```

Plugin Commands 在**内置命令和 LLM 之前**处理，适合 `/tts`、`/ping`、`/reload` 这类简单状态操作。

`**registerProvider**`：

```TypeScript
// 代码块
api.registerProvider({
  id: "my-ollama",
  label: "My Ollama Instance",
  auth:  };
    }
  }]
});

```

### 2.4 Plugin 的四种来源

```TypeScript
// 代码块
type PluginOrigin = "bundled" | "global" | "workspace" | "config";

```


| origin | 位置 | 说明 |
| --- | --- | --- |
| `bundled` | OpenClaw 内置 | 随 OpenClaw 发布，不需要安装 |
| `global` | `~/.openclaw/plugins/` | `openclaw plugin install` 全局安装 |
| `workspace` | `<workspace>/plugins/` | 项目级插件 |
| `config` | `openclaw.config.json` 的 plugins 字段 | 配置文件声明的 npm 包 |


---

## 三、Channel Plugin：IM 渠道接入规范

### 3.1 ChannelPlugin 类型定义（完整）

```TypeScript
// 代码块
type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;          // 唯一标识，如 "telegram"、"daxiang"
  meta: ChannelMeta;      // 名称、描述、图标
  capabilities: ChannelCapabilities;  // 能力声明（见下文）
  defaults?: {
    queue?: { debounceMs?: number };  // 消息防抖配置
  };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };  // 热重载规则
  
  // 各种 Adapter（按需实现）
  config: ChannelConfigAdapter;      // 必须：配置读写
  onboarding?: ChannelOnboardingAdapter;    // 交互式引导安装
  setup?: ChannelSetupAdapter;              // 非交互式安装（CI 友好）
  pairing?: ChannelPairingAdapter;          // 扫码/二维码配对
  security?: ChannelSecurityAdapter;        // 安全检查（发送方白名单等）
  groups?: ChannelGroupAdapter;             // 群组操作
  mentions?: ChannelMentionAdapter;         // @提及处理
  outbound?: ChannelOutboundAdapter;        // 发消息（核心）
  status?: ChannelStatusAdapter;            // 连接状态检查
  gateway?: ChannelGatewayAdapter;          // 收消息（长连接/webhook）
  auth?: ChannelAuthAdapter;                // 认证流程
  elevated?: ChannelElevatedAdapter;        // 特权操作（踢人、Ban 等）
  commands?: ChannelCommandAdapter;         // 渠道自定义命令
  streaming?: ChannelStreamingAdapter;      // 流式输出支持
  threading?: ChannelThreadingAdapter;      // 线程/话题支持
  messaging?: ChannelMessagingAdapter;      // 高级消息操作（引用/置顶/删除）
  agentPrompt?: ChannelAgentPromptAdapter;  // 自定义 agent prompt 前缀
  directory?: ChannelDirectoryAdapter;      // 用户/群组目录查询
  resolver?: ChannelResolverAdapter;        // ID/名称解析
  actions?: ChannelMessageActionAdapter;    // 消息动作（点赞/投票等）
  heartbeat?: ChannelHeartbeatAdapter;      // 渠道特定的 heartbeat 逻辑
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];  // 渠道专属 agent 工具
};

```

### 3.2 ChannelCapabilities：能力声明

Channel Plugin 通过 `capabilities` 声明它支持哪些功能，OpenClaw 核心层根据此决定启用哪些特性：

```TypeScript
// 代码块
type ChannelCapabilities = {
  inlineKeyboard?: boolean;    // 支持内联按钮（Telegram 支持，大象需要配置）
  voiceMessages?: boolean;     // 支持语音消息
  fileAttachments?: boolean;   // 支持文件附件
  imageAttachments?: boolean;  // 支持图片附件
  videoAttachments?: boolean;  // 支持视频
  reactions?: boolean;         // 支持消息表情反应
  threads?: boolean;           // 支持话题/线程
  streaming?: boolean;         // 支持流式输出（逐字显示）
  // ... 更多能力
};

```

核心层不需要知道"大象支不支持内联按钮"——查 `capabilities.inlineKeyboard` 就行。

### 3.3 消息发送流程（outbound）

```Plain
// 代码块
OpenClaw 内部 Markdown 文本
    ↓
chunker（分块）：长消息按 maxLength 切割
    ↓
格式转换（每个 Channel 自己实现）：
    Telegram  → HTML 子集（<b>、<i>、<code>、<pre>）
    Discord   → Markdown（**bold**、`code`、```block```）
    大象       → 自定义富文本 JSON
    ↓
deliveryMode 路由：
    direct   → Channel Plugin 直接调 IM API
    gateway  → 通过 Gateway WebSocket 转发
    hybrid   → 优先 direct，失败回退 gateway
    ↓
IM API 发送

```

### 3.4 消息接收流程（gateway / inbound）

每个渠道收消息方式不同：


| 渠道 | 接收方式 | 实现位置 |
| --- | --- | --- |
| Telegram | Webhook 或 Long Poll | `gateway` adapter |
| Discord | Gateway Bot（WebSocket） | `gateway` adapter |
| 大象(Daxiang) | 长连接推送 | `gateway` adapter（外部 npm 包） |
| WhatsApp | Webhook | `gateway` adapter |

所有渠道收到消息后都转换成统一的 `IncomingMessage` 格式，进入 OpenClaw Auto-Reply 流程。

### 3.5 大象为什么是外部 npm 包

内置的 Telegram、Discord 随 OpenClaw 源码一起发布，而大象（Daxiang）是：


1. 美团内部私有协议，不能开源
2. 以外部 npm 包形式发布（`@mt/openclaw-daxiang`）
3. 通过 `openclaw plugin install @mt/openclaw-daxiang` 安装
4. 调用 `api.registerChannel(daxiangPlugin)` 注册

这证明了 Channel Plugin 机制的设计目标：**OpenClaw 核心完全不感知任何具体 IM 的存在**，所有 IM 都是平等的外部扩展。


---

## 四、Plugin vs Channel Plugin：边界在哪里


| Plugin | Channel Plugin |
| --- | --- |
| 用途 | 通用扩展（工具/服务/Provider/Hook…） | 专门接入 IM 渠道 |
| 注册方式 | `api.registerTool()` 等 | `api.registerChannel(plugin)` |
| 核心接口 | `OpenClawPluginApi` | `ChannelPlugin` 类型（Adapter 集合） |
| 是否有 Adapter | 无 | 有（outbound/gateway/security/…15+ Adapter） |
| 示例 | 内部搜索 Tool、Raptor 监控 Service | Telegram、Discord、大象 |

Channel Plugin 本质上是通过 `api.registerChannel()` 注册的 Plugin——它**是** Plugin 的一种，只是结构更复杂，因为 IM 协议本身就复杂。


---

## 五、Plugin 开发的完整生命周期

```Plain
// 代码块
1. 创建 npm 包
   package.json 中：
   {
     "main": "dist/index.js",
     "openclaw": { "plugin": true }
   }

2. 实现 Plugin 逻辑
   export default function(api) {
     api.registerTool(...)
     api.registerService(...)
     api.on("before_tool_call", ...)
   }

3. 安装
   openclaw plugin install ./my-plugin
   或
   openclaw plugin install my-plugin-npm-package

4. 在 openclaw.config.json 中配置（如果 Plugin 有配置项）

5. openclaw gateway restart  # 重启生效

```


---

## 六、扩展能力层次总结（从轻到重）

```Plain
// 代码块
HEARTBEAT.md（不是 Plugin，是工作区文件）
    → 告诉 agent 醒来后做什么，最零代码的方式

Internal Hooks（HOOK.md + handler.ts）
    → 监听系统事件，写少量 TypeScript

Plugin registerCommand
    → 注册斜杠命令，完全绕过 LLM

Plugin registerTool
    → 给 LLM 新增可调用工具

Plugin api.on("before_tool_call", ...)   ← Plugin Hooks
    → 拦截/修改 LLM 行为（改 prompt、阻断危险调用）

Plugin registerService
    → 后台常驻服务（不依赖 agent 触发）

Plugin registerProvider
    → 接入全新 AI 模型

Plugin registerChannel（= Channel Plugin）
    → 接入全新 IM 渠道（最重，需实现全套 Adapter）

```

Done

