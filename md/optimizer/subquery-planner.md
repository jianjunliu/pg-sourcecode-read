# `subquery_planner` 函数主要逻辑分析

`subquery_planner` 位于 `planner.c:602`，是优化器对每个 Query（含子查询）递归处理的核心函数。其工作可分为以下 **8 个阶段**：

---

## 1. 构建 `PlannerInfo`（`planner.c:615-661`）

初始化当前查询层级的规划状态 `root`，包括：
- 绑定 `parse`（Query AST）、`glob`（全局状态）、`parent_root`（父层信息）
- 计算 `query_level`（递归深度）
- 初始化等价类、join domain、append_rel、row identity vars 等空列表
- 处理递归 WITH 查询（`hasRecursion`），分配 `wt_param_id`
- 创建顶层 join domain 节点（等价类构建需要引用它）

---

## 2. 子查询预处理 — 查询重写/拉平（`planner.c:667-711`）

四个关键变换，目标是将子查询尽可能"拉平"到上层，减少嵌套：

| 步骤 | 函数 | 作用 |
|------|------|------|
| CTE 处理 | `SS_process_ctes()` | WITH 列表 → 转为 RTE_SUBQUERY 或 initplan |
| MERGE 转换 | `transform_MERGE_to_join()` | MERGE 命令 → JOIN 形式 |
| 子链接拉平 | `pull_up_sublinks()` | WHERE/ON 中的 ANY/EXISTS SubLink → 半连接/反连接 |
| 函数 RTE 内联 | `preprocess_function_rtes()` | 函数 RTE → 常量简化 + 内联展开为子查询 |
| 子查询拉平 | `pull_up_subqueries()` | 可合并的子查询 → 直接拉入 FROM |
| UNION ALL 展平 | `flatten_simple_union_all()` | 简单 UNION ALL → appendrel 结构 |

---

## 3. 范围表扫描 — 收集特性标记（`planner.c:723-788`）

遍历 `rtable`，标记：
- `hasJoinRTEs`：是否存在 JOIN RTE（决定是否需展开 join alias vars）
- `hasLateralRTEs`：是否存在 LATERAL 引用
- `hasOuterJoins`：是否存在外连接（后续需 `reduce_outer_joins`）
- `hasResultRTEs`：是否存在 RTE_RESULT（后续需清理）
- `qual_security_level`：安全条件最高层级
- 继承关系的 `inh` 标记检查（`has_subclass()`）
- 标记 `leaf_result_relids`

---

## 4. 行标记预处理（`planner.c:794`）

`preprocess_rowmarks(root)` — 处理 FOR UPDATE/SHARE 等锁行标记。必须在子查询拉平之后执行，确保所有基表已就位。

---

## 5. 表达式预处理 — 全面规范化（`planner.c:809-980`）

`preprocess_expression()` 对查询中所有表达式做统一处理，包括：
- **展开 join alias vars**（`flatten_join_alias_vars`）
- **常量折叠**（`eval_const_expressions`）
- **规范化 AND/OR**（`canonicalize_qual`）
- **SubLink → SubPlan 替换**（`SS_replace_correlated_vars`）

依次处理的对象：

| 表达式 | kind |
|--------|------|
| targetList | EXPRKIND_TARGET |
| withCheckOptions | EXPRKIND_QUAL |
| returningList | EXPRKIND_TARGET |
| jointree quals | EXPRKIND_QUAL |
| havingQual | EXPRKIND_QUAL |
| 窗口子句 startOffset/endOffset | EXPRKIND_LIMIT |
| limitOffset/limitCount | EXPRKIND_LIMIT |
| ON CONFLICT 各表达式 | QUAL/TARGET |
| MERGE action targetList/qual | TARGET/QUAL |
| append_rel_list | EXPRKIND_APPINFO |
| RTE 内的 tablesample/functions/values/securityQuals | 各对应 kind |

处理完成后，清除 `joinaliasvars` 列表（不再匹配已处理的表达式树，保留会造成隐患）。

---

## 6. HAVING → WHERE 下推（`planner.c:1013-1041`）

将 HAVING 中不含聚合、不含 volatile 函数、不含子计划的条件推到 WHERE，让过滤尽早发生（减少参与聚合的行数）：

| 条件类型 | 处理方式 |
|----------|----------|
| 含聚合/volatile/子计划 | 保留在 HAVING |
| 有 GROUP BY 无 grouping sets | 移到 WHERE |
| 无 GROUP BY（退化 HAVING） | 复制一份到 WHERE，原样保留在 HAVING |

---

## 7. 外连接简化与结果 RTE 清理（`planner.c:1048-1058`）

- `reduce_outer_joins(root)` — 若 WHERE 条件使外连接的 NULL 侧不可能为空，则将 OUTER JOIN 降级为 INNER JOIN
- `remove_useless_result_rtes(root)` — 从 jointree 中删除无用的 RTE_RESULT 节点，合并单子节点 FromExpr

这两个步骤依赖表达式预处理的结果（规范化后的条件才能正确判断连接简化可能性）。

---

## 8. 核心规划 + 收尾（`planner.c:1063-1087`）

- **`grouping_planner(root, tuple_fraction)`** — 核心规划入口，内部调用 `query_planner()`（路径搜索）并处理 GROUP BY / ORDER BY / LIMIT / DISTINCT / 聚合等上层操作
- **`SS_identify_outer_params(root)`** — 标识当前层级可访问的外层参数 ID
- **`SS_charge_for_initplans(root, final_rel)`** — 将 initPlan 的代价计入最终关系的路径代价
- **`set_cheapest(final_rel)`** — 确定最终关系的最便宜路径

返回 `root`（`PlannerInfo`），其中 `(UPPERREL_FINAL, NULL)` upperrel 上挂载的最优 Path 将由上层 `standard_planner` 通过 `create_plan()` 转为可执行 Plan 树。

---

## 整体流程图

```
subquery_planner()
  ├─ 1. 构建 PlannerInfo (root)
  ├─ 2. 查询重写/拉平: CTE → SubLinks → 函数内联 → 子查询拉平 → UNION ALL
  ├─ 3. rtable 特性扫描
  ├─ 4. 行标记预处理
  ├─ 5. 表达式全面预处理 (preprocess_expression)
  ├─ 6. HAVING → WHERE 下推
  ├─ 7. 外连接简化 + RTE_RESULT 清理
  ├─ 8. grouping_planner() → 路径搜索 + 上层规划
  │     └─ query_planner() → make_one_rel()
  └─ 返回 root (含最优 Path)
```

结合 `md` 目录的调用链路描述，`subquery_planner` 是 `standard_planner` 中**第4阶段**的委托，它完成了从 Query AST 到最优 Path 的全部逻辑变换和路径搜索工作，是优化器最核心的递归处理单元。