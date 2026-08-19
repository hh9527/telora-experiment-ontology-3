# Ent-1 查询设计者教程

本教程面向**不掌握物理模型**的查询设计者。你只需要理解业务词汇、Request 形状与
规则，就可以通过公共查询面 `lower(Request) -> Query` 获得可执行的 SQLite Query。
你不需要知道数据存放在哪张表、列名是什么、如何 join。

## 1. 获取查询面

在你的 crate 中（把 ent-1 作为依赖）：

```telora
import "ent-1/query.telora" {
    Request, lower, MeasureId, DimensionId, GrainId, Subject,
};
```

（ent-1 内部模块使用 `import "@src/query.telora" { ... }`。）

`lower` 的签名是 `Fn(Request) -> Query`。`Query` 由 QueryBuilder 公共契约定义：

```text
Query = { sql: String, bindings: Array(Val) }
```

`sql` 是可直接执行的 SQLite SQL，`bindings` 是 SQL 中 `?` 占位符对应的有类型
参数值。同一个 Request 每次 `lower` 都会得到**逐字节相同**的 Query。

## 2. 业务词汇

### 指标（MeasureId）

| 指标 | 可用粒度 | 说明 |
| --- | --- | --- |
| `OrdersCreated` | `Order` | 创建的订单数量 |
| `DeliveredPackages` | `Package` | 已履约包裹数量（依赖订单） |
| `UnitsShipped` | `PackageItem` | 发货的商品条目数量（依赖包裹与订单） |

### 维度（DimensionId）

| 维度 | 请求粒度 | 状态 | 说明 |
| --- | --- | --- | --- |
| `OrderMonth` | `Order` | 可用 | 订单发生的月份 |
| `CustomerTier` | `Order` | 可用 | 客户层级 |
| `OriginRegion` | `Order` | 可用 | **发货仓库（发货来源）所属地区** |
| `CarrierName` | `Order` | 可用 | 承运商名称 |
| `ServiceName` | `Order` | 可用 | 服务水平名称 |
| `ProductCategory` | `Order` | 不可用（unsafe-path） | 商品类目；该粒度下目标仅经粒度扩张可达 |
| `DeliveryException` | — | 不可用（无获准能力） | 封闭词汇成员，不构成能力 |

说明：

- `OriginRegion` 指**发货仓库（发货来源）所属地区**，即订单从哪个仓库发出、
  该仓库位于哪个地区。
- `ProductCategory` 是获准能力，但只能按 `Order` 粒度请求，而该请求会被拒绝
  （目标仅经粒度扩张可达）。
- `DeliveryException` 属于封闭维度词汇，但**没有**获准能力；请求它只会失败。

### 粒度（GrainId）与授权主体（Subject）

- 粒度：`Order`、`Package`、`PackageItem`。一次请求的所有项必须共享同一个粒度。
- 授权主体：`Analyst`。请求中的 `subject` 必须是该能力授权的主体。

## 3. 编写 Request

```telora
let request: Request = {
    measures: [
        {id: 'OrdersCreated, subject: 'Analyst, input: 'Order},
    ],
    dimensions: [
        {id: 'OrderMonth, subject: 'Analyst, input: 'Order},
    ],
};
```

规则：

- `measures` 与 `dimensions` 都是数组；每个请求项是
  `{id, subject, input}`：能力 id + 授权主体 + 粒度输入。
- 请求只表达「要什么」：不要、也不能携带任何物理表达式、映射、Plan 节点或 SQL。
- 所有请求项的 `input` 必须相同（共享粒度），并且必须等于该能力声明的粒度。

## 4. 得到 Query

```telora
let query = lower(request);
# query.sql  : 可执行的 SQLite SQL
# query.bindings : 有类型参数，与 SQL 中占位符一一对应
```

- `lower` 会校验并组装规范 Plan，再确定具体化为 SQLite Query。
- 成功时返回完整 Query；任何失败都原子失败——不会发布部分 Plan 或部分 Query。

## 5. 失败语义

请求不合法时，`lower` 失败（Host 会看到诊断），不会产生 Query：

| 情形 | 诊断前缀 |
| --- | --- |
| 指标/维度不在目录中 | `enterprise-knowledge.missing-measure` / `missing-dimension` |
| subject 未被授权 | `enterprise-knowledge.unauthorized-measure` / `unauthorized-dimension` |
| 粒度不一致或与能力声明不符 | `enterprise-knowledge.measure-grain-conflict` / `dimension-grain-conflict` |
| 粒度没有基础数据源 | `enterprise-knowledge.no-base-for-grain` |
| 能力目标在当前粒度不可达（粒度扩张/缺失/超深） | `enterprise-knowledge.unsafe-path` / `missing-path` / `truncated-path` |
| 请求为空 | `enterprise-knowledge.empty-request` |

常见判断：

- `ProductCategory` 是获准能力，但在 `Order` 粒度下目标只能经粒度扩张到达，
  因此按 `Order` 粒度请求会失败（`unsafe-path`）。
- `DeliveryException` 是封闭词汇成员，但没有获准能力，请求会报
  `missing-dimension`。

## 6. 更多示例

```telora
# 只问一个指标
let only_measure: Request = {
    measures: [{id: 'DeliveredPackages, subject: 'Analyst, input: 'Package}],
    dimensions: [],
};

# 指标 + 一个维度
let measure_and_dim: Request = {
    measures: [{id: 'OrdersCreated, subject: 'Analyst, input: 'Order}],
    dimensions: [{id: 'CustomerTier, subject: 'Analyst, input: 'Order}],
};
```

把上面的 Request 交给 `lower`，你就得到确定、可执行的 Query。祝你查询愉快。
