# Path 数据结构分析

`Path` 是 PostgreSQL 查询优化器中表示**候选执行策略**的核心数据结构。每个 `RelOptInfo` 维护一个 `pathlist`，保存所有可行的 Path，最终由 `set_cheapest()` 选出最优者并转换为 `Plan` 树。定义在 `src/include/nodes/pathnodes.h:1593-1640`。

## 一、基类 Path 结构体

```c
typedef struct Path
{
    NodeTag     type;               /* T_Path, T_IndexPath, T_NestPath, ... */
    NodeTag     pathtype;           /* 对应 Plan 的 NodeTag: T_SeqScan, T_IndexScan, ... */
    RelOptInfo *parent;             /* 该 Path 扫描/构建的关系 */
    PathTarget *pathtarget;         /* 输出的列表达式 + 代价/宽度 */
    ParamPathInfo *param_info;      /* 参数化信息（NULL 表示非参数化） */
    bool        parallel_aware;     /* 是否并行感知 */
    bool        parallel_safe;      /* 是否并行安全 */
    int         parallel_workers;   /* 并行 worker 数量 */
    Cardinality rows;               /* 估计输出行数 */
    Cost        startup_cost;       /* 返回第一个元组前的代价 */
    Cost        total_cost;         /* 返回全部元组的总代价 */
    List       *pathkeys;           /* 输出排序顺序（PathKey 列表） */
} Path;
```

**核心字段解读**：

| 字段 | 含义 |
|---|---|
| `type` | 标识具体 Path 子类型（T_IndexPath、T_NestPath 等） |
| `pathtype` | 该 Path 可生成的 Plan 节点类型（T_IndexScan、T_NestLoop 等） |
| `parent` | 指向所属 `RelOptInfo`，建立 Path ↔ Relation 的双向关联 |
| `param_info` | 参数化路径非 NULL，表示该 Path 依赖外部关系提供的参数 |
| `startup_cost` | 启动代价——执行到能返回第一行前的工作量 |
| `total_cost` | 总代价——返回所有行的累计工作量 |
| `pathkeys` | 输出排序顺序，NIL 表示无序；索引扫描天然有序，SeqScan 无序 |

**宏定义**：
```c
#define PATH_REQ_OUTER(path) \
    ((path)->param_info ? (path)->param_info->ppi_req_outer : (Relids) NULL)
```
快速获取 Path 依赖的外部关系集合。

## 二、Path 子类全览

### 2.1 扫描类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **IndexPath** | `indexinfo`, `indexclauses`, `indexorderbys`, `indexscandir`, `indextotalcost`, `indexselectivity` | 索引扫描 / Index-Only Scan |
| **BitmapHeapPath** | `bitmapqual` (指向 IndexPath/BitmapAndPath/BitmapOrPath) | 位图堆扫描 |
| **BitmapAndPath** | `bitmapquals`, `bitmapselectivity` | 位图 AND 组合 |
| **BitmapOrPath** | `bitmapquals`, `bitmapselectivity` | 位图 OR 组合 |
| **TidPath** | `tidquals` | CTID 精确扫描 |
| **TidRangePath** | `tidrangequals` | CTID 范围扫描 |
| **SubqueryScanPath** | `subpath` | 子查询扫描 |
| **ForeignPath** | `fdw_outerpath`, `fdw_private` | 外部表扫描（FDW） |
| **CustomPath** | `custom_paths`, `custom_private`, `methods` | 自定义扫描（扩展接口） |

### 2.2  Append/Merge 类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **AppendPath** | `subpaths`, `first_partial_path`, `limit_tuples` | 顺序执行多个子计划（继承/UNION ALL） |
| **MergeAppendPath** | `subpaths`, `limit_tuples` | 合并多个有序子计划的结果 |

### 2.3 连接类 Path

**JoinPath**（抽象基类）：
```c
typedef struct JoinPath {
    Path        path;
    JoinType    jointype;           /* INNER, LEFT, RIGHT, FULL, SEMI, ANTI */
    bool        inner_unique;       /* 内表是否唯一（可优化为最多匹配一行） */
    Path       *outerjoinpath;      /* 外连接路径 */
    Path       *innerjoinpath;      /* 内连接路径 */
    List       *joinrestrictinfo;   /* 连接过滤条件 */
} JoinPath;
```

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **NestPath** | （无，直接使用 JoinPath） | 嵌套循环连接 |
| **MergePath** | `path_mergeclauses`, `outersortkeys`, `innersortkeys`, `skip_mark_restore`, `materialize_inner` | 归并连接 |
| **HashPath** | `path_hashclauses`, `num_batches`, `inner_rows_total` | 哈希连接 |

### 2.4 并行类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **GatherPath** | `subpath`, `single_copy`, `num_workers` | 收集并行 worker 结果 |
| **GatherMergePath** | `subpath`, `num_workers` | 有序收集并行 worker 结果 |

### 2.5 排序/投影类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **SortPath** | `subpath` | 显式排序 |
| **IncrementalSortPath** | `spath`, `nPresortedCols` | 增量排序（利用部分有序性） |
| **ProjectionPath** | `subpath`, `dummypp` | 投影（列计算） |
| **ProjectSetPath** | `subpath` | 集合返回函数（SRF） |

### 2.6 分组/聚合类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **GroupPath** | `subpath`, `groupClause`, `qual` | 分组（假设输入已排序） |
| **AggPath** | `subpath`, `aggstrategy`, `aggsplit`, `numGroups`, `transitionSpace`, `groupClause`, `qual` | 聚合（HashAgg / SortedAgg） |
| **GroupingSetsPath** | `subpath`, `aggstrategy`, `rollups`, `qual`, `transitionSpace` | GROUPING SETS / ROLLUP / CUBE |
| **UpperUniquePath** | `subpath`, `numkeys` | 去重（Unique） |
| **MinMaxAggPath** | `mmaggregates`, `quals` | MIN/MAX 索引优化 |
| **WindowAggPath** | `subpath`, `winclause`, `qual`, `topwindow` | 窗口函数 |

### 2.7 特殊类 Path

| 结构体 | 专属字段 | 用途 |
|---|---|---|
| **UniquePath** | `subpath`, `umethod`, `in_operators`, `uniq_exprs` | 重复消除（NOOP/HASH/SORT） |
| **MaterialPath** | `subpath` | 物化（缓存中间结果） |
| **MemoizePath** | `subpath`, `hash_operators`, `param_exprs`, `singlerow`, `binary_mode`, `calls`, `est_entries` | 参数化路径缓存 |
| **GroupResultPath** | `quals` | 退化 GROUP BY（无输入，输出单行） |
| **SetOpPath** | `subpath`, `cmd`, `strategy`, `distinctList`, `flagColIdx`, `firstFlag`, `numGroups` | INTERSECT / EXCEPT |
| **RecursiveUnionPath** | `leftpath`, `rightpath`, `distinctList`, `wtParam`, `numGroups` | 递归 UNION（CTE） |
| **ModifyTablePath** | `subpath`, `operation`, `resultRelations`, `onconflict`, `returningLists`, ... | INSERT / UPDATE / DELETE / MERGE |
| **LimitPath** | `subpath`, `limitOffset`, `limitCount`, `limitOption` | LIMIT / OFFSET |
| **LockRowsPath** | `subpath`, `rowMarks`, `epqParam` | FOR UPDATE / SHARE 行锁 |

## 三、相关结构

### 3.1 PathKey（排序顺序）

```c
typedef struct PathKey {
    EquivalenceClass *pk_eclass;   /* 被排序的等价类 */
    Oid         pk_opfamily;       /* 定义排序的 btree operator family */
    int         pk_strategy;       /* ASC (BTLess) 或 DESC (BTGreater) */
    bool        pk_nulls_first;    /* NULL 是否排在前面 */
} PathKey;
```

PathKeys 是一个 `List *`，每个 PathKey 代表一个排序维度。空列表（NIL）表示无序。PathKey 经过规范化（canonical），等价比较简化为指针比较：

```c
compare_pathkeys(keys1, keys2) 返回值：
  PATHKEYS_EQUAL      — 完全相同
  PATHKEYS_BETTER1    — keys1 是 keys2 的超集（更有序）
  PATHKEYS_BETTER2    — keys2 是 keys1 的超集
  PATHKEYS_DIFFERENT  — 互不包含
```

### 3.2 ParamPathInfo（参数化）

```c
typedef struct ParamPathInfo {
    Relids      ppi_req_outer;    /* 提供参数的外部关系集合 */
    Cardinality ppi_rows;         /* 该参数化下的估计行数 */
    List       *ppi_clauses;      /* 外部关系提供的连接条件 */
    Bitmapset  *ppi_serials;      /* 强制执行的 rinfo_serial 集合 */
} ParamPathInfo;
```

**参数化路径**指扫描依赖其他关系提供的值，典型场景是嵌套循环的内侧索引扫描——外层每输出一行，内层用该行的列值作为索引参数执行一次。

关键规则：
- 相同 `ppi_req_outer` 的所有路径共享同一个 `ParamPathInfo`
- `ppi_clauses` 仅对基表路径填充，连接路径的条件在 `joinrestrictinfo` 中
- 参数化路径只能用作嵌套循环（或类似连接）的内侧

### 3.3 PathTarget（输出列）

```c
typedef struct PathTarget {
    List       *exprs;                    /* 需要计算的表达式列表 */
    Index      *sortgrouprefs;            /* 对应的 sort/group refnos */
    QualCost    cost;                     /* 计算表达式的代价 */
    int         width;                    /* 估计的平均结果宽度 */
    VolatileFunctionStatus has_volatile_expr;
} PathTarget;
```

每个 `RelOptInfo` 有默认的 `reltarget`，大多数 Path 直接引用它。但某些 Path（如 Index-Only Scan 返回索引表达式）会有自定义 PathTarget。上层关系（GROUP BY、ORDER BY 等）总是有自定义 PathTarget。

## 四、Path 的生命周期

### 阶段 1：创建

各类 `create_*_path()` 函数在 `src/backend/optimizer/util/pathnode.c` 中：

```c
// 顺序扫描路径
Path *create_seqscan_path(PlannerInfo *root, RelOptInfo *rel,
                          Relids required_outer, int parallel_workers)
{
    Path *pathnode = makeNode(Path);
    pathnode->pathtype = T_SeqScan;
    pathnode->parent = rel;
    pathnode->pathtarget = rel->reltarget;
    pathnode->param_info = get_baserel_parampathinfo(root, rel, required_outer);
    pathnode->parallel_safe = rel->consider_parallel;
    pathnode->parallel_workers = parallel_workers;
    pathnode->pathkeys = NIL;                    // SeqScan 无序
    cost_seqscan(pathnode, root, rel, ...);      // 代价估算
    return pathnode;
}
```

```c
// 索引扫描路径
IndexPath *create_index_path(PlannerInfo *root, IndexOptInfo *index,
                             List *indexclauses, List *indexorderbys, ...)
{
    IndexPath *pathnode = makeNode(IndexPath);
    pathnode->path.pathtype = indexonly ? T_IndexOnlyScan : T_IndexScan;
    pathnode->path.parent = rel;
    pathnode->path.pathkeys = pathkeys;          // 索引天然有序
    pathnode->indexinfo = index;
    pathnode->indexclauses = indexclauses;
    cost_index(pathnode, root, loop_count, ...); // 代价估算
    return pathnode;
}
```

连接路径（`create_nestloop_path`、`create_mergejoin_path`、`create_hashjoin_path`）模式类似，但会设置 `outerjoinpath` / `innerjoinpath` 和 `joinrestrictinfo`。

### 阶段 2：代价估算

每个 Path 类型对应 `cost_*` / `final_cost_*` 函数（`src/backend/optimizer/path/costsize.c`），计算：
- `startup_cost`：启动代价
- `total_cost`：总代价
- `rows`：估计行数

### 阶段 3：添加到 pathlist（add_path）

创建的 Path 通过 `add_path()` 加入 `RelOptInfo->pathlist`，触发**支配淘汰**逻辑（见第五节）。

### 阶段 4：选择最优路径（set_cheapest）

所有路径生成完毕后，`set_cheapest()` 遍历 `pathlist`，选出：
- `cheapest_startup_path` — 启动代价最低的非参数化路径
- `cheapest_total_path` — 总代价最低的非参数化路径（若无则选最小参数化的最优者）
- `cheapest_parameterized_paths` — 每种参数化下的最优路径
- `cheapest_unique_path` — 按需计算的去重路径

### 阶段 5：转换为 Plan

`create_plan()`（`src/backend/optimizer/plan/createplan.c`）从最优 Path 开始递归转换：

```
create_plan_recurse(Path *path)
├── T_Path          → create_scan_plan()   → SeqScan / SampleScan
├── T_IndexPath     →                      → IndexScan / IndexOnlyScan
├── T_BitmapHeapPath →                     → BitmapHeapScan
├── T_NestPath      → create_join_plan()   → NestLoop
├── T_MergePath     →                      → MergeJoin (+ Sort / Material 子节点)
├── T_HashPath      →                      → HashJoin
├── T_AppendPath    →                      → Append
├── T_SortPath      → create_sort_plan()   → Sort
├── T_AggPath       → create_agg_plan()    → Aggregate
├── T_LimitPath     → create_limit_plan()  → Limit
└── ...
```

## 五、add_path() 支配淘汰逻辑

`add_path()`（`pathnode.c:422-624`）是优化器路径剪枝的核心算法。将新 Path 与 `pathlist` 中已有 Path 逐一比较，决定**是否保留新 Path** 以及**是否淘汰旧 Path**。

### 5.1 模糊代价比较

使用 `STD_FUZZ_FACTOR = 1.01`（1% 容差）：
```
COSTS_EQUAL     — 启动和总代价均在 1% 内
COSTS_BETTER1   — path1 在某维度上更优，另一维度不差
COSTS_BETTER2   — path2 支配 path1
COSTS_DIFFERENT — 一个启动代价低，一个总代价低（互不支配）
```

### 5.2 支配矩阵

新 Path 支配旧 Path 需满足**全部维度**不劣：

| 维度 | 新 Path 必须... |
|---|---|
| 代价 | 启动和总代价均不差（模糊比较） |
| Pathkeys | 超集或相等（更有序或相同） |
| 参数化 | 子集或相等（依赖更少外部关系） |
| 行数 | 更少或相等 |
| 并行安全 | 更高或相等 |

### 5.3 算法流程

```
for each old_path in pathlist:
    1. 模糊代价比较
       → COSTS_DIFFERENT: 互不支配，跳过后续比较，保留两者
    2. 比较 pathkeys
       → PATHKEYS_DIFFERENT: 互不包含，保留两者
    3. 比较参数化 (bms_subset_compare)
       → BMS_DIFFERENT: 参数化不可比，保留两者
       → BMS_SUBSET2: 旧路径参数化更好，新路径被支配
       → BMS_SUBSET1: 新路径参数化更好
    4. 全部相等时决胜局：
       并行安全 → 行数 → 超精细代价比较 (1.0000000001) → 保留旧的

    if accept_new == false:  break  // 新路径已被支配，提前退出
    if remove_old == true:   标记删除
```

**关键优化**：`pathlist` 按 `total_cost` 排序（最低在前），允许提前终止。

### 5.4 特殊处理

- **IndexPath 回收**：被删除的 Path 内存会被 `pfree()`，但 `IndexPath` 除外——因为它可能同时作为独立路径存在于 `pathlist` 中，又被 `BitmapHeapPath` 的 `bitmapqual` 引用。
- **add_path_precheck()**：快速预筛选，只比较代价和 pathkeys（跳过行估算），返回 `false` 则跳过完整 Path 创建。
- **add_partial_path()**：并行部分路径的简化版本，不考虑参数化和启动代价。

## 六、Path 与 RelOptInfo、Plan 的关系

```
RelOptInfo
├── pathlist ───→ [Path₁, Path₂, Path₃, ...]   （所有候选策略）
├── cheapest_startup_path ──→ 最优 Path（启动代价）
├── cheapest_total_path ────→ 最优 Path（总代价）
└── cheapest_parameterized_paths ──→ [ParamPath₁, ParamPath₂, ...]

Path.parent ───→ 回指 RelOptInfo
Path.pathtype ───→ 对应 Plan 的 NodeTag

    Path 树                                Plan 树
    ┌──────────────┐                       ┌──────────────┐
    │  NestPath    │                       │  NestLoop    │
    │  outer ──────┼─── create_plan() ────→│  outer ──────┤
    │  inner ──────┤                       │  inner ──────┤
    └──────┬───────┘                       └──────┬───────┘
           │                                      │
    ┌──────┴───────┐                       ┌──────┴───────┐
    │  IndexPath   │                       │  IndexScan   │
    └──────────────┘                       └──────────────┘
```
