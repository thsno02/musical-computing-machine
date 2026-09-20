# [RFC 0001] 我们究竟要复现什么：Clio-style 层次语义聚类的目标、边界与验收

> **建议分类**：General  
> **状态**：征求意见  
> **目标**：接受、修改或拒绝项目章程；在写主体实现前形成可引用的决策记录。

## TL;DR

我建议把本仓库定义为：

> 一个面向中文读者、provider-agnostic、evaluation-first 的 Clio-style reference implementation：从非结构化记录中抽取分析 facet，基于 embedding 形成 base clusters，利用 contrastive examples 生成可区分标签，再把 cluster 作为语义单元逐层构建 hierarchy，并只发布通过质量与隐私 gate 的聚合结果。

这意味着我们复现的是**公开方法与可观察系统行为**，不是 Anthropic 的内部数据、未公开参数、安全运营流程或产品 UI。

完整提案：

- [RFC 0001：项目章程与边界](../rfcs/0001-project-charter.md)
- [RFC 0002：评估协议](../rfcs/0002-evaluation-protocol.md)
- [RFC 0003：架构与 artifact contract](../rfcs/0003-architecture-contracts.md)
- [RFC 0004：隐私 threat model](../rfcs/0004-privacy-threat-model.md)
- [生态与相邻系统](../landscape.md)

## 为什么先讨论需求，而不是马上选模型

“用 embedding 做聚类”不足以定义这个项目，因为以下选择会改变问题本身：

- embed 原文，还是 embed `request/task/failure_mode` facet？
- 只做 flat clusters，还是构建可解释 parent-child hierarchy？
- label 只看 cluster 内样本，还是同时看最近邻反例？
- 所有点强制归类，还是允许 `unassigned` / noise？
- 中文和英文放在同一语义空间，还是先归一化/翻译？
- 输出给受信 analyst，还是要公开发布？
- “准确”指几何紧凑、人工可解释、taxonomy 恢复，还是稳定性？

如果这些不先落定，很容易做出一个 UI 很像、但语义目标、质量证据和隐私边界都不明确的系统。

## 建议的六条核心不变量

### 1. Facet-conditioned semantic space

同一批数据按 `request`、`task`、`language` 或 `failure_mode` 会形成不同结构。Facet 必须是版本化的一等配置，而不是藏在 prompt 里的字符串。

### 2. 可解释的 base clusters

Base cluster 应内部一致、外部可区分，允许 outlier 和不确定样本；UMAP 的视觉分离不能代替语义评估。

### 3. Contrastive labeling

标签生成同时使用 cluster 内代表样本和最近邻 cluster 的 hard negatives，避免产生宽泛、重复的标题。

### 4. 真正的 bottom-up hierarchy

“层层递增”不是在 raw points 上分别运行 `k=1000/100/10`，而是：

```text
base cluster
  -> parent proposal
  -> parent dedup
  -> child-to-parent assignment
  -> parent relabel from final children
  -> repeat
```

V1 先用 tree：每个 child 最多一个 parent。

### 5. Privacy gate 是发布路径的一部分

Embedding、摘要和聚类都不等于匿名化。Raw、derived、release artifacts 必须分区；public release 在缺少 actor evidence、阈值、审计或人工 review 时应 fail closed。

### 6. 每个 run 可复现、可比较

记录 dataset/config fingerprint、seed、code revision、models、prompts、参数、成本、warnings、quality 和 privacy metrics。

## 建议的 V1 In scope

- Batch pipeline；
- conversation JSONL canonical input；
- 中文、英文、中英混合 benchmark；
- pluggable LLM / embedding / clusterer；
- paper-like KMeans baseline；
- 至少一个 alternative baseline；
- contrastive labels；
- bottom-up hierarchy；
- run manifest、cache、checkpoint、resume；
- quality/stability/privacy evaluation；
- tree/report artifacts；
- 可选轻量 explorer。

## 建议的 V1 Out of scope

- Anthropic UI 的 pixel-perfect clone；
- 实时流处理；
- 生产级 multi-tenant SaaS；
- raw conversation search/review tool；
- 通用 RAG / knowledge graph；
- foundation model training；
- 自动执法、自动封禁或个体绩效判断；
- 未实现的 differential privacy / k-anonymity 声明；
- “适用于所有领域”的固定 ontology；
- 用一个聚类分数代表全部质量。

## “准确”的建议定义

不使用单一指标。至少分别验证：

1. facet faithfulness；
2. embedding neighborhood quality；
3. base cluster quality；
4. label faithfulness / specificity / distinctiveness；
5. parent-child fit 和 sibling duplication；
6. end-to-end taxonomy/distribution reconstruction；
7. seed/model stability；
8. 中文/英文 parity；
9. privacy leakage；
10. latency/cost。

[RFC 0002](../rfcs/0002-evaluation-protocol.md) 给出了一组 provisional gates。它们是工程起点，不是对 Clio 论文数字的直接复制。

## 隐私措辞建议

在完整部署证据出现前，仓库使用：

- `privacy-oriented`；
- “具有 defense-in-depth privacy controls”；
- “在给定 policy 与 benchmark 下减少泄露”。

避免默认声称：

- 匿名化；
- 绝对安全；
- embedding 后无法还原；
- 与 Anthropic 内部系统具有相同保证。

## 需要本帖决定的事项

请对下面每项给出 `接受 / 修改 / 反对`，并说明原因。

### D1. 复现对象

接受“method/behavior reproduction，而不是 exact product reproduction”吗？

### D2. V1 输入边界

建议：核心只保证 conversation JSONL；generic text adapter 作为 experimental。

### D3. 默认 assignment

建议：默认允许低置信样本进入 `unassigned`；另提供 `paper_baseline_forced` 做 ablation。

### D4. Base baseline

建议：先实现 paper-like KMeans pipeline，再增加 density/balanced/hierarchical alternative，而不是一开始争论唯一最优算法。

### D5. 中文策略

建议：默认 report/labels 为 `zh-CN`，code/API 为英文；representation language 通过 benchmark 决定，不强行固定为中文或英文 pivot。

### D6. Public-release mode

建议：V1 包含 contract 和 fail-closed gate；即使 UI 尚未完成，也不能把 privacy 当后续功能。

### D7. Provisional quality gates

是否接受 RFC 0002 的门槛作为 M1/M2 起点，并允许在实验后通过 RFC 调整？

### D8. Explorer 优先级

建议：先稳定 artifacts 与 evaluation，再构建 explorer；M1 不要求 pixel-level 完整 UI。

## 建议的回复模板

```markdown
### 总体意见
接受 / 修改后接受 / 反对

### 各项决定
- D1：
- D2：
- D3：
- D4：
- D5：
- D6：
- D7：
- D8：

### 最大风险

### 建议补充的 use case / non-goal

### 能接受的 M1 最小交付
```

## Decision log

> 在形成共识后，把最终决定写回 RFC，并在此保留日期、决定、理由和 dissent。

| 日期 | 决定 | 理由 | 状态 |
|---|---|---|---|
| 2026-09-20 | 提交初稿 | 先明确目标、边界和验收，再开始实现 | Proposed |
