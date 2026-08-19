# A4 目标：从文字意图构造公共 typed Request

你是不了解数据库物理实现的查询设计者。只依据 A3 发布的公共查询面，将
`intent-1/INTENT.md` 中的文字意图忠实表达为公共契约定义的 typed Request，并调用
公共 `lower(Request)` 得到 Query 或诊断。

## 完整输入清单

- `bin/telora`
- `docs/LANG-TUTORIAL.md`
- `docs/TELORA-CLI.md`
- `intent-1/GOAL.md`
- `intent-1/INTENT.md`
- `intent-1/telora-deps.json`
- `intent-1/FEEDBACK.md`
- A4 自己已经产生的 `intent-1/src/**`、`intent-1/tests/**`
- Host 放行后：`ent-1/QUERY-DESIGNER-TUTORIAL.md` 与
  `ent-1/PUBLIC-QUERY-CONTRACT.md`

不得读取企业私有 DOMAIN、A3 私有模型源码、QueryBuilder/ontology 的文档或源码。
不得通过错误消息、`show` 或其他工具反向恢复表、列、alias、join mapping 或 SQL 模板。

## 交付物

- `intent-1/src/`：只包含公共 typed Request 与调用公共 `lower` 所需代码；
- `intent-1/src/bin/main.telora`：合法文字意图得到 Query；
- `intent-1/src/bin/verify.telora`：重复 lowering 的 Query 严格相同，并验证公共结果契约；
- `intent-1/src/bin/invalid.telora`：已知公共词汇构成的非法意图被拒绝且无 Query；
- `intent-1/tests/intent.telora`：公共类型与合法链路检查；
- `intent-1/FEEDBACK.md`：公共查询面存在的具体歧义、缺口或不必要泄漏；
- `intent-1/NOTES.md`：Request 选择、真实验证结果和剩余风险。

不得修改 `intent-1/telora-deps.json`，不得定义替代 Request DSL、Plan、QueryBuilder、
SQL renderer 或诊断容器。A4 不负责解析任意自然语言，只处理本轮两个有界意图。

## 验证

```text
./bin/telora run main -C intent-1
./bin/telora run verify -C intent-1
./bin/telora run invalid -C intent-1
./bin/telora run invalid -C intent-1 --best-effort
./bin/telora check @test/intent.telora -C intent-1
./bin/telora show @bin/main.telora -C intent-1
```

非法入口的两个 `run` 应失败；普通模式应给出最直接的原因，`--best-effort` 可用于获取
更多诊断，但两种模式必须同属失败。完成后执行 `oc-task submit a4 intent-1.a4`。
