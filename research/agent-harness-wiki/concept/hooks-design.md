---
type: concept
name: hooks-design
status: draft
sources:
  - "[[../source/codex]]"
  - "[[../source/claude-code]]"
  - "[[../source/hermes-agent]]"
  - "/Users/youren/myspace/codex/codex-rs/hooks/src/lib.rs"
  - "/Users/youren/myspace/hermes-agent/hermes_cli/plugins.py"
related_harnesses: [codex, claude-code, hermes-agent, openclaw]
---

# Hooks Design — 数量、粒度、精简程度

## 定义

**Hooks** = 在 agent lifecycle 关键时刻(tool 前/后、compact 前/后、session 开始/结束等)允许外部代码注入逻辑的机制。用于:权限检查、审计日志、prompt 改写、tool 拦截、自定义 telemetry。

## 设计空间(6 维)

### 1. 数量与粒度(2026-05-18 加 [[../harness/goose]] 后 4 档阶梯)

- **极简**(Codex):**8 lifecycle 事件**(`hooks/src/lib.rs:18-27`)
- **中等 + 4 细粒度 file/shell**([[../harness/goose]]):**11 hooks**(`crates/goose/src/hooks/mod.rs:49-77`)— ⭐ Open Plugins 标准对齐(`mod.rs:1-2` 顶注 _"modelled after the Open Plugins hooks specification"_, open-plugins.com/agent-builders/components/hooks)
- **中等**(Hermes):**16 hooks**(`VALID_HOOKS`,`hermes_cli/plugins.py:75-111`)
- **细粒度**(Claude Code):**27 hooks zod-validated**(`entrypoints/sdk/coreTypes.ts:25-53`)

```
Codex 8 (lifecycle)   <  goose 11 (Open Plugins)  <  Hermes 16 (Python)  <  Claude Code 27 (zod)
   精简                       中等 + 4 细粒度 file/shell             细粒度 framework
```

**goose 11 = 7 通用 + 4 细粒度** = 7 通用(PreToolUse / PostToolUse / **PostToolUseFailure** / SessionStart / SessionEnd / UserPromptSubmit / Stop)+ 4 细粒度 file/shell(**BeforeReadFile / AfterFileEdit / BeforeShellExecution / AfterShellExecution**)— 业内独家在 _读文件 / 改文件 / 执行 shell_ 这三个 _高风险动作_ 上前后包夹,适合 security inspector(`adversary_inspector` + `egress_inspector`)介入。

**Open Plugins 标准对齐意义**:goose hook 跟业界标准共建,比 Claude Code/Codex/Hermes 各自独立设计强 — _跨 framework plugin 复用_ 是新维度。SkillForge 若对齐,立刻获得生态收益。

差 ~3 倍(8 → 27)。原则:**越多的 hook 越细粒度,但用户学习成本越高,plugin 互相干扰风险越大**。

### 2. 是否区分 dispatch matcher
- **Codex**:8 个事件里只有 6 个 `HOOK_EVENT_NAMES_WITH_MATCHERS`(`hooks/src/lib.rs:34-41`),`Stop` 跟 `UserPromptSubmit` 不需要 matcher 因为没有 tool name / compaction trigger / session source 可匹配 — **显式区分,精确**
- **Claude Code**:所有 hook 都可带 matcher
- **Hermes**:不显式区分,plugin 自己 filter

### 3. 输出爆量防护
- **Codex**:`output_spill.rs` 独立模块 — 一等公民
- **Claude Code**:有 `MAX_HOOK_OUTPUT_TOKENS` 截断(具体值见 [[harness/claude-code]])
- **Hermes**:有 `tool_results.MAX_TOOL_RESULT_TOKENS` 类似机制(具体见 [[harness/hermes-agent]])

### 4. Plugin 注册模式
- **Codex `plugin_hook_declarations`**(`hooks/src/lib.rs:declarations`)— **白名单式**,plugin 显式声明它要监听哪些事件,框架预先知道 dispatch 目标
- **Claude Code SDK 注册**(`entrypoints/sdk/`)— SDK 隐式注册,运行时发现
- **Hermes plugin 装饰器** — Python 装饰器 + manifest

### 5. 向后兼容机制
- **Codex `legacy_notify.rs`** — 提供 `legacy_notify_json` + `notify_hook` 兼容旧 API
- **Claude Code** — 多版本 hook event names 共存(具体未深查)
- **Hermes** — 没显式兼容层

### 6. Hook 跟 Compact 的协作
- **Codex**:8 个事件里 `PreCompact` + `PostCompact` 是独立事件(`hooks/src/lib.rs:23-24`),`core/src/hook_runtime.rs` 暴露 `run_pre_compact_hooks` + `run_post_compact_hooks`
- **Claude Code**:`PreCompact` 是 27 个之一,跟 `PostCompact` 都存在
- **Hermes**:压缩没有专门 hook(压缩是内部流程,详见 [[concept/context-compaction]])

## 各 harness 完整对比

| 维度 | Codex (v0.0.0-dev) | Claude Code (v2.1.88) | Hermes | OpenClaw |
|---|---|---|---|---|
| 数量 | **8** | 27 | 16 | ? (没深查) |
| Event 列表 | PreToolUse / PermissionRequest / PostToolUse / PreCompact / PostCompact / SessionStart / UserPromptSubmit / Stop | (27 个,见 `coreTypes.ts:25-53`) | 16 valid hooks (`plugins.py:75-111`) | ? |
| Matcher 区分 | 6/8 显式有 | 全部可带 | plugin 自 filter | ? |
| 输出爆量保护 | `output_spill.rs` 独立模块 | `MAX_HOOK_OUTPUT_TOKENS` 截断 | `MAX_TOOL_RESULT_TOKENS` | ? |
| Plugin 注册 | 白名单 declarations | SDK 隐式 | Python 装饰器 | ? |
| 兼容机制 | `legacy_notify` 模块 | 多版本共存 | 无 | ? |
| 实现位置 | 独立 `hooks/` crate | `services/hooks/` | `hermes_cli/plugins.py` | ? |
| Compact hook | PreCompact + PostCompact | PreCompact + PostCompact | 无独立 hook | ? |
| Validation | Rust 静态 + serde | zod runtime | Python type hints | ? |

## Trade-off

### 8 vs 16 vs 27 的折中

**Codex 8** 哲学:lifecycle 关键时刻足够,过细的 hook 让 plugin 互相覆盖
- 优势:用户易理解,plugin 互不干扰,框架内部演进自由
- 代价:细粒度需求要在 plugin 内 if-else 分支自己处理(如 "PreToolUse" 想根据 tool name 不同行为)

**Claude Code 27** 哲学:每一个有意义的注入点都暴露
- 优势:plugin 可以精确针对 `PreSubagentStop` / `PreContextOverflow` / `PreAutoDream` 等细节
- 代价:学习曲线陡,plugin 间互相覆盖风险高(尤其 `PreToolUse` 多 plugin),zod 验证开销

**Hermes 16** 哲学:介于两者之间
- 优势:Python 易写,装饰器自然
- 代价:没有显式 matcher 系统,plugin 自己 filter event payload 容易出错

## SkillForge 当前选择 + 演进方向

**当前**:SkillForge ~6 个 lifecycle hooks(PreToolUse, PostToolUse, Stop, SafetyHook, AskUser + 待定)— 跟 Codex 8 个非常接近

**演进选项**:
1. **保持 6-8 个 lifecycle hook**(推荐)— 跟 Codex 路线一致
2. **加 `output_spill` 等价物**(必加)— 防 hook 输出爆量
3. **加 plugin 白名单声明机制**(Codex `plugin_hook_declarations`)— Java 注解 + Spring `ApplicationContextInitializer` 自然实现
4. **加 `PreCompact` / `PostCompact` 独立事件** — SkillForge Light/Full compaction 配合 hook 是清洁 design

判定:**8 lifecycle + output_spill + plugin 白名单声明是 SkillForge 应学的工程模板**。不要走 Claude Code 27 这条细粒度路线。

## 学习视角:为什么 Codex 选 8 个

OpenAI 的工程文化在 AGENTS.md 里清晰:
- "resist adding code to codex-core"(反 god crate)
- "Do not create small helper methods that are referenced only once"(反过度抽象)
- 模块上限 500 LoC,超 800 LoC 强制拆(`AGENTS.md:42-50`)

**8 个 hook 的精简是同一种"反 over-engineering"文化的体现**。学习时要把这当成**文化选择**,而不是单纯实现差异。

Claude Code 27 个也不是 over-engineering,而是 Anthropic 在 hooks 系统上吃过苦头后的对应:`autoDream` / `Coordinator` / `Fork SubAgent` / `Skills` / `Compaction 5 级` 等每一个新功能都对应 1-3 个 hook 事件,演化出 27。

**两条路都不算错,但 SkillForge 当前体量适合学 Codex 路线**。
