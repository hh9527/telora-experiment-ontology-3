# A4 NOTES：Request 选择、真实验证结果与剩余风险

## Request 选择

依据 `intent-1/INTENT.md` 的两个有界意图，仅使用已批准的公共查询面
（`ent-1/QUERY-DESIGNER-TUTORIAL.md`、`ent-1/PUBLIC-QUERY-CONTRACT.md`）中的
公共词汇构造 typed Request，并只调用公共 `lower`。最新修订版（放行版）的
`Request` 为 `measures` / `dimensions` / `filters` / `ordering` / `limit`
五字段；本意图不含筛选、排序或 Top N，故按教程第 2 节写法使用
`filters: []`、`ordering: []`、`limit: 'None`。

### 合法意图：`src/request.telora` 的 `legal_request`

文字意图：统计已创建订单数，并按订单创建月份、客户等级和发货仓库所属地区分组。

| 意图成分 | 公共词汇 | 说明 |
| --- | --- | --- |
| 已创建订单数 | `OrdersCreated`（Order grain 指标） | 指标决定 base grain = Order |
| 订单创建月份 | `OrderMonth`（安全维度） | 与 Order grain 可组合 |
| 客户等级 | `CustomerTier`（安全维度） | 契约明确即「客户等级」 |
| 发货仓库所属地区 | `OriginRegion`（安全维度） | 契约明确即「发货仓库所属地区」 |

`subject` 使用 `"analyst@example.com"`（请求者标识，只用于授权与诊断，不写入
SQL/bindings），各条目 `input` 为 `'All`。

### 非法意图：`src/request.telora` 的 `illegal_request`

文字意图：统计已创建订单数，并按所含商品的商品类别分组。

| 意图成分 | 公共词汇 | 说明 |
| --- | --- | --- |
| 已创建订单数 | `OrdersCreated`（Order grain 指标） | 保持指标 grain 不变 |
| 所含商品的商品类别 | `ProductCategory`（维度） | 忠实保留，不替换 |

契约明确 `ProductCategory` 从 Order grain 只能经 grain 扩张到达，与 Order grain
指标组合会失败（分组与筛选都失败）；从 Package item grain 才可安全组合。
按要求不删除商品类别、不改变指标 grain、不换成其他维度，保留请求并验证它被
拒绝且没有 Query。

## 真实验证结果（./bin/telora，-C intent-1，2026-08-21）

### `run main`（合法路径）

成功，输出合法 Query（公共 `lower` 的完整结果，同时保留 SQL 与有序 bindings）：

```json
{"bindings":[{"Int":1},{"Int":7},{"Int":1},{"Int":7}],"sql":"SELECT count(o.id) AS orders_created, substr(o.created_at, ?, ?) AS order_month, cu.tier AS customer_tier, rg.name AS origin_region FROM orders AS o INNER JOIN customers AS cu ON o.customer_id = cu.id INNER JOIN warehouses AS wh ON o.warehouse_id = wh.id INNER JOIN regions AS rg ON wh.region_id = rg.id GROUP BY substr(o.created_at, ?, ?), cu.tier, rg.name"}
```

- `sql`：全参数化（`?` 占位符），无内联业务值；
- `bindings`（有序）：`[1, 7, 1, 7]`，与 `sql` 中 `?` 的出现顺序一致（SELECT
  与 GROUP BY 中 `substr(o.created_at, ?, ?)` 各占两个占位符）；
- 分组维度保留三个：订单创建月份、客户等级、发货仓库所属地区。

### `run verify`

输出 `verify ok`：

1. 确定性：同一 `legal_request` 重复 lower 的 `sql` 与 `bindings` 逐字节/逐元素
   严格相同；
2. Query 形状：bindings 每个元素经穷尽 match 均为 Val 的四个 variant
   （`'String/'Int/'Float/'Bool`，不含 Bytes）之一；
3. 结果可编码：`codec.encode(Value, query)` 成功。

### `run invalid`（普通模式）

失败（预期），退出码非零，无 Query 输出：

```text
error: ontology/lib.telora:568:23: dimension grain conflict: only reachable via fan-out relations
```

### `run invalid --best-effort`

失败（预期）：`telora.run/v1` 诊断 `severity=error`，`summary status=error`，
无 Entry effect / 无 output。普通模式与 best-effort 模式同属失败。

### `check @test/intent.telora`

`telora.check/v1` summary `status=ok`（公共类型与合法链路检查通过）。

### `query exports @bin/main.telora`

```json
{"authority":"authoritative","module":"@bin/main.telora","name":"output","record":"export","schema":"telora.query/v1","type":"String"}
```

导出 `output: String`（authoritative）。

## 公共模型反馈

对放行版公共查询面的检视意见与复核结论见 `intent-1/FEEDBACK.md`：送审版两处
语义歧义（`OriginRegion` 是否即「发货仓库所属地区」、「CustomerTier` 是否即
「客户等级」）已在修订版明确固定且无回退；新增筛选/排序/Top N 能力与本轮两个
意图正交；放行版无未解决的词汇/契约/教程/lowering 问题。

## 剩余风险

- 非法路径失败诊断文本（如 `dimension grain conflict: only reachable via
  fan-out relations`）来自 Host 诊断，公共契约明确不构成稳定契约，未作为验证
  依据解析；非法路径的验收只依赖「run 非零退出 + 无 Query 输出」，两种模式
  同属失败。
- 合法 Query 的 SQL 由 `run main` 输出展示并原样记录；契约保证确定性（同一
  Request 每次 lower 逐字节相同），`verify` 已对重复 lower 做严格相同验证。
- 未读取企业私有 DOMAIN、A3 私有模型源码、QueryBuilder/ontology 文档或源码；
  未通过错误消息或 `query` 反向恢复表、列、alias、join mapping 或 SQL 模板。
  上述 SQL 文本仅作为公共 `lower` 的输出产物原样记录，不对其作物理模型推断。
