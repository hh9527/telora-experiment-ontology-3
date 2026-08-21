# QueryBuilder 公共契约

本文件描述 `query-builder` package 的稳定公共类型、能力与保证。它是领域无关
的基础 package：只处理标准 Plan 算子、PlanProfile、验证和 SQLite 具体化，
不包含 ontology 概念或企业领域事实。

## 导入边界

本 package 内的模块和测试使用 `@src` 源码根路径：

```telora
import "@src/lib.telora" as qb;
```

外部依赖 crate 必须使用 manifest 固定的依赖名（依赖名以 manifest 为准，
示例中为 `query-builder`），不能照抄 `@src`：

```telora
import "query-builder/lib.telora" as qb;
```

公共入口是 `@src/lib.telora`（也按模块 `@src/types.telora`、
`@src/profile.telora`、`@src/validate.telora`、`@src/render.telora` 提供）。

## 稳定公共类型

```telora
# 运行时绑定值（不含 Bytes）
type Val = enum {
    'String(String),
    'Int(Int),
    'Float(Float),
    'Bool(Bool),
};

# 具体化结果
type Query = struct {
    sql: String,
    bindings: Array(Val),
};

# 校验后的 SQL 标识符（单段，[A-Za-z_][A-Za-z0-9_]*）
type Ident = struct { text: String };

type Source = struct { name: Ident, alias: Ident };
type ColumnRef = struct { source: Ident, column: Ident };

type ScalarFunction = enum {
    'Abs, 'Round, 'Floor, 'Ceil,
    'Lower, 'Upper, 'Length, 'Trim, 'Substr, 'Coalesce,
    'Eq, 'Ne, 'Lt, 'Le, 'Gt, 'Ge,
    'And, 'Or, 'Not,
};
type AggFunction = enum { 'Count, 'Sum, 'Avg, 'Min, 'Max };

type AggCall = struct {
    function: AggFunction,
    arg: Option(Expr),       # 'None 仅用于 COUNT(*)
    distinct: Bool,
};
type ScalarCall = struct {
    function: ScalarFunction,
    args: Array(Expr),
};
type Expr = enum {
    'Column(ColumnRef),
    'Literal(Val),
    'ScalarCall(ScalarCall),
    'AggCall(AggCall),
};

type JoinKind = enum { 'Inner, 'Left };
type EquiCondition = struct { left: ColumnRef, right: ColumnRef };
type Join = struct {
    kind: JoinKind,
    right: Source,
    on: Array(EquiCondition),
};
type OrderDirection = enum { 'Asc, 'Desc };
type OrderItem = struct { expr: Expr, direction: OrderDirection };
type Projection = struct { expr: Expr, alias: Ident };

type Plan = struct {
    revision: String,
    source: Source,
    projections: Array(Projection),
    filter: Option(Expr),
    joins: Array(Join),
    group_by: Array(Expr),
    ordering: Array(OrderItem),
    limit: Option(Int),
};

type PlanProfile = struct {
    allow_joins: Bool,
    join_kinds: Array(JoinKind),
    allow_filter: Bool,
    allow_group_by: Bool,
    allow_order_by: Bool,
    allow_limit: Bool,
    allow_distinct: Bool,
    scalar_functions: Array(ScalarFunction),
    aggregate_functions: Array(AggFunction),
};

type ValidationIssue = enum {
    'Profile(String),
    'Structure(String),
    'Identifier(String),
    'Transform(String),
};
```

## 公共函数

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `ident` | `Fn(String) -> Ident` | 校验并包装标识符；非法输入 `fail!` |
| `is_sql_identifier` | `Fn(String) -> Bool` | 整串匹配标识符规则 |
| `source` | `Fn(String, String) -> Source` | 表名 + 别名 |
| `col_ref` | `Fn(String, String) -> ColumnRef` | 别名 + 列名 |
| `col_expr` | `Fn(String, String) -> Expr` | 列表达式 |
| `lit` | `Fn(Val) -> Expr` | 绑定字面量表达式 |
| `scalar_call` | `Fn(ScalarFunction, Array(Expr)) -> Expr` | 标量调用 |
| `agg_call` | `Fn(AggFunction, Option(Expr), Bool) -> Expr` | 聚合调用 |
| `standard_profile` | `PlanProfile` | 完整 SQLite 能力集 |
| `select_profile` | `PlanProfile` | 仅 SELECT/FROM/WHERE |
| `read_profile` | `PlanProfile` | 读查询：无 join/group/聚合/distinct |
| `with_join_kinds` | `Fn(PlanProfile, Array(JoinKind)) -> PlanProfile` | 派生 |
| `with_scalar_functions` | `Fn(PlanProfile, Array(ScalarFunction)) -> PlanProfile` | 派生 |
| `with_aggregate_functions` | `Fn(PlanProfile, Array(AggFunction)) -> PlanProfile` | 派生 |
| `join_kind_allowed` | `Fn(Array(JoinKind), JoinKind) -> Bool` | 成员判断 |
| `scalar_function_allowed` | `Fn(Array(ScalarFunction), ScalarFunction) -> Bool` | 成员判断 |
| `aggregate_function_allowed` | `Fn(Array(AggFunction), AggFunction) -> Bool` | 成员判断 |
| `validate` | `Fn(Plan, PlanProfile) -> Result(Plan, ValidationIssue)` | 纯验证 |
| `transform_sqlite` | `Fn(Plan, PlanProfile) -> Query` | 验证并具体化 |
| `max_expr_depth` | `Int`（= 32） | 表达式深度上限 |

## 语义保证

1. **封闭 vocabulary**：算子类型全部是封闭的具名 enum/struct；同名算子语义
   不随 profile 或调用方变化。
2. **Profile 是能力声明**：`PlanProfile` 不改变任何算子语义；只决定
   `validate` 是否接受 Plan。`allow_distinct` 只约束 `AggCall.distinct`：
   为 `'False` 时，任何 `distinct = 'True` 的聚合调用都被视为 profile
   violation（`'Profile`）。本 package 不提供顶层 `SELECT DISTINCT` 算子。
3. **纯验证**：`validate` 无副作用、不 `fail!`、不发布部分 Query；按
   Identifier、Structure、Profile 顺序返回第一个失败。
4. **确定性具体化**：同一个合法 Plan 逐字节产生相同 `sql` 与相同顺序的
   `bindings`；所有运行时值（字面量与 limit）进入 bindings，SQL 中只有
   `?` 占位符。
5. **失败原子性**：`transform_sqlite` 遇到任何违规以 `fail!` 失败并带分类
   消息；不产生部分 Query。
6. **类型边界**：公共 API 不使用 `Any`、`Dyn`、native 声明；SQL 标识符
   （`Ident`）与用户值（`Val`）在类型与 lowering 路径上完全分开。
7. **revision 原样保留**：`Plan.revision` 是 `String`，由调用方持有（例如
   Ent-1 的 `"logistics-ontology-v1"`）；`validate` 与 `transform_sqlite`
   都不修改它，也不参与验证或具体化。不得用额外字段或外部备注绕过 Plan
   本身的 revision 契约。

## 结构约束（validate 强制）

- 至少一个 projection。
- `limit >= 0`。
- 表达式深度 `<= max_expr_depth`（32）。
- 聚合：`COUNT(*)` 必须 `distinct = 'False`；其他聚合必须有参数。
- 标量元数：一元 `Abs/Floor/Ceil/Lower/Upper/Length/Trim/Not`；`Round` 1-2
  参；`Substr` 2-3 参（`substr(value, start)` 与 `substr(value, start,
  length)`）；`Coalesce >= 1` 参；二元 `Eq/Ne/Lt/Le/Gt/Ge/And/Or`。
- join 别名互不相同，且不等于基础源别名。
- 每个列引用必须指向基础源别名或某个 join 右源别名。
- 标识符匹配 `[A-Za-z_][A-Za-z0-9_]*`。

## SQLite 输出形状

`SELECT <projections> FROM <source> [<JOIN> ...] [WHERE <filter>] [GROUP BY <group_by>] [ORDER BY <ordering>] [LIMIT ?]`

- 投影：`<expr> AS <alias>`，按声明顺序，逗号连接。
- join：`INNER JOIN <right> ON <l1> = <r1> AND ...` 或
  `LEFT JOIN ...`，按声明顺序。
- 标量：函数式 `name(args...)`；中缀 `(left op right)`；前缀 `(NOT expr)`。
- 聚合：`count(*)`、`count(DISTINCT expr)`、`sum(expr)` 等。
- bindings 顺序 = SQL 文本中 `?` 的出现顺序。
