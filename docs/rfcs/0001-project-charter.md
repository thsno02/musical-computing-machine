# RFC 0001：V1 Tracer Bullet 项目章程

- **状态**：Accepted for V1
- **日期**：2026-09-20
- **讨论入口**：GitHub Issue #2
- **相关 RFC**：
  - [RFC 0002：V1 最小验收协议](0002-evaluation-protocol.md)
  - [RFC 0003：V1 单一路径与最小 Artifact Contract](0003-architecture-contracts.md)
  - [RFC 0004：V1 数据与安全边界](0004-privacy-threat-model.md)

## 0. 决定

V1 采用 **tracer bullet** 策略。

它不是缩小版“完整平台”，也不是先把未来所有 interface、benchmark、privacy mode 和 deployment contract 设计齐全。它只负责用最短、最真实的纵向链路穿过核心系统：

```text
raw conversations
  -> one facet
  -> embeddings
  -> base clusters
  -> readable labels
  -> fewer semantic parent clusters
  -> hierarchy artifact
```

V1 的价值在于尽快暴露核心未知数，而不是证明仓库已经具备生产完备性。

## 1. 为什么改成 tracer bullet

上一版章程把 target-state 与 V1 混在了一起，导致 V1 同时承担：

- 多 backend abstraction；
- 多算法 baseline；
- 完整 benchmark；
- 多语言 parity；
- privacy release system；
- cache / checkpoint / resume；
- explorer；
- 大规模运行。

这些都可能有价值，但它们不能在我们尚未跑出第一棵可信 hierarchy 时成为前置条件。

当前最大的产品与技术风险只有三个：

1. 中文对话经 `request` facet 后，embedding 是否能形成有意义的邻域和 base clusters；
2. cluster label 是否足够准确、具体、可区分；
3. 把已命名 cluster 当作新语义单元继续聚类时，能否产生真正“更粗一层”的结构，而不是随机分组或同义词堆叠。

V1 必须直接回答这三个问题。

## 2. Tracer bullet 的含义

### 2.1 必须是真的

Tracer bullet 可以很窄，但不能是假链路。

V1 必须调用真实的 facet extractor、真实 embedding、真实 clustering 和真实 labeler，并生成真实 artifacts。不得用手写 cluster、固定输出或 mock hierarchy 绕过核心风险。

### 2.2 可以不完备

以下做法在 V1 中是允许的：

- 只支持一个输入 schema；
- 只支持一个 facet；
- 只支持一套模型配置；
- 使用显式 `k` 和 hierarchy schedule；
- pipeline 写成顺序执行的单体模块；
- 配置和 artifact schema 尚未稳定；
- 失败后重跑整个小数据集；
- 只对 synthetic / public demo data 工作。

### 2.3 不是 throwaway mock

实现应尽量保留可演进的代码质量，但不为尚未出现的第二个 use case 提前抽象。

原则是：

> 第一个实现先跑通；第二个实现出现时再提取 interface。

## 3. V1 唯一用户故事

> 作为一个中文读者/研究者，我把仓库内置的一小批中文 AI 对话交给 CLI；系统提取每条对话的主要 request，通过 embedding 形成 base clusters，生成可读标签，再把这些 cluster 逐层归并为更粗的 parent concepts，最终输出一棵可检查的中文 hierarchy 和简短报告。

V1 不承诺任意领域、任意私有数据或任意规模。

## 4. V1 单一路径

### 4.1 输入

- `100–300` 条中文 synthetic 或明确可公开的 AI conversations；
- 单一 JSONL schema；
- 每条记录只要求稳定 `id` 与 `messages`；
- demo 数据具有已知 leaf topic 和 parent topic，仅用于 sanity check。

### 4.2 Facet

只实现一个 facet：

```text
request = 用户希望助手完成什么？
```

输出是一句简洁中文描述。V1 不实现 `task`、`language`、`failure_mode`、safety score 或自定义 facet registry。

### 4.3 Representation 与 embedding

- 只 embed `request` facet，不 embed 完整对话；
- 只选一套可工作的 embedding 模型；
- 不建立 provider abstraction；
- 记录模型名、维度和输入文本。

### 4.4 Base clustering

- 使用一个简单 baseline，例如 KMeans；
- `leaf_k` 由 demo config 显式指定；
- 固定 seed；
- V1 可以强制每条记录进入一个 cluster；
- 自动选 `k`、noise detection 和 `unassigned` 后移。

### 4.5 Cluster labeling

每个 leaf cluster 使用：

- 若干个最接近 centroid 的代表 facet；
- 若干个最近邻外部样本作为 contrastive examples；
- 一个固定 prompt 生成中文 `title + description`。

V1 不做多 prompt 比较、judge model、复杂 sampling 或 label repair。

### 4.6 Hierarchy

V1 使用最短可工作的递归方案：

1. 将每个 child node 表示为 `title + description`；
2. 对 child nodes 重新 embedding；
3. 按 config 中显式的 `hierarchy_k` schedule 聚成更少的 parent groups；
4. 根据最终 children 生成 parent `title + description`；
5. 重复到 schedule 结束。

示例：

```yaml
leaf_k: 8
hierarchy_k: [3, 1]
```

这足以验证“cluster 作为语义单元继续上卷”的产品逻辑。自动 parent proposal、dedup、动态 stop policy 和 DAG 不属于 V1。

### 4.7 输出

V1 只输出：

- `run.json`；
- `facets.jsonl`；
- `leaf_assignments.jsonl`；
- `nodes.jsonl`；
- `hierarchy.json`；
- `report.md`。

不要求数据库、网页、UMAP 或 interactive explorer。

## 5. V1 Definition of Done

V1 完成需要同时满足：

1. 一条命令从 demo JSONL 跑到最终 report；
2. 核心链路没有 mock stage；
3. hierarchy 至少包含 leaf 与 parent 两个语义层级；
4. 每条输入记录恰好属于一个 leaf，并可追溯到 root；
5. node count 逐层减少，没有 cycle 或 orphan；
6. 通过 RFC 0002 的 clustering 和人工 label sanity checks；
7. 保存 config、seed、模型、耗时和可得成本；
8. report 默认只包含聚合标签与数量，不包含原始对话；
9. README 能让新贡献者理解如何运行、看结果和定位失败。

这组标准证明的是“方向值得继续”，不是“系统已经可生产使用”。

## 6. V1 In scope

- 中文 demo conversation JSONL；
- 单一 `request` facet；
- 一套 LLM + embedding stack；
- 一个 base clusterer；
- 简单 representative / contrastive sampling；
- leaf labels；
- 至少一层 parent hierarchy；
- JSON + Markdown artifacts；
- 最小自动 sanity metrics；
- 一次轻量人工 review；
- 基本运行 provenance。

## 7. V1 Out of scope

下列内容不得阻塞 V1：

- 通用 `SourceAdapter`、facet registry 和 plugin framework；
- 多 provider、local/cloud 双栈或模型 router；
- alternative clusterer、自动选 `k`、HDBSCAN、balanced clustering；
- `unassigned`、复杂 confidence 和 repair stage；
- 多语言 benchmark 与中文/英文 parity；
- 300–500 条人工 gold set、multi-seed stability 和全面 ablation；
- cache、checkpoint、resume、streaming 和 distributed execution；
- `10^4+` scale promise；
- UMAP、静态网站、完整 explorer；
- 私有生产数据和 public-release mode；
- 完整 privacy auditor、正式 anonymity 或 differential privacy；
- 生产级 RBAC、multi-tenancy、billing 和 ops；
- RAG、knowledge graph、搜索、实时路由或自动处置；
- 稳定公共 API 和向后兼容承诺。

## 8. Scope creep 判断规则

任何 V1 新需求都要回答：

> 如果不做这件事，我们是否无法判断 facet embedding、base clustering 或 recursive hierarchy 是否可行？

- 如果答案是“是”，可以进入 V1；
- 如果答案是“只是未来会需要”“更优雅”“更通用”“更安全地公开发布”，进入 post-V1 backlog。

## 9. V1 之后再决定什么

Tracer bullet 运行后，根据实际 failure mode 决定下一步，而不是预先承诺路线：

- cluster 不准：优先研究 facet、embedding、`k` 或 clusterer；
- label 宽泛：优先研究 sampling、contrastive context 或 prompt；
- hierarchy 失真：优先研究 parent proposal / assignment / relabel；
- 中文效果明显差：建立 multilingual evaluation；
- 真实数据需求出现：再设计 privacy、provider 和 release contract；
- 结果有用但难浏览：再构建 explorer；
- 重跑成本成为瓶颈：再加 cache/checkpoint/resume。

## 10. 决策记录

| 日期 | 决定 | 理由 |
|---|---|---|
| 2026-09-20 | V1 改为 tracer bullet | 最快、最简地验证核心链路，不以完备性为目标 |
| 2026-09-20 | V1 只保证中文 demo、一套 stack、一个 facet、一个 clusterer | 避免在获得第一份证据前做过早抽象 |
| 2026-09-20 | 完整 evaluation、privacy、multi-provider 和 explorer 后移 | 它们不是回答当前核心风险的必要条件 |
