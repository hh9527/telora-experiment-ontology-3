# 本体 eDSL 设计契约

本 eDSL 让企业以结构化方式表达 `EnterpriseKnowledge`，并提供稳定的 lowering：

```text
make_query_creator:
  EnterpriseKnowledge -> Fn(Request) -> Plan
```

`Plan`、标准算子和 `PlanProfile` 由 QueryBuilder 所有。eDSL 不定义替代 Plan，不负责
`Plan -> Query`，也不包含任何企业领域事实。

## EnterpriseKnowledge

EnterpriseKnowledge 至少表达：

- revision 与封闭的领域 vocabulary；
- 实体、属性、指标、维度和关系等 ontology 事实；
- 能力目录、授权和每项能力所需的有类型输入；
- 基础数据源、结构化物理表达式与结构化关系 mapping；
- 安全关系与会扩张 grain 的关系；
- 此知识接受的 QueryBuilder `PlanProfile`；
- 将能力、关系和物理事实 lowering 为标准 Plan 算子的规则。

企业知识不得使用预渲染 SQL、任意 builder 回调或 String 反查来绕过公共规则。
`PlanProfile` 只收窄标准算子能力，不能改变任何算子的语义。

## Request 与 lowering

Request 是查询方提交的有类型意图。参考形状为：

```telora
type Request(Id, Subject, Input) = struct {
  id: Id,
  subject: Subject,
  input: Input,
};

type QueryRequest(MeasureId, DimensionId, Subject, MeasureInput, DimensionInput) = struct {
  measures: Array(Request(MeasureId, Subject, MeasureInput)),
  dimensions: Array(Request(DimensionId, Subject, DimensionInput)),
};
```

请求只能表达“要什么”，不能携带领域表达式、mapping、Plan 节点或 SQL。measure 与
dimension 保持原始顺序；原始 subject 必须进入相关诊断。

`make_query_creator(knowledge)(request)` 必须：

1. 独立解析并授权每项能力；
2. 验证指标 grain 兼容性；
3. 推导维度所需实体并选择安全关系；
4. 组装覆盖所有请求且没有额外请求的标准 Plan；
5. 验证 Plan 只使用 `knowledge.plan_profile` 接受的能力；
6. 成功时发布 Plan，失败时通过 `fail!` 记录诊断且不发布部分 Plan。

公共 API 不返回 Rejection、诊断数组或逐请求 Evidence。可恢复诊断由 Telora Host
机制负责；eDSL 作者只按正常表达式语义使用 `fail!`。

## 关系与路径

关系目录区分 grain-safe 和 fan-out。每条关系保留有类型端点和企业拥有的结构化
mapping。安全路径选择遵循：最短边数优先，同长度按目录索引序列字典序最小；多个
目标按请求顺序合并，共享边只保留首次出现。遍历必须对有向环稳定，最大深度为八。

完整可达性使用 safe 与 fan-out 的并集。目标恰好分类为 safe、fan-out-only 或
missing。恰好八条边可接受；只有仍存在未访问后继且边界可能隐藏可达性时才产生
truncation。任何不安全、缺失、未授权、grain 冲突或 profile 越界都阻止 Plan 发布。

## Plan 组装边界

eDSL 使用 QueryBuilder 的公共标准算子构造 Plan。Plan 至少保留 knowledge revision、
基础数据源、有序 measure/dimension 投影、与维度一致的 grouping，以及按规范顺序
选择的关系和 mapping。每个成功请求恰有一个对应投影，每个非基点需求都有路径覆盖。

企业不能提供一个“最终 Plan builder”替 eDSL 手工完成组装；eDSL 也不能根据 String
label 反查领域数据。Plan 完成后必须调用 QueryBuilder 的 profile 验证。eDSL 的成功
示例可以继续调用 QueryBuilder 的 SQLite transform 展示端到端 Query，但该转换不是
eDSL API，也不得把 SQLite 细节写回方言中立 Plan 的语义。

## 公共边界

公共 family、EnterpriseKnowledge 构造能力和 query creator 必须保持精确类型，不得
使用 `Any`、`Dyn` 或 native 声明。API 名称、模块布局、图算法和内部状态表示由实现
选择，但公共教程必须足以让企业作者在不读源码的情况下建立知识。
