# RelOptInfo 数据结构分析

`RelOptInfo` 是 PostgreSQL 查询优化器的**核心数据结构**，代表一个"待优化的关系"——可以是一张基表、一个连接结果、一个分组结果等。定义在 `src/include/nodes/pathnodes.h:850-1037`。

## 一、关系类型（RelOptKind）

```c
typedef enum RelOptKind
{
    RELOPT_BASEREL,           /* 基表（单个 RTE：表/子查询/函数） */
    RELOPT_JOINREL,           /* 两个或多个基表连接后的结果 */
    RELOPT_OTHER_MEMBER_REL,  /* 继承/分区子表 */
    RELOPT_OTHER_JOINREL,     /* 分区子表间的连接（partitionwise join） */
    RELOPT_UPPER_REL,         /* 上层关系（GROUP BY/DISTINCT/ORDER BY 等） */
    RELOPT_OTHER_UPPER_REL    /* 其他上层关系 */
} RelOptKind;
```

辅助宏：
- `IS_SIMPLE_REL(rel)` — 基表或其他成员关系
- `IS_JOIN_REL(rel)` — 连接关系
- `IS_UPPER_REL(rel)` — 上层关系
- `IS_OTHER_REL(rel)` — 任何"其他"类型

## 二、结构体字段分组

| 分组 | 关键字段 | 作用 |
|---|---|---|
| **标识** | `reloptkind`, `relids` | 关系类型 + 涉及的 RT 索引集合（基表=单元素，连接=多元素） |
| **基数估计** | `rows`, `pages`, `tuples`, `allvisfrac` | 输出行数估计、磁盘页数、总元组数、全可见页比例 |
| **路径（核心）** | `pathlist`, `partial_pathlist`, `cheapest_startup_path`, `cheapest_total_path`, `cheapest_parameterized_paths` | 候选执行路径列表 + 最优路径缓存 |
| **输出列** | `reltarget` (PathTarget\*) | 该关系需要输出的表达式列表及代价/宽度 |
| **LATERAL** | `lateral_relids`, `lateral_vars`, `lateral_referencers` | LATERAL 引用导致的参数化信息 |
| **基表专属** | `relid`, `rtekind`, `min_attr`, `max_attr`, `attr_needed`, `attr_widths`, `indexlist`, `subroot` | 仅基表有值，连接 rel 为空 |
| **约束/连接条件** | `baserestrictinfo`, `baserestrictcost`, `joininfo`, `has_eclass_joins` | WHERE 过滤条件 + 涉及该表的连接条件 |
| **分区** | `part_scheme`, `nparts`, `part_rels`, `live_parts`, `boundinfo` | 分区剪枝和 partitionwise join 所需 |
| **外部表** | `serverid`, `fdwroutine`, `fdw_private` | FDW 相关 |

## 三、生命周期

### 1. 创建

由不同函数根据关系类型创建：

- **基表**：`build_simple_rel()` — `query_planner()` 遍历 jointree，为每个 RTE 分配 RelOptInfo，调用 `get_relation_info()` 填充 pages/tuples/indexlist，存入 `root->simple_rel_array[relid]`
- **连接**：`build_join_rel()` — `standard_join_search()` 逐层合并时调用，先查 `find_join_rel()` 避免重复，计算 `lateral_relids`、构建 targetlist、收集 restrict/join 子句，存入 `join_rel_list` / `join_rel_hash`
- **上层**：`fetch_upper_rel()` — `grouping_planner()` 为 GROUP BY/ORDER BY/LIMIT 等调用

### 2. 填充（`make_one_rel`）

```
make_one_rel()
├── set_base_rel_sizes()     → 计算每个基表的 rows（应用选择率估计）
├── set_base_rel_pathlists() → 生成 SeqScan/IndexScan/BitmapScan 等 Path
├── join_search              → 逐层生成连接路径 (NestLoop/MergeJoin/HashJoin)
│   └── add_paths_to_joinrel()
└── set_cheapest()           → 选出最优路径存入 cheapest_* 字段
```

`set_cheapest()` 的关键选择：
- `cheapest_startup_path` — 启动代价最低的**非参数化**路径
- `cheapest_total_path` — 总代价最低的**非参数化**路径（若无则选最小参数化）
- `cheapest_parameterized_paths` — 每种参数化下的最优路径

### 3. 消费（`create_plan`）

```
grouping_planner()
├── 接收 final_rel（最顶层连接 RelOptInfo）
├── 为 GROUP BY/DISTINCT/ORDER BY 创建上层 RelOptInfo 并附加 Path
└── final_rel->cheapest_total_path 作为最终计划

create_plan()
└── create_plan_recurse() 从最优 Path 开始
    ├── create_scan_plan()  → SeqScan/IndexScan/etc.
    └── create_join_plan()  → NestLoop/MergeJoin/HashJoin
```

## 四、与其他结构的关系

```
PlannerInfo (root)
├── simple_rel_array[relid] ───→ RelOptInfo (基表)
├── join_rel_list / join_rel_hash ──→ RelOptInfo (连接)
├── upper_rels[kind] ───→ RelOptInfo (上层)
├── eq_classes ───→ EquivalenceClass (推导连接条件)
└── join_info_list ───→ SpecialJoinInfo (外连接语义)

RelOptInfo
├── reltarget ───→ PathTarget (输出列)
├── pathlist ───→ [Path, Path, ...] (候选执行策略)
│   ├── Path.parent ───→ 回指 RelOptInfo
│   ├── Path.param_info ───→ ParamPathInfo (参数化)
│   └── Path.pathtype ───→ 对应 Plan 的 NodeTag
├── cheapest_*_path ───→ 最优 Path 缓存
├── baserestrictinfo ───→ [RestrictInfo, ...] (WHERE 子句)
├── joininfo ───→ [RestrictInfo, ...] (连接条件)
└── indexlist ───→ [IndexOptInfo, ...] (可用索引)
```

## 五、关键设计要点

1. **`relids` 是唯一标识**：基表的 `relids` 是单元素 Bitmapset，连接 rel 的 `relids` 是多个 RT 索引的并集。优化器通过 `relids` 查找/去重连接关系。

2. **Path 是抽象执行策略**：一个 RelOptInfo 的 `pathlist` 保存所有候选方案（SeqScan、各 IndexScan、BitmapScan 等），`add_path()` 自动淘汰被支配的路径。

3. **`cheapest_*` 字段是优化器选择结果**：`set_cheapest()` 从 `pathlist` 中选出启动代价最低和总代价最低的路径，后续 `create_plan()` 基于这些最优路径生成最终的 Plan 树。
