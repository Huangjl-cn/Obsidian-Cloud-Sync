
> 聚焦三个问题:**能做什么 → 怎么用 → 底层是怎么实现的**。
> 扩展概念(worktree、handle、runtime 等)在正文首次提到处用 `>` 引用块就地解释。

---

## 1. Orca 编排能做什么

一句话:Orca 编排是**多 Agent 协作的协调层**——解决"多个 Agent 如何并行干活、如何互通消息、如何追踪进度、如何收尾"。

| 能力             | 具体是什么                                 |
| -------------- | ------------------------------------- |
| **并行派活**       | 把若干任务同时分给不同 Agent(每个在自己独立的工作区里),互不干扰  |
| **任务工单(DAG)**  | 任务可声明依赖,Orca 判定谁 ready、谁等谁            |
| **Agent 间消息**  | 互相发消息:支持类型/优先级/线程,可追问可回复              |
| **生命周期上报**     | 被派发的 Agent 干完主动报告,系统自动结算任务;卡住可求助      |
| **决策门**        | 协调者可在关键节点挂起任务,等人拍板再放行                 |
| **失败重试与熔断**    | 同一任务可重派,连续失败自动标记 failed               |
| **资源收尾**       | worker 完成后释放/保留其终端,输出留档可再审            |
| **跨服务器**       | 通过稳定逻辑地址把 worker 派到远程 Orca 服务器        |

---

## 2. 怎么用:CLI 命令的完整过程

### 2.1 一条任务的完整生命周期

```bash
# ── 1. 立项:一个 Run = 一次协作会话 ──────────────────────────
orca orchestration run-create --objective "登录功能前后端开发"
# → run_abc123(之后所有东西都挂在这个 run_id 下)
#
#   关键:run-create 默认把"当前终端"(或 --from 指定终端)绑定为该 Run 的协调者
#   runs.coordinator_handle = 绑定终端 handle(单协调者模型)
#   因此后续 check 不带 --run 时,默认检查的就是"这个终端绑定的 Run"
#   多个编排者各自 run-create → 各自绑定,check 互不干扰
#   例外:恢复/兼容路径产生的 Run 可能未绑定(coordinator_handle=NULL),
#   此时 check 必须显式 --run

# ── 2. 建工单:Task 描述"要做什么"(纯声明,还没人做)────────────
orca orchestration task-create --spec "后端实现 /api/login"
orca orchestration task-create --spec "前端实现登录页" --deps '["<task1>"]'
# → task_100 / task_200

# ── 3. 派活:worker-start = 开 worktree 终端 + 注入任务书 + 派发 ──
orca orchestration worker-start --task task_100 --worktree tmp-backend --agent pi
orca orchestration worker-start --task task_200 --worktree tmp-frontend --agent pi
# → 每个 worker 拿到一个 dispatch 身份(taskId + dispatchId)

# ── 4. 协调者等结果(阻塞,等特定类型消息)────────────────────────
orca orchestration check --wait --types worker_done,escalation,question --timeout-ms 900000

# ── 5. worker 端:被派发的 Agent 做完后上报 ─────────────────────
orca orchestration send --type worker_done --task-id task_100 --dispatch-id <d1> \
  --outcome succeeded --files-modified "src/a.java" --body "完成摘要"

# ── 6. 协调者收到后收尾:复用或释放该 worker ────────────────────
orca orchestration worker-release --dispatch <d1>

# ── 7. 最后在 git 层合并各分支(push + PR 或本地 merge)──────────
```

### 2.2 协调者与 worker 的"电话本"

```bash
# 找工作区(所有项目的工作树都能看到)
orca worktree list --json

# 找某工作区下的 Agent 终端(拿 handle)
orca terminal list --worktree name:tmp-frontend --json

# 读了/写了,但不用编排时(轻量提醒):
orca terminal send --terminal <handle> --text "继续" --enter --json

# 需要追踪的协作消息:
orca orchestration send --run run_abc123 --to <run|dispatch> --subject ... --body ... --json
orca orchestration check --run run_abc123 --format --json
orca orchestration reply --id <msg_id> --body "..." --json
```

### 2.3 两种"发消息"的取舍

| 场景                   | 用哪个                                                |
| -------------------- | -------------------------------------------------- |
| 一句话提醒、打断、给指令         | `terminal send`(即时,像人打字)                           |
| 任务书、工单、需要对方确认收到      | `orchestration send`(落库,可追踪,对方 check 拉取)           |
| worker 干完汇报 / 被卡住求助  | `send --type worker_done / question / escalation`  |
| 协调者回 worker 的问题      | `reply --id <msg_id>`                              |

> **worktree 是什么**:git 的"一个仓库多份工作区"机制。每个 worktree 有独立的工作区文件 + 私有 HEAD/index,但共享同一个对象库和全部分支。多个 Agent 各占一个 worktree,`git add/commit` 互不干扰,最后各自分支 merge 回主线。worktree 的 `.git` 只是一个 100 字节的指针文件(`gitdir: 主仓库/.git/worktrees/<id>`),**不复制对象库**。

> **handle 是什么**:Runtime 给每个 Agent 终端会话发的 UUID(`term_xxx`),是 PTY 注册表的 key。Runtime 重启后失效,需重新 `terminal list`;需要长期稳定地址时用 `dispatch:<id>`。

> **runtime 是什么**:Orca 后台真正干活的常驻服务(Electron 主进程或 `orca serve` 无头运行)。所有 CLI 命令都是 JSON-RPC 到它;它持有全部 PTY、写 SQLite、跑编排逻辑。

---

## 3. 底层实现:两条管道

核心结论:**Orca 有两条完全独立的通信管道,对应两种发送方式**。

```

┌────────────────────────────────────────────────────┐

│                  Orca Runtime                       │

│                                                    │

│  实时管道                    持久管道               │

│  PTY 直写                    SQLite 信箱            │

│  terminal send ──┐           orchestration send ──┐ │

│                  ▼                                ▼ │

│  node-pty.write()             INSERT INTO messages │ │

│  对方 TUI 立即显示           对方 check 主动拉取    │ │

└────────────────────────────────────────────────────┘

```

### 3.1 实时管道:terminal send

终端本质是 **node-pty** 创建的伪终端(PTY):

```js
// 主进程实际代码形态(截取):
pty.spawn({
  cols: 120,
  rows: 40,
  cwd: e.worktree.path,       // 工作区目录
  command: O.launchCommand,   // agent 启动命令
  ...{shellOverride: "wsl.exe"}  // 某些环境走 WSL
})
```

发送就是往 PTY master 写字节流:

```js
terminalSend(handle, text) {
  const pty = registry.get(handle)   // 按 handle 从注册表找 PTY 实例
  pty.write(text + '\n')             // 写 master = 等效用户击键
  return { bytesWritten: text.length }
}
```

```
IPC 通道(运行时协议):
  pty.spawn / pty.attach / pty.data / pty.exit
  pty.replay / pty.serialize / pty.revive / pty.resize / pty.shutdown
```

> **PTY 是什么**:内核提供的模拟终端设备对(master/slave)。Agent TUI 打开 slave 端当 stdin/stdout;master 端由 Orca 持有,写它 = 往终端"按键",读它 = 拿 Agent 的屏幕输出。Windows 走 ConPTY,Linux/macOS 走 /dev/ptmx。与真人打字在终端侧无法区分。

**特点**:即时、无状态(不写库)、无 ack——发完就完,对方 TUI 立即显示并处理。

### 3.2 持久管道:orchestration send

#### 发送 = INSERT INTO messages

```sql
-- 实际执行的核心 SQL(截取自打包代码):
INSERT INTO messages (
  id, run_id, delivery_contract, from_handle, to_handle,
  subject, body, type, priority, thread_id, payload, read, sequence
)
```

```js
sendMessage({to, subject, body, type}) {
  const msg = {
    id: `msg_${randomId()}`,
    run_id: currentRun.id,
    delivery_contract: pickContract(to),  // 关键:按地址选合约
    from_handle: currentTerminal.handle,  // 自动解析为调用者终端
    to_handle: to,                        // run:xxx / dispatch:xxx / term_xxx
    read: 0, sequence: nextSequence()     // FIFO
  }
  db.prepare("INSERT INTO messages (...) VALUES (...)").run(msg)
}
```

**注意 `--run` 与 `--to` 是两个维度**:`--run run_xxx` 声明消息**归属于哪个 Run**(写进 `messages.run_id`,决定记账/审计范围,可省略——默认取当前终端绑定的 Run);`--to` 才是**投递到哪个信箱**(写进 `to_handle`,决定谁能 check 到)。投 Run 信箱时两者是同一个 run id:`--run run_abc --to run:run_abc`(一个裸写、一个带 `run:` 前缀)。

**send 返回的两类 ID**:`result.message.id`(如 `msg_41ded8606b77`)是**单条消息 ID**,供 `reply --id` / `ask --resume` 定位;`check` 返回的 `result.deliveryId` 是**批次 ID**,供 `check --ack` 结算。回复串线程用 message id,消费确认用 delivery id。

**地址 → 合约分支**:

| 地址                  | contract            | 效果                                                                    |
| ------------------- | ------------------- | --------------------------------------------------------------------- |
| `run:run_xxx`       | `current_delivery`  | 标准持久信箱,可 reply/ack/thread                                             |
| `dispatch:<id>`     | `current_delivery`  | 精确派发,可跨服务器 relay                                                      |
| `term_xxx`(handle)  | `legacy_direct`     | 轻量;绑定终端生命周期,接收方只能只读查看(提示 "reply and acknowledgment are unavailable")  |

#### 拉取 = SELECT 最老未确认批次

```js
check(runId) {
  const d = db.prepare(
    "SELECT * FROM deliveries WHERE run_id=? AND status='outstanding' ORDER BY consumer_generation LIMIT 1"
  ).get(runId)
  if (!d) return { messages: [], count: 0 }
  return {
    deliveryId: d.id,
    messages: JSON.parse(d.message_ids).map(id => getMessage(id))
  }
}
```

#### ack = UPDATE deliveries

```sql
UPDATE deliveries SET status='acknowledged', acknowledged_at=now() WHERE id=?
```

**「能不能 check 到自己的消息」取决于 `--to` 地址,不取决于发送者身份**——`send` 只负责写 `messages` 表并生成批次,不检查发信人。投到 `run:run_xxx`(共享信箱)的消息,协调者自己 `check` 也会拉到自己发的那条;投到 `dispatch:<id>` 或 `term_xxx`(定向信箱)则只有接收方拉得到。这也解释了为什么正常编排中协调者给 worker 的指令走定向地址,只有 worker 回的 `worker_done`/`reply` 才进 Run 信箱由协调者消费。

**未 ack 前同一批次反复返回 → 至少一次投递(at-least-once)**;ack 后消息本身 `read` 置 1。

#### 生命周期图

```
send(INSERT messages, sequence++)
   → 投递批次(deliveries: message_ids=JSON数组, status='outstanding')
   → 接收方 check 拉取最老批次
   → 处理完 check --ack(status → 'acknowledged')
   → 不 ack 就每次都返回同一批(at-least-once)
```

配套实现细节:

- `thread_id`:`reply --id X` 生成的新消息带 `thread_id=X`,串成线程
- `mutation_receipts / mutation_receipt_ledger`:操作幂等账本,`--retry-request` 重放复用同一身份
- `priority`(normal/high/urgent)影响唤醒顺序;`read` 是消息级已读标记

---

## 4. 状态存储:SQLite

- **文件**:`AppData/Roaming/orca/orchestration.db`
- **驱动**:Node 24 内置的 `node:sqlite`(`DatabaseSync`),通过 `db.prepare(...).run/get/all` 访问
- **表(25 张,按职责)**:

```
命名空间/信箱  runs, coordinator_runs, run_coordinator_handles, deliveries, messages
任务/派发      tasks, dispatch_contexts, worker_dispatches,
               worker_terminal_resources, worker_terminal_archives
决策/问答      decision_gates, question_threads, remote_questions
幂等/审计      mutation_receipts, mutation_receipt_ledger, mutation_caller_identities
兼容/远程      legacy_*, remote_dispatch_attachments, federation_relay_items,
               federated_dispatches
```

### 4.1 核心表完整 DDL

```sql
CREATE TABLE runs (
  id                    TEXT PRIMARY KEY,
  objective             TEXT NOT NULL,
  home_database         TEXT NOT NULL DEFAULT 'this_database',
  coordinator_handle    TEXT,             -- 绑定哪个终端是协调者
  coordinator_pane_key  TEXT,
  consumer_generation   INTEGER NOT NULL DEFAULT 0,  -- 信箱消费代数
  legacy                INTEGER NOT NULL DEFAULT 0,  -- 1=旧兼容占位(只读)
  created_at / updated_at
)

CREATE TABLE messages (
  id            TEXT PRIMARY KEY,
  run_id        TEXT NOT NULL DEFAULT 'run_legacy_local',
  delivery_contract TEXT NOT NULL DEFAULT 'current_delivery'
    CHECK(delivery_contract IN ('legacy_direct', 'current_delivery', 'audit_only')),
  from_handle   TEXT NOT NULL,    -- 发送方终端
  to_handle     TEXT NOT NULL,    -- run:xxx / dispatch:xxx / term_xxx
  subject       TEXT NOT NULL,
  body          TEXT NOT NULL DEFAULT '',
  type          TEXT NOT NULL DEFAULT 'status'
    CHECK(type IN ('status','dispatch','worker_done','merge_ready',              'escalation','handoff','decision_gate','question','heartbeat')),
  priority      TEXT NOT NULL DEFAULT 'normal' CHECK(priority IN ('normal','high','urgent')),
  thread_id     TEXT,             -- reply 串联
  payload       TEXT,             -- JSON(taskId/dispatchId/files-modified)
  read          INTEGER NOT NULL DEFAULT 0,
  sequence      INTEGER           -- FIFO 顺序号
)

CREATE TABLE deliveries (
  id                    TEXT PRIMARY KEY,
  run_id                TEXT NOT NULL,
  consumer_generation   INTEGER NOT NULL,
  message_ids           TEXT NOT NULL,   -- JSON 数组:一批包含哪些消息
  status                TEXT NOT NULL DEFAULT 'outstanding'
    CHECK(status IN ('outstanding', 'acknowledged', 'fenced')),
  created_at / acknowledged_at
);

CREATE TABLE tasks (
  id            TEXT PRIMARY KEY,
  run_id        TEXT NOT NULL,
  parent_id     TEXT,                    -- 子任务(可选)
  created_by_terminal_handle TEXT,       -- 谁建的工单
  created_by_run_generation  INTEGER,
  task_title / display_name TEXT,
  spec          TEXT NOT NULL,           -- 任务书全文
  status        TEXT NOT NULL DEFAULT 'pending'
    CHECK(status IN ('pending','ready','dispatched','completed','failed','blocked')),
  deps          TEXT NOT NULL DEFAULT '[]',   -- JSON:依赖哪些 task
  result        TEXT,                    -- 结算时写入的结果
  created_at / completed_at
);

CREATE TABLE dispatch_contexts (
  id                  TEXT PRIMARY KEY,
  run_id / task_id    TEXT NOT NULL,
  contract_version    INTEGER NOT NULL DEFAULT 1,   -- 生命周期合约版本
  launch_token_hash   TEXT,             -- 启动凭据哈希(防伪造)
  assignee_handle     TEXT,             -- 被派给的终端
  assignee_pane_key   TEXT,
  capability_hash     TEXT,
  process_incarnation TEXT,
  capability_revoked_at TEXT,
  status              TEXT NOT NULL DEFAULT 'pending'
    CHECK(status IN ('pending','dispatched','completed','failed','circuit_broken')),
  failure_count       INTEGER NOT NULL DEFAULT 0,   -- 失败计数(3 次熔断)
  last_failure        TEXT,
  termination_reason  TEXT,             -- 进程消失原因
  depth               INTEGER NOT NULL DEFAULT 1,   -- 嵌套深度:root 的 worker=1
  dispatched_at / completed_at / last_heartbeat_at
);

CREATE TABLE worker_dispatches (
  dispatch_id            TEXT PRIMARY KEY,
  runtime_epoch          TEXT,
  state                  TEXT NOT NULL DEFAULT 'starting'
    CHECK(state IN ('starting','ready','start_unknown','failed','succeeded',
                    'stopping','stop_unknown','stopped','abandoned')),
  stage                  TEXT NOT NULL DEFAULT 'accepted',
  worktree_id            TEXT,          -- 哪个 worktree
  agent_terminal_handle  TEXT,          -- 哪个 agent 终端
  setup_state            TEXT NOT NULL DEFAULT 'not_applicable',
  effects                TEXT NOT NULL DEFAULT '[]',   -- 启动造成的效果清单
  residual_resources     TEXT NOT NULL DEFAULT '[]',   -- 未释放资源
  start_options          TEXT NOT NULL DEFAULT '{}',
  last_error             TEXT,
  created_at / updated_at
);

CREATE TABLE decision_gates (
  id            TEXT PRIMARY KEY,
  run_id / task_id  TEXT NOT NULL,
  question      TEXT NOT NULL,          -- 拍板问题
  options       TEXT NOT NULL DEFAULT '[]',   -- JSON:候选项
  status        TEXT NOT NULL DEFAULT 'pending'
    CHECK(status IN ('pending','resolved','timeout')),
  resolution    TEXT,
  created_at / resolved_at
);

CREATE TABLE question_threads (
  message_id            TEXT PRIMARY KEY,   -- question 消息 id
  run_id / dispatch_id  TEXT NOT NULL,
  asker_handle          TEXT NOT NULL,
  status                TEXT NOT NULL DEFAULT 'pending'
    CHECK(status IN ('pending','answered','closed')),
  answer_message_id     TEXT,           -- reply 生成的消息
  answer_body           TEXT,
  answered_by_generation INTEGER,
  created_at / answered_at / closed_at
);

CREATE TABLE worker_terminal_resources (
  id                       TEXT PRIMARY KEY,
  origin_dispatch_id       TEXT NOT NULL,   -- 最初占有者
  owner_dispatch_id        TEXT NOT NULL,   -- 当前占有者
  prior_owner_dispatch_ids TEXT NOT NULL DEFAULT '[]',  -- 转移历史
  worktree_id / terminal_handle / pane_key / process_incarnation / host_scope,
  ownership_state          TEXT NOT NULL DEFAULT 'owned'
    CHECK(ownership_state IN ('owned','transferred','user_owned','external','released')),
  release_state            TEXT NOT NULL DEFAULT 'not_requested'
    CHECK(release_state IN ('not_requested','retained','requested','releasing','released','unknown')),
  retained_reason / release_requested_at / release_completed_at / release_error,
  archive_source / archive_status,     -- 输出留档
  created_at / updated_at
);
```

### 4.2 设计要点

- **runs**:一个 Run 只绑定**一个** `coordinator_handle`(单协调者);`consumer_generation` 是信箱已消费到第几代的标记
- **tasks.status 六态**:`pending`(建了没派)→ `ready`(依赖满足)→ `dispatched`(已派)→ `completed/failed`(结算)→ `blocked`(被 gate 卡住)
- **dispatch_contexts**:`failure_count` 3 次失败 → `circuit_broken`;`depth` 记录嵌套层级(默认 1,防 worker 派子 worker);`launch_token_hash` 防伪造派发
- **worker_dispatches.state**:`starting→ready→succeeded/failed`,另有 `stopping/stop_unknown/stopped/abandoned` 完整停机状态机
- **worker_terminal_resources**:终端所有权(+转移历史 `prior_owner_dispatch_ids`),`release_state` 逐级变化(requested→releasing→released),`archive_*` 留档输出供 `worker-read`
- **question_threads**:worker 的 `ask`/协调者的 `reply` 形成**阻塞问答线程**,`pending→answered→closed`

---

## 5. Worker 通信机制:ask / heartbeat / escalation

**`send` 不是 worker 专属**——它只是「往信箱写一条消息」的通用命令,协调者与 worker 都能用,但惯用方向不同:协调者主要用 `worker-start`/`dispatch`(派活)、`reply`(回 worker 问题)、`terminal send`(实时指令),真正用 send 的场景是广播 status 或给特定 worker 单独指导(`--to dispatch:<id>`);worker 则几乎只能靠 `send` 上报(`--type worker_done`/`heartbeat` 没有替代命令)。边界:worker_done/heartbeat 是 **Dispatch 级信号**,必须由被派发的 worker 发(带 task-id/dispatch-id,校验 assignee 匹配),且不能发 group;协调者发 status 类消息无此限制。

除了 `send` 状态消息,worker 还有三类上行信号(它们都有独立的表/字段支持):

### 5.1 ask(worker 向协调者问阻塞性问题)

```bash
# worker 端(默认投到所属 Dispatch 的 Run)
orca orchestration ask --question "接口返回结构是 X 还是 Y?" --options "X,Y" --timeout-ms 600000

# 协调者端
orca orchestration reply --id <msg_id> --body "用 X"
```

**实现**:`ask` 创建 `question` 类型消息 + `question_threads` 行(status=`pending`),阻塞等待回复;超时/断连后 question **保持 pending**,协调者可 `reply --id` 或 worker `ask --resume <msg_id>` 恢复(resume 是幂等的,不新建问题)。

```sql
-- question_threads:ask/reply 的线程状态
status: pending → answered → closed
answer_message_id: reply 生成的消息 id
```

### 5.2 heartbeat(报活)

```bash
orca orchestration send --type heartbeat --subject "alive"   --payload '{"taskId":"...","dispatchId":"...","phase":"implementing"}'
```

**实现**:更新 `dispatch_contexts.last_heartbeat_at`。注意 `worker_done`/`heartbeat` 是 **Dispatch 级信号**——省略 `--to` 即用所属 Run,不能发 group。协调者看到 heartbeat = worker 还活着,**不等于干完了**。

### 5.3 escalation(升级求助)

```bash
orca orchestration send --type escalation --subject "阻塞" --body "需要你介入..." --task-id X --dispatch-id Y
```

**实现**:与 question 类似,但语义更强——worker 判断"自己干不动了 / 需要协调者接管"。协调者收到后通常在 worker 继续等待时介入处理,或决定重派。

### 5.4 worker_done 的唯一性

worker 只能发**一次** `worker_done`(带 `--outcome succeeded|failed`),发完必须结束该轮,"idle 在 agent 提示符"。协调者收到后三选一:

1. **复用**:同一 worker 立即接下一个 Task → `worker-start --task <next> --terminal <handle>`
2. **保留**:用户要求调试 → `worker-retain --dispatch <id>`
3. **释放**:默认 → `worker-release --dispatch <id>`(关掉该 dispatch 专属的终端,输出已存档可 `worker-read` 再读)

### 5.5 消息流向的不对称性

```
协调者 → worker:  send --to dispatch:<id> 或 term_xxx(定向信箱,只有接收方拉得到)
worker → 协调者:  reply / worker_done(默认投 Run 信箱,to_handle = run:run_xxx)
```

worker 的 reply **不需要也不能**自己指定 `--to`——它"回"的是原消息归属的 Run 信箱(`to_handle=run:run_xxx`),协调者 check 该 Run 即拿到;`thread_id` 把问答串成线。数据库实测:`msg_186ff9007853 | to=run:run_8f607654de5c | thread=msg_37cdf8a4bd67`。

---

## 6. 核心概念在代码里的体现:Run / Task / Dispatch

```
runs 表(run_xxx) ── run_id 是所有表的 FK
 ├── tasks(task_1): spec / status / deps
 └── dispatch_contexts + worker_dispatches:
       task_id + terminal handle + 状态
       → 派发时把 preamble(任务书)注入 Agent 会话
       → Agent 做完发 worker_done(task_id + dispatch_id)
       → 自动: task completed / dispatch settled
```

三个概念的生命周期差异:

|       | Run           | Task     | Dispatch    |
| ----- | ------------- | -------- | ----------- |
| 生命周期  | 长(一个会话一个)     | 中(可能重派)  | 短(一次执行)     |
| 作用    | 命名空间 + 协调者收件箱 | 描述"做什么"  | 这次派谁干、干得怎样  |
| 不做什么  | 不调度任何东西       | 无执行者     | ——          |

### 6.1 Run 与终端的绑定机制(隐式但关键)

```sql
-- runs 表(创建时的默认行为):
coordinator_handle   = 创建者终端 handle
coordinator_pane_key = 创建者 pane
consumer_generation  = 0(还没消费过)
```

- **run-create 时自动绑定**:创建者终端(或 `--from` 指定终端)成为该 Run 的唯一协调者,写入 `runs.coordinator_handle`
- **例外(未绑定)**:兼容/恢复路径产生的 Run(如 `legacy_adoptions`)可能 `coordinator_handle=NULL`,此时 `check` 没有默认目标,必须显式 `--run`
- **check 的默认目标 = 当前终端绑定的 Run**(不传 `--run` 时);绑定可通过:
  - `run-create`(创建即绑定)
  - `run-use --id <run_id>`(显式绑定/切换)
  - worker 被 dispatch 时自动绑定到所属 Run(注入的 preamble 指定)
- **多编排者场景**:A 终端的 Run 和 B 终端的 Run 是两个独立信箱;即使不传 `--run`,各自 check 各自绑定的 Run,互不干扰
- **什么时候要显式 `--run`**:从非绑定终端调用、脚本化/幂等调用、run-use 后想确认、上下文不明时——显式声明避免依赖隐式绑定
- **share 一个 Run 不是并行**:旧协调者会被 fence(只读),新协调者 `run-use --takeover-legacy` 接管后才有消费权——设计上"单协调者模型"

worker_done 校验逻辑:

```js
// 校验 dispatch.assignee == from_handle 且 taskId/dispatchId 匹配
if (dispatch.assignee === from && dispatch.taskId === taskId) {
  task.status = outcome === 'succeeded' ? 'completed' : 'failed'
  dispatch.status = 'settled'
  // 3 次失败 → task.status = 'failed'(熔断)
}
```

### 6.2 派活的两种方式:worker-start vs dispatch --inject

|     | `worker-start`(推荐)                          | `dispatch --task X --to <h> --inject`(低层) |
| --- | ------------------------------------------- | ----------------------------------------- |
| 组成  | worktree/终端 + 派发 + 注入 合一步                   | 仅派发(终端要自己先建)                              |
| 返回  | 完整 receipt(ready/effects/residualResources) | 仅派发状态                                     |
| 监控  | 有(worker-show/read/stop/release 全套)         | 无(operator 启动的终端视为 unsupervised)          |
| 何时用 | 默认                                          | 自定义 argv(如 codex 定制 model/effort)、特殊拓扑    |
| 注入  | 自动注入 preamble                               | `--inject` 才注入                            |

```
worker-start 的调用形态(合成代码):
  worktree 复用/新建 + terminal create + dispatch --inject
  → 返回 { worktree, terminalHandle, dispatchId, effects }
```

**注意**:`dispatch --inject` 派发的 worker 是 **unsupervised** 的——`worker-stop`/`worker-release` 不会关闭它的进程(资源仍归 operator);要可管理必须有 `worker_dispatches` 行(worker-start 才有)。

### 6.3 嵌套深度限制

```
depth 字段:root 协调者的 worker = 1,worker 的 worker = 2 ...
默认 depth=1 → worker 再派子 worker 会报 nested_worker_depth_exceeded
可在 Settings → Orchestration → Nested worker depth 调大
计数从"发出命令的终端"算起,新建 Run 不会重置
```

### 6.4 任务依赖与 dispatch 前置校验

```
task-create(T3, deps=[T1,T2])
   ├─ 依赖不全 → T3.status='blocked'(不占 Agent;此时 dispatch 会报错拒绝)
   ├─ T1 完成(worker_done) → 事件触发检查:依赖不全 → 仍 blocked
   ├─ T2 完成(worker_done) → 事件触发检查:依赖全满足 → T3.status='ready'  ← 自动!
   └─ 协调者 dispatch T3 → 成功 → Agent 开跑
```

- **dispatch 前置校验**:任务必须 `ready` 才能派(`ERROR: Task ... is not ready ... ready tasks can be dispatched`);依赖未满足时派发直接被拒绝,任务静默停留在 `pending/blocked`
- **ready 是"自动变"但"不自动派"**:依赖完成事件驱动 blocked→ready;但开跑仍需协调者 dispatch(资源/优先级由协调者决定)
- **协调者节奏**:`task-list --ready` 拿新就绪任务 → dispatch → 等 worker_done → 再派下一波;`ready` 任务已有活跃 dispatch 时会拒绝重复派发

---

## 7. 编排与 git 的协作

编排层记账,Agent 干活,git 只管历史:

```
编排层(Orca)                              git 层(Worker)
─────────                                 ─────────────
run-create/task-create                     无 git 动作
worker-start → worktree 里开终端 ─────────▶ git worktree add(或复用)
dispatch → 注入任务书 ────────────────────▶ Agent 在私有 worktree 里:
                                              cd [[ORCA_RICH_MD:d48ea122555189d6ddf1904b6193e395:inline-html:%3Cworktree%3E]] && git add/commit
worker_done(摘要 + files-modified 元数据)    ↑ 提交发生在 Agent 那边
worker-release → 收终端
git merge <分支> / push+PR ────────────────▶ master 指针汇合
```

边界:

- 编排消息**不写进 git**(worker_done 的 files-modified 只是元数据)
- 合并是标准 git 操作,编排不参与
- worktree 的 Orca 元数据(comment/状态)与 git 分支正交,通过"worktree 绑定 repo + 私有 HEAD"挂钩

---

## 8. 附录:命令速查表

```bash
# 发现
orca worktree list --json
orca terminal list --json
orca terminal list --worktree <selector> --json
orca terminal read --terminal <handle> --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json

# 实时管道
orca terminal send --terminal <handle> --text "..." --enter --json

# 编排管道
orca orchestration run-create --objective "..." --json
orca orchestration task-create --spec "..." [--deps '["id"]'] --json
orca orchestration worker-start --task <id> --worktree current --agent codex [--model ...] --json
orca orchestration dispatch --task <id> --to <handle> --inject --json
orca orchestration send --run <run> --to <addr> --subject ... --body ... [--type ...] --json
orca orchestration check --run <run> --format --json
orca orchestration check --ack <delivery_id> --json
orca orchestration reply --id <msg_id> --body "..." --json
orca orchestration worker-release --dispatch <id> --json
orca orchestration worker-read --dispatch <id> --limit 50 --json
orca orchestration gate-create --task <id> --question "..." --json
```

### 关键代码位置

| 机制            | 位置                                                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------------------------------ |
| PTY 创建/写入     | `resources/node_modules/node-pty/`(v1.1.0);主进程 `pty.spawn({cols,rows,cwd,command})`                                |
| PTY IPC 通道    | app.asar:`"pty.spawn"/"pty.data"/"pty.replay"/"pty.serialize"/...`                                                 |
| SQLite 状态库    | `AppData/Roaming/orca/orchestration.db`;Node `node:sqlite`(`DatabaseSync`);`INSERT INTO messages` 等                |
| 编排 CLI 处理器    | app.asar:`out/cli/handlers/orchestration/*.js`                                                                     |
| worktree 私有状态 | 主仓库 `.git/worktrees/[[ORCA_RICH_MD:d48ea122555189d6ddf1904b6193e395:inline-html:%3Cid%3E]]/{HEAD,index,logs,refs}` |
