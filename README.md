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
- coordinator 只按角色完成状态推进，不解释任务或修改文件。

A1 完成后 A2 才开始实现；A2 完成后 A3 才开始建模。A2、A3 可在等待上游时并行
学习 Telora、CLI 和各自题面。A3 首次交付后流程挂起，Host 可筛选反馈并最多启动
一次 A1 -> A2 -> A3 的增量修订。

## 运行

在外部终端先登记 execution：

```bash
./oc-run ontology-3 t001 --port 4196
```

控制端准备完成后启动：

```bash
./oc-ctl start t001
```

观察使用 `oc-ctl status/recent/children/child-recent/files`。首次交付结束后，如 Host
已经把 `ent-1/FEEDBACK.md` 整理为批准反馈，可执行一次：

```bash
./oc-ctl iterate t001
```

完成后运行 `./oc-ctl validate t001` 与 `./oc-ctl finish t001`。`experiment.json`
是 Host 配置，由 OpenCode 权限显式隐藏，不是角色输入。
