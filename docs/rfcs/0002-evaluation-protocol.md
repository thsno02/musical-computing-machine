# RFC 0002：V1 最小验收协议

- **状态**：Accepted for V1
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)

## 0. 目的

V1 的验证目标不是建立完整 benchmark program，而是回答：

> 这条最短 pipeline 是否真的能把一小批中文对话变成一棵大体正确、可读、可追溯的多层聚类树？

因此，本 RFC 只定义 tracer bullet 的 smoke criteria。它不证明产品质量，不支持跨模型排名，也不构成隐私、安全或规模保证。

## 1. Demo 数据集

V1 使用一份小型、版本化的中文数据集：

- `100–300` 条 AI conversations；
- 优先 synthetic，或使用明确许可的公开数据；
- 约 `6–12` 个已知 leaf topics；
- 约 `2–4` 个已知 parent topics；
- 包含同义改写和少量 hard negatives；
- 不包含真实敏感信息；
- 每条记录具有 `gold_leaf` 和 `gold_parent`，仅用于 sanity check。

数据集应足够小，使失败后整条 pipeline 可以直接重跑；也应足够多，使 embedding 和 clustering 不是对十几条样本的玩具展示。

## 2. 自动验收

### 2.1 Pipeline completion

必须满足：

- 输入 schema 全部通过；
- facet extraction、embedding、base clustering、labeling、hierarchy 和 rendering 全部执行；
- 没有使用预写 cluster 或 mock hierarchy；
- 失败时命令返回非零状态并保留错误信息。

### 2.2 Record coverage

V1 使用 forced assignment，因此：

- 每条记录恰好属于一个 leaf cluster；
- 不得静默丢失、重复或新增记录；
- `input_count == assigned_count`；
- 每个 assignment 可追溯到 `record_id`。

### 2.3 Base clustering sanity

在已知 leaf labels 的 demo 数据上至少报告：

- Adjusted Rand Index（ARI）；
- Normalized Mutual Information（NMI）；
- cluster size distribution。

V1 的最低 sanity floor：

| Metric | Floor |
|---|---:|
| ARI | `>= 0.45` |
| NMI | `>= 0.60` |

这些数字只用于阻止“pipeline 跑完但 clustering 基本随机”的结果。它们不是最终质量门槛，也不用于宣称复现了论文指标。

### 2.4 Hierarchy invariants

自动检查：

- 至少有 leaf 与 parent 两层；
- 每个 non-root node 恰好一个 parent；
- 没有 cycle；
- 每个 leaf 都能到达 root；
- 每层 node count 严格减少；
- parent 的 `member_count` 等于 descendants 的聚合；
- 没有 orphan node。

### 2.5 Parent-level sanity

把 discovered parent 映射到其 children 中占多数的 `gold_parent`，报告 majority-mapped accuracy。

V1 floor：

```text
top_level_accuracy >= 0.75
```

这只是快速检测 hierarchy 是否与已知粗粒度结构完全背离。

### 2.6 Provenance

`run.json` 至少包含：

- code revision；
- input fingerprint；
- LLM model；
- embedding model；
- `leaf_k`；
- `hierarchy_k`；
- seed；
- prompt version；
- wall-clock time；
- 可得的 token / cost；
- stage status。

## 3. 一次轻量人工检查

V1 不建立多人标注流程。由维护者完成一次结构化 review，并把结果提交到 `report.md`。

### 3.1 Leaf label review

如果 leaf clusters 不超过 20 个，则检查全部；否则随机检查 20 个。

每个 label 回答三个 yes/no 问题：

1. **Faithful**：label 是否由 cluster 内主要样本支持？
2. **Specific**：是否比“技术问题”“写作相关”更具体？
3. **Distinct**：是否能与最近邻 cluster 区分？

一个 label 三项都为 yes 才算 pass。

V1 floor：

```text
leaf_label_pass_rate >= 0.80
```

### 3.2 Parent review

检查全部 parent nodes：

1. children 是否大体属于这个 parent？
2. parent 是否比 children 更抽象，而不是简单复制某个 child？
3. sibling parents 是否明显重复？

V1 floor：

```text
parent_fit_pass_rate >= 0.80
```

### 3.3 中文可读性

维护者确认：

- 标题和描述是自然中文；
- 没有明显机翻式表达；
- 英文术语只在必要时保留；
- 同义 cluster 没有因为中文措辞差异而被误认为不同概念。

V1 不要求正式 locale benchmark。

## 4. 运行成本

每次 demo run 记录：

- wall-clock time；
- LLM calls / tokens；
- embedding calls；
- estimated cost（可得时）；
- stage-level duration。

V1 不设置成本门槛。记录这些数字的目的，是决定下一个最值得优化的环节。

## 5. 失败如何处理

任一 floor 未通过时：

1. 仍保存 artifacts 和 report；
2. 明确标记 run 为 `failed_sanity`；
3. 列出最可能的失败阶段；
4. 只修最靠前、最有解释力的一个问题；
5. 不通过增加框架层、更多 backend 或更复杂 UI 来掩盖质量问题。

典型决策：

- ARI/NMI 低：先检查 facet、embedding 和 `leaf_k`；
- leaf labels 差：检查 representative / contrastive samples 和 prompt；
- parent fit 差：检查 node representation、`hierarchy_k` 和 parent relabel；
- 成本太高：再考虑 cache/batching，而不是预先实现完整 job system。

## 6. 明确后移的评估工作

以下都不是 V1 blocker：

- 300–500 条人工 gold set；
- 多位 annotator 与 adjudication；
- 多语言 parity；
- translation invariance；
- multi-seed stability；
- 跨 model/provider comparison；
- alternative clusterer ablation；
- B-cubed、pairwise F1、dendrogram purity 等完整指标矩阵；
- LLM-as-judge program；
- privacy canary benchmark；
- `10^4+` scale / performance benchmark；
- confidence interval 和长期 regression dashboard。

只有 tracer bullet 证明链路有价值后，才根据实际 failure mode选择其中最需要的一项。

## 7. V1 通过意味着什么

通过本 RFC 只意味着：

- pipeline 是真实的；
- base clustering 不是明显随机；
- hierarchy 结构合法；
- 大部分标签和 parent 对人类是可理解的；
- 下一轮工程投入有了可观察依据。

它不意味着：

- 对真实生产数据同样有效；
- 自动发现的 taxonomy 是唯一正确答案；
- 对多语言、罕见主题或长尾样本可靠；
- 结果具备隐私或匿名化保证；
- 系统已达到 Clio 的内部质量。
