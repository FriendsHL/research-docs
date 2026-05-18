---
type: concept
name: agent-interop
status: draft
sources:
  - "[[../source/codex]]"
  - "/Users/youren/myspace/codex/codex-rs/external-agent-sessions/src/detect.rs"
  - "/Users/youren/myspace/codex/codex-rs/external-agent-sessions/src/ledger.rs"
  - "/Users/youren/myspace/codex/codex-rs/external-agent-sessions/src/records.rs"
related_harnesses: [codex]
---

# Agent Interop — 跨 harness session 互通

> **4 家独家提示**:本概念目前**只在 Codex 一家**显式实现(`external-agent-sessions/` + `external-agent-migration/` 两个独立 crate)。其他 3 家(Hermes/Claude Code/OpenClaw)都没有跨 harness 迁移机制。

## 定义

**Agent Interop** = 允许用户在不同 agent harness 之间迁移会话状态。例如用户在 Claude Code 跑了 5 个 turn 然后转 Codex 继续,Codex 把 Claude Code 的 `.jsonl` rollout 解析成 Codex thread 并继续。

跟 MCP(model context protocol)是不同问题:
- **MCP**:统一 tool 接口让多家 harness 调用同一个 tool server
- **Agent Interop**:统一 session 状态让 session 在多家 harness 间迁移

**关于 agent 互通协议栈**:本页关注 _"跨 harness session 迁移"_ 这一个垂直问题;协议栈整体(MCP / A2A / AGNTCY / AG-UI + OTel telemetry 贯穿)详见 [[agent-protocols]] 5 层栈 anchor。

## 设计空间(5 维)

### 1. 迁移方向
- **单向导入**(Codex 当前):**他家 → Codex**。`detect.rs:19` `detect_recent_sessions(external_agent_home, codex_home)` 扫描其他 agent 的 home 目录(Claude Code 的 `~/.claude/`、Cursor 的 `~/.cursor/` 等)
- **双向**:Codex → 他家也支持(未在源码确认,理论上 `export.rs` 是为此存在)
- **三方桥**:第三方服务作为转换器(未见)

### 2. Session 形态
- **JSONL rollout**(Codex 自己用)— `rollout/src/recorder.rs` 持久化 + SQLite 索引
- **导入的他家 session 也是 JSONL**(`detect.rs:47-48` `if path.extension() ... != Some("jsonl")` continue)— 假设他家也用 JSONL,跟 Claude Code rollout 兼容

### 3. 去重
- **SHA256 content hash**(`ledger.rs:20-26`):`ImportedExternalAgentSessionRecord { source_path, content_sha256, imported_thread_id, imported_at }`
- 同一文件再次导入会跳过(`load_import_ledger(codex_home).contains_current_source(source_path)`)
- ledger 文件:`external_agent_session_imports.json`(`ledger.rs:12`)

### 4. 数量 / 时间窗口限制
- `SESSION_IMPORT_MAX_COUNT: usize = 50`(`detect.rs:10`)— 单次最多导入 50 个 session
- `SESSION_IMPORT_MAX_AGE: Duration = Duration::from_secs(30 * 24 * 60 * 60)`(`detect.rs:11`)— 30 天内的 session 才考虑
- **避免一次性导入历史全量造成 Codex 数据冗余**

### 5. 字段截断
- `NOTE_MAX_LEN: 2_000`(`records.rs:14`)
- `TOOL_RESULT_MAX_LEN: 4_000`(`records.rs:15`)
- 标签:`EXTERNAL_AGENT_TOOL_CALL_TAG = "external_agent_tool_call"` + `EXTERNAL_AGENT_TOOL_RESULT_TAG = "external_agent_tool_result"`(`records.rs:16-17`)— **导入的 tool call/result 被打标签,避免 Codex 把他家 tool 当成自己的可调用 tool**

## 各 harness 当前状态

| Harness | 跨 harness 迁移 | 备注 |
|---|---|---|
| **Codex** | ✅ 显式(2 个 crate) | 唯一一家 |
| Claude Code | ❌ | 有 `--resume <id>` 但只复用自家 session |
| Hermes | ❌ | 有 session id 复用但限 Hermes 内 |
| OpenClaw | ❌ | Memory Wiki 跨 session 但限 OpenClaw 内 |

## Trade-off

### Codex 这条路的优势
1. **真正的"agent 中立性"** — 用户可以试别家发现不合适,无缝转回 Codex 不丢上下文
2. **跨 agent 学习数据** — 看到他家 session 长什么样,反过来改进自己 prompt
3. **企业场景**:大公司可能同时部署多个 agent harness,session 互通解决治理难题

### 代价
1. **维护 N 个 importer**:每加一家支持就要 reverse engineer 他家 session 格式
2. **格式漂移**:他家版本升级 session schema 会破坏 importer
3. **法律 / 隐私**:用户授权范围模糊(导入意味着读他家 history)
4. **特性丢失**:他家独有的特性(Claude Code Skills / autoDream artifact / OpenClaw memory wiki)在迁移时无对应,只能丢弃或注释化

## SkillForge 当前选择 + 演进方向

**当前**:SkillForge 没有跨 harness 迁移机制,session 只在 SkillForge 内

**演进选项**:
1. **不做**(默认)— 等业内有共识 session schema 再做
2. **做 importer**(差异化候选)— 从 Codex / Claude Code / Cursor 导入 session 进 SkillForge:
   - 复用 SkillForge `ChannelAdapter` SPI 思路,做"`AgentAdapter` SPI"
   - 每家 harness 一个 adapter,转换为 SkillForge 内部 `Conversation` 实体
   - SHA256 去重 + 30 天窗口 + 字段截断 直接抄 Codex 阈值
3. **做 exporter**(对外)— 让 SkillForge session 导出成 Codex / Claude Code 兼容 JSONL,反向兼容

**判定**:**作为 SkillForge 差异化候选,优先级中等**。先把核心 13 维做扎实(Eval Pipeline / Self-Evolution / Multi-Agent 等),agent-interop 是 nice-to-have。但 Codex 阈值常量(50 / 30 天 / 2000/4000 字段)可以直接抄,实现成本不高。

## 学习视角:为什么只有 Codex 做这件事

OpenAI 的产品策略明确:
- Codex 是 ChatGPT 套餐内功能(README "Using Codex with your ChatGPT plan")
- Cloud + Local + IDE + Desktop App 全形态布局
- 目标用户已经付费 ChatGPT 订阅,**降低试用其他家的门槛** = 提高 retention

这是**商业策略驱动的技术决策**:
- Anthropic 卖 Claude Code 没有这种焦虑(用户已经在 Anthropic 生态)
- 开源 harness(Hermes/OpenClaw)没动力做(用户不是商业 retention 目标)
- Codex 有 retention 目标 → 做 importer

学习时要把"为什么这家做了那家没做"归结到**商业策略 + 模型生态位**,而不是单纯"工程能力"。
