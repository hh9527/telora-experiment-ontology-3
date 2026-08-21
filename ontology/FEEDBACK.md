# 参数化业务查询增量反馈

> 本文件已经完全替换此前的反馈。本轮唯一目标是为 eDSL 增加
> `filter`、`ordering` 和 `limit` 的受控业务表达与 lowering。
> 不要只复验此前的 computed dimension、multi-grain 或 String revision；
> 若 `QueryRequest` 仍不能表达参数化筛选、排序和 Top N，则本轮尚未完成。

现有 `QueryRequest` 只能表达 measure、dimension 与 `input = All`，`make_query_creator` 固定生成 `filter = None`、空 ordering 和空 limit。这使公共查询面只能生成全量分组清单，无法覆盖真实查询中的特定范围和 Top N。

请在不泄漏物理模型、不接受任意 QueryBuilder 表达式或 SQL 的前提下，为 eDSL 增加以下最小标准能力。

## 1. 有类型维度筛选

- Query 请求可以携带有序筛选项，每项引用封闭的 DimensionId、Subject、领域自定义的有类型输入，以及标准比较操作 `Eq`、`Ge`、`Le`。
- 企业知识负责把领域输入转换为 QueryBuilder `Val`；查询方不能提交 `qb.Expr`、列、表、alias、SQL 或 mapping。
- 多个筛选按请求顺序用标准 `And` 组合；值必须成为 `qb.Literal`，最终进入 bindings。
- 筛选维度可以不在 projections/group_by 中，但仍须执行能力授权，并为其实体选择 grain-safe 路径、合并必要 join。
- 同一维度可以出现多个筛选（例如月份 `Ge 2026-04` 与 `Le 2026-06`），顺序必须稳定。

## 2. 有序排序

- 请求可按已经请求的 measure 或 dimension 指定 `Asc` / `Desc`。
- eDSL 只能从已解析的 measure/dimension 规范表达式构造 `qb.OrderItem`，不得接受任意表达式。
- 支持按指标降序后按维度升序作为稳定 tie-break。
- 引用未请求的排序目标必须给出诊断并原子失败。

## 3. Top N

- 请求可携带可选正整数 limit；非正数必须诊断失败。
- 成功时写入 `Plan.limit`，由 QueryBuilder 生成 `LIMIT ?` 并把 N 放入 bindings。

## 4. 语义与验证

- 请求仍只表达业务意图，不得携带物理信息或 Plan 节点。
- 授权、缺失能力、grain 冲突、筛选值不合法、排序目标不合法、limit 不合法或 profile 不接受时均原子失败，不发布部分 Plan/Query。
- `PlanProfile` 必须明确接受实际使用的 `filter/order_by/limit` 及 `Eq/Ge/Le/And`。
- 保持确定性：相同请求逐字节产生相同 SQL 和 bindings，bindings 顺序与 SQL 中 `?` 顺序一致。
- 教程、公共契约、main/verify/invalid 和测试应覆盖：等值筛选、范围筛选、未投影筛选维度、指标降序 + 维度升序、Top N、非空 bindings 及失败原子性。

QueryBuilder 已有 `filter`、`ordering`、`limit`、`Literal` 和上述标准算子。本轮优先在 eDSL 的 Request/Knowledge/lowering 层完成受控映射；只有确有公共算子缺口时才反馈上游，不要另造 Plan 或渲染器。
