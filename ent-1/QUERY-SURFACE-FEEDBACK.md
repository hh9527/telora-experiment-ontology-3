# Ent-1 公共参数化查询面增量反馈

当前公共 facade 的 `Request` 与文档仍只有 measures/dimensions，无法让查询设计者
表达真实业务范围、稳定排序和 Top N。请基于已批准的同一份私有 Knowledge，把
刚验证通过的能力安全地暴露到公共业务查询面。

## 公共 Request

- `Request` 必须包含 `measures`、`dimensions`、`filters`、`ordering`、`limit`，
  并公开查询设计者实际需要的封闭业务类型；不能要求调用方知道私有 model 模块。
- 筛选只允许封闭的 `DimensionId`、`Eq/Ge/Le` 与领域 `FilterInput`：
  `Month(String)`、`Tier(String)`、`Region(String)`、`Category(String)`。
- 排序只允许引用请求中已有的 measure/dimension，并使用 `Asc/Desc`；Top N 是正整数。
- `subject` 是查询发起者，只用于授权和诊断，文档明确说明它不会自动变成筛选条件。
- facade 继续只导出业务词汇、typed Request 与 `lower(Request) -> Query`，不得暴露
  表名、列名、alias、join 路径/mapping、Plan、SQL 片段或 QueryBuilder 表达式。

## 文档与 JSON

- `PUBLIC-QUERY-CONTRACT.md` 与 `QUERY-DESIGNER-TUTORIAL.md` 给出完整、可直接用于
  codec/JSON 的 Request 形状和 enum 编码示例，不能只展示 Telora 内部写法。
- 文档至少覆盖：
  1. `DeliveredPackages` + 2026-07 + 华东 + CarrierName Top 5；
  2. `UnitsShipped` + 2026-04..2026-06 + Gold + ServiceName Top 3；
  3. `UnitsShipped` + 2026-07 + ProductCategory Top 10。
- 解释缺失信息、歧义与已知不支持能力应由查询设计者先澄清；若提交无效 Request，
  `lower` 的 Telora 诊断是修正或归因的依据，不得通过删除条件扩大查询范围。

## 验证

- `query-surface.telora` 与测试必须实际 lower 上述参数化请求，断言 SQL/bindings、
  稳定排序、limit 和确定性；所有业务值与 N 必须进入 bindings。
- 保留并扩展失败测试：缺失 capability、输入变体不匹配、非正 limit、未请求排序
  目标、Order grain + ProductCategory grain conflict 都应原子失败。
- 完成公共实现、教程、契约和真实命令验证后再提交 `ent-1-query-surface.a3`。
