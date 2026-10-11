# PostgreSQL 优化器 (optimizer) 分析总结

> 本文档是对 `md/` 目录下 7 篇优化器分析笔记的综合总结，涵盖从优化器入口到核心数据结构的完整知识体系。
>
> 源文档：`optimizer-entry.md`、`standard-planner.md`、`subquery-planner.md`、`query-planner.md`、`grouping-planner.md`、`reloptinfo.md`、`path.md`

---

## 一、全局视角：优化器调用链

PostgreSQL 优化器的核心调用链贯穿 5 个关键函数，形成"编排 → 递归处理 → 上层规划 → 路径搜索"的分层结构：

```
解析器输出 Query (AST)
      │
      ▼
planner()                        ← 公开入口 (planner.c:273)，检查 planner_hook
      │
      ▼
standard_planner()               ← 编排器 (planner.c:286)，10 个阶段
      │  构建 PlannerGlobal / 并行评估 / tuple_fraction 决策
      ▼
subquery_planner()               ← 递归处理单元 (planner.c:602)，8 个阶段
      │  查询重写拉平 / 表达式预处理 / 外连接简化
      ▼
grouping_planner()               ← 上层操作规划 (planner.c:1272)，10 个阶段
      │  GROUP BY / 窗口 / DISTINCT / ORDER BY / LIMIT（upperrel 链式叠加）
      ▼
query_planner()                  ← 扫描+连接规划 (planmain.c:55)，14 个步骤
      │  等价类构建 / 连接树解构 / 分区展开
      ▼
make_one_rel()                   ← 核心路径搜索 (allpaths.c:174)
      │  基表大小估算 / 扫描路径生成 / 连接顺序搜索
      ▼
最优 Path → create_plan() → set_plan_references() → PlannedStmt
      │
      ▼
执行器 (executor) 消费
```

**职责分层**（关注点分离）：

| 函数 | 角色 | 职责边界 |
|---|---|---|
| `planner` | 门面 | 钩子检查，委托 `standard_planner` |
| `standard_planner` | 编排器 | 全局状态、并行/游标策略、最终组装 `PlannedStmt`；**不做**路径选择 |
| `subquery_planner` | 递归处理器 | 每层 Query 的重写与预处理，递归处理子查询 |
| `grouping_planner` | 上层规划器 | 聚合/窗口/去重/排序/限制的 upperrel 叠加 |
| `query_planner` | 基础规划器 | 只负责 FROM/WHERE（扫描+连接），返回 `RelOptInfo` |
| `make_one_rel` | 路径搜索引擎 | 基表扫描路径 + 连接路径的生成与代价比较 |

---

## 二、各阶段核心逻辑摘要

### 1. `standard_planner` — 10 个阶段

1. **构建 `PlannerGlobal`** — 跨子查询层级共享的全局状态（子计划列表、范围表、ID 计数器等）
2. **并行可行性评估** — `max_parallel_hazard()` 检查 `PROPARALLEL_UNSAFE` 函数
3. **`tuple_fraction` 决策** — 快速启动游标 vs 全量优化，直接影响索引扫描 vs 全表扫描的选择
4. **委托 `subquery_planner()`** — 核心优化
5. **选最优路径生成计划** — `get_cheapest_fractional_path()` + `create_plan()`
6. **可滚动游标处理** — 必要时顶部加 `Material` 节点
7. **调试并行注入** — `debug_parallel_query` 强制加 `Gather`
8. **`SS_finalize_plan()`** — 计算每个节点的 `extParam/allParam` 集合
9. **`set_plan_references()`** — 计划树最终清理
10. **组装 `PlannedStmt`** — 含 JIT 决策（超过 `jit_above_cost` 则启用）

### 2. `subquery_planner` — 8 个阶段

核心主题是**查询重写与规范化**，将 Query AST 变换为易于优化的形式：

- **拉平（flatten）**：CTE 处理、SubLink → 半/反连接、函数 RTE 内联、子查询拉平、UNION ALL 展平 —— 目标是尽量减少嵌套
- **收集特性标记**：扫描 rtable 标记 `hasJoinRTEs`/`hasLateralRTEs`/`hasOuterJoins` 等，驱动后续步骤
- **表达式全面预处理**：`preprocess_expression()` 统一做常量折叠、join alias 展开、qual 规范化、SubLink → SubPlan 替换
- **谓词下推与简化**：HAVING → WHERE 下推；`reduce_outer_joins()` 将 OUTER JOIN 降级 INNER JOIN；清理无用 RTE_RESULT
- **收尾**：调用 `grouping_planner()`，然后 `set_cheapest(final_rel)` 确定最优路径

### 3. `grouping_planner` — upperrel 链式构建

在 `query_planner` 返回的扫描/连接路径之上，按固定顺序逐层叠加上层操作，每层创建一个新的 upperrel：

```
scan/join → GROUP_AGG → WINDOW → DISTINCT → ORDERED → FINAL
```

| 阶段 | 函数 | upperrel |
|---|---|---|
| 分组/聚合 | `create_grouping_paths()` | `UPPERREL_GROUP_AGG`（HashAgg / Sort+GroupAgg / MixedAgg） |
| 窗口函数 | `create_window_paths()` | `UPPERREL_WINDOW` |
| 去重 | `create_distinct_paths()` | `UPPERREL_DISTINCT`（Unique / Hash） |
| 排序 | `create_ordered_paths()` | `UPPERREL_ORDERED`（利用已有 pathkey 可免 Sort） |
| 最终加工 | 遍历 pathlist 叠加节点 | `UPPERREL_FINAL`（LockRows / Limit / ModifyTable） |

关键设计：
- **PathTarget 逆推**：从 `final_target` 逆推 `sort_input_target` → `grouping_target` → `scanjoin_target`，每层只传递必需列
- **SRF 延迟求值**：集合返回函数尽量推迟到排序/去重之后（`split_pathtarget_at_srfs()`）
- **LIMIT 语义**：存在分组/聚合/窗口/DISTINCT/SRF 时 `limit_tuples = -1`，LIMIT 不能直接下推
- **可扩展性**：FDW 的 `GetForeignUpperPaths` 和 `create_upper_paths_hook` 允许外部注入路径

### 4. `query_planner` — 等价类驱动的 14 步管线

只处理 FROM/WHERE，核心是**等价类三阶段管线**与**延迟决策**：

1. 初始化连接规划数据结构
2. 快速路径：单 RTE_RESULT 平凡查询直接返回 `GroupResultPath`
3. `add_base_rels_to_query()` — 为基表创建 `RelOptInfo`
4. 构建基表目标列表 / 占位符 / LATERAL 引用
5. **`deconstruct_jointree()`** — 最关键步骤：分发 WHERE 条件、构建 `SpecialJoinInfo`、**创建等价类**
6. `reconsider_outer_join_clauses()` — 重新审视被延迟的外连接条件
7. `generate_base_implied_equalities()` — **应用等价类**生成基表隐含条件（如 `{a.x, b.y, 5}` → `a.x=5`），让过滤尽早发生
8. `qp_callback()` — 等价类合并完成后计算规范化路径键（query/group/window/distinct pathkeys）
9. 占位符层级修正 → 移除无用连接 → 降级唯一半连接（**顺序敏感**：占位符修正必须先于连接移除）
10. 占位符分发 + `create_lateral_join_info()`
11. 外键匹配 + OR 条件提取
12. 继承/分区展开（**延迟到最后**，因需要完整限制条件做分区裁剪）+ 行标识变量分发
13. **`make_one_rel()`** — 核心路径搜索：基表大小估算 → 扫描路径生成 → 连接路径搜索
14. 验证并返回 `final_rel`

---

## 三、核心数据结构

### 1. `RelOptInfo` — "待优化的关系"（pathnodes.h:850）

代表基表、连接结果或上层关系，是路径的容器：

- **6 种类型**（`RelOptKind`）：`BASEREL`（基表）、`JOINREL`（连接）、`OTHER_MEMBER_REL`（分区子表）、`OTHER_JOINREL`（partitionwise join）、`UPPER_REL`（上层关系）、`OTHER_UPPER_REL`
- **关键字段组**：标识（`relids` 是唯一标识）、基数估计（`rows/pages/tuples`）、路径（`pathlist` + `cheapest_*` 缓存）、输出列（`reltarget`）、约束条件（`baserestrictinfo`/`joininfo`）、分区、FDW
- **生命周期**：
  - 创建：基表 `build_simple_rel()` / 连接 `build_join_rel()`（经 `find_join_rel()` 去重）/ 上层 `fetch_upper_rel()`
  - 填充：`make_one_rel()` 计算大小、生成路径、`set_cheapest()` 选最优
  - 消费：`create_plan()` 从 `cheapest_total_path` 递归生成 Plan 树

### 2. `Path` — "候选执行策略"（pathnodes.h:1593）

- **基类核心字段**：`pathtype`（对应 Plan 节点类型）、`parent`（回指 RelOptInfo）、`param_info`（参数化）、`startup_cost`/`total_cost`（双维度代价）、`rows`、`pathkeys`（输出有序性）、并行三字段
- **子类家族**：
  - 扫描类：IndexPath、BitmapHeapPath（+And/Or）、TidPath、SubqueryScanPath、ForeignPath、CustomPath
  - 连接类：NestPath、MergePath、HashPath（共享 JoinPath 基类，含 `jointype`/`inner_unique`/内外路径）
  - 并行类：GatherPath、GatherMergePath
  - 排序/投影：SortPath、IncrementalSortPath、ProjectionPath、ProjectSetPath
  - 分组/聚合：GroupPath、AggPath、GroupingSetsPath、WindowAggPath、MinMaxAggPath
  - 特殊：UniquePath、MaterialPath、MemoizePath、SetOpPath、RecursiveUnionPath、ModifyTablePath、LimitPath、LockRowsPath
- **配套结构**：
  - `PathKey` — 规范化排序维度，比较简化为指针比较（`compare_pathkeys` 返回 EQUAL/BETTER1/BETTER2/DIFFERENT）
  - `ParamPathInfo` — 参数化路径信息，典型场景是嵌套循环内侧索引扫描；只能用作 NestLoop 内侧
  - `PathTarget` — 输出列表达式 + 代价/宽度；上层关系总有自定义 PathTarget
- **`add_path()` 支配淘汰**（pathnode.c:422）— 路径剪枝核心算法：
  - 模糊代价比较（1% 容差 `STD_FUZZ_FACTOR`）
  - 支配需**全维度不劣**：代价、pathkeys（超集）、参数化（子集）、行数、并行安全
  - `pathlist` 按 `total_cost` 排序支持提前终止；`IndexPath` 删除时不 `pfree`（可能被 BitmapHeapPath 引用）

### 3. 结构关系图

```
PlannerInfo (root)
├── simple_rel_array[]  ──→ RelOptInfo (基表)
├── join_rel_list/hash  ──→ RelOptInfo (连接)
├── upper_rels[kind]    ──→ RelOptInfo (上层)
├── eq_classes          ──→ EquivalenceClass (条件传播)
└── join_info_list      ──→ SpecialJoinInfo (外连接语义)

RelOptInfo
├── reltarget   ──→ PathTarget (输出列)
├── pathlist    ──→ [Path, ...] (候选策略，add_path 剪枝)
├── cheapest_*  ──→ 最优 Path (set_cheapest 选出)
└── Path.pathtype ──→ create_plan() ──→ Plan 树 ──→ PlannedStmt
```

---

## 四、贯穿全局的设计原则

1. **Path/Plan 两阶段分离**：优化器先在 Path 层面做多维度（代价、有序性、参数化、并行）候选比较，最后才一次性转换为 Plan 树。Path 是轻量抽象，Plan 是完整执行描述。

2. **等价类驱动的条件传播**：`deconstruct_jointree`（构建）→ `reconsider_outer_join_clauses`（扩展）→ `generate_base_implied_equalities`（应用）三阶段管线，是谓词推导和过滤下推的核心机制。

3. **尽早过滤**：HAVING → WHERE 下推、等价类生成基表隐含条件、外连接降级为内连接、移除无用连接——所有手段都为了让过滤发生在数据量最大的最早阶段。

4. **upperrel 链式叠加**：上层操作（分组→窗口→去重→排序→最终）每层独立创建 upperrel 并保留多条候选路径，层与层之间通过 PathTarget 逆推最小化列传递。

5. **延迟决策**：分区展开延迟到限制条件完整之后（支持分区裁剪）；占位符修正先于连接移除；SRF 延迟到排序后求值。

6. **双维度代价模型**：`startup_cost` + `total_cost` 配合 `tuple_fraction`，使同一查询在"快速取前几行"（游标/LIMIT）和"取全部行"两种场景下选出不同最优路径。

7. **支配淘汰剪枝**：`add_path()` 用多维支配关系淘汰劣路径，控制搜索空间；配合模糊比较（1% 容差）避免保留几乎等价的路径。

8. **处处可扩展**：`planner_hook`（pg_hint_plan 等）、`create_upper_paths_hook`、FDW 的 `fdwroutine`、CustomPath——优化器各层都为插件预留了注入点。

---

## 五、文档索引

| 文档 | 主题 | 对应源码 |
|---|---|---|
| [optimizer-entry.md](optimizer-entry.md) | 优化器入口与调用链总览 | `optimizer/plan/planner.c:273` |
| [standard-planner.md](standard-planner.md) | 编排器 10 阶段 | `planner.c:286` |
| [subquery-planner.md](subquery-planner.md) | 查询重写与预处理 8 阶段 | `planner.c:602` |
| [grouping-planner.md](grouping-planner.md) | 上层操作规划 10 阶段 | `planner.c:1272` |
| [query-planner.md](query-planner.md) | 扫描/连接规划 14 步 | `optimizer/plan/planmain.c:55` |
| [reloptinfo.md](reloptinfo.md) | 核心结构 RelOptInfo | `include/nodes/pathnodes.h:850` |
| [path.md](path.md) | 核心结构 Path 与 add_path 剪枝 | `pathnodes.h:1593`、`util/pathnode.c` |

> 尚未覆盖的相关主题（可作为后续笔记方向）：`make_one_rel()`/`allpaths.c` 路径生成细节、连接顺序搜索（`standard_join_search`/动态规划/GEQO）、代价估算（`costsize.c`）、选择率估算（`clausesel.c`/`selfuncs.c`）、`createplan.c` Path→Plan 转换、`setrefs.c` 最终清理。
