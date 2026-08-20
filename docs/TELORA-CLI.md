# Telora 验证流程

每个 Telora crate 的可复用模块位于 `src/`，应用入口位于 `src/bin/`，测试入口位于
`tests/`。`telora-deps.json` 固定该 crate 的依赖边界，不得修改。

Telora 从当前目录向上查找最近的 `telora-deps.json`，因此命令可以从目标 crate 根
目录或其任意子目录执行。`-C` 可以显式改变查找的起始目录，该目录不必是 crate 根。
命令参数使用稳定逻辑模块 ID，不使用物理文件名：

```text
./bin/telora run main -C ontology
./bin/telora run verify -C ontology
./bin/telora check @test/ontology.telora -C ontology
./bin/telora show @bin/main.telora -C ontology
./bin/telora show @src/ontology.telora -C ontology -k type,let,def,import
./bin/telora show @src/ontology.telora -C ontology --exports
./bin/telora show @src/ontology.telora -C ontology --at 12:4
./bin/telora show std/string -C ontology --exports
./bin/telora show std/array -C ontology -p flat_map
./bin/telora run -S path/to/file.telora
./bin/telora run main -C ontology --entry path/to/entry.telora
./bin/telora run invalid -C ontology --best-effort
```

企业 crate 同理，在 `ent-1/` 下运行 `run main` 和 `run verify`；`check
@test/logistics.telora` 只作为开发期分析。

`check` 的输入仍是完整 Module，不是任意表达式 scratch。模块顶层使用 `def` 声明
计算根并至少显式 export 一项；顶层 `let`、裸调用和 final expression 均不合法。
需要局部步骤时把它们放进 `do`：

```telora
export def lowering_case = do {
    let plan = lower(request);
    validate_plan.must_ok!(plan)
};
```

多个独立检查应写成多个具名 export，使 best-effort `check` 可以继续不依赖失败项的根。

- `run name` 固定执行 `@bin/name.telora`；实验使用的内置 Entry 把其 String export
  `output` 作为 `Output(String)` effect 发给 Host。`name` 是不含路径分隔符和
  `.telora` 后缀的单个 stem。
- `run name -C context` 从 `context` 开始向上发现 manifest。
- `run ... --best-effort` 只在遇到问题时用于扩大诊断覆盖。它在启动 Entry 前对 Main 做
  best-effort 诊断求值；只要出现任何 error，stderr 输出 `telora.run/v1` JSONL 诊断与
  error summary，非零退出且不产生任何 Entry effect，即使一个不依赖失败的干净根值仍能
  算出。没有 error 时仍重新走严格 Entry/Host lifecycle；成功结果的最终验收使用普通
  `run`。实验中的 `invalid` 是有意失败的诊断入口，本参数用于调查问题时扩大诊断覆盖。
- `run ... --entry file.telora` 由 Host 显式选择纯 Edge Entry；省略时使用内置 Entry，
  从 Main 的显式 export record 读取 String `output`。自定义 Entry 的 `MainType` 和
  输出编码由它自己规定，可以通过 stdio-child effects 编排进程。实验中的普通建模、
  探索和验收均省略此参数，不自行创建 Entry。Entry 文件路径相对命令进程的当前目录
  解析。
- `run -S file` 进入 standalone 模式，不发现 manifest，只使用根文件内的
  `crate.dependency` / `crate.format` options；这些 options 相对文件父目录解析。
  `-S` 与 binary name、`-C` 互斥。
- `run`、`check` 和 `show` 的 `-C context` 都从 `context` 开始向上发现 manifest；
  `check` 和 `show` 接受完整稳定模块 ID，`check @test/...` 检查测试入口，`run` 只
  接受 binary name。
- `check` 用 best-effort 模式继续彼此独立的求值，以一次收集更多诊断；最终判定仍然
  严格。stdout 完全采用 `telora.check/v1` JSONL：先输出诊断 records，最后输出一条
  `summary` record。只有完整求值并形成内部 semantic Module graph 时 summary 才是
  `status: "ok"`；它不把递归 TypeMetadata 等内部图物化为 legacy Host value。任何
  语法、类型、解析或运行时失败都会得到 `status: "error"` 和非零退出。
  最终应用验收仍以 `run` 为准，因为 `run` 还经过 Entry 调度。
- `show` 输出 `telora.show/v1` JSONL 语义记录；默认列出选中模块的顶层 local
  definitions。它查询 recoverable CST 和部分语义/求值证据图，因此在模块损坏时仍可
  返回不受影响的事实；命令成功只表示查询完成，不表示模块能够通过 `check` 或 `run`。
- `show std/...` 直接查询 Host 注册的公开标准库逻辑模块，与源码 `import "std/..."`
  使用同一模块身份；不需要也不能把 `std` 声明成 workspace dependency。
- `-p` 按名称的大小写敏感字面子串过滤，不是 glob 或正则。
- `-k` 接受逗号分隔的 `type,let,def,import`；`--exports` 改查公共接口并与 `-k`
  互斥。
- Namespace import 的记录用 `target` 给出目标模块 ID，不带普通值 `type`；用
  `show <target> --exports` 查询其成员的精确 type/scheme。Selective import 的记录
  直接携带所选成员的精确 type/scheme。
- `--at line[:column]` 使用从一开始计数的位置：行号选择与该行相交的事实，行列选择
  覆盖该点的事实；它与 `-p`、`-k`、`--exports` 互斥。

程序中的 `dbg!(expr)` 和 `expr.dbg!()` 把旁路观察写入 stderr，不改变 stdout 的
`output`。每个事件是一行紧凑 JSON：

```json
{"name":"var","repr":"3","module":"@bin/main.telora","line":12}
{"name":"plan","repr":"{...}","module":"@bin/main.telora","line":13,"message":"generated"}
```

固定字段为 `name`、`repr`、`module`、`line`；只有显式 message 时才有 `message`。
`repr` 是有界 debug 表示，不是可反序列化的 JSON 值。Host 是否输出或丢弃事件对
Telora 程序不可感知。Float 的 `repr` 使用 Debug 表示，例如 `3.0` 和 `-0.0`；它与
字符串插值和 `fmt.display` 使用的 Display 表示不同。

命令退出码为零表示请求成功；非零表示 CLI 或 Telora 拒绝。`show` 的空匹配成功且
没有输出。记录中的 `authority` 区分 `authoritative`、`recovery` 与 `debug` 事实。
表达式级记录属于 `debug`；错误恢复记录的 authority 服从其事实和模块状态。
