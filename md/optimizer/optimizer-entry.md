# PostgreSQL 优化器 (optimizer) 入口函数分析

## 主入口：`planner()`

- **文件**：`src/backend/optimizer/plan/planner.c:273`
- **签名**：
  ```c
  PlannedStmt *planner(Query *parse, const char *query_string,
                       int cursorOptions, ParamListInfo boundParams)
  ```
- **注释标注**：`Query optimizer entry point`

## 调用链路

```
planner()                          ← 优化器公开入口 (planner.c:273)
  ├── 检查是否有 planner_hook      ← 插件钩子，允许外部拦截/修改
  └── standard_planner()           ← 标准规划实现 (planner.c:286)
        ├── 构建 PlannerGlobal     ← 全局规划状态
        ├── 评估并行执行可行性      ← parallel mode 检查
        ├── subquery_planner()     ← 主规划入口 (递归处理子查询, planner.c:411)
        │     ├── 预处理 (prep/)   ← 子查询展开、谓词推导、等价类
        │     ├── query_planner()  ← 核心规划 (path/)
        │     │     ├── make_one_rel()  ← 构建所有关系的最优路径
        │     │     └── add_paths to final_rel
        │     └── grouping_planner()  ← GROUP BY / ORDER BY / LIMIT / DISTINCT
        └── 选择最优 Path → 生成 Plan → 返回 PlannedStmt
```

## `planner()` 工作流程

1. **插件钩子检查**：检查是否有 `planner_hook`，有则交给插件处理（如 pg_hint_plan）
2. **调用 `standard_planner()`** — 真正的优化器实现
3. **`standard_planner()` 内部**：
   - 构建 `PlannerGlobal` 全局规划状态
   - 评估是否可以并行执行（parallel mode 检查）
   - 调用 `subquery_planner()` 进行实际的查询优化（递归处理子查询）
   - 从 `final_rel` 中选出代价最低的 `Path`
   - 将最优 `Path` 转换为执行计划 `Plan`
   - 返回 `PlannedStmt` — 可被 executor 直接执行的计划结构

## 谁调用优化器

`planner()` 由执行器模块 `tcop/postgres.c` 中的 `pg_plan_query()` 调用，是连接解析器输出（`Query` AST）和执行器输入（`PlannedStmt`）的桥梁。

```
解析器输出 → Query (AST) → planner() → PlannedStmt → executor()
```

## optimizer 模块子目录

| 子目录 | 用途 |
|---|---|
| `path/` | 路径生成与代价估算 — `make_one_rel()`、各种路径类型 |
| `plan/` | 计划生成 — `planner.c`（入口）、`createplan.c`、`setrefs.c` 等 |
| `prep/` | 预处理 — 子查询展开、谓词推导、等价类处理 |
| `util/` | 工具函数 — `PlannerInfo` 构建、关系信息等 |
| `geqo/` | 遗传查询优化器 — 用于大连接数的查询 |
