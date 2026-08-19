# A4 记录：Request 选择、真实验证结果与剩余风险

## Request 选择

### 合法意图（`intent-1/src/request.telora` 中的 `legal_request`）

文字意图：统计已创建订单数，并按订单创建月份、客户等级和发货仓库所属地区分组；
结果必须保留这三个分组维度。

- 指标：`OrdersCreated`（Order 粒度）——对应“已创建订单数”。
- 维度：
  - `OrderMonth`（Order 粒度）——对应“订单创建月份”；
  - `CustomerTier`（Order 粒度）——对应“客户等级”；
  - `OriginRegion`（Order 粒度）——教程/契约已确认该维度定义为“发货仓库（发货来源）
    所属地区”，与“发货仓库所属地区”一一对应。
- 所有项 `subject: 'Analyst`（公共查询面唯一授权主体），`input: 'Order` 共享同一粒度，
  符合 Request 组合规则。三个分组维度全部保留，未用相近业务概念替代。

### 非法意图（`illegal_request`）

文字意图：统计已创建订单数，并按所含商品的商品类别分组。忠实表达为
`OrdersCreated`（Order 粒度）+ `ProductCategory`（Order 粒度）。`ProductCategory`
是公共 `DimensionId` 封闭成员，教程/契约说明按 Order 粒度请求会被拒绝（unsafe-path）。

## 真实验证结果

| 命令 | 结果 |
| --- | --- |
| `./bin/telora run main -C intent-1` | 成功，输出 `Output(String)`：Query 的稳定 JSON（`sql` + `bindings`） |
| `./bin/telora run verify -C intent-1` | 成功，输出 `verify-ok`（两次 lowering 严格相同；sql 非空，满足公共结果契约） |
| `./bin/telora run invalid -C intent-1` | 失败（非零退出），诊断为 `enterprise-knowledge.unsafe-path`，无 Query 发布 |
| `./bin/telora run invalid -C intent-1 --best-effort` | 失败（非零退出），status: error，无 Output effect |
| `./bin/telora check @test/intent.telora -C intent-1` | `status: ok`（公共类型与合法链路检查） |
| `./bin/telora show @bin/main.telora -C intent-1` | 可见 `output: String` 与各 import/def 定义 |

- 合法路径：`lower(legal_request)` 返回完整 `Query`；main 通过
  `codec.encode(Value, query) |> result.unwrap` 后 `json.stringify` 输出稳定 JSON。
- 确定性：同一 Request 多次 `run main` 输出逐字节相同；verify 入口在单进程内重复
  lowering 并断言严格相等。
- 非法路径：`lower(illegal_request)` 原子失败，`invalid.telora` 不发布任何 Output。

## 剩余风险

- `Query` 的 JSON 表示由使用方 crate 建立（QueryBuilder 公共契约不提供 JSON/schema
  边界）；当前 main 的输出形状依赖 `codec` 对 `Query`/`Val` 的稳定编码，若未来公共
  契约变更 Query 结构，main 输出形状需同步复查。
- `bindings` 的取值内容（如月份提取参数）是 lowering 的物理实现产物，本记录不做
  结构推断；verify 只按公共契约断言结果形状与确定性。
- 非法意图仅覆盖本轮两个有界意图之一（ProductCategory）；其它组合不在本任务范围。
