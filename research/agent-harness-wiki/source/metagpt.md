---
name: metagpt
type: source
status: B-mode source-grounded
updated: 2026-05-17
project: metagpt
version: (semver via setup.py)
commit: 11cdf466d042aece04fc6cfd13b28e1a70341b1f
license: MIT
stars: 45k+
last_ingested_at: 2026-05-17
arxiv: "2308.00352"
papers:
  - "AFlow (ICLR 2025 oral, ranking #2 in LLM-based Agent)"
  - "SPO arxiv 2502.06855"
  - "AOT arxiv 2502.12018"
contributed_to:
  - "[[harness/metagpt]]"
  - "[[concept/agent-team-patterns]]"
---

# Source — MetaGPT @ commit 11cdf466

> geekan/MetaGPT — **学术级 _"软件公司 as multi-agent system"_** 范式(45k+ stars,MIT,arxiv 2308.00352)。**Team + Role + SOP(Standard Operating Procedures)+ Environment** 抽象,**16+ 预定义 role**(Architect / ProductManager / Engineer / QAEngineer / etc)+ **25+ 预定义 action**。2025-02 推出商业产品 **MGX**(`mgx.dev`)— natural language programming "world's first AI agent development team"。clone @ `/Users/youren/myspace/MetaGPT/` ~150 MB。

---

## 1. 仓库结构

```
MetaGPT/
├── metagpt/                # ⭐ 主包
├── config/
├── docs/
├── examples/
├── tests/
├── setup.py
├── requirements.txt
├── pyproject.toml
├── Dockerfile
└── README.md (En/Cn/Fr/Ja 4 国)
```

**4 国 README** + Discord + Twitter — 国际化做的最广(对比 RAGFlow 10 国是 ASCII-art level,MetaGPT 是 community level)。

---

## 2. `metagpt/` 28 子模块 [verified]

```
metagpt/
├── _compat.py
├── actions/                # ⭐ 25+ 预定义 action
├── base/                   # 基类
├── config2.py / configs/   # 配置
├── const.py
├── context.py / context_mixin.py
├── document.py / document_store/
├── environment/            # ⭐ 7+ 类 environment (software/android/mgx/minecraft/stanford_town/werewolf)
├── exp_pool/               # experiment pool
├── ext/                    # ⭐ 7 个扩展领域 (aflow/android/cr/sela/spo/stanford_town/werewolf)
├── learn/
├── llm.py
├── logs.py
├── management/             # cost / project mgmt
├── memory/                 # memory
├── prompts/                # prompt 库
├── provider/               # LLM provider
├── rag/                    # RAG
├── repo_parser.py          # 仓库分析
├── roles/                  # ⭐ 16+ 预定义 role
├── schema.py
├── skills/
├── software_company.py     # ⭐ CLI 入口 (软件公司模拟)
├── startup.py
├── strategy/               # ⭐ planner
├── subscription.py
└── team.py                 # ⭐ Team class
```

---

## 3. 核心抽象 — Team + Role + Environment

### 3.1 `Team` class [code: team.py:32-138]

```python
class Team(BaseModel):
    """
    Team: Possesses one or more roles (agents), SOP (Standard Operating Procedures),
    and a env for instant messaging, dedicated to env any multi-agent activity,
    such as collaboratively writing executable code.
    """
    model_config = ConfigDict(arbitrary_types_allowed=True)
    
    env: Optional[Environment] = None
    investment: float = Field(default=10.0)        # ⭐ 预算上限 ($)
    idea: str = Field(default="")
    use_mgx: bool = Field(default=True)             # ⭐ MGXEnv vs Environment
    
    def __init__(self, context: Context = None, **data: Any):
        ...
        if not self.env and not self.use_mgx:
            self.env = Environment(context=ctx)
        elif not self.env and self.use_mgx:
            self.env = MGXEnv(context=ctx)         # ⭐ 默认用 MGX env
        ...
    
    def hire(self, roles: list[Role]):              # ⭐ 雇 role 进 env
        self.env.add_roles(roles)
    
    def invest(self, investment: float):
        """Invest company. raise NoMoneyException when exceed max_budget."""
        self.investment = investment
        self.cost_manager.max_budget = investment   # ⭐ cost manager 强制预算
    
    def _check_balance(self):
        if self.cost_manager.total_cost >= self.cost_manager.max_budget:
            raise NoMoneyException(...)             # ⭐ 超预算 raise
    
    def run_project(self, idea, send_to: str = ""):
        """Run a project from publishing user requirement."""
        self.env.publish_message(Message(content=idea))
    
    def serialize(self, stg_path: Path = None): ...
    @classmethod
    def deserialize(cls, stg_path: Path, ...) -> "Team": ...
```

**3 个 production engineering 独家**:
1. **`investment` + `cost_manager.max_budget`** — agent team 跑过预算自动 raise `NoMoneyException`。**业内独家 cost-aware multi-agent**(CrewAI / AG2 都没有)
2. **`use_mgx: bool` 双 env 模式** — `MGXEnv` 是新版商业 product(mgx.dev),`Environment` 是 OSS 老版
3. **Team 可 serialize / deserialize**(`SERDESER_PATH/team/team.json`)— 可保存恢复整个 team state

### 3.2 `software_company.py` — 经典 SDLC 模拟 [code: software_company.py:18+]

```python
def generate_repo(
    idea, investment=3.0, n_round=5, code_review=True, run_tests=False,
    implement=True, project_name="", inc=False, project_path="", reqa_file="",
    max_auto_summarize_code=0, recover_path=None,
):
    """Run the startup logic. Can be called from CLI or other Python scripts."""
    ...
    from metagpt.roles import (
        Architect,
        DataAnalyst,
        Engineer2,
        ProductManager,
        TeamLeader,
    )
    from metagpt.team import Team
    
    company = Team(context=ctx)
    company.hire([
        TeamLeader(),
        ProductManager(),
        Architect(),
        Engineer2(),
        # ... + DataAnalyst / QAEngineer 等
    ])
    company.invest(investment)
    company.run_project(idea)
```

**经典 SDLC 5 角色**:TeamLeader / ProductManager / Architect / Engineer / DataAnalyst — 这是 MetaGPT 范式起点(_"软件公司"_)。

### 3.3 `Role` base class [code: roles/role.py:125-, 592 行]

```python
class RoleReactMode(str, Enum):     # :82
    """ReAct 反应模式"""
    REACT = "react"                   # 默认 ReAct loop
    BY_ORDER = "by_order"             # 按 action list 顺序
    PLAN_AND_ACT = "plan_and_act"     # 先 plan 再 act

class RoleContext(BaseModel):          # :92
    """Role runtime context"""
    # msg_buffer / state / planning / news / todo / important_memory / ...

class Role(BaseRole, SerializationMixin, ContextMixin, BaseModel):
    """All roles inherit from this base"""
    
    # _observe / _think / _act / react / publish_message / put_message
    # subscribed_tags (message routing)
    # Memory + Planner (strategy/planner.py)
```

**3 react mode** vs CrewAI Process enum(sequential/hierarchical/consensual TODO):
- `REACT` — 经典 ReAct loop(observe → think → act → loop)
- `BY_ORDER` — 按预设 action list 顺序跑(类 pipeline)
- `PLAN_AND_ACT` — 先生成 plan,再按 plan 执行(类 hierarchical)

### 3.4 16+ 预定义 Role [code: roles/]

```
roles/
├── architect.py           # 软件架构师 (Architect)
├── engineer.py            # 工程师 (Engineer)
├── product_manager.py     # 产品经理 (ProductManager)
├── project_manager.py     # 项目经理 (ProjectManager)
├── qa_engineer.py         # QA 工程师
├── researcher.py          # 研究员
├── customer_service.py    # 客服
├── sales.py               # 销售
├── searcher.py            # 搜索
├── teacher.py             # 教师
├── tutorial_assistant.py  # 教程助理
├── assistant.py           # 通用助理
├── invoice_ocr_assistant.py  # 发票 OCR 助理
├── di/                    # ⭐ Data Interpreter (含 team_leader.py)
├── role.py                # Role 基类
└── prompt.py              # role prompts
```

**16+ 预定义 role 含完整软件公司 SDLC 角色**。每个 role 是具体的 Python class 继承 Role,有自家 `_act` / `_think` 实现 + 预设 actions list。

### 3.5 25+ 预定义 Action [code: actions/]

```
actions/
├── action.py / action_node.py / action_graph.py / action_output.py
├── action_outcls_registry.py
├── add_requirement.py        # UserRequirement
├── analyze_requirements.py
├── debug_error.py            # 调试错误
├── design_api.py             # API 设计
├── design_api_an.py          # API 分析
├── design_api_review.py      # API review
├── execute_task.py
├── extract_readme.py
├── fix_bug.py
├── generate_questions.py
├── import_repo.py
├── invoice_ocr.py
├── prepare_documents.py
├── prepare_interview.py
├── project_management.py     # 项目管理
├── project_management_an.py  # 项目分析
├── rebuild_class_view.py     # 类图重建
├── rebuild_sequence_view.py  # 时序图重建
└── di/                       # data interpreter actions
```

**Action vs Tool 区别**(MetaGPT 哲学):
- **Action** = 高级 _"一个 role 在做什么"_(_"设计 API"_ / _"修 bug"_)— role 的 _"职责"_
- **Tool** = 底层执行(file IO / web search / shell)
- 一个 Action 内部可调多个 Tool

**Action 是 role 的 _"行为模式"_,跟 _"我能调什么 tool"_ 是不同抽象层**。

---

## 4. `ext/` — 7 个扩展领域

```
ext/
├── aflow/             # ⭐ AFlow paper (ICLR 2025 oral)
├── android_assistant/ # Android UI agent
├── cr/                # code review
├── sela/              
├── spo/               # ⭐ SPO paper (2025-02)
├── stanford_town/     # ⭐ Generative Agents Smallville 简化版
└── werewolf/          # ⭐ Werewolf game (经典 multi-agent benchmark)
```

**3 个学术级独家 ext**:
- **aflow** — _"AFlow: Automating Agentic Workflow Generation"_ ICLR 2025 oral
- **stanford_town** — Stanford _Generative Agents_(Smallville)论文的实现 — 25 个 agent 模拟小镇社交
- **werewolf** — Werewolf game(经典 multi-agent benchmark,需要 deception + alliance + consensus)

---

## 5. `environment/` 7 类 environment

```
environment/
├── base_env.py
├── api/              # API env
├── software/         # ⭐ software dev env
├── android/          # Android env
├── mgx/              # ⭐ MGXEnv (商业新版,Team 默认用)
├── minecraft/        # Minecraft env
├── stanford_town/    # Smallville env
└── werewolf/         # Werewolf env
```

**Multi-environment 范式**:同一 Team / Role 抽象在不同 env 跑(software dev / game / smallville)— 不同领域的 multi-agent benchmark 复用 framework。

---

## 6. 关键 verified 锚点

| 路径 | 主题 |
|---|---|
| `team.py:32-138` | Team class + hire + invest + run_project + serialize/deserialize |
| `team.py:99-100` | NoMoneyException 预算 enforcement |
| `team.py:43, 50-51` | use_mgx 双 env 切换 |
| `software_company.py:18-` | 5 role SDLC 模拟入口(TeamLeader + ProductManager + Architect + Engineer + DataAnalyst) |
| `roles/role.py:82` | RoleReactMode enum(REACT / BY_ORDER / PLAN_AND_ACT) |
| `roles/role.py:92` | RoleContext schema |
| `roles/role.py:125-` | Role base class(592 行) |
| `roles/{architect,engineer,product_manager,project_manager,qa_engineer,researcher,...}.py` | 16+ 预定义 role |
| `roles/di/team_leader.py` | TeamLeader(团队领导,新版用) |
| `actions/` 25 文件 | Action 抽象 + 25+ 预定义 action |
| `ext/aflow/` | AFlow ICLR 2025 oral paper 实现 |
| `ext/stanford_town/` | Generative Agents Smallville 简化版 |
| `ext/werewolf/` | Werewolf 经典 multi-agent benchmark |
| `environment/mgx/mgx_env.py` | MGXEnv(商业新版) |
| `strategy/planner.py` | Planner integration |
| `management/` | cost_manager + project_repo + 预算 enforcement |

---

## 7. 引用 / wikilink

- 主仓库:[/Users/youren/myspace/MetaGPT/](file:///Users/youren/myspace/MetaGPT/)
- GitHub: `geekan/MetaGPT` 45k+ stars / MIT
- Arxiv:[2308.00352](https://arxiv.org/abs/2308.00352) MetaGPT 原论文
- AFlow:ICLR 2025 oral(top 1.8%)/ SPO + AOT 2025-02
- 商业产品:**MGX**(`mgx.dev`)— _"world's first AI agent development team"_(2025-02 launch)
- 衍生 wiki:[[harness/metagpt]] / [[concept/agent-team-patterns]]
