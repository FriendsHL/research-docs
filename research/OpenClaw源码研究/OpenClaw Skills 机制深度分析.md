---
created: '2026-03-30T10:52:28.296841'
tags:
- openclaw
- skills
- agent
---

# OpenClaw Skills 机制深度分析

> 源码路径：`src/agents/skills/`
> 分析时间：2026-03-30

## 一、Skills 加载全流程

### 六大来源（优先级从低到高）

| 优先级 | 来源 | 路径 |
|--------|------|------|
| 1（最低） | extra | `config.skills.load.extraDirs` + 插件注册目录 |
| 2 | openclaw-bundled | 跟随 openclaw 安装包内置 |
| 3 | openclaw-managed | `~/.openclaw/skills/`（用户通过 `openclaw skills install` 安装） |
| 4 | agents-skills-personal | `~/.agents/skills/`（个人级，跨项目共享） |
| 5 | agents-skills-project | `<workspace>/.agents/skills/`（项目级） |
| 6（最高） | openclaw-workspace | `<workspace>/skills/`（工作区覆盖一切） |

同名 Skill 后者覆盖前者，使用 `Map<name, Skill>` 实现。

### 嵌套根自动检测

支持将整个 skills 仓库 clone 到工作区根目录，自动识别 `skills/` 子目录：

```
给定 dir
  → dir/skills/ 存在？
    → 子目录有 SKILL.md？→ 以 dir/skills/ 为根
    → 否 → 以 dir 为根
  → 否 → 以 dir 为根
```

### 安全门

- 每个 SKILL.md 加载前检查文件大小，超过 **256KB** 直接跳过
- 每个来源最多扫描 **300** 个子目录（`maxCandidatesPerRoot`）
- 每个来源最多加载 **200** 个 Skill（`maxSkillsLoadedPerSource`）

### 运行时资格过滤（shouldIncludeSkill）

文件：`src/agents/skills/config.ts`

过滤条件（任一不满足即排除）：
- `skillConfig.enabled !== false`（用户未主动禁用）
- bundled Skill 在 `allowBundled` 白名单内
- `evaluateRuntimeEligibility`：OS 匹配 + bins 存在 + env 有值 + config 路径为 truthy

> **always: true** 的 Skill 跳过所有 requires 检查，始终注入。适合无外部依赖的纯知识型 Skill。

### Frontmatter 元数据字段

```yaml
---
name: skill-name           # 唯一标识（去重 key）
description: >             # 触发描述（LLM 据此选择）
  Use when: ...
always: true               # 无条件注入，跳过 requires
requires:
  bins: [git, node]        # 必须全部存在的系统命令
  anyBins: [npm, pnpm]     # 至少一个存在
  env: [OPENAI_API_KEY]    # 必须有值的环境变量
  config: [browser.enabled]
primaryEnv: OPENAI_API_KEY # 主 API Key 的 env 变量名
os: [darwin, linux]        # 限定操作系统
user-invocable: true       # 是否暴露为 /slash 命令（默认 true）
disable-model-invocation: false  # true = 不注入 prompt（默认 false）
---
```

---

## 二、模型自主判断机制

### System Prompt 中的指令（原文）

```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.
```

### 三条关键规则

1. **每次回复前强制扫描**：不会因对话轮次增加而遗忘
2. **精确性优先**：多候选选"最具体的"，防止通用 Skill 遮蔽专用 Skill
3. **惰性加载约束**：只有确定选中后才读一个 SKILL.md，绝不预先批量读取

### 注入的 XML 格式

完整格式（skills 数量不多时）：
```xml
<available_skills>
  <skill>
    <name>obsidian-direct</name>
    <description>Work with Obsidian vaults...</description>
    <location>~/.claude/skills/obsidian-direct/SKILL.md</location>
  </skill>
</available_skills>
```

compact 格式（skills 过多，描述被省略）：
```xml
<available_skills>
  <skill>
    <name>obsidian-direct</name>
    <location>~/.claude/skills/obsidian-direct/SKILL.md</location>
  </skill>
</available_skills>
```

> compact 模式下模型只能根据 name 猜测，触发精度会显著下降。

### Prompt 预算三级降级

```
全量 Skills（≤150 个）
  → formatSkillsForPrompt ≤ 30,000 字符？→ 完整格式
  → 否 → compact 格式（仅 name + location）≤ 29,850？→ compact 输出
  → 否 → 二分搜索最大可容纳数量 + ⚠️ 截断警告
```

### 路径 Token 优化

将 `/Users/alice/...` 替换为 `~/...`，每条路径节省约 5-6 token。150 个 Skills 时约节省 750-900 token。

---

## 三、渐进式披露（Progressive Disclosure）

### 三级信息密度

| 阶段 | 信息内容 | 何时发生 | Token 成本 |
|------|---------|---------|-----------|
| Level 1：System Prompt 注入 | name + description + location | 每次会话启动，无条件 | 约 50-200 token/Skill |
| Level 2：SKILL.md 读取 | 完整 Skill 指令 | 模型判断匹配，主动调用 read | 按需，1次/轮次 |
| Level 3：Bundled Resources | SKILL.md 引用的附属文件 | Skill 指令内要求读取时 | 完全按需 |

### 完整交互时序

```
用户："帮我在 Obsidian 里搜索关于微服务架构的笔记"
  ↓
模型扫描 <available_skills>，命中 obsidian-direct 的 description
  ↓
调用 read("~/.claude/skills/obsidian-direct/SKILL.md")  ← Level 2
  ↓
遵循 SKILL.md 指令，执行搜索脚本                          ← Level 3
```

---

## 四、脚本执行机制

### 安装阶段（openclaw skills install）

文件：`src/agents/skills-install.ts`

| kind | 执行命令 | 安全措施 |
|------|---------|---------|
| brew | `brew install <formula>` | - |
| node | `npm/pnpm/yarn/bun install -g --ignore-scripts <pkg>` | **--ignore-scripts** 防止 postinstall 执行 |
| go | `go install <module>` | - |
| uv | `uv tool install <package>` | 自动先安装 uv |
| download | HTTP 下载 + 解压到 `~/.openclaw/tools/<hashed-skill-key>/` | 路径隔离 |

安装前执行安全扫描（`security/skill-scanner.ts`）：
- **critical**：包含明确危险代码模式 → 输出 WARNING（不阻止安装）
- **warn**：有可疑模式 → 建议 `openclaw security audit --deep`

### 运行时 Env 变量注入（applySkillEnvOverrides）

文件：`src/agents/skills/env-overrides.ts`

**工作原理**：在 Agent 启动时将用户配置的 API key 注入 `process.env`，会话结束后自动还原。

**引用计数管理**：用 `activeSkillEnvEntries: Map<key, {baseline, value, count}>` 处理多 Skill 竞争同一 env key 的冲突：

```typescript
// 注入时：引用计数 +1
acquireActiveSkillEnvKey(key, value)

// 会话结束时：引用计数 -1，归零后还原原始值
releaseActiveSkillEnvKey(key)
```

**安全规则**：
- 系统已有值的 env key 不覆盖（尊重用户手动配置）
- `OPENSSL_CONF`、`LD_PRELOAD` 等危险 env key 永久阻止
- ACP spawn 子进程前，将 Skill 注入的 key 从子进程环境中剥离，防止 API key 泄漏

---

## 五、热更新与快照版本管理

文件：`src/agents/skills/refresh.ts`

### Chokidar 监听策略

**只监听 SKILL.md**，不监听整个目录树，防止 FD（文件描述符）耗尽：

```
监听目标：
  <workspace>/skills/*/SKILL.md
  <workspace>/.agents/skills/*/SKILL.md
  ~/.openclaw/skills/*/SKILL.md
  ~/.agents/skills/*/SKILL.md
```

### 版本号机制

```typescript
// 保证单调递增，即使时钟不严格准确
function bumpVersion(current: number): number {
  const now = Date.now();
  return now <= current ? current + 1 : now;
}
```

### SkillSnapshot

```typescript
type SkillSnapshot = {
  prompt: string;          // 已构建好的 prompt 字符串（直接注入，避免重复 IO）
  skills: Array<{
    name: string;
    primaryEnv?: string;   // 供 env-overrides 使用
    requiredEnv?: string[];
  }>;
  resolvedSkills?: Skill[]; // 完整 Skill 对象（含绝对路径）
  version?: number;         // 快照版本号（用于失效检测）
};
```

**短路策略**：snapshot.prompt 非空时直接返回，不重新扫描文件系统。只有版本号变化后才触发完整扫描。

---

## 相关笔记

- [[OpenClaw Agent 架构总览]]