# A2 设计、验证结果与限制

## 设计要点

- **公共边界**：`EnterpriseKnowledge -> make_query_creator -> Fn(Request) -> Plan`
  保持精确类型。`Plan`/`PlanProfile`/`Query` 完全来自 QueryBuilder 公共交付；
  eDSL 只组装标准算子并验证 profile，不定义替代 Plan，不负责 `Plan -> Query`。
- **类型化身份**：实体/属性/指标/维度/关系/grain 全部使用封闭 enum 类型参数；
  String 只用于物理标识符（表名、列名、投影 alias）、revision 标签与诊断消息，
  不充当领域身份。`EnterpriseKnowledge.revision` 是稳定、可读的 String 标签，
  由 Plan 原样保留（QueryBuilder 最新草案将 `Plan.revision` 定义为 String）。
- **结构化物理表达式**：`ExprNode = Attribute | Bound(Val) | ScalarCall(ScalarFn,
  Array(ExprNode))`，封闭、类型化、递归；派生维度（如
  `substr(created_at, 1, 7)`）同时 lowering 到 projection 与 grouping，绑定
  常量进入 Query bindings。Telora 不支持参数化递归 family，因此 `ExprNode` 是
  具体递归类型，属性叶子用目录索引（`AttributeRef`），由类型化辅助
  `attribute_node(attrs, id)` 从 `AttributeId` 建立并在知识构造时递归校验。
- **grain 通用化**：`GrainId` 是企业封闭词汇的一部分，作为类型参数参与
  `EnterpriseKnowledge`、能力与请求；不再有 eDSL 固定的 `Grain` 枚举。
- **per-grain base**：`grain_bases: Array(GrainBase(GrainId, EntityId))` 结构化
  声明每个自然 grain 的基础数据源；请求的共同 grain 决定本次计划从哪个 base
  出发。缺失 base、重复 base 定义、请求 grain 冲突都会失败。
- **Request 形状**：沿用 DESIGN.md 参考形状 `Request(Id, Subject, Input)` 与
  `QueryRequest(MeasureId, DimensionId, Subject, MeasureInput, DimensionInput)`；
  本实现把 `MeasureInput`/`DimensionInput` 统一为 grain 类型参数。请求只表达
  「要什么」，不携带物理表达式、mapping、Plan 节点或 SQL；成功 Plan 精确覆盖
  原始 measure/dimension 顺序。
- **grain 规则**：每个发布的 Plan 恰好运行在一个 grain 上。所有 measure 请求的
  grain 必须彼此相等且等于各自能力定义；dimension 请求的 grain 必须等于计划
  grain。这是对 DESIGN.md「指标 grain 兼容性 / grain 冲突」的确定性解释。
- **授权**：每个能力携带 `subjects: Array(Subject)`；请求 subject 必须命中，
  否则 `fail!`。所有请求级失败把原始 subject 作为诊断 subject。
- **required entities**：每个 measure/dimension 能力携带 `requires:
  Array(EntityId)`（结构化、精确类型的事实，不是注释）。lowering 为能力自身
  实体与每个 required entity 分别选择安全路径，再按稳定顺序合并（能力实体路径
  在前，required 按声明顺序；required == base/主实体时产生空路径并自然去重；
  共享边只保留首次出现）。fan-out-only、missing、truncated 继续原子失败。
- **路径选择**：BFS 逐层扩展关系目录，只使用 `'Safe` 边；同层实体保留
  lexicographically 最小的目录索引序列；深度上限 8；`reached` 集合阻止环。
  完整可达性用 safe ∪ fan-out 判断，从而把目标分类为 safe / fan-out-only /
  missing / truncated，后三类都阻止发布。
- **路径合并**：每个能力先产生「自身实体路径 + 各 required entity 路径」（按
  `requires` 声明顺序），全部能力再按请求顺序（measure 在前、dimension 在后）
  拼接，按关系索引去重（共享边只保留首次出现）。每个路径都从所选 base 出发，
  因此合并序列的每个 join 的 left alias 都已被更早的 source/join 定义。
- **alias 方案**：位置化 alias —— 基础实体 `e0`，第 p 条边引入的实体
  `e{p+1}`；同一实体在计划内只有一个 alias，保证唯一且确定性。
- **组装校验顺序**：知识构造期校验（`validate_knowledge`，只运行一次）→
  请求期（解析/授权/grain/base/路径）→ `validate_plan` 结构验证 →
  `within_profile` profile 验证 → 发布。

## 验证结果

```text
./bin/telora run main -C ontology
# profile_ok=True sql=SELECT SUM("e0"."amount") AS "revenue", "e1"."country" AS "country",
#   substr("e0"."created_at", ?, ?) AS "order_month" FROM "orders" AS "e0"
#   JOIN "customers" AS "e1" ON "e0"."customer_id" = "e1"."id"
#   GROUP BY "e1"."country", substr("e0"."created_at", ?, ?)
# bindings = [Int(1), Int(7), Int(1), Int(7)]（投影内 substr、grouping 内 substr，
# 与 SQL 占位符顺序一一对应）

./bin/telora run verify -C ontology
# verify ok bases=orders,packages,package_items projections=3,2,1,1 joins=1,1,1,2
#   bindings=4 required=ok profile=ok bases_differ=True
# （OrderGrain 从 orders 出发：revenue + country(join customers) + order_month(substr)；
#   ItemGrain 从 package_items 出发：item_count + category(join products)；
#   PackageGrain 从 packages 出发：DeliveredPackages 因 requires=['Order] 包含
#   Package -> Order join；ItemGrain 的 UnitsShipped 因 requires=['Package,'Order]
#   按顺序包含 PackageItem -> Package -> Order 两个 join。三个不同 grain 选择三个
#   不同 base，证明不是单一 base 的别名改写。）

# 跨 grain 端到端 SQL（main/verify 之外补充观察）：
# Package grain（DeliveredPackages）：
#   SELECT COUNT("e0"."id") AS "delivered_packages" FROM "packages" AS "e0"
#   JOIN "orders" AS "e1" ON "e0"."order_id" = "e1"."id"
# PackageItem grain（UnitsShipped）：
#   SELECT SUM("e0"."amount") AS "units_shipped" FROM "package_items" AS "e0"
#   JOIN "packages" AS "e1" ON "e0"."package_id" = "e1"."id"
#   JOIN "orders" AS "e2" ON "e1"."order_id" = "e2"."id"

./bin/telora run invalid -C ontology --best-effort
# enterprise-knowledge.unsafe-path: capability target reachable only via fan-out
# relationships（CampaignDim 只能经 fan-out 到达）→ status error，无 output

./bin/telora run invalid-grain -C ontology --best-effort
# enterprise-knowledge.no-base-for-grain: no base data source declared for grain
#（knowledge_no_base 只声明 OrderGrain，ItemGrain 请求失败）→ status error

./bin/telora run invalid-conflict -C ontology --best-effort
# enterprise-knowledge.measure-grain-conflict: requested measures must share one
# grain（OrderGrain 与 ItemGrain 混用）→ status error

./bin/telora run invalid-profile -C ontology --best-effort
# enterprise-knowledge.profile-violation: plan uses capabilities outside
# knowledge.plan_profile（knowledge_no_substr 不允许 'Substr，但请求包含
# order_month 派生维度）→ status error

./bin/telora run invalid-attribute -C ontology --best-effort
# enterprise-knowledge.invalid-knowledge: expression attribute belongs to a
# different entity（Revenue 表达式引用 CustomerCountry，但指标属于 Order；
# make_query_creator 构造时即失败）→ status error

./bin/telora check @test/ontology.telora -C ontology
# status ok（公共契约检查：覆盖、join、grouping、profile、structure、bindings、
# 双 grain 双 base；纯断言：within_profile(plan, profile_no_substr) == False）

./bin/telora show @bin/main.telora -C ontology
# 正常列出 main 的定义（creator/plan/query/output 等）
```

虚构示例知识使用零售目录领域（orders/packages/package_items/customers/
products/campaigns），不包含物流题面中的实体、表、列、公式或 mapping。

## 已知限制

- 关系 mapping 只能是列等值（QueryBuilder `Join` 的固有边界）。
- 请求不支持 filter / ordering / limit；`Plan.filter`/`Plan.ordering` 为空、
  `limit` 为 `'None`。
- 计划只使用 `'Inner` join；`'Left` 与 fan-out 边不进入计划（fan-out 目标直接
  阻止发布）。
- 深度上限固定为 8；恰好在边界上仍有未访问后继时按 truncation 处理并阻止发布。
- Telora 不支持参数化递归 family，因此 `ExprNode` 是具体递归类型，属性叶子用
  目录索引 `AttributeRef`；类型化构造辅助 + 构造期递归校验保证正确性
  （见 PUBLIC-CONTRACT.md 的 ExprNode 说明）。
- 聚合只使用 QueryBuilder 的 `AggregateFn`；指标表达式是 `Aggregate(kind, arg)`
  其中 `arg` 为 `ExprNode` 的 lowering 结果（Column/ScalarCall/Bound）。
- QueryBuilder 不为 `Plan`/`Query` 提供 JSON/schema 边界；main 示例直接输出
  SQL 文本。
- eDSL 模块没有导出内部路径/校验辅助函数；企业作者只需构造知识记录并调用
  `make_query_creator`（见 DSL-TUTORIAL.md）。

## 对 QueryBuilder 的依赖确认

- 已确认公共导入路径 `query-builder/query-builder.telora`（见
  QUERY-BUILDER-FEEDBACK.md 的修订记录）。
- `validate_plan` / `within_profile` / `transform_sqlite_with_profile` 行为与
  公共契约一致（黑盒验证）。
