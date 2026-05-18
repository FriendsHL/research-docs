---
type: deep
name: codex-actor-resume-agent
status: B-mode source-grounded
harness: codex
mechanism: tokio actor mailbox with monotonic seq + watch channel + RAII SpawnReservation + tree-resume from rollout
updated: 2026-05-17
sources:
  - "[[harness/codex]]"
  - "/Users/youren/myspace/codex/codex-rs/core/src/agent/mailbox.rs:1-161"
  - "/Users/youren/myspace/codex/codex-rs/core/src/agent/registry.rs:1-344"
  - "/Users/youren/myspace/codex/codex-rs/core/src/agent/control.rs:165-200,440-523,670-760"
  - "/Users/youren/myspace/codex/codex-rs/core/src/agent/status.rs:1-28"
---

# Codex actor model + `resume_agent_from_rollout` — Rust 把 multi-agent 当 Erlang-style actor 跑

> 要点:Codex `core/src/agent/` 用 **tokio mpsc + watch channel + AtomicU64 seq** 构造 agent mailbox,**SpawnReservation RAII pattern** 管理 spawn slot,**`resume_agent_from_rollout` 用 BFS 队列恢复整棵 agent tree**。这是业内**唯一把 multi-agent 当 Erlang actor 设计的 Rust agent runtime**。
> 
> 这页深挖 3 个反直觉工程决策:**(1) mailbox 同时用 mpsc(传消息)和 watch channel(传 seq number)— 双 channel 解决 _"receiver 要异步等通知,而非 poll"_**;**(2) `trigger_turn: bool` flag 区分 push-style 和 pull-style 消息**;**(3) `resume_agent_from_rollout` 用 BFS 队列 + `list_thread_spawn_children_with_status(Open)` 递归恢复 agent tree**。
> 
> 跟 [[concept/agent-state-separation]] AgentScope StateModule(PyTorch nn.Module 风格)对照,Codex 这套是 Erlang/Rust 风。两条不同 multi-agent state 路线。

---

## 1. 为什么这个机制重要(也容易被忽略)

普通看 Codex 文档会以为它就是 _"OpenAI 官方 Rust 重写 ChatGPT-CLI"_。但深读 `core/src/agent/` 会发现:

- 5 个文件(`mod.rs / mailbox.rs / registry.rs / control.rs / status.rs / role.rs / agent_resolver.rs`)+ 3 个 test file 共 **6000+ 行 Rust**
- 完整的 **actor model**:每个 agent 是独立 thread + mailbox + status state machine
- 支持 **跨 process / 跨 host resume**:把 rollout JSONL 文件加载就恢复整棵 agent tree
- 支持 **agent fork from N turns**(`SpawnAgentForkMode::FullHistory | LastNTurns(usize)`)
- 完整的 **lifecycle**:spawn → run → interrupt → shutdown_live → close(持久化 spawn edge status)

**为什么容易被忽略**:
1. 跟 Claude Code SubAgent / Hermes child / AgentScope MsgHub 用法看起来"一样"(都是 spawn sub-agent),但**底层 isolation 模型截然不同**
2. Rust async + tokio + Erlang-actor 这套**对 Python/TS reader 是 cognitive load**,容易直接跳过
3. `resume_agent_from_rollout` 是 _"重启 process 后从硬盘恢复整棵 agent tree"_,不是 _"短期 in-memory pause/resume"_ — 业内独家 capability

**对 multi-agent runtime 设计的启示**:**multi-agent isolation 不只是 SubAgent vs Fork 二选一**,Erlang-actor 风(独立 thread + mailbox + monotonic seq)是**第三条路径**,适合 _"agent 要长期存活 + 可恢复 + 可观测每条 inter-agent message"_ 的场景。

---

## 2. 机制 — mailbox.rs 161 行做对了什么

源码 `mailbox.rs:11-49` 核心 struct:

```rust
pub(crate) struct Mailbox {
    tx: mpsc::UnboundedSender<InterAgentCommunication>,  // 消息通道
    next_seq: AtomicU64,                                  // 单调递增 seq
    seq_tx: watch::Sender<u64>,                           // ⭐ seq 通知通道
}

pub(crate) struct MailboxReceiver {
    rx: mpsc::UnboundedReceiver<InterAgentCommunication>,
    pending_mails: VecDeque<InterAgentCommunication>,      // 本地缓冲
}

impl Mailbox {
    pub(crate) fn new() -> (Self, MailboxReceiver) {
        let (tx, rx) = mpsc::unbounded_channel();
        let (seq_tx, _) = watch::channel(0);
        ...
    }

    pub(crate) fn subscribe(&self) -> watch::Receiver<u64> {
        self.seq_tx.subscribe()                            // ← 关键 API
    }

    pub(crate) fn send(&self, communication: InterAgentCommunication) -> u64 {
        let seq = self.next_seq.fetch_add(1, Ordering::Relaxed) + 1;
        let _ = self.tx.send(communication);
        self.seq_tx.send_replace(seq);                     // ⭐ 双发: 消息 + seq
        seq
    }
}
```

### 2.1 为什么有两个 channel?

**Mailbox 同时用 `mpsc` 和 `watch` channel**,看似冗余:
- `mpsc::UnboundedSender` — 传消息内容 `InterAgentCommunication`
- `watch::Sender<u64>` — 传当前 seq number

**真实需求**:receiver 要异步 _"等到有新 mail"_,**不能 poll**(浪费 CPU)。

```rust
// Option A: 只用 mpsc → receiver 一直 await rx.recv() → 阻塞
// Option B: mpsc + 外部 notify → receiver 复杂
// Option C: 只用 watch → 收不到消息内容
// Option D ⭐: mpsc + watch → receiver 既能等通知又能拿消息内容
```

Codex 选 D。Receiver 调 `subscribe()` 拿 `watch::Receiver<u64>`,在 _select! macro_ 里跟其他 future 一起 await:

```rust
tokio::select! {
    _ = seq_rx.changed() => {           // ← watch 通知
        let mails = receiver.drain();   // ← mpsc 拿内容
        // process mails ...
    }
    _ = some_other_future() => { ... }
}
```

**优势**:_"等通知"_ 和 _"拿内容"_ 解耦,receiver 可以选择 _"看了一眼 seq 但暂时不取消息"_(`has_pending`)。

### 2.2 `trigger_turn: bool` flag(`mailbox.rs:63-66`)

```rust
pub(crate) fn has_pending_trigger_turn(&mut self) -> bool {
    self.sync_pending_mails();
    self.pending_mails.iter().any(|mail| mail.trigger_turn)
}
```

每条 inter-agent message 带 `trigger_turn: bool`:
- `true` → receiver agent **必须**在收到后跑下一个 turn(push-style,紧急)
- `false` → receiver agent 在下次自然 turn 时**顺便**消费(pull-style,排队)

**业内对照**:
- **Claude Code Teammate Mailbox**(`utils/messages.ts:3457`)是 push-style,所有消息都触发处理
- **Codex** 两种都支持,**节省 LLM call**(false 的消息可以批量)

**测试印证**(`mailbox.rs:141-160`):
```rust
mailbox.send(make_mail(..., /*trigger_turn*/ false));
assert!(!receiver.has_pending_trigger_turn());

mailbox.send(make_mail(..., /*trigger_turn*/ true));
assert!(receiver.has_pending_trigger_turn());
```

### 2.3 Monotonic seq number(为什么)

`AtomicU64::fetch_add(1, Relaxed)` 保证单调递增。**3 用途**:
1. **去重**:receiver 看到 seq < 已处理 → 跳过
2. **排序**:多 sender 时 seq 决定 happens-before
3. **Debug / telemetry**:trace 一条 mail 从 send 到 deliver 的延时

**测试印证**(`mailbox.rs:94-116`):
```rust
let seq_a = mailbox.send(...);
let seq_b = mailbox.send(...);
assert_eq!(seq_a, 1);
assert_eq!(seq_b, 2);
```

---

## 3. AgentRegistry — RAII pattern 管理 spawn slot

`registry.rs:79-292` 核心抽象:

```rust
#[derive(Default)]
pub(crate) struct AgentRegistry {
    active_agents: Mutex<ActiveAgents>,
    total_count: AtomicUsize,
}

impl AgentRegistry {
    pub(crate) fn reserve_spawn_slot(
        self: &Arc<Self>,
        max_threads: Option<usize>,
    ) -> Result<SpawnReservation> {
        if let Some(max_threads) = max_threads {
            if !self.try_increment_spawned(max_threads) {
                return Err(CodexErr::AgentLimitReached { max_threads });
            }
        } else {
            self.total_count.fetch_add(1, Ordering::AcqRel);
        }
        Ok(SpawnReservation { state: ..., active: true, ... })
    }
}
```

`SpawnReservation`(`registry.rs:294-340`)是个 RAII guard:
- 创建时:`total_count.fetch_add(1)` 抢 slot
- 显式 `commit()`:正式注册 + `self.active = false`(取消 drop 时回滚)
- **Drop 时如果还 `active`**:`total_count.fetch_sub(1)` 回滚 + 释放 reserved path

```rust
impl Drop for SpawnReservation {
    fn drop(&mut self) {
        if self.active {
            if let Some(agent_path) = self.reserved_agent_path.take() {
                self.state.release_reserved_agent_path(&agent_path);
            }
            self.state.total_count.fetch_sub(1, Ordering::AcqRel);
        }
    }
}
```

**为什么 RAII**:Rust 没有 try/finally,RAII 是规范模式。Spawn agent 是多步操作(reserve slot → reserve nickname → reserve agent_path → actually spawn thread),**任何一步失败都自动回滚**。

**业内对照**:
- **AgentScope StateModule**(Python `__del__` 不可靠)用 `try/finally` 显式 commit
- **Claude Code SubAgent**(TypeScript)用 explicit cleanup function + error boundary
- **Codex** Rust RAII 是最优雅的(借编译器强制)

### 3.1 Atomic compare-exchange 防 race

```rust
[code: registry.rs:275-291]
fn try_increment_spawned(&self, max_threads: usize) -> bool {
    let mut current = self.total_count.load(Ordering::Acquire);
    loop {
        if current >= max_threads {
            return false;
        }
        match self.total_count.compare_exchange_weak(
            current, current + 1,
            Ordering::AcqRel, Ordering::Acquire,
        ) {
            Ok(_) => return true,
            Err(updated) => current = updated,  // ← race lost, retry
        }
    }
}
```

**CAS loop** 防 race condition:多个 thread 同时 spawn,只有一个成功,其他重试。**这是 lock-free 风** — 比 Mutex 性能高,无锁竞争。

### 3.2 Nickname pool with reset count(`registry.rs:202-240`)

```rust
fn reserve_agent_nickname(...) -> Option<String> {
    let available_names = names.iter()
        .map(|name| format_agent_nickname(name, active_agents.nickname_reset_count))
        .filter(|name| !active_agents.used_agent_nicknames.contains(name))
        .collect();
    if let Some(name) = available_names.choose(...) {
        name.clone()
    } else {
        // Pool exhausted → reset
        active_agents.used_agent_nicknames.clear();
        active_agents.nickname_reset_count += 1;
        codex_otel::counter("codex.multi_agent.nickname_pool_reset", 1);  // ⭐ 跟踪
        ...
    }
}
```

Agent 有 nickname(从 `AGENT_NAMES = include_str!("agent_names.txt")` 加载)。用完后 reset 加序号后缀:_"Alice"_ → _"Alice the 2nd"_(`format_agent_nickname:44-61`)。

**OTel metric `codex.multi_agent.nickname_pool_reset`**:每次 pool 耗尽都跟踪,production 可以发现"用户 spawn 太多 agent 导致 nickname pool 反复 reset"。

---

## 4. resume_agent_from_rollout — BFS 恢复整棵 agent tree

`control.rs:448-523` 这个函数是 Codex multi-agent 杀手锏:

```rust
pub(crate) async fn resume_agent_from_rollout(
    &self,
    config: crate::config::Config,
    thread_id: ThreadId,
    session_source: SessionSource,
) -> CodexResult<ThreadId> {
    let root_depth = thread_spawn_depth(&session_source).unwrap_or(0);
    let resumed_thread_id = Box::pin(self.resume_single_agent_from_rollout(...)).await?;
    
    // ⭐ BFS 队列恢复所有 descendant
    let mut resume_queue = VecDeque::from([(thread_id, root_depth)]);
    while let Some((parent_thread_id, parent_depth)) = resume_queue.pop_front() {
        let child_ids = state_db_ctx
            .list_thread_spawn_children_with_status(
                parent_thread_id,
                DirectionalThreadSpawnEdgeStatus::Open,  // ⭐ 只恢复 open edge
            )
            .await?;
        for child_thread_id in child_ids {
            let child_depth = parent_depth + 1;
            if state.get_thread(child_thread_id).await.is_ok() { ... }
            else {
                let child_session_source = SessionSource::SubAgent(...);
                self.resume_single_agent_from_rollout(..., child_session_source).await?;
            }
            resume_queue.push_back((child_thread_id, child_depth));
        }
    }
    Ok(resumed_thread_id)
}
```

**3 个核心设计**:

### 4.1 持久化的 spawn edge state(`DirectionalThreadSpawnEdgeStatus`)

每条 _"parent spawn child"_ 关系在 DB 里有 `status` 字段:`Open | Closed`。

- `close_agent(agent_id)`(`control.rs:736`)把 spawn edge 标 `Closed` + 关闭 agent tree
- `shutdown_live_agent(agent_id)`(`control.rs:714`)只关 agent,不动 spawn edge

**为什么区分**:_"shutdown"_ 是临时关(可能下次 resume),_"close"_ 是永久关。resume 时只跟着 `Open` edge 走。

### 4.2 BFS 而非 DFS

`VecDeque::pop_front()` + `push_back()` = FIFO = BFS。

**为什么 BFS**:agent tree 通常是 _"几层浅 + 每层多 child"_(parent 1 → child 5 → grandchild 25),BFS 让每层先恢复完再去下一层。如果用 DFS,可能 _"沿一条 path 深下去把 quota 用完,其他兄弟还没启动"_。

### 4.3 Recursive session source 重建

```rust
let child_session_source = SessionSource::SubAgent(SubAgentSource::ThreadSpawn {
    parent_thread_id,
    depth: child_depth,
    agent_path: None,
    agent_nickname: None,
    agent_role: None,
});
```

每个 resumed child 重建一个 `SessionSource` — `depth` 累加,parent_thread_id 链 — 保证 metadata 一致。

---

## 5. WHY — 为什么这么设计

### 5.1 为什么用 actor model 而非共享内存?

**3 个真实约束**:
- **Tool execution 可能 block**(等 user input / 等网络),所以 agent 必须 async
- **Long-lived state**(memory / file / process)跨 turn 持续,不是无状态 RPC
- **可被中断**(用户 Esc 取消),需要 `select!` 风格 cancel

Erlang actor 模型 = 独立 process(thread) + mailbox + supervised lifecycle — **正好对上 3 个约束**。

**对比 AgentScope StateModule(详见 [[concept/agent-state-separation]])**:
- AgentScope = _"agent state 是一个 nested dict,可序列化可跨进程迁移"_
- Codex = _"agent 是一个独立 thread,有自己 mailbox,可在 process 内通信"_
- 两者**正交**:AgentScope 强在 _跨 process / time travel_,Codex 强在 _in-process 多 agent 协调 + tool 执行隔离_

### 5.2 为什么 `trigger_turn` flag 分两类消息?

如前 §2.2,**节省 LLM call**。

具体场景:agent A 在 spawn agent B 之前先发一条 "你的上下文是 X" 给 B(`trigger_turn=false`),然后再发一条 "请帮我做 Y"(`trigger_turn=true`)。B 在第二条到的时候才跑一次 turn,把两条 message 一起消费 — **只跑 1 次 LLM call 而非 2 次**。

### 5.3 为什么用 RAII SpawnReservation 而非 try/finally?

Rust 没有 try/finally。RAII 是规范模式,而且 _更安全_:任何 early return / panic / error 都自动 cleanup。

试想 `try/finally` 版本:
```rust
// Wrong pseudocode
let _ = total_count.fetch_add(1);
// ... 中间任何一行 panic 都泄漏 slot ...
let result = spawn_thread(...).await;
if result.is_err() {
    let _ = total_count.fetch_sub(1);  // 必须显式 cleanup
}
```

RAII 版本(实际):
```rust
let reservation = self.state.reserve_spawn_slot(...)?;  // ← 自带 Drop cleanup
// ... 任何 panic / early return 都自动 sub 1 ...
let result = spawn_thread(...).await?;
reservation.commit(metadata);                            // ← 显式 commit
```

**编译器强制 cleanup** — 这是 Rust 比 Python/TS 在 multi-agent state 管理上的根本优势。

### 5.4 为什么 resume_agent 用 rollout JSONL 文件?

Codex 把每个 agent turn 写 rollout JSONL(`/Users/youren/myspace/codex/codex-rs/...`)。这意味着:

- **崩了重启就恢复**:类似 Erlang supervisor restart strategy
- **跨 process 迁移**:把 rollout file copy 到另一台机器就 resume
- **审计**:每个 inter-agent message 都有时间戳 + seq + 内容,可以重放 debug

业内对照:
- **Claude Code SubAgent**:in-memory state,process 重启丢失
- **AgentScope StateModule**:可序列化但**用户必须显式 save**(用 SessionBase),不是 _"每 turn 自动 append rollout"_
- **Codex** 是 _"持续写 JSONL 让 OS / fs 帮你 durability"_ 路径 — 类似 PostgreSQL WAL

---

## 6. 失败模式 — 这个机制防了哪些 bug

| 失败模式 | Codex 怎么防 | 代码位置 |
|---|---|---|
| **Spawn N 个 agent 用完资源(OOM / 过多 LLM call)** | `agent_max_threads` 配置 + `reserve_spawn_slot` CAS loop 强制限额 | `registry.rs:80-97, 275-291` |
| **Spawn 失败后 total_count 泄漏** | RAII `Drop for SpawnReservation` 自动 sub 1 | `registry.rs:331-340` |
| **Multiple writer race on `total_count`** | `compare_exchange_weak` CAS loop(lock-free) | `registry.rs:281-289` |
| **Process 崩了,丢 agent state** | `rollout JSONL` durable storage + `resume_agent_from_rollout` BFS 恢复 | `control.rs:448-523` |
| **Resume 时漏掉某个 child agent** | `list_thread_spawn_children_with_status(Open)` BFS 沿 open edge 走 | `control.rs:472-475` |
| **关闭的 agent edge 被误 resume** | `DirectionalThreadSpawnEdgeStatus::{Open, Closed}` 区分,resume 时 filter Open | `control.rs:475` + `close_agent` |
| **Inter-agent mail 丢失** | `mpsc::UnboundedChannel` 保证不丢(unbounded queue) | `mailbox.rs:23` |
| **Inter-agent mail 乱序** | `AtomicU64 next_seq` monotonic | `mailbox.rs:43-48` |
| **Receiver poll 浪费 CPU** | `watch::Sender<u64>` notify-on-change(receiver 在 select! 里 await) | `mailbox.rs:14, 39` |
| **每条消息都 trigger LLM call 浪费** | `trigger_turn: bool` flag 区分 push/pull,false 的 mail 排队 | `mailbox.rs:63-66` |
| **Nickname 用完崩溃** | Pool reset + 序号后缀("Alice the 2nd") + OTel metric 跟踪 | `registry.rs:213-234` |

---

## 7. 横向校准 — 三家 multi-agent state 模型对照

| | AgentScope StateModule | Claude Code SubAgent + Fork + Coordinator | Codex actor + mailbox |
|---|---|---|---|
| **抽象** | PyTorch `nn.Module` 风 nested state_dict | TypeScript class + in-memory | Erlang-style actor + mailbox |
| **State 容器** | StateModule._module_dict OrderedDict | Session class fields | Tokio thread + Mailbox struct |
| **持久化** | 4 backend(JSON/Redis/Tablestore/abstract)显式 save | session 序列化 JSON | rollout JSONL durable per-turn |
| **Resume 单位** | 整棵 StateModule tree(用户显式 save/load)| 整个 session | **整棵 agent tree(BFS)**|
| **跨进程迁移** | ✅(JSON 跨 lang)| 只能同 process | ✅(rollout file 可 copy) |
| **In-process inter-agent comm** | MsgHub(应用层抽象) | TeammateMailbox(message 拼到 Msg)| ⭐ mpsc + watch channel + seq(infra 级) |
| **Spawn slot 资源限制** | 无显式 | 无显式 | ⭐ `agent_max_threads` + RAII guard |
| **Push vs Pull message** | 无区分 | 都 push | ⭐ `trigger_turn: bool` 区分 |
| **Lifecycle 状态机** | 无显式 | 4 status(running/completed/failed/cancelled)| ⭐ AgentStatus 6 状态(PendingInit/Running/Completed/Interrupted/Errored/Shutdown)|
| **核心语言** | Python | TypeScript | **Rust(借编译器强制 lifetime + RAII)** |

**核心洞察**:**Codex 是把 Erlang(BEAM) 哲学搬到 Rust agent runtime**。Erlang 的 _"let it crash + supervision tree + actor isolation + durable mailbox"_ 思想全有。这是其他 Python/TS framework 学不来的(语言层面就不支持 lock-free CAS + RAII + zero-cost async)。

**对 multi-agent 设计的启示**:**没有"最好"的 multi-agent 模型**,要按业务挑:
- 要 _跨语言 / 跨平台 state portability_ → AgentScope StateModule
- 要 _Claude 内部多种 multi-agent 模式_ → Claude Code 3 模式
- 要 _长期存活 + 可恢复 + 高并发 inter-agent comm_ → Codex actor model

---

## 8. 学习视角 — 写这页让我学到的 3 件事

1. **Channel 不是只能用一个**:Codex `Mailbox` 用 `mpsc` + `watch` 两个 channel 协作 — _"消息传 mpsc,通知传 watch"_。这种 _"按 semantics 拆 channel"_ 是 Rust async 工程主流模式,但 Python/TS reader 不熟。
2. **RAII 在 multi-agent state 管理上比 try/finally 优雅 10 倍**:Spawn 多步操作任何一步失败都自动 cleanup,编译器强制 — _不可能_ 泄漏。Python `__del__` 不可靠,TS 没有析构,所以 Codex 的优雅是 _"Rust borrow checker 的 dividend"_。
3. **Durable JSONL rollout = WAL for agents**:把每个 turn append 到 JSONL,崩了重启就 replay 恢复。这是 PostgreSQL WAL / Erlang ETS DETS 的 _agent 版本_。**这种 _"靠 fs durability 做 agent state"_ 思路对 SaaS 平台很有启发**(SkillForge SubAgentRunEntity 现在是 Spring JPA,如果走 _per-turn JSONL append_ 路径会更 fault-tolerant)。

---

## 引用链路

- 源码:`/Users/youren/myspace/codex/codex-rs/core/src/agent/`
  - `mailbox.rs:1-161` 完整(Mailbox struct + Receiver + tests)
  - `registry.rs:1-344` 完整(AgentRegistry + SpawnReservation RAII + Nickname pool)
  - `control.rs:165-200,440-523,670-760` 关键函数(spawn / resume / send / shutdown / close)
  - `status.rs:1-28` 完整(agent_status_from_event + is_final)
  - `mod.rs:1-14` re-export
- Harness 入口:[[harness/codex]] §multi_agent
- 横向 concept:[[concept/multi-agent-isolation]] / [[concept/agent-state-separation]] (AgentScope StateModule 对照)
- 横向 deep:[[deep/claude-code-autodream-3-gate]] (Claude Code 3 gate 工程化对照)
