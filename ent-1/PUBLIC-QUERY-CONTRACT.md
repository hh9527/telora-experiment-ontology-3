# Ent-1 公共查询契约

公共查询面（`ent-1/query.telora`）是从同一份私有 `EnterpriseKnowledge` 形成的
公共 facade。它只暴露业务 vocabulary、有类型 `Request` 与
`lower(Request) -> Query`；**不暴露**任何物理模型细节（表名、列名、alias、
join 路径/映射、SQL 片段或模板）。lowering 完全由私有知识捕获，不维护第二份
领域知识。

## 公共 API

```telora
import "ent-1/query.telora" {
    Request, lower,
    MeasureId, DimensionId, GrainId, Subject,
    Query,              # 来自 QueryBuilder 公共契约
};

type Request = { measures: Array({id, subject, input}),
                 dimensions: Array({id, subject, input}) };

lower : Fn(Request) -> Query
Query = { sql: String, bindings: Array(Val) }
```

## 业务词汇

- `MeasureId`（指标）：`OrdersCreated`、`DeliveredPackages`、`UnitsShipped`。
- `DimensionId`（维度）：`OrderMonth`、`CustomerTier`、`OriginRegion`、
  `CarrierName`、`ServiceName`、`ProductCategory`、`DeliveryException`。
- `GrainId`（粒度）：`Order`、`Package`、`PackageItem`。
- `Subject`（授权主体）：`Analyst`。

业务定义消歧：

- `OriginRegion` = 发货仓库（发货来源）所属地区，即订单从哪个仓库发出、该仓库
  位于哪个地区。
- `ProductCategory` 是获准能力；其唯一可请求粒度是 `Order`，但该粒度下目标仅经
  粒度扩张可达，因此按 `Order` 请求会被拒绝（unsafe-path）。它不是“没有能力”，
  而是“能力存在但当前粒度不可达”。
- `DeliveryException` 是封闭维度词汇成员，但**没有**获准能力；任何请求都会
  失败（missing-dimension）。

能力身份全部是有类型的封闭 enum，不使用 String/`Any` 充当语义身份。

## Request 形状与组合规则

- 每个请求项是 `{id, subject, input}`：能力 id + 授权主体 + 粒度输入。
- 请求只表达「要什么」，不携带物理表达式、mapping、Plan 节点或 SQL。
- **粒度规则**：一次计划的所有请求必须共享同一个 `input`（粒度）；每个请求的
  粒度必须等于其能力定义的粒度。
- **能力规则**：请求的指标/维度必须存在于目录；`subject` 必须在能力授权主体
  列表中。
- **可达性规则**（业务语义，不涉及物理映射）：能力目标实体在当前粒度下必须
  安全可达；目标只能经粒度扩张到达、缺失或超出深度时请求失败。
  - `ProductCategory` 是获准能力，但按 `Order` 粒度请求会失败（目标仅经粒度
    扩张可达）。
  - `DeliveryException` 属于封闭维度词汇，但没有获准能力，任何请求都会失败。

## 可用能力概览

| 能力 | 类型 | 请求粒度 | 状态 |
| --- | --- | --- | --- |
| `OrdersCreated` | 指标 | `Order` | 可用 |
| `DeliveredPackages` | 指标 | `Package` | 可用 |
| `UnitsShipped` | 指标 | `PackageItem` | 可用 |
| `OrderMonth` | 维度 | `Order` | 可用 |
| `CustomerTier` | 维度 | `Order` | 可用 |
| `OriginRegion` | 维度 | `Order` | 可用（发货仓库所属地区） |
| `CarrierName` | 维度 | `Order` | 可用 |
| `ServiceName` | 维度 | `Order` | 可用 |
| `ProductCategory` | 维度 | `Order` | 不可用（unsafe-path） |
| `DeliveryException` | 维度 | — | 不可用（无获准能力） |

## Lowering 保证

- `lower(request)` 依次完成能力解析与授权、粒度校验、基础数据源解析、安全路径
  选择、Plan 组装、结构验证与 profile 覆盖检查，然后发布规范 Plan，再经
  QueryBuilder 的 SQLite transform 得到 `Query`。
- **确定性**：同一 Request 两次 `lower` 得到逐字节相同的 `sql` 与相同顺序的
  `bindings`。
- **绑定值**：所有动态/常量参数作为有类型 `Val` 进入 `bindings`；SQL 中只有
  `?` 占位符与双引号标识符，不承担字符串 escape。
- **revision**：生成的 Plan 携带领域计划 revision 标签
  `logistics-ontology-v1`；它不进入 SQL/Query。
- **原子性**：任一步失败都 `fail!`，不发布部分 Plan 或部分 Query；最终结果原子
  发布。

## 失败语义

失败使用 `fail!`，诊断前缀（详见 `QUERY-DESIGNER-TUTORIAL.md`）：

- `enterprise-knowledge.missing-measure` / `missing-dimension`
- `enterprise-knowledge.unauthorized-measure` / `unauthorized-dimension`
- `enterprise-knowledge.measure-grain-conflict` / `dimension-grain-conflict`
- `enterprise-knowledge.no-base-for-grain`
- `enterprise-knowledge.unsafe-path` / `missing-path` / `truncated-path`
- `enterprise-knowledge.empty-request`
- `enterprise-knowledge.invalid-knowledge` / `structure` / `profile-violation`

公共 API 不返回 Rejection、诊断数组或逐请求 Evidence；需要业务恢复时，调用方
应先通过纯校验/规则判断再调用 `lower`。

## 边界 / 非目标

- 请求不支持 filter / ordering / limit；不支持 OR/NOT/CASE/HAVING/子查询。
- 公共面不暴露 Plan 内部结构（表/列/alias/join）；如需 Plan 级审计，应使用
  私有模型交付物。
- `Query` 的 JSON 表示由使用方 crate 建立（QueryBuilder 不提供 JSON/schema
  边界）；`Query` 与 `Val` 可被 `codec` 稳定编码。
