# EnterpriseKnowledge eDSL 教程

本文面向第一次使用本 eDSL 的企业知识作者。目标是让企业以结构化方式表达
`EnterpriseKnowledge`，并获得稳定的 lowering：

```text
make_query_creator:
  EnterpriseKnowledge -> Fn(QueryRequest) -> Plan
```

`Plan`、`PlanProfile`、`Query` 与标准算子由 QueryBuilder package 所有；本 eDSL
只负责企业知识的建模与 lowering 规则，不定义替代 Plan，不负责 `Plan -> Query`
（但成功示例可以调用 QueryBuilder 的 `transform_sqlite` 展示端到端 Query）。

本教程覆盖四类业务意图：投影（measure/dimension）、有类型维度筛选
（`Eq`/`Ge`/`Le`，`And` 组合）、排序（已请求目标的 `Asc`/`Desc`）与 Top N
（可选正整数 limit）。

## 0. 导入

本 eDSL 公共入口是 `ontology/src/lib.telora`。外部 crate 使用 manifest 依赖名：

```telora
import "ontology/lib.telora" as e;
import "query-builder/lib.telora" as qb;
```

本 crate 内的模块使用 `@src` 路径：

```telora
import "@src/lib.telora" as e;
import "query-builder/lib.telora" as qb;
```

下文示例假设已经 `import "@src/lib.telora" as e;`。

## 1. 定义领域的 ID 类型

eDSL 的全部目录都以有类型的 ID 引用。先用封闭 enum 定义领域词汇：

```telora
type MeasureId = enum { 'Revenue, 'AwardCount };
type DimensionId = enum { 'CourseCategory, 'EnrollmentMonth };
type EntityId = enum { 'Enrollments, 'Courses, 'CertificateAwards };
type AttributeId = enum { 'EnrollmentId, 'AmountPaid, 'CourseId, 'Category, 'CreatedAt, 'CertificateId };
type RelationId = enum { 'EnrollCourse, 'CertCourse };
type Subject = String;
type MeasureInput = enum { 'All };
type DimensionInput = enum { 'All };
type FilterInput = enum { 'Category(String), 'Month(String) };
```

ID 类型由企业定义，eDSL 完全参数化：`Knowledge` 与 `QueryRequest` 都是带这些
类型参数的 family。`FilterInput` 是筛选请求的领域输入类型；不要用 String 标识
代替 ID——所有目录查找都是类型级身份，不是 String 反查。

## 2. 建模实体与属性

每个实体有一个物理数据源（表名 + 别名）和属性目录（属性 -> 物理列）：

```telora
let enrollments = e.entity('Enrollments, e.source_ref("enrollments", "en"), [
  e.attribute('EnrollmentId, 'Enrollments, "enrollment_id"),
  e.attribute('AmountPaid, 'Enrollments, "amount_paid"),
  e.attribute('CourseId, 'Enrollments, "course_id"),
  e.attribute('CreatedAt, 'Enrollments, "created_at"),
]);

let courses = e.entity('Courses, e.source_ref("courses", "co"), [
  e.attribute('CourseId, 'Courses, "course_id"),
  e.attribute('Category, 'Courses, "category"),
]);
```

- `e.source_ref(table, alias)`：物理表名 + 别名。别名会进入最终 SQL，必须唯一且
  是合法 SQL 标识符。
- `e.attribute(id, entity, column)`：属性属于哪个实体、对应哪一列。

## 3. 建模关系与 grain

关系是有向边（`left -> right`），携带企业拥有的结构化 mapping（等值列对）：

```telora
let enroll_course = e.relation(
  'EnrollCourse,                 # RelationId
  'Enrollments,                  # left 实体
  'Courses,                      # right 实体
  e.relation_mapping([
    e.attribute_pair(
      e.attribute_ref('Enrollments, 'CourseId),
      e.attribute_ref('Courses, 'CourseId)),
  ]),
  'GrainSafe,                    # RelationGrain: 'GrainSafe 或 'FanOut
  'Inner);                       # qb.JoinKind: 'Inner 或 'Left
```

- mapping 的每个条件都是 `left 属性 == right 属性` 的结构化等值对；不使用预渲染
  SQL 片段，join 条件由 eDSL 生成 `qb.EquiCondition`。
- `'GrainSafe` 表示该方向不扩张行数；`'FanOut` 表示会扩张 grain。
- 关系的**目录顺序**参与路径选择的字典序 tie-break，先声明的关系在同长度时
  优先。

## 4. 建模指标（多 grain）

指标登记在各自所属的 grain 实体上；同一个知识可以同时登记多个 grain 的获准
指标（例如事实表与另一张明细表）：

```telora
let measures = [
  e.measure('Revenue, 'Enrollments, 'AmountPaid,
    'Aggregate({function: 'Sum, distinct: 'False}), "revenue"),
  e.measure('AwardCount, 'CertificateAwards, 'CertificateId,
    'Aggregate({function: 'Count, distinct: 'False}), "award_count"),
];
```

- `e.measure(id, grain_entity, attribute, kind, alias)`：`kind` 可以是
  `'Plain`（直接投影列）或 `'Aggregate({function, distinct})`（QueryBuilder
  聚合函数 `'Count 'Sum 'Avg 'Min 'Max`）。
- **一次请求的 measure 决定该 Plan 的 base grain**：所有请求 measure 必须位于
  同一实体；不兼容时 eDSL 明确失败。维度与所需实体从该 base grain 走安全关系
  路径。
- 指标/维度最后的 String 是输出别名（SQL 标识符），进入投影 alias。

## 5. 建模维度（直接属性或受控计算表达式）

维度可以引用一个直接属性，也可以使用基于属性的受控标量表达式：

```telora
let dimensions = [
  e.dimension('CourseCategory, 'Courses,
    e.dimension_attribute('Courses, 'Category), "course_category"),
  # 受控计算维度：substr(en.created_at, 1, 7)（按月截断）
  e.dimension('EnrollmentMonth, 'Enrollments,
    e.dimension_computed(e.computed_expr('Substr, [
      e.scalar_arg_attribute('Enrollments, 'CreatedAt),
      e.scalar_arg_literal('Int(1)),
      e.scalar_arg_literal('Int(7)),
    ])),
    "enrollment_month"),
];
```

- `e.dimension_attribute(entity, attribute)`：直接属性维度。
- `e.dimension_computed(e.computed_expr(function, args))`：受控标量表达式。
  函数必须是 QueryBuilder 的 `qb.ScalarFunction`（`'Substr` 等）；实参是
  `e.scalar_arg_attribute(entity, attribute)` 或
  `e.scalar_arg_literal(qb.Val)`。属性实参必须属于该维度实体。
- 字面量在 lowering 时成为 `qb.Val`/binding（参数化 SQL），不保存预渲染 SQL，
  也不用 String 反查语义对象。
- 计算维度使用的标量函数必须进入 `knowledge.plan_profile.scalar_functions`
  约束，否则最终 `qb.validate` 拒绝 Plan。

## 6. 建模能力目录与授权

```telora
let measure_capabilities = [
  e.capability('Revenue, fn(subject, input) { input == 'All }),
  e.capability('AwardCount, fn(subject, input) { input == 'All }),
];

let dimension_capabilities = [
  e.capability('CourseCategory, fn(subject, input) { input == 'All }),
  e.capability('EnrollmentMonth, fn(subject, input) { input == 'All }),
];
```

每个能力 = `(id, 授权谓词)`。授权谓词接收请求的 `subject` 与 `input`，返回
Bool。请求中的每个 measure/dimension 都必须命中一个能力且通过授权，否则
lowering 失败。

## 7. 建模筛选能力（有类型维度筛选）

筛选请求引用 `DimensionId` + 领域输入 + 标准比较操作（`'Eq`/`'Ge`/`'Le`）。
企业知识用 `e.filter_capability(id, authorized, to_val)` 提供授权与值转换：

```telora
let filter_capabilities = [
  e.filter_capability('CourseCategory, fn(subject, input) {
    input == 'Category("Analytics") || input == 'Category("Design")
  }, fn(input) {
    match input {
      'Category(s) => 'String(s),
      _ => fail!("invalid filter value for CourseCategory", input),
    }
  }),
  e.filter_capability('EnrollmentMonth, fn(subject, input) { 'True }, fn(input) {
    match input {
      'Month(s) => 'String(s),
      _ => fail!("invalid filter value for EnrollmentMonth", input),
    }
  }),
];
```

- `authorized(subject, input)`：该筛选是否获准（与 measure/dimension 授权并列）。
- `to_val(input)`：把领域输入转换为 `qb.Val`（`'String(s)`/`'Int(i)`/...）。
  查询方不能提交 `qb.Expr`、列、表、alias、SQL 或 mapping；值必须由企业知识
  转换，最终成为 `qb.Literal`/binding。
- 转换失败（例如维度收到错误变体的输入）以 `fail!` 原子失败。

## 8. 声明接受的 PlanProfile

```telora
let plan_profile = qb.with_scalar_functions(
  qb.with_aggregate_functions(
    qb.with_join_kinds(qb.standard_profile, ['Inner]),
    ['Sum, 'Count]),
  ['Substr, 'Eq, 'Ge, 'Le, 'And]);
```

`plan_profile` 是企业知识接受的 QueryBuilder 能力声明。eDSL 组装完 Plan 后调用
`qb.validate(plan, plan_profile)`。**实际使用的能力必须显式进入 profile**：
筛选使用 `'Eq`/`'Ge`/`'Le` 与组合用的 `'And`，必须列入 `scalar_functions`；
`standard_profile` 派生的 profile 已开启 `allow_filter`/`allow_order_by`/
`allow_limit`。profile 越界会阻止 Plan 发布。

## 9. 组装 EnterpriseKnowledge

```telora
def knowledge: e.Knowledge(MeasureId, DimensionId, EntityId, AttributeId, RelationId, Subject, MeasureInput, DimensionInput, FilterInput) = {
  revision: "course-analytics-v1",
  entities: [enrollments, courses],
  relations: [enroll_course],
  measures: measures,
  dimensions: dimensions,
  measure_capabilities: measure_capabilities,
  dimension_capabilities: dimension_capabilities,
  filter_capabilities: filter_capabilities,
  plan_profile: plan_profile,
};
```

`revision: String` 是企业持有的版本标识，会原样进入 `Plan.revision`（不参与
验证或具体化）。`Knowledge` 不设单一 base 实体：base grain 由每次请求的
measure 决定。

## 10. 提交查询请求

```telora
def request: e.QueryRequest(MeasureId, DimensionId, Subject, MeasureInput, DimensionInput, FilterInput) = {
  measures: [
    {id: 'Revenue, subject: "analyst@example.com", input: 'All},
  ],
  dimensions: [
    {id: 'CourseCategory, subject: "analyst@example.com", input: 'All},
  ],
  filters: [
    {id: 'EnrollmentMonth, subject: "analyst@example.com", input: 'Month("2026-04"), op: 'Ge},
    {id: 'EnrollmentMonth, subject: "analyst@example.com", input: 'Month("2026-06"), op: 'Le},
    {id: 'CourseCategory, subject: "analyst@example.com", input: 'Category("Analytics"), op: 'Eq},
  ],
  ordering: [
    {target: 'Measure('Revenue), direction: 'Desc},
    {target: 'Dimension('CourseCategory), direction: 'Asc},
  ],
  limit: 'Some(10),
};
```

请求只表达"要什么"：`id` + `subject` + `input`（筛选另加 `op`）。不携带领域
表达式、mapping、Plan 节点或 SQL。各数组保持原始顺序；原始 subject 会进入
相关诊断。请求必须至少包含一个 measure（它决定 base grain）。

- `filters`：按请求顺序用标准 `And` 组合。筛选维度可以不在
  projections/group_by 中，但仍须执行能力授权，并为其实体选择 grain-safe 路径、
  合并必要 join。同一维度可以出现多个筛选（例如月份 `Ge 2026-04` 与
  `Le 2026-06`）。
- `ordering`：只引用**已经请求**的 measure/dimension。支持"指标降序 + 维度
  升序"的稳定 tie-break；引用未请求的目标会诊断失败。
- `limit`：可选正整数（Top N）；非正数会诊断失败。

## 11. 得到 Plan 并端到端展示 Query

```telora
let creator = e.make_query_creator(knowledge);
let plan = creator(request);                       # qb.Plan
let query = qb.transform_sqlite(plan, knowledge.plan_profile);   # qb.Query
```

`make_query_creator(knowledge)` 返回 `Fn(QueryRequest) -> Plan`。lowering 依次：

1. 独立解析并授权每项能力（measure/dimension/filter）；
2. 解析 base grain：请求 measure 必须全部位于同一实体（不兼容时明确失败）；
   验证指标 grain 兼容性（维度路径必须安全）；
3. 推导维度所需实体并选择安全关系（最短边数优先，同长度目录索引序列字典序
   最小；多目标按请求顺序合并，共享边只保留首次出现；最大深度 8）。筛选与
   排序涉及的维度同样走该路径选择并合并必要 join；
4. 组装覆盖所有请求且没有额外请求的标准 Plan：受控计算维度 lower 为
   `qb.ScalarCall`；筛选按请求顺序 `And` 组合、值成为 `qb.Literal`/binding；
   排序只从已请求目标构造 `qb.OrderItem`；limit 写入 `Plan.limit`；
5. 验证 Plan 只使用 `knowledge.plan_profile` 接受的能力；
6. 成功发布 Plan，任何失败（授权、grain、路径、筛选值、排序目标、limit、
   profile）通过 `fail!` 记录诊断且不发布部分 Plan。

`transform_sqlite` 不是本 eDSL 的 API，只是成功示例展示端到端 Query 的方式。

## 12. 完整可运行示例

`ontology/src/bin/main.telora` 是一个完整的虚构示例（在线课程平台分析，含
多 grain 指标、受控计算维度、范围 + 等值筛选、指标降序 + 维度升序与 Top N），
`ontology/src/bin/verify.telora` 演示请求覆盖、关系选择、多 grain base、计算
维度、筛选、排序与 profile 覆盖的验证，`ontology/src/bin/invalid.telora`
演示非法请求（不兼容 grain、非正 limit、未请求排序目标、缺失/未授权筛选能力、
非法筛选值）原子失败。运行：

```text
./bin/telora run main -C ontology
./bin/telora run verify -C ontology
./bin/telora run invalid -C ontology --best-effort
./bin/telora check @test/ontology.telora -C ontology
```
