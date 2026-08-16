# Ontology 3 evaluation method

本文只供 Host 使用，不注入角色提示。实验问题是：三个隔离角色能否仅通过稳定公共
契约，依次建立 QueryBuilder、EnterpriseKnowledge eDSL 和具体企业知识，并完成：

```text
EnterpriseKnowledge + Request -> Plan -> SQLite Query
```

## 受控输入

所有角色固定使用 `deepseek/deepseek-v4-flash`。对每个 generation 分别记录 plan
commit/origin、Telora revision 与 binary hash、OpenCode/model 配置，以及
`experiment.json`、`opencode.json`、角色文件、语言/CLI 教程、三个 GOAL、两个
DESIGN、DOMAIN 和该代 FEEDBACK 的 hash。只有 generation 输入一致的结果才直接比较；
runtime 更新前后的结果属于同一 execution 中两个不同 epoch。

## 隔离要求

- A1 只读自身设计与源码，不读 ontology 和企业领域。
- A2 只读 A1 的公共教程/契约，不读 A1 源码、私有设计、tests 或 notes。
- A3 只读 A1/A2 公共教程/契约，不读上游源码、私有设计、tests 或 notes。
- coordinator 保留并恢复同一组原生 child session，不补写任务定义。
- `ent-1/FEEDBACK.md` 初始必须为零字节；角色只在各自 GOAL 列出的 marker 出现后
  读取动态输入。
- coordinator 只能创建协议列出的 `control/*` marker，对角色始终发送同一句状态
  变化通知；核对 marker 创建顺序与交付就绪顺序一致。

核对 ACP 事件与归档，确认语言学习并行、A2/A3 的 QueryBuilder 学习并行，且角色
没有提前读取动态输入或通过 glob/grep/命令输出越过边界。

## A1 评估

检查标准 Plan 算子和语义是否封闭稳定；PlanProfile 是否显式收窄能力而不改变语义；
动态值是否只进入 bindings；SQLite transform 是否确定；非法或 profile 越界 Plan
是否产生诊断且无 Query。使用隐藏 Plan 覆盖投影、filter、join、aggregate、group、
sort、limit、绑定顺序和重复转换。

QueryBuilder 先发布草案，由 A2 根据 DESIGN、A3 根据 DOMAIN 独立审查；只有 A1
处理两份反馈并重新验证后，最终公共契约才可供下游实现使用。

## A2 评估

检查公共边界能否精确表达 EnterpriseKnowledge、PlanProfile 和
`EnterpriseKnowledge -> Request -> Plan`。验证能力授权、grain、最短安全路径、
目录序 tie-break、共享边去重、有向环、八边边界、fan-out-only、missing、请求覆盖、
profile 覆盖和失败时不发布部分 Plan。A2 不得定义替代 Plan 或自行生成 SQL。一次
lowering 自然产生的诊断数量只由 Host 隐藏观察，不向角色暴露为验收目标。

## A3 评估

检查具体企业知识是否仅使用公共 API，是否完整表达 DOMAIN 的封闭 vocabulary、能力、
关系、物理 mapping 与 PlanProfile。合法 Request 必须产生覆盖完整的 Plan，并由 A1
得到稳定 SQLite Query；非法 Request 必须产生来源明确的诊断且无可信 Plan/Query。

## 过程与反馈

分别记录三个角色首次语言探索、首次写入、修正轮次、wall time、token、命令失败、
类型擦除、公共泛型复杂度和文档/实现差异。每代 A3 交付后 Host 判断 stop 或发布
下一 generation。只有当前语言能力内仍有明显的 QueryBuilder/eDSL 改进空间时，
才发布批准反馈；语言机制问题仍转为独立 issue。若 Host 在 idle 边界发布新版
binary/教程，必须记录原子发布证据，并把角色变化归属于 runtime epoch，而非自身修正。

批准反馈按 QueryBuilder/eDSL 分区。当前 execution 最多追加一个 generation，并严格
复用原 A1 -> 原 A2 -> 原 A3 session。记录各代隐藏用例差异。

## 归因

根因使用最窄分类：输入契约、QueryBuilder 设计、eDSL 设计、企业建模、教程可发现性、
语言表面、类型系统、标准库、诊断或基础设施。先从实验提取最小重现，再把语言机制
问题独立跟踪；不能因更丰富语言可能避免错误，就把普通算法/API 缺陷归因给语言。
