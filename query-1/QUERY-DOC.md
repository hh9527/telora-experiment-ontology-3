# Ent-1 查询能力

## JSON intent

```json
{
  "measures": [
    {"id": "DeliveredPackages", "subject": "network-ops@example.com", "input": "All"}
  ],
  "dimensions": [
    {"id": "CarrierName", "subject": "network-ops@example.com", "input": "All"}
  ],
  "filters": [
    {"id": "OrderMonth", "subject": "network-ops@example.com", "input": {"Month": "2026-07"}, "op": "Eq"},
    {"id": "OriginRegion", "subject": "network-ops@example.com", "input": {"Region": "华东"}, "op": "Eq"}
  ],
  "ordering": [
    {"target": {"Measure": "DeliveredPackages"}, "direction": "Desc"},
    {"target": {"Dimension": "CarrierName"}, "direction": "Asc"}
  ],
  "limit": 5
}
```

请求必须至少包含一个 measure。`subject` 是查询发起者标识，只用于授权和诊断，
不会自动成为筛选条件或 binding。数组顺序会保留到最终查询中。所有 measure 必须
属于同一个 grain。

`filters` 使用封闭维度、比较操作和有类型值：

- `OrderMonth`：`{"Month":"2026-07"}`，支持 `Eq`、`Ge`、`Le`。
- `CustomerTier`：`{"Tier":"Gold"}`，支持 `Eq`、`Ge`、`Le`。
- `OriginRegion`：`{"Region":"华东"}`，支持 `Eq`、`Ge`、`Le`。
- `ProductCategory`：`{"Category":"电子产品"}`，支持 `Eq`、`Ge`、`Le`。

输入变体必须和维度匹配。多个筛选按数组顺序使用 AND。筛选维度不必同时出现在
`dimensions` 中。

`ordering` 只能引用已经请求的 measure 或 dimension，方向为 `Asc` 或 `Desc`。
`limit` 必须是正整数；不限制时写 `null`。所有筛选值和 limit 都进入 bindings，
不得为了绕过诊断删除条件或扩大查询范围。

## 可查询指标

- `OrdersCreated`：创建的订单数，Order grain。
- `DeliveredPackages`：已送达的包裹数，Package grain。
- `UnitsShipped`：已发货的件数，Package item grain。

## 可查询维度

- `OrderMonth`：订单创建月份。
- `CustomerTier`：客户等级。
- `OriginRegion`：发货仓库所属地区。
- `CarrierName`：承运商名称。
- `ServiceName`：服务水平名称。
- `ProductCategory`：产品类别；从 Order grain 使用会产生 grain 扩张冲突。
- `DeliveryException`：投递异常；属于已知词汇，但当前没有查询能力。

`OrderMonth`、`CustomerTier`、`OriginRegion`、`CarrierName` 和 `ServiceName` 可以与
三个 grain 的已授权指标组合。`ProductCategory` 可与 `UnitsShipped` 安全组合，但从
Order grain 使用会产生 grain 扩张冲突。不同 grain 的指标不能放入同一个请求。
`CarrierName`、`ServiceName` 可用于分组，但当前不能作为筛选维度。

信息缺失或业务语义有歧义时，先向用户询问，不要猜测。明确提交后若无法生成查询，
根据 Telora 诊断修正结构；如果业务能力本身不存在，则用业务语言说明原因。
