# A1 目标：实现 QueryBuilder

使用 Telora 实现 `query-builder/DESIGN.md`。这是领域无关的基础 package，负责标准
Plan 算子、PlanProfile、Plan 验证和确定性的 SQLite `Plan -> Query`，不得包含
ontology 概念或企业领域事实。本轮只实现 SQLite，不扩展其他 SQL 后端。

## 完整输入清单

固定路径输入（内容属于当前 generation）：

- `bin/telora`
- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `query-builder/GOAL.md`
- `query-builder/DESIGN.md`
- `query-builder/telora-deps.json`
- A1 自己已经产生的 `query-builder/src/**` 与 `query-builder/tests/**`

动态输入：

- `ent-1/FEEDBACK.md`：G001 时是空文件，后续 generation 可以包含 Host 批准反馈；
- `control/GNNN-INPUTS-READY`：编号最大的 marker 宣布该 generation 的 binary、教程、
  设计和 feedback 已经原子发布，可以开始学习、实现或修订。

不在清单中的文件不是 A1 输入。A1 不得创建或修改任何 control marker。

## 交付物

- `query-builder/src/`：可复用 package；
- `query-builder/src/bin/main.telora`：展示合法 Plan 到参数化 SQLite Query；
- `query-builder/src/bin/verify.telora`：验证 profile 覆盖、顺序与确定性；
- `query-builder/src/bin/invalid.telora`：展示越界或非法 Plan 产生 Host 诊断且无 Query；
- `query-builder/tests/query-builder.telora`：公共类型与模块契约检查；
- `query-builder/QUERY-BUILDER-TUTORIAL.md`：A2/A3 可独立使用的教程；
- `query-builder/PUBLIC-CONTRACT.md`：稳定公共类型、能力与保证；
- `query-builder/NOTES.md`：设计选择、验证结果和限制。

不得修改 `query-builder/telora-deps.json`。公共 API 不得使用 `Any`、`Dyn`、native
声明或 String 语义身份逃逸。运行时值必须进入 bindings。

## 验证

```text
./bin/telora run main -C query-builder
./bin/telora run verify -C query-builder
./bin/telora run invalid -C query-builder --best-effort
./bin/telora check @test/query-builder.telora -C query-builder
./bin/telora show @bin/main.telora -C query-builder
```

完成时报告真实交付、验证结果与剩余限制，不要求 Git commit。

## 反馈修订

G001 实现完整 QueryBuilder。后续 generation 重新阅读语言教程并使用新 binary，只
处理 `ent-1/FEEDBACK.md` 中明确归属于 QueryBuilder 的批准项，同时重新验证既有
交付；即使 feedback 为空，也要验证 runtime revision 是否影响实现。不得读取企业源码。
