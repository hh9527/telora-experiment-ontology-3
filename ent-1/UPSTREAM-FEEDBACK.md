# A3 对上游公共交付的反馈

本文件记录 A3 在按 `ent-1/DOMAIN.md` 建立私有 EnterpriseKnowledge 时对
QueryBuilder 与 ontology eDSL 公共交付的具体使用摩擦。分区记录，仅描述公共
交付可直接观察到的行为，不推测上游私有实现。本版基于 `ent-1-model-feedback`
修订轮：模型已加入有类型筛选、排序与 Top N，不再使用空值兼容写法。

## QueryBuilder（`query-builder` package）

按批准版公共契约使用 `import "query-builder/lib.telora" as qb;`，建模中没有遇到
阻断性摩擦。可观察到的具体点：

1. **`Plan.revision: String` 原样保留符合预期。** 领域把 `"logistics-ontology-v1"`
   放进 `Plan.revision`，`validate`/`transform_sqlite` 均不修改它，验收断言直接
   成立，无需额外映射。

2. **`'Substr` 与 `standard_profile` 派生能力满足题面。** `OrderMonth` 的
   `substr(orders.created_at, 1, 7)` 用 `qb.scalar_call('Substr, [...])` 表达，2-3
   参元数校验与渲染 `substr(...)` 均与文档一致；`with_scalar_functions`/
   `with_aggregate_functions`/`with_join_kinds` 派生 profile 行为符合文档，
   `standard_profile` 派生的 profile 已开启 `allow_filter`/`allow_order_by`/
   `allow_limit`。

3. **bindings 按 `?` 出现顺序收集。** 同一表达式出现在 SELECT 与 GROUP BY 时
   重复收集（规范场景 bindings 为 `[Int(1), Int(7), Int(1), Int(7)]`）；筛选
   字面量与 LIMIT 也按 SQL 文本中 `?` 的顺序进入 bindings，业务值绝不内联。已
   实际断言三个跨 grain 场景：Package grain
   `[1, 7, "2026-07", "华东", 5]`；PackageItem grain
   `[1, 7, "2026-04", 1, 7, "2026-06", "Gold", 3]`；
   `[1, 7, "2026-07", 10]`。

4. **`read_profile` 的“读”定义偏窄（建议性）。** 题面要求的只读 Plan 包含 join、
   group 与聚合（不写数据），而 `read_profile` 限定“无 join/group/聚合/distinct”。
   领域从 `standard_profile` 派生自己的 profile，不依赖该命名；但该命名容易让
   新作者误以为“读查询 = 单表无聚合”。

5. **筛选/排序/limit 的具体化符合契约。** `Eq/Ge/Le` 渲染为中缀 `(left op right)`，
   多个筛选按请求顺序 `And` 组合；`ORDER BY` 从已请求的 measure/dimension 表达式
   构造（如 `ORDER BY count(pk.id) DESC, ca.name ASC`）；`LIMIT ?` 与 N 的 binding
   均按契约生成，无摩擦。

## ontology eDSL（`ontology` package）

按批准版公共契约使用 `import "ontology/lib.telora" as e;`，多 grain 指标、受控
计算维度、有类型筛选与排序/Top N 按文档工作。可观察到的具体点：

1. **多 grain 指标按请求决定 base grain，符合题面。** `OrdersCreated`（Order）、
   `DeliveredPackages`（Package）、`UnitsShipped`（PackageItem）同时登记在同一个
   `Knowledge`；一次请求的 measure 决定 base grain，跨 grain 请求明确失败。题面
   “不同 natural grain 的指标不能自动组合”直接成立。

2. **受控计算维度满足 `OrderMonth`。**
   `dimension_computed(computed_expr('Substr, [scalar_arg_attribute('Orders,
   'CreatedAt), scalar_arg_literal('Int(1)), scalar_arg_literal('Int(7))]))` 正确
   lower 为 `qb.ScalarCall`，字面量进入 bindings，SQL 中只有 `?` 占位符。

3. **能力缺失与 fan-out grain 冲突都会以 `fail!` 拒绝且不发布 Plan。** 非法场景
   （`OrdersCreated` 按 `ProductCategory` 与 `DeliveryException` 分组）产生两条
   诊断：`missing dimension capability`（DeliveryException 无能力条目）与
   `dimension grain conflict: only reachable via fan-out relations`
   （ProductCategory 从 Order 只能经 fan-out 到达），status error 且无 output，
   符合题面“失败且不产生 SQL”。

4. **家族值需要显式 concrete 类型标注（使用注意，非缺陷）。** 不带标注调用
   `e.entity`/`e.relation`/`e.measure`/`e.dimension`/`e.capability`/`e.filter_capability`
   时，类型推断会把封闭 Atom 实参固定为各自的 singleton 类型，导致同一数组内
   不同 Atom 无法统一。给每个 family 值加 concrete 应用标注后一切正常。建议教程
   明确要求企业知识为 family 值提供 concrete 标注。

5. **“要求 Order”等前置条件没有显式声明（建议性）。** `DeliveredPackages` 的
   “要求 Order”与 `UnitsShipped` 的“要求 Package 和 Order”目前通过关系图（请求
   需要 Order grain 维度时选择 `Package -> Order`、`PackageItem -> Package` 等
   安全路径）自然满足，没有显式 requires 字段。当前验收场景不依赖它；若后续轮次
   需要强制这些前置关系，建议 eDSL 提供显式要求声明。

6. **有类型筛选能力按文档工作。** `filter_capability(id, authorized, to_val)` 的
   `to_val` 把领域输入（`'Month`/`'Tier`/`'Region`/`'Category`）转为 `qb.Val`，
   值最终成为 bindings；`subject` 只用于授权/诊断，不进入 SQL/bindings；维度与
   输入变体不匹配（如 OrderMonth 收到 `'Tier(...)`）在 `to_val` 以 `fail!` 原子
   拒绝。筛选维度即使不投影也走 grain-safe 路径并合并必要 join。

7. **排序与 limit 的失败语义符合契约。** `ordering` 只引用已请求目标，引用未
   请求目标诊断 `order target dimension not requested` 并原子失败；`limit` 非正
   数诊断 `limit must be a positive integer` 并原子失败。`DeliveryException` 的
   筛选变体类型上可表达但无 `filter_capabilities` 条目，能力解析诊断
   `missing filter capability` 并原子失败。

8. **fan-out-only 筛选的诊断消息随求值上下文变化（观察，非缺陷）。** 独立求值时，
   Order grain 下 `ProductCategory` 筛选（能力获准但仅能经 fan-out 到达）诊断
   `dimension grain conflict: only reachable via fan-out relations`；在同一个
   `--best-effort` 模块里先求值其他非法请求时，同一请求可诊断
   `missing filter capability`。两种情况下都原子失败且不发布 Plan/Query；诊断
   消息不属于公共 API，A3 据此在 verify/invalid 中只断言“必须失败”，不断言该
   场景的具体消息。

9. **`Knowledge`/`QueryRequest` 的 filter/ordering/limit 字段在启用后工作正常。**
   本轮模型实际使用这些字段（不再提供空值），`Knowledge.filter_capabilities`、
   `QueryRequest.filters/ordering/limit` 与 `FilterInput` 类型参数全部按文档生效，
   没有额外摩擦。
