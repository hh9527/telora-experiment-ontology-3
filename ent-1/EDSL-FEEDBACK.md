# A3 对 eDSL 送审版的能力审查（第三轮）

更新说明：eDSL 第三轮草案把 `EnterpriseKnowledge.revision` 改为 `String`（与
QueryBuilder `Plan.revision: String` 对齐），并在 `MetricDef`/`DimensionDef` 新增
`requires: Array(EntityId)` 结构化声明。本文件记录第三轮审查结论。

## 结论

- eDSL 第三轮草案已满足 `ent-1/DOMAIN.md` 的全部建模需要，且解决了上轮
  E2（“指标要求实体”无机器表达）与跨包 Q4（revision 类型不对齐）。
- 合法场景（`OrdersCreated` 按五个维度分组）可完整表达并确定 lowering；
  非法场景（`ProductCategory` fan-out-only、`DeliveryException` 无能力）仍由
  capability/grain 校验在构造 Plan 前拒绝。
- 不再有阻塞项；仅保留语言级使用注意（E1）与建模边界说明。

## 上轮意见的落实情况

1. **revision 对齐**：`EnterpriseKnowledge.revision: String`，由 Plan 原样保留、
   不参与判定、不进入 SQL/Query；与 QueryBuilder 第三轮 `Plan.revision: String`
   类型一致。✅（解决跨包 Q4；领域 `logistics-ontology-v1` 可直接作为知识
   revision。）
2. **required entities**：`MetricDef`/`DimensionDef` 新增
   `requires: Array(EntityId)`；lowering 为能力自身实体与每个 required entity
   分别选择安全路径并按稳定顺序合并（等于 base/主实体时自然去重）。✅
   （解决 E2；`DeliveredPackages` 可声明 `requires: ['Order]`，
   `UnitsShipped` 可声明 `requires: ['Package, 'Order]`。）

## 领域需要 vs eDSL 能力核对（第三轮）

| 领域需要 | eDSL 对应 | 覆盖情况 |
| --- | --- | --- |
| `OrdersCreated = COUNT(orders.id)` | `MetricDef(aggregate: 'Count, expr: attribute_node(...), requires: [])` | ✅ |
| `OrderMonth = substr(orders.created_at, 1, 7)` | `DimensionDef(levels: [scalar_node('Substr, ...)])` | ✅ |
| `CustomerTier`/`OriginRegion`/`CarrierName`/`ServiceName` | `DimensionDef(levels: [attribute_node(...)])` | ✅ |
| `DeliveredPackages` 要求 Order | `MetricDef(requires: ['Order])` | ✅ |
| `UnitsShipped` 要求 Package 与 Order | `MetricDef(requires: ['Package, 'Order])` | ✅ |
| 五个安全 many-to-one 关系 + 等值列 mapping | `RelationshipDef(kind: 'Safe, mapping: {...})` | ✅ |
| 多 grain 指标 | `GrainBase` + `GrainId` 参数 | ✅ |
| `ProductCategory` 从 Order grain 仅 fan-out | `'FanOut` 分类 → unsafe-path 拒绝 | ✅ |
| `DeliveryException` 无获准能力 | 目录中不存在 → missing-dimension 拒绝 | ✅ |
| revision 标签 `logistics-ontology-v1` | `EnterpriseKnowledge.revision: String` | ✅ |

## 建模侧使用注意与边界（不要求 eDSL 修改）

- **E1（保留）**：`let _ = require(...)` 绑定形式不被前端接受，验证/测试模块的
  断言绑定必须具名。建模侧 `verify.telora`/`tests/logistics.telora` 使用具名
  `c_*` 绑定。
- **requires 与路径合并顺序**：能力实体路径在 required 路径之前、required 按
  `requires` 声明顺序。合法场景所有能力 `requires: []`，join 顺序不变（五个
  join 按请求顺序 + 规范路径合并）。
- **ExprNode 属性索引**：知识与表达式共享同一属性目录（`model.telora` 模块级
  先定义目录再构造表达式）。
- **封闭 vocabulary 含无能力成员**：`'DeliveryException` 纳入封闭 `DimensionId`
  但不进入目录，使非法请求可类型化并在运行时被 missing-dimension 拒绝。
- **非法场景诊断**：`ProductCategory` + `DeliveryException` 请求以
  missing-dimension 失败；DOMAIN 明确诊断数量不属于验收目标。

## 非目标确认

- 不支持非等值 mapping、请求级 filter/ordering/limit、OR/NOT/CASE/HAVING/
  子查询、Bytes：领域不需要。
- 公共 API 无 `Any`/`Dyn`/native/String 身份：符合工作规则。
