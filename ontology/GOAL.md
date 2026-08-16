# A2 目标：实现 EnterpriseKnowledge eDSL

使用 Telora 实现 `ontology/DESIGN.md`：让企业表达结构化 EnterpriseKnowledge，
并实现 `EnterpriseKnowledge -> Request -> Plan`。Plan、PlanProfile 与 Query 只能使用
QueryBuilder 的公共交付，不得在 ontology 中另行定义。

## 稳定输入

- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `query-builder/QUERY-BUILDER-TUTORIAL.md`
- `query-builder/PUBLIC-CONTRACT.md`
- `ontology/GOAL.md`
- `ontology/DESIGN.md`

不得读取 QueryBuilder 私有设计、源码、tests 或 notes；不得读取企业 DOMAIN 或源码。

## 交付物

- `ontology/src/`：领域无关的可复用 eDSL；
- `ontology/src/bin/main.telora`：虚构的小型知识经 Request 得到 Plan，并调用公共
  QueryBuilder 得到 Query 的成功示例；
- `ontology/src/bin/verify.telora`：验证请求覆盖、关系选择和 profile 覆盖；
- `ontology/src/bin/invalid.telora`：多个非法意图产生诊断且不发布 Plan/Query；
- `ontology/tests/ontology.telora`：公共类型与模块契约检查；
- `ontology/DSL-TUTORIAL.md`：A3 可独立使用的教程；
- `ontology/PUBLIC-CONTRACT.md`：EnterpriseKnowledge 输入与 lowering 保证；
- `ontology/NOTES.md`：设计、验证结果和限制。

不得修改 `ontology/telora-deps.json`。不得包含物流题面中的实体、表、列、公式或
mapping。不得用最终 builder、预渲染 SQL、`Any`、`Dyn` 或 String 身份逃逸。

## 验证

```text
./bin/telora run main -C ontology
./bin/telora run verify -C ontology
./bin/telora run invalid -C ontology --best-effort
./bin/telora check @test/ontology.telora -C ontology
./bin/telora show @bin/main.telora -C ontology
```

完成时报告真实交付、验证结果与剩余限制，不要求 Git commit。

## 反馈修订

收到 Host 批准的反馈时，只处理明确归属于 eDSL 的项目。先重新读取 QueryBuilder
公共交付以适配其批准更新，再记录每项接受或拒绝，更新实现与文档并重新验证。
