# eDSL 第二次送审反馈

Host 审核 Ent-1 最终模型时确认：题面中的 capability required entities 没有进入机器模型。当前 `MetricDef` 只有表达式所属实体；`DeliveredPackages` 注释声称“要求 Order”，但 Package grain 计划以 Package 为 base 时不会选择 `Package -> Order`；`UnitsShipped` 同样不会强制选择 Package 与 Order。notes/评论不能代替 EnterpriseKnowledge 事实。

请在 QueryBuilder 新 revision 类型发布后增量修订：

1. 为 measure/dimension capability 提供结构化、精确类型的 required entities（可统一为 `requires: Array(EntityId)`；命名由实现决定）。不得使用 String、预构造 join 或 builder callback。
2. lowering 必须为能力自身实体及 required entities 分别选择安全路径，再按稳定顺序合并。required entity 等于 base/主实体时自然去重；fan-out-only、missing、truncated 继续原子失败。
3. 知识构造时验证 required entity 存在；文档明确顺序与去重语义。
4. 增补跨 grain 端到端测试：
   - Package grain 的 DeliveredPackages Plan 从 `packages` 出发，并包含 `Package -> Order` join；
   - PackageItem grain 的 UnitsShipped Plan 从 `package_items` 出发，并按顺序包含 `PackageItem -> Package -> Order` 两个 join。
5. 同步适配 QueryBuilder 的可读 String revision，使最终 Plan 直接携带领域 revision 标签，而非把标签放在 Plan 外。
6. 保持已经通过的 PhysicalExpr/Substr、自定义 GrainId、多 grain base、路径确定性和 profile 语义不变。

该问题属于 eDSL 抽象缺口，不是 A3 可以在现有 API 内绕过的问题。
