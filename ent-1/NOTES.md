# A3 NOTES：物流履约 EnterpriseKnowledge（私有模型）

## 模型选择

### 词汇与目录
- `MeasureId`：`OrdersCreated`（Order grain）、`DeliveredPackages`（Package grain）、
  `UnitsShipped`（PackageItem grain），全部登记在各自 grain 实体上（eDSL 多 grain
  支持）。
- `DimensionId`：`OrderMonth`、`CustomerTier`、`OriginRegion`、`CarrierName`、
  `ServiceName`（合法安全维度）+ `ProductCategory`（能力获准，但从 Order grain
  只能经 fan-out 到达；从 PackageItem grain 可经安全路径到达，用于跨 grain 合法
  场景，分组与筛选都受同一路径语义约束）+ `DeliveryException`（封闭词汇，故意
  不登记任何能力，请求命中即失败）。
- `FilterInput` 有类型变体：`'Month(String)`（OrderMonth）、`'Tier(String)`
  （CustomerTier）、`'Region(String)`（OriginRegion）、`'Category(String)`
  （ProductCategory）、`'Exception(String)`（DeliveryException：类型上可表达但无
  任何获准能力，能力解析必然失败）。`to_val` 拒绝维度与输入变体不匹配。
- `EntityId` 10 个：customers/orders/warehouses/regions/carriers/service_levels/
  packages/package_items/products/categories。
- `AttributeId` 16 个：`Id` 与各外键/名称列。`Id` 在多个实体上复用（各表主键列
  都是 `"id"`），通过 `AttributeRef(entity, attribute)` 消歧。
- `RelationId` 11 个：9 条安全多对一（`'GrainSafe`, `'Inner`）+
  2 条 fan-out（`Order -> Package`、`Package -> PackageItem`，`'FanOut`, `'Inner`）。

### 物理映射
- 表别名：orders=`o`、customers=`cu`、warehouses=`wh`、regions=`rg`、carriers=`ca`、
  service_levels=`sl`、packages=`pk`、package_items=`pi`、products=`pr`、
  categories=`ct`，全局唯一。
- 每条关系使用结构化等值列对（`attribute_pair`），如
  `Order -> Customer` = `orders.customer_id == customers.id`；
  `Warehouse -> Region` = `warehouses.region_id == regions.id`；不保存预渲染 SQL。

### 指标与维度表达式
- `OrdersCreated = COUNT(orders.id)`：`'Aggregate({function: 'Count, distinct:
  'False})` on `'Orders.'Id`。
- `OrderMonth = substr(orders.created_at, 1, 7)`：受控计算维度
  `dimension_computed(computed_expr('Substr, [attribute('Orders,'CreatedAt),
  literal('Int(1)), literal('Int(7))]))`，lower 为 `qb.ScalarCall`，字面量成为
  bindings。
- `CustomerTier = customers.tier`、`OriginRegion = regions.name`、
  `CarrierName = carriers.name`、`ServiceName = service_levels.name`：直接属性维度。

### 能力与 PlanProfile
- 每个已登记 measure/dimension 有 `capability(id, fn(subject, input) { input ==
  'All })`；`DeliveryException` 无能力条目。
- 每个可筛选维度有 `filter_capability(id, authorized, to_val)`：
  `OrderMonth/CustomerTier/OriginRegion/ProductCategory` 共 4 条；`authorized` 不
  依赖 subject（subject 只用于授权/诊断，不进入 SQL/bindings），`to_val` 把领域
  输入转为 `qb.Val` 并拒绝错误变体。
- `plan_profile`：`standard_profile` 收窄为 `['Inner]` join、`['Count]` aggregate、
  `['Substr, 'Eq, 'Ge, 'Le, 'And]` scalar；`allow_filter/order_by/limit` 由
  standard_profile 派生保留。
- `revision = "logistics-ontology-v1"` 原样进入 `Plan.revision`。
- 请求至少一个 measure 决定 base grain；筛选/排序/limit 为可选能力，按
  `QueryRequest` 的 `filters/ordering/limit` 字段表达。

## 验证结果（全部实际运行）

- `./bin/telora run main -C ent-1`（严格）：输出 summary + SQL，满足
  `revision=logistics-ontology-v1`、6 个投影、5 个 join（cu/wh/rg/ca/sl）、
  5 个 group_by；SQL 为 `SELECT count(o.id) AS orders_created, substr(o.created_at,
  ?, ?) AS order_month, cu.tier, rg.name, ca.name, sl.name FROM orders AS o INNER
  JOIN customers/warehouses/regions/carriers/service_levels ... GROUP BY
  substr(o.created_at, ?, ?), cu.tier, rg.name, ca.name, sl.name`，动态值全部进入
  bindings（`[1,7,1,7]`）。
- `./bin/telora run verify -C ent-1`（严格）：全部检查通过——
  - 场景 0（Order grain）：revision、`qb.validate` Ok、覆盖 6/5/5/0（投影/
    join/group/ordering）、join 别名集合、确定性、SQL 形状、bindings、多 grain
    指标目录存在；
  - 场景 A（Package grain，`DeliveredPackages` + OrderMonth Eq 2026-07 +
    OriginRegion Eq 华东 + CarrierName 分组 + 指标降序/名称升序 + limit 5）：
    2 投影/4 join/1 group/2 ordering，SQL 含 WHERE/GROUP BY/ORDER BY/LIMIT ?，
    业务值（月份、地区、N）不内联，bindings 精确
    `[1, 7, "2026-07", "华东", 5]`；
  - 场景 B（PackageItem grain，`UnitsShipped` + OrderMonth Ge 2026-04/Le 2026-06 +
    CustomerTier Eq Gold + ServiceName 分组 + 降序/升序 + limit 3）：bindings
    精确 `[1, 7, "2026-04", 1, 7, "2026-06", "Gold", 3]`；
  - 场景 C（PackageItem grain，`UnitsShipped` + OrderMonth Eq 2026-07 +
    ProductCategory 分组 + 降序/升序 + limit 10）：从 PackageItem 经
    PackageItem -> Product -> Category 安全路径成功，bindings 精确
    `[1, 7, "2026-07", 10]`；
  - profile 与筛选能力目录检查通过（Eq/Ge/Le/And/Substr/Count/Inner 在 profile
    内，`allow_filter/order_by/limit` 开启，4 条 filter_capabilities 存在）。
- `./bin/telora run invalid -C ent-1 --best-effort`：非零退出、status error、
  无 output；每个 export 原子失败并产生诊断——既有回归（`missing dimension
  capability` + `dimension grain conflict`）、`missing filter capability`
  （DeliveryException 筛选）、`invalid filter value for OrderMonth: expected Month`
  （变体不匹配）、`limit must be a positive integer`、`order target dimension not
  requested`、Order grain 下 ProductCategory 筛选失败（独立求值为 grain conflict；
  多场景模块中同一请求可显示为 `missing filter capability`，见风险 4）。
- `./bin/telora check @test/logistics.telora -C ent-1`：status ok（revision、
  覆盖、validate、确定性、SQL 形状、bindings、多 grain、filter capability、
  profile 九项检查）。
- `./bin/telora query exports @bin/main.telora -C ent-1`：正常列出 `output`。

## 风险与说明

1. **bindings 数量依赖 SQL 渲染。** 规范场景 `OrderMonth` 表达式同时出现在
   SELECT 与 GROUP BY，bindings 为 4 个 `[1,7,1,7]`；筛选场景的字面量与 LIMIT
   也按 SQL 中 `?` 顺序进入 bindings（场景 A 5 个、B 8 个、C 4 个）。若上游未来
   改为 GROUP BY 使用别名或去重字面量，verify 的 bindings 断言需同步更新；当前
   与公共契约一致。
2. **join 顺序由 eDSL 确定性选择。** verify 使用集合成员检查（covers_alias）
   而非顺序断言，对上游 tie-break 变化更稳健。
3. **“要求 Order”/“要求 Package 和 Order”没有显式 eDSL 声明。** 通过关系图
   （Package->Order、PackageItem->Package->Order 安全路径）自然满足；若未来需要
   强制前置关系，需要 eDSL 提供显式 requires（已写入 UPSTREAM-FEEDBACK）。
4. **fan-out-only 筛选的诊断消息随求值上下文变化。** 独立求值时 Order grain 下
   ProductCategory 筛选为 `dimension grain conflict`；多非法场景 `--best-effort`
   模块中同一请求可显示 `missing filter capability`。原子失败在两种情况下都成立
   （不发布 Plan/Query），诊断消息不属于公共 API；verify/invalid 只断言失败，
   不断言该场景具体消息（见 UPSTREAM-FEEDBACK 第 8 条）。
5. **`DeliveryException` 的拒绝依赖“不登记能力”。** 这是题面语义（“没有获准
   capability”）；若未来误加能力会使非法场景失效，模型注释已说明。
6. **家族值必须加 concrete 类型标注。** 不加标注时封闭 Atom 实参会固定为
   singleton 类型导致无法统一（见 UPSTREAM-FEEDBACK 第 4 条）；当前模型全部
   family 值均带 concrete 应用标注。
7. **公共查询面约束。** 后续公共 facade 阶段不得泄漏表名/列名/alias/join
   mapping/SQL 片段；本 NOTES 中的物理细节只属于私有模型交付物。
