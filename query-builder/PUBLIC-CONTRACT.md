# QueryBuilder 公共契约

QueryBuilder 是领域无关的基础 package：定义稳定、方言中立的 `Plan`，并内置
SQLite 的具体化支持（`Plan -> Query`）。它不了解 ontology、企业实体、指标、
维度或查询题面。本实验只支持 SQLite，不要求其他 SQL 后端。

公共模块：`@src/query-builder.telora`（crate `query-builder`）。依赖方通过逻辑
模块 ID `query-builder/query-builder.telora` 导入（对应
`<query-builder>/src/query-builder.telora`）。

## 公共类型

```telora
type Val = enum {
    'String(String),
    'Int(Int),
    'Float(Float),
    'Bool(Bool),
};

type DataSource = struct {
    table: String,
    alias: String,
};

type ColumnRef = struct {
    alias: String,
    column: String,
};

type JoinKind = enum { 'Inner, 'Left };

type CompareOp = enum { 'Eq, 'Ne, 'Lt, 'Le, 'Gt, 'Ge };

type ScalarFn = enum {
    'Lower, 'Upper, 'Length, 'Abs, 'Round, 'Coalesce, 'Substr,
};

type AggregateFn = enum {
    'Count, 'CountDistinct, 'Sum, 'Avg, 'Min, 'Max,
};

type OrderDirection = enum { 'Asc, 'Desc };

type ScalarCall = struct {
    kind: ScalarFn,
    args: Array(Expr),
};

type CompareExpr = struct {
    op: CompareOp,
    left: Expr,
    right: Expr,
};

type AggregateCall = struct {
    kind: AggregateFn,
    arg: Expr,
};

type Expr = enum {
    'Column(ColumnRef),
    'Bound(Val),
    'ScalarCall(ScalarCall),
    'Compare(CompareExpr),
    'Aggregate(AggregateCall),
};

type ProjectionItem = struct {
    expr: Expr,
    alias: String,
};

type Join = struct {
    kind: JoinKind,
    source: DataSource,
    left: ColumnRef,
    right: ColumnRef,
};

type OrderItem = struct {
    expr: Expr,
    direction: OrderDirection,
};

type Plan = struct {
    revision: String,
    source: DataSource,
    projection: Array(ProjectionItem),
    filter: Array(Expr),
    joins: Array(Join),
    grouping: Array(Expr),
    ordering: Array(OrderItem),
    limit: Option(Int),
};

type Query = struct {
    sql: String,
    bindings: Array(Val),
};

type PlanProfile = struct {
    allow_joins: Array(JoinKind),
    allow_scalar: Array(ScalarFn),
    allow_aggregate: Array(AggregateFn),
    allow_filter: Bool,
    allow_grouping: Bool,
    allow_ordering: Bool,
    allow_limit: Bool,
};

type OperatorSet = struct {
    joins: Array(JoinKind),
    scalars: Array(ScalarFn),
    aggregates: Array(AggregateFn),
    uses_filter: Bool,
    uses_grouping: Bool,
    uses_ordering: Bool,
    uses_limit: Bool,
};
```

`Option(Int)` 与 `Result(Plan, String)` 使用 Telora prelude 的 `Option`/`Result`
类型。`Val` 不支持 Bytes；`Query.bindings` 是 `Array(Val)`。

## 公共函数

| 函数 | 签名 | 语义 |
| --- | --- | --- |
| `operators` | `Fn(Plan) -> OperatorSet` | 计算 plan 实际使用的算子集合（去重、首次出现顺序）；纯、总函数 |
| `profile_covers` | `Fn(OperatorSet, PlanProfile) -> Bool` | `operators(plan) <= profile` 的集合层面判断；纯 |
| `within_profile` | `Fn(Plan, PlanProfile) -> Bool` | `profile_covers(operators(plan), profile)`；纯 |
| `validate_plan` | `Fn(Plan) -> Result(Plan, String)` | 纯结构验证：标识符、引用、算子位置、arity、limit |
| `transform_sqlite` | `Fn(Plan) -> Query` | 结构验证后确定具体化为 SQLite Query；非法结构 `fail!` |
| `transform_sqlite_with_profile` | `Fn(Plan, PlanProfile) -> Query` | profile 覆盖检查 + `transform_sqlite`；越界 `fail!` |

公共 API 不使用 `Any`、`Dyn`、native 声明，也不以 String 充当算子语义身份：
算子种类（join kind、比较、标量函数、聚合函数、排序方向）都是封闭 enum，
String 只用于 SQL 标识符（表名、列名、alias）与错误消息。

## 能力与保证

### Plan 构造

- Plan 至少完整保留：revision、基础数据源、有序投影、filter、按顺序选择的
  join 及其结构化等值条件、grouping、ordering、limit。
- `revision` 是稳定、可读的 String 标签（如 `"logistics-ontology-v1"`），由
  Plan 自身逐字节保留；它不参与 profile/validation 判定，也不出现在生成的
  SQL/Query 中。
- 动态数据只能作为有类型绑定值（`'Bound(Val)` 与 `limit`），不能预先 escape
  或拼入 SQL；SQL 标识符与用户值在类型和 lowering 路径上分开。
- 不接受预渲染的 SELECT/JOIN/GROUP BY 片段作为表达式。

### 标准算子 vocabulary

封闭、可枚举，至少覆盖：数据源、列与绑定值表达式、标量调用、聚合、投影、
过滤、等值连接、分组、排序、限制。同名算子的语义不随 profile 或调用方变化。

算子位置规则（结构验证强制）：

- 比较只允许作为 filter 顶层条件；多条 filter 条件隐式 AND。
- 聚合允许在投影与 ordering（可嵌套在标量参数内）；不允许在 filter、grouping
  或另一聚合的参数内。
- 绑定值允许在投影、filter 比较操作数、ordering，以及 grouping 的标量调用参数
  内（如 `GROUP BY substr(col, ?, ?)`）；不允许在 grouping 顶层与 join 条件中。
- join 条件 left 必须引用已定义 alias，right 必须引用当前 join 的 source
  alias；alias 不得重复。
- 投影必须非空，投影 alias 唯一；标识符须匹配 SQLite bare identifier 词法
  `^[A-Za-z_][A-Za-z0-9_]*$`；limit 非负；标量 arity 固定（Round 1–2，
  Coalesce ≥2，Substr 2–3，其余 1）。

### PlanProfile

- Profile 是标准算子能力的显式子集声明，可细化允许的 join kind、aggregate
  function、scalar function 及 filter/grouping/ordering/limit。
- Profile 不是隐式全局状态，也不改变算子语义；它只决定 `within_profile` 与
  `transform_sqlite_with_profile` 的判定。

### Query 与确定性转换

- `Query` 形状为 `{ sql: String, bindings: Array(Val) }`。
- SQLite transform 纯且确定：同一合法 Plan 逐字节产生相同 SQL 与相同顺序的
  bindings。
- SQL 只生成目标方言的占位符（`?`）与双引号标识符，不承担字符串 escape；
  所有运行时值按占位符出现顺序进入 bindings。
- 投影、join、grouping、ordering 的规范顺序为：
  `SELECT ... FROM ... [JOIN ...] [WHERE ...] [GROUP BY ...] [ORDER BY ...] [LIMIT ?]`。

### 失败语义

- 发现越界算子、非法结构、无效标识符或未绑定值时使用 `fail!`，不发布部分
  Query；最终结果原子发布。
- 失败类别通过消息前缀区分：`query-builder.structure:`（结构）、
  `query-builder.profile:`（profile 越界）。
- 纯检查函数 `within_profile`/`profile_covers` 返回 Bool，`validate_plan`
  返回 `Result(Plan, String)`；需要恢复或分支时先使用纯检查，需要失败时直接
  使用 transform。

## 非目标

- 不含 ontology、企业实体、指标、维度或查询题面概念。
- 不扩展 PostgreSQL/MySQL 或其他 SQL 后端，不抽象多后端插件协议。
- `Val` 不支持 Bytes；无子查询、HAVING、OR/NOT、CASE 或投影 alias 引用排序。
- 不提供 JSON/schema 边界；codec 与 JSON 表示由使用方在其 crate 中建立。
