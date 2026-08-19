# A3 对上游公共交付的反馈（最终建模后，第三轮上游）

按 QueryBuilder / eDSL 分区记录在建立 Ent-1 私有 EnterpriseKnowledge 与合法/
非法链路验证过程中遇到的具体使用摩擦。第三轮上游已解决 revision 类型与
required-entity 两项，以下为实际建模验证后的使用反馈。

## QueryBuilder

### Q1. 数组原子字面量在断言中需要显式类型标注

- 现象：`ops.joins == ['Inner]`、`ops.aggregates == ['Count]`、
  `query.bindings == ['Int(1), 'Int(7), ...]` 等数组字面量与
  `OperatorSet`/`Query.bindings` 比较时，前端报
  `cannot unify JoinKind with 'Inner`，元素枚举类型没有从等号左侧获得。
- 处理：`verify.telora` 中显式标注
  `let expected_joins: Array(JoinKind) = ['Inner];` 后再比较。
- 建议：在教程的“验证 operators(plan) <= profile”示例中展示显式标注写法，或
  提供 `OperatorSet`/`Query` 的逐字段断言辅助，减少企业侧试错。

### Q2. Plan/Query 的稳定 JSON 表示需使用方自行建立

- 现象：公共契约明确“不为 Plan/Query 提供 JSON/schema 边界”。`main.telora`
  需要以 String 输出规范 Plan 与 Query 时，本 crate 自行用
  `codec.encode(Value, plan)` + `json.stringify` 建立 JSON 边界；工作正常，
  Plan/Query 全部字段 codec 可编码（无 Bytes/函数）。
- 建议：无阻塞；若教程能给出一个推荐编码示例（`codec.encode(Value, plan)` +
  `json.stringify`），可减少企业侧重复摸索。

### Q3. 常量参数统一进入 bindings

- 现象：`substr(created_at, 1, 7)` 的常量 `1`、`7` 在投影与 grouping 各出现
  一次，SQL 中共四个 `?`，bindings 为 `['Int(1), 'Int(7), 'Int(1), 'Int(7)]`。
  确定性成立，但与“静态表达式”直觉略有出入（无动态用户数据时也有占位符）。
- 建议：无阻塞；若希望常量内联为 SQL 字面量，需要新增“非绑定常量”表达式，
  会改变 binding 顺序契约，非必要。

### Q4.（已解决）`Plan.revision` 由 Int 改为 String

- 第三轮已把 `Plan.revision` 改为稳定、可读的 String 标签，由 Plan 逐字节保留、
  不参与 profile/validation、不进入 SQL/Query。领域
  `logistics-ontology-v1` 直接作为 `EnterpriseKnowledge.revision` 写入 Plan，
  无需再维护 Int↔String 映射。✅

## eDSL

### E1. 验证/测试模块中的断言绑定必须具名

- 现象：`let _ = require(cond, label);` 写法被前端拒绝（`binding has no name`），
  `verify.telora` 与 `tests/logistics.telora` 中所有断言绑定必须逐个具名
  （如 `let c_proj0 = require(...)`）。
- 建议：在教程“失败类别/验证”一节给出具名断言示例；`_` 通配绑定当前不可用于
  模块级 `let`，对企业验证模块有实际影响。

### E2.（已解决）`requires: Array(EntityId)` 结构化声明

- 第三轮 `MetricDef`/`DimensionDef` 新增 `requires`；lowering 为能力自身实体与
  每个 required entity 分别选择安全路径并按稳定顺序合并。本模型
  `DeliveredPackages` 声明 `requires: ['Order]`、
  `UnitsShipped` 声明 `requires: ['Package, 'Order]`，满足 DOMAIN 的“要求
  Order/Package”语义。✅

### E3. 封闭 Dimension vocabulary 与无能力维度的建模约定

- 现象：领域要求 `DeliveryException` 属于封闭 dimension vocabulary 但没有获准
  capability。为使非法请求可类型化并在运行时被目录校验拒绝，模型把
  `'DeliveryException` 纳入封闭 `DimensionId` enum、但不放入
  `dimensions` 目录，得到 `enterprise-knowledge.missing-dimension`。
- 建议：无阻塞；可在教程中提示“封闭词汇可包含无能力成员以支持非法请求的
  类型化构造”。

### E4. ExprNode 属性索引要求共享同一属性目录

- 现象：`attribute_node(attrs, id)` 解析为目录索引，知识与表达式必须共享同一个
  `attrs` 数组。本模型在 `model.telora` 中先定义属性目录再构造所有表达式，
  工作正常；若表达式在另一模块用不同数组构造会得到错误索引。
- 建议：无阻塞；契约中已有说明，保持现状即可。

### E5.（使用注意）`requires` 与路径合并顺序

- 能力实体路径在 required 路径之前、required 按 `requires` 声明顺序。
- 合法场景所有能力 `requires: []`，五个 join 顺序不变；`requires` 只在
  Package/PackageItem grain 指标上使用。
