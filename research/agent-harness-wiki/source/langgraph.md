---
type: source
name: langgraph
origin: /Users/youren/myspace/langgraph (langchain-ai/langgraph @ 076e2a3, 2026-05-12)
version: 1.2.0
ingested_at: 2026-05-18
contributed_to:
  - "[[../harness/langgraph]]"
read_files:
  - "AGENTS.md (1-80)"
  - "libs/langgraph/langgraph/graph/state.py (grep class boundaries + line count 1964)"
  - "libs/checkpoint/langgraph/checkpoint/base/__init__.py (line count 860)"
  - "ls libs/ + libs/langgraph/langgraph/ + libs/checkpoint/.../ + libs/prebuilt/.../"
---

# langgraph 源码摘要(压缩版)

## 核心论点

LangChain 出品 stateful agent graph framework v1.2.0,**业内 Pregel + 状态图 paradigm 旗手** — 8 包 monorepo + Checkpoint 独立 package 3 backends + conformance test suite + StateGraph 4 generic type parameter,跟 ReAct loop / event loop / actor model 形成第 4 种 agent execution paradigm。

## 关键事实

- **Version**:1.2.0 @ 076e2a3(2026-05-12)
- **8 包 monorepo**:checkpoint / checkpoint-postgres / checkpoint-sqlite / checkpoint-conformance / cli / langgraph / prebuilt / sdk-py + sdk-js
- **dependency graph**(AGENTS.md:34-53):checkpoint base → 4 上游;prebuilt → langgraph;sdk-py → langgraph+cli;sdk-js standalone
- **StateGraph**:`graph/state.py:130` `Generic[StateT, ContextT, InputT, OutputT]` 1964 行
- **CompiledStateGraph**:`graph/state.py:1391` 编译产物分离
- **Pregel execution**:`libs/langgraph/langgraph/pregel/` 完整模块
- **Channels**:`channels/` LastValue/Topic/EphemeralValue/BinaryOperatorAggregate
- **Checkpoint base 860 行**:`libs/checkpoint/.../base/__init__.py` 完整 BaseCheckpointSaver
- **prebuilt 高层 SDK**:chat_agent_executor / tool_node / tool_validator / interrupt + _tool_call_stream / _tool_call_transformer

## 与现有 wiki 关系

- **新增**:第 4 种 agent execution paradigm(Pregel)/ Checkpoint as first-class concern 工程姿态 / StateGraph 4 generic typing
- **印证**:[[../eval/langchain]] callback system 事实标准;[[../eval/langmem]] BaseStore native binding

## 待回答

- pregel/ 内部 superstep + barrier + message dispatch 细节
- safety 5 维
- LangGraph Server 闭源 SaaS 形态
