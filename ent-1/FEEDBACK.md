# Ent-1 参数化查询模型增量反馈

> 本轮不是只适配 eDSL 新增字段。当前 `FilterInput = 'None`、
> `filter_capabilities: []` 以及 NOTES 中“本领域不开放 typed filter/排序/Top N”
> 的结论不能支撑真实业务查询，必须修订。

请保持现有物理模型、封闭词汇和 grain 安全规则，为后续公共查询面加入以下领域能力。

## 1. 领域筛选输入

- 为 `OrderMonth`、`CustomerTier`、`OriginRegion` 提供有类型筛选输入；至少能表达
  月份字符串、客户等级字符串、地区名称字符串，并拒绝维度与输入变体不匹配。
- 为从 `PackageItem` grain 安全可达的 `ProductCategory` 提供类别名称筛选能力，
  但不得因此放松它从 `Order` grain 只能经 fan-out 到达的既有拒绝语义。
- 领域值必须由 `FilterCapability.to_val` 转为 `qb.Val`，最终成为 SQL bindings；
  不得把表名、列名、alias、SQL 或 `qb.Expr` 暴露给查询方。
- `subject` 只用于能力授权和诊断，不是查询筛选值，不得自动写入 SQL/bindings。

## 2. 跨 grain 验证

实际构造并执行下列代表性 Plan/Query，不能只验证空字段：

- Package grain：`DeliveredPackages`，筛选月份等于 `2026-07`、地区等于 `华东`，
  按 `CarrierName` 分组，按指标降序、名称升序，limit 5。
- PackageItem grain：`UnitsShipped`，筛选月份 `Ge 2026-04`、`Le 2026-06` 且
  客户等级等于 `Gold`，按 `ServiceName` 分组，按指标降序、名称升序，limit 3。
- PackageItem grain：`UnitsShipped`，筛选月份等于 `2026-07`，按
  `ProductCategory` 分组，按指标降序、名称升序，limit 10。该场景应成功；保留
  `OrdersCreated + ProductCategory` 的 fan-out grain conflict 失败回归。

## 3. 契约要求

- `plan_profile` 明确允许实际使用的 filter/order_by/limit 与
  `Substr/Eq/Ge/Le/And`、`Count`、`Inner`。
- 每个成功场景断言 SQL 占位符与 bindings 的顺序和值，且业务月份、地区、等级、
  类别和 N 均不内联。
- 同一请求重复 lowering/transform 必须产生逐字节一致的 SQL 与 bindings。
- 缺失 capability、输入变体不匹配、非正 limit、未请求的排序目标与既有 grain
  conflict 必须原子失败，不发布部分 Plan/Query。
- 更新 model、main/verify/invalid、tests、NOTES 与上游反馈；不要把本轮能力描述为
  “仅提供空值兼容”。完成并实际运行所有 Ent-1 验证后再提交 `ent-1-model.a3`。
