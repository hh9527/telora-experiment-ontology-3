# Ent-1 公共查询契约

本文件描述 `ent-1` 公共查询面（public facade）的稳定契约：业务词汇、Request
形状（含筛选/排序/Top N）、组合/grain/capability 规则、失败语义与 lowering
保证。面向不懂物理模型的查询设计者与下游调用方。

## 公共入口

`ent-1/src/query.telora`（crate 内以 `import "@src/query.telora" as q;` 使用；
下游 crate 以 manifest 依赖名导入，如 `import "ent-1/query.telora" as q;`，
依赖名以 manifest 为准）。

公共导出：

- 业务词汇：`MeasureId`、`DimensionId`、`Subject`、`MeasureInput`、
  `DimensionInput`、`FilterInput`、`FilterOp`、`OrderDirection`、`OrderTarget`
- `Request`：typed 查询请求（measures / dimensions / filters / ordering / limit）
- `lower: Fn(Request) -> Query`

公共查询面从同一份私有 EnterpriseKnowledge 捕获 lowering；调用方不需要、也无法
通过 facade 接触物理模型。facade 不暴露表名、列名、alias、join 路径/mapping、
Plan、SQL 片段或模板。

## 业务词汇

### 指标（MeasureId）

| id | 语义 | grain |
| --- | --- | --- |
| `OrdersCreated` | 创建的订单数 | Order |
| `DeliveredPackages` | 已送达的包裹数 | Package |
| `UnitsShipped` | 已发货的件数 | Package item |

### 维度（DimensionId）

| id | 语义 | 可分组 | 可筛选 |
| --- | --- | --- | --- |
| `OrderMonth` | 订单创建月份 | 是 | 是（`Month(String)`） |
| `CustomerTier` | 客户等级（“客户层级”是同一含义的解释，二者等价） | 是 | 是（`Tier(String)`） |
| `OriginRegion` | 发货仓库所属地区（订单发货仓所在地区；不是承运商取件点、集散中心或其他相近的“发货起点”概念） | 是 | 是（`Region(String)`） |
| `CarrierName` | 承运商名称 | 是 | 否（没有筛选能力） |
| `ServiceName` | 服务水平名称 | 是 | 否（没有筛选能力） |
| `ProductCategory` | 产品类别（能力获准；从 Order grain 只能经 grain 扩张到达，从 Package item grain 可安全到达） | 是 | 是（`Category(String)`） |
| `DeliveryException` | 投递异常（属于封闭词汇，但没有获准能力；公共面不暴露其筛选输入） | 否 | 否 |

### 筛选输入（FilterInput）

封闭枚举，与支持筛选的维度一一对应；`to_val` 会拒绝维度与输入变体不匹配：

- `Month(String)` —— 月份字符串，用于 `OrderMonth`（如 `"2026-07"`）
- `Tier(String)` —— 客户等级字符串，用于 `CustomerTier`（如 `"Gold"`）
- `Region(String)` —— 地区名称字符串，用于 `OriginRegion`（如 `"华东"`）
- `Category(String)` —— 类别名称字符串，用于 `ProductCategory`（如 `"Electronics"`）

### 其他

- `Subject = String`：查询发起者标识，只用于授权与诊断；**不会自动变成筛选
  条件**，也不会写入 SQL/bindings。
- `MeasureInput` / `DimensionInput`：当前均为 `enum { 'All }`。
- `FilterOp = enum { 'Eq, 'Ge, 'Le }`：筛选比较操作。
- `OrderDirection = enum { 'Asc, 'Desc }`：排序方向。
- `OrderTarget = enum { 'Measure(MeasureId), 'Dimension(DimensionId) }`：排序目标，
  只允许引用请求中已有的 measure/dimension。

## Request 形状

```json
{
  "measures": [{ "id": "<MeasureId>", "subject": "<Subject>", "input": "All" }],
  "dimensions": [{ "id": "<DimensionId>", "subject": "<Subject>", "input": "All" }],
  "filters": [
    { "id": "<DimensionId>", "subject": "<Subject>", "input": { "<FilterInput变体>": "<值>" }, "op": "Eq|Ge|Le" }
  ],
  "ordering": [
    { "target": { "Measure": "<MeasureId>" } | { "Dimension": "<DimensionId>" }, "direction": "Asc|Desc" }
  ],
  "limit": 5
}
```

（`limit` 省略或为 `null` 表示不限制；必须是正整数。）

### enum / Option 的 JSON 编码（codec 输出）

- 无载荷 enum 编码为字符串：`"All"`、`"Eq"`、`"Desc"`、`"CarrierName"`。
- 单载荷 enum 编码为对象：`{"Month":"2026-07"}`、`{"Measure":"DeliveredPackages"}`。
- `Option`：`None` 编码为 `null`；`Some(x)` 编码为载荷本身（如 `5`）。
- 结构体编码为对象，键为字段名（codec 按稳定顺序输出）。

## 约束

1. 请求必须至少包含一个 measure；`measures` 的 measure 决定本次查询的 base
   grain，同一请求的所有 measure 必须属于同一 grain，跨 grain 组合直接失败。
2. `measures`、`dimensions`、`filters`、`ordering` 保持声明顺序；结果按该顺序
   确定。
3. 请求只表达“要什么”（id + subject + input/op），不携带任何领域表达式、
   mapping、Plan 节点、SQL 或 QueryBuilder 表达式。
4. `filters` 引用 `DimensionId` + `FilterInput` + `FilterOp`；筛选维度即使不在
   projections/group_by 中也会执行授权并纳入查询范围。同一维度
   可多次筛选（如月份 `Ge 2026-04` 与 `Le 2026-06`），按请求顺序组合。
5. `ordering` 只引用**已经请求**的 measure/dimension；引用未请求目标失败。
6. `limit` 必须是正整数；非正数失败。

## 组合 / grain / capability 规则

- **安全维度**：`OrderMonth`、`CustomerTier`、`OriginRegion`、`CarrierName`、
  `ServiceName` 可与任意已获准指标的 base grain 组合（Order / Package /
  Package item grain 均可）。
- **grain 扩张维度**：`ProductCategory` 从 Order grain 只能经 grain 扩张到达，
  因此与 Order grain 指标组合的请求会失败（分组与筛选都失败）；从 Package item
  grain 可安全组合（无需 grain 扩张）。
- **无能力维度**：`DeliveryException` 属于封闭词汇但没有获准 capability（含筛选
  能力），任何请求引用它都会失败。
- **无筛选能力的维度**：`CarrierName`、`ServiceName` 可以分组，但没有筛选
  能力；对它们提交 `filters` 会失败（缺失筛选能力）。
- **不同 natural grain 的指标不能自动组合**；当前没有预聚合或 allocation
  policy。

## 失败语义

授权、grain、路径、筛选值、排序目标、limit、profile 任何一项失败都会原子失败：
不发布部分 Query、不产生 SQL。失败以 Host 诊断报告；具体诊断文本不构成稳定契约
（不应被下游解析依赖）。查询设计者在提交请求前应先澄清缺失信息、歧义与已知不
支持能力；无效 Request 的 Telora 诊断是修正或归因的依据，**不得通过删除条件
扩大查询范围**（例如去掉 filter 或 limit 来“绕过”失败）。

## Lowering 保证

1. **确定性**：同一 `Request` 每次 `lower` 都产生逐字节相同的 `Query`
   （`sql` 与 `bindings` 顺序都确定）。
2. **参数化**：所有动态值（筛选值、月份/地区/等级/类别、limit N）进入
   `bindings`；`Query.sql` 中只有标识符、关键字、运算符与 `?` 占位符，
   `bindings` 顺序与 `?` 出现顺序一致。
3. **Query 形状**：`Query = { sql: String, bindings: Array(Val) }`，
   `Val = 'String / 'Int / 'Float / 'Bool`（不含 Bytes）。
4. **结果可编码**：`Query` 可以通过 codec 编码为 `Value` 并由 JSON 文本化。
5. **不泄漏物理模型**：facade 的类型、签名与文档不包含表名、列名、alias、join
   路径/mapping、SQL 片段或模板。
