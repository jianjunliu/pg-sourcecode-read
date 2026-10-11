# `query_planner` 函数主要逻辑分析

`query_planner` 位于 `planmain.c:55`，负责生成基本查询（可能涉及连接，但不涉及聚合/分组/排序等上层特性）的最优路径。它不直接选择最终路径，而是返回 `RelOptInfo`，由 `grouping_planner` 在上层操作中挑选。

函数签名：
```c
RelOptInfo *query_planner(PlannerInfo *root,
                          query_pathkeys_callback qp_callback, void *qp_extra)
```

---

## 1. 初始化规划数据结构（`planmain.c:68-81`）

将 `root` 中与连接规划相关的列表清空为 NIL/NULL：
- `join_rel_list`、`join_rel_hash`、`join_rel_level` — 连接关系缓存
- `canon_pathkeys` — 规范化路径键
- `left/right/full_join_clauses` — 各类外连接条件
- `join_info_list` — SpecialJoinInfo 列表
- `placeholder_list/array` — 占位符信息
- `fkey_list` — 外键列表
- `initial_rels` — 初始基表关系列表

> 注意：`append_rel_list` 不在此处重置，因为它由 `subquery_planner` 预先设置。

---

## 2. 快速路径 — 单 RTE_RESULT 的平凡查询（`planmain.c:94-157`）

当 `jointree` 只有一个 `RTE_RESULT`（如 `SELECT expression`、`INSERT ... VALUES()`），直接跳过所有后续步骤：

- `build_simple_rel(root, varno, NULL)` — 构造唯一的 `RelOptInfo`
- 检查并行安全性（仅在 `debug_parallel_query` 启用时有意义）
- `create_group_result_path()` — 创建一个 `GroupResultPath` 作为唯一路径，将 quals 直接嵌入
- `set_cheapest()` — 选择最便宜路径（此处只有一条路径）
- 标记 `ec_merging_done = true` — 假装等价类合并已完成
- 调用 `qp_callback` — 处理可能存在的 ORDER BY（如 `SELECT 2+2 ORDER BY 1`）
- **直接返回 `final_rel`**

---

## 3. 构建基表 RelOptInfo（`planmain.c:168`）

```c
add_base_rels_to_query(root, (Node *) parse->jointree);
```

遍历 jointree 中的每个 `RangeTblRef`，为每个基表创建 `RelOptInfo` 并注册到 `simple_rel_array`。此函数搜索的是 jointree（而非 rtable），因为 rtable 中可能包含不活跃的 RTE（如未被引用的视图）。

定义：`initsplan.c:156`

---

## 4. 构建基表目标列表与占位符（`planmain.c:180-184`）

| 函数 | 定义位置 | 作用 |
|------|----------|------|
| `build_base_rel_tlists()` | `initsplan.c:233` | 遍历 `processed_tlist`，将引用的 Var 添加到对应基表的 `reltarget`，使每个基表知道需要输出哪些列 |
| `find_placeholders_in_jointree()` | `placeholder.c:178` | 扫描 jointree，为每个 `PlaceHolderVar` 创建 `PlaceHolderInfo`，记录其计算所需的最低 join 层级 |
| `find_lateral_references()` | `initsplan.c:357` | 找出 LATERAL 子查询对外层关系的引用，将它们记录到基表的 `lateral_relids` 中 |

---

## 5. 解构连接树 — 核心步骤（`planmain.c:186`）

```c
joinlist = deconstruct_jointree(root);
```

定义：`initsplan.c:729`

这是最关键的预处理步骤，它遍历 jointree 并完成：
- **分发 WHERE 条件**：将 quals 分配到最合适的层级（基表限制条件、连接条件、外连接延迟条件）
- **构建 SpecialJoinInfo**：记录每个外连接的语义（左右侧、连接类型、延迟条件等）
- **构建等价类（EquivalenceClass）**：从等值连接条件（如 `a.x = b.y`）创建等价类，合并已存在的等价类
- **构建连接条件列表**：`left_join_clauses`、`right_join_clauses`、`full_join_clauses`
- **生成目标 joinlist**：返回扁平化的连接列表，供 `make_one_rel()` 使用

---

## 6. 外连接条件重新考虑（`planmain.c:193`）

```c
reconsider_outer_join_clauses(root);
```

定义：`equivclass.c:1975`

在 `deconstruct_jointree` 阶段，外连接的某些条件被延迟处理（因为外连接可能改变条件的语义）。等价类构建完成后，重新审视这些延迟条件：
- 如果延迟条件可以安全地合并到已有等价类中，则合并
- 可能导致等价类进一步扩展或合并

---

## 7. 生成基表隐含等值条件（`planmain.c:200`）

```c
generate_base_implied_equalities(root);
```

定义：`equivclass.c:1030`

利用已构建的等价类，为每个基表生成隐含的限制条件。例如，等价类 `{a.x, b.y, c.z}` 中如果存在常量成员 `5`，则为每个包含该等价类成员的基表生成 `col = 5` 条件。这使过滤尽早发生在基表扫描阶段。

---

## 8. 计算查询路径键（`planmain.c:207`）

```c
(*qp_callback)(root, qp_extra);
```

调用回调函数（实际是 `standard_qp_callback`），在等价类合并完成后计算：
- `query_pathkeys` — ORDER BY 的规范化路径键
- `group_pathkeys` — GROUP BY 的路径键
- `window_pathkeys` — 窗口函数的排序路径键
- `distinct_pathkeys` — DISTINCT 的路径键

路径键必须等价类合并完成后才能构建为规范化形式。

---

## 9. 占位符层级修正与连接移除（`planmain.c:216-229`）

| 函数 | 定义位置 | 作用 |
|------|----------|------|
| `fix_placeholder_input_needed_levels()` | `placeholder.c:300` | 标记每个占位符在哪个 join 层级必须被计算，确保上层关系知道需要输出占位符依赖的 Var |
| `remove_useless_joins()` | `analyzejoins.c:64` | 移除无用的外连接 — 如果外连接不改变结果（例如左连接的内侧列不被引用且无限制条件），则降级或删除 |
| `reduce_unique_semijoins()` | `analyzejoins.c:624` | 将内侧唯一的半连接降级为普通内连接 — 因为内侧唯一时 SEMI JOIN 和 INNER JOIN 结果相同 |

**顺序依赖**：占位符修正必须先于连接移除，因为占位符可能使某些被认为无用的列变为必需，从而阻止连接移除。

---

## 10. 占位符分发与 LATERAL 信息构建（`planmain.c:236-242`）

| 函数 | 定义位置 | 作用 |
|------|----------|------|
| `add_placeholders_to_base_rels()` | `placeholder.c:329` | 将占位符添加到其可计算的最低基表的 `reltarget`，确保占位符在合适的位置被求值 |
| `create_lateral_join_info()` | `initsplan.c:500` | 利用已收集的 `lateral_relids` 和占位符依赖，构建完整的 LATERAL join 信息，标记哪些关系之间有 LATERAL 依赖，影响连接顺序选择 |

---

## 11. 外键匹配与 OR 条件提取（`planmain.c:250-256`）

| 函数 | 定义位置 | 作用 |
|------|----------|------|
| `match_foreign_keys_to_quals()` | `initsplan.c:2929` | 将声明的外键约束匹配到等价类和连接条件，生成额外的等值连接信息（有助于优化器判断连接唯一性等） |
| `extract_restriction_or_clauses()` | `orclauses.c:76` | 从连接 OR 条件中提取单关系限制 OR 条件。例如 `WHERE (a.x=1 OR a.x=2) AND b.y=a.x` 可从后者提取 `b.y=1 OR b.y=2` 作为 b 的限制条件 |

---

## 12. 继承/分区展开与行标识变量分发（`planmain.c:266-273`）

| 函数 | 定义位置 | 作用 |
|------|----------|------|
| `add_other_rels_to_query()` | `initsplan.c:194` | 为继承表/分区表的子表创建"other rel" `RelOptInfo`，展开 appendrel。延迟到此处执行是因为需要先收集完整的限制条件用于分区裁剪 |
| `distribute_row_identity_vars()` | `appendinfo.c:966` | 将 UPDATE/DELETE/MERGE 的行标识变量（如 `ctid`）分发到目标关系，确保执行器可以定位要修改的行 |

---

## 13. 核心路径搜索 — `make_one_rel()`（`planmain.c:278`）

```c
final_rel = make_one_rel(root, joinlist);
```

定义：`allpaths.c:174`

这是路径搜索的核心入口，内部执行：
- `set_base_rel_consider_startup()` — 标记哪些基表需要考虑快速启动路径（SEMI/ANTI JOIN 的内侧）
- `set_base_rel_sizes()` — 计算每个基表的行数/宽度估算，设置 `consider_parallel` 标记
- 计算 `total_table_pages` — 所有基表的总页数，用于代价估算
- `set_base_rel_pathlists()` — 为每个基表生成所有可用访问路径（顺序扫描、索引扫描、位图扫描、TID扫描等）
- `make_rel_from_joinlist()` — 从 joinlist 构建完整的连接路径树，搜索最优连接顺序和连接方法

---

## 14. 验证与返回（`planmain.c:281-285`）

```c
if (!final_rel || !final_rel->cheapest_total_path ||
    final_rel->cheapest_total_path->param_info != NULL)
    elog(ERROR, "failed to construct the join relation");
return final_rel;
```

确保至少有一条可用的非参数化最便宜路径，返回包含所有扫描/连接路径的 `final_rel`。

---

## 整体流程图

```
query_planner()
  ├─ 1. 初始化规划数据结构 (清空连接相关列表)
  ├─ 2. 快速路径: RTE_RESULT 平凡查询 → 直接返回 GroupResultPath
  ├─ 3. add_base_rels_to_query()  → 为基表创建 RelOptInfo
  ├─ 4. build_base_rel_tlists()   → 构建基表目标列表
  │   find_placeholders_in_jointree() → 发现占位符
  │   find_lateral_references()   → 收集 LATERAL 引用
  ├─ 5. deconstruct_jointree()    → 解构连接树 (核心: 分发条件/构建等价类/SpecialJoinInfo)
  ├─ 6. reconsider_outer_join_clauses() → 重新考虑延迟的外连接条件
  ├─ 7. generate_base_implied_equalities() → 从等价类生成隐含限制条件
  ├─ 8. qp_callback()            → 计算规范化路径键 (ORDER BY/GROUP BY/DISTINCT)
  ├─ 9. fix_placeholder_input_needed_levels() → 修正占位符层级
  │   remove_useless_joins()      → 移除无用外连接
  │   reduce_unique_semijoins()   → 降级唯一内侧半连接
  ├─ 10. add_placeholders_to_base_rels() → 分发占位符到基表
  │   create_lateral_join_info()  → 构建 LATERAL 连接信息
  ├─ 11. match_foreign_keys_to_quals() → 外键匹配
  │   extract_restriction_or_clauses() → 提取单关系 OR 条件
  ├─ 12. add_other_rels_to_query() → 展开继承/分区子表
  │   distribute_row_identity_vars() → 分发行标识变量
  ├─ 13. make_one_rel()           → 核心路径搜索 (扫描路径 + 连接路径)
  │     ├─ set_base_rel_sizes()   → 行数估算
  │     ├─ set_base_rel_pathlists() → 基表扫描路径生成
  │     └─ make_rel_from_joinlist() → 连接路径搜索
  └─ 14. 验证 + 返回 final_rel
```

## 与上下游的关系

- **上游**：`grouping_planner`（`planner.c:1494`）调用 `query_planner`，传入 `standard_qp_callback` 和 `qp_extra`，获取扫描/连接阶段的最优路径集合
- **下游**：`query_planner` 返回的 `final_rel` 上的路径只是"扫描+连接"部分，`grouping_planner` 在此基础上叠加聚合、窗口、排序、限制等上层操作路径，最终写入 `(UPPERREL_FINAL, NULL)` upperrel

## 关键设计原则

1. **等价类驱动**：`deconstruct_jointree` → `reconsider_outer_join_clauses` → `generate_base_implied_equalities` 形成等价类构建→扩展→应用的三阶段管线，是条件传播的核心机制
2. **延迟决策**：继承展开（`add_other_rels_to_query`）延迟到最后，因为需要完整的限制条件来做分区裁剪；占位符层级修正必须先于连接移除，因为前者可能使后者的判断失效
3. **关注点分离**：`query_planner` 只负责"FROM/WHERE"部分（扫描+连接），聚合/排序等上层逻辑完全由 `grouping_planner` 处理