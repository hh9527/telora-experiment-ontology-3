# A3 对 QueryBuilder 公共草案的能力审查（最终建模前）

审查依据：`ent-1/DOMAIN.md`（物流履约分析题面）与 `query-builder` 公共教程/契约
（`QUERY-BUILDER-TUTORIAL.md`、`PUBLIC-CONTRACT.md`）。本文件只评估 QueryBuilder
草案能否承载题面要求的规范 Plan 与 SQLite 具体化，不评估 ontology eDSL。

## 领域实际需要的算子与 Plan 形状

按 DOMAIN.md 的物理查询事实，合法场景要求 Order grain 的只读规范 Plan 至少携带：

- 基础源 `orders`（带别名），可表达为 `qb.source("orders", alias)`；
- 指标表达式 `COUNT(orders.id)`，可表达为
  `qb.agg_call('Count, 'Some(qb.col_expr(...)), 'False)`；
- 五个安全关系的结构化等值 join 与对应物理 mapping：
  `orders → customers`、`orders → warehouses`、`warehouses → regions`、
  `orders → carriers`、`orders → service_levels`；
- 五个分组表达式：`OrderMonth`（= `substr(orders.created_at, 1, 7)`）、
  `CustomerTier`（= `customers.tier`）、`OriginRegion`（= `regions.name`）、
  `CarrierName`（= `carriers.name`）、`ServiceName`（= `service_levels.name`）；
- 与投影同构的 `GROUP BY` 表达式集合；
- 最终 Plan revision 固定为 `logistics-ontology-v1`；
- 所有动态值进入 `bindings`、SQL 逐字节确定的保证。

## 逐项结论（基于最新草案）

### 已解决项

1. **`Substr` 标量函数已补齐。** 新草案在 `ScalarFunction` 中加入 `'Substr`，
   元数为 2–3 参（`substr(value, start)`、`substr(value, start, length)`），并在
   教程中给出与题面完全一致的示例
   `qb.scalar_call('Substr, [col_expr("o","created_at"), lit('Int(1)), lit('Int(7))])`
   渲染为 `substr(o.created_at, ?, ?)`。`OrderMonth` 的物理事实
   `substr(orders.created_at, 1, 7)` 由此可以合法表达，合法场景按 `OrderMonth`
   分组的阻断项解除。

2. **`Plan.revision` 已改为 `String` 并保证原样保留。** 新草案明确
   `Plan.revision` 是 `String`，由调用方持有（例如 Ent-1 的
   `"logistics-ontology-v1"`），`validate` 与 `transform_sqlite` 都不修改它，也
   不参与验证或具体化，且不得用额外字段或外部备注绕过。题面“最终计划 revision
   固定为 `logistics-ontology-v1`”可以直接由 `Plan.revision` 承载，A3 不再需要
   Int/String 映射或外部备注。

3. **`allow_distinct` 语义已明确。** 新草案说明 `allow_distinct` 只约束
   `AggCall.distinct`，本 package 不提供顶层 `SELECT DISTINCT` 算子，profile 字段
   不再悬空；当前题面不使用 distinct，无阻碍。

4. **导入边界已明确。** 新草案区分 package 内部 `@src` 路径与外部 crate 的
   manifest 依赖名。ent-1 的 `telora-deps.json` 以 `query-builder` 为依赖名，最终
   建模将使用 `import "query-builder/lib.telora" as qb;`，不照抄 `@src`。

### 满足项（草案已覆盖，无需改动）

5. **结构化等值 join 与物理 mapping。** `Join` 使用 `right: Source` + `on:
   Array(EquiCondition)`（`left/right: ColumnRef`），以源表/目标表/等值列对表达
   关系；不是预渲染 predicate String，满足题面“至少表达源表、目标表和等值 join
   两侧的列引用”的要求。五个安全关系均为等值 join，`'Inner` 足够。

6. **聚合与投影形状。** `COUNT(orders.id)` 可由 `'Count` + `'Some(arg)` + `'False`
   表达；`Projection {expr, alias}` 能承载指标/维度别名。

7. **分组形状。** `Plan.group_by: Array(Expr)` 与投影中的分组表达式并列，能够
   表达 `GROUP BY substr(...), customers.tier, regions.name, carriers.name,
   service_levels.name` 所需的全部结构。

8. **确定性具体化与 bindings。** 契约明确“同一个合法 Plan 逐字节产生相同 `sql`
   与相同顺序的 `bindings`，所有运行时值进入 bindings，`LIMIT ?`”，满足题面
   “逐字节相同的 Query，所有动态数据位于 bindings”的要求。

9. **PlanProfile 派生能力。** `standard_profile` + `with_scalar_functions` /
   `with_aggregate_functions` / `with_join_kinds` 足以声明题面接受的 profile（读
   查询、内连接、`Count`、`Substr` 等），符合“领域知识必须声明自己接受的
   PlanProfile”。

10. **失败原子性。** `transform_sqlite` 遇违规以 `fail!` 失败且不发布部分 Query；
    可作为非法场景（`ProductCategory`/`DeliveryException` 分组）在模型层拒绝之后的
    兜底，不产生 SQL 的要求由模型层保证。

11. **标识符与边界。** 单段标识符规则能容纳 `orders`、`customers`、`warehouses`、
    `regions`、`carriers`、`service_levels` 等表名；`Val` 不含 Bytes 不构成阻碍。

### 建议性观察（不阻断当前题面）

12. **`read_profile` 的“读”定义过窄。** 题面要求的“只读”Plan 包含 join、group 与
    聚合（仍不写数据），而 `read_profile` 限定“无 join/group/聚合/distinct”。领域
    将派生自己的 profile，不依赖该命名；仅提示命名与语义容易引起误解。

## 结论

新草案已补齐 `'Substr`、把 `Plan.revision` 改为 `String` 并保证原样保留、明确
`allow_distinct` 与导入边界，A3 审查的阻断项全部解除。算子、Plan 形状、profile、
验证与 SQLite 具体化能够承载题面的合法场景，A3 可以按 DOMAIN.md 进行最终建模。
