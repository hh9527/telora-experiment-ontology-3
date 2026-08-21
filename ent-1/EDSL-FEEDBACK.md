# A3 对 ontology eDSL 送审版的能力审查（最终建模前）

审查依据：`ent-1/DOMAIN.md`（物流履约分析题面）与 `ontology` 公共教程/契约
（`DSL-TUTORIAL.md`、`PUBLIC-CONTRACT.md`，本次复核基于 `edsl.a2` 最新送审版，
含新增的 typed filter / ordering / limit 能力）。本文件只评估 eDSL 能否承载题面
要求的 `EnterpriseKnowledge + Request -> Plan` 建模与 lowering；Plan/Query 具体化
由 QueryBuilder 负责，已在 `QUERY-BUILDER-FEEDBACK.md` 单独审查。

## 领域需要 eDSL 表达的内容

- 10 个实体（`orders`、`customers`、`warehouses`、`regions`、`carriers`、
  `service_levels`、`packages`、`package_items`、`products`、`categories`），
  各带表名/别名/属性列目录；
- 9 条安全关系（均为多对一等值 join）+ 2 条 fan-out 关系
  （`Order -> Package`、`Package -> PackageItem`），mapping 必须是结构化等值列对；
- 三个不同 grain 的获准指标：Order grain 的 `OrdersCreated`
  （`COUNT(orders.id)`）、Package grain 的 `DeliveredPackages`（要求 Order）、
  PackageItem grain 的 `UnitsShipped`（要求 Package 和 Order）；
- 5 个安全维度：`OrderMonth`（= `substr(orders.created_at, 1, 7)`）、
  `CustomerTier`（= `customers.tier`）、`OriginRegion`（= `regions.name`）、
  `CarrierName`（= `carriers.name`）、`ServiceName`（= `service_levels.name`）；
- `ProductCategory`（能力获准但从 Order grain 只能经 fan-out 到达）与
  `DeliveryException`（封闭词汇但无获准能力）两个必须被拒绝的分组维度；
- 合法场景必须发布 Order grain 的只读 Plan；非法场景必须失败且不发布部分 Plan；
- 最终 Plan revision 固定为 `logistics-ontology-v1`；
- 题面可见场景只要求投影 + 分组；筛选（filter）、排序（ordering）与 Top N
  （limit）是送审版新增能力，本领域当前不要求使用，但需确认不构成阻碍。

## 逐项结论（基于 `edsl.a2` 最新送审版）

### 已解决项（核心建模能力）

1. **维度标量表达式已补齐。** 送审版引入 `DimensionExpr`（`'Attribute` /
   `'Computed`）、`ComputedExpr {function: qb.ScalarFunction, args: Array(ScalarArg)}`
   与 `ScalarArg`（`'Attribute(AttributeRef)` / `'Literal(qb.Val)`），并在教程给出
   与题面一致的按月截断示例。`OrderMonth` 可以建模为
   `dimension_computed(computed_expr('Substr, [scalar_arg_attribute('Orders,
   'CreatedAt), scalar_arg_literal('Int(1)), scalar_arg_literal('Int(7))]))`，lowering
   为 `qb.ScalarCall`、字面量进入 bindings。阻断项解除。

2. **多 grain 指标已支持。** 送审版移除单一 `base_entity`：指标登记在各自的
   grain 实体上，一次请求的 measure 决定 base grain，所有请求 measure 必须位于
   同一实体、不兼容时明确失败。题面 `OrdersCreated`（Order）、`DeliveredPackages`
   （Package，经 Package -> Order 安全关系要求 Order）、`UnitsShipped`
   （PackageItem，经 PackageItem -> Package -> Order 要求 Package 和 Order）都可以
   登记；题面“不同 natural grain 的指标不能自动组合”与“请求 measure 必须同一
   实体”直接对应。

3. **revision 已改为 `String`。** `Knowledge.revision: String` 原样进入
   `qb.Plan.revision`（不参与验证/具体化）；题面“最终计划 revision 固定为
   `logistics-ontology-v1`”可直接承载，无需 Int/String 映射或外部备注。

### 满足项（草案已覆盖，无需改动）

4. **类型级 ID 目录。** `MeasureId/DimensionId/EntityId/AttributeId/RelationId`
   全部由企业定义，`Knowledge`/`QueryRequest` 参数化；契约明示目录查找不使用
   String 反查，符合题面“不使用 String 语义身份”。

5. **实体与属性建模。** `entity(id, source_ref(table, alias), attributes)` 与
   `attribute(id, entity, column)` 足以表达 10 个实体的表名、别名与属性列目录；
   契约还要求别名互不重复，本领域 10 个别名全局唯一可满足。

6. **结构化关系 mapping。** `relation(...)` + `relation_mapping(attribute_pair(...))`
   把每个关系表达为 `left 属性 == right 属性` 的结构化等值列对，lowering 为
   `qb.EquiCondition`，满足题面“至少表达源表、目标表和等值 join 两侧的列引用，
   不保存预渲染 join predicate String”。

7. **grain 分类与 fan-out 拒绝（可达性三分类已明确）。** `'GrainSafe`/`'FanOut`
   方向分类配合契约第 3 步：完整可达性使用 safe 与 fan-out 的并集，把目标分为
   safe、fan-out-only（仅能经 fan-out 到达，视为 grain 冲突）或 missing（不可达，
   视为缺失）。`ProductCategory` 从 Order 出发仅能经 `Order->Package`、
   `Package->PackageItem`（fan-out）+ `PackageItem->Product`、`Product->Category`
   （safe）到达，属于 fan-out-only，会被判为 grain 冲突并拒绝，与非法场景预期
   完全一致。

8. **能力目录。** `capability(id, authorized)` 按请求逐项授权；`DeliveryException`
   作为无能力条目存在时，请求在能力解析阶段失败，不发布部分 Plan，符合题面
   “没有获准 capability”。

9. **PlanProfile 透传与校验。** `knowledge.plan_profile` 由企业声明，计算维度
   使用的标量函数必须在 `scalar_functions` 内、指标聚合必须在
   `aggregate_functions` 内，组装后调用 `qb.validate`；题面“领域知识必须声明
   自己接受的 PlanProfile”可满足。领域将派生 profile 为
   `with_scalar_functions(with_aggregate_functions(with_join_kinds(standard_profile,
   ['Inner]), ['Count]), ['Substr])`。

10. **确定性 lowering 与原子发布。** 同一 knowledge + request 逐字节产生相同
    Plan；任何授权/grain/路径/profile 失败都不发布部分 Plan，符合题面“失败且不
    产生 SQL”。

11. **维度跨实体安全路径（路径选择算法已明确）。** 契约第 3 步明确从 base grain
    实体 BFS：安全路径最短边数优先，同长度按目录索引序列字典序最小（先声明的
    关系优先），多目标按请求顺序合并、共享边只保留首次出现，最大深度 8。本领域
    各非 base 维度都有唯一安全路径：`CustomerTier`（1 跳 Order->Customer）、
    `OriginRegion`（2 跳 Order->Warehouse->Region）、`CarrierName`/`ServiceName`
    （各 1 跳），路径长度均远小于 8；字典序 tie-break 不触发，但选择保持确定。

### 新增能力评估（typed filter / ordering / limit）

12. **有类型筛选（`FilterInput` + `FilterOp` + `FilterCapability`）。** 筛选引用
    `DimensionId` + 领域输入 + 标准比较操作（`'Eq`/`'Ge`/`'Le`），`to_val` 由企业
    把领域输入转换为 `qb.Val`，多个筛选按请求顺序 `And` 组合、值成为
    `qb.Literal`/binding；查询方不能提交 `qb.Expr`、列、表、alias、SQL 或
    mapping。筛选维度即使不投影也会执行授权、走 grain-safe 路径并合并必要 join。
    机制与题面“不使用 String 反查、不泄漏物理细节”一致；题面可见场景不使用筛选，
    不构成阻碍。

13. **排序（`OrderRequest`）只引用已请求目标。** `ordering` 引用 `'Measure(id)` /
    `'Dimension(id)` + `Asc`/`Desc`，eDSL 只从已解析的 measure/dimension 规范表达式
    构造 `qb.OrderItem`；引用未请求的目标诊断并原子失败。机制正确；题面可见场景
    不使用排序，不构成阻碍。

14. **Top N（`limit`）。** `limit: Option(Int)` 只接受正整数，非正数诊断失败；
    成功时写入 `Plan.limit`，由 QueryBuilder 生成 `LIMIT ?` 并把 N 放入 bindings。
    机制正确；题面可见场景不使用 limit，不构成阻碍。

15. **profile 覆盖新增能力。** 契约明确实际使用的 `filter`/`order_by`/`limit` 及
    `Eq/Ge/Le/And` 必须被 profile 接受（`standard_profile` 派生 profile 已开启
    `allow_filter`/`allow_order_by`/`allow_limit`）。本领域当前 profile 只声明
    `['Count]`/`['Substr]`，不声明筛选/排序/limit 能力，符合“只接受题面实际需要
    的能力”的收窄方向。

### 建议性观察（不阻断当前题面）

16. **“要求 Order”等前置条件没有显式字段。** `DeliveredPackages` 的“要求 Order”、
    `UnitsShipped` 的“要求 Package 和 Order”目前通过关系图（请求维度需要时选择
    Package -> Order 等安全路径）自然表达，没有显式的 requires 声明。当前验收场景
    不依赖该字段；若后续轮次需要强制这些前置关系，建议 eDSL 提供显式要求声明。

17. **同长度路径的字典序 tie-break 依赖关系声明顺序。** 契约把同长度路径选择
    绑定到目录索引序列的字典序，领域可通过关系声明顺序控制选择；本领域没有同长
    度替代路径，因此不构成阻碍，仅提示后续作者需要留意声明顺序会影响路径 join
    顺序。

18. **`Knowledge`/`QueryRequest` 现在强制携带 `FilterInput` 与
    `filter_capabilities` 字段。** 即使领域不使用筛选，`Knowledge` 结构也包含
    `filter_capabilities: Array(FilterCapability(...))`、`QueryRequest` 包含
    `filters`/`ordering`/`limit`，需要企业定义 `FilterInput` 类型并在最终建模时提供
    （可为空数组/`None`）。这是类型参数化带来的建模成本，不是功能缺陷；最终建模
    将按“空筛选/排序/limit”处理。

## 结论

`edsl.a2` 最新送审版在已补齐维度标量表达式、多 grain 指标、`revision: String`
与明确的路径选择/可达性三分类基础上，新增了有类型筛选（`Eq/Ge/Le` + `And`、
`to_val` 值转换）、排序（仅已请求目标）与 Top N（正整数 limit）能力，并同步扩展
profile 覆盖规则。A3 审查的阻断项与摩擦项全部解除：类型目录、实体/关系/grain/
能力建模与 lowering 保证能够承载题面的合法与非法场景；`ProductCategory` 被判为
fan-out-only grain 冲突、`DeliveryException` 因无能力条目被拒，均符合题面预期。
新增能力是加法式的，不要求当前题面使用；最终建模只需为 `Knowledge`/
`QueryRequest` 的新字段提供空值。A3 可以按 DOMAIN.md 进行最终建模。
