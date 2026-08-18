# Ontology 3: QueryBuilder -> eDSL -> EnterpriseKnowledge

本仓库是 Telora 主仓库通过 submodule 固定的 OpenCode 实验计划。它把原来由一个
角色同时承担的查询计划、SQL lowering 和本体 eDSL 拆成四个隔离角色：

```text
A1 QueryBuilder: Plan -> SQLite Query
A2 ontology eDSL: EnterpriseKnowledge -> Request -> Plan
A3 ent-1 model: domain facts -> EnterpriseKnowledge
A4 intent-1: private text intent -> typed Request -> public lower -> Query
```

`Plan` 是使用标准算子的方言中立计划；`QueryBuilder` 在本轮只内置 SQLite 具体化，
产物 `Query` 的参考形状为 `{ sql: String, bindings: Array(Val) }`。不要求其他 SQL
后端或通用插件框架。A2 声明其 EnterpriseKnowledge
接受的 `PlanProfile`，且产生的每个 Plan 都必须受该 profile 约束。A3 最终验证：

```text
EnterpriseKnowledge + Request -> Plan -> Query
```

## 角色边界

- A1 只能实现 `query-builder/`，不接触 ontology 或企业领域。
- A2 只能实现 `ontology/`，只看 A1 的公共教程和契约，不看其设计、源码和 notes。
- A3 只能实现 `ent-1/`，只看 A1/A2 的公共教程和契约，不看上游私有实现。
- A3 从同一私有 EnterpriseKnowledge 额外发布不含物理 mapping 的公共查询 facade。
- A4 只能实现 `intent-1/`，只看 A3 的公共查询教程/契约和自己的私有文字意图；不能
  读取 DOMAIN、A3 源码或 A1/A2 交付。
- coordinator 只启动 A1/A2/A3/A4 各一次，此后不再解释或推进任务。

流程由 `experiment.json.workflow` 中的文件 DAG 决定：

```text
lang.ready + domain.ready
  -> A1 QueryBuilder -> qb.rc
  -> A2/A3 可独立发布 qb-review-a2.rc / qb-review-a3.rc
  -> Host 可选发布 qb-feedback-a2.feedback / qb-feedback-a3.feedback -> A1 修订 qb.rc
  -> Host 发布 qb.ready
  -> A2 ontology eDSL -> edsl.rc（可吸收尚未完成的 A2 review）
  -> Host 发布 edsl.ready
  -> A3 EnterpriseKnowledge -> ent-1-model.rc（可吸收尚未完成的 A3 review）
  -> Host 发布 ent-1-model.ready
  -> A3 公共查询面 -> ent-1-query-surface.rc
  -> Host 发布 ent-1-query-surface.ready
  -> A4 typed Request + public lower -> intent-1.rc
  -> Host 可选发布 ent-1-query-feedback-a4.feedback -> A3/A4 各修订一次
  -> Host 发布 intent-1.ready
```

start 后 A1/A2/A3/A4 并行学习本轮 Telora；A1 发布可执行 QueryBuilder 候选后，A2 依据
DESIGN、A3 依据 DOMAIN 审查公共能力。若此时只有 `qb.rc`，角色领取独立 review；若
Host 已发布 `qb.ready` 且角色的 build 也 runnable，调度器优先领取 build，并把尚未
完成的 review 放入其 `absorbed` 列表。角色在 build 过程中先 `mark-done` review `.rc`，
父 claim 保持有效，再完成 build `.rc`。原始评审不阻断 Host 放行。

A3 的私有模型和公共查询面是两个 Host 审核边界。只有私有模型放行后，A3 才能领取
公共 facade 任务；facade 必须捕获同一份 `knowledge`，不得维护第二份领域事实。A4
只在公共查询面放行后领取意图任务。`intent-1/telora-deps.json` 为模块解析登记传递
package，但 A4 权限仍不允许读取它们；这不是输入边界的扩大。

每个角色循环调用 `bin/oc-task next <role>` 领取唯一 runnable task，完成后调用
`mark-done <role> <name.rc>`。任务名就是其 `.rc` 节点，后缀必须显式给出。Agent 只
控制配置赋予自己的 `.rc`；Host 独占 `.feedback` 和 `.ready`。
输出文件只用于完整性检查，不因存在而触发下游。每次正式跨角色移交必须经过 Host
审核并发布 `.ready`；下游在此之前没有可领取的输出任务。claim 使用文件锁原子创建，
任务运行期间输入变化会拒绝完成并要求重新领取。

常用人工控制命令是：

```bash
./oc-ctl tasks t001
./oc-ctl feedback t001 qb-feedback-a2.feedback --body-file /tmp/a2-feedback.md
./oc-ctl ready t001 qb.ready
./oc-ctl ready t001 edsl.ready
./oc-ctl ready t001 ent-1-model.ready
./oc-ctl ready t001 ent-1-query-surface.ready
./oc-ctl feedback t001 ent-1-query-feedback-a4.feedback --body-file /tmp/a4-feedback.md
./oc-ctl ready t001 intent-1.ready
```

工具会拒绝提前放行或发布空反馈。新反馈晚于 `qb.rc` 时使候选失效；A1 修订并重新发布
后，旧反馈自动变为历史版本。重新发布 `.rc` 也会使旧 `.ready` 失效，必须重新审核。
一次反馈发布会立即使当前候选失效，因此 Host 应先筛选并合并本次准备采纳的意见，再
通过对应来源节点发布一个 body；若之后还要发布另一来源反馈，须等待 A1 产生新候选且
A2/A3 完成对新候选的评审，这会形成下一次显式修订。

A4 首轮完成后，Host 若确认问题只属于公共契约、教程或 lowering，可筛选
`intent-1/FEEDBACK.md` 后发布一次 `ent-1-query-feedback-a4.feedback`。它使旧公共面与
A4 结果失效，A3/A4 各重跑一次；语言机制问题单独跟踪，不要求角色绕行。实验不追加
第二轮这种迭代。

`ent-1/FEEDBACK.md` 在 plan 中是零字节文件；两份 `QUERY-BUILDER-FEEDBACK.md`
在消费者审查前为空白。各角色 GOAL 与角色协议均列出完整输入清单，并规定各 task
可以读取和修改的内容。`experiment.json` 与 `.oc-task/**` 对角色不可见。

## 运行

在外部终端先登记 execution：

```bash
./oc-run ontology-3 t001 --port 4196
```

控制端准备完成后启动：

```bash
./oc-ctl start t001
```

观察使用 `oc-ctl status/recent/children/child-recent/files`；DAG、claim 和阻塞状态使用
`./oc-ctl tasks t001`。成本与产出统计使用：

```bash
./oc-ctl stats t001
```

该命令从 child session 的逐消息元数据统计各角色和学习/工作阶段的 active、elapsed、
waiting 时间以及 input/output/reasoning/cache token；从最终工作区统计各角色保留的
Telora 代码和文档文件数、物理行数与 bytes。阶段名称、首次正式工作文件和产出归属由
`experiment.json.metrics` 固定，因此不依赖解析 Agent 的自然语言总结；临时 probe 只有
在 metrics artifact patterns 中显式列出时才计入产出。运行中、idle、finished、retired
execution 使用同一个 `telora.opencode-stats/v1` JSON schema，准备实验时 metrics 配置
会冻结到 execution state。A3 的 `modeling` 与 `query_surface_design` 由各自首个文件
写入分段，A4 单独统计 `intent_modeling`；未配置角色仍会统计时间、token 与模型，但阶段明确标记为
`unclassified`，不推测学习/工作边界。

完成后运行 `./oc-ctl validate t001` 与 `./oc-ctl finish t001`。`experiment.json`
是 Host 配置，由 OpenCode 权限显式隐藏，不是角色输入。

准备阶段使用 `cargo build --release -p telora`，并把 release binary 固定复制到实验
工作区。coordinator、A1、A2、A3、A4 均固定使用 `deepseek/deepseek-v4-flash`；全局配置与角色
frontmatter 双重声明同一模型，避免继承外部 OpenCode 默认值。角色可见目标只要求
非法请求失败且不发布可信产物；诊断数量和恢复结构仅由 Host 隐藏评估。
