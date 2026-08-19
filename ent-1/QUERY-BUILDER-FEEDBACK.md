# A3 对 QueryBuilder 公共草案的能力审查（第三轮）

更新说明：`qb.a1` 第三轮草案把 `Plan.revision` 从 `Int` 改为稳定、可读的 `String`
标签（如 `"logistics-ontology-v1"`），由 Plan 逐字节保留、不参与 profile/validation
判定、不进入 SQL/Query。本文件记录第三轮审查结论。

## 结论

- `Plan.revision: String` 直接满足领域“最终计划 revision 固定为
  `logistics-ontology-v1`”的要求：模型无需再维护 Int↔String 映射，可把该标签
  直接写入 Plan。
- `Substr`、grouping 绑定值规则、导入路径、确定性保证、profile 机制均与第二轮
  相同，领域合法场景的覆盖保持不变。
- 新增跨包协调事项：eDSL 的 `EnterpriseKnowledge.revision` 目前仍是 `Int`，
  eDSL 组装 Plan 时（`revision = 知识 revision`）与新的
  `Plan.revision: String` 类型不兼容（实测报 `cannot unify Int with String`）。
  需要 eDSL 侧把知识 revision 同步改为 `String`（或提供等价映射），否则企业
  模型无法通过合法链路。详见下方 Q4。

## 上轮意见的落实情况（含本轮变更）

1. `ScalarFn` 含 `'Substr`，arity 2–3；grouping 标量参数内允许 `'Bound`
   （`GROUP BY substr(col, ?, ?)` 合法且确定），顶层绑定值禁止。✅（不变）
2. 导入路径 `query-builder/query-builder.telora`。✅（不变）
3. `Plan.revision` 改为 `String`，逐字节保留，不参与 profile/validation，不出现在
   SQL/Query 中。✅（本轮新变更，见 Q4）

## 领域算子覆盖核对（第三轮）

| 领域需求 | 对应算子/结构 | 覆盖情况 |
| --- | --- | --- |
| `OrdersCreated = COUNT(orders.id)` | `'Aggregate({kind: 'Count, arg: 'Column(...)})` | ✅ |
| `OrderMonth = substr(orders.created_at, 1, 7)` | `'ScalarCall({kind: 'Substr, ...})` | ✅ |
| `CustomerTier`/`OriginRegion`/`CarrierName`/`ServiceName` | `'Column`（join 后） | ✅ |
| 五个安全 many-to-one join | `joins: Array(Join)` + `'Inner` | ✅ |
| 分组（列 + 标量调用） | `grouping: Array(Expr)` | ✅ |
| revision 标签 `logistics-ontology-v1` | `Plan.revision: String` | ✅ |
| 确定性 / bindings 顺序 | `transform_sqlite` | ✅ |

## 新增协调事项

### Q4. `Plan.revision` 类型变更需要 eDSL 对齐

- 现象：eDSL `EnterpriseKnowledge.revision` 仍是 `Int`，其 lowering 保证
  “`revision` = 知识 revision” 会以 Int 值填充新的 `Plan.revision: String`，
  在 eDSL 实现处直接类型冲突。企业 crate 实测
  `./bin/telora run main -C ent-1` 报
  `error: @src/model.telora:294:15: field revision: cannot unify Int with String`。
- 建议：eDSL 侧把 `EnterpriseKnowledge.revision` 改为 `String`（或提供
  类型化 revision 标签构造），并同步教程/契约；企业模型会把
  `logistics-ontology-v1` 作为知识 revision 直接传入。此事项已同步记录到 eDSL
  反馈（`ent-1/EDSL-FEEDBACK.md`）。

## 建模侧边界确认（不要求 QueryBuilder 修改）

- **多 grain 指标**：`DeliveredPackages`（Package grain）、`UnitsShipped`
  （PackageItem grain）经各自 `GrainBase` 表达；QueryBuilder 不限制合法表名
  标识符，不构成阻塞。
- **非法场景**：`ProductCategory`（fan-out-only）与 `DeliveryException`（无能力）
  由 EnterpriseKnowledge 侧 capability/grain 校验在进入 QueryBuilder 前拒绝。
- **PlanProfile**：`allow_joins: ['Inner]`、`allow_scalar: ['Substr]`、
  `allow_aggregate: ['Count]`、`allow_grouping: 'True` 足以覆盖合法场景。
- **确定性**：合法场景 bindings 为 substr 常量
  `['Int(1), 'Int(7), 'Int(1), 'Int(7)]`（投影 + grouping），逐字节确定。
- **join 顺序**：按依赖顺序输出五个 join（Order→Customer、Order→Warehouse、
  Warehouse→Region、Order→Carrier、Order→ServiceLevel）。

## 非目标确认

- `Val` 不含 Bytes、无 OR/NOT/HAVING/CASE、无子查询：领域不需要。
- 公共 API 无 `Any`/`Dyn`/native/String 语义身份：符合工作规则。
