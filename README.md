# Ontology 3: QueryBuilder -> eDSL -> EnterpriseKnowledge

本仓库是 Telora 主仓库通过 submodule 固定的 OpenCode 实验计划。它把原来由一个
角色同时承担的查询计划、SQL lowering 和本体 eDSL 拆成三个顺序交付：

```text
A1 QueryBuilder: Plan -> SQLite Query
A2 ontology eDSL: EnterpriseKnowledge -> Request -> Plan
A3 ent-1 model: domain facts -> EnterpriseKnowledge
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
- coordinator 只按角色完成状态推进，不解释任务；它唯一能修改的是 `control/`
  下协议规定的一次性 marker。

每轮输入使用一个 generation：

```text
G001-INPUTS-READY
  -> G001-QUERY-BUILDER-READY
  -> G001-EDSL-READY
  -> G001-ENTERPRISE-READY
```

inputs-ready 后 A1/A2/A3 并行学习本代 Telora；A1 完成后，A2 与 A3 并行学习
QueryBuilder，同时 A2 开始实现 eDSL；A2 完成后 A3 才读取 eDSL 并建模。较新的
`GNNN-INPUTS-READY` 会使旧代下游 ready 状态失效，但保留既有文件供增量修改。
因此一次新 generation 可以发布批准反馈，也可以在所有角色 idle 时原子更新
binary、语言/CLI 教程，或同时完成二者，再沿相同依赖链重验。

`ent-1/FEEDBACK.md` 在 plan 中是零字节文件。各角色 GOAL 与角色协议均列出完整
输入清单，并规定每个 generation marker 出现前后允许读取什么、允许执行什么。coordinator 对
三个角色始终只发送“输入状态已变化”的固定消息。

## 运行

在外部终端先登记 execution：

```bash
./oc-run ontology-3 t001 --port 4196
```

控制端准备完成后启动：

```bash
./oc-ctl start t001
```

观察使用 `oc-ctl status/recent/children/child-recent/files`。成本与产出统计使用：

```bash
./oc-ctl stats t001
```

该命令从 child session 的逐消息元数据统计各角色和学习/工作阶段的 active、elapsed、
waiting 时间以及 input/output/reasoning/cache token；从最终工作区统计各角色保留的
Telora 代码和文档文件数、物理行数与 bytes。阶段名称、首次正式工作文件和产出归属由
`experiment.json.metrics` 固定，因此不依赖解析 Agent 的自然语言总结；临时 probe 只有
在 metrics artifact patterns 中显式列出时才计入产出。运行中、idle、finished、retired
execution 使用同一个 `telora.opencode-stats/v1` JSON schema，准备实验时 metrics 配置
会冻结到 execution state。未配置角色仍会统计时间、token 与模型，但阶段明确标记为
`unclassified`，不推测学习/工作边界。

一代完成后，如 Host
已经整理批准反馈，或者实验基础设施已经原子发布新的 binary 与教程，可启动下一代：

```bash
./oc-ctl iterate t001
```

当前控制器对一次 execution 仍只开放一个追加 generation；generation 协议本身不
复用 marker，未来放宽次数时无需修改角色语义。runtime 重发布必须由 Host/controller
先完成文件替换与摘要记录，coordinator 只负责最后创建 inputs-ready。

完成后运行 `./oc-ctl validate t001` 与 `./oc-ctl finish t001`。`experiment.json`
是 Host 配置，由 OpenCode 权限显式隐藏，不是角色输入。
