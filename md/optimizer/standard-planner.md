# PostgreSQL 优化器 standard_planner 函数分析

`standard_planner` 函数位于 `src/backend/optimizer/plan/planner.c:286`，是 PostgreSQL 优化器的核心实现。

## 函数签名

```c
PlannedStmt *standard_planner(Query *parse, const char *query_string,
                              int cursorOptions, ParamListInfo boundParams)
```

---

## 主要逻辑（10个阶段）

### 1. 构建全局规划状态 (`PlannerGlobal`)

```c
glob = makeNode(PlannerGlobal);
```

在 `planner.c:305-323` 初始化 `PlannerGlobal` 结构体，用于跨越所有子查询层级共享的全局状态，包括：子计划列表(`subplans`)、范围表(`finalrtable`)、参数执行类型(`paramExecTypes`)、各种ID计数器(`lastPHId`, `lastRowMarkId`, `lastPlanNodeId`)等。

### 2. 评估并行执行可行性

在 `planner.c:344-380`，判断查询是否可以使用并行执行：

- **必要条件**：游标允许并行、运行在postmaster下、SELECT命令、无修改性CTE、`max_parallel_workers_per_gather > 0`、不在并行worker内部
- **扫描查询树**：调用 `max_parallel_hazard()` 检查查询树中是否存在 `PROPARALLEL_UNSAFE` 的函数
- **调试模式**：如果设置了 `debug_parallel_query`，会强制启用并行模式以测试

### 3. 确定预期扫描元组比例 (`tuple_fraction`)

在 `planner.c:383-408`，根据游标选项决定优化策略：

- **快速启动游标** (`CURSOR_OPT_FAST_PLAN`)：使用 `cursor_tuple_fraction` GUC参数，只优化返回前部分元组的代价
- **默认情况**：假设需要所有元组（`tuple_fraction = 0.0`）

这直接影响优化器是选择全表扫描还是索引扫描。

### 4. 调用核心规划器 — `subquery_planner()`

```c
root = subquery_planner(glob, parse, NULL, false, tuple_fraction);
```

`planner.c:411` — 实际优化工作的入口，递归处理子查询。内部执行：
- **预处理**（`prep/`）：子查询展开、谓词推导、等价类构建
- **核心路径规划**（`path/`）：`query_planner()` → `make_one_rel()` 构建所有关系的最优路径
- **分组/排序/限制规划**（`grouping_planner()`）：处理 GROUP BY、ORDER BY、LIMIT、DISTINCT

### 5. 选择最优路径并生成执行计划

```c
final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);
top_plan = create_plan(root, best_path);
```

在 `planner.c:415-418`：
- 获取最终的关系优化信息
- 根据 `tuple_fraction` 选择代价最低的路径
- 将最优 `Path` 转换为可执行的 `Plan` 树

### 6. 特殊游标处理

```c
if (cursorOptions & CURSOR_OPT_SCROLL)
    top_plan = materialize_finished_plan(top_plan);
```

在 `planner.c:424-428`，对于可滚动游标，如果计划不支持反向扫描，则在顶部添加 `Material` 节点。

### 7. 调试用并行 Gather 节点注入

在 `planner.c:434-475`，如果设置了 `debug_parallel_query` 且顶层计划是并行安全的，强制在顶部添加一个 `Gather` 节点来测试并行模式。

### 8. 参数 finalized — `SS_finalize_plan()`

在 `planner.c:483-494`，如果计划中产生了参数（Params），遍历整个计划树（先子计划后主计划），计算每个节点的 `extParam/allParam` 集合。

### 9. 计划树最终清理 — `set_plan_references()`

```c
top_plan = set_plan_references(root, top_plan);
```

在 `planner.c:502`，对主计划和所有子计划进行最终清理：移除冗余信息、设置节点引用、优化可变长度字段等。

### 10. 构建并返回 `PlannedStmt`

在 `planner.c:514-568`，将所有优化结果组装成 `PlannedStmt` 结构——这是执行器可以直接消费的最终输出。包括：
- 命令类型、计划树、范围表、结果关系
- 子计划、行标记、关系OID
- **JIT决策**（`planner.c:542-564`）：如果计划总代价超过 `jit_above_cost` 阈值，设置 JIT 执行标志

---

## 整体数据流

```
Query (解析器输出)
  → PlannerGlobal (全局状态初始化)
  → subquery_planner() (递归优化，产生 PlannerInfo + 最优路径)
  → create_plan() (路径 → 执行计划树)
  → set_plan_references() (清理/优化计划树)
  → PlannedStmt (执行器输入)
```

## 函数定位

`standard_planner` 本质上是一个**编排器**——它不直接做路径选择或代价估算，而是初始化全局状态、判定并行/游标策略、委托给 `subquery_planner` 完成核心优化，最后将结果组装成执行器可用的格式。
