# 私有企业题面：物流履约分析

Customer 创建 Order，Order 从 Warehouse 发货，Warehouse 位于 Region。每个
Order 选择 Carrier 和 ServiceLevel，并产生一个或多个 Package。Package 包含
PackageItem，PackageItem 指向 Product，Product 属于 Category。

## 关系与物理映射

下列 many-to-one 方向在当前 grain 下安全：

```text
Order -> Customer
Order -> Warehouse
Warehouse -> Region
Order -> Carrier
Order -> ServiceLevel
Package -> Order
PackageItem -> Package
PackageItem -> Product
Product -> Category
```

下列方向会扩张 grain：

```text
Order -> Package
Package -> PackageItem
```

每个关系携带由企业所有的结构化物理 mapping，至少表达源表、目标表和等值 join
两侧的列引用；不能只保存预渲染的 join predicate String。

## 指标与维度

- `OrdersCreated`：Order grain，`COUNT(orders.id)`，获准。
- `DeliveredPackages`：Package grain，获准，并要求 Order。
- `UnitsShipped`：PackageItem grain，获准，并要求 Package 和 Order。
- `OrderMonth`：要求 Order，获准。
- `CustomerTier`：要求 Customer，获准。
- `OriginRegion`：要求 Region，获准。
- `CarrierName`：要求 Carrier，获准。
- `ServiceName`：要求 ServiceLevel，获准。
- `ProductCategory`：要求 Category，能力获准，但从 Order grain 只能经 fan-out
  到达。
- `DeliveryException`：属于封闭 Dimension vocabulary，但没有获准 capability。

不同 natural grain 的指标不能自动组合。当前没有预聚合或 allocation policy。

## 物理查询事实

基础数据源是 `orders`。领域模型必须以公共 eDSL 要求的结构化关系表达式表达以下
物理事实，使规范 Plan 自身足以被确定地转换为 SQL，而 transform 不需要反查
模型。下列 SQL 写法定义预期语义，不授权把它们作为预渲染片段直接塞入 Plan：

- `OrdersCreated`：`COUNT(orders.id)`；
- `OrderMonth`：`substr(orders.created_at, 1, 7)`；
- `CustomerTier`：`customers.tier`；
- `OriginRegion`：`regions.name`；
- `CarrierName`：`carriers.name`；
- `ServiceName`：`service_levels.name`。

每条安全关系的物理 mapping 必须足以生成对应的只读 join。领域知识必须声明自己
接受的 QueryBuilder PlanProfile；同一个规范 Plan 经公共 QueryBuilder 的 SQLite
transform 必须产生逐字节相同的 Query，其中所有动态数据位于 bindings。

## 可见验证场景

合法场景：`OrdersCreated`，按 `OrderMonth`、`CustomerTier`、`OriginRegion`、
`CarrierName` 和 `ServiceName` 分组。它应发布 Order grain 的只读规范 Plan，
完整保留全部请求投影、grouping、到这些实体的安全关系和物理 mapping，并由公共
QueryBuilder 将该 Plan 转成包含 `SELECT`、`FROM orders`、五个必要 join 与对应
`GROUP BY` 的 Query。

非法场景：`OrdersCreated`，按 `ProductCategory` 和 `DeliveryException` 分组。该请求
必须失败，不发布任何部分计划，也不产生 SQL。具体诊断数量和恢复求值结构不属于领域
模型的验收目标。

最终计划 revision 固定为 `logistics-ontology-v1`。
