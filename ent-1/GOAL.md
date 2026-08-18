# A3 目标：建立物流 EnterpriseKnowledge

作为企业知识作者，使用 ontology eDSL 表达 `ent-1/DOMAIN.md`，得到具体的
EnterpriseKnowledge。用合法和非法 Request 验证完整链路：

```text
EnterpriseKnowledge + Request -> Plan -> Query
```

不得查看上游私有实现，不得在企业 package 中重写 eDSL、Plan 或 QueryBuilder。

## 完整输入清单

固定路径输入（内容属于当前发布轮次）：

- `bin/telora`
- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `ent-1/GOAL.md`
- `ent-1/DOMAIN.md`
- `ent-1/telora-deps.json`
- `ent-1/FEEDBACK.md`：首次为空，首次建模后由 A3 写入；Host 可以在后续发布轮次
  前将它整理为批准反馈；
- A3 自己已经产生的 `ent-1/src/**` 与 `ent-1/tests/**`

动态输入由 `oc-task` 按任务依赖开放：

- `qb-review-a3.rc`：读取 QueryBuilder 公共候选，依据 DOMAIN 完成能力审查并
  写入 `ent-1/QUERY-BUILDER-FEEDBACK.md`，但不开始最终建模；
- `ent-1-model.rc`：在 `qb.ready` 与 `edsl.ready` 均发布后，读取 QueryBuilder 和
  ontology 公共教程与契约并开始最终建模。

不得读取 `query-builder/{GOAL,DESIGN,NOTES,src,tests}` 或
`ontology/{GOAL,DESIGN,NOTES,src,tests}`。
任务就绪与重跑由 `oc-task` 根据文件时间戳确定。

## 交付物

- `ent-1/src/`：封闭 vocabulary、领域事实、能力、表达式、关系 mapping、
  PlanProfile，以及实例化的 EnterpriseKnowledge；
- `ent-1/src/bin/main.telora`：合法 Request 得到完整 Plan 和 Query；
- `ent-1/src/bin/verify.telora`：验证 Plan 覆盖、profile 约束和 Query 确定性；
- `ent-1/src/bin/invalid.telora`：非法场景产生 Host 诊断且无 Plan/Query；
- `ent-1/tests/logistics.telora`：公共契约检查；
- `ent-1/FEEDBACK.md`：按 QueryBuilder/eDSL 分区记录具体使用摩擦；
- `ent-1/QUERY-BUILDER-FEEDBACK.md`：最终建模前对 QueryBuilder 草案的能力审查；
- `ent-1/NOTES.md`：模型选择、验证结果和风险。

不得修改 `ent-1/telora-deps.json`。不得复制共享算法、定义替代 Plan、用任意 builder
手工组装 Plan、使用预渲染 SQL、`Any`、`Dyn` 或 String 语义身份。

合法入口必须分别保留并验证 Plan 与 Query：

```text
plan  = make_query_creator(knowledge)(request)
query = query_builder.transform(plan)
```

## 验证

```text
./bin/telora run main -C ent-1
./bin/telora run verify -C ent-1
./bin/telora run invalid -C ent-1 --best-effort
./bin/telora check @test/logistics.telora -C ent-1
./bin/telora show @bin/main.telora -C ent-1
```

完成时报告真实结果与具体反馈，不要求 Git commit。

## 上游更新复验

后续发布轮次的 `ent-1-model.rc` 就绪后，重新读取两层公共交付，保持同一题面和验收
场景，适配批准更新并重新验证；更新 feedback 与 notes，不推测上游私有实现。每项
任务完成后必须执行对应的 `oc-task mark-done`。
