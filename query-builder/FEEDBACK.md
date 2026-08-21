# QueryBuilder Host feedback

当前 `qb.a1` 暂不放行。请仅处理以下阻断/契约问题，完整重验后重新提交 `qb.a1`：

1. 在标准 `ScalarFunction` vocabulary 中增加 `Substr`，表达 SQLite
   `substr(value, start)` 与 `substr(value, start, length)`。`validate` 必须只接受 2 或
   3 个参数，`transform_sqlite` 必须确定性渲染；profile 白名单、公共契约、教程和测试
   同步覆盖。Ent-1 的 `OrderMonth` 依赖 `substr(orders.created_at, 1, 7)`，这是阻断项。
2. 明确并实现 `PlanProfile.allow_distinct`：它约束 `AggCall.distinct`；为 `False` 时，
   `validate` 必须以 profile violation 拒绝任何 `distinct = 'True` 的聚合。当前不增加
   顶层 `SELECT DISTINCT` 算子。
3. 修正文档中的导入边界：QueryBuilder 包内模块/测试可以使用
   `import "@src/lib.telora" as qb;`；外部依赖 crate 必须使用
   `import "query-builder/lib.telora" as qb;`（依赖名以 manifest 为准）。公共教程和契约
   都要明确这两种路径，不能让外部 eDSL 作者照抄 `@src`。

非阻塞建议（profile helper、projection alias 唯一性、`read_profile` 教程补充）本轮不要求
扩展，避免扩大修订范围。

## 第二次 Host review

真实领域 review 进一步发现 revision 类型不满足题面。QueryBuilder 的设计要求只规定
Plan 必须完整保留 knowledge revision，并未规定它是 `Int`；Ent-1 明确要求最终 Plan 的
revision 为 `"logistics-ontology-v1"`。请把 `Plan.revision` 从 `Int` 改为 `String`，同步
更新构造、示例、测试、教程和公共契约，并完整重验。不得用额外字段或外部备注绕过 Plan
本身的 revision 契约。
