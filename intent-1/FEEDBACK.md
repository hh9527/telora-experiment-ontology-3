# 公共查询面检视意见（ent-1-query-surface-feedback.a4）

依据 `intent-1/INTENT.md` 的私有文字意图，检视已放行的公共查询面
（`ent-1/QUERY-DESIGNER-TUTORIAL.md`、`ent-1/PUBLIC-QUERY-CONTRACT.md`）。
以下只记录公共词汇、契约、教程或 lowering 的具体问题，不评价不可见实现。

## 最新修订版复核结果（2026-08-21，新增筛选/排序/Top N）

本轮 `ent-1-query-surface.a3` 更新：`Request` 扩展为
`measures` / `dimensions` / `filters` / `ordering` / `limit` 五字段，新增
`FilterInput`、`FilterOp`、`OrderDirection`、`OrderTarget` 词汇，教程加入三个
Host 认可的代表性场景（Telora 写法与 codec/JSON 编码均给出）。

### 合法意图复核

文字意图：统计已创建订单数，并按订单创建月份、客户等级和发货仓库所属地区分组。

- `OrdersCreated`（Order grain 指标）与 `OrderMonth`、`CustomerTier`、
  `OriginRegion` 三个维度均有精确对应的公共词汇；三者均为安全维度，可与
  Order grain 任意组合。
- 语义继续固定：`OriginRegion` 即「发货仓库所属地区（订单发货仓所在地区；
  不是承运商取件点、集散中心或其他相近的“发货起点”概念）」；`CustomerTier`
  即「客户等级（“客户层级”是同一含义的解释，二者等价）」。两处送审版歧义
  在最新修订版中均无回退。
- 新请求形状要求显式携带 `filters` / `ordering` / `limit`。本意图不含筛选、
  排序或 Top N；教程第 2 节给出空数组与 `'None` 的写法
  （`filters: []`、`ordering: []`、`limit: 'None`），表达无歧义。
- 契约明确 `Subject` 只用于授权与诊断、不会自动变成筛选条件，也消除了
  “subject 是否隐式过滤”的歧义。

### 非法意图复核

文字意图：统计已创建订单数，并按所含商品的商品类别分组。

- `OrdersCreated` + `ProductCategory` 可被忠实表达：契约明确 `ProductCategory`
  从 Order grain 只能经 grain 扩张到达，与 Order grain 指标组合会失败
  （分组与筛选都失败）；从 Package item grain 才可安全组合。该组合可被
  如实提交并原子拒绝、不产生 Query，符合意图「保留请求并验证它被拒绝且
  没有 Query」。
- 修订版新增「先澄清，再提交」「不得通过删除条件扩大查询范围」的失败语义
  说明，与本意图「不删除商品类别、不改变指标 grain、不换成其他维度」的处理
  方式一致。

### 新增能力相关确认（非问题）

- `FilterInput` 与可筛选维度一一对应（`Month`/`Tier`/`Region`/`Category`），
  `to_val` 会拒绝维度与输入变体不匹配；`CarrierName`、`ServiceName` 可分组
  但无筛选能力。
- `ordering` 只引用请求中已有的 measure/dimension；`limit` 必须是正整数。
- 三个代表性场景与契约规则逐条一致（filters 可不投影仍纳入查询范围、同维度
  可多次筛选、OrderMonth 用 `Month(String)`、枚举/`Option` 的 codec JSON
  编码等）。以上能力与本轮两个意图正交，不影响合法/非法路径的表达。

## 送审版问题（已被修订版解决）

1. **`OriginRegion` 与「发货仓库所属地区」的语义对应不明确**：送审版把
   `OriginRegion` 标注为「发货起点地区」，无法确定它是否等于合法意图要求的
   「发货仓库所属地区」，与意图「不能用其他相近业务概念替代」冲突。
2. **`CustomerTier` 与「客户等级」措辞差异**：送审版标注为「客户层级」，未明确
   即「客户等级」。

修订版及最新修订版均已把 `OriginRegion` 明确定义为「发货仓库所属地区」、
`CustomerTier` 明确定义为「客户等级」，两处问题已解决，无回退。

## 结论

最新修订版公共查询面已消除送审版的两处语义歧义，合法意图的三个分组维度
（`OrderMonth`、`CustomerTier`、`OriginRegion`）与指标 `OrdersCreated` 均有
精确对应的公共词汇，非法意图（`OrdersCreated` + `ProductCategory`）也可被
忠实表达并预期原子拒绝；新增的筛选/排序/Top N 能力与本轮意图正交，契约与
教程保持一致。当前无未解决的公共词汇、契约、教程或 lowering 问题。
