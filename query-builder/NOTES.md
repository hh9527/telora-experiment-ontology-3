# QueryBuilder 设计笔记

## 交付物

- `query-builder/src/query-builder.telora`：领域无关的可复用 package（类型、
  验证、operator 收集、profile 覆盖、SQLite 具体化）。
- `query-builder/src/bin/main.telora`：合法 Plan 到参数化 SQLite Query 的展示。
- `query-builder/src/bin/verify.telora`：profile 覆盖、算子收集、结构验证、
  确定性、规范顺序、bindings/占位符一致性的逐字节断言。
- `query-builder/src/bin/invalid.telora`：越界或非法 Plan 的 Host 诊断展示，
  有意失败、无 output。
- `query-builder/tests/query-builder.telora`：公共类型与模块契约检查。
- `query-builder/QUERY-BUILDER-TUTORIAL.md`、`PUBLIC-CONTRACT.md`、`NOTES.md`。

## 设计选择

### 单模块 package

所有公共类型与函数集中在 `@src/query-builder.telora`，避免跨模块递归类型
（`Expr -> ScalarCall -> Expr`）与 family 解析带来的额外复杂度，公共边界清晰。
依赖方以逻辑模块 ID `query-builder/query-builder.telora` 导入。

### 封闭 enum 表达算子语义

算子种类（`JoinKind`、`CompareOp`、`ScalarFn`、`AggregateFn`、
`OrderDirection`）全部是封闭 enum，而不是 String。这满足“公共边界不得以 String
充当所有语义身份”，并让 `operators(plan)` 可以返回精确的去重算子集合。
String 只用于 SQL 标识符（表名、列名、alias），它们本质是 SQL 文本标识符，
与用户值（`Val`）在类型与 lowering 路径上分开。

`ScalarFn` 包含 SQLite 的 `substr(X, Y[, Z])`（2–3 个参数），使
`substr(orders.created_at, 1, 7)` 这类合法题面可以结构化表达：起点与长度作为
`'Bound('Int(...))` 绑定值，lowering 后进入 bindings。

### filter 是隐式 AND 的条件数组

`filter: Array(Expr)`，每个元素必须是顶层 `'Compare`，多条条件以 ` AND `
连接。这是常见查询构建器设计：不需要布尔连接算子（OR/NOT），结构验证简单、
语义确定。作为限制记录在本笔记末尾。

### PlanProfile 是显式能力声明

`PlanProfile` 列出允许的 join/scalar/aggregate 能力与 feature 开关；它不是
隐式全局状态，不参与 lowering，只影响 `within_profile`/`transform_sqlite_with_profile`
的判定。`operators(plan)` 计算实际使用集合，`profile_covers` 做集合包含判断。

### grouping 中的绑定值规则

`GROUP BY substr("o"."created_at", ?, ?)` 必须合法：起始位置与长度是运行时值，
需要绑定。实现采用两层规则：grouping 顶层只允许列或标量调用；标量调用参数内的
绑定值允许（`validate_grouping_root` 先检查顶层 variant，再以
`allow_bound: 'True` 递归验证）。`GROUP BY ?`（顶层绑定值）仍属结构非法。
该规则在公共契约与教程中明确记录，且保持确定性：bindings 顺序与 `?` 占位符
顺序一致。

### 失败分类与纯检查

- 纯检查：`operators`（总函数）、`within_profile`/`profile_covers`（Bool）、
  `validate_plan`（`Result(Plan, String)`）。
- 失败路径：`transform_sqlite`/`transform_sqlite_with_profile` 在无法产出合法
  Query 时 `fail!`，消息前缀 `query-builder.structure:`/`query-builder.profile:`
  便于 eDSL 作者区分失败来源（教程第 5 步）。

### 确定性 lowering

- 渲染按规范顺序：SELECT 投影 → FROM 数据源 → JOIN（按顺序）→ WHERE →
  GROUP BY → ORDER BY → LIMIT。
- 占位符为 SQLite 位置参数 `?`；bindings 顺序 = `?` 出现顺序，与 SQL 文本
  逐位对应。
- 所有标识符双引号包裹（`"alias"."column"`），避免 SQLite 关键字冲突，且
  bare identifier 词法已由 `std/regex` 校验，无引号注入风险。
- 运行时值（`'Bound(Val)` 与 `limit` 整数）全部进入 bindings，SQL 不承担
  escape。

### 结构验证规则

在 `validate_plan` 中一次性完成：标识符词法、投影非空且 alias 唯一、join
顺序与左右侧 alias 约束、表达式位置约束（比较仅 filter 顶层；聚合仅投影/
ordering 且不可嵌套；绑定值不允许 grouping 顶层/join 条件、允许 grouping
标量参数；标量 arity）、limit 非负。所有错误返回带上下文的 String 诊断。

## 送审修订（qb-feedback）

### 第一轮

- 新增 `ScalarFn.'Substr`（`substr(X, Y[, Z])`，arity 2–3）。
- grouping 允许标量调用参数内的绑定值，使 `substr(orders.created_at, 1, 7)`
  可合法出现在 projection 与 grouping 并确定 lowering。
- 教程修正依赖导入示例为 `import "query-builder/query-builder.telora" { ... };`。
- 增补测试：三参数 Substr 的 projection/grouping、SQLite SQL 与 bindings、
  arity 越界（1 参数与 4 参数）、profile 不允许 `'Substr`。

### 第二轮

- `Plan.revision` 从 `Int` 改为 `String`：公共类型直接、逐字节保留稳定可读的
  revision 标签（如 `"logistics-ontology-v1"`），不引入 `Any`/`Dyn` 或复杂版本
  协议。
- revision 不参与 profile/validation 判定，也不出现在生成的 SQL/Query 中；
  transform/profile/validation 行为不受影响。
- 示例、fixtures、测试、教程、公共契约与 notes 全部同步更新；`main`/`verify`/
  `tests` 增加 `plan.revision == "logistics-ontology-v1"` 断言。

## 验证结果

全部按 GOAL.md 验证命令执行：

```text
./bin/telora run main -C query-builder
  -> 输出 Plan revision、profile 覆盖、SQL 与 bindings，退出 0

./bin/telora run verify -C query-builder
  -> "verify: ok"；包含 profile 覆盖、算子收集、结构验证、确定性
     （两次 transform 逐字节一致）、规范顺序（与固定 SQL 完全一致）、
     bindings 数与占位符一致，退出 0

./bin/telora run invalid -C query-builder --best-effort
  -> 11 次独立失败尝试产生 Host 诊断（status: error），无 output，
     非零退出；覆盖结构类别（未定义 alias、非法表标识符、投影 alias
     重复、filter 内聚合、负数 limit、join 右侧错误、substr arity 越界、
     grouping 顶层绑定值）与 profile 类别（不允许 join/Sum/Substr）

./bin/telora check @test/query-builder.telora -C query-builder
  -> status: ok（公共类型可引用、结构验证 Ok/Err、operators、profile
     覆盖、确定性、SQL 形状、bindings/占位符、Substr 成功与失败用例、
     失败分类）

./bin/telora show @bin/main.telora -C query-builder
  -> telora.show/v1 记录正常输出
```

示例 SQL（main/verify）：

```sql
SELECT "o"."region" AS "region", substr("o"."created_at", ?, ?) AS "month", round(SUM("o"."amount"), ?) AS "total"
FROM "orders" AS "o"
JOIN "customers" AS "c" ON "o"."customer_id" = "c"."id"
WHERE ("o"."amount" > ?)
GROUP BY substr("o"."created_at", ?, ?)
ORDER BY round(SUM("o"."amount"), ?) DESC
LIMIT ?
```

bindings：`[1, 7, 2, 500, 1, 7, 2, 10]`（与 `?` 占位符顺序一一对应）。

## 限制

- 仅 SQLite 后端；不抽象多后端插件协议。
- filter 仅支持顶层比较条件（隐式 AND）；无 OR/NOT、CASE、子查询、HAVING。
- 排序不支持引用投影 alias（需重复表达式）；join 条件必须是结构化等值
  （ColumnRef = ColumnRef），不支持任意表达式。
- grouping 顶层不允许绑定值（`GROUP BY ?` 非法），但标量调用参数内的绑定值
  允许（如 `substr(col, ?, ?)`）。
- 聚合不可嵌套（如 `SUM(COUNT(...))`），也不允许在 filter/grouping 中。
- 投影必须非空且 alias 唯一。
- `Val` 不支持 Bytes；公共类型不含 `Any`/`Dyn`/native。
- `invalid.telora` 的多个 profile 失败共享同一 `fail!` 消息，Host 诊断可能按
  消息去重；不影响“越界 Plan 产生诊断且无 Query”的验收。
