# Ent-1 公共查询面反馈

A4 依据私有文字意图做黑盒检视后，公共 API 能力完整且无物理泄漏，但公共业务说明有两处需要消歧。只修订公共查询面文档/必要的公共测试，不修改私有模型语义。

1. 明确 `OriginRegion` 的业务定义是“发货仓库（发货来源）所属地区”。Host 已对照 DOMAIN 确认：该维度通过 Order -> Warehouse -> Region 表达，正是私有意图中的“发货仓库所属地区”。查询设计者不应靠猜测把两个近似词映射起来。
2. 统一 `ProductCategory` 的能力表述。不要在“可用粒度”列写 `Order（不可用）` 这种自相矛盾文本；改成独立的“请求粒度 = Order”和“状态 = 不可用（unsafe-path）”列，或等价的单义结构。`DeliveryException` 同样清楚区分词汇成员与未获准能力。
3. 保持公共类型、`lower(Request)->Query`、确定性和物理隔离不变；非法 ProductCategory 意图仍应可忠实构造并由 lower 拒绝。
4. 同步 `QUERY-DESIGNER-TUTORIAL.md` 与 `PUBLIC-QUERY-CONTRACT.md`，必要时在 query-surface 测试中保留业务 vocabulary 可导入验证。
