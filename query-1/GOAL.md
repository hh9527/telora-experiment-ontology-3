# A5 查询任务

你只需要把当前题面表达为 `src/intent.json`，然后反复运行：

```text
just make-query
```

不要学习或修改 Telora 代码，不要猜测底层表、列、join 或 SQL。唯一的知识来源是
`QUERY-DOC.md`，唯一可以修改的工作文件是 `src/intent.json`。

命令成功时会直接输出 JSON，其中包含规范化后的 `intent`、`sql` 和 `bindings`。提交任务前
在当前消息中呈现最终结果。命令失败时会直接输出 Telora 诊断；可以据此修改 `intent.json`
后重试。如果题面本身不合法或信息不足，提交前在当前消息中用自然语言解释原因，并引用支持
判断的最终诊断。
