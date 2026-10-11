# PostgreSQL 简单查询协议入口：`exec_simple_query()` 分析

> 说明：本文是学习笔记，追踪「一条 SQL 从字符串到执行结果」的完整链路。
> 权威依据是源码与 `src/backend/optimizer/README` 等设计文档，本文仅作导览。

## 一、定位与签名

- **文件**：`src/backend/tcop/postgres.c:1013`
- **签名**：
  ```c
  static void
  exec_simple_query(const char *query_string)
  ```
- **注释标注**：`Execute a "simple Query" protocol message.`
- **所属模块**：`tcop`（traffic cop / command processor），后端主循环与协议处理层。

`exec_simple_query()` 是 **Simple Query 协议**（前端发送 `Q` 消息，一次一个 SQL 字符串）
的唯一执行入口。它把「一条（或多条）SQL 文本」走完
**原始解析 → 语义分析 → 重写 → 规划 → 执行 → 返回结果** 的全过程。

> 与之并列的是扩展协议（Extended Query）：`exec_parse_message()`(`P`)、
> `exec_bind_message()`(`B`)、`exec_execute_message()`(`E`)，把上面这条流水线拆成
> Parse / Bind / Execute 三个阶段，支持预处理语句与参数绑定。`exec_simple_query()`
> 则把三阶段「内联」在一次调用里完成。

## 二、谁调用它

后端主循环 `PostgresMain()` 在读到前端消息首字符时分发
（`src/backend/tcop/postgres.c:4614` 起的 `switch (firstchar)`）：

```c
case 'Q':   /* simple query */
    SetCurrentStatementStartTimestamp();
    query_string = pq_getmsgstring(&input_message);
    pq_getmsgend(&input_message);

    if (am_walsender)
    {
        if (!exec_replication_command(query_string))
            exec_simple_query(query_string);
    }
    else
        exec_simple_query(query_string);
    ...
    send_ready_for_query = true;
```

即：普通后端 → `exec_simple_query()`；walsender 进程先尝试按复制命令处理，
不是复制命令再回退到普通查询路径。执行完毕后主循环会发送 `ReadyForQuery`。

## 三、调用链路总览

```
PostgresMain()                         postgres.c:4614  switch('Q')
  └── exec_simple_query()              postgres.c:1013  ← 本文主角
        │
        ├── start_xact_command()       postgres.c:2747  StartTransactionCommand
        ├── drop_unnamed_stmt()        postgres.c:2854  丢弃旧的未命名预处理语句
        ├── pg_parse_query()           postgres.c:611   raw_parser() → List<RawStmt>
        │     └── raw_parser()          gram.y / scan.l 原始语法分析
        │
        └── foreach(RawStmt)                            ← 逐条语句循环
              ├── CreateCommandTag()    utility.c:2363  取命令标签(如 SELECT)
              ├── BeginCommand()        dest.c          通知目标接收器
              ├── [aborted 事务检查]
              ├── start_xact_command()
              ├── [BeginImplicitTransactionBlock()]     xact.c:4172 多语句隐式事务块
              ├── [PushActiveSnapshot()]                需要快照时
              │
              ├── pg_analyze_and_rewrite_fixedparams()  postgres.c:671
              │     ├── parse_analyze_fixedparams()      parser/analyze.c:107  语义分析
              │     └── pg_rewrite_query()               rewrite/rewriteHandler.c  规则重写
              │
              ├── pg_plan_queries()      postgres.c:972
              │     └── pg_plan_query()  postgres.c:886
              │           └── planner()  optimizer/plan/planner.c:273  ← 优化器入口
              │
              ├── [PopActiveSnapshot()]
              ├── CreatePortal()         utils/mmgr/portalmem.c:176
              ├── PortalDefineQuery()    utils/mmgr/portalmem.c:283
              ├── PortalStart()          tcop/pquery.c:433
              ├── PortalSetResultFormat()tcop/pquery.c:623
              ├── CreateDestReceiver()   tcop/dest.c:113
              ├── PortalRun()            tcop/pquery.c:684  ← 真正执行
              │     ├── PortalRunSelect()   pquery.c:867   查询类语句
              │     ├── PortalRunUtility()  pquery.c:1127  工具类语句(CREATE/...)
              │     ├── PortalRunMulti()    pquery.c:1190  重写产生多条时
              │     └── PortalRunFetch()    pquery.c:1391  FETCH
              ├── receiver->rDestroy()
              ├── PortalDrop()           utils/mmgr/portalmem.c:469
              │
              ├── [事务收尾] finish_xact_command() / CommandCounterIncrement()
              └── EndCommand()           dest.c  发送 CommandComplete
```

## 四、逐段剖析

### 1）监控与运行状态上报（1015–1038）

```c
CommandDest dest = whereToSendOutput;   /* 结果发往哪里 */
...
debug_query_string = query_string;                  /* 供调试/日志使用 */
pgstat_report_activity(STATE_RUNNING, query_string);/* 上报 pg_stat_activity */
TRACE_POSTGRESQL_QUERY_START(query_string);         /* DTrace 探针 */
if (save_log_statement_stats)
    ResetUsage();                                   /* 统计计时复位 */
```

`whereToSendOutput` 决定结果的去向，取值来自 `CommandDest` 枚举
（`DestNone`/`DestDebug`/`DestRemote`/`DestRemoteExecute`/`DestRemoteSimple`/
`DestSPI`/`DestSQLFunction`/`DestNone` 等），最终由 `CreateDestReceiver()` 转成
具体的元组接收器对象。

### 2）开启事务命令 + 清理未命名语句（1040–1055）

```c
start_xact_command();   /* 若未在事务命令中则 StartTransactionCommand() */
drop_unnamed_stmt();    /* 释放上一次 Simple Query 的未命名预处理语句 */
```

- `start_xact_command()`（2747）是「每个命令一个事务命令（transaction command）」
  的抽象：`StartTransactionCommand()` 只在 `xact_started == false` 时调用，
  避免重复开启；同时启用语句超时（`statement_timeout`）与客户端断连检测超时。
- `drop_unnamed_stmt()`（2854）把 Simple Query 语义上视为「使用了未命名
  语句和 portal」，因此先释放上一次残留的未命名 `CachedPlanSource`，
  保证旧内存被回收。

> 注意：`start_xact_command()` 里的 `StartTransactionCommand()` 会切换当前内存上下文，
> 因此后续切换 context 时要保存/恢复 `oldcontext`。

### 3）原始解析（1057–1081）

```c
oldcontext = MemoryContextSwitchTo(MessageContext);
parsetree_list = pg_parse_query(query_string);   /* List<RawStmt> */
if (check_log_statement(parsetree_list))
    ereport(LOG, ... "statement: %s" ...);
MemoryContextSwitchTo(oldcontext);
```

- 解析在 `MessageContext` 中进行，因为解析树要跨越「命令级事务」存活。
- `pg_parse_query()`（611）内部就是 `raw_parser()`，返回 **`List` of `RawStmt`**，
  因为一个字符串里可能有多条语句。此阶段 **不做任何数据库访问**，所以即使
  事务已 abort 也能安全执行（后面的中止检查依赖这一点：只有原始解析能识别出
  `COMMIT`/`ABORT`）。
- `check_log_statement()` 按 `log_statement` 设置决定是否立即记录日志。

### 4）隐式事务块判定（1083–1091）

```c
use_implicit_block = (list_length(parsetree_list) > 1);
```

历史上，简单查询里若一次发送多条语句，它们默认在 **同一个事务** 中执行，
除非显式写入了事务控制命令。为在事务机制中正确表达这一行为，引入「隐式事务块」
（implicit transaction block）。

### 5）逐条语句主循环（1096 起）

循环体每次处理一个 `RawStmt`，主要动作：

**(a) 命令标签与目标通知（1111–1124）**

```c
commandTag = CreateCommandTag(parsetree->stmt);          /* 如 CMDTAG_SELECT */
cmdtagname = GetCommandTagNameAndLen(commandTag, &cmdtaglen);
set_ps_display_with_len(cmdtagname, cmdtaglen);          /* 更新进程标题 */
BeginCommand(commandTag, dest);                          /* 目标接收器开始 */
```

**(b) 中止事务检查（1126–1140）**

```c
if (IsAbortedTransactionBlockState() &&
    !IsTransactionExitStmt(parsetree->stmt))
    ereport(ERROR, ... "current transaction is aborted, ...");
```

必须在分析/重写/规划之前判断——因为那些阶段都要访问数据库，在中止状态下
可能失败。只有 `IsTransactionExitStmt()`（2807）认可的
`COMMIT`/`PREPARE`/`ROLLBACK`/`ROLLBACK TO` 才放行。

**(c) 事务命令与隐式块（1142–1156）**

```c
start_xact_command();
if (use_implicit_block)
    BeginImplicitTransactionBlock();
CHECK_FOR_INTERRUPTS();
```

**(d) 按需建立快照（1158–1165）**

```c
if (analyze_requires_snapshot(parsetree))   /* analyze.c:431 */
{
    PushActiveSnapshot(GetTransactionSnapshot());
    snapshot_set = true;
}
```

并非所有语句都需要快照（例如部分 utility 语句不需要），按需压栈。

**(e) 选择内存上下文（1167–1188）**

```c
if (lnext(parsetree_list, parsetree_item) != NULL)
{
    /* 不是最后一条：单独开一个 per-parsetree 上下文，用完即删 */
    per_parsetree_context = AllocSetContextCreate(MessageContext, ...);
    oldcontext = MemoryContextSwitchTo(per_parsetree_context);
}
else
    oldcontext = MemoryContextSwitchTo(MessageContext);  /* 最后一条用 MessageContext */
```

多语句场景下为每条语句单独分配上下文，处理完立刻释放，避免内存堆积；
最后一条（或唯一一条）直接用 `MessageContext`（稍后会被 reset）。
发生错误时，`per_parsetree_context` 随 `MessageContext` 一起删除。

**(f) 分析 + 重写（1190–1191）**

```c
querytree_list = pg_analyze_and_rewrite_fixedparams(parsetree, query_string,
                                                    NULL, 0, NULL);
```

`pg_analyze_and_rewrite_fixedparams()`（671）两步：

1. `parse_analyze_fixedparams()`（`parser/analyze.c:107`）：原始语法树 → `Query`
   （语义分析：名称解析、类型推断、`$n` 参数类型确定等）。
2. `pg_rewrite_query()`（`rewrite/rewriteHandler.c`）：应用规则系统（视图展开等），
   一个 `Query` 可能膨胀成多个，故返回 **`List<Query>`**。

`_fixedparams` 表示参数类型已由调用方给定（Simple Query 无参数，故传 `NULL, 0`）；
扩展协议另有 `_varparams` 版本。

**(g) 规划（1193–1194）**

```c
plantree_list = pg_plan_queries(querytree_list, query_string,
                                CURSOR_OPT_PARALLEL_OK, NULL);
```

`pg_plan_queries()`（972）逐个 `Query`：

- `commandType == CMD_UTILITY`（如 `CREATE TABLE`）→ **不需要规划**，直接包一个
  只含 `utilityStmt` 的 `PlannedStmt`。
- 否则调用 `pg_plan_query()`（886）→ `planner()`（优化器入口），返回 `PlannedStmt`。

`CURSOR_OPT_PARALLEL_OK` 表示允许为游标使用并行计划。

**(h) 释放分析期快照（1196–1210）**

```c
if (snapshot_set)
    PopActiveSnapshot();
CHECK_FOR_INTERRUPTS();
```

> 源码特意 **不复用** 该快照去执行：`GetTransactionSnapshot()` 是在对查询涉及的表
> 加锁之前取得的，复用会产生用户可见的异常，因此执行阶段重新取快照。

**(i) 创建并启动 Portal（1212–1235）**

```c
portal = CreatePortal("", true, true);   /* 未命名 portal，允许重名并静默替换 */
portal->visible = false;                 /* 不出现在 pg_cursors */
PortalDefineQuery(portal, NULL, query_string, commandTag, plantree_list, NULL);
PortalStart(portal, NULL, 0, InvalidSnapshot);
```

Portal 是「一条已规划查询的运行实例」，负责游标状态、结果格式、执行计数。
此处无需把任何东西拷贝进 Portal，因为传入的对象位于 `MessageContext` /
`per_parsetree_context`，生命周期本就长于 Portal。

**(j) 结果格式（1237–1257）**

```c
format = 0;   /* TEXT 默认 */
if (IsA(parsetree->stmt, FetchStmt)) { ... 若为二进制游标则 format = 1 ... }
PortalSetResultFormat(portal, 1, &format);
```

只剩 `FETCH` 一个历史遗留例外会走二进制格式（源码注释自嘲为
"the joys of backward compatibility"）。

**(k) 目标接收器（1259–1269）**

```c
receiver = CreateDestReceiver(dest);
if (dest == DestRemote)
    SetRemoteDestReceiverParams(receiver, portal);
MemoryContextSwitchTo(oldcontext);   /* 切回事务上下文执行 */
```

**(l) 执行（1271–1284）**

```c
(void) PortalRun(portal,
                 FETCH_ALL,     /* 取全部行 */
                 true,          /* always top level */
                 true,          /* run_once */
                 receiver,      /* 主目标：结果行发往此处 */
                 receiver,      /* 备用目标（如 SELECT INTO 之外） */
                 &qc);          /* 输出：QueryCompletion，含行数等 */
receiver->rDestroy(receiver);
PortalDrop(portal, false);
```

`PortalRun()`（pquery.c:684）内部按语句类型分派到
`PortalRunSelect` / `PortalRunUtility` / `PortalRunMulti` / `PortalRunFetch`，
把 `Plan` 交给执行器 `ExecutorStart/Run/End` 真正跑起来，结果通过
`DestReceiver` 的 `receiveSlot` 回送给前端。

**(m) 事务收尾（1286–1330）**

```c
if (lnext(parsetree_list, parsetree_item) == NULL)
{
    /* 最后一条：先结束隐式块，再关闭事务命令
       以便事务结束期的错误在 CommandComplete 之前报告 */
    if (use_implicit_block)
        EndImplicitTransactionBlock();
    finish_xact_command();
}
else if (IsA(parsetree->stmt, TransactionStmt))
{
    /* 事务控制语句：立即提交，下一条语句另起事务命令 */
    finish_xact_command();
}
else
{
    Assert(!(MyXactFlags & XACT_FLAGS_NEEDIMMEDIATECOMMIT));
    CommandCounterIncrement();      /* 让本语句的改动对后续语句可见 */
    disable_statement_timeout();    /* 多语句串中每条语句独立计时 */
}
```

这里体现了三种收尾策略，是理解「一个 Simple Query 字符串里多条语句如何组事务」
的关键：

| 情形 | 处理 | 事务效果 |
|---|---|---|
| 最后一条语句 | `finish_xact_command()`（必要时先 `EndImplicitTransactionBlock()`） | 关闭当前事务命令 |
| 非最后一条、且是事务控制语句 | `finish_xact_command()` | 提交/回滚，下一条另起事务 |
| 非最后一条、普通语句 | `CommandCounterIncrement()` + 禁用超时 | 与后续语句同属一个事务命令 |

**(n) 发送完成消息（1332–1342）**

```c
EndCommand(&qc, dest, false);        /* 每个 RawStmt 恰好一次 CommandComplete */
if (per_parsetree_context)
    MemoryContextDelete(per_parsetree_context);
```

### 6）循环之后（1345–1383）

```c
finish_xact_command();               /* 空 parsetree_list 时的兜底收尾 */

if (!parsetree_list)
    NullCommand(dest);               /* 空查询 → EmptyQueryResponse */

switch (check_log_duration(msec_str, was_logged)) { ... 记录 duration ... }

if (save_log_statement_stats)
    ShowUsage("QUERY STATISTICS");

TRACE_POSTGRESQL_QUERY_DONE(query_string);
debug_query_string = NULL;
```

## 五、关键数据结构

| 结构 | 产生阶段 | 说明 |
|---|---|---|
| `RawStmt` | 原始解析 | gram.y 输出，纯语法树，不含语义信息 |
| `Query` | 语义分析 | 已解析、带类型信息的查询树；重写后可能一条变多条 |
| `PlannedStmt` | 规划 | 可执行计划；utility 语句则是仅含 `utilityStmt` 的包装 |
| `Portal` | 执行 | 已规划查询的运行实例，管理游标、格式、执行计数 |
| `DestReceiver` | 执行 | 结果行接收器，按 `CommandDest` 选择实现 |
| `QueryCompletion` | 执行 | 执行结果摘要（命令标签 + 行数），用于 `CommandComplete` |

## 六、贯穿全流程的三条主线

1. **协议语义**：Simple Query 天然支持「一字符串多语句」。通过
   `use_implicit_block` + 循环内的事务收尾分支，把这些语句组织成
   一个隐式事务块；遇到显式 `BEGIN/COMMIT/ROLLBACK` 时再切分。
2. **内存上下文**：解析/分析/规划产物必须活得比「命令级事务」更久，故放在
   `MessageContext`；多语句时用 `per_parsetree_context` 逐条隔离并尽早释放；
   执行阶段切回事务上下文。
3. **快照与中断**：分析/规划按需 `GetTransactionSnapshot()`（执行时重新取，
   不复用）；每个关键阶段之间用 `CHECK_FOR_INTERRUPTS()` 响应取消信号。

## 七、与扩展协议的关系

```
Simple Query:    [Q] ── exec_simple_query ──▶ parse+analyze+rewrite+plan+execute 一次完成
Extended Query:  [P] exec_parse_message       ← parse + analyze + rewrite
                 [B] exec_bind_message        ← 绑定参数
                 [E] exec_execute_message     ← plan + execute
```

`exec_simple_query()` 相当于把扩展协议三阶段合并执行，且有「未命名 portal」
「多语句单事务」等历史语义。理解它是理解扩展协议各阶段职责的基础。

## 八、速查（关键源码位置）

| 函数 | 位置 |
|---|---|
| `exec_simple_query` | `src/backend/tcop/postgres.c:1013` |
| `pg_parse_query` | `src/backend/tcop/postgres.c:611` |
| `pg_analyze_and_rewrite_fixedparams` | `src/backend/tcop/postgres.c:671` |
| `parse_analyze_fixedparams` | `src/backend/parser/analyze.c:107` |
| `pg_rewrite_query` | `src/backend/tcop/postgres.c:804` |
| `pg_plan_queries` / `pg_plan_query` | `src/backend/tcop/postgres.c:972` / `:886` |
| `planner` | `src/backend/optimizer/plan/planner.c:273` |
| `start_xact_command` / `finish_xact_command` | `src/backend/tcop/postgres.c:2747` / `:2775` |
| `drop_unnamed_stmt` | `src/backend/tcop/postgres.c:2854` |
| `BeginImplicitTransactionBlock` / `EndImplicitTransactionBlock` | `src/backend/access/transam/xact.c:4172` / `:4197` |
| `CreatePortal` / `PortalDefineQuery` / `PortalDrop` | `src/backend/utils/mmgr/portalmem.c:176` / `:283` / `:469` |
| `PortalStart` / `PortalSetResultFormat` / `PortalRun` | `src/backend/tcop/pquery.c:433` / `:623` / `:684` |
| `CreateDestReceiver` | `src/backend/tcop/dest.c:113` |
| `CreateCommandTag` | `src/backend/tcop/utility.c:2363` |
| 调用点（`PostgresMain` 的 `'Q'` 分支） | `src/backend/tcop/postgres.c:4614` |