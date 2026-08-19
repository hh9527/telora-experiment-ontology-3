# Ontology 3: QueryBuilder -> eDSL -> EnterpriseKnowledge

本仓库是 Telora 主仓库固定的 OpenCode 实验计划，包含四个隔离角色：

```text
A1 QueryBuilder: Plan -> SQLite Query
A2 ontology eDSL: EnterpriseKnowledge -> Request -> Plan
A3 ent-1 model: domain facts -> EnterpriseKnowledge
A4 intent-1: private text intent -> typed Request -> public lower -> Query
```

`Plan` 使用标准算子且方言中立；本轮 QueryBuilder 只具体化 SQLite，`Query` 形状为
`{ sql: String, bindings: Array(Val) }`。A2 的 EnterpriseKnowledge 声明可接受的
`PlanProfile`。A3 验证 `EnterpriseKnowledge + Request -> Plan -> Query`。

## 角色与可见性

- A1 只实现 `query-builder/`。
- A2 只实现 `ontology/`，只看 A1 的公共教程和契约。
- A3 只实现 `ent-1/`，只看 A1/A2 的公共教程和契约；公共查询面不得泄漏物理 mapping。
- A4 只实现 `intent-1/`，只看 A3 的公共查询教程/契约和自己的私有文字意图。
- coordinator 只启动四个长期角色，之后不解释或调度工作。

每个角色只使用两个 DAG 命令：

```text
./bin/oc-task pull <role>
./bin/oc-task submit <role> <artifact...>
```

角色只能提交以自己的角色名结尾的 artifact，例如 A3 只能提交 `.a3`。具体所有权、依赖、
检查项和 freshness 均由 DAG 引擎检查，不在角色权限中逐项枚举。`pull` 无工作时最多等待
60 秒，返回各候选正在等待的依赖；角色随后再次 pull。

## Artifact DAG

```text
lang + qb-req -> qb.a1
qb.a1 -> qb-review.a2 / qb-review.a3
qb-feedback-a2? + qb-feedback-a3? -> qb.a1 修订
qb.a1 -> Host 发布 qb

lang + edsl-req -> lang-learn.a2
qb + lang-learn.a2 -> edsl.a2
edsl-feedback? -> edsl.a2 修订
edsl.a2 -> Host 发布 edsl

lang + domain-ent-1 -> lang-learn.a3
qb + edsl + lang-learn.a3 -> ent-1-model.a3
ent-1-model.a3 -> Host 发布 ent-1-model
ent-1-model -> ent-1-query-surface.a3
ent-1-query-feedback-a4? -> ent-1-query-surface.a3 修订
ent-1-query-surface.a3 -> Host 发布 ent-1-query-surface

lang + intent-req -> lang-learn.a4
ent-1-query-surface + lang-learn.a4 -> intent-1.a4
intent-1.a4 -> Host 发布 intent-1
```

带 `?` 的输入只是普通可选 artifact：缺失时 mtime 视为 0，不阻断首版；发布后使较旧的
候选及其下游自动 stale。实际交付文件只作为存在且非空的检查项，不直接触发 DAG。
所有状态都由 `control/artifacts/*` 的 mtime 推导，不存在 claim、generation 或专用
feedback 状态。

所有跨角色正式移交都由 Host 审核后发布无角色后缀的 artifact：

```bash
./oc-ctl tasks t001
./oc-ctl publish t001 qb-feedback-a2
./oc-ctl publish t001 qb
./oc-ctl publish t001 edsl-feedback
./oc-ctl publish t001 edsl
./oc-ctl publish t001 ent-1-model
./oc-ctl publish t001 ent-1-query-surface
./oc-ctl publish t001 ent-1-query-feedback-a4
./oc-ctl publish t001 intent-1
```

发布反馈 artifact 前，Host 先把筛选后的正文写入其 checks 指定的反馈文件。语言机制问题
单独跟踪，不要求角色绕行。A4 首轮后最多追加一次语言能力范围内的公共查询面迭代。

## 运行

```bash
./oc-run ontology-3 t001 --port 4196
./oc-ctl start t001
```

使用 `oc-ctl status/recent/children/child-recent/files` 观察，`oc-ctl tasks` 查看 DAG，
`oc-ctl stats` 统计各角色时间、token 和产出。完成后运行：

```bash
./oc-ctl validate t001
./oc-ctl finish t001
```

准备阶段构建并复制 release Telora binary。coordinator 与 A1-A4 均固定使用
`deepseek/deepseek-v4-flash`。`experiment.json` 是 Host 配置，对角色不可见。
