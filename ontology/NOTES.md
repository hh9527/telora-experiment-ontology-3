# NOTES：设计、验证结果与限制

## 设计

### 模块布局

- `ontology/src/lib.telora`：领域无关的可复用 eDSL。一个文件承载公共类型、
  构造器、BFS 路径搜索与 lowering，避免跨模块泛型参数泄漏。
- `ontology/src/bin/main.telora`：虚构的在线课程平台分析知识 -> Request -> Plan
  -> QueryBuilder SQLite Query 的成功示例（含多 grain 指标、受控计算维度、
  范围 + 等值筛选、指标降序 + 维度升序与 Top N）。
- `ontology/src/bin/verify.telora`：请求覆盖、关系选择（最短路径、共享边合并、
  同长度字典序最小）、多 grain base、受控计算维度、筛选（等值/范围/未投影
  维度/And 组合/bindings）、排序（指标降序 + 维度升序）与 Top N、profile
  覆盖的断言验证。
- `ontology/src/bin/invalid.telora`：非法请求（不兼容 grain 的 measure、
  非正 limit、未请求排序目标、缺失/未授权筛选能力、非法筛选值）失败且不发布
  Plan/Query 的示例。
- `ontology/tests/ontology.telora`：公共类型与契约检查（10 个断言）。

### 关键设计决策

1. **全部目录以类型级 ID 参数化**：`Knowledge` family 带 9 个类型参数
   （MeasureId、DimensionId、EntityId、AttributeId、RelationId、Subject、
   MeasureInput、DimensionInput、FilterInput），与 DESIGN.md 的
   `Request(Id, Subject, Input)` 参考形状一致。目录查找用 `==` 比较类型级 id，
   不使用 String 反查。

2. **多 grain 指标（无单一 base_entity）**：每个 `measure.entity` 是它的 grain
   实体；Order/Package/PackageItem 等不同 grain 的获准指标可以在同一个
   Knowledge 中同时登记。一次请求的 measure 决定该 Plan 的 base grain——所有
   请求 measure 必须位于同一实体，不兼容时 `resolve_base` 以 `fail!` 明确失败
   （消息 "measures have incompatible base entities"）。维度与所需实体从该
   base grain 走安全关系路径。

3. **关系是有向边（left -> right）**：`grain` 分类针对该方向；mapping 是
   结构化等值列对（`AttributePair`），lowering 时生成 `qb.EquiCondition`。
   `join_kind` 由企业显式声明（`'Inner` / `'Left`）。

4. **受控计算维度**：`DimensionExpr` 支持直接属性或 `ComputedExpr`（
   `qb.ScalarFunction` + 属性/`qb.Val` 字面量实参）。lowering 构造
   `qb.ScalarCall`，字面量成为 `qb.Val`/binding；属性实参必须属于维度实体
   （否则 fail!）。函数必须进入 `knowledge.plan_profile.scalar_functions`
   约束，最终由 `qb.validate` 强制。不使用预渲染 SQL，不用 String 反查语义
   对象。示例：`substr(en.created_at, ?, ?)`（按月截断）。

5. **有类型维度筛选**：`QueryRequest.filters` 携带有序筛选项，每项 =
   `DimensionId` + `Subject` + 领域输入 + `FilterOp`（`'Eq`/`'Ge`/`'Le`）。
   企业知识用 `FilterCapability(D, S, FI)`（`authorized` + `to_val`）把领域
   输入转换为 `qb.Val`；查询方不能提交 `qb.Expr`/列/表/alias/SQL/mapping。
   多个筛选按请求顺序用 `qb.scalar_call('And, ...)` 左折叠组合；值成为
   `qb.Literal` 并进入 bindings。筛选维度可以不在 projections/group_by 中，
   但仍须授权、走 grain-safe 路径并合并必要 join（复用 `resolve_dimension`）。
   同一维度可以出现多个筛选（如月份 `Ge 2026-04` 与 `Le 2026-06`），顺序稳定。

6. **排序只引用已请求目标**：`QueryRequest.ordering` 携带
   `OrderTarget`（`'Measure(id)`/`'Dimension(id)`）+ `Asc`/`Desc`。eDSL 只从
   已解析的 measure/dimension 规范表达式构造 `qb.OrderItem`（measure 用
   `measure_expr`，dimension 用 `build_dimension_expr`），不接受任意表达式；
   引用未请求的排序目标以 `fail!` 原子失败。支持"指标降序 + 维度升序"的稳定
   tie-break（按请求顺序写入 `Plan.ordering`）。

7. **Top N**：`QueryRequest.limit: Option(Int)` 只接受正整数；非正数以
   `fail!` 失败。成功时写入 `Plan.limit`，QueryBuilder 生成 `LIMIT ?` 并把 N
   放入 bindings。

8. **BFS 路径选择**：从 base grain 实体出发，逐层扩展（按目录顺序），
   - 安全 BFS（只走 `'GrainSafe` 关系）给出实际路径；
   - 完整 BFS（safe + fan-out 并集）给出可达性分类（safe / fan-out-only /
     missing）；
   - 最短边数优先；逐层 FIFO + 目录顺序扩展保证同长度时目录索引序列字典序
     最小（队列顺序即路径字典序）；
   - 最大深度 8（`max_path_depth`），对环稳定（visited 标记）；
   - 截断（truncation）：深度 8 仍有未访问后继时置位，仅用于信息记录，不阻止
     Plan 发布（深度超过 8 的路径本就不被支持）。

9. **共享边合并**：维度/筛选/排序按请求顺序处理，join 按"边首次出现"去重
   （按关系索引）。`resolve_dimension` 是维度、筛选、排序共用的路径解析器。

10. **授权**：请求的每个 measure/dimension/filter id 必须命中对应能力目录且
    `authorized(subject, input)` 为真；失败时 `fail!` 携带 `req.subject` 与
    `req.id` 作为证据（DESIGN.md 要求原始 subject 进入相关诊断）。

11. **profile 边界**：lowering 完成后调用 `qb.validate(plan, plan_profile)`，
    `'Err` 时 `fail!`（消息带分类前缀）。实际使用的 `filter`/`order_by`/
    `limit` 与 `Eq/Ge/Le/And` 必须被 profile 接受；示例 profile 由
    `standard_profile` 派生（allow_filter/allow_order_by/allow_limit 开启）并把
    `['Substr, 'Eq, 'Ge, 'Le, 'And]` 列入 `scalar_functions`。`allow_distinct`
    只约束 `AggCall.distinct`。

12. **revision 是企业持有的 String**：`Knowledge.revision` 原样进入
    `qb.Plan.revision`（QueryBuilder 契约 #7：不参与验证或具体化）。

13. **`fail!` 而非结果类型**：公共 API 不返回 Rejection / 诊断数组 / Evidence；
    可恢复诊断交给 Telora Host 机制。

## 验证结果

全部通过（本轮实现版本，含 edsl-feedback 参数化业务查询修订）：

```text
./bin/telora run main -C ontology
  -> plan revision=course-analytics-v1 source=en projections=4 joins=1 group_by=2 filter=set ordering=2 limit=10
  -> SELECT sum(en.amount_paid) AS revenue, count(DISTINCT en.learner_id) AS distinct_learners, co.category AS course_category, substr(en.created_at, ?, ?) AS enrollment_month FROM enrollments AS en INNER JOIN courses AS co ON en.course_id = co.course_id WHERE (((substr(en.created_at, ?, ?) >= ?) AND (substr(en.created_at, ?, ?) <= ?)) AND (co.category = ?)) GROUP BY co.category, substr(en.created_at, ?, ?) ORDER BY sum(en.amount_paid) DESC, substr(en.created_at, ?, ?) ASC LIMIT ?
  -> bindings=[1, 7, 1, 7, "2026-04", 1, 7, "2026-06", "Analytics", 1, 7, 1, 7, 10]

./bin/telora run verify -C ontology
  -> verify ok
  -> coverage: projections/group_by match requests
  -> relations: shortest path, shared-edge merge, lex-smallest tie-break ok
  -> grain: request measures decide base grain (multi-grain knowledge) ok
  -> computed: substr dimension lowers to ScalarCall with literal bindings ok
  -> filter: equality and range filters lower to Eq/Ge/Le with And, values become bindings ok
  -> filter: non-projected filter dimension is authorized, path-selected and joined ok
  -> ordering: requested measure desc + dimension asc + top n ok
  -> profile: all plans accepted by knowledge plan_profile

./bin/telora run invalid -C ontology --best-effort
  -> error: measures have incompatible base entities（含级联 measure not on the request base grain）
  -> error: limit must be a positive integer
  -> error: order target measure not requested
  -> error: missing filter capability
  -> error: unauthorized filter request
  -> error: invalid filter value for CourseCategory
  -> status: error（非零退出，无 output）

./bin/telora check @test/ontology.telora -C ontology
  -> status: ok（10 个契约断言：revision、投影覆盖、grouping/join、profile、
     SQL 输出、多 grain base、受控计算维度、筛选 + Top N、排序、未投影筛选
     维度）

./bin/telora query exports @bin/main.telora -C ontology
  -> output: String
```

verify.telora 覆盖的 lowering 性质：

- **请求覆盖**：每个请求恰有一个投影，别名/顺序与请求一致，grouping 与维度
  一致；
- **最短边数优先**：`Courses` 同时有 1 边路径（EnrollCourse）与 2 边路径
  （EnrollLearner + LearnerCourse），选择 1 边路径；
- **共享边只保留首次出现**：`LearnerCountry` + `LearnerName` 两个维度都经
  EnrollLearner，Plan 只有一个 join；
- **同长度字典序最小**：`Badges` 有两条 1 边路径（EnrollBadgePrimary 在前、
  EnrollBadgeLate 在后），选择前者（ON 条件为 `course_id` 而非
  `enrollment_id`）；
- **多 grain base**：`AwardCount`（CertificateAwards grain）请求的 Plan 以
  `ca` 为 base source 并经 CertCourse 连接 Courses；
- **受控计算维度**：`EnrollmentMonth` lower 为 `'Substr` 的 `ScalarCall`，
  `transform_sqlite` 的字面量 binding 为 `[1, 7, 1, 7]`（投影 + group_by 各
  一组）；
- **等值筛选 + 未投影筛选维度**：`CourseCategory` 只在 `filters` 中（不在
  projections/group_by），仍授权、走 EnrollCourse 路径并合并 join `co`；
  top-level filter 是 `'Eq`，左端引用 join 别名，值 `'String("Analytics")`
  进入 bindings；
- **范围筛选 + Top N**：同一维度 `EnrollmentMonth` 两个筛选（`Ge` 2026-04、
  `Le` 2026-06）按顺序用 `'And` 组合；limit 7；bindings
  `[1, 7, "2026-04", 1, 7, "2026-06", 7]`（含投影外的 substr 字面量与 LIMIT）；
- **排序**：`Revenue` 降序（`sum`）后 `CourseCategory` 升序（`col:co.category`），
  limit 10，bindings 尾部 `'Int(10)`；
- **profile 覆盖**：全部 Plan 通过 `qb.validate(plan, plan_profile)`。

invalid.telora 覆盖的失败原子性：

- 混用两个 grain 的 measure -> `resolve_base` 明确失败；
- `limit = 'Some(0)` -> "limit must be a positive integer"；
- 排序引用未请求的 `'Measure('AwardCount)` -> "order target measure not
  requested"；
- 筛选引用无筛选能力的维度（EnrollmentMonth）-> "missing filter capability"；
- 筛选授权谓词拒绝（blocked subject）-> "unauthorized filter request"；
- 筛选值变体错误（CourseCategory 收到 Month）-> "invalid filter value for
  CourseCategory"。

## 限制

1. **关系方向固定**：反向遍历需企业显式声明反向关系。
2. **筛选算子受限**：只支持 `'Eq`/`'Ge`/`'Le` 三种比较与 `'And` 组合（对齐
   QueryBuilder 公共教程的受控比较集合）；不支持 `Or`/`Not`/`Lt`/`Gt` 等。
   若企业需要更宽的比较算子，属于 QueryBuilder profile/语义扩展，不在本 eDSL
   造算子。
3. **排序目标必须是已请求目标**：不能按未请求的 measure/dimension 排序（这是
   契约保证而非缺口）；排序表达式取自已解析的规范表达式。
4. **limit 必须是正整数**：非正数原子失败；`Plan.limit` 由 QueryBuilder 以
   `LIMIT ?` 参数化。
5. **截断仅信息性**：深度 8 边界上的截断不阻止 Plan 发布；企业应保证所需路径
   不超过 8 条边。
6. **profile 冲突的失败位置**：profile 越界在 Plan 组装完成后才被
   `qb.validate` 发现（DESIGN.md 第 5 步）；授权/grain/路径/筛选值/排序目标/
   limit 错误会在更早阶段 `fail!`。
7. **输入约束由运行期 fail! 承担**：别名唯一、mapping 引用有效等输入约束在
   lowering 时以 `fail!` 检查，不在构造期验证（构造器保持轻量）。
8. **`verify` 不能捕获 `fail!`**：fan-out-only / 未授权 / 缺失 / 非法筛选值 /
   非法排序目标 / 非法 limit 等失败路径由 `invalid.telora` 与教程/契约文档
   说明，验证二进制只断言成功路径与 profile 覆盖。
9. **计算维度属性实参限于维度实体**：`ComputedExpr` 的属性实参必须属于该维度
   实体（在维度 alias 上解析），不支持跨实体引用。

## 与 QueryBuilder 的边界

- 公共 API 复用 `qb.Plan`、`qb.PlanProfile`、`qb.JoinKind`、`qb.AggFunction`、
  `qb.ScalarFunction`、`qb.Val`、`qb.ValidationIssue`、`qb.OrderDirection` 等
  类型，不重新定义。
- 成功示例调用 `qb.transform_sqlite` 展示端到端 Query；该转换不是 eDSL API。
- eDSL 不包含任何物流题面中的实体、表、列、公式或 mapping；示例领域为虚构的
  在线课程平台分析（受控计算维度等价支持 Ent-1 的
  `substr(orders.created_at, 1, 7)` 能力，但使用虚构的
  `substr(en.created_at, 1, 7)`；筛选/排序/limit 使用虚构的月份、类别与 Top N）。
