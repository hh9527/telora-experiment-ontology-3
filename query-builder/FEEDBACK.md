# QueryBuilder 第二次送审反馈

Host 审核最终领域模型时发现一个跨层契约偏差：`Plan.revision` 当前固定为 `Int`，但私有题面明确要求最终规范 Plan 的 revision 为字符串 `logistics-ontology-v1`。把 `1` 放入 Plan、把可读标签保存在 Plan 外不满足“Plan 自身完整保留 revision”的要求。

请增量修订：

1. 让公共 `Plan.revision` 能直接、逐字节保留稳定可读 revision 标签。本实验使用 `String` 即可，不需要引入 `Any`/`Dyn` 或复杂版本协议。
2. 更新所有 QueryBuilder 示例、fixtures、测试、教程、公共契约和 notes；确定性转换仍完整保留 Plan 输入，SQL/Query 不需要包含 revision。
3. 增加断言证明 `Plan.revision == "..."`，并确保 transform/profile/validation 行为不受影响。
4. 保持已通过的 `Substr`、grouping 嵌套 bindings、import 路径及其他算子契约不变。

这是公共类型修正，下游 eDSL/领域模型会在新版 `qb` 发布后同步重建。
