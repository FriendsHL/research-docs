---
type: eval-framework
name: helicone
status: active
language: typescript

# === 版本时效 ===
last_ingested_version: "helicone-monorepo 0.1.0 @ commit 3f4bd44 (2026-05-01)"
last_ingested_at: 2026-05-16
community_known_newer: true
known_post_ingest_features:
  - 2026-05-01 之后的 ai-gateway 新 model / feature(2 周未跟进)

sources:
  - "[[../source/helicone]]"

# === 9 维度 ===
target: |
  **AI Gateway + LLM Observability 二合一 platform**(MIT,Y Combinator,Helicone Inc)。
  独家定位:**OpenAI API 兼容 proxy**(`https://ai-gateway.helicone.ai`),用户改 1 行代码切换 baseURL 即接入,**0 SDK 集成**。100+ AI models 通过 1 API key + 自动 fallback + cost/latency 跟踪。

  跟其他 platform 路线对比:
  - [[../eval/langfuse]]:**SDK-first**(Python `@observe()` / JS callback)
  - [[../eval/opik]]:**SDK + 4 维分数 + Optimizer**
  - [[phoenix]]:**OpenInference / OTel-first**
  - **Helicone**:**Gateway-first**(单 proxy 自动收 trace)

  跟 LiteLLM / OpenRouter 关系:**Helicone = OpenRouter + observability + prompt management**
  [verified-from-source]

dataset: |
  - **Datasets & Fine-tuning** 第 5 大支柱(README "📚 Datasets & Fine-tuning")
  - 跟 OpenPipe / Autonomi 合作伙伴 fine-tuning 集成
  - 没有 platform 级 Dataset 实体细节(待 verify `packages/` 实现)
  - 主要走"production trace → dataset → fine-tune"路线
  [doc-only,packages 实现未深读]

scorer: |
  - **Observability + Cost / Latency / Quality metric**(README "📈 Analyze")
  - **跟 PostHog 集成**(一行代码导出到 PostHog custom dashboard)
  - **被 Ragas first-class import**(`ragas/src/ragas/evaluation.py:25 from ragas.integrations.helicone import helicone_config`)— Helicone 是 RAG eval 生态 first-class
  - **没有 27+ 内置 metric**(对照 [[ragas]])
  - **没有 50+ assertion**(对照 [[promptfoo]])
  - **核心 score 是自动收集的 cost / latency / token usage**,LLM-as-judge 需用户自接
  [doc-only,具体 metric 实现未深读]

reproducibility: |
  - **Self-hostable**(`docker-compose.yml` + Helm)
  - **SOC 2 + GDPR 合规**
  - **EU region**(`prod_push_eu.sh`)
  - **`mitmproxy_*.sh`** MITM proxy debugging — Helicone 自己也是 LLM proxy,自家用 MITM 调试
  [verified-from-source]

coverage: |
  ### 7 大功能
  1. **AI Gateway**(100+ models,1 API key,OpenAI 兼容)
  2. **Observability**(trace + session inspection)
  3. **Agent Tracing**(README "🕸️ Agent Tracing")
  4. **LLM Routing + Automatic Fallbacks**
  5. **Cost & Latency Tracking**
  6. **Datasets & Fine-tuning**(OpenPipe / Autonomi)
  7. **Prompt Management**(Production data 版本)

  ### 集成
  - OpenAI / Anthropic / LangChain / Gemini / Vercel AI SDK / **100+ models** via Gateway
  - PostHog 一行代码集成

  ### 不覆盖
  - 没有 Optimizer(对照 [[../eval/opik]] / [[dspy]])
  - 没有 production loop 自动归集 + 归因(对照 [[../eval/langfuse]] / SkillForge ATTRIBUTION-AGENT)
  - 没有 50+ assertion(对照 [[promptfoo]])
  - 没有 27+ RAG metric(对照 [[ragas]])
  - 没有 trace tree UI(对照 [[phoenix]] 13 组件)
  - 没有 red teaming(对照 [[promptfoo]])
  [verified-from-source + doc-only]

production_loop: |
  **Gateway-driven production loop**:
  - LLM 调用经过 `ai-gateway.helicone.ai` proxy
  - Proxy 自动收 cost / latency / model / prompt / response
  - Dashboard 看 trace + session inspection
  - **Prompt Management** 可在 UI 改 prompt,**通过 Gateway 部署不改代码**(README "Deploy prompts through the AI Gateway without code changes")

  **关键 insight**:Helicone 不做"自动归因 / dataset 归集 / canary 灰度",但**"prompt 通过 gateway 部署"是独家**(SkillForge canary 灰度走自家 backend)。
  [doc-only,具体 prompt deploy 实现未深读]

automation: |
  - **OpenAI API baseURL 切换**:1 行代码集成(README Quick Start)
  - **MCP server**(`helicone-mcp/`)— 业内 LLM proxy 主流 MCP
  - **Claude Code agent 集成**(`cc-agent/`)— 业内独家把 Claude Code 集成 cloned 在仓库内
  - **`helicone-cron/`** + **`helicone-heartbeat/`** 服务化
  - **`prod_push_eu.sh` 区域化部署**
  [verified-from-source]

cost: |
  - **核心卖点是 cost tracking**(README "💰 Cost & Latency Tracking" 第 1 大功能之一)
  - **`buildPricesOpenRouter.py`** 自动同步 OpenRouter 价格(Python 脚本)
  - **`packages/cost/` + `packages/pricing/`** 独立 module
  - **`helicone-cron/`** 跑定时价格 / cost 计算
  - **10k requests/month 免费 tier**(无信用卡)
  [verified-from-source]

# === SkillForge 校准 takeaway ===
skillforge_takeaway: |
  ## 🎯 关键定位:Helicone 是 **AI Gateway-first observability**,跟 SkillForge 当前 SDK-based observability 不同路线

  SkillForge 当前:
  - **`skillforge-observability`** 独立模块
  - **OBS-1/2/4** trace 模型 + session × trace 合并 + 跨 agent 串联
  - **PROMPT-CACHE-MVP**(2026-05-08)— 5 provider 全链路 + Claude 3 breakpoint cache

  Helicone 走 "**proxy → 自动收集 → dashboard**"路线,**SkillForge 当前是 Java backend 内部直接 instrument**。

  ## 可借鉴(给 SkillForge observability 工程细节)

  ### 高优(1-2 周)
  1. **`buildPricesOpenRouter.py` 自动同步 OpenRouter 价格** — SkillForge 模型定价当前手动维护,可借鉴 Helicone 自动同步 OpenRouter API 模式(对照 Langfuse `default-model-prices.json` generated artifact)
  2. **`packages/cost/` + `packages/pricing/` 独立 module** — SkillForge cost / pricing 可拆独立 module(对照 [[../harness/codex]] 100+ crate 极细拆分)
  3. **MCP server 暴露 Helicone 数据** — SkillForge dashboard 可暴露 MCP server 让用户 IDE 一键查 trace(对照 [[phoenix]] MCP)

  ### 中优(1 个月)
  4. **AI Gateway 路线对照** — SkillForge multi-provider SPI 抽象已轻量(SkillForge 自家路线),**不必走 Helicone gateway 路线**,但工程模板可对照(对照 SkillForge ChannelAdapter SPI 是同思路)
  5. **`mitmproxy_*.sh` MITM debugging** — SkillForge 调试 LLM 调用时可用类似工具(Charles / mitmproxy)
  6. **`helicone-heartbeat/`** 服务化 — SkillForge 监控可拆 heartbeat service

  ### 不必跟
  - **Gateway-first 集成路线**(SkillForge 是 platform 不是 proxy)
  - **0 SDK 集成**(SkillForge 已有自家 SDK 抽象,不重做)
  - **`cc-agent/` 仓库内置 Claude Code** — SkillForge 走自家 dogfood 路线
  - **完整 100+ model gateway**(SkillForge multi-provider SPI 已足够)

  ## SkillForge 已领先
  - **跨 agent / 跨 session trace 串联**(OBS-4 root_trace_id)— Helicone 是单 session trace
  - **Prompt cache 5 provider 全链路 + Claude 3 breakpoint**(PROMPT-CACHE-MVP)— Helicone Prompt Management 较轻
  - **ATTRIBUTION-AGENT + canary 灰度回流**(飞轮 V2+V3)— Helicone 完全没有
  - **MULTIMODAL-MVP + MCP-CLIENT-MVP** — Helicone 不做

  ## 关键决策建议

  **如果 SkillForge 商业化考虑加 AI Gateway 路线**:
  - **不推荐**:SkillForge 当前是 server-side platform,不是 LLM proxy,定位不冲突但 gateway 路线工程量大且偏离 platform 价值
  - **如果用户场景需要 multi-model API gateway**:可考虑跟 Helicone / OpenRouter / LiteLLM 集成而非自建

  **如果 SkillForge 跟踪 cost 加强**:
  - **借鉴 `buildPricesOpenRouter.py` 自动同步价格**(对照当前 SkillForge 是否自动同步)
  - **`packages/cost/` + `packages/pricing/` 拆分** — 工程组织参考

learning_takeaway: |
  ## 跟其他 eval 不同的是 (differentiator)
  1. **AI Gateway + LLM Observability 二合一** — 业内独家定位(对照 LiteLLM/OpenRouter 只 gateway / Langfuse/Opik/Phoenix 只 observability)
  2. **OpenAI API baseURL 兼容 proxy**(`https://ai-gateway.helicone.ai`)+ 1 行代码集成 — 业内最简集成
  3. **100+ models 1 API key + 自动 fallback** — Gateway 卖点
  4. **Prompt 通过 Gateway 部署不改代码** — 业内独家"prompt deploy without code change"
  5. **`buildPricesOpenRouter.py` 自动同步 OpenRouter 价格** — 业内独家自动化价格同步
  6. **`mitmproxy_*.sh` MITM proxy debugging** — Helicone 自己是 proxy,用 MITM 调试自己(meta!)
  7. **`cc-agent/` 仓库内置 Claude Code 集成** — 业内 LLM observability 独家把 Claude Code 跟自家产品深度集成
  8. **`bifrost/` 营销站命名为北欧彩虹桥**(LLM 多 model 路由桥)— 工程命名诗意
  9. **`packages/secrets/` 独立** — 业内少见把 secrets 单独 package(类 Codex `secrets/` crate)
  10. **EU region**(`prod_push_eu.sh`)+ SOC 2 + GDPR — 合规一等公民
  11. **10k requests/month 免费 tier 无信用卡** — 比 Langfuse Cloud / Comet Opik 更慷慨入门
  12. **Y Combinator backed**(对照 Langfuse W23)— YC 系 LLM observability 双雄

  ## 优势 (strengths)
  1. **0 SDK 集成**:`baseURL` 切换 + 同一 OpenAI client — dev 体验业内最佳
  2. **`packages/llm-mapper/`** LLM provider mapping — 100+ models 统一接口
  3. **`packages/filters/`** + `filterAST/` filter AST — query 工程化
  4. **PostHog 一行代码导出** — 跟 BI 工具互通
  5. **`helicone-cron/`** 服务化定时任务
  6. **`helicone-heartbeat/`** 健康检查独立 service
  7. **`docs/`** 完整文档站 + `DIAGRAMS.md` 架构图
  8. **`FULL_AGENT_LOOP.md`** 独家工程文档
  9. **`INTEGRATE_PROVIDER_TO_GATEWAY.md`** 加 provider 指南
  10. **`minio_hcone.py`** MinIO blob storage — self-host 完整体系
  11. **`genSupabaseTypes.sh`** Supabase types 自动生成 — DB schema 跟 TS 类型同步

  ## 哪些点做得好 (what_done_well)
  1. **README 顶部 6 大支柱 emoji 表格**(🔍 Observability | 🕸️ Agent Tracing | 🚂 LLM Routing | 💰 Cost & Latency | 📚 Datasets & FT | 🎛️ Auto Fallbacks)— **业内最清晰定位**
  2. **`bifrost/` 命名**(北欧彩虹桥)— LLM gateway 寓意诗意
  3. **`buildPricesOpenRouter.py` Python 脚本同步价格** — 没堆 TS 因为 Python 数据脚本天然
  4. **`cc-agent/` 仓库内置 Claude Code** — meta-level dogfood
  5. **`docker-compose.yml` + `helicone-compose.sh` 包装** — self-host 体验
  6. **`mitmproxy_*.sh` debugging** — proxy 的 proxy 调试,工程文化漂亮
  7. **`prod_push.sh + prod_push_eu.sh` 区域化** — 国际化合规
  8. **`packages/` 内部拆 7+ package**(common / cost / filters / llm-mapper / pricing / prompts / secrets)— **集中度高于 Langfuse `packages/shared/` 1 package,拆分粒度业内中等**
  9. **`README.md` 头 `Free tier 10k requests/month - No credit card required`** — onboarding 友好
  10. **`SOC 2 + GDPR` 一等公民** — Enterprise 卖点
---

# Helicone(Y Combinator, MIT, 0.1.0)

## 一句话定位

**AI Gateway + LLM Observability 二合一 platform**(MIT,Y Combinator)。1 行代码 baseURL 切换接入 + 100+ models 1 API key + 自动 fallback + cost/latency 跟踪 + Prompt 通过 Gateway 部署不改代码。

## 6 大支柱

```
🔍 Observability  |  🕸️ Agent Tracing  |  🚂 LLM Routing
💰 Cost & Latency |  📚 Datasets & FT  |  🎛️ Auto Fallbacks
```

## 4 路线对照(Helicone 在 Gateway-first 第 5 层)

```
Platform 集成路线 5 层:
1. Langfuse / Opik     — SDK-first(Python `@observe()` / JS `Langfuse()`)
2. Phoenix             — OpenInference / OTel-first(semconv 标准)
3. DSPy                — Framework(import dspy)
4. Promptfoo           — CLI-first(YAML config + 50 assertion)
5. Helicone            — Gateway-first(baseURL 切换 + 0 SDK)←—— 你在这层
6. Ragas               — Metric library(被其他平台集成)
```

## SkillForge 校准

详见 frontmatter `skillforge_takeaway`。核心:

- **Helicone 路线不适合 SkillForge**(SkillForge 是 platform 不是 proxy,定位不同)
- **可借鉴的工程细节**:`buildPricesOpenRouter.py` 自动同步价格 + `packages/cost+pricing` 拆分模块 + MCP server 暴露 trace
- **SkillForge 已领先**:跨 session trace 串联 / Prompt cache 5 provider / ATTRIBUTION-AGENT / 飞轮闭环

## 待回答
- `ai-gateway/` 主实现代码位置(顶层 ls 没看到,可能在 packages 或独立)
- `FULL_AGENT_LOOP.md` 具体描述的 agent loop integration 流程
- `cc-agent/` 内 Claude Code 集成细节
- Prompt 通过 Gateway 部署的具体机制(zero-code change deployment)
- Helicone 跟 PostHog 一行代码集成的实现
