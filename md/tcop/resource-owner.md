# ResourceOwner 详解

> 学习笔记。权威依据：`src/backend/utils/resowner/README`、
> `src/backend/utils/resowner/resowner.c`、`src/include/utils/resowner.h`、
> `src/include/utils/resowner_private.h`。本文所有行号均已核对源码。

## 1. 一句话定位

`ResourceOwner`（资源所有者）是 PostgreSQL 用来**可靠地跟踪与释放「查询生命周期资源」**
（buffer pin、表锁、各类引用计数、临时文件等）的对象。它的头文件原话：

> Query-lifespan resources are tracked by associating them with ResourceOwner
> objects. This provides a simple mechanism for ensuring that such resources are
> freed at the right time.

它是一个**不透明的句柄类型**（只暴露指针，结构体定义藏在 resowner.c 内部）：

```c
/* src/include/utils/resowner.h */
typedef struct ResourceOwnerData *ResourceOwner;
```

这样设计的核心动机，README 讲得很直白：**不指望执行器每个数据结构都能做到
「百密无疏」的清理**，而是把这类资源的跟踪**集中到一个模块**里。于是无论查询
是正常结束还是中途 `ereport(ERROR)` 退出，都能保证资源在同一时间点被回收干净。

其 API 的层次结构（父子森林）**刻意模仿 `MemoryContext`** —— README：

> The design of the ResourceOwner API is modeled on our MemoryContext API,
> which has proven very flexible and successful in preventing memory leaks.
> ... we allow ResourceOwners to have child ResourceOwner objects so that there
> can be forests of the things; releasing a parent ResourceOwner acts on all its
> direct and indirect children as well.

## 2. 要解决的问题

没有 ResourceOwner 时，执行器的资源清理会非常脆弱：

- 一次查询可能 `ReadBuffer()` 钉住（pin）多个 buffer、`LockAcquire()` 拿一堆锁、
  `RelationOpenRelation()` 增加 relcache 引用、注册 snapshot、`OpenTemporaryFile()` 开临时文件……
- 执行路径上有无数错误分支；一个函数可以 `ereport(ERROR)` 直接 longjmp 出去，
  栈上的局部清理代码根本来不及执行。
- 如果每个数据结构都自己兜底清理，代码会爆炸且极易漏。

ResourceOwner 给出的方案：**所有这类资源在「获取时」登记到当前 ResourceOwner，
在「正常释放时」注销；一旦出错，事务/子事务/Portal 结束时的统一释放流程会把
仍然登记着的资源全部收尸。** 于是：

- 正常路径：资源按时注销，ResourceOwner 通常为空，代码整洁。
- 错误路径：longjmp 跳过注销，残余资源由 `ResourceOwnerRelease()` 统一释放。
- 调试构建：正常结束时若还有残留，会打印泄漏警告（见 §11）。

## 3. 对象结构：`ResourceOwnerData`

定义于 `resowner.c:115`：

```c
typedef struct ResourceOwnerData
{
    ResourceOwner parent;        /* NULL 表示顶层 owner */
    ResourceOwner firstchild;    /* 子 owner 链表头 */
    ResourceOwner nextchild;     /* 同一父下的下一个兄弟 */
    const char *name;            /* 仅用于调试 */

    /* 内建支持的资源类别 —— 每类一个 ResourceArray */
    ResourceArray bufferarr;      /* 钉住的 buffer (pin) */
    ResourceArray bufferioarr;    /* 进行中的 buffer IO */
    ResourceArray catrefarr;      /* catcache 引用 */
    ResourceArray catlistrefarr;  /* catcache 列表引用 */
    ResourceArray relrefarr;      /* relcache 引用 */
    ResourceArray planrefarr;     /* plancache 引用 */
    ResourceArray tupdescarr;     /* tupdesc 引用 */
    ResourceArray snapshotarr;    /* snapshot 引用 */
    ResourceArray filearr;        /* 打开的临时文件 */
    ResourceArray dsmarr;         /* 动态共享内存段 */
    ResourceArray jitarr;         /* JIT 上下文 */
    ResourceArray cryptohasharr;  /* cryptohash 上下文 */
    ResourceArray hmacarr;        /* HMAC 上下文 */

    /* 最多缓存 MAX_RESOWNER_LOCKS 个本地锁引用 */
    int          nlocks;
    LOCALLOCK   *locks[MAX_RESOWNER_LOCKS];
} ResourceOwnerData;
```

要点：

- **树形结构**：`parent` / `firstchild` / `nextchild`，与 `MemoryContext` 一样是森林。
- **13 类内建资源**各用一个 `ResourceArray` 存 ID；所有 ID 都塞进一个 `Datum`
  （通常是指针或 int；`File`/`Buffer` 用 `Int32GetDatum` 转换，见 `resowner.c:44`）。
- **锁是特例**：不是用 `ResourceArray`，而是一个定长 15 的**有损缓存**
  `locks[MAX_RESOWNER_LOCKS]`（`MAX_RESOWNER_LOCKS = 15`，`resowner.c:110`，
  注释说明这个数字来自对 pg_dump 大 schema 的实测，就是为了让
  `ResourceOwnerForgetLock` 不必扫描超大数组）。溢出后就不再记录，
  回退到锁管理器自己的哈希表。见 §6.4。

## 4. `ResourceArray`：既省内存又够快的小集合

`ResourceArray`（`resowner.c:65` 附近）是一个通用的小集合容器，两种模式自适应：

| 规模 | 存储方式 | 查找 |
|---|---|---|
| 小（`capacity <= 64`） | 线性数组，`itemsarr[0..nitems)` | 线性扫描 + `lastidx` 快速命中 |
| 大 | 开放寻址哈希表，容量是 2 的幂 | `hash & (capacity-1)` 探测 |

- 初始容量 `RESARRAY_INIT_SIZE = 16`；`RESARRAY_MAX_ARRAY = 64` 是模式切换阈值。
- 哈希模式下 `maxitems = capacity/4*3`，故意留空槽保证探测不退化。
- `lastidx` 记住上次插入/取出的位置，让「后进先出」式的成对 `Remove` 很快
  （`ResourceArrayAdd` 设 `lastidx`，`ResourceArrayRemove` 先试 `lastidx`）。
- 数组模式删除时用「把最后一个元素搬过来覆盖」的交换删除法，O(1)。

## 5. 全局 owner 与层级

头文件里导出四个全局变量：

```c
extern PGDLLIMPORT ResourceOwner CurrentResourceOwner;       /* 当前归属对象 */
extern PGDLLIMPORT ResourceOwner CurTransactionResourceOwner;/* 当前事务/子事务 */
extern PGDLLIMPORT ResourceOwner TopTransactionResourceOwner;/* 顶层事务 */
extern PGDLLIMPORT ResourceOwner AuxProcessResourceOwner;    /* 辅助进程 */
```

创建点（`ResourceOwnerCreate(parent, name)`）：

| owner | 创建位置 | 父 |
|---|---|---|
| `TopTransactionResourceOwner` | `xact.c:1211`（`AtStart_ResourceOwner`） | NULL |
| 子事务 owner | `xact.c:1262`（`AtSubStart_ResourceOwner`） | 父事务 owner |
| Portal owner（`portal->resowner`） | `portalmem.c:206`（`CreatePortal`） | `CurTransactionResourceOwner` |
| `AuxProcessResourceOwner` | `resowner.c:918`（`CreateAuxProcessResourceOwner`） | NULL |
| base backup owner | `basebackup.c:248` | NULL |

层级语义：

```
TopTransactionResourceOwner
├── SubTransactionResourceOwner (子事务)
│   └── ...
└── PortalResourceOwner          (CreatePortal 时挂到 CurTransactionResourceOwner 下)
```

- **`CurrentResourceOwner` 决定「谁登记」**：执行期间指向当前 Portal 的 owner，
  所以 `ReadBuffer`/`LockAcquire` 会把资源记到 Portal 名下。
- **释放父 owner 会递归处理所有子孙**（`ResourceOwnerReleaseInternal` 先递归子节点）。
- **Portal 关闭**时，普通情况下它的锁**不真正释放**，而是「转交」给父（当前事务）
  ——因为锁在正常语义下应持有到事务结束。见 §7、§8。
- 无事务（或事务已失败）时 `CurrentResourceOwner == NULL`，此时**不允许获取**
  查询生命周期资源（README 明确写了这一点）。

## 6. 核心 API 与三步入库协议

### 6.1 通用 API（`resowner.h`）

```c
ResourceOwner ResourceOwnerCreate(ResourceOwner parent, const char *name);
void ResourceOwnerRelease(ResourceOwner owner, ResourceReleasePhase phase,
                          bool isCommit, bool isTopLevel);
void ResourceOwnerDelete(ResourceOwner owner);
ResourceOwner ResourceOwnerGetParent(ResourceOwner owner);
void ResourceOwnerNewParent(ResourceOwner owner, ResourceOwner newparent);
void RegisterResourceReleaseCallback(...);   /* 供扩展模块用 */
void UnregisterResourceReleaseCallback(...);
void CreateAuxProcessResourceOwner(void);
void ReleaseAuxProcessResources(bool isCommit);
```

- `ResourceOwnerCreate`（`resowner.c:429`）在 `TopMemoryContext` 分配对象并初始化
  13 个 `ResourceArray`；有父则链入父的 `firstchild` 链表。
- `ResourceOwnerDelete`（`resowner.c:762`）递归删除子孙，**要求资源已全部释放**
  （函数开头十余个 `Assert(...nitems == 0)`），最后 `pfree` 对象。
- `ResourceOwnerNewParent`（`resowner.c:828`）把 owner 从旧父摘除、挂到新父，
  这是 Portal 结束时把资源「过户」给事务的底层操作。

### 6.2 每种资源的三段式 API（以 buffer 为例，`resowner_private.h`）

每种资源都有 **Enlarge / Remember / Forget** 三个函数：

```c
void ResourceOwnerEnlargeBuffers(ResourceOwner owner);              /* 预留空间 */
void ResourceOwnerRememberBuffer(ResourceOwner owner, Buffer buffer);/* 登记 */
void ResourceOwnerForgetBuffer(ResourceOwner owner, Buffer buffer); /* 注销 */
```

使用协议（注释反复强调）：

> Caller must have previously done ResourceOwnerEnlargeBuffers()

即调用方必须**先 Enlarge，再真正获取资源，再 Remember**。为什么拆开？
`resowner.c:964` 的注释给出理由：

> This is separate from actually inserting an entry because if we run out of
> memory, it's critical to do so *before* acquiring the resource.

也就是说：`Enlarge` 会 `palloc`，可能失败并抛错——必须**在获取资源之前**做，
否则「资源已到手却登记不下」会变成无法收拾的泄漏。典型用法见
`bufmgr.c:642` `ResourceOwnerEnlargeBuffers(CurrentResourceOwner)` 紧接
`bufmgr.c:2311` `ResourceOwnerRememberBuffer(...)`。

### 6.3 `Forget` 的强校验

`ResourceOwnerForgetBuffer`（`resowner.c:994`）：

```c
if (!ResourceArrayRemove(&(owner->bufferarr), BufferGetDatum(buffer)))
    elog(ERROR, "buffer %d is not owned by resource owner %s", buffer, owner->name);
```

即「注销一个不属于自己的资源」会直接报错。同理 `ResourceOwnerForgetBufferIO`
用的是 `elog(PANIC, ...)`（IO 状态错乱更严重）。这既是双重释放/串号检测，
也强制「注销时的 `CurrentResourceOwner` 必须与登记时相同」（README 末尾专门说明）。

### 6.4 锁的 Remember/Forget 是「有损缓存」

`ResourceOwnerRememberLock`（`resowner.c:1050`）与 `ResourceOwnerForgetLock`
（`resowner.c:1070`）：只维护最多 15 个锁的**缓存**，溢出后（`nlocks > 15`）
直接不记。注释：

> the list of locks is only a lossy cache. ... when it overflows, we stop
> tracking locks. The point of only remembering only up to MAX_RESOWNER_LOCKS
> entries is that if a lot of locks are held, ResourceOwnerForgetLock doesn't
> need to scan through a large array.

这不会丢正确性，因为锁管理器自己的 local lock 哈希表有权威信息，
缓存仅在释放时作为「快速通道」；溢出处走 `locks = NULL` 让锁管理器全表扫描。

## 7. 三阶段释放：`ResourceReleasePhase`

释放**不是一次做完**，而是分三个阶段，由 `xact.c` 在三个阶段之间插入别的操作
（这正是拆开的原因，见 `resowner.c:461` 的注释）。枚举：

```c
RESOURCE_RELEASE_BEFORE_LOCKS
RESOURCE_RELEASE_LOCKS
RESOURCE_RELEASE_AFTER_LOCKS
```

`ResourceOwnerReleaseInternal`（`resowner.c:498`）按相位做不同的事。
**顺序至关重要**：`BEFORE_LOCKS` 必须先释放「对其他后端可见」的资源，
这样当本事务放掉某个别人可能在等的锁时，别人看到的是「本事务已完全退出」。

| 阶段 | 释放的资源 | 动作 | 典型泄漏告警 |
|---|---|---|---|
| **BEFORE_LOCKS** | buffer IO | `AbortBufferIO()`（提交时若还有则 PANIC） | `lost track of buffer IO` |
| | buffer pin | `ReleaseBuffer()` | `PrintBufferLeakWarning` |
| | relcache 引用 | `RelationClose()` | `PrintRelCacheLeakWarning` |
| | DSM 段 | `dsm_detach()` | `PrintDSMLeakWarning` |
| | JIT 上下文 | `jit_release_context()` | — |
| | cryptohash / HMAC | `pg_cryptohash_free()` / `pg_hmac_free()` | 各自 LeakWarning |
| **LOCKS** | 表锁 | 顶层：一次 `ProcReleaseLocks()` + `ReleasePredicateLocks()`；子层：逐个 retail，`isCommit` 时 `LockReassignCurrentOwner()` 转交父、否则 `LockReleaseCurrentOwner()` | — |
| **AFTER_LOCKS** | catcache 引用/列表 | `ReleaseCatCache()` / `ReleaseCatCacheList()` | LeakWarning |
| | plancache 引用 | `ReleaseCachedPlan()` | LeakWarning |
| | tupdesc 引用 | `DecrTupleDescRefCount()` | LeakWarning |
| | snapshot | `UnregisterSnapshot()` | LeakWarning |
| | 临时文件 | `FileClose()` | LeakWarning |

两个优化/要点：

1. **顶层锁一次性释放**（`resowner.c:609`）：`isTopLevel && owner ==
   TopTransactionResourceOwner` 时直接 `ProcReleaseLocks(isCommit)`，
   不必沿树逐锁 retail。
2. **子层锁在提交时「转交」而非释放**（`resowner.c:649`）：
   ```c
   if (isCommit)
       LockReassignCurrentOwner(locks, nlocks);   /* 过户给父 */
   else
       LockReleaseCurrentOwner(locks, nlocks);    /* 真正释放 */
   ```
   对应 README 的话：非错误情形下，锁即使由子事务或 Portal 取得，也应持有到
   事务结束，所以「释放子 owner」在 `isCommit` 时只是把所有权转给父。

释放完成后，每个相位末尾还会调用注册的 **`ResourceReleaseCallback`**
（`resowner.c:724`），让扩展模块（例如 `contrib/pgcrypto/openssl.c:193/787`
用 `RegisterResourceReleaseCallback` 注册 `digest_free_callback`/
`cipher_free_callback`）也能`扫描自己的数据结构`做清理。注释提醒：回调发生在
提交/中止之后，**只能做非关键清理**。

## 8. 与事务 / Portal 生命周期的结合

### 8.1 创建

- `StartTransaction()` → `AtStart_ResourceOwner()`（`xact.c:1199`）：
  创建 `TopTransactionResourceOwner`，并令
  `CurTransactionResourceOwner = CurrentResourceOwner = 它`。
- `StartSubTransaction()` → `AtSubStart_ResourceOwner()`（`xact.c:1251`）：
  子事务 owner 挂在父事务 owner 下。
- `CreatePortal()`（`portalmem.c:206`）：`portal->resowner = ResourceOwnerCreate(
  CurTransactionResourceOwner, "Portal")`，执行 Portal 时 `CurrentResourceOwner`
  指向它，于是执行期间拿到的资源都记在 Portal 名下。

> 这里与上一份笔记 `transaction-handling.md` 完全对应：
> `start_xact_command` 最终走到 `AtStart_ResourceOwner`（建 owner），
> `finish_xact_command` 最终在 `CommitTransaction`/`AbortTransaction` 里
> **三阶段 `ResourceOwnerRelease` + `ResourceOwnerDelete`**（拆 owner）。
> 即 **ResourceOwner 的生命周期与事务命令严格同生共死**。

### 8.2 提交/中止时的释放

`CommitTransaction()`（`xact.c:2152`）与 `AbortTransaction()`（`xact.c:2697`）
中都有：

```c
ResourceOwnerRelease(TopTransactionResourceOwner, RESOURCE_RELEASE_BEFORE_LOCKS, isCommit, true);
ResourceOwnerRelease(TopTransactionResourceOwner, RESOURCE_RELEASE_LOCKS,        isCommit, true);
ResourceOwnerRelease(TopTransactionResourceOwner, RESOURCE_RELEASE_AFTER_LOCKS,  isCommit, true);
...
CurrentResourceOwner = NULL;
ResourceOwnerDelete(TopTransactionResourceOwner);
```

在 LOCKS 相位之前/之间，xact.c 还穿插了 `AtEOXact_Buffers`、`AtEOXact_Inval`
（让目录改动在放锁前对其他后端可见）等操作——这正是「相位」设计的用意。

### 8.3 Portal 的收尾：资源「过户」

`PortalDrop()`（`portalmem.c:469`，释放逻辑在 558–574）：

```c
if (portal->resowner &&
    (!isTopCommit || portal->status == PORTAL_FAILED))
{
    bool isCommit = (portal->status != PORTAL_FAILED);
    ResourceOwnerRelease(portal->resowner, RESOURCE_RELEASE_BEFORE_LOCKS, isCommit, false);
    ResourceOwnerRelease(portal->resowner, RESOURCE_RELEASE_LOCKS,        isCommit, false);
    ResourceOwnerRelease(portal->resowner, RESOURCE_RELEASE_AFTER_LOCKS,  isCommit, false);
    ResourceOwnerDelete(portal->resowner);
}
portal->resowner = NULL;
```

源码注释解释了三种情形（概括）：

- **普通 Portal drop**（非 FAILED）：`isCommit = true`，锁在 LOCKS 相位被
  `LockReassignCurrentOwner` **过户给父（事务 owner）**，不真正释放——因为
  Portal 的 owner 本来就是 `CurTransactionResourceOwner` 的子节点，锁就此归入事务。
- **顶层提交**：跳过此段，交给事务级的统一释放流程处理（Portal owner 是子树，递归覆盖）。
  只是若 Portal 已 FAILED，为避免泄漏告警才在此清理。
- **事务中止**：`portal->resowner` 已被置 NULL，资源早在事务 abort 中释放。

此外 `AtAbort_Portals` / `AtCommit_Portals` 等事务收尾钩子会成批 drop portal，
与这里配合完成整棵树的自底向上清理。

## 9. 与 MemoryContext 的对比

两者都是「树形 + 父释放带动子孙」，但**刻意不合并**（README 明说 usage patterns
差异大，合并无益）。对照：

| 维度 | `MemoryContext` | `ResourceOwner` |
|---|---|---|
| 管理对象 | `palloc` 出来的内存 | buffer pin / 锁 / 引用 / 文件等**非内存资源** |
| 归属变量 | `CurrentMemoryContext` | `CurrentResourceOwner` |
| 释放方式 | `MemoryContextDelete` 级联删子 | `ResourceOwnerRelease`（三相位）+ `ResourceOwnerDelete` |
| 资源登记 | 分配即登记（自动） | 需**显式** Enlarge/Remember/Forget |
| 错误时 | delete 会释放全部内存 | release 会释放全部资源 |
| 结构存放 | 自身在 `TopMemoryContext` | 自身也在 `TopMemoryContext` |

生命周期的呼应：`StartTransaction()` 里 `AtStart_Memory()` 建
`TopTransactionContext`、`AtStart_ResourceOwner()` 建
`TopTransactionResourceOwner`；`CommitTransaction()` 里
`AtCommit_Memory()` 删前者、`ResourceOwnerDelete()` 删后者。二者同生共死。

## 10. 错误恢复中 ResourceOwner 的关键价值

这是整个机制存在的意义。一次查询中途 `ereport(ERROR)` 时的路径：

```
某处 ereport(ERROR)
  └─ longjmp 回到 PostgresMain 的顶层错误块        (postgres.c:4352)
        └─ AbortCurrentTransaction()               (xact.c:3305)
              └─ AbortTransaction()                (xact.c:2697)
                    ├─ ResourceOwnerRelease(Top, BEFORE_LOCKS, isCommit=false, true)
                    │     └─ ReleaseBuffer / RelationClose / dsm_detach / ... 全自动回收
                    ├─ ResourceOwnerRelease(Top, LOCKS, false, true)
                    │     └─ ProcReleaseLocks(false) —— 释放全部锁
                    └─ ResourceOwnerRelease(Top, AFTER_LOCKS, false, true)
                          └─ ReleaseCatCache / UnregisterSnapshot / FileClose / ...
              └─ CleanupTransaction()              (xact.c:2888)
                    └─ ResourceOwnerDelete(TopTransactionResourceOwner)
```

**执行器不需要为每个错误分支手写清理**：只要资源在获取时登记过，abort 时就会被
统一收走。`isCommit = false` 时不做泄漏告警（本就是预期内的残留）。

## 11. 调试与泄漏检测能力

- **正常结束仍残留** → `isCommit = true`，各 `Print*LeakWarning()` 打印告警
  （如 `PrintBufferLeakWarning`、`PrintRelCacheLeakWarning`、`PrintFileLeakWarning`…），
  帮助发现「忘记 Forget」的 bug。
- **`ResourceOwnerDelete` 断言**：删除前断言全部 `ResourceArray` 为空、
  `nlocks` 为 0 或已溢出（`resowner.c:768`），把泄漏变成 assert 失败。
- **`ResourceArrayRemove` 失败即报错**：注销不属于自己的资源会
  `elog(ERROR)`，能抓出双重释放或 owner 串号。
- **buffer IO 丢失** → `elog(PANIC)`（`resowner.c:536`），因为提交时还有
  进行中的 IO 属于致命状态错乱。
- `ResourceOwnerReleaseInternal` 期间会把 `CurrentResourceOwner` 临时切到
  正在释放的 owner（`resowner.c:517`），保证 `ReleaseBuffer` 等能找到正确的
  登记方；结束时恢复。

## 12. `AuxProcessResourceOwner`：辅助进程的 owner

checkpointer、bgwriter、walwriter、autovacuum、walsender、logical worker 等
**辅助进程没有事务**，却也偶尔需要获取 buffer pin 等资源
（`postinit.c` / `xlog.c` / `autovacuum.c` / `auxprocess.c` / `walsender.c` /
`worker.c` 都用到）。为此：

- `CreateAuxProcessResourceOwner()`（`resowner.c:914`）为当前进程建一个
  顶层 owner 并设为 `CurrentResourceOwner`，同时注册 `on_shmem_exit`
  回调 `ReleaseAuxProcessResourcesCallback`。
- `ReleaseAuxProcessResources(isCommit)`（`resowner.c:934`）同样走三相位释放
  （注释说目前实际会释放的基本只有 buffer pin）。

## 13. 一图流总结

```
                          获取资源时
   业务代码 ──Enlarge──▶ ResourceOwner(当前 CurrentResourceOwner) ◀──Remember──┐
                                                                               │
   正常释放 ──Forget────────────────────────────────────────────────────────┘
                                                                              
   出错 longjmp ──▶ 事务/Portal 结束
                       └─ ResourceOwnerRelease × 3 相位  → 逐个 Release*
                       └─ ResourceOwnerDelete            → 拆掉整棵 owner 树
```

一句话：**ResourceOwner = 查询生命周期资源的「记账本 + 兜底回收器」**，
平时记账，出错时统一收尸，用树形结构把「事务/子事务/Portal」的清理边界对齐到
与内存上下文一致的模型上。

## 14. 速查表

| 内容 | 位置 |
|---|---|
| 公开头（类型、全局变量、相位枚举、API） | `src/include/utils/resowner.h` |
| 各类 Remember/Forget 私有声明 | `src/include/utils/resowner_private.h` |
| 设计文档（README） | `src/backend/utils/resowner/README` |
| `ResourceOwnerData` 结构 | `resowner.c:115` |
| `ResourceArray` 及阈值常量 | `resowner.c:65` 及 `:79`/`:84` |
| `MAX_RESOWNER_LOCKS = 15` | `resowner.c:110` |
| `ResourceOwnerCreate` | `resowner.c:429` |
| `ResourceOwnerRelease` / `ResourceOwnerReleaseInternal` | `resowner.c:488` / `:498` |
| `ResourceOwnerReleaseAllPlanCacheRefs` | `resowner.c:743` |
| `ResourceOwnerDelete` | `resowner.c:762` |
| `ResourceOwnerNewParent` / `ResourceOwnerGetParent` | `resowner.c:828` / `:819` |
| `RegisterResourceReleaseCallback` | `resowner.c:876` |
| `CreateAuxProcessResourceOwner` / `ReleaseAuxProcessResources` | `resowner.c:914` / `:934` |
| `ResourceOwnerEnlargeBuffers` / `RememberBuffer` / `ForgetBuffer` | `resowner.c:972` / `:985` / `:994` |
| `ResourceOwnerRememberLock` / `ForgetLock` | `resowner.c:1050` / `:1070` |
| 顶层事务 owner 创建 | `xact.c:1211`（`AtStart_ResourceOwner`） |
| 子事务 owner 创建 | `xact.c:1262`（`AtSubStart_ResourceOwner`） |
| 提交/中止时的三相位释放 | `xact.c:2152` `CommitTransaction` / `xact.c:2697` `AbortTransaction` |
| Portal owner 创建 | `portalmem.c:206`（`CreatePortal`） |
| Portal owner 释放/过户 | `portalmem.c:563`（`PortalDrop`） |
| 调用方示例（buffer pin） | `bufmgr.c:642` / `:2311` / `:2391` |
| 扩展注册释放回调示例 | `contrib/pgcrypto/openssl.c:193` / `:787` |