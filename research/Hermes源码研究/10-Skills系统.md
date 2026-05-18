---
created: '2026-04-30'
tags:
- hermes-agent
- skills
status: done
---

# Skills 系统源码解析

## 先说结论

Hermes 的 Skills 可以理解成“给 Agent 用的操作手册”。

它不是普通文档库，也不是插件代码。它主要解决一个问题：

> 当某类任务有固定流程、坑点、命令、模板时，不要每次都靠模型临场发挥，而是让模型先加载对应的 `SKILL.md`，按里面的步骤做。

所以一个 skill 通常长这样：

```text
skills/github/pr-review/SKILL.md
skills/github/pr-review/references/checklist.md
skills/github/pr-review/scripts/collect_context.py
```

`SKILL.md` 里面写“什么时候用这个 skill、应该按什么步骤做、有哪些坑、需要读哪些辅助文件、怎么验证结果”。

Hermes 的设计重点是：

1. system prompt 里只放 skill 目录，不放全文。
2. 真正需要时，再通过 `skill_view` 加载全文。
3. 用户也可以直接输入 `/skill-name`，强制把某个 skill 加载进本轮对话。
4. 安装、卸载、reload skills 默认尽量不破坏 prompt cache。
5. Agent 可以用 `skill_manage` 自己创建或更新 skill，把经验沉淀下来。

## 关键文件

- `skills/`：仓库内置 skills，启动或同步时会复制到用户目录。
- `optional-skills/`：官方可选 skills，不默认启用，需要通过 Skills Hub 安装。
- `tools/skills_sync.py`：把仓库内置 skills 同步到 `HERMES_HOME/skills`。
- `tools/skills_tool.py`：提供 `skills_list` 和 `skill_view` 两个 Agent 工具。
- `tools/skill_manager_tool.py`：提供 `skill_manage`，让 Agent 创建、编辑、删除 skill。
- `agent/prompt_builder.py`：把 skills 索引放进 system prompt。
- `agent/skill_commands.py`：扫描 `/skill-name` slash commands，并构造 skill 注入消息。
- `agent/skill_utils.py`：frontmatter、平台过滤、外部目录、禁用列表等通用逻辑。
- `agent/skill_preprocessing.py`：模板变量替换、inline shell 预处理。
- `hermes_cli/skills_hub.py`：`hermes skills ...` 和 `/skills ...` 命令入口。
- `tools/skills_hub.py`：Skills Hub 的来源适配、下载、隔离区、安全扫描、lock 文件。
- `hermes_cli/skills_config.py`：启用/禁用某个 skill，支持全局和按平台禁用。
- `tools/skill_usage.py`：记录 agent-created skills 的查看、使用、修改次数，给 curator 用。

## Skill 放在哪里

Hermes 真正使用的本地 skills 目录是：

```text
HERMES_HOME/skills
```

默认 profile 下通常就是：

```text
~/.hermes/skills
```

如果启用了 profile，就会变成对应 profile 的 home。

代码里不要直接写 `~/.hermes/skills`，而是通过 `get_hermes_home()` 算出来。这一点和 Hermes 的 profile 机制有关。

## 几类 Skill

### 1. Built-in skills

仓库里的 `skills/` 是内置 skills。

但 Agent 运行时不是直接读仓库目录，而是通过 `tools/skills_sync.py` 同步到：

```text
HERMES_HOME/skills
```

同步时会写一个 manifest：

```text
HERMES_HOME/skills/.bundled_manifest
```

它记录“哪些 skill 是从仓库内置 skills 复制过来的”。

同步逻辑比较谨慎：

- 新 skill：如果用户目录里没有，就复制过去。
- 用户改过的内置 skill：不会被覆盖。
- 仓库内置 skill 更新了：只有当用户本地版本没有改过时，才自动更新。
- 用户删除过的内置 skill：尊重删除，不会强行加回来。

### 2. Optional skills

`optional-skills/` 是官方可选 skills。

它们也是官方维护的，但默认不复制到用户目录，也不会出现在 system prompt 里。

安装方式类似：

```bash
hermes skills install official/research/some-skill
```

安装后，它才会落到：

```text
HERMES_HOME/skills/<category>/<skill>/
```

这时它对 Agent 来说就和普通本地 skill 一样了。

### 3. Hub skills

Hub skills 可以来自很多来源：

- 官方 optional skills
- Hermes 官方 index
- skills.sh
- well-known skills
- GitHub repo
- ClawHub
- Claude Marketplace
- LobeHub
- 直接给一个 URL

核心实现是 `tools/skills_hub.py` 里的 source adapter。

安装流程是：

```text
搜索/解析 identifier
  -> 下载 SkillBundle
  -> 写入 .hub/quarantine 隔离区
  -> security scan
  -> 用户确认或 slash command 隐式确认
  -> 移动到 HERMES_HOME/skills
  -> 写 .hub/lock.json
  -> 写 .hub/audit.log
```

安装后会记录来源：

```text
HERMES_HOME/skills/.hub/lock.json
```

所以后面可以做：

```bash
hermes skills check
hermes skills update
hermes skills audit
hermes skills uninstall <name>
```

### 4. User / agent-created skills

用户自己写的，或者 Agent 通过 `skill_manage` 创建的，也放在：

```text
HERMES_HOME/skills
```

这类 skill 没有 `.bundled_manifest` 和 `.hub/lock.json` 来源记录，所以 `tools/skill_usage.py` 会把它们视为 agent-created / local skill。

Curator 只会对这类 skill 做 stale、archive、restore，不会动内置或 Hub 安装的 skill。

### 5. External skills

配置里可以设置：

```yaml
skills:
  external_dirs:
    - /path/to/shared/skills
```

这些目录也会被扫描。

但外部 skills 更像“只读挂载”：

- 可以出现在 skill index。
- 可以用 `skill_view` 加载。
- 可以有 `/skill-name` 命令。
- 新建 skill 仍然写到本地 `HERMES_HOME/skills`。

如果本地 skill 和外部 skill 重名，本地优先。

## SKILL.md 基本格式

一个 skill 的核心文件是：

```text
SKILL.md
```

前面必须有 YAML frontmatter。典型结构：

```markdown
---
name: github-pr-review
description: Review GitHub pull requests with focus on bugs, regressions, and missing tests.
version: 1.0.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [github, review]
    config:
      - key: github.default_repo
        description: Default repo for PR review.
        default: ""
---

# GitHub PR Review

Use this skill when the user asks to review a pull request.

Steps:
1. Fetch PR metadata.
2. Read the diff.
3. Prioritize correctness issues.
4. Report findings first.
```

`tools/skills_tool.py` 会校验：

- `name` 必填。
- `description` 必填，长度不能超过 1024。
- `platforms` 可以限制 OS。
- `metadata.hermes.tags` 用于搜索和展示。
- `metadata.hermes.config` 声明这个 skill 需要哪些配置值。
- `required_environment_variables` 或旧版 `prerequisites.env_vars` 可以声明需要哪些环境变量。

## System Prompt 里放什么

Hermes 不会把所有 `SKILL.md` 全文塞进 system prompt。

它只会生成一个“目录”：

```text
## Skills (mandatory)
Before replying, scan the skills below...

<available_skills>
  github:
    - github-pr-review: Review GitHub pull requests...
  research:
    - paper-research: Research academic papers...
</available_skills>
```

生成逻辑在：

```text
agent/prompt_builder.py -> build_skills_system_prompt()
```

这个 prompt 的意思是：

> 你先看目录。如果任务和某个 skill 有关，必须用 `skill_view(name)` 加载全文，再继续做。

这样设计有两个好处：

1. 不浪费上下文。几十个 skills 只放名字和描述。
2. 保留 prompt cache。system prompt 比较稳定，不会因为每次任务不同都塞入一大段 skill 全文。

## Skills Prompt 缓存

`build_skills_system_prompt()` 有两层缓存：

第一层是进程内 LRU cache。

key 大概包括：

- skills 目录路径
- external dirs
- 当前可用 tools
- 当前可用 toolsets
- 当前平台
- disabled skills

第二层是磁盘 snapshot：

```text
HERMES_HOME/.skills_prompt_snapshot.json
```

snapshot 记录每个 `SKILL.md` 的 mtime、size 和解析后的 metadata。

如果本地 skills 没变，就可以直接读 snapshot，不必每次全目录扫描。

注意：外部 skills 目录不走 snapshot，一般直接扫描。

## Skill 是否展示

一个 skill 不一定会进入 system prompt。

过滤条件包括：

### 平台过滤

如果 frontmatter 写：

```yaml
platforms: [macos]
```

那 Linux / Windows 上不会展示。

### 用户禁用

配置里可以禁用：

```yaml
skills:
  disabled:
    - github-pr-review
```

也可以按平台禁用：

```yaml
skills:
  platform_disabled:
    telegram:
      - image-edit
```

CLI 入口是：

```bash
hermes skills config
```

实现文件：

```text
hermes_cli/skills_config.py
```

### 工具条件过滤

frontmatter 里可以声明条件。

例如：

```yaml
metadata:
  hermes:
    requires_tools: [github_fetch_pr]
```

意思是：只有 `github_fetch_pr` 可用时，这个 skill 才展示。

也可以写 fallback：

```yaml
metadata:
  hermes:
    fallback_for_toolsets: [github]
```

意思是：如果 github toolset 已经可用，就隐藏这个 skill；只有没有官方工具时，才把这个 skill 当替代方案展示出来。

这部分逻辑在：

```text
agent/skill_utils.py -> extract_skill_conditions()
agent/prompt_builder.py -> _skill_should_show()
```

## Agent 如何发现 Skill

Agent 有两个工具：

```text
skills_list
skill_view
```

它们在 `tools/skills_tool.py` 里注册到 `skills` toolset。

### skills_list

`skills_list` 只返回 metadata。

大概返回：

```json
{
  "success": true,
  "skills": [
    {
      "name": "github-pr-review",
      "description": "Review GitHub pull requests...",
      "category": "github"
    }
  ],
  "categories": ["github"],
  "count": 1
}
```

它不会返回全文。

适合模型不确定该用哪个 skill 时，先列一下。

### skill_view

`skill_view(name)` 会加载完整 `SKILL.md`。

例如：

```json
{
  "name": "github-pr-review",
  "preprocess": true
}
```

返回内容里会包含：

- `content`：处理后的 `SKILL.md` 内容。
- `raw_content`：原始内容。
- `path`：skill 文件路径。
- `skill_dir`：skill 目录。
- `linked_files`：references、templates、scripts、assets 等辅助文件。
- `env_requirements`：需要的环境变量。
- `missing_envs`：缺失的环境变量。
- `setup_needed` / `setup_note`：是否需要额外设置。

如果要读辅助文件，可以这样：

```json
{
  "name": "github-pr-review",
  "file_path": "references/checklist.md"
}
```

`file_path` 有路径穿越保护，不能读到 skill 目录外面。

## 一个普通加载例子

用户说：

```text
帮我 review 这个 PR
```

模型在 system prompt 里看到：

```text
github:
  - github-pr-review: Review GitHub pull requests...
```

于是第一轮可能调用：

```text
skill_view(name="github-pr-review")
```

工具返回完整 `SKILL.md`：

```text
Use this skill when reviewing a GitHub pull request.

Steps:
1. Fetch PR metadata.
2. Read changed files.
3. Focus on bugs and regressions.
4. Put findings first.
```

然后模型再按这个流程继续调用 GitHub 工具、读 diff、输出 review。

这里的关键点是：

> system prompt 只告诉模型“有这个 skill”，真正的操作流程来自后续 `skill_view` 的工具结果。

## Slash Skill 命令

除了让模型自己判断，也可以用户主动指定：

```text
/github-pr-review review PR #123
```

这条命令不会直接变成普通用户文本发给模型。

CLI 会先拦截它：

```text
cli.py
  -> build_skill_invocation_message()
  -> 把 skill 全文包装成一条 user message
  -> 放入 pending input
  -> 进入正常 agent loop
```

Gateway 也有同样逻辑：

```text
gateway/run.py
  -> resolve_skill_command_key()
  -> build_skill_invocation_message()
  -> event.text = skill message
  -> 继续正常消息处理
```

构造出来的 user message 大概是：

```text
[IMPORTANT: The user has invoked the "github-pr-review" skill, indicating they want you to follow its instructions. The full skill content is loaded below.]

<完整 SKILL.md 内容>

[Skill directory: /Users/.../.hermes/skills/github/github-pr-review]
Resolve any relative paths in this skill against that directory...

[This skill has supporting files:]
- references/checklist.md -> /Users/.../references/checklist.md

The user has provided the following instruction alongside the skill invocation: review PR #123
```

也就是说 `/skill-name` 的本质是：

> 用户强制加载某个 skill，并把“加载后的 skill 内容 + 用户附带指令”作为新的 user message 给 Agent。

它不是 system prompt 注入。

这样可以避免频繁修改 system prompt，从而保护 prompt cache。

## Skill 命令如何扫描

扫描逻辑在：

```text
agent/skill_commands.py -> scan_skill_commands()
```

它会扫描：

1. `HERMES_HOME/skills`
2. `skills.external_dirs`

跳过：

- `.git`
- `.github`
- `.hub`
- `.archive`
- 平台不兼容的 skill
- 被禁用的 skill

然后把 skill name 转成 slash command：

```text
name: "GitHub PR Review"
  -> /github-pr-review

name: "claude_code"
  -> /claude-code
```

Telegram 命令不喜欢 hyphen，所以 gateway 里还支持下划线回转：

```text
/claude_code -> /claude-code
```

## /reload-skills 做什么

`/reload-skills` 只重新扫描 slash skill commands。

它不会清空 skills system prompt cache。

原因是：

> skills 可以通过 `/skill-name`、`skills_list`、`skill_view` 运行，不一定非要先出现在 system prompt 里。

如果每次 reload 都清 system prompt，会破坏 prefix cache，成本变高。

所以 reload 后：

- slash command 立即可用。
- system prompt 里的 skill 目录可能要等下个 session 或显式 cache invalidation 后才更新。

Gateway 还会把 reload 结果作为下一轮的一次性 note 注入，避免直接乱写 session transcript。

## Skills Hub 安装流程

命令入口：

```bash
hermes skills search <query>
hermes skills inspect <identifier>
hermes skills install <identifier>
```

聊天里也可以：

```text
/skills search kubernetes
/skills install official/devops/kubernetes --now
```

核心安装逻辑在：

```text
hermes_cli/skills_hub.py -> do_install()
tools/skills_hub.py
```

更直白地说，安装分成 7 步：

1. 找到来源 adapter。
2. 下载成 `SkillBundle`。
3. 校验 skill name 和文件路径，禁止绝对路径和 `..`。
4. 写入隔离目录：

```text
HERMES_HOME/skills/.hub/quarantine/<skill-name>
```

5. 调用 `tools.skills_guard.scan_skill()` 安全扫描。
6. 通过后移动到：

```text
HERMES_HOME/skills/<category>/<skill-name>
```

7. 写入：

```text
HERMES_HOME/skills/.hub/lock.json
HERMES_HOME/skills/.hub/audit.log
```

安装时默认会清 skills prompt cache，让新 skill 马上出现在后续 system prompt。

但是聊天里的 `/skills install` 默认不清 cache，除非加：

```text
--now
```

原因还是同一个：避免随手安装一个 skill 就破坏当前 session 的 prompt cache。

## 安装例子

用户输入：

```text
/skills install official/research/literature-review
```

发生的事情大概是：

```text
handle_skills_slash()
  -> do_install(identifier="official/research/literature-review")
  -> OptionalSkillSource.fetch()
  -> quarantine_bundle()
  -> scan_skill()
  -> install_from_quarantine()
  -> 写入 HERMES_HOME/skills/research/literature-review/
  -> 写 .hub/lock.json
```

因为没有 `--now`，当前 system prompt 里的 skill 目录可能暂时还看不到它。

但用户可以马上用：

```text
/literature-review 帮我整理这篇论文
```

因为 slash command 可以通过 `/reload-skills` 重新扫描，不依赖 system prompt 目录。

## Agent 创建和修改 Skill

Agent 管理 skill 用的是：

```text
skill_manage
```

定义在：

```text
tools/skill_manager_tool.py
```

支持动作：

- `create`
- `edit`
- `patch`
- `delete`
- `write_file`
- `remove_file`

典型场景：

用户和 Agent 反复调试出一套稳定流程。

用户说：

```text
以后遇到这个项目的发布流程，就按刚刚的步骤来。
```

Agent 可以调用：

```json
{
  "action": "create",
  "name": "project-release",
  "category": "devops",
  "content": "---\nname: project-release\ndescription: Release this project using the verified local workflow.\n---\n\n# Project Release\n\nSteps:\n1. Run ...\n2. Check ...\n3. Publish ..."
}
```

如果后面发现 skill 里某一步错了，Agent 可以用 `patch`：

```json
{
  "action": "patch",
  "name": "project-release",
  "old_text": "Run npm test",
  "new_text": "Run scripts/run_tests.sh"
}
```

`skill_manage` 做了很多保护：

- skill name 只能是安全字符。
- `SKILL.md` 必须有合法 frontmatter。
- supporting file 只能写到 `references`、`templates`、`scripts`、`assets`。
- 禁止路径穿越。
- 写文件用 atomic replace。
- pinned skill 不允许修改。
- 如果开启了 `skills.guard_agent_created`，写完还会跑安全扫描，不通过就 rollback。

修改成功后会调用：

```text
clear_skills_system_prompt_cache(clear_snapshot=True)
```

因为新建或修改本地 skill 后，system prompt 里的 skill index 需要更新。

## Skill 预处理

`agent/skill_preprocessing.py` 处理两类东西。

第一类是模板变量：

```text
${HERMES_SKILL_DIR}
${HERMES_SESSION_ID}
```

如果 skill 内容写：

```text
Run ${HERMES_SKILL_DIR}/scripts/check.py
```

加载时会替换成真实路径。

第二类是 inline shell：

```text
!`python scripts/detect.py`
```

但 inline shell 默认关闭：

```yaml
skills:
  inline_shell: false
```

原因很简单：skill 内容可能来自第三方，自动执行里面的 shell 片段风险太高。

如果开启，会用 bash 执行，工作目录是 skill dir，并有 timeout 和输出长度限制。

## Skill 配置注入

skill 可以声明自己需要配置：

```yaml
metadata:
  hermes:
    config:
      - key: github.default_repo
        description: Default GitHub repo.
        default: ""
```

当用户通过 `/skill-name` 调用 skill 时，`agent/skill_commands.py` 会读取配置值，并注入到 user message：

```text
[Skill config (from ~/.hermes/config.yaml):
  github.default_repo = owner/repo
]
```

这样模型不用自己去读 config 文件，也知道这个 skill 当前配置是什么。

## 环境变量和凭证

skill 可以声明需要环境变量。

加载时 `skill_view` 会检查：

- 哪些环境变量需要但缺失。
- 是否可以走 secret capture callback。
- gateway 场景下是否只能给用户 setup hint。

如果环境变量存在，`skill_view` 还会注册 env passthrough。

目的：

> 如果后续工具在 sandbox 或远端环境里运行，也能拿到这个 skill 声明需要的环境变量。

如果 skill 声明 credential files，也会注册到 remote sandbox 的凭证文件传递逻辑里。

## Plugin Skill

`skill_view` 还支持 plugin 提供的 skill。

调用形式是：

```text
plugin_name:skill_name
```

例如：

```json
{
  "name": "some-plugin:deploy"
}
```

这时 `skill_view` 会：

1. `discover_plugins()`
2. `get_plugin_manager().find_plugin_skill(name)`
3. 检查插件是否禁用。
4. 检查平台兼容。
5. 返回插件内 skill 内容。

这类 skill 不一定在普通本地目录下，所以名字需要带 namespace，避免和本地 skill 冲突。

## 使用统计和 Curator

`tools/skill_usage.py` 记录 agent-created skills 的使用情况。

文件是：

```text
HERMES_HOME/skills/.usage.json
```

记录字段包括：

- `use_count`
- `view_count`
- `patch_count`
- `last_used_at`
- `last_viewed_at`
- `last_patched_at`
- `state`
- `pinned`

注意：它只记录 agent-created / local skills。

内置 skill 和 Hub 安装的 skill 不记入这个 sidecar，因为它们由上游维护，不应该被 curator 自动归档。

## 一个完整例子

假设用户输入：

```text
/paper-research 帮我调研这篇论文
```

流程大概是：

```text
1. CLI 看到这是 slash command。
2. 在 _skill_commands 里找到 /paper-research。
3. 调用 skill_view(skill_dir, preprocess=False) 读 SKILL.md。
4. 构造 user message：
   - 说明用户显式调用了 paper-research skill
   - 放入完整 SKILL.md
   - 放入 skill_dir
   - 列出 references/templates/scripts/assets
   - 附上用户原始指令：“帮我调研这篇论文”
5. 这条 user message 进入正常 agent loop。
6. 模型看到 skill 全文后，按里面步骤行动。
7. 如果需要更多资料，再调用 skill_view(file_path="references/xxx.md")。
8. 如果执行中发现 skill 写错了，最后可以用 skill_manage(action="patch") 修正。
```

这个过程里，system prompt 没变。

所以它对 prompt cache 比较友好。

## 再举一个 Agent 自己选择 Skill 的例子

用户没有打 slash command，只说：

```text
帮我给这个 Terraform 模块做一次安全审计。
```

system prompt 里有：

```text
security:
  - terraform-security-review: Review Terraform modules for risky IAM, network, and secret settings.
```

模型应该先调用：

```text
skill_view(name="terraform-security-review")
```

然后 skill 里可能写：

```text
Steps:
1. Find all *.tf files.
2. Check public ingress rules.
3. Check wildcard IAM permissions.
4. Check hardcoded secrets.
5. Summarize risk by severity.
```

模型就照这个流程继续读文件、调用工具、输出结果。

这就是 system prompt skill index 的作用：提醒模型“这里有专门流程，不要凭空发挥”。

## 和插件的区别

Skill 和 plugin 很容易混淆，但它们不是一类东西。

Skill 是“说明书”：

- 主要是 Markdown。
- 告诉模型怎么做。
- 可以带模板、脚本、参考文件。
- 加载进上下文后由模型执行。

Plugin 是“扩展系统能力”：

- 可以注册工具。
- 可以注册 hook。
- 可以注册 CLI 命令。
- 运行的是 Python 代码。

一句话：

> Skill 改变模型的工作方法，Plugin 改变 Hermes 的能力边界。

## 设计取舍

Hermes 的 Skills 系统整体是“索引轻、加载重、变更谨慎”。

轻索引：

- system prompt 只放 name + description。

按需加载：

- 用 `skill_view` 或 `/skill-name` 再读全文。

变更谨慎：

- `/reload-skills` 不清 system prompt cache。
- `/skills install` 在聊天里默认不 `--now`。
- `skill_manage` 修改本地 skill 才清 cache。

这样能同时满足两个目标：

1. Agent 能使用大量专门流程。
2. 不因为 skills 太多或频繁变动，把上下文和 prompt cache 搞得很重。

## 下一步可以继续看的模块

下一个比较自然的模块是：

```text
11-配置系统与Profile.md
```

因为 skills、tools、gateway、logs、sessions 都依赖 `HERMES_HOME` 和 profile-aware path。理解配置和 profile 后，前面很多“为什么不能硬编码 ~/.hermes”的规则会更清楚。
