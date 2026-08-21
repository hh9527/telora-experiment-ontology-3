# QueryBuilder 设计说明、验证结果与限制

## 设计选择

### 标识符与用户值分离

SQL 标识符用名义包装类型 `Ident`（`struct { text: String }`），运行时值用封闭
enum `Val`（String/Int/Float/Bool，无 Bytes）。`ident`/`source`/`col_ref`/
`col_expr` 构造器在构造点校验标识符并 `fail!`；`validate` 仍在 Plan 上重新检查
全部标识符，因此即使调用方绕过安全构造器直接构造 struct，非法标识符也会在
验证/具体化边界被拒绝。这满足 DESIGN.md「SQL 标识符与用户值必须在类型和
lowering 路径上分开」。

### 比较与逻辑算子作为标量函数

`Expr` 只需要四种变体（Column/Literal/ScalarCall/AggCall）。把
`'Eq/'Ne/'Lt/'Le/'Gt/'Ge/'And/'Or/'Not` 放进 `ScalarFunction` 后，filter 可以
统一表达为标量调用，同时 profile 能细化「允许哪些比较/逻辑算子」。渲染时按
`ScalarKind`（Infix/Prefix/Function）生成 `(left op right)`、`(NOT expr)` 或
`name(args...)`，并始终加括号避免优先级歧义。`Substr` 也在 `ScalarFunction`
中，支持 `substr(value, start)` 与 `substr(value, start, length)` 两种形式
（2 或 3 个参数），覆盖 `substr(orders.created_at, 1, 7)` 这类按月截取的用法。

### allow_distinct 语义

`PlanProfile.allow_distinct` 只约束 `AggCall.distinct`。为 `'False` 时，
`validate` 以 profile violation（`'Profile`）拒绝任何 `distinct = 'True` 的
聚合调用。本 package 不提供顶层 `SELECT DISTINCT` 算子，因此该开关不涉及
投影级去重。

### 结构化等值连接

`Join.on` 是 `Array(EquiCondition)`（列对），不接受预渲染 SQL 片段，也不携带
用户值；ON 子句因此不可能引入绑定或注入。join 保留声明顺序。

### Limit 也是绑定值

`Plan.limit: Option(Int)` 渲染为 `LIMIT ?`，值进入 bindings。这样所有动态数据
（字面量 + limit）都走参数化路径，SQL 文本不含任何用户值。

### 确定性的来源

渲染完全按 Plan 中数组的声明顺序遍历：projections、joins、group_by、
ordering；bindings 按 SQL 文本中 `?` 出现的顺序收集（SELECT 先、WHERE 其次、
GROUP BY/ORDER BY 之后、LIMIT 最后）。不使用 Dict 迭代、hash 或任何无序结构，
因此同一 Plan 必然逐字节产生同一 SQL 与同一 bindings。

### 验证失败分类

`ValidationIssue` 的四个变体对应 DESIGN.md 教程第 5 点的三个来源，并额外区分
标识符：

- `'Profile`：profile 越界（join 种类、聚合/标量函数、filter/group/order/
  limit/distinct 开关）。
- `'Structure`：非法结构（空投影、负 limit、未知源别名、深度超限、聚合/标量
  参数错误、join 别名冲突）。
- `'Identifier`：非法 SQL 标识符。
- `'Transform`：SQLite 具体化失败。当前只有 SQLite 后端且验证覆盖全部结构
  约束，此类别实际不会触发；保留它以便未来扩展多后端时区分「结构合法但该
  后端无法具体化」。

`validate` 是纯能力，返回 `Result(Plan, ValidationIssue)`；`transform_sqlite`
内部先 `validate`，违规时 `fail!` 并携带分类消息，不发布部分 Query。

### 深度上限

表达式渲染与验证使用递归；Telora 要求算法暴露语义深度界限，不能依赖 Host
fuel。因此 `max_expr_depth = 32` 作为公共常量导出，`validate` 在结构检查中
拒绝超过上限的 Plan。上层可据此把可接受的深度写成自己的契约。

### 无多后端抽象

DESIGN.md 明确本轮只实现 SQLite，不要求 PostgreSQL/MySQL 或其他后端，也不
要求提前抽象多后端插件协议。`transform_sqlite` 是单一具体化路径；`'Transform`
类别与 `PlanProfile` 的显式能力声明为未来扩展预留了接口形状，但本轮不实现
插件机制。

## 验证结果

按 GOAL.md 的验证命令逐条执行：

```text
./bin/telora run main -C query-builder
# sql: SELECT c.country AS country, count(*) AS order_count, sum(o.total) AS total_revenue FROM orders AS o INNER JOIN customers AS c ON o.customer_id = c.id WHERE (o.total > ?) GROUP BY o.customer_id, c.country ORDER BY sum(o.total) DESC LIMIT ?
# bindings: [50, 10]
```

```text
./bin/telora run verify -C query-builder
# determinism: ok / expected_sql: ok / expected_bindings: ok /
# no_inline_values: ok / clause_order: ok / standard_accepts: ok /
# select_profile_accepts_simple: ok / select_profile_rejects_join: ok /
# read_profile_rejects_aggregate: ok /
# scalar_profile_rejects_function: ok / identifier_violation: ok /
# structure_violation: ok / unknown_source_violation: ok /
# substr_rendering: ok / substr_arity: ok / allow_distinct_semantics: ok
# verify: 16 checks passed
```

```text
./bin/telora run invalid -C query-builder --best-effort
# 三条 error 诊断（profile violation: joins are not allowed by profile；
# profile violation: an aggregate function is not allowed by profile；
# plan structure violation: a column reference uses an unknown source alias）
# summary: status "error"，非零退出，无 Entry output
```

```text
./bin/telora check @test/query-builder.telora -C query-builder
# summary: status "ok"
```

```text
./bin/telora query exports @bin/main.telora -C query-builder
# output: String
```

额外验证（探针，已删除）：

- `LEFT JOIN`、`COUNT(DISTINCT col)`、String/Float/Bool 字面量、`coalesce`
  多参渲染正确；占位符数量与 bindings 数量一致。
- 深度恰好等于上限（32）被接受，超过上限（41）被 `validate` 拒绝。

`tests/query-builder.telora` 包含 26 个可执行契约检查，覆盖：validate 成功、
精确 SQL/bindings、确定性、绑定类型、占位符计数、各 profile 的接受/拒绝、
join kind 细化、标量函数细化、COUNT(*) + distinct、allow_distinct 语义、
Substr 渲染与元数（2/3 参接受，1/4 参拒绝）、revision 原样保留（String，
如 `"logistics-ontology-v1"`）、标量元数、负 limit、非法标识符、未知源
别名、空投影。

## 反馈修订

按 Host `qb-feedback` 完成以下修订并重验：

1. `ScalarFunction` 增加 `'Substr`：`validate` 只接受 2 或 3 个参数，
   `transform_sqlite` 确定性渲染为 `substr(value, start[, length])`；profile
   白名单、公共契约、教程和测试已同步覆盖。
2. 明确 `PlanProfile.allow_distinct` 只约束 `AggCall.distinct`；为 `'False`
   时 `validate` 以 `'Profile` violation 拒绝 `distinct = 'True` 的聚合，且
   不引入顶层 `SELECT DISTINCT` 算子。
3. 文档明确导入边界：包内模块/测试用 `@src`，外部依赖 crate 用
   `import "query-builder/lib.telora" as qb;`（依赖名以 manifest 为准）。

第二次 Host review 追加一项并已完成：

4. `Plan.revision` 从 `Int` 改为 `String`：题面只要求 Plan 完整保留
   knowledge revision，并未规定它是 `Int`；Ent-1 要求最终 Plan 的 revision
   为 `"logistics-ontology-v1"`。已同步更新构造、示例、测试、教程和公共
   契约；`validate` 与 `transform_sqlite` 均原样保留 revision，不引入额外
   字段或外部备注绕过 Plan 本身的 revision 契约。

非阻塞建议（profile helper、projection alias 唯一性、`read_profile` 教程
补充）按反馈未在本轮扩展。

## 限制

- 只有 SQLite 后端；`Query.sql` 的方言细节（如 `?` 占位符、`INNER JOIN` /
  `LEFT JOIN` 拼写、`LIMIT ?`）仅对 SQLite 有效。
- 标识符为单段 `[A-Za-z_][A-Za-z0-9_]*`；不支持 schema 限定名（如
  `main.orders`）、带引号标识符或 Unicode 标识符。
- `Val` 不含 Bytes（DESIGN.md 明确本轮不需要）。
- 无子查询、CTE、`DISTINCT` 投影关键字、`HAVING`、`OFFSET` 等算子；需要时
  属于后续轮次的 vocabulary 扩展。
- `revision` 是 `String`，仅原样保留（例如 Ent-1 的
  `"logistics-ontology-v1"`），不参与验证或具体化。
- `'Transform` 失败类别当前无实际触发路径（仅 SQLite 后端且验证覆盖完整）；
  多后端支持属于未来工作。
- 表达式的递归深度受 `max_expr_depth`（32）限制。
