# Ent-1 模型送审反馈

Host 真实复跑确认 Order-grain 合法场景、String revision、五个 join、Substr bindings、verify/check 与组合非法入口均通过。但本轮要求的领域级跨 grain 验收尚未落到 Ent-1 产物：`verify.telora`/`tests/logistics.telora` 没有构造 DeliveredPackages 或 UnitsShipped 请求，`NOTES.md` 仍明确写“跨 grain 指标未端到端验证”。ontology package 的通用 fixtures 不能代替企业模型实例验证。

请做一次仅限 Ent-1 的增量修订：

1. 在 `verify.telora` 与 `tests/logistics.telora` 使用本模型 `knowledge` 构造 Package grain 的 `DeliveredPackages` 请求，断言：source=`packages`、revision=`logistics-ontology-v1`、投影为 Count(packages.id)、恰有 `Package -> Order` join，并能确定转换为 Query。
2. 构造 PackageItem grain 的 `UnitsShipped` 请求，断言：source=`package_items`、投影为 Count(package_items.id)、join 顺序为 `PackageItem -> Package -> Order`，并能确定转换为 Query。
3. 单独验证 `ProductCategory` 在 Order grain 因 fan-out-only 失败；当前组合非法请求先命中无能力的 DeliveryException，只证明 missing-dimension，不能证明本模型的 fan-out mapping。可以增加独立 invalid 入口并记录真实输出。
4. 更新 `NOTES.md`：删除“跨 grain 未端到端验证”的过时风险，记录三个 grain 的实际 Plan/Query 验证结果；保留由领域题面未固定的表/列命名边界。
5. 不修改 QueryBuilder 或 eDSL；现有 Order-grain 主场景与组合非法场景保持不变。
