# Ent-1 查询能力

## JSON intent

```json
{
  "measures": [
    {"id": "OrdersCreated", "subject": "analyst@example.com", "input": "All"}
  ],
  "dimensions": [
    {"id": "OrderMonth", "subject": "analyst@example.com", "input": "All"}
  ]
}
```

请求必须至少包含一个 measure。`subject` 是查询发起者标识，当前 `input` 只能是 `"All"`。
数组顺序会保留到最终查询中。所有 measure 必须属于同一个 grain。

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

`OrderMonth`、`CustomerTier`、`OriginRegion`、`CarrierName` 和 `ServiceName` 可以与三个 grain
的已授权指标组合。不同 grain 的指标不能放入同一个请求。非法组合不会产生部分 SQL，而会
返回带来源位置的诊断。
