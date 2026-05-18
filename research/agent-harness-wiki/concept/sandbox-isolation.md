---
type: concept
name: sandbox-isolation
status: stable
updated: 2026-05-17
mode: hybrid (B-mode for 6 wiki harness anchors + doc-only for 9 external systems)
related_harnesses:
  - codex
  - ragflow
  - hermes-agent
  - opencode
  - openclaw
  - dify
sources:
  # Wiki 内已 verify(直接复用 file:line 锚点)
  - "[[../harness/codex]]"           # 3 OS + V8 isolate
  - "[[../harness/ragflow]]"         # gVisor
  - "[[../harness/hermes-agent]]"    # 7 backends
  - "[[../harness/opencode]]"        # shell+pty
  - "[[../harness/openclaw]]"        # Dockerfile.sandbox
  - "[[../harness/dify]]"            # 无独立 sandbox
  # 外部 doc-only(2026-05-17 WebFetch)
  - "https://e2b.dev/docs"
  - "https://www.daytona.io/docs"
  - "https://modal.com/docs"
  - "https://vercel.com/docs/vercel-sandbox"
  - "https://gvisor.dev/docs/"
  - "https://firecracker-microvm.github.io/"
  - "https://developers.cloudflare.com/sandbox/"
  - "https://webcontainers.io/"
  - "https://github.com/containers/bubblewrap"
---

# Sandbox / 执行隔离

> 决定 _"LLM 输出 / 用户代码 跑在哪个边界内,能 touch 什么资源,出错时炸多大"_ 的工程选型 — 从最弱的 OS 进程隔离到最强的 microVM,有 6 个明确技术阶梯。

## 1. 定义与边界

**Sandbox / 执行隔离** = 给 _"不完全可信的代码"_ 划一个**物理边界**(OS API / 文件系统 / 网络 / CPU / mem / time),让它出错或被恶意利用时的爆炸半径可控。

**跟其他 concept 的边界**:

| 这页关注 | 不关注 |
|---|---|
| 单 agent 内 _"代码 / shell / tool"_ 在哪个边界跑 | agent 之间的 isolation — 那是 [[multi-agent-isolation]] |
| trust model 怎么落到 _"OS / VM / userspace kernel"_ 实现 | _"工具选哪个,谁能用"_ — 那是 [[../harness/]] tool_protocol 维度 |
| 防 _"代码越界"_ 这一类失败 | 防 prompt injection / 不可信 tool output — 那是各 harness `safety.prompt_injection` |

---

## 2. 6 阶梯隔离机制(从弱到强)

| 阶梯 | 机制 | 共享内核? | 攻击面 | 代表实现 |
|---:|---|---|---|---|
| **0** | 无隔离(进程 + 文件权限) | 是 | 全 OS API | [[../harness/dify]] 无独立 sandbox / [[../harness/opencode]] 仅 shell+pty |
| **1** | Linux namespaces + seccomp + landlock | 是 | OS API 缩窄 | **bubblewrap**([[../harness/codex]] Linux 用)/ landlock LSM |
| **2** | OS-native syscall filter | 是 | OS API 缩窄 | macOS Seatbelt(`sandbox_init(3)` + `.sbpl`,[[../harness/codex]] macOS 用)/ Windows AppContainer([[../harness/codex]] Windows) |
| **3** | 容器(OCI) | 是 + cgroups | 共享内核但加固 | Docker([[../harness/openclaw]] Dockerfile.sandbox 3 变体 / [[../harness/hermes-agent]] docker backend / Daytona / Cloudflare Sandbox) |
| **4** | Userspace kernel | **否**(自实现 syscall) | ~10% syscall overhead | **gVisor**([[../harness/ragflow]] `agent/sandbox/` 唯一显式)— Sentry 拦截 syscall + Gofer 经 9P 中转文件系统,OCI runtime `runsc` |
| **5** | microVM(KVM) | **否**(独立 kernel) | VM 边界 | **Firecracker**([Vercel Sandbox 明示 / E2B / Fly.io / AWS Lambda+Fargate])— **125ms cold start**,**<5 MiB overhead**,150 VMs/sec |
| **5'** | 单语言虚拟机隔离 | N/A | **仅同语言越界** | **V8 isolate**([[../harness/codex]] Code Mode `sandbox = ['v8/v8_enable_sandbox']` [code: `code-mode/Cargo.toml:13`])/ **WebAssembly**(WebContainers / [[../harness/smolagents]] `WasmExecutor` Pyodide+Deno)/ **Python AST interpreter**([[../harness/smolagents]] `LocalPythonExecutor` `evaluate_python_code` [code: `local_python_executor.py:1583`],**业内罕见路线** — 不是 V8 不是 WASM,是直接 ast.parse + recursive evaluate,叠加 4 道防御 + 4 硬限额) |

### ⚠️ 重要校准:5' 不是 OS-level sandbox

**V8 isolate 和 WebAssembly 不能单独承担 sandbox 责任**:

- Codex Code Mode `sandbox = ['v8/v8_enable_sandbox']` 防的是 **JS 引擎内存越界**(_"don't escape the JS heap"_),**不防文件系统访问** —— 那个 Codex 还要靠外层 bwrap+landlock+seatbelt 兜底。
- WebContainers 跑在浏览器进程内,**安全边界 = 浏览器 tab 安全边界**,不是独立 sandbox。
- Cloudflare Sandbox 官方 doc(2026-05-17)**明示用 container-based**,不是 V8 isolate —— 修正常见 community 误解。

**实际部署里,5' 经常作为第二层,叠加在 1/2/3/4/5 之上**(Codex 就是 V8 isolate + Linux bwrap 双层)。

---

## 3. 8 维设计空间

| 维度 | 选项 | 备注 |
|---|---|---|
| **D1 隔离机制** | 见 §2 6 阶梯 | 从 0 到 5 阶梯渐强 |
| **D2 执行对象** | shell command / user code(Python/JS)/ tool sandbox / agent-as-tool | RAGFlow gVisor 跑用户上传代码;opencode shell+pty 跑 LLM 生成的 shell |
| **D3 部署形态** | 本地同进程 / 本地容器 / 远程 cloud / Edge / Browser | Hermes 7 backends 覆盖 5 种 |
| **D4 资源控制** | CPU / mem / time / network / fs path | 容器以上阶梯都支持 cgroups 风格;1/2 阶梯只能粗粒度 |
| **D5 启动延迟** | ms(V8 isolate / Firecracker 125ms / Daytona 90ms)/ s(Docker)/ 10s+(完整 VM) | AI agent 场景对启动敏感 — 用户等不了 10 秒 |
| **D6 可观测性** | stdout/stderr 回流 / trace 跨边界 / log persistence | 远程 cloud sandbox 需要 SDK 桥;gVisor 走 OCI runtime 跟 docker logs 兼容 |
| **D7 跨平台** | Linux only / Linux+macOS / 三平台 / 浏览器 | [[../harness/codex]] 唯一三平台抽象(`policy_transforms.rs`)|
| **D8 Trust model** | 信任本地用户 / 不信任 LLM 输出 / 不信任跨租户 / 完全不信任(防 RCE) | 同 harness 不同 deployment 选不同阶梯 — opencode 单租户本地 vs 假设的 multi-tenant SaaS |

---

## 4. Wiki 内 6 家实现对照(verified-from-source)

| harness | 阶梯 | 关键事实 | file:line 锚点 |
|---|---|---|---|
| [[../harness/codex]] | **1+2+5'** | **业内唯一三平台抽象**:Linux bwrap+landlock / macOS Seatbelt 3 个 `.sbpl` policy / Windows 独立 crate;`policy_transforms.rs` OS abstraction layer;Code Mode V8 isolate 作为**第二层**叠加 | `sandboxing/` + `linux-sandbox/` + `bwrap/` + `windows-sandbox-rs/` + `restricted_read_only_platform_defaults.sbpl` + `code-mode/Cargo.toml:13` |
| [[../harness/ragflow]] | **4** | **业内 RAG framework 唯一显式 gVisor**;只在启用 code executor 时才必需(README explicit);Docker Compose + executor_manager + sandbox protocol spec | `agent/sandbox/` + `sandbox_base_image` + `sandbox_spec.md` |
| [[../harness/hermes-agent]] | **0-5 全谱(backend abstraction)** | **业内最广 backend 数**:7 backends — local / docker / ssh / modal / daytona / singularity / vercel_sandbox。自己不实现隔离,**做 backend SPI 抽象**,production 需要哪个挂哪个 | `tools/environments/` |
| [[../harness/opencode]] | **0** | 进程级 + shell + pty;**最轻量**,信任本地用户;不做 OS-level sandbox | `packages/opencode/src/shell/` + `pty/` |
| [[../harness/openclaw]] | **3** | 3 种 Dockerfile.sandbox 镜像:base / browser / common;容器路线;**dev environment per skill** 形态 | `Dockerfile.sandbox` + `Dockerfile.sandbox-browser` + `Dockerfile.sandbox-common` |
| [[../harness/dify]] | **0** | **无独立 sandbox**;workflow 节点**不跑用户代码**(架构约束规避问题);plugin 系统靠 plugin_tool 隔离 | `core/tools/` plugin_tool 类型 |

**横向观察**:
- Codex 是唯一**真正"做完"** 跨平台 OS-level sandbox(因为它就是 IDE 替代品,必须三平台 ship);其他都依赖单一平台或外部 SaaS。
- Hermes 的 _"backend SPI"_ 思路是**产品哲学差异** —— 它承认"sandbox 不是一个技术问题,是部署问题",做 abstraction 不做 implementation。
- Dify 的 _"承认不做"_ 也是合法工程决策 —— 通过**架构约束**(workflow 不跑代码)绕开 sandbox 整块复杂度。

---

## 5. 外部 9 家对照(doc-only,2026-05-17 fetched)

| 实现 | 阶梯 | 关键事实 | 谁在用 |
|---|---|---|---|
| **E2B** | 推测 5 | _"isolated sandboxes that let agents safely execute code"_;Templates + LLM integration first-class;**doc 未明示 isolation tech**(社区知识:Firecracker microVM)| Cloud-only,需要 API key;**wiki 内 [[../harness/smolagents]] verified 集成**(`E2BExecutor` at `remote_executors.py:338`) |
| **Daytona** | 4-5 | OCI/Docker-compatible;**<90ms cold start**;Python/TS/JS;**snapshots + stateful persistence** 是差异化点;明确 AI agent 定位 | Cloud + self-host |
| **Modal** | 推测 4-5 | Python-first serverless;**doc 未明示 isolation tech**;featured example _"Sandbox a LangGraph agent's code"_;ML/LLM 重负载场景 | [[../harness/hermes-agent]] 直接集成 + [[../harness/smolagents]] **verified 集成**(`ModalExecutor` at `remote_executors.py:729`) |
| **Vercel Sandbox** | **5(明示)** | **明示 Firecracker microVM**;Amazon Linux 2023;`node22/24/26` + `python3.13`;ms-level startup;**snapshots + persistent sandboxes(beta)**;sudo + 自定义包安装 | [[../harness/hermes-agent]] vercel_sandbox backend |
| **gVisor** | **4** | Sentry(userspace kernel)+ Gofer(filesystem 经 9P)+ OCI runtime `runsc`;**较高 per-syscall overhead**;Google 出品 | [[../harness/ragflow]] / Modal(部分) |
| **Firecracker** | **5** | KVM microVM;**125ms cold start** / **<5 MiB overhead** / 150 VMs/sec;Rust;**AWS Lambda+Fargate origin**;crosvm fork | E2B / Vercel Sandbox / Fly.io / Koyeb / Northflank |
| **Cloudflare Sandbox** | **3(明示)** | **明示 container-based(不是 V8 isolate!)** ⚠️;Python/Node/shell;AI agent 定位;依赖 Cloudflare Workers 入口;Workers Paid plan | 跟 Cloudflare Workers stack 集成 |
| **WebContainers** | **5'** WASM | 浏览器内 Node.js runtime(WebAssembly);**Node only**(无 Python/Bun);**客户端独占**;StackBlitz / Bolt.new 原产地 | 浏览器 IDE 产品 |
| **bubblewrap (bwrap)** | **1** | Linux unprivileged namespaces(user/IPC/PID/network/UTS)+ seccomp;Flatpak origin;**README 明示"不是完整 sandbox,边界完全取决于命令行参数"**;`PR_SET_NO_NEW_PRIVS` | [[../harness/codex]] Linux 直接用 / Flatpak / rpm-ostree |

### 5.1 Linux/macOS 三个 primitive(Codex 全部用)

| primitive | 角色 | 跟谁配 |
|---|---|---|
| **bubblewrap** | Linux 进程级隔离工具(§5.1 上表) | Codex Linux 第一层 |
| **landlock** | Linux LSM(kernel 5.13+),文件/网络 access policy | Codex Linux 第二层(配合 bwrap)|
| **macOS Seatbelt** | `sandbox_init(3)` + `.sbpl` policy 文件 | Codex macOS 三平台抽象的 macOS 分支 |

这 3 个加起来是 **OS-level sandbox 的 building block** —— Codex `policy_transforms.rs` 是把它们抽象成统一 API 的稀有案例。

---

## 6. Trade-off 横切

### 6.1 安全 vs 启动延迟

| 阶梯 | 启动延迟 | 安全强度 |
|---|---|---|
| 0 进程 | 0 | 弱 |
| 1-2 namespaces/seatbelt | ms | 中(共享内核) |
| 3 容器 | 100ms-s | 中+(共享内核加固) |
| 4 gVisor | s | 强(独立 syscall 层) |
| 5 microVM | 125ms+ | **最强(独立内核)** |

**反直觉点**:**5 比 3-4 快**(Firecracker 125ms < Docker 1s+) —— microVM 是个反常识工程胜利。这是为什么 cloud AI sandbox(E2B / Vercel / Fly)清一色 Firecracker。

### 6.2 跨平台成本

| 路线 | 跨平台 |
|---|---|
| bwrap / landlock | Linux only |
| Seatbelt | macOS only |
| Windows AppContainer | Windows only |
| Docker | 三平台(但需要 Docker Desktop on macOS/Windows)|
| gVisor | Linux only |
| Firecracker | Linux host only(macOS/Windows 用户经 cloud) |
| V8 isolate | 三平台(进程内) |

**关键**:**完整三平台 OS-native sandbox = 工程地狱**,这是为什么 Codex `policy_transforms.rs` 抽象层难得;大多数 harness 选 _"Docker on dev / Firecracker on cloud"_ 跳过。

### 6.3 资源粒度 vs 实现复杂度

- 阶梯 0-2:**只能粗粒度限制**(CPU 整核 / mem 整 GB / 不支持精确 fs path)
- 阶梯 3+ :**cgroups 风格精细控制**,CPU shares / mem.limit_in_bytes / network namespace
- 阶梯 5 :**完整 VM**,可以精确到 vCPU 数 / mem MB / IOPS

### 6.4 "信任 LLM 输出" 这个 trust model 怎么落

- LLM 输出 shell command → 阶梯 1-2 够(opencode 路线,信任本地用户)
- LLM 输出 Python/JS 代码 → **至少阶梯 3**(容器),production 推 4(gVisor)或 5(microVM)
- LLM 输出 + multi-tenant(SaaS)→ **强制阶梯 5**(Firecracker),否则跨租户提权风险

---

## 7. SkillForge 选型

> ⚠️ **SkillForge `Code Sandbox` 当前实现路径未 verify** — Step 1 跟用户对齐:暂标 `?`,等后续读 `/Users/youren/myspace/skillforge` 源码再补充。

### 7.1 SkillForge 现状

- ? **未 verify**:SkillForge 有 sandbox 模块但实现细节(选哪个阶梯 / 哪个 backend)未读源码确认。
- ? **待 verify**:跟 Spring Boot stack 的集成形态(是否走 Docker / 是否有 backend SPI 抽象类似 Hermes)。

**Verify 路径**:`grep -rn "sandbox\|Sandbox" /Users/youren/myspace/skillforge/` 起手,找入口模块后读核心 class。

### 7.2 条件性建议(按 SkillForge 可能场景)

| 场景 | 推荐阶梯 | 路线 | 理由 |
|---|---|---|---|
| **A. SkillForge skill 不跑用户代码,只调 typed tool** | **0** | 不上 sandbox(走 [[../harness/dify]] 路线) | 用架构约束规避问题,工程成本零 |
| **B. SkillForge skill 跑 LLM-generated shell** | **1+2** | bwrap+landlock+seatbelt 三件套 + 跨平台抽象 | 参考 [[../harness/codex]] `policy_transforms.rs` 模板;但 Spring Boot 用 Java/JVM,需自己实现 OS shell out 抽象 |
| **C. SkillForge skill 跑用户代码(Python/JS)single-tenant** | **4** | **gVisor**(OCI runtime `runsc`) | 参考 [[../harness/ragflow]] R-RF-2;Docker Compose + `runsc` 一行替换,**production-ready 最低门槛** |
| **D. SkillForge SaaS multi-tenant skill marketplace** | **5** | **Firecracker microVM** 或接入 E2B/Vercel Sandbox | 不建议自建 microVM 基础设施;接 E2B/Vercel SaaS,工程门槛低 10x |
| **E. SkillForge 想做 Code Mode V8 路线** | **5'+1** | V8 isolate + 外层 OS sandbox 双层 | **不推荐** — Codex 工程成本极高,Spring/Java stack 嵌 V8 难度大,ROI 不如 C/D |

### 7.3 推荐路线(待 SkillForge 现状 verify 后回填)

**最有可能的建议**:**走 C(gVisor)起步,D(接 E2B/Vercel SaaS)用于 cloud deployment** —— 跟 RAGFlow R-RF-2 同思路,且不绑死单一 backend(Hermes backend SPI 思路可借)。

待 verify 后,这一节会从条件性建议变成**具体决策建议 + R-SBX-N 编号**(参考 [[../harness/codex]] R-CODEX-N 风格)。

---

## 8. 学习视角

### 8.1 跟其他 concept 不同的是(differentiator)

- **Sandbox 不是 agent 设计选择,是安全护栏的物理边界** — 跟 [[multi-agent-isolation]] 的 _"agent 间隔离"_ 是不同问题,但常被混在一起讨论。
- **Sandbox 是 trust model 的实例化** —— 同一个 harness 在 _"信任本地用户 / 信任 LLM 输出 / 不信任跨租户"_ 三种 trust model 下选不同阶梯。Codex 三平台 OS sandbox + V8 isolate 双层 是 _"不信任 LLM 输出"_ 的认真实现,opencode 只 shell+pty 是 _"信任本地用户"_ 的合理偷懒。
- **跨平台 OS sandbox 是工程地狱** —— Codex `policy_transforms.rs` + 3 `.sbpl` + Windows 独立 crate 之所以难得,是因为 _"完整支持 macOS/Linux/Windows 三平台"_ 是真实需求(IDE 替代品必须做)。其他 harness 跳过,**反向印证 Docker / Firecracker / E2B 跨平台 SaaS 之所以存在**。

### 8.2 优势(strengths,横向比谁做得对)

- **Codex** 3 平台抽象做得最完整(`policy_transforms.rs`)— 业内唯一**显式 OS abstraction layer**,是 _"open-source coding agent 跨平台 sandbox 模板"_。
- **Hermes** 7 backends 做得最广(`tools/environments/`)— 把 sandbox 抽象成 backend SPI,**承认 production 需求多样,不绑定单一实现**。这种 _"backend matrix"_ 思路 SkillForge 可以直接抄。
- **RAGFlow** gVisor first-class 是 RAG framework 唯一,**做得对的事情是把 sandbox 当 P1 production 资源** —— 不是 dev-only debug 工具,而是 _"启用 code executor 时强制要求"_。
- **Vercel Sandbox / E2B / Daytona** 三家产品级 SaaS 做对的是 _"减少决策"_ —— Vercel 直接给 Amazon Linux + 三个 Node 版本 + Python 3.13,**用户不选 OS image**,降低 AI agent 接入摩擦。

### 8.3 哪些点做得好(what_done_well,elegant 设计)

- **Vercel Sandbox 的"Firecracker + Amazon Linux + 三 Node 版本 + Python 3.13"组合** — 不让用户选 OS image,直接给一个 _"AI agent 默认环境"_,减少决策。这是个 elegant 的**产品决策**。
- **Daytona snapshots + persistent sandbox** — agent 多 session 续接的隐藏需求,Daytona 用 snapshot 抽象搞定,优于 _"每次新 sandbox + 手动同步状态"_。这是看到 [[../harness/codex]] `resume_agent` 同方向的硬件级实现。
- **bubblewrap"不是完整 sandbox,只是 primitive"的诚实** — README 明示 _"the security boundary entirely depends on command-line arguments"_。这种 _"我只给积木,policy 你自己定"_ 的态度,在 sandbox 领域是健康的,避免被广泛误解的 _"开箱即用的安全"_ 错觉。Codex 用 bwrap 时,policy 写在 Codex `policy_transforms.rs` 里 —— 这才是负责任的姿态。
- **Firecracker 125ms cold start + <5 MiB overhead** — 反直觉地比 Docker 快、比 VM 轻,**改变了 _"sandbox 必须慢"_ 的工程默认假设**。这一条让 _"per-request sandbox"_ 从理论变现实,直接催生 E2B / Vercel Sandbox / Cloudflare Workers / Fly.io 整批产品。
- **Dify 承认不做** — 用架构约束(workflow 不跑用户代码)绕开整块 sandbox 复杂度。这种 _"问题最好的解决方案是消除问题"_ 的工程态度,值得在 _"什么时候需要 sandbox"_ 决策前先问一遍。

---

## 9. 待回答的问题(lint 扫描)

- [ ] **SkillForge 现状 verify** — 读 `/Users/youren/myspace/skillforge` 源码,补 §7.1 + 把 §7.3 从条件性建议改成具体 R-SBX-N 决策。
- [ ] **E2B / Modal isolation tech 二次 verify** — doc 未明示,需要从 github 源码或 blog 确认是不是 Firecracker。
- [ ] **Cloudflare Sandbox vs Workers Sandbox 名词混淆** — Cloudflare doc 把 Workers Containers / Sandbox 描述上有重叠,需 disambiguate。
- [ ] **WebContainers 是否真的 WebAssembly** — doc 只说 "browser-based runtime",WASM 是间接证据,未直接确认。
- [ ] **是否补 `deep/codex-sandbox-3-platform.md`** — Codex 跨平台抽象值得单点深挖,在 [[../backlog]] §0.1 已列可选。
- [ ] **gVisor vs Firecracker production 选型** 在 _"SkillForge multi-tenant"_ 场景下的具体 tradeoff 还可以再深挖一篇 deep 页。

---

## 10. 引用链路

**Wiki 内已 verify 的 6 个 harness 反向链接到此页**:
- [[../harness/codex]] safety.sandbox
- [[../harness/ragflow]] gVisor 段
- [[../harness/hermes-agent]] 7 backends
- [[../harness/opencode]] 进程级
- [[../harness/openclaw]] Dockerfile.sandbox
- [[../harness/dify]] 无独立 sandbox

**相关 concept**:
- [[multi-agent-isolation]] — agent 间隔离,跟本页正交
- [[code-mode-orchestration]] — V8 isolate 在 Code Mode 的角色(§2 阶梯 5' 校准)

**外部 9 doc(2026-05-17 fetched)**:见 frontmatter `sources`。
