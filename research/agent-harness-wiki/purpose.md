---
name: purpose
type: meta
updated: 2026-05-15
---

# Purpose — Agent Harness & Eval Wiki

## Why this wiki exists

这个 wiki 服务两个目的,**两者完全平级,不可偏废**。任何 ingest / query / lint 都要**同时**满足两个视角,产出 `skillforge_takeaway` 和 `learning_takeaway` 两个 frontmatter 字段。

### 1. 个人学习 / 理解 harness 设计空间(学习视角)

**核心三问**(每个 harness 页都要回答):
- 这个 harness **跟其他 harness 不同的是什么**?(differentiator)
- 它的**优势在哪**?(strengths,横向对比)
- 哪些点**做得好**?(what_done_well,值得学的 elegant 设计)

系统理解 agent harness 和评测框架的**设计空间**:
- Agent loop 有哪些范式?各自做了什么取舍?
- Context / Memory / Tool use 的工程选型有哪些?
- 多 agent 协作和自进化怎么落地?谁做得 elegant?
- 评测怎么真正闭环到 production?
- 安全护栏的最佳实践是什么?

避免每次调研都从零开始,知识必须可累积、可回查、可对比。**也避免只知"how to use"不知"why this design"**。

> 注意:**用户原有的 `research/<X>源码研究/` 笔记是学习路径记录**,有的细致有的粗糙,跟 wiki 是两个产物。Wiki 以源码为权威,笔记作为参考 + hypothesis 来源,**反过来 wiki 还能帮识别哪些笔记粗糙需补完**。

### 2. 服务 SkillForge 项目(工程视角)
SkillForge (`/Users/youren/myspace/skillforge`) 是正在迭代的服务端 Agent 平台,核心能力:
- Agent Loop 引擎(Spring Boot 实现)
- Skill 系统(可配置工具)
- SubAgent(树形)+ Multi-Agent Team(网状)
- Context Compaction(Light/Full + persistence-shape 不变量)
- Memory(pgvector)
- Eval Pipeline(Prompt A/B + 自动晋升)
- Self-Evolution(从 session 提取 skill/memory + 迭代建议)
- Trace 可观测(Langfuse 风格)
- Lifecycle Hooks + 安全护栏(命令黑名单、路径穿越防护、防失控循环)
- Multi-Provider + Channel SPI

每一个研究对象都要回答:**"对 SkillForge 意味着什么?"** —— 这就是 `skillforge_takeaway` frontmatter 字段。

## How to use

- **`/wiki-ingest <source>`** — 把一个原始资料(本地源码、博客、论文、源码笔记)纳入 wiki
- **`/wiki-query <question>`** — 基于已有 wiki 综合回答问题
- **`/wiki-compare <dim>`** — 跨 harness 横向对比某个维度,产出 comparison 页
- **`/wiki-lint`** — 健康检查:矛盾、孤儿页、维度缺失、未消化的 takeaway

## Evolving thesis

随着调研推进,这里记录持续演化的核心判断(LLM 在 ingest/query 时会读这一段)。

**完整 thesis 见 [[overview]]**(2026-05-16 首版,基于 5 harness + 2 eval framework + 12 concept ingest)。

**核心 7 个学习判断 + 8 个 SkillForge 决策建议** 一句话总结:

1. **Multi-agent 已分化为 4 条路线**(隔离 / Fork / Actor+resume / Tab+工具触发)— 选哪条取决于 trust model 跟性能预算
2. **Self-evolution 有 4 类形态**(runtime 提炼 / meta dev-facing / runtime budget 内嵌 / algorithm-driven prompt optimization)— 不互斥可叠加
3. **Compaction 7 维设计空间** Codex 训练对齐注入位置最精细,opencode SUMMARY_TEMPLATE 强制结构最易复制
4. **Hooks 数量阶梯** Codex 8 / Hermes 16 / Claude Code 27 — 跟工程文化绑定,SkillForge 应学 Codex 路线
5. **Wide Events 2.0 vs OTel 1.0 三分法** — LLM observability 行业事实路线,> 10K traces/day 切 ClickHouse
6. **Code Mode V8 isolate 是 OpenAI 独家新范式**,工程门槛高,SkillForge 不必跟
7. **`.agents/` dogfood 程度跟 LLM agent 友好度强相关** — Opik / opencode / Langfuse 是行业标杆

**SkillForge 必做(短期)**:Score 数据模型(Langfuse 抽象)/ 8 hook + output_spill / Main+Secondary 双队列分流多租户
**SkillForge 重要(中期)**:LSP first-class 集成(opencode 路线)/ Goals + token_budget per goal(Codex 路线)/ Production loop 单线打通(Langfuse 路线)
**SkillForge 演进(长期)**:`/evolve` + Opik runtime Optimizer 互补 / `.agents/` dogfood + LLM-readable AGENTS.md
**SkillForge 已领先**:Eval Pipeline + Prompt A/B + 自动晋升 / `/evolve` meta self-evolution / 统一 Provider/Channel SPI 抽象

## What success looks like

- 三个月后,有人问"你怎么看 X harness 的 context 管理",**5 分钟内能从 wiki 里给出带引用的答案**,不需要重新读源码。
- SkillForge 做新功能决策时,**先查 wiki 看业界做法**,而不是从空白纸开始。
- `log.md` 里的"触发问题"栏能反向指引:哪些问题答过、答案在哪、哪些还没答。
