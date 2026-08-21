# QueryBuilder 教程

QueryBuilder 是一个领域无关的 Telora package：它定义标准查询计划算子
（`Plan`）、能力声明（`PlanProfile`）、纯验证（`validate`）和确定性的
SQLite 具体化（`transform_sqlite`）。本教程面向不了解本 package 实现的 eDSL
作者，按 DESIGN.md 的五个目标步骤展开。文中所有示例都可以直接粘贴进
`src/bin/` 或 `tests/` 的 Telora 模块运行。

先导入公共 API。本 package 内的模块/测试使用 `@src` 路径：

```telora
import "@src/lib.telora" as qb;
```

外部依赖 crate 必须使用 manifest 固定的依赖名，不能照抄 `@src`（依赖名以
manifest 为准，示例中为 `query-builder`）：

```telora
import "query-builder/lib.telora" as qb;
```

下文假设已经 `import "@src/lib.telora" as qb;`，外部 crate 只需替换 import 行。

## 1. 声明你接受的 PlanProfile

`PlanProfile` 是能力声明，不是全局状态，也绝不改变算子语义。它只决定一个
Plan 是否被接受。最常用的两个现成 profile：

- `qb.standard_profile`：完整 SQLite 能力集（join、filter、group、order、
  limit、distinct、全部标量与聚合函数）。
- `qb.select_profile`：只允许 `SELECT ... FROM ... WHERE`，不允许 join、
  grouping、ordering、limit、distinct 和聚合。

从已有 profile 派生更严格的 profile：

```telora
# 只允许 Inner join，不允许 Left join
let inner_only = qb.with_join_kinds(qb.standard_profile, ['Inner]);

# 只允许 count/sum 两个聚合函数
let few_aggs = qb.with_aggregate_functions(qb.standard_profile, ['Count, 'Sum]);

# 禁止大于比较（示例：你的应用只允许等值过滤）
let no_comparison = qb.with_scalar_functions(qb.standard_profile, [
    'Abs, 'Round, 'Floor, 'Ceil,
    'Lower, 'Upper, 'Length, 'Trim, 'Substr, 'Coalesce,
    'Eq, 'Ne,
    'And, 'Or, 'Not,
]);
```

profile 的每个字段都是显式的：`allow_joins`、`join_kinds`、
`allow_filter`、`allow_group_by`、`allow_order_by`、`allow_limit`、
`allow_distinct`、`scalar_functions`、`aggregate_functions`。其中
`allow_distinct` 只约束 `AggCall.distinct`：为 `'False` 时，任何
`distinct = 'True` 的聚合调用都会被 `validate` 以 profile violation 拒绝；
本 package 不提供顶层 `SELECT DISTINCT` 算子。

## 2. 使用标准算子构造 Plan

用构造器安全地建立 Plan。标识符经过校验，运行时值全部走 `Val`。

```telora
def my_plan: qb.Plan = {
    revision: "my-plan-v7",
    source: qb.source("orders", "o"),
    projections: [
        {expr: qb.col_expr("o", "id"), alias: qb.ident("order_id")},
        {expr: qb.col_expr("o", "total"), alias: qb.ident("total")},
        {expr: qb.agg_call('Sum, 'Some(qb.col_expr("o", "amount")), 'False),
         alias: qb.ident("total_amount")},
    ],
    filter: 'Some(qb.scalar_call('Gt, [qb.col_expr("o", "total"), qb.lit('Int(100))])),
    joins: [
        {kind: 'Inner, right: qb.source("customers", "c"), on: [
            {left: qb.col_ref("o", "customer_id"), right: qb.col_ref("c", "id")},
        ]},
    ],
    group_by: [qb.col_expr("o", "customer_id")],
    ordering: [{expr: qb.col_expr("o", "total"), direction: 'Desc}],
    limit: 'Some(20),
};
```

要点：

- 每个源都必须有别名；列引用一律使用别名，不使用物理表名。
- 表达式（`Expr`）是封闭 enum：列、字面量、标量调用、聚合调用。
- 字面量使用 `qb.lit(qb.Val)`，值是 `'String(s)` / `'Int(i)` / `'Float(f)` /
  `'Bool(b)` 之一，不含 Bytes。
- join 条件只能是结构化等值列对（`EquiCondition`），不接受预渲染 SQL 片段。
- 聚合 `COUNT(*)` 写作 `qb.agg_call('Count, 'None, 'False)`；
  `COUNT(DISTINCT col)` 写作 `qb.agg_call('Count, 'Some(expr), 'True)`。

`Plan.revision` 是 `String`，由调用方持有并原样保留（例如
`"logistics-ontology-v1"`）；它不参与验证或具体化。

`Substr` 支持 SQLite 的两种形式：`substr(value, start)` 与
`substr(value, start, length)`，分别用 2 个和 3 个参数；`validate` 只接受 2
或 3 个参数。例如按月截取日期字符串：

```telora
# substr(orders.created_at, 1, 7) AS order_month
let order_month = qb.scalar_call('Substr, [
    qb.col_expr("o", "created_at"),
    qb.lit('Int(1)),
    qb.lit('Int(7)),
]);
```

渲染为 `substr(o.created_at, ?, ?)`，bindings 为 `['Int(1), 'Int(7)]`。

### 标准算子 vocabulary

| 类别 | 类型 | 成员 |
| --- | --- | --- |
| 数据源 | `Source` | 表名 + 别名（`Ident`） |
| 列引用 | `ColumnRef` | `source.column` |
| 标量调用 | `ScalarFunction` | `'Abs 'Round 'Floor 'Ceil 'Lower 'Upper 'Length 'Trim 'Substr 'Coalesce 'Eq 'Ne 'Lt 'Le 'Gt 'Ge 'And 'Or 'Not` |
| 聚合调用 | `AggFunction` | `'Count 'Sum 'Avg 'Min 'Max` |
| 投影 | `Projection` | `expr AS alias` |
| 过滤 | `Plan.filter` | `Option(Expr)` |
| 等值连接 | `Join` | `'Inner / 'Left` + `Array(EquiCondition)` |
| 分组 | `Plan.group_by` | `Array(Expr)` |
| 排序 | `OrderItem` | `expr ASC/DESC` |
| 限制 | `Plan.limit` | `Option(Int)` |

## 3. 验证 `operators(plan) <= profile`

`validate` 是纯能力，不 `fail!`、不产生部分 Query。要得到人可读的失败原因，
先用 `match` 把 `ValidationIssue` 变成文本（枚举不能直接插值）：

```telora
def classify: Fn(qb.ValidationIssue) -> String = fn(issue) {
    match issue {
        'Profile(message) => `out of profile: \{message}`,
        'Structure(message) => `bad structure: \{message}`,
        'Identifier(message) => `bad identifier: \{message}`,
        'Transform(message) => `transform failed: \{message}`,
    }
};

match qb.validate(my_plan, qb.standard_profile) {
    'Ok(_) => "plan accepted",
    'Err(issue) => classify(issue),
}
```

`ValidationIssue` 的变体区分失败来源：

- `'Profile(String)`：越界算子，profile 不允许该能力；
- `'Structure(String)`：Plan 结构非法（空投影、负数 limit、未知源别名、
  表达式超过深度上限、聚合/标量参数错误、join 别名冲突等）；
- `'Identifier(String)`：标识符不是合法 SQLite 标识符；
- `'Transform(String)`：SQLite 具体化阶段失败（当前实现中，由于验证覆盖
  全部结构约束，此类别通常不会出现，保留用于未来多后端扩展）。

判断一个 Plan 是否完全在 profile 内，使用同一函数并检查 `'Ok`。

## 4. 把 Plan 确定具体化为 SQLite Query

```telora
let query: qb.Query = qb.transform_sqlite(my_plan, qb.standard_profile);
```

`Query` 的形状是：

```telora
type Query = struct {
    sql: String,
    bindings: Array(Val),
};
```

保证：

- 确定性：同一个合法 Plan 逐字节产生相同的 `sql` 和相同顺序的
  `bindings`。
- 参数化：所有运行时值都进入 `bindings`；`sql` 中只出现标识符、关键字、
  运算符和 `?` 占位符。`LIMIT` 也作为绑定值（`LIMIT ?`）。
- 顺序规范：`SELECT`、`FROM`、`JOIN ... ON`、`WHERE`、`GROUP BY`、
  `ORDER BY`、`LIMIT` 按此顺序生成；投影、join、group、order 保持 Plan
  中的声明顺序。
- 占位符与 bindings 一一对应：按 SQL 中出现的顺序收集。

上面 `my_plan` 生成：

```sql
SELECT o.id AS order_id, o.total AS total, sum(o.amount) AS total_amount FROM orders AS o INNER JOIN customers AS c ON o.customer_id = c.id WHERE (o.total > ?) GROUP BY o.customer_id ORDER BY o.total DESC LIMIT ?
```

bindings：`['Int(100), 'Int(20)]`。

## 5. 判断失败来自 profile 越界、Plan 结构还是 Query 转换

先 `validate` 拿到分类错误：

```telora
match qb.validate(plan, profile) {
    'Ok(_) => "in profile",
    'Err('Profile(message)) => `out of profile: \{message}`,
    'Err('Structure(message)) => `bad structure: \{message}`,
    'Err('Identifier(message)) => `bad identifier: \{message}`,
    'Err('Transform(message)) => `transform failed: \{message}`,
}
```

直接调用 `transform_sqlite(plan, profile)` 时，任何违规都会以
`fail!` 产生 Host 诊断，且不发布部分 Query。诊断消息带类别前缀
`profile violation:` / `plan structure violation:` / `invalid identifier:` /
`sqlite transform failure:`，可据此定位。

## 边界与限制

- 标识符必须是 `[A-Za-z_][A-Za-z0-9_]*`（单段，不支持 schema 限定名）。
- 表达式深度上限 `qb.max_expr_depth = 32`；超过的 Plan 在 `validate` 中
  被拒绝。
- `Val` 只支持 String、Int、Float、Bool，不含 Bytes。
- 只有 SQLite 后端；不包含其他 SQL 方言。
- 公共 API 不使用 `Any`、`Dyn` 或 native 声明；所有语义身份都是封闭的
  具名类型。
