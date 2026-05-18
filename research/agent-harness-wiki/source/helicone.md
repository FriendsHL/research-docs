---
type: source
name: helicone
origin: "/Users/youren/myspace/helicone/ (Helicone Inc, MIT)"
ingested_at: 2026-05-16
ingested_commit: 3f4bd44
ingested_commit_date: 2026-05-01
ingested_version_label: "monorepo 0.1.0 + 各 package 独立 semver(以 commit 为准)"
contributed_to:
  - eval/helicone.md
  - concept/observability.md
---

# Source — Helicone

> **Y Combinator-backed LLM observability + AI Gateway**(MIT,Helicone Inc)。**核心差异:AI Gateway + observability 二合一**(100+ AI models 通过 `https://ai-gateway.helicone.ai` 单 API key,**OpenAI API 兼容,one-line code 切换**)。Self-hostable + SOC 2 / GDPR 合规。

## 仓库结构概览(monorepo,15+ 顶层 package)

```
helicone/
├── web/                            # Next.js web platform(主 UI)
├── bifrost/                        # Next.js 营销站(www.helicone.ai)
├── ai-gateway-archive/             # 旧 AI Gateway 实现
├── ai-gateway/                     # AI Gateway 主实现?(待 verify,但顶层 ls 没看到,可能在 packages/)
├── packages/                       # 核心 packages
│   ├── common/                     # 共享
│   ├── cost/                       # 成本计算
│   ├── filters/                    # 过滤器
│   ├── llm-mapper/                 # LLM provider mapping
│   ├── pricing/                    # 定价
│   ├── prompts/                    # Prompt management
│   ├── secrets/                    # Secrets
│   ├── buildPricesOpenRouter.py    # 自动同步 OpenRouter 价格
│   └── jest.config.ts
├── clickhouse/                     # ⭐ ClickHouse schema(主分析库)
├── helicone-cron/                  # Cron job 系统
├── helicone-heartbeat/             # Heartbeat service
├── helicone-mcp/                   # ⭐ MCP server(对照 Phoenix MCP)
├── cc-agent/                       # ⭐ Claude Code agent 集成
├── docker/                         # Docker 部署
├── e2e/                            # E2E 测试
├── examples/                       # 用户示例
├── docs/                           # 文档
├── docker-compose.yml + Dockerfile
├── FULL_AGENT_LOOP.md              # ⭐ Agent loop 文档
├── INTEGRATE_PROVIDER_TO_GATEWAY.md # ⭐ Provider 集成指南
├── DEVELOPER_README.md
├── DIAGRAMS.md
├── AGENTS.md + CLAUDE.md
├── mitmproxy_mac.sh + mitmproxy.sh # MITM proxy 调试
├── minio_hcone.py                  # MinIO blob storage
└── prod_push.sh + prod_push_eu.sh  # 部署脚本
```

## 关键 file:line 锚点

### 5 大支柱(README 顶部表格)

```
🔍 Observability  |  🕸️ Agent Tracing  |  🚂 LLM Routing
💰 Cost & Latency |  📚 Datasets & FT  |  🎛️ Auto Fallbacks
```

详细分:
- **🌐 AI Gateway**:100+ AI models 通过 1 API key,OpenAI API 兼容 + 智能 routing + 自动 fallback
- **🔌 Quick integration**:**1 行代码切换 `baseURL` 即接入**(README Quick Start):
  ```typescript
  const client = new OpenAI({
    baseURL: "https://ai-gateway.helicone.ai",
    apiKey: process.env.HELICONE_API_KEY,
  });
  ```
  → 自动收集 trace,跨 100+ model
- **📊 Observe**:trace / session inspection(对照 [[../eval/langfuse]] / [[phoenix]])
- **📈 Analyze**:cost / latency / quality metric + PostHog 集成(一行代码)
- **🎮 Playground**:UI iteration
- **🧠 Prompt Management**:Production data 版本 prompt + 通过 Gateway 部署不改代码
- **🎛️ Fine-tune**:OpenPipe / Autonomi 合作伙伴集成
- **🛡️ Enterprise**:SOC 2 + GDPR

### Quick Start(README,1 行代码切换)

跟 [[../eval/langfuse]] / [[phoenix]] / [[../eval/opik]] 集成方式对比:
- Langfuse:Python `@observe()` 装饰器 + JS SDK
- Phoenix:OpenInference / OTel ingestion
- Opik:Python / TS / Ruby SDK `@track`
- **Helicone:直接 OpenAI API baseURL 换 helicone proxy** — **业内最简集成**(0 SDK,1 行代码)

### `INTEGRATE_PROVIDER_TO_GATEWAY.md`

文档化了**怎么加新 provider 到 AI Gateway**(对照 OpenAI Codex 100+ crate 每 provider 一个 crate / Langfuse 23+ 集成 / Helicone 100+ models 1 gateway)。

**Helicone 走的是 single gateway 路线**(类 OpenRouter / LiteLLM Proxy),业内简化方案。

### `FULL_AGENT_LOOP.md`(独家工程文档)

文档化 **Agent 跟 Helicone 集成的完整 loop**(具体步骤未深读),业内独家把 agent loop integration 写成 markdown 文档。

### MCP Server(`helicone-mcp/`)

Helicone 自家 MCP server(对照 [[phoenix]] `js/packages/phoenix-mcp/`)— 业内 LLM observability 主流 MCP 集成。

### Claude Code Agent 集成(`cc-agent/`)

**业内独家:Helicone 仓库内置 Claude Code 集成**(具体未深读),meta-level dogfood。

### ClickHouse First-class(`clickhouse/`)

跟 [[../eval/langfuse]] / [[../eval/opik]] 同选 ClickHouse 作为主分析库。

### 多语言 + 配置文件
- `buildPricesOpenRouter.py` Python — 自动同步 OpenRouter 价格
- `mitmproxy_mac.sh + mitmproxy.sh` — MITM proxy debugging
- `minio_hcone.py` — MinIO blob storage
- `prod_push.sh + prod_push_eu.sh` — 区域化部署

### Web platform 结构(`web/`)

Next.js 主 UI 含:
- `app/` Next.js app router
- `components/` UI components
- `data/` static data
- `db/` DB 抽象
- `filterAST/` filter AST(对照 Langfuse query)
- `hooks/` React hooks
- `lib/` 工具
- `instrumentation.ts` 自家被观测

### Marketing 站(`bifrost/`)— "bifrost" 命名

Next.js 营销站(www.helicone.ai),Bifrost 是北欧神话彩虹桥,**寓意 LLM 多 model 路由桥**(暗示 AI Gateway 定位)。

## 与 wiki 现有

### 印证
- **ClickHouse + Postgres 异构存储** 跟 Langfuse / Opik / Phoenix 一致
- **MCP server**(`helicone-mcp/`)跟 Phoenix / opencode / Codex 一致业内主流
- **OpenAI API 兼容 baseURL 切换** 是 LLM proxy 业内事实标准(LiteLLM / OpenRouter / Helicone)
- **被 Ragas first-class import**(`ragas/src/ragas/evaluation.py:25 from ragas.integrations.helicone import helicone_config`)印证 Helicone 在 RAG eval 生态地位
- **Y Combinator + 商业 SaaS + self-host 开源** 跟 Langfuse / Opik 一致

### 矛盾
- **Helicone "AI Gateway" 路线**(100+ models 1 API key)跟 [[../eval/langfuse]] / [[../eval/opik]] / [[phoenix]] **observability-only 路线不同**
- **`prompts/` package 但不是 Prompt Management 标杆**(对照 Langfuse Prompt Management 一等公民 + 4 国语言 + Server+Client cache)— Helicone Prompt Management 较轻

### 新增
- **`eval/helicone.md`**(新建)— **AI Gateway-first observability** 路线
- **扩 [[../concept/observability]]**:加 Helicone "AI Gateway 路线"(单 proxy 自动收集 trace,vs SDK / OTel-based observability)

## 事实漂移 / 注意点

- **monorepo 0.1.0**(package.json `"version": "0.1.0"`)— 跟 Langfuse v3.174.1 / Opik 1.7.0 数字看似低,但 Helicone production 已多年
- **10k requests/month 免费 tier**(README)— 商业模式跟 Langfuse Cloud / Comet Opik 类似
- **`prod_push_eu.sh`** 表明 Helicone 有欧洲区(GDPR 合规)
- **没有 Optimizer**(对照 [[../eval/opik]] 7 / [[dspy]] 15)
- **没有 production loop 自动归集**(对照 Langfuse)
- **没有 ATTRIBUTION 等价**(对照 SkillForge ATTRIBUTION-AGENT / Phoenix Score.explanation)
- **MITM proxy 调试**(`mitmproxy_*.sh`)— Helicone 是 LLM 调用 proxy 性质,自己也用 MITM 调试,**工程文化漂亮**
- **`buildPricesOpenRouter.py`** 自动同步价格,对照 Langfuse `default-model-prices.json` 手动 generated artifact
