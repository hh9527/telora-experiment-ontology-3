# Ent-1 建模笔记

## 1. 范围

依据 `ent-1/DOMAIN.md` 用 ontology eDSL 建立物流履约分析私有模型，并验证
`EnterpriseKnowledge + Request -> Plan -> Query` 完整链路（QueryBuilder SQLite
transform），覆盖 Order / Package / PackageItem 三个 grain。

## 2. 模型选择

### 2.1 Vocabulary

- `EntityId`：Order、Customer、Warehouse、Region、Carrier、ServiceLevel、
  Package、PackageItem、Product、Category（10 个实体）。
- `AttributeId`：25 个物理属性（每个实体含 id 列；Order 含 customer_id/
  warehouse_id/carrier_id/service_level_id；Warehouse 含 region_id；Package 含
  order_id；PackageItem 含 package_id/product_id；Product 含 category_id）。
- `RelationshipId`：11 条关系，其中 9 条 `'Safe`（many-to-one，与 DOMAIN 安全
  方向一致），2 条 `'FanOut`（Order→Package、Package→PackageItem，扩张 grain）。
- `GrainId`：{'Order, 'Package, 'PackageItem}，grain_bases 各自独立（Order→
  orders、Package→packages、PackageItem→package_items）。
- `Subject`：{'Analyst}（DOMAIN 无 subject 细粒度，全部能力授权给单一主体）。

### 2.2 能力目录

- 指标：
  - `OrdersCreated`（Order grain，COUNT(orders.id)，requires: []）；
  - `DeliveredPackages`（Package grain，COUNT(packages.id)，requires: ['Order]）；
  - `UnitsShipped`（PackageItem grain，COUNT(package_items.id)，
    requires: ['Package, 'Order]）。
  `requires` 由 eDSL 提供结构化声明，lowering 强制并入所需实体的安全路径，
  机器可校验（见第 3 节三个 grain 的 Plan/Query）。
- 维度（全部 Order grain、requires: []）：
  `OrderMonth`（派生维度 `substr(orders.created_at, 1, 7)`，ExprNode ScalarCall）、
  `CustomerTier`、`OriginRegion`、`CarrierName`、`ServiceName`（属性列维度）、
  `ProductCategory`（能力获准，但从 Order grain 仅 fan-out 可达）。
  `DeliveryException` 属于封闭 `DimensionId` 词汇但无获准 capability，不在目录中
  → 非法请求在运行时被 missing-dimension 拒绝。

### 2.3 物理映射

- 基础数据源：Order grain 基表 `orders`（alias `e0`）；Package grain 基表
  `packages`；PackageItem grain 基表 `package_items`。
- 每个 safe 关系都是单列等值 join 的结构化 mapping（from_attribute /
  to_attribute），由 eDSL 降级为 `'Inner` join，`left` 引用已定义 alias。
- Order-grain 合法场景五个 join 顺序（请求顺序 + 规范路径合并）：
  Order→Customer(e1)、Order→Warehouse(e2)、Warehouse→Region(e3)、
  Order→Carrier(e4)、Order→ServiceLevel(e5)。

### 2.4 PlanProfile

```text
allow_joins: ['Inner], allow_scalar: ['Substr], allow_aggregate: ['Count],
allow_filter: 'False, allow_grouping: 'True,
allow_ordering: 'False, allow_limit: 'False
```

### 2.5 revision

- `EnterpriseKnowledge.revision = "logistics-ontology-v1"`（String，与
  QueryBuilder `Plan.revision: String` 一致），由 Plan 原样保留，不进入
  SQL/Query。满足 DOMAIN“最终计划 revision 固定为 logistics-ontology-v1”。

## 3. 验证结果（三个 grain 的真实输出）

### 3.1 Order grain（合法主场景）

```text
./bin/telora run main -C ent-1
  → Plan JSON（revision: "logistics-ontology-v1"）+ Query JSON
  → SQL: SELECT COUNT("e0"."id") AS "orders_created",
        substr("e0"."created_at", ?, ?) AS "order_month",
        "e1"."tier" AS "customer_tier", "e3"."name" AS "origin_region",
        "e4"."name" AS "carrier_name", "e5"."name" AS "service_name"
    FROM "orders" AS "e0"
    JOIN "customers" AS "e1" ON "e0"."customer_id" = "e1"."id"
    JOIN "warehouses" AS "e2" ON "e0"."warehouse_id" = "e2"."id"
    JOIN "regions" AS "e3" ON "e2"."region_id" = "e3"."id"
    JOIN "carriers" AS "e4" ON "e0"."carrier_id" = "e4"."id"
    JOIN "service_levels" AS "e5" ON "e0"."service_level_id" = "e5"."id"
    GROUP BY substr("e0"."created_at", ?, ?), "e1"."tier", "e3"."name",
             "e4"."name", "e5"."name"
  → bindings: ['Int(1), 'Int(7), 'Int(1), 'Int(7)]
```

### 3.2 Package grain（DeliveredPackages，requires: ['Order]）

```text
./bin/telora run verify -C ent-1（内部断言）
  → source = packages / e0；revision = logistics-ontology-v1；
    投影 = COUNT(e0.id) AS delivered_packages；恰一个 join Package -> Order；
    确定性通过。
  → SQL: SELECT COUNT("e0"."id") AS "delivered_packages"
    FROM "packages" AS "e0"
    JOIN "orders" AS "e1" ON "e0"."order_id" = "e1"."id"
  → bindings: []
```

### 3.3 PackageItem grain（UnitsShipped，requires: ['Package, 'Order]）

```text
./bin/telora run verify -C ent-1（内部断言）
  → source = package_items / e0；revision = logistics-ontology-v1；
    投影 = COUNT(e0.id) AS units_shipped；join 顺序
    PackageItem -> Package -> Order；确定性通过。
  → SQL: SELECT COUNT("e0"."id") AS "units_shipped"
    FROM "package_items" AS "e0"
    JOIN "packages" AS "e1" ON "e0"."package_id" = "e1"."id"
    JOIN "orders" AS "e2" ON "e1"."order_id" = "e2"."id"
  → bindings: []
```

### 3.4 非法场景（真实输出）

```text
./bin/telora run invalid -C ent-1 --best-effort
  → 非零退出；诊断
    enterprise-knowledge.missing-dimension: dimension capability not in catalog
    （组合非法请求：ProductCategory + DeliveryException；先命中无能力维度）
  → 无 Output(String) effect（无 plan/query 发布）

./bin/telora run invalid-product-category -C ent-1 --best-effort
  → 非零退出；诊断
    enterprise-knowledge.unsafe-path: capability target reachable only via fan-out relationships
    （单独验证 ProductCategory 从 Order grain 因 fan-out-only 失败，
      证明模型对扩张 grain 方向的 fan-out mapping 分类正确）
  → 无 Output(String) effect
```

### 3.5 其他

```text
./bin/telora check @test/logistics.telora -C ent-1
  → telora.check/v1 summary status: "ok"（含三个 grain 的契约断言）

./bin/telora show @bin/main.telora -C ent-1
  → 查询成功，输出合法入口的语义定义
```

## 4. 风险与边界

1. **Packages/package_items 等表名由模型选定**：DOMAIN 只固定 `orders` 相关
   物理事实；`packages`/`package_items`/`categories` 等表名与列名是本模型按
   领域关系推断的合法标识符。
2. **常量参数以 bindings 呈现**：`substr` 的 1、7 是绑定值（`?`），无动态数据时
   SQL 也有四个占位符；确定且符合 QueryBuilder 契约。
3. **数组字面量断言需显式标注**：`verify` 中比较 `OperatorSet`/`bindings` 时
   数组字面量需 `Array(JoinKind)`/`Array(Val)` 标注（`UPSTREAM-FEEDBACK.md`
   Q1）；`let _ = ...` 绑定形式不被前端接受（E1）。
4. **非法场景诊断顺序**：组合非法请求（ProductCategory + DeliveryException）以
   missing-dimension 失败；单独 ProductCategory 入口证明 unsafe-path
   （fan-out-only）。DOMAIN 明确“诊断数量与恢复结构不属于验收目标”。
