# A2 对 QueryBuilder 公共草案的检视意见

依据 `ontology/DESIGN.md` 检视 QueryBuilder 公共候选（`QUERY-BUILDER-TUTORIAL.md`
与 `PUBLIC-CONTRACT.md`）。本次检视只使用公共教程与公共契约，并通过对公共 API
的黑盒调用（从 ontology crate 导入 `query-builder/query-builder.telora` 并运行
`check`/`run`）验证契约行为；不读取 QueryBuilder 设计、源码、tests 或 notes。

## 结论

QueryBuilder 公共交付完整覆盖 `DESIGN.md` 的 lowering 需求：标准算子词汇、
`PlanProfile` 能力声明、profile 覆盖检查、确定性的 SQLite 具体化与原子失败语义
均与设计契约一致，没有发现必须由 QueryBuilder 补齐才能实现 eDSL 的能力缺口。
首轮检视发现的 import 路径文档问题已在修订草案中修正。其余为若干 eDSL
必须遵守的算子边界（equi-join-only、标识符词法、Bound 位置、join 顺序等），
均不影响本 package 的放行。

## 需求覆盖映射

| DESIGN.md 需求 | QueryBuilder 能力 | 状态 |
| --- | --- | --- |
| Plan 至少保留 revision、基础数据源、有序投影、filter、按顺序选择的 join 及其结构化等值条件、grouping、ordering、limit | `Plan` 字段完整保留上述全部内容；join 条件为结构化 `ColumnRef` 等值 | 满足 |
| 标准算子词汇封闭、可枚举 | `JoinKind`/`CompareOp`/`ScalarFn`/`AggregateFn`/`OrderDirection` 均为封闭 enum；Expr 为封闭 enum（Column/Bound/ScalarCall/Compare/Aggregate）；修订草案新增 `'Substr` 标量 | 满足 |
| `PlanProfile` 只收窄标准算子能力、不改变算子语义 | `PlanProfile` 为纯数据声明；`within_profile`/`profile_covers` 只做集合层面判断，transform 不因 profile 改变 SQL | 满足 |
| Plan 完成后必须调用 profile 验证 | `within_profile(plan, profile)`、`transform_sqlite_with_profile(plan, profile)` 可用；越界时 `fail!` 且不发布部分 Query | 满足 |
| 企业不能提供“最终 Plan builder”；eDSL 用公共标准算子组装 Plan | 公共 API 只有纯函数与确定 transform，无任何 builder 注入点；SQL 标识符与用户值在类型和 lowering 路径上分离 | 满足 |
| 成功示例可调用 SQLite transform 展示端到端 Query，但该转换不是 eDSL API | `transform_sqlite`/`transform_sqlite_with_profile` 作为独立公共函数提供，可被示例使用 | 满足 |

## 已验证行为（黑盒）

对修订草案再次黑盒验证（scratch 文件已删除）：

- 导入路径：`import "query-builder/query-builder.telora" { ... }` 可解析并导出契约
  所列全部类型与函数（`Plan`、`PlanProfile`、`Val`、`OperatorSet`、`operators`、
  `within_profile`、`profile_covers`、`validate_plan`、`transform_sqlite`、
  `transform_sqlite_with_profile`）。
- 首轮验证：教程示例 Plan 通过 `check`；`transform_sqlite(plan)` 与
  `transform_sqlite_with_profile(plan, profile)` 生成的 SQL 逐字节一致，且与教程
  文档给出的 SQL 完全一致（含 bindings 占位符顺序）；`validate_plan` 拒绝非法
  标识符（如 `bad-column`/`bad-alias` 违反 `^[A-Za-z_][A-Za-z0-9_]*$`）与未定义
  alias 引用；`within_profile` 正确区分宽松/严格 profile。
- 修订草案新增验证：含 `'Substr` 的教程示例 Plan（投影与 grouping 均使用
  `substr("o"."created_at", ?, ?)`）通过 `validate_plan` 与 `transform_sqlite`，
  生成的 SQL 与教程文档一致；`within_profile` 在 profile 允许 `'Substr` 时为 True。
- 修订草案的 grouping 绑定值规则已验证：`GROUP BY 'Bound(...)`（顶层绑定值）被
  `validate_plan` 拒绝；绑定值出现在 grouping 的标量调用参数内则合法。

## 需要 eDSL 遵守的算子边界（非阻断）

这些是公共契约强制、eDSL 组装 Plan 时必须满足的约束：

1. **关系 mapping 只能是等值 join**：`Join` 只有 `left`/`right` 两个 `ColumnRef`
   等值条件，没有非等值/范围条件。DESIGN.md 的“结构化关系 mapping”必须在知识
   构造时约束为列等值形式；无法表达非等值关系的企业 mapping 属于 eDSL 边界，
   应显式拒绝而不是用字符串或 Any 绕过。
2. **join 顺序必须拓扑合法**：join 的 `left` 必须引用已定义 alias，`right` 必须
   引用当前 join 的 `source.alias`，alias 不得重复。eDSL 路径选择产生的关系顺序
   必须保证每个 join 的 left 由根 source 或更早的 join 定义。
3. **标识符词法**：table/column/alias 必须匹配 `^[A-Za-z_][A-Za-z0-9_]*$`。eDSL
   应在知识构造或 lowering 前校验企业标识符，避免 Plan 结构失败。
4. **Bound 位置**：动态值允许出现在投影、filter 比较操作数、ordering，以及
   grouping 的标量调用参数内（如 `GROUP BY substr(col, ?, ?)`）；不允许出现在
   grouping 顶层与 join 条件中。eDSL 不得把用户输入值用作顶层分组键或连接键。
5. **比较只在 filter 顶层**：filter 条件必须为 `'Compare(...)`，多条隐式 AND。
   无 OR/NOT/CASE/HAVING/子查询；eDSL 的过滤语义必须保持合取。
6. **聚合位置**：聚合只允许在投影与 ordering（可嵌套在标量参数内），不允许在
   filter、grouping 或另一聚合内。指标表达式必须按此位置规则组装。
7. **标量 arity**：`Round` 1–2 参、`Coalesce` ≥2、`Substr` 2–3 参、其余恰好 1；
   聚合集合为 Count/CountDistinct/Sum/Avg/Min/Max。eDSL 的物理表达式只应使用
   这些算子。
8. **limit 非负、投影非空且 alias 唯一**：eDSL 应保证任何发布的 Plan 至少有
   一个投影（空请求应失败），且投影 alias 唯一。
9. **Val 不含 Bytes**：公共 `Val` 只有 String/Int/Float/Bool。eDSL 公共边界应
   排除 Bytes（与 LANG-TUTORIAL 的 JSON 边界建议一致）。
10. **Query 无 JSON/schema 边界**：QueryBuilder 不为 `Plan`/`Query` 提供 codec 或
    schema。eDSL 若需把 Plan/Query 转成稳定 JSON（如 main 示例的 String output），
    必须由 eDSL 在自己的 crate 中建立 codec/序列化边界。

## 公共文档修订情况

- 首轮检视指出的 import 路径问题已在修订草案中修正：教程与契约均改为
  `import "query-builder/query-builder.telora" { ... }`，并注明该逻辑模块 ID 由
  manifest 固定的 package 名与 package 内模块路径组成，对应
  `<query-builder>/src/query-builder.telora`。该写法与实测一致。
- 修订草案新增 `'Substr` 标量（arity 2–3），并把绑定值规则细化为“允许出现在
  grouping 的标量调用参数内、不允许 grouping 顶层”，与实测行为一致。
- 最新草案把 `Plan.revision` 从 Int 改为稳定、可读的 String 标签（如
  `"logistics-ontology-v1"`）：由 Plan 逐字节保留，不参与 profile/validation
  判定，也不出现在生成的 SQL/Query 中。实测确认：String revision 的 Plan 可
  正常通过 `validate_plan`/`within_profile`/`transform_sqlite`，且 SQL 文本不含
  revision。eDSL 已相应把 `EnterpriseKnowledge.revision` 建模为 String 标签并
  原样写入 Plan。

## 放行建议

- 公共契约满足 DESIGN.md 的全部 lowering 需求，建议放行。
- eDSL 设计（`edsl.a2`）将把上述 1–10 的算子边界固化为知识构造期的校验
  （等值 mapping、标识符词法、表达式位置规则、空请求拒绝），并把
  `within_profile`/`transform_sqlite_with_profile` 作为 Plan 发布的最终关卡。
