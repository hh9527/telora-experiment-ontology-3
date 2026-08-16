# A2 目标：实现 EnterpriseKnowledge eDSL

使用 Telora 实现 `ontology/DESIGN.md`：让企业表达结构化 EnterpriseKnowledge，
并实现 `EnterpriseKnowledge -> Request -> Plan`。Plan、PlanProfile 与 Query 只能使用
QueryBuilder 的公共交付，不得在 ontology 中另行定义。

## 完整输入清单

固定路径输入（内容属于当前 generation）：

- `bin/telora`
- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `ontology/GOAL.md`
- `ontology/DESIGN.md`
- `ontology/telora-deps.json`
- A2 自己已经产生的 `ontology/src/**` 与 `ontology/tests/**`

动态输入：

- `control/GNNN-INPUTS-READY`：编号最大的 marker 宣布当前 generation 输入已发布；
- `control/GNNN-QUERY-BUILDER-DRAFT-READY`：同 generation 的该 marker 出现后读取
  QueryBuilder 公共草案并完成能力审查，结果写入 `ontology/QUERY-BUILDER-FEEDBACK.md`；
- `control/GNNN-QUERY-BUILDER-READY`：同 generation 的该 marker 出现后才能读取
  `query-builder/QUERY-BUILDER-TUTORIAL.md` 与 `query-builder/PUBLIC-CONTRACT.md`，
  并开始实现 eDSL；
- `ent-1/FEEDBACK.md`：G001 时是空文件，后续 generation 可以包含 Host 批准反馈。
- `ontology/QUERY-BUILDER-FEEDBACK.md`：A2 对 QueryBuilder 公共草案的审查交付。

不得读取 QueryBuilder 私有设计、源码、tests 或 notes；不得读取企业 DOMAIN 或源码。
A2 不得创建或修改任何 control marker。

## 交付物

- `ontology/src/`：领域无关的可复用 eDSL；
- `ontology/src/bin/main.telora`：虚构的小型知识经 Request 得到 Plan，并调用公共
  QueryBuilder 得到 Query 的成功示例；
- `ontology/src/bin/verify.telora`：验证请求覆盖、关系选择和 profile 覆盖；
- `ontology/src/bin/invalid.telora`：非法请求失败且不发布可信 Plan/Query；
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

后续 generation 的 QueryBuilder ready 后，重新读取其公共交付，只处理明确归属于
eDSL 的批准项目并重新验证既有交付；feedback 为空时仍验证 runtime revision 影响。
