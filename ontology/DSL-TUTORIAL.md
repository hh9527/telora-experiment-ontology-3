# EnterpriseKnowledge eDSL 教程

本教程面向企业作者（A3）。你可以用本 eDSL 表达企业知识，并让同一套代码把
「知识 + 请求」降级为 QueryBuilder 的标准 `Plan`（并进一步得到 SQLite `Query`）。
你不需要阅读 `ontology/src/ontology.telora` 的实现源码。

企业依赖方（另一个 crate）用 `ontology/ontology.telora` 导入本 eDSL：

```telora
import "ontology/ontology.telora" {
    EnterpriseKnowledge, EntityDef, AttributeDef, MetricDef, DimensionDef,
    RelationshipDef, RelationshipMapping, RelationshipKind, Vocabulary,
    GrainBase, QueryRequest, ExprNode, attribute_node, bound_node, scalar_node,
    make_query_creator,
};
```

（`@src/...` 形式只用于 ontology crate 内部模块；企业 crate 使用上述
package 逻辑模块 ID，对应 `<ontology>/src/ontology.telora`。）

## 1. 总体流程

```text
EnterpriseKnowledge -> make_query_creator(knowledge) -> Fn(Request) -> Plan
Plan -> transform_sqlite_with_profile(plan, profile) -> Query
```

- `EnterpriseKnowledge`：你建立的企业知识（封闭 vocabulary、能力目录、物理映射、
  关系目录、接受的 `PlanProfile`）。
- `Request`：查询方提交的有类型意图，只表达「要什么」。
- `Plan` / `Query`：由 QueryBuilder 公共交付定义；eDSL 只负责组装和验证。

## 2. 建立企业知识

企业知识由 `EnterpriseKnowledge(EntityId, AttributeId, MeasureId, DimensionId,
RelationshipId, GrainId, Subject)` 记录构成。七个类型参数由你的企业词汇决定，
全部使用封闭 enum（不能使用 String 或 `Any` 充当身份）。`GrainId` 是你的 grain
词汇；`Subject` 是你的授权主体词汇。

### 2.1 实体与属性

```telora
type EntityId = enum { 'Order, 'PackageItem, 'Customer, 'Product, 'Campaign };
type AttributeId = enum { 'OrderId, 'OrderAmount, 'OrderCreatedAt, ... };

# EntityDef: id + 物理表名
{id: 'Order, table: "orders"}
# AttributeDef: id + 所属实体 + 物理列名
{id: 'OrderAmount, entity: 'Order, column: "amount"}
```

表名、列名、投影 alias 必须匹配 SQL 标识符词法 `^[A-Za-z_][A-Za-z0-9_]*$`。

### 2.2 结构化物理表达式

`ExprNode` 是封闭、类型化、递归的物理表达式：

```text
ExprNode = Attribute(catalog ref) | Bound(Val) | ScalarCall(ScalarFn, Array(ExprNode))
```

用三个类型化构造辅助函数建立表达式（`Val` 与 `ScalarFn` 来自 QueryBuilder 公共
契约）：

```telora
let attrs = [ {id: 'OrderCreatedAt, entity: 'Order, column: "created_at"}, ... ];

# order_month = substr(orders.created_at, 1, 7)
let order_month_expr: ExprNode = scalar_node('Substr, [
    attribute_node(attrs, 'OrderCreatedAt),   # 类型化属性引用
    bound_node('Int(1)),                      # 绑定常量
    bound_node('Int(7)),
]);
```

`attribute_node(attrs, id)` 从属性目录解析出该属性；`bound_node` 包裹有类型绑定
值；`scalar_node(kind, args)` 递归组合标量调用。表达式中的属性在知识构造时会被
递归校验：必须存在且属于该能力声明的实体。

### 2.3 指标（measure capability）

```telora
type MeasureId = enum { 'Revenue, 'ItemCount, 'DeliveredPackages, 'UnitsShipped };
type Subject = enum { 'Analyst, 'Manager, 'Auditor };

# MetricDef: 实体 + required entities + 聚合函数 + 表达式 + grain + 投影 alias + 授权
{id: 'Revenue, entity: 'Order, requires: [],
 aggregate: 'Sum, expr: attribute_node(attrs, 'OrderAmount),
 grain: 'OrderGrain, alias: "revenue", subjects: ['Analyst, 'Manager]}
{id: 'DeliveredPackages, entity: 'Package, requires: ['Order],
 aggregate: 'Count, expr: attribute_node(attrs, 'PackageId),
 grain: 'PackageGrain, alias: "delivered_packages", subjects: ['Analyst, 'Manager]}
{id: 'UnitsShipped, entity: 'PackageItem, requires: ['Package, 'Order],
 aggregate: 'Sum, expr: attribute_node(attrs, 'PackageItemAmount),
 grain: 'ItemGrain, alias: "units_shipped", subjects: ['Analyst]}
```

`requires: Array(EntityId)` 是能力所要求实体的结构化声明（精确类型，不是注释）：
lowering 会为能力自身实体以及每个 required entity 分别选择安全路径，再按稳定
顺序合并。required entity 等于 base/主实体时自然去重；fan-out-only、missing、
truncated 仍然原子失败。`aggregate` 来自 QueryBuilder 的 `AggregateFn`：
`'Count 'CountDistinct 'Sum 'Avg 'Min 'Max`。

### 2.4 维度（dimension capability）

```telora
type DimensionId = enum { 'RegionDim, 'CountryDim, 'OrderMonthDim, 'CategoryDim };

# 层级是 ExprNode 数组（至少一项）；既可以是普通属性列，也可以是派生表达式。
# 派生维度同时 lowering 到 projection 与 grouping。requires 语义同指标。
{id: 'RegionDim, entity: 'Order, requires: [],
 levels: [attribute_node(attrs, 'OrderRegion)],
 grain: 'OrderGrain, alias: "region", subjects: ['Analyst, 'Manager]}
{id: 'OrderMonthDim, entity: 'Order, requires: [],
 levels: [order_month_expr],
 grain: 'OrderGrain, alias: "order_month", subjects: ['Analyst]}
```

### 2.5 关系（relationship catalog）

```telora
type RelationshipId = enum { 'OrderToCustomer, 'OrderToCampaign };

# 每条关系：类型化端点 + 类别（Safe / FanOut）+ 结构化等值 mapping
{id: 'OrderToCustomer, from: 'Order, to: 'Customer, kind: 'Safe,
 mapping: {from_attribute: 'OrderCustomer, to_attribute: 'CustomerId}}
{id: 'OrderToCampaign, from: 'Order, to: 'Campaign, kind: 'FanOut,
 mapping: {from_attribute: 'OrderCampaign, to_attribute: 'CampaignId}}
```

- `'Safe`（grain-safe）关系参与计划路径选择；`'FanOut`（会扩张 grain）关系只参与
  可达性分类。目标只能经 fan-out 到达时，请求会被拒绝。
- mapping 必须是列等值形式，且 `from_attribute` 属于 `from` 实体、
  `to_attribute` 属于 `to` 实体（构造时会校验）。

### 2.6 grain 与基础数据源

`GrainBase(GrainId, EntityId)` 声明每个自然 grain 的基础实体。同一个知识可以
声明多个 grain，每个 grain 独立选择 base：

```telora
let grain_bases: Array(GrainBase(GrainId, EntityId)) = [
    {grain: 'OrderGrain, base: 'Order},
    {grain: 'ItemGrain, base: 'PackageItem},
];
```

请求的共同 grain 决定本次计划从哪个 base 出发；缺失该 grain 的 base 会失败。

### 2.7 组装知识

```telora
let profile: PlanProfile = {
    allow_joins: ['Inner],
    allow_scalar: ['Substr],
    allow_aggregate: ['Sum, 'Count],
    allow_filter: 'False,
    allow_grouping: 'True,
    allow_ordering: 'False,
    allow_limit: 'False,
};

let knowledge: EnterpriseKnowledge(EntityId, AttributeId, MeasureId, DimensionId, RelationshipId, GrainId, Subject) = {
    revision: "retail-catalog-v7",
    grain_bases: grain_bases,
    vocabulary: { entities: entities, attributes: attrs, relationships: rels },
    metrics: metrics,
    dimensions: dimensions,
    plan_profile: profile,
};
```

`revision` 是稳定、可读的 String 标签（如 `"retail-catalog-v7"`），由 Plan 原样
保留；它不参与 profile/validation 判定，也不会出现在生成的 SQL/Query 中。

知识在 `make_query_creator(knowledge)` 构造时被校验：grain base 实体存在且不重复、
实体/属性/关系引用完整、required entities 存在、表达式递归有效（属性存在且属于
能力实体）、mapping 端点归属正确、维度至少一个层级、投影 alias 唯一且合法。

## 3. 提交请求

```telora
let request: QueryRequest(MeasureId, DimensionId, Subject, GrainId, GrainId) = {
    measures: [
        {id: 'Revenue, subject: 'Analyst, input: 'OrderGrain},
    ],
    dimensions: [
        {id: 'CountryDim, subject: 'Analyst, input: 'OrderGrain},
        {id: 'OrderMonthDim, subject: 'Analyst, input: 'OrderGrain},
    ],
};
```

- 每个请求项 = 能力 id + 原始 subject + 有类型输入（grain）。
- 请求只表达「要什么」；不携带物理表达式、mapping、Plan 节点或 SQL。
- 一次计划的所有请求必须共享同一个 grain。

## 4. 生成 Plan 与 Query

```telora
let creator = make_query_creator(knowledge);
let plan: Plan = creator(request);
let query: Query = transform_sqlite_with_profile(plan, knowledge.plan_profile);
```

`make_query_creator` 会：解析并授权每个能力、校验 grain 兼容性、按请求共同 grain
解析基础数据源、推导实体并选择安全关系、组装覆盖全部请求且没有额外请求的 Plan、
验证结构并通过 `knowledge.plan_profile` 的 profile 覆盖检查，然后发布 Plan。
任一步失败都会 `fail!`，不会发布部分 Plan。

生成的 Plan 使用位置化 alias：基础实体是 `e0`，后续 join 引入的实体依次是
`e1`、`e2`……（确定性、保证唯一）。派生表达式中的绑定常量（如 `substr` 的 1、7）
按 SQL 占位符顺序进入 Query bindings。

## 5. 失败类别

| 消息前缀 | 含义 |
| --- | --- |
| `enterprise-knowledge.missing-measure` / `missing-dimension` | 请求的能力不在目录中 |
| `enterprise-knowledge.unauthorized-measure` / `unauthorized-dimension` | subject 未被该能力授权 |
| `enterprise-knowledge.measure-grain-conflict` / `dimension-grain-conflict` | grain 与能力定义或计划 grain 不一致 |
| `enterprise-knowledge.no-base-for-grain` | 请求的共同 grain 没有声明 base |
| `enterprise-knowledge.unsafe-path` / `missing-path` / `truncated-path` | 能力目标实体无法通过安全关系到达 |
| `enterprise-knowledge.empty-request` | 请求没有命名任何 measure 或 dimension |
| `enterprise-knowledge.invalid-knowledge` | 知识构造校验失败（含表达式引用无效属性） |
| `enterprise-knowledge.structure` | 组装出的 Plan 未通过 QueryBuilder 结构验证 |
| `enterprise-knowledge.profile-violation` | Plan 使用了 profile 不接受的能力（如不允许 `'Substr`） |

`fail!` 的 subject 保留来源；每个请求级失败的诊断都会包含原始 subject。
