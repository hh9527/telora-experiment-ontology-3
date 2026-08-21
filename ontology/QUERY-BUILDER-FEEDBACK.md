# QueryBuilder 公共草案检视意见（qb-feedback.a2）

- 检视对象：`query-builder/QUERY-BUILDER-TUTORIAL.md`、`query-builder/PUBLIC-CONTRACT.md`
- 检视依据：`ontology/DESIGN.md`
- 轮次：第 3 轮（第 1 轮两项阻塞问题已在第 2 轮解决；本轮修订将
  `Plan.revision` 从 `Int` 改为 `String` 并补充语义保证 #7，见 §2.3）

## 总体结论

草案可以支撑 ontology eDSL 的全部 lowering 需求。算子 vocabulary、`PlanProfile`
能力声明、纯验证 `validate` 与确定性具体化 `transform_sqlite` 与 DESIGN.md 的
五步流程吻合；公共 API 的类型边界（封闭 nominal 类型、无 `Any`/`Dyn`/native、
`Val` 不含 Bytes）也符合 DESIGN.md 的公共边界要求。

前两轮的阻塞问题均已解决（§2.1、§2.2）；本轮 `Plan.revision` 改为 `String` 是
兼容性改进（§2.3），已由 A2 在 eDSL 中承接并重新验证。剩余 4 项为非阻塞建议
（§3），是否采纳不影响 eDSL 实现。

## 1. DESIGN.md 需求覆盖对照

| DESIGN.md 要求 | QueryBuilder 支撑 | 结论 |
| --- | --- | --- |
| Plan 保留 knowledge revision | `Plan.revision: String`（调用方持有，原样保留，不参与验证/具体化） | 满足 |
| 基础数据源 | `Plan.source: Source`（表名 + 别名） | 满足 |
| 有序 measure/dimension 投影 | `Plan.projections: Array(Projection)`，顺序保留 | 满足 |
| 与维度一致的 grouping | `Plan.group_by: Array(Expr)` | 满足 |
| 按规范顺序选择的关系和 mapping | `Plan.joins: Array(Join)` 顺序保留；mapping 表达为 `EquiCondition` 列对 + 投影表达式 | 满足 |
| 成功时发布 Plan，失败时 `fail!` 且不发布部分 Plan | `transform_sqlite` 违规时 `fail!`；`validate` 纯验证不发布部分结果 | 满足 |
| 验证 Plan 只使用 `knowledge.plan_profile` 接受的能力 | `validate(plan, profile) -> Result(Plan, ValidationIssue)` | 满足 |
| 成功示例可端到端展示 Query | `transform_sqlite(plan, profile) -> Query`（`sql` + `bindings`） | 满足 |
| `PlanProfile` 只收窄标准算子能力，不改变算子语义 | profile 字段均为显式能力开关/白名单 | 满足 |
| 公共 API 无 `Any`/`Dyn`/native；标识符与用户值类型分离 | `Ident` 与 `Val` 分开；全部封闭具名类型 | 满足 |
| 禁止预渲染 SQL / String 反查 | join 条件仅 `EquiCondition`，无 SQL 片段；`Ident` 是校验后包装，非领域查询键 | 满足 |

`'Substr`（2-3 参）是原 vocabulary 的超集，arity 已在结构约束中定义，不改变
既有算子语义，无问题。路径选择、grain 兼容性、授权等属于 ontology 侧知识，
不需要 Plan 表达；`Plan.joins` 对 join 链长度无文档限制，足以承载 eDSL 的
最大深度 8 的路径。

## 2. 阻塞问题的解决确认

### 2.1 `allow_distinct` 与 `AggCall.distinct` 的关系 —— 已解决（第 2 轮）

`allow_distinct` 只约束 `AggCall.distinct`；为 `'False` 时，任何
`distinct = 'True` 的聚合调用都会被 `validate` 以 `'Profile` violation 拒绝；
本 package 不提供顶层 `SELECT DISTINCT` 算子（教程 §1 与契约语义保证 #2 一致）。

eDSL 承接：企业知识若将 `allow_distinct` 设为 `'False`，lowering 不得生成
`AggCall.distinct = 'True`；验证步骤可直接依赖 `validate` 得到 `'Profile`。

### 2.2 外部 crate 的 import 路径 —— 已解决（第 2 轮）

"导入边界"说明 package 内模块/测试用 `@src/lib.telora`；外部依赖 crate 必须
使用 manifest 固定的依赖名（示例 `query-builder/lib.telora`）。ontology 的
`telora-deps.json` 已声明 `query-builder` 依赖，按
`import "query-builder/lib.telora" as qb;` 导入。

### 2.3 `Plan.revision` 改为 `String` —— 本轮修订，已承接

修订版将 `Plan.revision` 从 `Int` 改为 `String`，并新增语义保证 #7：revision
由调用方持有（例如 `"logistics-ontology-v1"`），`validate` 与
`transform_sqlite` 都不修改它，也不参与验证或具体化；不得用额外字段或外部备注
绕过 Plan 本身的 revision 契约。

检视意见：该改动与 DESIGN.md"Plan 至少保留 knowledge revision"一致，且明确
revision 是调用方（eDSL）持有的不透明标识，正是企业知识版本号的自然建模。
A2 已相应把 `EnterpriseKnowledge.revision` 定为 `String` 并原样写入
`qb.Plan.revision`，重新通过全部验证（`run main` / `run verify` /
`run invalid --best-effort` / `check @test/ontology.telora`）。

## 3. 仍开放的非阻塞建议

以下建议不影响 eDSL 实现，供 QueryBuilder 后续版本参考；不采纳也不阻塞放行。

### 3.1 profile 派生 helper 只覆盖三类白名单

`with_join_kinds` / `with_scalar_functions` / `with_aggregate_functions` 只派生
白名单；`allow_joins`、`allow_filter`、`allow_group_by`、`allow_order_by`、
`allow_limit`、`allow_distinct` 等开关没有 `with_*` helper。eDSL 作者可以构造
`PlanProfile` 记录字面量关闭这些能力（字段都是公开的），但补一组 `with_allow_*`
helper 会降低误写风险。

### 3.2 `validate` 结构约束未列出投影 alias 唯一性

约束列表覆盖了 join 别名冲突和列引用合法性，但没有要求 projection alias 唯一。
eDSL 会自行生成唯一别名，不构成阻塞；建议在结构约束中明确（允许或拒绝），
使具体化 SQL 的语义对调用方透明。

### 3.3 `read_profile` 未在教程中出现

公共契约列出了 `read_profile`（读查询：无 join/group/聚合/distinct），教程未
提及。它可能是 eDSL 作者常用 profile（点查/单实体查询），建议在教程 §1 中
给出与 `select_profile` 的对比。

### 3.4 教程未说明 `ValidationIssue` 不可插值之外的处理模式

教程给出了 `classify` 示例（match 后转文本），很好；建议在 §5 或"边界与限制"
中补一句：eDSL 应在 `validate` 返回 `'Err` 时以 `fail!(...)` 携带 issue 文本，
而不要试图把 `ValidationIssue` 本身作为返回值/诊断对象（DESIGN.md 的公共 API
不返回 Rejection 或诊断数组）。

## 4. 对 eDSL 的承接

A2 已按放行版草案实现并验证，具体承接方式：

- `EnterpriseKnowledge.plan_profile: qb.PlanProfile` 由企业知识持有；
- `EnterpriseKnowledge.revision: String` 原样写入 `qb.Plan.revision`；
- lowering 以 `qb` 的标准算子构造 `qb.Plan`（投影、join、group_by 等），
  完成后调用 `qb.validate(plan, knowledge.plan_profile)` 做 profile 验证，
  失败按 `ValidationIssue` 分类 `fail!`；
- 按 2.1 的明确语义，`allow_distinct = 'False` 时绝不生成 distinct 聚合；
- 成功示例调用 `qb.transform_sqlite(plan, knowledge.plan_profile)` 展示端到端
  Query，但该转换不作为 eDSL API。

A2 对 `qb.a1` 的检视结论：**放行**。
