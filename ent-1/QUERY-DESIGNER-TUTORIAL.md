# Ent-1 查询设计者教程

本教程面向不了解物理模型的查询设计者：如何用公共查询面（public facade）构造
业务查询并取得确定性的只读 Query。你只需要理解业务词汇（指标/维度/筛选/排序/
Top N）与组合规则，不需要知道任何底层数据组织。

## 1. 导入 facade

```telora
# ent-1 crate 内
import "@src/query.telora" as q;

# 下游 crate（依赖名以 manifest 为准）
import "ent-1/query.telora" as q;
```

公共 API：业务词汇类型、`Request` 与 `lower(Request) -> Query`。

## 2. 构造 Request

`Request` 是“要什么”的声明：

- `measures`：至少一个指标，决定统计口径与 base grain；
- `dimensions`：零个或多个维度，决定分组；
- `filters`：零个或多个范围筛选（`Eq/Ge/Le`），值用领域 `FilterInput` 表达；
- `ordering`：零个或多个排序项，**只引用请求中已有的 measure/dimension**；
- `limit`：可选正整数 Top N（`'None` 表示不限制）。

```telora
def my_request: q.Request = {
    measures: [
        {id: 'OrdersCreated, subject: "analyst@example.com", input: 'All},
    ],
    dimensions: [
        {id: 'OrderMonth, subject: "analyst@example.com", input: 'All},
        {id: 'CarrierName, subject: "analyst@example.com", input: 'All},
    ],
    filters: [],
    ordering: [],
    limit: 'None,
};
```

- `subject` 是查询发起者，只用于授权与诊断；**它不会自动变成筛选条件**，也不会
  写入 SQL/bindings。要做范围筛选，请用 `filters`。
- 所有数组保持声明顺序，结果顺序确定。
- 请求只表达“要什么”，不携带任何领域表达式、SQL 或物理细节。

## 3. 取得 Query

```telora
def query = q.lower(my_request);   # {sql: String, bindings: Array(Val)}
```

`lower` 返回确定性只读 Query：同一个 `Request` 每次得到逐字节相同的 `sql` 与
相同顺序的 `bindings`；所有动态值（月份、地区、等级、类别、limit N）都在
`bindings` 里，`sql` 中不内联任何业务值。

## 4. 选择指标：base grain

一次请求的指标决定查询的 base grain：

- `OrdersCreated` —— Order grain；
- `DeliveredPackages` —— Package grain；
- `UnitsShipped` —— Package item grain。

**同一请求的多个指标必须属于同一 grain**；跨 grain 的指标组合会失败。

## 5. 选择维度与筛选：组合规则

- 安全维度（可与任意已获准指标的 grain 组合）：
  `OrderMonth`、`CustomerTier`、`OriginRegion`、`CarrierName`、`ServiceName`。
- `ProductCategory` 从 Order grain 只能经 grain 扩张到达：与 Order grain 指标
  组合会失败；从 Package item grain 可安全组合。
- `DeliveryException` 属于封闭词汇但没有获准能力：任何请求引用它都会失败。
- 可筛选维度：`OrderMonth`（`Month(String)`）、`CustomerTier`（`Tier(String)`）、
  `OriginRegion`（`Region(String)`）、`ProductCategory`（`Category(String)`）。
  `CarrierName`、`ServiceName` 没有筛选能力，不能出现在 `filters`。

语义说明（不需要猜测，契约已固定）：

- `OriginRegion` 就是**发货仓库所属地区**（订单发货仓所在地区）；它不是承运商
  取件点、集散中心或其他相近的“发货起点”概念。
- `CustomerTier` 就是**客户等级**；“客户层级”只是同一含义的解释，二者等价。

## 6. 代表性场景

以下三个场景是 Host 认可的代表性业务查询，可直接按此构造（Telora 写法与
codec/JSON 编码都给出）。

### 场景 1：`DeliveredPackages` + 月份 2026-07 + 地区 华东 + `CarrierName` Top 5

```telora
def s1: q.Request = {
    measures: [{id: 'DeliveredPackages, subject: "designer@example.com", input: 'All}],
    dimensions: [{id: 'CarrierName, subject: "designer@example.com", input: 'All}],
    filters: [
        {id: 'OrderMonth, subject: "designer@example.com", input: 'Month("2026-07"), op: 'Eq},
        {id: 'OriginRegion, subject: "designer@example.com", input: 'Region("华东"), op: 'Eq},
    ],
    ordering: [
        {target: 'Measure('DeliveredPackages), direction: 'Desc},
        {target: 'Dimension('CarrierName), direction: 'Asc},
    ],
    limit: 'Some(5),
};
```

JSON（codec 编码）：

```json
{"dimensions":[{"id":"CarrierName","input":"All","subject":"designer@example.com"}],"filters":[{"id":"OrderMonth","input":{"Month":"2026-07"},"op":"Eq","subject":"designer@example.com"},{"id":"OriginRegion","input":{"Region":"华东"},"op":"Eq","subject":"designer@example.com"}],"limit":5,"measures":[{"id":"DeliveredPackages","input":"All","subject":"designer@example.com"}],"ordering":[{"direction":"Desc","target":{"Measure":"DeliveredPackages"}},{"direction":"Asc","target":{"Dimension":"CarrierName"}}]}
```

### 场景 2：`UnitsShipped` + 月份 2026-04..2026-06 + 等级 Gold + `ServiceName` Top 3

```telora
def s2: q.Request = {
    measures: [{id: 'UnitsShipped, subject: "designer@example.com", input: 'All}],
    dimensions: [{id: 'ServiceName, subject: "designer@example.com", input: 'All}],
    filters: [
        {id: 'OrderMonth, subject: "designer@example.com", input: 'Month("2026-04"), op: 'Ge},
        {id: 'OrderMonth, subject: "designer@example.com", input: 'Month("2026-06"), op: 'Le},
        {id: 'CustomerTier, subject: "designer@example.com", input: 'Tier("Gold"), op: 'Eq},
    ],
    ordering: [
        {target: 'Measure('UnitsShipped), direction: 'Desc},
        {target: 'Dimension('ServiceName), direction: 'Asc},
    ],
    limit: 'Some(3),
};
```

JSON（codec 编码）：

```json
{"dimensions":[{"id":"ServiceName","input":"All","subject":"designer@example.com"}],"filters":[{"id":"OrderMonth","input":{"Month":"2026-04"},"op":"Ge","subject":"designer@example.com"},{"id":"OrderMonth","input":{"Month":"2026-06"},"op":"Le","subject":"designer@example.com"},{"id":"CustomerTier","input":{"Tier":"Gold"},"op":"Eq","subject":"designer@example.com"}],"limit":3,"measures":[{"id":"UnitsShipped","input":"All","subject":"designer@example.com"}],"ordering":[{"direction":"Desc","target":{"Measure":"UnitsShipped"}},{"direction":"Asc","target":{"Dimension":"ServiceName"}}]}
```

### 场景 3：`UnitsShipped` + 月份 2026-07 + `ProductCategory` Top 10

```telora
def s3: q.Request = {
    measures: [{id: 'UnitsShipped, subject: "designer@example.com", input: 'All}],
    dimensions: [{id: 'ProductCategory, subject: "designer@example.com", input: 'All}],
    filters: [
        {id: 'OrderMonth, subject: "designer@example.com", input: 'Month("2026-07"), op: 'Eq},
    ],
    ordering: [
        {target: 'Measure('UnitsShipped), direction: 'Desc},
        {target: 'Dimension('ProductCategory), direction: 'Asc},
    ],
    limit: 'Some(10),
};
```

JSON（codec 编码）：

```json
{"dimensions":[{"id":"ProductCategory","input":"All","subject":"designer@example.com"}],"filters":[{"id":"OrderMonth","input":{"Month":"2026-07"},"op":"Eq","subject":"designer@example.com"}],"limit":10,"measures":[{"id":"UnitsShipped","input":"All","subject":"designer@example.com"}],"ordering":[{"direction":"Desc","target":{"Measure":"UnitsShipped"}},{"direction":"Asc","target":{"Dimension":"ProductCategory"}}]}
```

## 7. 结果可编码

`Query` 可以直接用 codec 编码为 `Value` 并 JSON 文本化，便于缓存、日志或下游
消费：

```telora
import "std/codec" as codec;
import "std/value" { Value };
import "std/result" as result;
import "std/json" as json;

let encoded = codec.encode(Value, q.lower(s1)) |> result.unwrap;
let text = json.stringify(encoded);   # JSON 文本
```

## 8. 失败语义与澄清

`lower` 对任何非法请求原子失败：不发布部分 Query、不产生 SQL。常见失败原因：

- 引用了没有获准能力的维度（如 `DeliveryException`），或对没有筛选能力的维度
  提交 `filters`（如 `CarrierName`）；
- 维度只能经 grain 扩张到达（如从 Order grain 请求或筛选 `ProductCategory`）；
- 同一请求包含不同 grain 的指标；
- 筛选输入与维度变体不匹配（如 `OrderMonth` 用了 `Tier`）；
- `limit` 非正数，或 `ordering` 引用了未请求的目标。

**先澄清，再提交。** 缺失信息、歧义与已知不支持能力应由查询设计者先澄清；若
提交无效 Request，`lower` 的 Telora 诊断是修正或归因的依据。**不要通过删除
条件扩大查询范围**（例如去掉 filter 或 limit 来“绕过”失败）。失败以 Host 诊断
报告；不要把具体诊断文本当作稳定契约。

## 边界

- 公共查询面不暴露物理模型：你看不到表名、列名、alias、join 路径或 SQL 模板；
  只需使用上面的业务词汇。
- 指标/维度/筛选输入/排序方向的 id 是封闭业务词汇；不要用自由文本猜测 id。
