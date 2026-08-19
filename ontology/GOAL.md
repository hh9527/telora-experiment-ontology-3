# A2 目标：实现 EnterpriseKnowledge eDSL

使用 Telora 实现 `ontology/DESIGN.md`：让企业表达结构化 EnterpriseKnowledge，
并实现 `EnterpriseKnowledge -> Request -> Plan`。Plan、PlanProfile 与 Query 只能使用
QueryBuilder 的公共交付，不得在 ontology 中另行定义。

## 完整输入清单

固定路径输入（内容属于当前发布轮次）：

- `bin/telora`
- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `ontology/GOAL.md`
- `ontology/DESIGN.md`
- `ontology/telora-deps.json`
- A2 自己已经产生的 `ontology/src/**` 与 `ontology/tests/**`

动态输入：

- `oc-task` 返回 `qb-feedback.a2` 后读取 QueryBuilder 公共候选并完成能力
  审查，结果写入 `ontology/QUERY-BUILDER-FEEDBACK.md`；
- `oc-task` 返回 `edsl.a2` 后读取 QueryBuilder 已放行的公共教程与契约并实现 eDSL；
- `ontology/QUERY-BUILDER-FEEDBACK.md`：A2 对 QueryBuilder 公共草案的审查交付。
- `edsl-feedback` artifact：首版 `edsl.a2` 后由 Host 筛选发布的 eDSL
  修订反馈；存在时必须完整读取、修订并重验。

不得读取 QueryBuilder 私有设计、源码、tests 或 notes；不得读取企业 DOMAIN 或源码。
任务就绪与重跑由 `oc-task` 根据文件时间戳确定。

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

`qb` 或 `edsl-feedback` 更新使 `edsl.a2` 重新就绪时，重新读取当前公共输入并验证既有
eDSL。完成后用一次 `oc-task submit a2 ...` 提交本次 pull 返回的唯一 artifact。
