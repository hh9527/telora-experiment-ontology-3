# EnterpriseKnowledge eDSL 公共契约

本文件描述 `ontology/src/lib.telora` 的稳定公共类型、输入约束与 lowering 保证。
它是领域无关的基础 eDSL：只处理企业知识的建模与 `EnterpriseKnowledge -> Request
-> Plan` 的 lowering，不包含企业领域事实。`Plan`、`PlanProfile`、`Query` 与标准
算子由 QueryBuilder package 所有，本 eDSL 不定义替代 Plan，不负责 `Plan -> Query`。

## 公共入口

`ontology/src/lib.telora`。外部 crate 使用 manifest 依赖名导入：
`import "ontology/lib.telora" as e;`（依赖名以 manifest 为准）。

## 稳定公共类型

```telora
type SourceRef = struct { table: String, alias: String };

type AttributeRef(EntityId, AttributeId) = struct { entity: EntityId, attribute: AttributeId };
type AttributePair(EntityId, AttributeId) = struct { left: AttributeRef(...), right: AttributeRef(...) };
type RelationMapping(EntityId, AttributeId) = struct { conditions: Array(AttributePair(...)) };
type RelationGrain = enum { 'GrainSafe, 'FanOut };

type Attribute(EntityId, AttributeId) = struct { id: AttributeId, entity: EntityId, column: String };
type Entity(EntityId, AttributeId) = struct { id: EntityId, source: SourceRef, attributes: Array(Attribute(...)) };
type Relation(EntityId, AttributeId, RelationId) = struct {
  id: RelationId, left: EntityId, right: EntityId,
  mapping: RelationMapping(...), grain: RelationGrain, join_kind: qb.JoinKind,
};

type MeasureAgg = struct { function: qb.AggFunction, distinct: Bool };
type MeasureKind = enum { 'Plain, 'Aggregate(MeasureAgg) };
type Measure(MeasureId, EntityId, AttributeId) = struct { id, entity, attribute, kind, alias: String };

type ScalarArg(EntityId, AttributeId) = enum {
  'Attribute(AttributeRef(...)),
  'Literal(qb.Val),
};
type ComputedExpr(EntityId, AttributeId) = struct { function: qb.ScalarFunction, args: Array(ScalarArg(...)) };
type DimensionExpr(EntityId, AttributeId) = enum {
  'Attribute(AttributeRef(...)),
  'Computed(ComputedExpr(...)),
};
type Dimension(DimensionId, EntityId, AttributeId) = struct { id, entity, expr: DimensionExpr(...), alias: String };

type Capability(Id, Subject, Input) = struct { id: Id, authorized: Fn(Subject, Input) -> Bool };

type FilterOp = enum { 'Eq, 'Ge, 'Le };
type FilterRequest(DimensionId, Subject, FilterInput) = struct {
  id: DimensionId, subject: Subject, input: FilterInput, op: FilterOp,
};
type FilterCapability(DimensionId, Subject, FilterInput) = struct {
  id: DimensionId, authorized: Fn(Subject, FilterInput) -> Bool, to_val: Fn(FilterInput) -> qb.Val,
};

type OrderTarget(MeasureId, DimensionId) = enum { 'Measure(MeasureId), 'Dimension(DimensionId) };
type OrderRequest(MeasureId, DimensionId) = struct { target: OrderTarget(...), direction: qb.OrderDirection };

type Knowledge(MeasureId, DimensionId, EntityId, AttributeId, RelationId, Subject, MeasureInput, DimensionInput, FilterInput) = struct {
  revision: String,
  entities: Array(Entity(EntityId, AttributeId)),
  relations: Array(Relation(EntityId, AttributeId, RelationId)),
  measures: Array(Measure(MeasureId, EntityId, AttributeId)),
  dimensions: Array(Dimension(DimensionId, EntityId, AttributeId)),
  measure_capabilities: Array(Capability(MeasureId, Subject, MeasureInput)),
  dimension_capabilities: Array(Capability(DimensionId, Subject, DimensionInput)),
  filter_capabilities: Array(FilterCapability(DimensionId, Subject, FilterInput)),
  plan_profile: qb.PlanProfile,
};

type Request(Id, Subject, Input) = struct { id: Id, subject: Subject, input: Input };
type QueryRequest(MeasureId, DimensionId, Subject, MeasureInput, DimensionInput, FilterInput) = struct {
  measures: Array(Request(MeasureId, Subject, MeasureInput)),
  dimensions: Array(Request(DimensionId, Subject, DimensionInput)),
  filters: Array(FilterRequest(DimensionId, Subject, FilterInput)),
  ordering: Array(OrderRequest(MeasureId, DimensionId)),
  limit: Option(Int),
};
```

全部 ID（`MeasureId`、`DimensionId`、`EntityId`、`AttributeId`、`RelationId`、
`Subject`、`MeasureInput`、`DimensionInput`、`FilterInput`）是知识作者提供的类型
参数。公共 API 不使用 `Any`、`Dyn` 或 native 声明；目录查找全部基于类型级身份，
禁止 String 反查领域数据。

## 公共函数与常量

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `source_ref` | `Fn(String, String) -> SourceRef` | 物理表名 + 别名 |
| `attribute` | `for(E, A) Fn(A, E, String) -> Attribute(E, A)` | 属性 id + 实体 + 物理列 |
| `entity` | `for(E, A) Fn(E, SourceRef, Array(Attribute(E, A))) -> Entity(E, A)` | 实体 |
| `attribute_ref` | `for(E, A) Fn(E, A) -> AttributeRef(E, A)` | 属性引用 |
| `attribute_pair` | `for(E, A) Fn(AttributeRef, AttributeRef) -> AttributePair` | 等值条件对 |
| `relation_mapping` | `for(E, A) Fn(Array(AttributePair)) -> RelationMapping` | 结构化 mapping |
| `relation` | `for(E, A, R) Fn(R, E, E, RelationMapping, RelationGrain, qb.JoinKind) -> Relation` | 关系 |
| `measure` | `for(M, E, A) Fn(M, E, A, MeasureKind, String) -> Measure` | 指标（登记在 grain 实体上） |
| `scalar_arg_attribute` | `for(E, A) Fn(E, A) -> ScalarArg(E, A)` | 计算表达式的属性实参 |
| `scalar_arg_literal` | `for(E, A) Fn(qb.Val) -> ScalarArg(E, A)` | 计算表达式的字面量实参 |
| `computed_expr` | `for(E, A) Fn(qb.ScalarFunction, Array(ScalarArg(E, A))) -> ComputedExpr(E, A)` | 受控标量表达式 |
| `dimension_attribute` | `for(E, A) Fn(E, A) -> DimensionExpr(E, A)` | 直接属性维度表达式 |
| `dimension_computed` | `for(E, A) Fn(ComputedExpr(E, A)) -> DimensionExpr(E, A)` | 计算维度表达式 |
| `dimension` | `for(D, E, A) Fn(D, E, DimensionExpr(E, A), String) -> Dimension` | 维度 |
| `capability` | `for(I, S, In) Fn(I, Fn(S, In) -> Bool) -> Capability` | 能力 + 授权谓词 |
| `filter_capability` | `for(D, S, FI) Fn(D, Fn(S, FI) -> Bool, Fn(FI) -> qb.Val) -> FilterCapability` | 筛选能力（授权 + 值转换） |
| `make_query_creator` | `for(M, D, E, A, R, S, MI, DI, FI) Fn(Knowledge) -> Fn(QueryRequest) -> qb.Plan` | lowering 入口 |
| `max_path_depth` | `Int`（= 8） | 路径最大深度 |

## EnterpriseKnowledge 输入约束

1. **多 grain 指标**：`Knowledge` 没有单一 base 实体；不同 grain 的获准指标可以
   同时登记（每个 `measure.entity` 是它的 grain 实体）。
2. **别名唯一**：所有实体的 `source.alias` 必须互不相同（最终 SQL 的 join 别名
   冲突会让 Plan 结构校验失败）。
3. **mapping 引用有效**：关系 mapping 中的属性必须存在于对应实体的属性目录；
   关系的 left/right 实体必须存在于 `entities`。计算维度表达式的属性实参必须
   属于该维度实体。
4. **标识符合法**：`source.table`、`source.alias`、属性 `column`、指标/维度
   `alias` 必须是合法 SQL 标识符（`[A-Za-z_][A-Za-z0-9_]*`）；非法时由
   QueryBuilder 的 `ident` 校验以 `fail!` 失败。
5. **关系方向**：关系是有向边（`left -> right`），`grain` 分类针对该方向。
6. **profile 收窄**：`plan_profile` 只收窄 QueryBuilder 标准算子能力；计算维度
   使用的标量函数必须在 `plan_profile.scalar_functions` 内，指标聚合必须在
   `aggregate_functions` 内，实际使用的 `filter`/`order_by`/`limit` 及
   `Eq/Ge/Le/And` 必须被 profile 接受。最终 Plan 必须通过
   `qb.validate(plan, plan_profile)`。
7. **筛选能力**：每个可筛选的维度都应提供 `filter_capabilities` 条目；
   `authorized(subject, input)` 决定该筛选是否获准，`to_val(input)` 把领域输入
   转换为 `qb.Val`（转换失败即原子失败）。企业知识负责把领域输入转换为
   `qb.Val`，查询方不能提交 `qb.Expr`、列、表、alias、SQL 或 mapping。

## QueryRequest 输入约束

请求只表达"要什么"：`id` + `subject` + `input`（筛选另加 `op`），不得携带领域
表达式、mapping、Plan 节点或 SQL。`measures`、`dimensions`、`filters` 与
`ordering` 保持原始顺序；原始 `subject` 进入相关诊断。请求必须至少包含一个
measure（base grain 由 measure 决定）。

- `filters`：每项引用一个 `DimensionId` + `FilterOp`（`'Eq`/`'Ge`/`'Le`）。
  多个筛选按请求顺序用标准 `And` 组合；值经 `FilterCapability.to_val` 成为
  `qb.Literal` 并最终进入 bindings。筛选维度可以不在 projections/group_by 中，
  但必须执行能力授权，并为其实体选择 grain-safe 路径、合并必要 join。同一维度
  可以出现多个筛选（例如月份 `Ge 2026-04` 与 `Le 2026-06`），顺序必须稳定。
- `ordering`：每项引用**已经请求**的 measure 或 dimension（`'Measure(id)` /
  `'Dimension(id)`）+ `Asc`/`Desc`。eDSL 只能从已解析的 measure/dimension 规范
  表达式构造 `qb.OrderItem`；引用未请求的排序目标必须诊断并原子失败。
- `limit`：可选正整数；非正数必须诊断失败。成功时写入 `Plan.limit`，由
  QueryBuilder 生成 `LIMIT ?` 并把 N 放入 bindings。

## lowering 保证

`make_query_creator(knowledge)(request)` 按以下顺序执行并发布 `qb.Plan`：

1. **独立解析并授权每项能力**：每个 measure/dimension/filter 请求必须在对应能力
   目录中找到 id 且 `authorized(subject, input)` 返回 `'True`；缺失或未授权即
   失败。
2. **解析 base grain 并验证指标兼容性**：请求的 measure 必须全部位于同一实体
   （base grain），不兼容时明确失败；维度路径必须使用 `'GrainSafe` 关系。仅能
   通过 `'FanOut` 关系到达的维度（fan-out-only）视为 grain 冲突；不可达视为
   缺失。
3. **推导维度所需实体并选择安全关系**：从 base grain 实体 BFS，安全路径最短
   边数优先，同长度按目录索引序列字典序最小；遍历对有向环稳定，最大深度 8
   （恰好 8 条边可接受）。完整可达性使用 safe 与 fan-out 的并集，用于把目标
   分类为 safe、fan-out-only 或 missing。筛选维度（即使未投影）与排序维度同样
   走该路径选择并合并必要 join。
4. **组装覆盖所有请求且没有额外请求的标准 Plan**：每个成功请求恰有一个投影
   （measures 投影聚合/列表达式，dimensions 投影列/受控标量表达式），grouping
   与维度一致；非 base 维度的路径以 join 覆盖；多目标按请求顺序合并，共享边
   只保留首次出现。受控计算维度 lower 为 `qb.ScalarCall`，字面量成为
   `qb.Val`/binding；筛选条件按请求顺序用 `qb.scalar_call('And, ...)` 组合，
   值成为 `qb.Literal`/binding；排序只从已请求的 measure/dimension 规范表达式
   构造 `qb.OrderItem`（例如指标降序 + 维度升序的稳定 tie-break）；limit 为正
   整数时写入 `Plan.limit`。不使用预渲染 SQL 或 String 反查。Plan 保留
   `knowledge.revision`（String，原样进入 `qb.Plan.revision`，不参与验证/具体
   化）、base 数据源、有序投影、grouping 与按规范顺序选择的关系 mapping。
5. **验证 Plan 只使用 `knowledge.plan_profile` 接受的能力**：`qb.validate`
   失败时以 `fail!` 记录诊断。
6. **原子发布**：成功时发布完整 Plan；授权、缺失能力、grain 冲突、筛选值不
   合法、排序目标不合法、limit 不合法或 profile 越界任一失败都不发布部分
   Plan。公共 API 不返回 Rejection、诊断数组或逐请求 Evidence。

## 语义保证

1. **确定性**：同一个 knowledge + request 逐字节产生相同的 Plan（join 顺序、
   投影顺序、grouping、filter 结构、ordering 顺序与 limit 都确定）；相同请求
   产生相同 SQL 和 bindings，bindings 顺序与 SQL 中 `?` 顺序一致。
2. **封闭类型**：公共类型全部是封闭的具名 enum/struct family；不使用
   `Any`/`Dyn`/native。
3. **无 String 反查**：目录查找只通过类型级 ID；String 只用于物理表名/列名/
   输出别名，不用于回查领域数据。
4. **受控表达式**：计算维度、筛选与排序只使用 `qb.ScalarFunction` + 属性/
   `qb.Val` 字面量构造，不保存预渲染 SQL。筛选值必须成为 `qb.Literal`/binding。
5. **Plan/Query 边界**：eDSL 只产出 `qb.Plan`；`qb.transform_sqlite` 不是
   eDSL API，成功示例可调用它以展示端到端 Query。
6. **失败即不发布**：授权、grain、路径、筛选值、排序目标、limit、profile
   任何一项失败都会阻止 Plan 发布。
