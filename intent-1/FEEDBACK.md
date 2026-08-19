# 公共查询面检视意见（依据私有文字意图）

## 1. `OriginRegion` 与“发货仓库所属地区”的语义对应 —— 已解决

教程 §2 与契约“业务定义消歧”现已明确：
`OriginRegion` = **发货仓库（发货来源）所属地区，即订单从哪个仓库发出、该仓库位于
哪个地区**。这与合法意图的第三个分组维度“发货仓库所属地区”直接对应，映射无歧义。
合法意图的三个分组维度（`OrderMonth`、`CustomerTier`、`OriginRegion`）与指标
`OrdersCreated` 均在 `Order` 粒度可用且共享同一粒度，符合 Request 组合规则，可以
忠实表达。

## 2. 非法意图可忠实表达且预期被拒绝 —— 无缺口

非法意图（统计已创建订单数并按商品类别分组）可以完全由公共词汇表达：
`ProductCategory` 是 `DimensionId` 的封闭成员，教程 §5 与契约均明确说明按 `Order`
粒度请求会以 `enterprise-knowledge.unsafe-path` 失败且不产生 Query。这满足意图中
“保留请求并验证它被拒绝且没有 Query”的要求，公共面无需变更。

## 3. 能力概览表格表述 —— 已解决

教程与契约的维度表格均已改为“请求粒度 + 状态”双列结构，`ProductCategory` 直接标为
“不可用（unsafe-path）”，`DeliveryException` 标为“不可用（无获准能力）”，不再存在
“可用粒度列写成 Order（不可用）”的误读空间。

## 4. 其余核对结果（未发现问题）

- 每个请求项必须显式携带 `subject`（当前唯一授权主体 `Analyst`），教程示例已覆盖。
- 失败诊断前缀与“同一 Request 每次 lower 得到逐字节相同 Query”的确定性保证，足以
  支撑后续 verify（重复 lowering 严格相同）与 invalid（被拒绝且无 Query）验证入口。
- 公共面未暴露表、列、alias、join 或 SQL 模板等物理细节，未发现不必要泄漏。
- 本轮未发现新的歧义、缺口或泄漏；公共查询面可以放行。

## 5. 实际 lowering 验证结果（与文档一致）

- 合法 Request（`OrdersCreated` + `OrderMonth`/`CustomerTier`/`OriginRegion`，均
  Order 粒度）：`lower` 成功，Query 含非空 `sql: String` 与 `bindings: Array(Val)`；
  重复 `lower` 结果逐字节相同。
- 非法 Request（`OrdersCreated` + `ProductCategory`，Order 粒度）：`lower` 以
  `enterprise-knowledge.unsafe-path`（“capability target reachable only via fan-out
  relationships”）失败，无任何 Query 发布；普通与 `--best-effort` 两种运行模式均失败。
- 未发现教程/契约与实际 lowering 行为之间的差异。
