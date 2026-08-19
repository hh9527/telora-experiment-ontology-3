# QueryBuilder 教程

本教程面向不了解 QueryBuilder 实现的 eDSL 作者（A2/A3）。它说明如何：

1. 声明其接受的 `PlanProfile`；
2. 使用标准算子构造 `Plan`；
3. 验证 `operators(plan) <= profile`；
4. 把 `Plan` 确定具体化为 SQLite `Query`；
5. 判断失败来自 profile 越界、Plan 结构还是 Query 转换。

运行示例与验证：

```text
./bin/telora run main -C query-builder
./bin/telora run verify -C query-builder
./bin/telora run invalid -C query-builder --best-effort
./bin/telora check @test/query-builder.telora -C query-builder
```

在你的 crate 中使用：

```telora
import "query-builder/query-builder.telora" { ... };
```

（该逻辑模块 ID 由依赖 crate 的 `telora-deps.json` 固定的 package 名
`query-builder` 与 package 内模块路径 `query-builder.telora` 组成，对应
`<query-builder>/src/query-builder.telora`。）

## 1. 声明 PlanProfile

`PlanProfile` 是标准算子能力的显式声明：它列出允许的 join kind、scalar
function、aggregate function，以及是否允许 filter、grouping、ordering 与 limit。
profile 是能力声明，不是隐式全局状态，也不改变算子语义——同一个 Plan 在
permissive 与 restrictive profile 下语义完全相同，只是后者会被判定为越界。

```telora
let my_profile: PlanProfile = {
    allow_joins: ['Inner, 'Left],
    allow_scalar: ['Lower, 'Upper, 'Length, 'Abs, 'Round, 'Coalesce, 'Substr],
    allow_aggregate: ['Count, 'CountDistinct, 'Sum, 'Avg, 'Min, 'Max],
    allow_filter: 'True,
    allow_grouping: 'True,
    allow_ordering: 'True,
    allow_limit: 'True,
};
```

空数组表示完全禁止该类算子：`allow_joins: []` 表示不接受任何 join。

## 2. 构造 Plan

`Plan` 完整保留 revision、基础数据源、有序投影、filter、按顺序选择的 join
及其结构化等值条件、grouping、ordering 与 limit。`revision` 是稳定、可读的
String 标签（例如 `"logistics-ontology-v1"`），由 Plan 自身逐字节保留；
它不参与 profile/validation 判定，也不会出现在生成的 SQL/Query 中。所有动态
数据都必须作为有类型绑定值 `Val`（`'String`/`'Int`/`'Float`/`'Bool`）放进
`'Bound(...)` 表达式；SQL 标识符（表名、列名、alias）是普通 String，与用户值在
类型和 lowering 路径上完全分开。

```telora
let substr_month: Expr = 'ScalarCall({
    kind: 'Substr,
    args: [
        'Column({alias: "o", column: "created_at"}),
        'Bound('Int(1)),
        'Bound('Int(7)),
    ],
});

let plan: Plan = {
    revision: "logistics-ontology-v1",
    source: {table: "orders", alias: "o"},
    projection: [
        {expr: 'Column({alias: "o", column: "region"}), alias: "region"},
        {expr: substr_month, alias: "month"},
        {
            expr: 'ScalarCall({
                kind: 'Round,
                args: [
                    'Aggregate({kind: 'Sum, arg: 'Column({alias: "o", column: "amount"})}),
                    'Bound('Int(2)),
                ],
            }),
            alias: "total",
        },
    ],
    filter: [
        'Compare({
            op: 'Gt,
            left: 'Column({alias: "o", column: "amount"}),
            right: 'Bound('Int(500)),
        }),
    ],
    joins: [
        {
            kind: 'Inner,
            source: {table: "customers", alias: "c"},
            left: {alias: "o", column: "customer_id"},
            right: {alias: "c", column: "id"},
        },
    ],
    grouping: [substr_month],
    ordering: [
        {
            expr: 'ScalarCall({
                kind: 'Round,
                args: [
                    'Aggregate({kind: 'Sum, arg: 'Column({alias: "o", column: "amount"})}),
                    'Bound('Int(2)),
                ],
            }),
            direction: 'Desc,
        },
    ],
    limit: 'Some(10),
};
```

该 Plan 生成的 SQLite SQL 为：

```sql
SELECT "o"."region" AS "region", substr("o"."created_at", ?, ?) AS "month", round(SUM("o"."amount"), ?) AS "total"
FROM "orders" AS "o"
JOIN "customers" AS "c" ON "o"."customer_id" = "c"."id"
WHERE ("o"."amount" > ?)
GROUP BY substr("o"."created_at", ?, ?)
ORDER BY round(SUM("o"."amount"), ?) DESC
LIMIT ?
```

bindings 为 `['Int(1), 'Int(7), 'Int(2), 'Int(500), 'Int(1), 'Int(7), 'Int(2), 'Int(10)]`
（顺序与占位符一一对应：投影内 substr 的起点/长度、投影内 round 的精度、filter
阈值、grouping 内 substr 的起点/长度、ordering 内 round 的精度、limit）。

### 标准算子参考

| 算子 | 表示 | 允许位置 |
| --- | --- | --- |
| 数据源 | `source: {table, alias}` | Plan 根 |
| revision | `revision: String` | Plan 根；逐字节保留，不参与 SQL/Query |
| 列 | `'Column({alias, column})` | 投影、filter、grouping、ordering |
| 绑定值 | `'Bound(val)` | 投影、filter（比较操作数）、ordering、grouping 标量参数；不允许 grouping 顶层与 join 条件 |
| 标量调用 | `'ScalarCall({kind, args})` | 投影、filter、grouping、ordering |
| 聚合 | `'Aggregate({kind, arg})` | 投影、ordering（可嵌套在标量参数内）；不允许 filter、grouping、聚合嵌套 |
| 比较 | `'Compare({op, left, right})` | 仅 filter 顶层条件（多条条件隐式 AND） |
| 投影 | `projection: Array(ProjectionItem)` | 至少一项、alias 唯一 |
| 等值连接 | `joins: Array(Join)` | 按顺序；left 引用已定义 alias，right 必须引用本 join 的 source alias |
| 分组 | `grouping: Array(Expr)` | 顶层只允许列或标量调用；标量参数内允许绑定值 |
| 排序 | `ordering: Array(OrderItem)` | 表达式 + `'Asc`/`'Desc` |
| 限制 | `limit: Option(Int)` | 非负整数，lowering 后成为绑定值 |

标量函数 arity：`Round` 1–2 个参数，`Coalesce` ≥2，`Substr` 2–3 个参数
（`substr(X, Y[, Z])`），其余恰好 1。比较算子：`'Eq 'Ne 'Lt 'Le 'Gt 'Ge`。

### grouping 中的绑定值规则

`GROUP BY substr("o"."created_at", ?, ?)` 是合法且确定的：起始位置与长度作为
有类型绑定值进入 bindings，lowering 后与 SQL 中的 `?` 占位符一一对应。
绑定值只允许出现在 grouping 的标量调用参数内；`GROUP BY ?`（顶层绑定值）属于
结构非法，会通过 `validate_plan`/`transform_sqlite` 失败。

## 3. 验证 operators(plan) <= profile

```telora
let ops: OperatorSet = operators(plan);          # plan 实际使用的算子集合
let covered: Bool = within_profile(plan, my_profile);
let covered2: Bool = profile_covers(ops, my_profile);  # 等价
```

- `operators(plan)`：纯函数，返回去重后的 join kinds、scalar functions、
  aggregate functions 及 filter/grouping/ordering/limit 使用标志。
- `within_profile(plan, profile)`：纯函数，`operators(plan) <= profile`。
- `profile_covers(ops, profile)`：纯函数，对已计算的 OperatorSet 做同一判断。

## 4. 确定具体化为 SQLite Query

```telora
let query: Query = transform_sqlite(plan);
let query2: Query = transform_sqlite_with_profile(plan, my_profile);
```

`Query` 形状为 `{ sql: String, bindings: Array(Val) }`。转换是纯且确定的：同一个
合法 Plan 逐字节产生相同 SQL 与相同顺序的 bindings；所有运行时值都进入
bindings，SQL 中只有 `?` 占位符和双引号标识符，不承担字符串 escape。

`transform_sqlite` 只做结构验证；`transform_sqlite_with_profile` 先做 profile
覆盖检查再走同一转换路径，SQL 结果完全一致。

## 5. 判断失败来源

三种失败类别通过 `fail!` 的消息前缀区分（`invalid.telora` 展示了全部类别）：

| 前缀 | 含义 | 先验检查 |
| --- | --- | --- |
| `query-builder.structure: ...` | Plan 结构非法（非法标识符、未定义 alias、越界算子位置、投影别名重复、arity、负数 limit、grouping 顶层绑定值等） | `validate_plan(plan)` 返回 `'Err(message)` |
| `query-builder.profile: ...` | plan 使用了 profile 不接受的能力（如不允许 `'Substr`） | `within_profile(plan, profile)` 返回 `'False` |
| （无前缀，仅转换失败） | 结构合法但 SQLite 无法具体化 | 当前实现中合法结构均可具体化 |

纯检查函数不 `fail!`：`within_profile`/`profile_covers` 返回 Bool，
`validate_plan` 返回 `Result(Plan, String)`。需要分支或恢复时先调用它们；
需要“无法产出合法 Query 就失败”时直接调用 transform，它会 `fail!` 且不发布
部分 Query。

示例：

```telora
let check = validate_plan(plan);
match check {
    'Ok(_) => transform_sqlite(plan),
    'Err(message) => fail!(`custom message: \{message}`, plan),
}
```

## 确定性保证

- 同一 `Plan` 值的两次 `transform_sqlite` 产生字节相同的 SQL 与相等顺序的
  bindings。
- 投影、join、grouping、ordering、limit 的规范顺序固定：
  `SELECT ... FROM ... [JOIN ...] [WHERE ...] [GROUP BY ...] [ORDER BY ...] [LIMIT ?]`。
- bindings 顺序 = SQL 文本中 `?` 占位符的出现顺序。
- `verify.telora` 对上述保证做逐字节断言。
