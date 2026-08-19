# EnterpriseKnowledge eDSL 公共契约

公共模块：`@src/ontology.telora`（crate `ontology`）。企业依赖方用逻辑模块 ID
`ontology/ontology.telora` 导入（对应 `<ontology>/src/ontology.telora`）；
`@src/...` 只用于 ontology crate 内部模块。本 eDSL 让企业表达结构化
`EnterpriseKnowledge`，并提供稳定的 lowering：

```text
make_query_creator: EnterpriseKnowledge -> Fn(Request) -> Plan
```

`Plan`、标准算子和 `PlanProfile` 由 QueryBuilder 所有（公共交付：
`query-builder/query-builder.telora`）。eDSL 不定义替代 Plan，不负责
`Plan -> Query`，也不包含任何企业领域事实。

## 公共 API

```telora
type Request(Id, Subject, Input) = struct {
    id: Id,
    subject: Subject,
    input: Input,
};

type QueryRequest(MeasureId, DimensionId, Subject, MeasureInput, DimensionInput) = struct {
    measures: Array(Request(MeasureId, Subject, MeasureInput)),
    dimensions: Array(Request(DimensionId, Subject, DimensionInput)),
};

type RelationshipKind = enum { 'Safe, 'FanOut };

# --- structured physical expressions (closed, typed, recursive) ---
type AttributeRef = struct { index: Int };
type ExprCall = struct { kind: ScalarFn, args: Array(ExprNode) };
type ExprNode = enum {
    'Attribute(AttributeRef),
    'Bound(Val),
    'ScalarCall(ExprCall),
};

attribute_node : for(E, A) Fn(Array(AttributeDef(E, A)), A) -> ExprNode
bound_node     : Fn(Val) -> ExprNode
scalar_node    : Fn(ScalarFn, Array(ExprNode)) -> ExprNode

# --- enterprise knowledge ---
type GrainBase(GrainId, EntityId) = struct { grain: GrainId, base: EntityId };
type EntityDef(EntityId) = struct { id: EntityId, table: String };
type AttributeDef(EntityId, AttributeId) = struct { id: AttributeId, entity: EntityId, column: String };

type MetricDef(EntityId, AttributeId, MeasureId, GrainId, Subject) = struct {
    id: MeasureId, entity: EntityId, requires: Array(EntityId),
    aggregate: AggregateFn, expr: ExprNode,
    grain: GrainId, alias: String, subjects: Array(Subject),
};

type DimensionDef(EntityId, AttributeId, DimensionId, GrainId, Subject) = struct {
    id: DimensionId, entity: EntityId, requires: Array(EntityId),
    levels: Array(ExprNode),
    grain: GrainId, alias: String, subjects: Array(Subject),
};

type RelationshipMapping(AttributeId) = struct {
    from_attribute: AttributeId,
    to_attribute: AttributeId,
};

type RelationshipDef(EntityId, AttributeId, RelationshipId) = struct {
    id: RelationshipId, from: EntityId, to: EntityId,
    kind: RelationshipKind, mapping: RelationshipMapping(AttributeId),
};

type Vocabulary(EntityId, AttributeId, RelationshipId) = struct {
    entities: Array(EntityDef(EntityId)),
    attributes: Array(AttributeDef(EntityId, AttributeId)),
    relationships: Array(RelationshipDef(EntityId, AttributeId, RelationshipId)),
};

type EnterpriseKnowledge(EntityId, AttributeId, MeasureId, DimensionId, RelationshipId, GrainId, Subject) = struct {
    revision: String,
    grain_bases: Array(GrainBase(GrainId, EntityId)),
    vocabulary: Vocabulary(EntityId, AttributeId, RelationshipId),
    metrics: Array(MetricDef(EntityId, AttributeId, MeasureId, GrainId, Subject)),
    dimensions: Array(DimensionDef(EntityId, AttributeId, DimensionId, GrainId, Subject)),
    plan_profile: PlanProfile,
};

make_query_creator:
    for(E, A, M, D, R, G, S)
    Fn(EnterpriseKnowledge(E, A, M, D, R, G, S))
    -> Fn(QueryRequest(M, D, S, G, G))
    -> Plan
```

能力身份（实体、属性、指标、维度、关系、grain）必须是封闭 enum 类型；请求的
`MeasureInput`/`DimensionInput` 统一为 grain 类型参数 `G`。公共 API 不使用
`Any`、`Dyn` 或 native 声明，也不以 String 充当领域身份。

### 关于 ExprNode 的属性引用

Telora 不支持参数化递归 family（`docs/LANG-TUTORIAL.md` 的稳定边界），因此递归
表达式树 `ExprNode` 是封闭的具体递归类型，属性叶子以 `AttributeRef`（属性目录
索引）表示。索引由类型化构造辅助 `attribute_node(attrs, id)` 从企业 `AttributeId`
解析，并在知识构造时递归校验（索引在范围内、属性属于能力实体）。`Bound(Val)`
与 `ScalarCall(ScalarFn, ...)` 是 QueryBuilder 的精确类型，不涉及 `Any`/`Dyn`/
String 语义身份。

## 输入保证（知识构造期校验）

`make_query_creator(knowledge)` 构造时会校验知识，失败立即 `fail!`：

- 每个 `GrainBase`：base 实体存在；grain 不重复（重复 base 定义失败）。
- 每个 metric：实体存在；`requires` 中的每个实体存在；`expr` 递归有效（每个
  `AttributeRef` 在范围内且属性属于该实体）；投影 alias 是合法 SQL 标识符。
- 每个 dimension：实体存在；`requires` 中的每个实体存在；至少一个层级；每个
  层级表达式递归有效；投影 alias 合法且全局唯一。
- 每个 relationship：from/to 实体存在；`from_attribute` 属于 `from` 实体、
  `to_attribute` 属于 `to` 实体。
- 表名/列名/alias 匹配 SQLite bare identifier 词法 `^[A-Za-z_][A-Za-z0-9_]*$`。

## Lowering 保证

对每个请求，`make_query_creator(knowledge)(request)` 依次：

1. 独立解析并授权每项能力（请求 id 必须命中目录；subject 必须在能力的
   `subjects` 列表中）。
2. 校验 grain 兼容性：所有请求共享一个 grain；每个请求的 grain 必须等于其
   能力定义的 grain。
3. 按请求的共同 grain 解析基础数据源（`GrainBase`）；缺失该 grain 的 base 失败。
4. 为每个 measure/dimension 的**能力自身实体**与每个**required entity** 分别
   选择安全路径；required entity 等于 base/主实体时产生空路径并自然去重。
5. 按请求顺序合并路径（measure 路径在前、dimension 路径在后；能力实体路径在
   该能力的 required 路径之前，required 按 `requires` 声明顺序；共享边只保留
   首次出现），组装标准 `Plan`。
6. 先通过 `validate_plan` 结构验证，再通过 `within_profile(plan,
   plan_profile)` profile 验证。
7. 成功时发布 `Plan`；任一步失败时 `fail!`，不发布部分 Plan。

### 表达式 lowering

- `'Attribute(ref)` → `'Column({alias, column})`：解析实体 alias 与属性列。
- `'Bound(v)` → `'Bound(v)`：有类型绑定值，进入 Query bindings。
- `'ScalarCall(call)` → `'ScalarCall({kind, args})`：递归 lowering 参数。
- 指标投影 = `'Aggregate({kind, arg: lower(expr)})`；维度投影与 grouping 使用
  `lower(level)`。派生维度（如 `substr(col, 1, 7)`）同时进入 projection 与
  grouping，绑定常量按 SQL 占位符顺序进入 bindings。

### 路径与关系

- 关系目录区分 `'Safe`（grain-safe）与 `'FanOut`（扩张 grain）。
- 安全路径选择：最短边数优先；同长度按关系目录索引序列字典序最小。
- 遍历对 directed cycles 稳定，最大深度为 8（恰好 8 条边可接受）。
- 完整可达性使用 safe 与 fan-out 的并集；目标分类为 safe、fan-out-only 或
  missing。fan-out-only、missing、授权失败、grain 冲突（含无 base）、结构或
  profile 越界都会阻止 Plan 发布。深度边界仍有未访问后继时报告 truncation 并
  阻止发布。
- 关系 mapping 必须是列等值；每条 safe 关系降级为一个 `'Inner` join，join 的
  `left` 引用已定义 alias（`e{pos}`），`right` 引用本 join 引入的 alias
  （`e{pos+1}`）。

### Plan 形状

- `revision` = 知识的 String revision 标签，由 Plan 原样保留，不参与判定、不
  出现在 SQL/Query 中；`source` = 所选 grain 基础实体的表，alias `e0`。
- `projection`：measure 投影（请求顺序）在前，dimension 投影（请求顺序）在后；
  每个请求恰有一个投影项，alias 来自能力定义。
- `grouping`：所有 dimension 层级的表达式（按请求顺序、按层级顺序，去重）。
- `joins`：合并后的安全路径边（规范顺序）。
- `filter`/`ordering` 为空，`limit` 为 `'None`（本 eDSL 的请求不携带这些）。

### 失败语义

- 失败使用 `fail!`，消息前缀见 DSL-TUTORIAL.md 第 5 节；请求级失败保留原始
  subject 作为诊断 subject。
- 公共 API 不返回 Rejection、诊断数组或逐请求 Evidence。

## 非目标 / 边界

- 不支持非等值关系 mapping（QueryBuilder join 只接受列等值）。
- 不支持请求级 filter / ordering / limit；不支持 OR/NOT/CASE/HAVING/子查询。
- `Val` 不支持 Bytes；公共边界不含 Bytes。
- 表达式递归受 Telora 参数化递归限制约束：`ExprNode` 是具体递归类型，属性叶子
  使用目录索引（见上文 ExprNode 说明）。
- QueryBuilder 不为 `Plan`/`Query` 提供 JSON/schema 边界；需要稳定 JSON 表示时
  由使用方 crate 自行建立。
- 企业领域事实（实体表数据、指标公式、映射业务含义）不属于本 eDSL。
