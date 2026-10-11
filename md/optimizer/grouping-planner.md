# PostgreSQL 优化器 grouping_planner 函数分析

`grouping_planner` 位于 `planner.c:1272`，是优化器在 `query_planner` 完成扫描/连接路径搜索之后，处理 **GROUP BY、聚合、窗口函数、DISTINCT、ORDER BY、LIMIT** 等高层操作的核心函数。

## 整体职责

`query_planner` 只处理 FROM/WHERE 部分，返回一个包含所有扫描+连接路径的 `RelOptInfo`。`grouping_planner` 在此基础上，通过**上层关系（upperrel）逐步叠加**的方式，依次构建每个处理阶段的候选路径，最终将最优路径集合挂载到 `(UPPERREL_FINAL, NULL)` 上。

## 函数签名

```c
static void grouping_planner(PlannerInfo *root, double tuple_fraction)
```

参数 `tuple_fraction`：
- `0`：期望获取所有元组（正常情况）
- `0 < tuple_fraction < 1`：期望获取可用元组的给定比例
- `tuple_fraction >= 1`：绝对元组数量（即 LIMIT 规格）

输出不直接返回，而是挂载到 `root` 的 `(UPPERREL_FINAL, NULL)` upperrel 上。

---

## 主要阶段（10个阶段）

### 阶段1：LIMIT/OFFSET 预处理（`planner.c:1288-1303`）

```c
if (parse->limitCount || parse->limitOffset)
    tuple_fraction = preprocess_limit(root, tuple_fraction, &offset_est, &count_est);
```

- 根据 LIMIT/OFFSET 调整 `tuple_fraction`，使后续代价估算能反映"只需返回部分行"的情况
- 计算 `limit_tuples = count_est + offset_est`，用于有界排序的代价估算
- 将 `tuple_fraction` 存入 `root->tuple_fraction`，供下层例程使用

### 阶段2：SET 操作分支（`planner.c:1305-1371`）

如果查询是 `UNION/INTERSECT/EXCEPT`：

```c
if (parse->setOperations) {
    current_rel = plan_set_operations(root);
    ...
}
```

- 调用 `plan_set_operations()` 构建集合操作的路径
- 结果只需要顶层排序和 LIMIT，不需要分组/聚合/窗口
- 禁止 `FOR [KEY] UPDATE/SHARE`（parser 已检查，此处断言）
- 构建排序路径键 `root->sort_pathkeys` 后直接跳到后面的 ORDER BY 阶段

### 阶段3：常规查询预处理（`planner.c:1373-1464`）

这是最复杂的部分，按顺序完成：

**3a. 分组预处理**
- `preprocess_grouping_sets()` — 处理 `GROUPING SETS/ROLLUP/CUBE`，展开并组织为 rollup 列表
- `preprocess_groupclause()` + `remove_useless_groupby_columns()` — 处理普通 `GROUP BY`，去除冗余列

**3b. 目标列表预处理**
- `preprocess_targetlist()` — 规范化目标列表，维护 `processed_tlist`

**3c. 聚合预处理**
- `preprocess_aggrefs()` — 标记聚合函数的 `aggtranstype`，检测可共享过渡状态的聚合、重复聚合
- `preprocess_minmax_aggregates()` — 优化 MIN/MAX 聚合（利用索引避免全表扫描）

**3d. 窗口函数预处理**
- `find_window_functions()` — 在目标列表中定位窗口函数
- `optimize_window_clauses()` — 优化窗口子句（如合并相同 PARTITION BY/ORDER BY 的窗口）
- `select_active_windows()` — 选择活动窗口，按执行顺序排列

**3e. 调用 `query_planner()`**（`planner.c:1494`）

```c
current_rel = query_planner(root, standard_qp_callback, &qp_extra);
```

传入 `standard_qp_callback` 和 `qp_extra`（包含 `activeWindows`、`gset_data`），完成 FROM/WHERE 的路径搜索，同时计算 `query_pathkeys`、`group_pathkeys`、`window_pathkeys`、`distinct_pathkeys` 等规范化路径键。

**3f. LIMIT 限制判断**（`planner.c:1467-1481`）

如果查询存在分组、聚合、窗口函数、DISTINCT 或 SRF，则 `root->limit_tuples = -1.0`，即不对 `query_planner` 的结果施加行数限制。因为这些上层操作会改变行数，LIMIT 不能直接下推。

### 阶段4：多层 PathTarget 构建（`planner.c:1496-1607`）

这是优化器路径目标链的关键设计。查询结果要经过多层处理，每层需要不同的输出列：

```
scan/join → grouping → window → sort → final
```

从后往前推导每层需要的目标列：

| 目标变量 | 含义 |
|----------|------|
| `final_target` | 最终查询输出所需的全部列 |
| `sort_input_target` | 排序步骤前的目标列（可能延迟 SRF 计算到排序之后） |
| `grouping_target` | 分组/聚合步骤前的目标列（窗口函数需要的列） |
| `scanjoin_target` | 扫描/连接阶段的目标列（分组需要的最小列集合） |

如果有 SRF（Set-Returning Functions），还会通过 `split_pathtarget_at_srfs()` 将每个目标拆分为 SRF-free 版本 + 后续投影步骤列表（如 `final_targets`、`sort_input_targets` 等），以便在执行时按需分离投影。

### 阶段5：扫描/连接目标应用（`planner.c:1609-1615`）

```c
apply_scanjoin_target_to_paths(root, current_rel, scanjoin_targets, ...);
```

将计算出的 `scanjoin_target` 应用到 `query_planner` 返回的已有扫描/连接路径上，可能引入额外的投影步骤以确保路径输出正确的列。

### 阶段6：分组/聚合路径生成（`planner.c:1636-1648`）

```c
if (have_grouping)
    current_rel = create_grouping_paths(root, current_rel, grouping_target, ...);
```

`have_grouping = groupClause || groupingSets || hasAggs || hasHavingQual`

调用 `create_grouping_paths()` 生成各种分组策略的路径，创建 `UPPERREL_GROUP_AGG` 关系，策略包括：
- HashAgg — 哈希聚合
- Sort + GroupAgg — 排序后流式聚合
- MixedAgg — 混合策略

如果 `grouping_target` 包含 SRF，调用 `adjust_paths_for_srfs()` 追加 SRF 投影步骤。

### 阶段7：窗口函数路径生成（`planner.c:1654-1668`）

```c
if (activeWindows)
    current_rel = create_window_paths(root, current_rel, grouping_target, ...);
```

在分组结果之上构建窗口处理路径，创建 `UPPERREL_WINDOW` 关系。窗口函数需要按 PARTITION BY 分组并按 ORDER BY 排序，路径可以是预排序的（利用已有 pathkey）或需要显式 Sort。

### 阶段8：DISTINCT 路径生成（`planner.c:1674-1678`）

```c
if (parse->distinctClause)
    current_rel = create_distinct_paths(root, current_rel);
```

在窗口结果之上构建去重路径，生成 `UPPERREL_DISTINCT` 关系，策略包括：
- Unique（基于排序去重）
- Hash（哈希去重）

### 阶段9：ORDER BY 路径生成（`planner.c:1688-1701`）

```c
if (parse->sortClause)
    current_rel = create_ordered_paths(root, current_rel, final_target, ...);
```

在去重结果之上构建排序路径，创建 `UPPERREL_ORDERED` 关系。如果已有路径恰好具有正确的 pathkey，则无需额外 Sort；否则插入 Sort 节点。如果存在被延迟的 SRF，不应用 `limit_tuples` 限制。

### 阶段10：最终关系构建（`planner.c:1706-1995`）

构建 `UPPERREL_FINAL` 关系，将经过上述所有阶段的路径进行最终加工。

**10a. 并行安全与 FDW 继承**

```c
if (current_rel->consider_parallel && is_parallel_safe(limitClause))
    final_rel->consider_parallel = true;

final_rel->serverid = current_rel->serverid;  // FDW 信息继承
```

**10b. 遍历 current_rel 的路径列表，依次叠加最终节点**

```c
foreach(lc, current_rel->pathlist) {
    path = ...;
    if (parse->rowMarks)    path = create_lockrows_path(...);    // FOR UPDATE/SHARE
    if (limit_needed())     path = create_limit_path(...);       // LIMIT/OFFSET
    if (commandType != SELECT) path = create_modifytable_path(...); // INSERT/UPDATE/DELETE/MERGE
    add_path(final_rel, path);
}
```

对于 DML 操作（INSERT/UPDATE/DELETE/MERGE），需要处理：
- 继承表的 leaf 子表展开
- 分区裁剪后空结果的处理
- 列号映射调整（继承/分区子表列号可能不同）
- WITH CHECK OPTIONS、RETURNING 列表的适配

**10c. 并行路径添加**

```c
if (final_rel->consider_parallel && root->query_level > 1 && !limit_needed())
    foreach(lc, current_rel->partial_pathlist)
        add_partial_path(final_rel, partial_path);
```

如果查询支持并行且外层可能利用部分路径，则将 partial_pathlist 也加入 final_rel。

**10d. FDW 与插件钩子扩展**

```c
if (final_rel->fdwroutine && fdwroutine->GetForeignUpperPaths)
    fdwroutine->GetForeignUpperPaths(root, UPPERREL_FINAL, ...);

if (create_upper_paths_hook)
    (*create_upper_paths_hook)(root, UPPERREL_FINAL, ...);
```

允许外部 FDW 和扩展插件注入自定义路径。

---

## 数据流概览

```
query_planner() 返回 scan/join 路径 (current_rel)
        │
        ▼
    ┌─ have_grouping? ──► create_grouping_paths()  (UPPERREL_GROUP_AGG)
    │                           │
    │     ▼              (可能有 SRF 调整)
    │
    └─ activeWindows? ──► create_window_paths()     (UPPERREL_WINDOW)
    │                           │
    │     ▼
    └─ distinctClause? ─► create_distinct_paths()    (UPPERREL_DISTINCT)
    │                           │
    │     ▼
    └─ sortClause?   ──► create_ordered_paths()      (UPPERREL_ORDERED)
                            │
                            ▼
                    fetch_upper_rel(FINAL)
                    ├── LockRows (rowMarks)
                    ├── Limit (LIMIT/OFFSET)
                    └── ModifyTable (DML)
                            │
                            ▼
                    UPPERREL_FINAL (路径集合)
```

## 与上下游的关系

- **上游**：`subquery_planner`（`planner.c:1063`）在完成预处理（子查询拉平、表达式规范化、外连接简化等）后调用 `grouping_planner`
- **下游**：`subquery_planner` 在 `grouping_planner` 返回后，通过 `fetch_upper_rel(root, UPPERREL_FINAL, NULL)` 获取最优路径，再由 `standard_planner` 调用 `create_plan()` 将其转换为可执行的 Plan 树

## 关键设计原则

1. **upperrel 链式构建**：每个上层操作（分组→窗口→去重→排序）都创建一个新的 upperrel，路径从下往上逐层叠加，每层可选择不同实现策略
2. **PathTarget 逆推**：从最终输出列出发，逆推每一层需要输出的列，避免不必要的列传递（减少内存和计算开销）
3. **SRF 延迟求值**：Set-Returning Functions 的计算被尽可能地延迟到排序/去重之后，避免影响行数和排序语义
4. **可扩展性**：通过 FDW 钩子和 `create_upper_paths_hook` 允许外部扩展注入自定义路径
