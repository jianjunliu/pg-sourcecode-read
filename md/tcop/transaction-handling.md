# exec_simple_query 的事务处理详解：`start_xact_command` / `finish_xact_command`

> 学习笔记（续 `exec-simple-query.md`）。本文只追事务主线，权威依据仍是源码，
> 重点文件：`src/backend/tcop/postgres.c`、`src/backend/access/transam/xact.c`。

## 0. 两个 wrapper 与真正的工作者

`exec_simple_query()` 里所有事务动作都通过两个极薄的静态 wrapper 完成：

| wrapper | 位置 | 语义 |
|---|---|---|
| `start_xact_command()` | `postgres.c:2747` | 确保「有一个已开始的事务命令」 |
| `finish_xact_command()` | `postgres.c:2775` | 结束当前事务命令（提交 / 只递增命令号 / 回滚） |

在 `exec_simple_query` 中的调用点：

- `postgres.c:1047` — 解析前先开启事务命令（此时 blockState = `TBLOCK_DEFAULT` → `TBLOCK_STARTED`）。
- `postgres.c:1143` — 循环内、每条语句「确保处于事务命令中」（幂等，通常 no-op）。
- `postgres.c:1299` — **最后一条**语句收尾 → `finish_xact_command()`。
- `postgres.c:1307` — 非最后一条、且是事务控制语句（`TransactionStmt`）→ `finish_xact_command()`。
- `postgres.c:1350` — 循环之后兜底（仅当 `parsetree_list` 为空才有实际作用）。

两者只做「守卫 + 时间统计 + 调用 xact.c」；真正的状态机实现全部在
`access/transam/xact.c`。理解它们必须理解 xact.c 的**两层状态机**。

## 1. 前置知识：xact.c 的两层状态机

xact.c 里每个事务都有一个 `TransactionStateData`（栈结构，支持子事务）：

```c
typedef struct TransactionStateData
{
    FullTransactionId fullTransactionId;  /* 我的 FullTransactionId */
    SubTransactionId  subTransactionId;   /* 我的子事务 ID */
    char             *name;               /* 保存点名字 */
    int               savepointLevel;
    TransState        state;              /* 低层状态：服务器视角 */
    TBlockState       blockState;         /* 高层状态：客户端查询视角 */
    int               nestingLevel;       /* 事务嵌套深度 */
    int               gucNestLevel;       /* GUC 嵌套深度 */
    MemoryContext     curTransactionContext;  /* 本事务生命周期内存 */
    ResourceOwner     curTransactionOwner;
    ...
    struct TransactionStateData *parent;  /* 子事务用：指向父 */
    struct TransactionStateData *child;
    struct TransactionStateData *next;
} TransactionStateData;
```

- 顶层节点是静态的 `TopTransactionStateData`，`CurrentTransactionState` 指向栈顶。
- **`state`（`TransState`，`xact.c:141`）** 描述事务「本身」处于哪个阶段：

  ```
  TRANS_DEFAULT → TRANS_START → TRANS_INPROGRESS → TRANS_COMMIT
                                                  → TRANS_ABORT
                                                  → TRANS_PREPARE
  ```

- **`blockState`（`TBlockState`，`xact.c:157`）** 描述「客户端命令会话」的块级状态，
  是 wrapper 分派的关键。完整枚举：

  | 类别 | 取值 |
  |---|---|
  | 不在事务块 | `TBLOCK_DEFAULT`(空闲)、`TBLOCK_STARTED`(单语句事务) |
  | 事务块 | `TBLOCK_BEGIN`、`TBLOCK_INPROGRESS`、`TBLOCK_IMPLICIT_INPROGRESS`、`TBLOCK_PARALLEL_INPROGRESS`、`TBLOCK_END`、`TBLOCK_ABORT`、`TBLOCK_ABORT_END`、`TBLOCK_ABORT_PENDING`、`TBLOCK_PREPARE` |
  | 子事务 | `TBLOCK_SUBBEGIN`、`TBLOCK_SUBINPROGRESS`、`TBLOCK_SUBRELEASE`、`TBLOCK_SUBCOMMIT`、`TBLOCK_SUBABORT`、`TBLOCK_SUBABORT_END`、`TBLOCK_SUBABORT_PENDING`、`TBLOCK_SUBRESTART`、`TBLOCK_SUBABORT_RESTART` |

> 关键记忆点：`state` 回答「事务活没活着」，`blockState` 回答「用户在不在显式
> 事务块里 / 是否处于隐式块 / 是否在等待 ROLLBACK」。wrapper 的每个 switch
> 都是对 `blockState` 的分派，而具体的 `StartTransaction`/`CommitTransaction`
> 才是对 `state` 的驱动。

## 2. `start_xact_command()` 详解

```c
static void
start_xact_command(void)
{
    if (!xact_started)
    {
        StartTransactionCommand();
        xact_started = true;
    }

    /*
     * Start statement timeout if necessary.  ... 不会重置已经在跑的计时器 ...
     */
    enable_statement_timeout();

    /* Start timeout for checking if the client has gone away if necessary. */
    if (client_connection_check_interval > 0 &&
        IsUnderPostmaster && MyProcPort &&
        !get_timeout_active(CLIENT_CONNECTION_CHECK_TIMEOUT))
        enable_timeout_after(CLIENT_CONNECTION_CHECK_TIMEOUT,
                             client_connection_check_interval);
}
```

三个要点：

1. **幂等守卫 `xact_started`**（`postgres.c:135` 定义，初值 `false`）。
   只有未开启时才调用 `StartTransactionCommand()`；重复调用是 no-op。
   这正是 exec_simple_query 在 1047 和 1143 两次调用不会双重开事务的原因。
   该标志在 `finish_xact_command()` 置回 `false`，在错误恢复路径
   （`postgres.c:4390`）也置 `false`。
2. **语句超时**：`enable_statement_timeout()` 启用 `statement_timeout`。
   注释特别说明：**不重置已在运行的计时器**——因为在 parse/bind/execute
   多次调用 `start_xact_command()` 而中间没有 `finish` 时，避免反复重置的
   计时开销。
3. **客户端断连检测**：按 `client_connection_check_interval` 设置一个
   `CLIENT_CONNECTION_CHECK_TIMEOUT` 定时器，用于探测客户端是否掉线。

### 2.1 深入：`StartTransactionCommand()`（`xact.c:2937`）

对 `blockState` 分派：

| blockState | 动作 |
|---|---|
| `TBLOCK_DEFAULT`（空闲） | **`StartTransaction()`**，然后 `blockState = TBLOCK_STARTED` |
| `TBLOCK_INPROGRESS` / `TBLOCK_IMPLICIT_INPROGRESS` / `TBLOCK_SUBINPROGRESS` | 什么都不做（已在新的事务/子事务中；命令号递增由上一条 `CommitTransactionCommand` 负责） |
| `TBLOCK_ABORT` / `TBLOCK_SUBABORT` | 什么都不做（保持中止态，等 ROLLBACK） |
| 其余（`STARTED`/`BEGIN`/`END`/`SUB*`/`PREPARE` 等） | `elog(ERROR, "StartTransactionCommand: unexpected state ...")` |

结尾：

```c
Assert(CurTransactionContext != NULL);
MemoryContextSwitchTo(CurTransactionContext);
```

即**返回前一定把内存上下文切到事务上下文**（`StartTransaction` 内已切过一次，
其他分支在这里补切）。这就是 `exec_simple_query` 里 1047 之后要
`MemoryContextSwitchTo(MessageContext)` 再切回来的原因。

### 2.2 深入：`StartTransaction()`（`xact.c:1993`）

把 `state` 从 `TRANS_DEFAULT` 推到 `TRANS_INPROGRESS`，主要步骤：

1. 栈复位：`s = &TopTransactionStateData; CurrentTransactionState = s;`
   `Assert(s->state == TRANS_DEFAULT);`
2. `s->state = TRANS_START;`（进入过渡态，此期间不能再失败）
3. 事务采样：`xact_is_sampled = ... log_xact_sample_rate ...`
4. 初始化嵌套与 child xid 数组：`nestingLevel = 1; gucNestLevel = 1; childXids = NULL; ...`
5. 保存当前用户与安全上下文：`GetUserIdAndSecContext(&s->prevUser, &s->prevSecContext);`
6. **重置 xact 级变量**：
   - 恢复中（`RecoveryInProgress()`）→ `startedInRecovery = true; XactReadOnly = true;`
   - 否则 `XactReadOnly = DefaultXactReadOnly;`
   - `XactDeferrable / XactIsoLevel` 复位，`forceSyncCommit = false; MyXactFlags = 0;`
7. 命令计数器复位：`subTransactionId = TopSubTransactionId;`
   `currentCommandId = FirstCommandId; currentCommandIdUsed = false;`
8. **先建资源管理设施**：
   - `AtStart_Memory()`（`xact.c:1152`）：创建 `TopTransactionContext`，
     令 `CurTransactionContext = TopTransactionContext`，并 `MemoryContextSwitchTo` 它。
     首次还会创建抗 OOM 的 `TransactionAbortContext`（32KB 预留，
     供 `AbortTransaction` 在内存耗尽时也能工作）。
   - `AtStart_ResourceOwner()`（`xact.c:1199`）：创建
     `TopTransactionResourceOwner`，并令 `CurTransactionResourceOwner = CurrentResourceOwner = 它`。
9. 分配**虚拟事务 ID**：
   ```c
   vxid.backendId = MyBackendId;
   vxid.localTransactionId = GetNextLocalTransactionId();
   VirtualXactLockTableInsert(vxid);      /* 先加锁再公布 */
   MyProc->lxid = vxid.localTransactionId;/* 在 proc array 中公布 */
   ```
   注意此时还没有真正的 `FullTransactionId`（`s->fullTransactionId = InvalidFullTransactionId`），
   直到第一次写操作调用 `AssignTransactionId()` 才真正分配——这是 PG 的惰性 XID 分配。
10. 时间戳：`xactStartTimestamp` 通常复用 `stmtStartTimestamp`（省一次
    `GetCurrentTimestamp()`）；仅在过程内非原子 SPI 上下文才重新取。
11. 初始化其他子系统：`AtStart_GUC(); AtStart_Cache(); AfterTriggerBeginXact();`
12. `s->state = TRANS_INPROGRESS;` 并 `ShowTransactionState("StartTransaction")`
    （仅在 `--enable-cassert` 或 `TRACE` 下才打印）。

> 与 exec_simple_query 的呼应：`AtStart_Memory` 创建的 `TopTransactionContext`
> 正是「事务生命周期」内存；而 `exec_simple_query` 把解析/计划树放在
> `MessageContext`/`per-parsetree_context`，就是为了让它们活得比这个
> 「每命令事务」更久（否则 COMMIT/ROLLBACK 会把它们清掉）。

## 3. `finish_xact_command()` 详解

```c
static void
finish_xact_command(void)
{
    /* cancel active statement timeout after each command */
    disable_statement_timeout();

    if (xact_started)
    {
        CommitTransactionCommand();

#ifdef MEMORY_CONTEXT_CHECKING
        MemoryContextCheck(TopMemoryContext);   /* 检查提交时未释放的 context */
#endif
#ifdef SHOW_MEMORY_STATS
        MemoryContextStats(TopMemoryContext);    /* 泄漏跟踪 */
#endif

        xact_started = false;
    }
}
```

要点：

1. **先取消语句超时**：`disable_statement_timeout()`，与 `start` 中的
   `enable_statement_timeout()` 对称，保证超时按「每条命令」独立计算。
2. `xact_started` 守卫：只有真正开着事务命令才 `CommitTransactionCommand()`。
3. 在 `MEMORY_CONTEXT_CHECKING` / `SHOW_MEMORY_STATS` 调试构建下，
   提交后检查顶层内存上下文，便于查内存泄漏。
4. 置 `xact_started = false`，为下一条命令重新开始做准备。

### 3.1 深入：`CommitTransactionCommand()`（`xact.c:3034`）

函数开头保存事务特征以便 `AND CHAIN` 复用：

```c
SavedTransactionCharacteristics savetc;
SaveTransactionCharacteristics(&savetc);   /* iso level / read only / deferrable */
```

然后按 `blockState` 分派。**这是全文最关键的分派表**：

| blockState | 动作 | 结果状态 |
|---|---|---|
| `TBLOCK_DEFAULT` / `TBLOCK_PARALLEL_INPROGRESS` | `elog(FATAL)`（不应出现） | — |
| `TBLOCK_STARTED`（单语句事务） | **`CommitTransaction()`** | → `TBLOCK_DEFAULT` |
| `TBLOCK_BEGIN`（刚执行完 BEGIN） | 仅改状态（BEGIN 无 DB 改动） | → `TBLOCK_INPROGRESS` |
| `TBLOCK_INPROGRESS` / `TBLOCK_IMPLICIT_INPROGRESS` / `TBLOCK_SUBINPROGRESS` | **`CommandCounterIncrement()`**（不结束事务） | 不变 |
| `TBLOCK_END`（收到 COMMIT） | `CommitTransaction()`；若 `s->chain` 再 `StartTransaction()` | → `TBLOCK_DEFAULT`（或链式 `INPROGRESS`） |
| `TBLOCK_ABORT` / `TBLOCK_SUBABORT` | 不动（等 ROLLBACK） | 不变 |
| `TBLOCK_ABORT_END`（中止后收到 ROLLBACK） | **`CleanupTransaction()`** | → `TBLOCK_DEFAULT` |
| `TBLOCK_ABORT_PENDING`（活事务收到 ROLLBACK） | **`AbortTransaction()` + `CleanupTransaction()`** | → `TBLOCK_DEFAULT` |
| `TBLOCK_PREPARE`（PREPARE TRANSACTION） | `PrepareTransaction()` | → `TBLOCK_DEFAULT` |
| `TBLOCK_SUBBEGIN` | `StartSubTransaction()` | → `TBLOCK_SUBINPROGRESS` |
| `TBLOCK_SUBRELEASE` | 循环 `CommitSubTransaction()` 直到回到 INPROGRESS 层级 | — |
| `TBLOCK_SUBCOMMIT` | 循环 `CommitSubTransaction()`；顶层再 `CommitTransaction()`/`PrepareTransaction()` | — |
| `TBLOCK_SUBABORT_END` | `CleanupSubTransaction()` + 递归 `CommitTransactionCommand()` | — |
| `TBLOCK_SUBABORT_PENDING` | `AbortSubTransaction()` + `CleanupSubTransaction()` + 递归 | — |
| `TBLOCK_SUBRESTART` / `TBLOCK_SUBABORT_RESTART` | 中止/清理子事务后 `DefineSavepoint` + `StartSubTransaction`（ROLLBACK TO 语义） | — |

> **与 exec_simple_query 的对应关系**——三种收尾策略本质就是这张表的三行：
>
> - 单条语句 / 最后一条语句：blockState 是 `TBLOCK_STARTED`（或隐式块结束时
>   被 `EndImplicitTransactionBlock` 改回 `STARTED`）→ **走 `CommitTransaction()`，
>   真正结束事务**。
> - 多语句中间的非事务控制语句：blockState 是 `TBLOCK_INPROGRESS`/
>   `TBLOCK_IMPLICIT_INPROGRESS` → **只 `CommandCounterIncrement()`，事务继续**。
> - 事务控制语句（`BEGIN`/`COMMIT`/`ROLLBACK`）：blockState 是 `TBLOCK_BEGIN`/
>   `TBLOCK_END`/`TBLOCK_ABORT_PENDING` 等 → 进入/结束事务块。
>
> 注意 `exec_simple_query` 循环里这两条路径是**互斥**的：非最后一条、
> 非事务控制语句走 `else` 分支，**直接** `CommandCounterIncrement()`
> （`postgres.c:1322`），根本不会调用 `finish_xact_command()`；
> 而调用 `finish_xact_command()` 的分支要么先把隐式块 `EndImplicit`
> 改回 `TBLOCK_STARTED`（最后一条），要么是事务控制语句。因此
> `CommitTransactionCommand` 中「`TBLOCK_INPROGRESS` → 只递增命令号」
> 这一行，在 `exec_simple_query` 语境下通常由别的调用者
> （如扩展协议）触发，而非本函数的收尾分支。

### 3.2 深入：`CommitTransaction()`（`xact.c:2152`）——真正的提交

提交分三大段，顺序经过精心安排（注释明确说明了排序理由）：

**段一：pre-commit，仍可能调用用户代码（可抛错 → 转入中止路径）**

```c
/* 反复循环：发延迟触发器 → 关 portal（holdable 转 static portal）
   关 portal 又可能排队新触发器，故循环到无事可做 */
for (;;)
{
    AfterTriggerFireDeferred();
    if (!PreCommit_Portals(false))
        break;
}

CallXactCallbacks(XACT_EVENT_PRE_COMMIT);   /* 插件回调 */
if (IsInParallelMode())
    AtEOXact_Parallel(true);                /* 清理并行 worker */
AfterTriggerEndXact(true);
PreCommit_on_commit_actions();              /* ON COMMIT 动作 */
smgrDoPendingSyncs(true, is_parallel_worker);/* 同步未走 WAL 的新建文件 */
AtEOXact_LargeObject(true);
PreCommit_Notify();                          /* 把 NOTIFY 塞进队列 */
if (!is_parallel_worker)
    PreCommit_CheckForSerializationFailure();/* 可串行化冲突检测，尽量晚做 */
```

之后 `HOLD_INTERRUPTS()` 禁止 cancel/die 中断，进入不可中断的提交核心；
并 `AtEOXact_RelationMap(true, ...)` 提交关系映射改动。

**段二：durable commit 与 proc array 注销**

```c
s->state = TRANS_COMMIT;
s->parallelModeLevel = 0;

if (!is_parallel_worker)
    latestXid = RecordTransactionCommit();  /* 在 pg_xact 标记已提交 + 写 WAL */
else
{
    latestXid = InvalidTransactionId;       /* 并行 worker 不自己标记 */
    ParallelWorkerReportLastRecEnd(XactLastRecEnd);
}

TRACE_POSTGRESQL_TRANSACTION_COMMIT(MyProc->lxid);
ProcArrayEndTransaction(MyProc, latestXid); /* 必须在放锁之前、记录提交之后 */
```

**段三：post-commit 清理（此段再抛错也无法回滚）**

顺序：先释放对其他后端可见的资源（文件、buffer pin），再放锁，
最后才是后端本地资源：

```c
CallXactCallbacks(XACT_EVENT_COMMIT);
ResourceOwnerRelease(TopTransactionResourceOwner, RESOURCE_RELEASE_BEFORE_LOCKS, ...);
AtEOXact_Buffers(true);        /* 确保 buffer pin 全释放 */
AtEOXact_RelationCache(true);  /* 清 relcache */
AtEOXact_Inval(true);          /* 让目录改动对所有后端可见（放锁前！） */
AtEOXact_MultiXact();
ResourceOwnerRelease(..., RESOURCE_RELEASE_LOCKS, ...);
ResourceOwnerRelease(..., RESOURCE_RELEASE_AFTER_LOCKS, ...);
smgrDoPendingDeletes(true);    /* 删文件放在放锁之后 */
AtCommit_Notify();             /* 发通知信号 */

/* 以下纯属后端内部清理 */
AtEOXact_GUC(true, 1); AtEOXact_SPI(true); AtEOXact_Enum();
AtEOXact_on_commit_actions(true); AtEOXact_Namespace(true, ...);
AtEOXact_SMgr(); AtEOXact_Files(true); AtEOXact_ComboCid();
AtEOXact_HashTables(true); AtEOXact_PgStat(true, ...); AtEOXact_Snapshot(true, false);
AtEOXact_ApplyLauncher(true); AtEOXact_LogicalRepWorkers(true);
pgstat_report_xact_timestamp(0);

CurrentResourceOwner = NULL;
ResourceOwnerDelete(TopTransactionResourceOwner);
s->curTransactionOwner = NULL;
CurTransactionResourceOwner = NULL;
TopTransactionResourceOwner = NULL;

AtCommit_Memory();             /* MemoryContextDelete(TopTransactionContext) */
s->fullTransactionId = InvalidFullTransactionId;
... 清空 childXids / nestingLevel ...
s->state = TRANS_DEFAULT;
RESUME_INTERRUPTS();
```

`AtCommit_Memory()`（`xact.c:1556`）切回 `TopMemoryContext` 并删除
`TopTransactionContext`——事务内存到此彻底回收。

## 4. 错误路径：`AbortCurrentTransaction()`（`xact.c:3305`）

**注意：它不是由 `finish_xact_command()` 调用的**，而是
`PostgresMain` 的顶层错误恢复块调用（`postgres.c:4352`）：

```c
EmitErrorReport();
debug_query_string = NULL;
AbortCurrentTransaction();     /* 从错误中恢复事务状态 */
...
xact_started = false;          /* postgres.c:4390 —— wrapper 的守卫同步复位 */
```

按 `blockState` 分派（节选与 exec_simple_query 相关的行）：

| blockState | 动作 |
|---|---|
| `TBLOCK_DEFAULT` 且 `state == TRANS_DEFAULT` | 空闲，无事可做 |
| `TBLOCK_DEFAULT` 但 `state` 非默认 | 事务启动中途出错：`AbortTransaction()` + `CleanupTransaction()` |
| `TBLOCK_STARTED` / `TBLOCK_IMPLICIT_INPROGRESS` | `AbortTransaction()` + `CleanupTransaction()`，回 `TBLOCK_DEFAULT`（隐式块按单语句处理） |
| `TBLOCK_BEGIN` | 同上，视 BEGIN 失败，回 `TBLOCK_DEFAULT` |
| `TBLOCK_INPROGRESS` / `TBLOCK_PARALLEL_INPROGRESS` | `AbortTransaction()`，进 `TBLOCK_ABORT`（等用户 ROLLBACK；Cleanup 延后） |
| `TBLOCK_END` | 提交途中失败：Abort + Cleanup，回 `TBLOCK_DEFAULT` |
| `TBLOCK_ABORT` / `TBLOCK_SUBABORT` | 保持（已经在中止态，又失败一次） |
| `TBLOCK_ABORT_END` | `CleanupTransaction()`（已 Abort 过），回 `TBLOCK_DEFAULT` |
| `TBLOCK_ABORT_PENDING` | `AbortTransaction()` + `CleanupTransaction()` |
| `TBLOCK_PREPARE` | PREPARE 失败：Abort + Cleanup |
| 各 `TBLOCK_SUB*` | 中止/清理当前子事务后**递归** `AbortCurrentTransaction()` 处理父层 |

**`AbortTransaction()`（`xact.c:2697`）** 的骨架：

1. `HOLD_INTERRUPTS()`；`AtAbort_Memory()`（切到预留的 `TransactionAbortContext`）、
   `AtAbort_ResourceOwner()`。
2. 尽快释放 LW 锁与 buffer 锁：`LWLockReleaseAll()`、`UnlockBuffers()`、
   `XLogResetInsertion()`、`ConditionVariableCancelSleep()`、`LockErrorCleanup()`、
   `reschedule_timeouts()`；重新放行信号 `sigprocmask(SIG_SETMASK, &UnBlockSig, ...)`。
3. `s->state = TRANS_ABORT;`，恢复用户/安全上下文 `SetUserIdAndSecContext(...)`。
4. 重置 REINDEX、逻辑流、快照导出状态；退出并行模式 `AtEOXact_Parallel(false)`。
5. 中止清理：`AfterTriggerEndXact(false)`、`AtAbort_Portals()`、
   `smgrDoPendingSyncs(false)`、`AtEOXact_LargeObject(false)`、`AtAbort_Notify()`、
   `AtEOXact_RelationMap(false)`、`AtAbort_Twophase()`。
6. `RecordTransactionAbort(false)` 在 pg_xact 标记中止；`ProcArrayEndTransaction()`。
7. post-abort 清理（与 commit 对称，`RESOURCE_RELEASE_*` 三连 +
   各 `AtEOXact_*`），**但状态停留在 `TRANS_ABORT`**，等 `CleanupTransaction()`。

**`CleanupTransaction()`（`xact.c:2888`）**：才真正释放 portal、快照、
ResourceOwner、事务内存（`AtCleanup_Memory`），把状态清空并置
`s->state = TRANS_DEFAULT`。也就是说 **abort 分两步：`AbortTransaction`
（做中止动作，state=ABORT）+ `CleanupTransaction`（释放资源，state=DEFAULT）**，
与 commit 的「`CommitTransaction` 一步到底」不同。

## 5. 隐式事务块：`BeginImplicitTransactionBlock` / `EndImplicitTransactionBlock`

这两个函数（`xact.c:4172` / `:4197`）被 `exec_simple_query` 直接调用，
是「一个 Simple Query 字符串多条语句 = 一个事务」的实现核心：

```c
void BeginImplicitTransactionBlock(void)
{
    TransactionState s = CurrentTransactionState;
    if (s->blockState == TBLOCK_STARTED)
        s->blockState = TBLOCK_IMPLICIT_INPROGRESS;
}

void EndImplicitTransactionBlock(void)
{
    TransactionState s = CurrentTransactionState;
    if (s->blockState == TBLOCK_IMPLICIT_INPROGRESS)
        s->blockState = TBLOCK_STARTED;
}
```

- 注释明确：这俩**直接从 postgres.c 主循环调用，不在 Portal 内**，
  所以只改 `blockState`，不需要 caller 做 Commit/Start 仪式。
- 机制：`exec_simple_query` 一开始 `use_implicit_block = (语句数 > 1)`；
  循环内每条语句前 `BeginImplicitTransactionBlock()`，把状态从
  `TBLOCK_STARTED`（会被当作单语句提交）改成 `TBLOCK_IMPLICIT_INPROGRESS`
  （在 `CommitTransactionCommand` 里只会 `CommandCounterIncrement`，事务不结束）。
  最后一条语句收尾时先 `EndImplicitTransactionBlock()` 改回 `TBLOCK_STARTED`，
  让随后的 `finish_xact_command()` 走 `CommitTransaction()` 真正提交。
- 两个函数对其他 `blockState` 一律宽容（no-op），注释说明这是为了方便 caller。

因此 `TBLOCK_IMPLICIT_INPROGRESS` 与 `TBLOCK_INPROGRESS` 在
`StartTransactionCommand`/`CommitTransactionCommand`/`AbortCurrentTransaction`
里的行为完全相同——区别只在于「进入方式」和「abort 时按单语句整体处理」。

## 6. `CommandCounterIncrement()`（`xact.c:1078`）

exec_simple_query 在循环的 `else` 分支显式调用它；`CommitTransactionCommand`
在 `TBLOCK_INPROGRESS` 情形也调用它。实现：

```c
if (currentCommandIdUsed)
{
    if (IsInParallelMode() || IsParallelWorker())
        elog(ERROR, "cannot start commands during a parallel operation");

    currentCommandId += 1;
    if (currentCommandId == InvalidCommandId)          /* 溢出保护 */
    {
        currentCommandId -= 1;
        ereport(ERROR, ... "cannot have more than 2^32-2 commands in a transaction");
    }
    currentCommandIdUsed = false;

    SnapshotSetCommandId(currentCommandId);   /* 把新命令号播进静态快照 */
    AtCCI_LocalCache();                        /* 让本次改动对本后端可见 */
}
```

三个精妙点：

1. **按需递增**：只有当 `currentCommandIdUsed == true`（本命令确实写过东西、
   需要区分可见性）时才递增。纯只读命令不递增，延缓命令号溢出。
2. **溢出保护**：命令号上限 `2^32-2`，达到即报错。
3. **`AtCCI_LocalCache()`**（`xact.c:1537`）= `AtCCI_RelationMap()` +
   `CommandEndInvalidationMessages()`——把刚结束命令产生的目录缓存失效消息
   立即处理，使下一命令能看到本命令的目录改动。

这正是「同一事务内、后一条 SQL 能看到前一条 SQL 的改动」的机制。

## 7. 内存上下文与 ResourceOwner 的生命周期对照

事务相关的三个全局内存上下文：

| 上下文 | 创建 | 销毁 | 用途 |
|---|---|---|---|
| `TransactionAbortContext` | `AtStart_Memory`（首次） | 永不删，`AtCleanup_Memory` 里 reset | 保证 abort 时内存不 OOM |
| `TopTransactionContext` | `AtStart_Memory` | `AtCommit_Memory` / `AtCleanup_Memory` | 顶层事务生命周期内存 |
| `CurTransactionContext` | 顶层即 `TopTransactionContext`；子事务另建 | 随事务/子事务结束 | 当前层级事务内存 |

ResourceOwner 侧：`AtStart_ResourceOwner` 创建 `TopTransactionResourceOwner`；
`CommitTransaction`/`AbortTransaction` 末尾通过
`ResourceOwnerRelease(...BEFORE_LOCKS / LOCKS / AFTER_LOCKS...)`
分三阶段释放，再 `ResourceOwnerDelete(TopTransactionResourceOwner)`。

对应关系：
`start_xact_command` → `AtStart_Memory`/`AtStart_ResourceOwner`（建）；
`finish_xact_command` → `CommitTransaction`→`AtCommit_Memory`（拆），
或错误路径 `AbortCurrentTransaction`→`CleanupTransaction`→`AtCleanup_Memory`（拆）。

## 8. 状态流转速览（exec_simple_query 视角）

**单条普通语句** `SELECT ...`：

```
[1047] start  : DEFAULT --StartTransaction--> STARTED   (state: INPROGRESS)
[1143] start  : (xact_started=true, no-op)
       execute
[1299] finish : STARTED --CommitTransaction--> DEFAULT  (state: DEFAULT)
[1350] finish : (no-op)
```

**多语句（隐式块）** `INSERT a; INSERT b;`：

```
[1047] start  : DEFAULT --StartTransaction--> STARTED
  iter1: [1143] start(no-op); BeginImplicit -> IMPLICIT_INPROGRESS
         execute; 非最后且非事务语句 -> 自调 CommandCounterIncrement()
  iter2: [1143] start(no-op); BeginImplicit(no-op)
         execute; 是最后一条 -> EndImplicit -> STARTED; finish -> CommitTransaction -> DEFAULT
```

**显式事务块** `BEGIN; INSERT; COMMIT;`（一次发送三条）：

```
iter1 BEGIN : ... -> TBLOCK_BEGIN (非最后且是 TransactionStmt -> finish: BEGIN->INPROGRESS)
iter2 INSERT: ... -> TBLOCK_INPROGRESS (非最后、非事务 -> CommandCounterIncrement)
iter3 COMMIT: ... -> TBLOCK_END (最后 -> finish: CommitTransaction -> DEFAULT)
```

**出错**：执行中任何阶段 `ereport(ERROR)` → longjmp 到 `PostgresMain` 错误块
（`postgres.c:4352`）→ `AbortCurrentTransaction()`；若在显式/隐式块内，可能只到
`TBLOCK_ABORT`，由后续 `ROLLBACK` 经 `finish_xact_command` 完成
`CleanupTransaction`。

## 9. 速查表

| 函数 | 位置 |
|---|---|
| `start_xact_command` | `postgres.c:2747` |
| `finish_xact_command` | `postgres.c:2775` |
| `xact_started` 定义 | `postgres.c:135` |
| `StartTransactionCommand` | `xact.c:2937` |
| `StartTransaction` | `xact.c:1993` |
| `CommitTransactionCommand` | `xact.c:3034` |
| `CommitTransaction` | `xact.c:2152` |
| `PrepareTransaction` | `xact.c:2412` |
| `AbortCurrentTransaction` | `xact.c:3305` |
| `AbortTransaction` | `xact.c:2697` |
| `CleanupTransaction` | `xact.c:2888` |
| `CommandCounterIncrement` | `xact.c:1078` |
| `BeginImplicitTransactionBlock` / `EndImplicitTransactionBlock` | `xact.c:4172` / `:4197` |
| `IsAbortedTransactionBlockState` | `xact.c:398` |
| `AtStart_Memory` / `AtStart_ResourceOwner` | `xact.c:1152` / `:1199` |
| `AtCommit_Memory` | `xact.c:1556` |
| `AtAbort_Memory` | `xact.c:1832` |
| `AtCleanup_Memory` | `xact.c:1922` |
| `AtCCI_LocalCache` | `xact.c:1537` |
| 错误恢复中调用 `AbortCurrentTransaction` | `postgres.c:4352` |
| `TransState` / `TBlockState` 枚举 | `xact.c:141` / `:157` |
| `TransactionStateData` | `xact.c:189` |