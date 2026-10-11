# 活动快照栈：`PushActiveSnapshot()` / `PopActiveSnapshot()`

> 说明：本文是学习笔记，聚焦 `exec_simple_query()` 中成对出现的
> `PushActiveSnapshot(GetTransactionSnapshot())` 与 `PopActiveSnapshot()` 两个调用，
> 追踪它们背后的「活动快照栈」（active snapshot stack）实现。
> 权威依据是源码及注释，本文仅作导览。

## 一、定位

`exec_simple_query()` 里有且仅有一对快照压/弹栈，位于「分析规划」前后：

```c
/* src/backend/tcop/postgres.c:1158-1165 */
/* Set up a snapshot if parse analysis/planning will need one. */
if (analyze_requires_snapshot(parsetree))
{
    PushActiveSnapshot(GetTransactionSnapshot());
    snapshot_set = true;
}
...
/* src/backend/tcop/postgres.c:1196-1207 */
/* Done with the snapshot used for parsing/planning. */
if (snapshot_set)
    PopActiveSnapshot();
```

| 函数 | 文件 |
|---|---|
| `exec_simple_query`（调用点 :1163 / :1207） | `src/backend/tcop/postgres.c` |
| `PushActiveSnapshot` / `PushActiveSnapshotWithLevel` | `src/backend/utils/time/snapmgr.c:683` / `:697` |
| `PopActiveSnapshot` | `src/backend/utils/time/snapmgr.c:778` |
| `GetTransactionSnapshot` | `src/backend/utils/time/snapmgr.c:251` |
| `CopySnapshot` | `src/backend/utils/time/snapmgr.c:609` |
| `SnapshotResetXmin` | `src/backend/utils/time/snapmgr.c:944` |
| `AtSubAbort_Snapshot` / `AtEOXact_Snapshot` | `src/backend/utils/time/snapmgr.c:989` / `:1025` |

### 为什么要压栈？

快照（`Snapshot`）回答的是「这条查询能看见哪些行」。解析/分析/规划阶段可能
调用用户自定义函数、访问系统目录（例如展开视图、查询类型），这些都需要一个
当前可见性的基准；执行器更是必须有一个快照才能开始扫描。管理器把「当前应使用
哪个快照」表达为**一个栈**，而非单个全局变量，因为一次命令里可能嵌套多层：

- 外层查询在执行时要保持自己的快照；
- 执行过程中触发的函数、触发器、`SPI`、内部查询又要压入各自的快照；
- 子事务（`SAVEPOINT`）也可能压入快照，回滚时按嵌套层级整体丢弃。

栈模型让「谁压谁弹」天然匹配 C 调用栈与子事务嵌套。

## 二、数据结构

```c
/* src/backend/utils/time/snapmgr.c:119-138 */
typedef struct ActiveSnapshotElt
{
    Snapshot    as_snap;    /* 指向 SnapshotData 的指针 */
    int         as_level;   /* 拥有该快照的事务嵌套层级 */
    struct ActiveSnapshotElt *as_next;
} ActiveSnapshotElt;

static ActiveSnapshotElt *ActiveSnapshot = NULL;      /* 栈顶 */
static ActiveSnapshotElt *OldestActiveSnapshot = NULL;/* 栈底 */
```

- 这是一个**单链表实现的栈**，`ActiveSnapshot` 指向栈顶（= 当前生效快照）。
- 注释强调两点不变量（snapmgr.c:124-125）：
  1. 链表按 `as_level` **非递增**排列——越靠栈顶层级越深；
  2. 链表以 `NULL` 结尾。
  这个有序性是子事务清理（按 level 批量弹出）的前提。
- 每个栈元素**恰好对应 `SnapshotData.active_count` 的一次计数**（snapmgr.c:122）。
  即：压栈 = `active_count++`，弹栈 = `active_count--`。
- 栈元素本身在 `TopTransactionContext` 中分配，因此随顶层事务结束自动回收；
  但仍需显式弹出，以便及时递减 `active_count` 并重置 xmin。

## 三、`PushActiveSnapshot()` 实现

```c
/* src/backend/utils/time/snapmgr.c:682-686 */
void
PushActiveSnapshot(Snapshot snapshot)
{
    PushActiveSnapshotWithLevel(snapshot, GetCurrentTransactionNestLevel());
}
```

`PushActiveSnapshot` 只是「用当前事务嵌套层级」的便捷包装，真正的实现在
`PushActiveSnapshotWithLevel`：

```c
/* src/backend/utils/time/snapmgr.c:696-724 */
void
PushActiveSnapshotWithLevel(Snapshot snapshot, int snap_level)
{
    ActiveSnapshotElt *newactive;

    Assert(snapshot != InvalidSnapshot);
    Assert(ActiveSnapshot == NULL || snap_level >= ActiveSnapshot->as_level);

    newactive = MemoryContextAlloc(TopTransactionContext, sizeof(ActiveSnapshotElt));

    /*
     * Checking SecondarySnapshot is probably useless here, but it seems
     * better to be sure.
     */
    if (snapshot == CurrentSnapshot || snapshot == SecondarySnapshot ||
        !snapshot->copied)
        newactive->as_snap = CopySnapshot(snapshot);
    else
        newactive->as_snap = snapshot;

    newactive->as_next = ActiveSnapshot;
    newactive->as_level = snap_level;

    newactive->as_snap->active_count++;

    ActiveSnapshot = newactive;
    if (OldestActiveSnapshot == NULL)
        OldestActiveSnapshot = ActiveSnapshot;
}
```

逐步解读：

1. **断言**：不允许压入无效快照；`snap_level` 不得比当前栈顶更深
   （否则破坏「非递增」不变量）。这正是子事务向外层压栈时的约束。
2. **分配栈元素**：位于 `TopTransactionContext`，与快照本体同一内存上下文，
   保证生命周期一致。
3. **是否复制快照本体——这是关键设计**：
   - `CurrentSnapshot` / `SecondarySnapshot` 是**静态存储**
     （`CurrentSnapshotData` / `SecondarySnapshotData`，见 snapmgr.c:96-97），
     会被后续 `GetTransactionSnapshot()` / `CommandCounterIncrement()` 覆盖，
     因此必须 `CopySnapshot()` 出一份私有副本；
   - `!snapshot->copied` 表示快照指向静态存储、随时可能被改写，同样要复制；
   - 否则（已是 `copied` 的私有快照，例如本事务首个快照在事务快照隔离级别下
     已复制过）**只递增引用计数、不复制**，多个栈元素可共享同一份 `SnapshotData`。
4. **压栈**：新元素指向旧栈顶，更新 `ActiveSnapshot`；若栈此前为空，
   同时把 `OldestActiveSnapshot` 指向它（栈底只在首次压栈时设置）。
5. **递增 `active_count`**：与前面「一个栈元素一次计数」的约定对应。

### `CopySnapshot()` 做了什么

```c
/* src/backend/utils/time/snapmgr.c:608-628 */
static Snapshot
CopySnapshot(Snapshot snapshot)
{
    ...
    size = subxipoff = sizeof(SnapshotData) +
        snapshot->xcnt * sizeof(TransactionId);
    if (snapshot->subxcnt > 0)
        size += snapshot->subxcnt * sizeof(TransactionId);

    newsnap = (Snapshot) MemoryContextAlloc(TopTransactionContext, size);
    memcpy(newsnap, snapshot, sizeof(SnapshotData));

    newsnap->regd_count = 0;
    newsnap->active_count = 0;
    newsnap->copied = true;
    newsnap->snapXactCompletionCount = 0;
    ...
}
```

要点：

- 在 `TopTransactionContext` 里分配，**并行的 XID 数组（`xip`/`subxip`）与
  `SnapshotData` 头放在同一块 palloc 中**，一次分配、随栈元素一同释放。
- 复制的副本把 `regd_count`/`active_count` 清零、`copied = true`
  —— 于是它被压栈时走「仅递增计数」的分支，而副本本身是私有的、可安全长期持有。
- `snapXactCompletionCount` 清零是快照失效判断（`snapshot->snapXactCompletionCount`
  与全局计数比较）的配套处理。

### `PushCopiedSnapshot()`

```c
/* src/backend/utils/time/snapmgr.c:734-738 */
void
PushCopiedSnapshot(Snapshot snapshot)
{
    PushActiveSnapshot(CopySnapshot(snapshot));
}
```

强制复制一份，供调用方后续要修改快照（如 `UpdateActiveSnapshotCommandId()`
更新 `curcid`）的场景使用——只有独占（`active_count == 1` 且 `regd_count == 0`）
的快照才允许修改。

## 四、`PopActiveSnapshot()` 实现

```c
/* src/backend/utils/time/snapmgr.c:777-798 */
void
PopActiveSnapshot(void)
{
    ActiveSnapshotElt *newstack;

    newstack = ActiveSnapshot->as_next;

    Assert(ActiveSnapshot->as_snap->active_count > 0);

    ActiveSnapshot->as_snap->active_count--;

    if (ActiveSnapshot->as_snap->active_count == 0 &&
        ActiveSnapshot->as_snap->regd_count == 0)
        FreeSnapshot(ActiveSnapshot->as_snap);

    pfree(ActiveSnapshot);
    ActiveSnapshot = newstack;
    if (ActiveSnapshot == NULL)
        OldestActiveSnapshot = NULL;

    SnapshotResetXmin();
}
```

逐步解读：

1. **取出下一元素**：先记录 `as_next`，随后才释放当前栈元素。
2. **递减 `active_count`**：与压栈对称。
3. **条件释放快照本体**：仅当 `active_count == 0`（无人再活跃引用）**且**
   `regd_count == 0`（无人注册，如被 `RegisterSnapshot()` 或 portal 持有）时，
   才 `FreeSnapshot()`。注意快照本体可能被多个栈元素共享（压栈时未复制的情形），
   所以必须等计数归零才能释放。
4. **释放栈元素并回退栈顶**：`pfree` 栈元素、`ActiveSnapshot = newstack`；
   若栈已空，一并清空 `OldestActiveSnapshot`。
5. **`SnapshotResetXmin()`**：见下节，这是弹栈的**副作用之一**，直接关系到
   全局 vacuum 能否推进。

`FreeSnapshot()` 只做三件事的断言检查后 `pfree`：

```c
/* src/backend/utils/time/snapmgr.c:664-672 */
static void
FreeSnapshot(Snapshot snapshot)
{
    Assert(snapshot->regd_count == 0);
    Assert(snapshot->active_count == 0);
    Assert(snapshot->copied);   /* 只有复制过的私有快照才归自己释放 */
    pfree(snapshot);
}
```

## 五、弹栈的副作用：`SnapshotResetXmin()`

```c
/* src/backend/utils/time/snapmgr.c:943-962 */
static void
SnapshotResetXmin(void)
{
    Snapshot minSnapshot;

    if (ActiveSnapshot != NULL)     /* 栈非空：什么都不做 */
        return;

    if (pairingheap_is_empty(&RegisteredSnapshots))
    {
        MyProc->xmin = InvalidTransactionId;   /* 彻底没有快照，清空 xmin */
        return;
    }

    minSnapshot = pairingheap_container(SnapshotData, ph_node,
                                        pairingheap_first(&RegisteredSnapshots));
    if (TransactionIdPrecedes(MyProc->xmin, minSnapshot->xmin))
        MyProc->xmin = minSnapshot->xmin;      /* 推进到注册快照中的最小 xmin */
}
```

- `MyProc->xmin` 是后端向全局公布的「我需要的、最老的事务 ID」。
  它越小，全局 xmin 越老，vacuum 能回收的死元组越少。
- 弹栈后若活动栈为空，就有机会**前推** `MyProc->xmin`：没有活动快照时，
  只需考虑 `RegisteredSnapshots`（注册快照，按 xmin 排序的配对堆，栈顶即最小 xmin）。
- 注意 `MyProc->xmin` 是**只增不减**的（此处在推进而非回退），清空仅发生在
  完全没有快照时。
- 注释（snapmgr.c:930-941）解释了为何「只在活动栈为空时才重算」：
  这样可以不必跟踪哪个活动快照最老；同时也提醒不能改用 `GetOldestSnapshot()`
  （它按 LSN 比较，语义不同）。

## 六、异常路径：子事务与事务结束如何清理

`PopActiveSnapshot()` 只在**正常路径**弹出。若中途出错、或快照属于被回滚的
子事务，栈的清理由事务管理器的回调完成：

```c
/* src/backend/utils/time/snapmgr.c:988-1018 */
void
AtSubAbort_Snapshot(int level)
{
    /* Forget the active snapshots set by this subtransaction */
    while (ActiveSnapshot && ActiveSnapshot->as_level >= level)
    {
        ActiveSnapshotElt *next = ActiveSnapshot->as_next;

        Assert(ActiveSnapshot->as_snap->active_count >= 1);
        ActiveSnapshot->as_snap->active_count -= 1;

        if (ActiveSnapshot->as_snap->active_count == 0 &&
            ActiveSnapshot->as_snap->regd_count == 0)
            FreeSnapshot(ActiveSnapshot->as_snap);

        pfree(ActiveSnapshot);

        ActiveSnapshot = next;
        if (ActiveSnapshot == NULL)
            OldestActiveSnapshot = NULL;
    }

    SnapshotResetXmin();
}
```

- 子事务（`SAVEPOINT`）回滚时，把 `as_level >= level` 的栈元素**整体弹出**，
  循环体与 `PopActiveSnapshot()` 同构——正是第二节那条「按 level 非递增排列」
  不变量在支撑这种批量清理。
- `AtSubCommit_Snapshot`（:967）则相反：把该子事务压入的栈元素 level
  从 `level` 改标为 `level-1`，让它们「继承」给父事务。
- `AtEOXact_Snapshot`（:1025）在事务结束时做整体清理并（按需）重置 xmin。

也就是说：**压栈后可以不弹而依赖事务清理**，但常规代码路径都应成对调用以尽早
释放引用、前推 xmin。`exec_simple_query()` 里 `snapshot_set` 标志就是保证
「压了才弹」的体现。

## 七、回到 `exec_simple_query()`：为何这样压/弹

### 1）为什么解析/规划前需要快照

`analyze_requires_snapshot()`（`parser/analyze.c:431`）判断该原始语法树在
分析阶段是否需要读取数据库——大多数 DML/查询需要，部分 utility 语句不需要。
`pg_plan_query()` 内部还有一条硬断言：

```c
/* src/backend/tcop/postgres.c:896 */
/* Planner must have a snapshot in case it calls user-defined functions. */
Assert(ActiveSnapshotSet());
```

规划器可能调用用户自定义函数（如代价估算、`CHECK` 表达式等），所以必须有生效快照。
这解释了「按需压栈」的位置必须在 `pg_analyze_and_rewrite_*` 与 `pg_plan_queries`
**之前**。

### 2）为什么规划完立刻弹，执行时不复用

源码注释（postgres.c:1196-1205）明确说明：**不能**把分析期快照复用于执行。

> 该快照是在对查询涉及的表加锁**之前**取得的，复用它会导致执行使用一个
> 「加锁前」的可见性视图，产生用户可见的异常。参考
> https://postgr.es/m/flat/5075D8DF.6050500@fuzzy.cz

因此语义是：

- 压栈 → 覆盖「分析 + 重写 + 规划」阶段（这段可能需要快照）；
- 弹栈 → 结束这段用途，**前推 `MyProc->xmin`**（若栈空）；
- 执行阶段由 `PortalStart()` / 执行器**重新取得**执行用快照，
  保证可见性发生在加锁之后。

这也提醒：弹栈不只是「释放内存」，它还是**推进全局 xmin、允许 vacuum 回收**的
时机。分析期快照若一直不弹，会不必要地拖住 `MyProc->xmin`。

### 3）与扩展协议的对照

同样的模式在扩展协议里重复出现：

| 位置 | 用途 |
|---|---|
| `exec_simple_query` :1163 / :1207 | 分析+重写+规划 |
| `exec_parse_message` :1512 / :1529 | 仅分析+重写（Parse 阶段） |
| `exec_bind_message` :1777（弹栈在 :2028） | 绑定参数 + 规划（Bind 阶段） |

三者都遵循「需要时 `PushActiveSnapshot(GetTransactionSnapshot())` +
`snapshot_set = true`，用完 `if (snapshot_set) PopActiveSnapshot()`」的固定模板，
且都**不复用**快照供执行使用。理解这一对函数，是理解三阶段协议各自快照边界的钥匙。

## 八、速查

| 关注点 | 结论 |
|---|---|
| 结构 | 单链表栈，`as_level` 非递增，元素于 `TopTransactionContext` |
| 压栈 | 递减无关；按需 `CopySnapshot`（静态/未复制则复制），`active_count++` |
| 弹栈 | `active_count--`；归零且未注册则 `FreeSnapshot`；随后 `SnapshotResetXmin` |
| 复制时机 | `CurrentSnapshot`/`SecondarySnapshot`/`!copied` → 复制；否则共享 |
| 共享快照 | 多个栈元素可指向同一 `SnapshotData`，靠 `active_count` 管理生命周期 |
| 异常清理 | 子事务回滚由 `AtSubAbort_Snapshot` 按 level 批量弹出 |
| 关键副作用 | 弹栈触发 xmin 前推，影响全局 vacuum 可回收性 |
| `exec_simple_query` 语义 | 快照覆盖分析/规划；执行阶段另取快照，绝不复用 |

## 九、关键源码位置

| 函数 | 位置 |
|---|---|
| `PushActiveSnapshot` / `PushActiveSnapshotWithLevel` | `src/backend/utils/time/snapmgr.c:683` / `:697` |
| `PushCopiedSnapshot` | `src/backend/utils/time/snapmgr.c:735` |
| `PopActiveSnapshot` | `src/backend/utils/time/snapmgr.c:778` |
| `GetActiveSnapshot` / `ActiveSnapshotSet` | `src/backend/utils/time/snapmgr.c:805` / `:817` |
| `GetTransactionSnapshot` | `src/backend/utils/time/snapmgr.c:251` |
| `CopySnapshot` / `FreeSnapshot` | `src/backend/utils/time/snapmgr.c:609` / `:665` |
| `SnapshotResetXmin` | `src/backend/utils/time/snapmgr.c:944` |
| `AtSubCommit_Snapshot` / `AtSubAbort_Snapshot` | `src/backend/utils/time/snapmgr.c:968` / `:989` |
| `AtEOXact_Snapshot` | `src/backend/utils/time/snapmgr.c:1025` |
| `ActiveSnapshotElt` 定义 | `src/backend/utils/time/snapmgr.c:127` |
| `exec_simple_query` 调用点 | `src/backend/tcop/postgres.c:1163` / `:1207` |
| `analyze_requires_snapshot` | `src/backend/parser/analyze.c:431` |
| `pg_plan_query` 断言 | `src/backend/tcop/postgres.c:896` |