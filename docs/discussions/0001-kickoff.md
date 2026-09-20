# [RFC 0001] V1 Tracer Bullet：最快跑通 Clio-style 层次聚类

> **建议分类**：General  
> **状态**：方向已接受，细节征求意见  
> **目标**：用最短、最真实的一条纵向链路验证核心产品逻辑，而不是在 V1 建设完备框架。

## TL;DR

V1 的定义是：

> 对一小批中文 AI 对话，只提取 `request` facet，使用一套固定 LLM + embedding + KMeans 配置形成 base clusters；用 representative 与 contrastive examples 生成中文 labels；再对 cluster nodes 重新 embedding，按显式 `k` schedule 逐层聚成 parent nodes；最终输出 `hierarchy.json` 和 `report.md`。

V1 是 **tracer bullet**：必须穿过真实组件，但只走一条路径。它可以硬编码、可以不通用、可以整批重跑；只要能最快回答“这种聚类与层层抽象是否真的有用”。

完整决定：

- [RFC 0001：V1 Tracer Bullet 项目章程](../rfcs/0001-project-charter.md)
- [RFC 0002：V1 最小验收协议](../rfcs/0002-evaluation-protocol.md)
- [RFC 0003：V1 单一路径与最小 Artifact Contract](../rfcs/0003-architecture-contracts.md)
- [RFC 0004：V1 数据与安全边界](../rfcs/0004-privacy-threat-model.md)
- [生态与相邻系统](../landscape.md)——仅作研究参考，不构成 V1 scope。

## 我们要最快验证什么

只有三个核心未知数：

1. 中文对话被概括为 `request` 后，embedding 是否能把相似需求放在一起；
2. cluster 内代表样本加邻近反例，是否能生成准确且可区分的 label；
3. 把 cluster 的 `title + description` 当作新语义单元继续聚类时，是否真的形成更粗一层的 parent concepts。

如果这三件事不成立，完整架构、UI、隐私系统和规模优化都没有意义。

## V1 最短路径

```text
100–300 条中文 demo conversations
  -> 一句 request facet
  -> request embeddings
  -> fixed-k KMeans leaves
  -> representative + contrastive labeling
  -> embed leaf title + description
  -> fixed-k parent clustering
  -> relabel parents
  -> hierarchy.json + report.md
```

示例 config：

```yaml
seed: 42
leaf_k: 8
hierarchy_k: [3, 1]
representatives: 5
contrastive: 3
```

这不是最终默认参数，只是让第一颗 tracer bullet 穿过整条链路。

## V1 In scope

- 单一 conversation JSONL；
- 单一中文 synthetic / public demo；
- 单一 `request` facet；
- 一套可工作的 LLM 与 embedding；
- 一种 KMeans baseline；
- leaf labels；
- 至少一层 parent hierarchy；
- JSON / Markdown artifacts；
- 最小 ARI/NMI 与 hierarchy invariants；
- 一次轻量人工 label / parent review；
- run config、seed、model、time、cost 记录。

## V1 Out of scope

- 通用 adapter、plugin system 和稳定 API；
- 多 provider、local/cloud 双栈；
- alternative clusterer、自动选 `k` 和全面 ablation；
- `unassigned`、noise handling 和复杂 repair；
- 多语言 benchmark；
- cache、checkpoint、resume、streaming 和大规模运行；
- UMAP、interactive explorer 和 UI clone；
- 私有生产数据、public-release system 和完整 privacy auditor；
- 形式化 anonymity / differential privacy；
- 生产级 SaaS、RBAC、RAG、knowledge graph 或自动处置。

这些不是“永远不做”，而是不能阻塞第一条真实链路。

## Definition of Done

- 一条命令完成所有 stage；
- 没有 mock cluster 或 mock hierarchy；
- 每条记录恰好进入一个 leaf，并能沿 parent path 到 root；
- 至少两层语义结构，node count 逐层减少，无 cycle/orphan；
- demo leaf clustering 达到最低 ARI/NMI sanity floor；
- 至少 80% leaf labels 通过 faithfulness / specificity / distinctiveness review；
- 至少 80% parent nodes 通过 fit review；
- `report.md` 默认不含原始对话；
- 运行信息足以复盘下一步该优化哪里。

## 当前需要决定的事项

### D1. Demo 数据

建议：先做 `100–300` 条中文 synthetic conversations，约 `6–12` 个 leaf topics、`2–4` 个 parent topics。

要决定：主题设计和数据量，而不是先找大规模真实数据。

### D2. 单一模型栈

建议：选择一套当前最容易跑通、中文效果足够的 LLM + embedding。

要决定：第一套工作配置。V1 不要求 provider abstraction。

### D3. 显式 `k` schedule

建议：demo config 直接写 `leaf_k` 和 `hierarchy_k`，例如 `8 -> 3 -> 1`。

要决定：第一份 demo 的 schedule。自动选 `k` 后移。

### D4. Label sample 数量

建议：每个 cluster 取 `5` 个 representatives 和 `3` 个 contrastive examples。

要决定：是否足够覆盖主要语义，同时保持 prompt 简单。

### D5. 最低 sanity floor

建议：ARI `>= 0.45`、NMI `>= 0.60`、top-level accuracy `>= 0.75`，leaf/parent 人工 pass rate `>= 0.80`。

这些只是发现明显失败的下限，不是产品 KPI。

### D6. 输出

建议：V1 只做 `hierarchy.json` 与 `report.md`，不做 UI。

要决定：Markdown tree 是否足够让我们判断下一步。

## 暂时不要在本帖解决

除非它直接阻塞 tracer bullet，否则先不要展开：

- 哪个 clusterer 长期最好；
- 完整 component interface；
- 任意数据 schema；
- 多语言 parity；
- 生产 privacy architecture；
- explorer 技术栈；
- 大规模性能；
- SaaS 与权限系统。

这些讨论应由第一份真实 artifacts 和 failure report 驱动。

## 建议回复模板

```markdown
### 总体意见
接受 / 修改后接受 / 反对

### Tracer bullet 选择
- D1 Demo 数据：
- D2 单一模型栈：
- D3 k schedule：
- D4 sample 数量：
- D5 sanity floor：
- D6 输出：

### 这条链路最可能先失败在哪里？

### 哪一项可以进一步删掉？

### 哪一项虽然重要，但应明确后移？
```

## Decision log

| 日期 | 决定 | 理由 | 状态 |
|---|---|---|---|
| 2026-09-20 | 初稿采用 specification-first 完整边界 | 试图一次定义 target-state | Superseded |
| 2026-09-20 | V1 改为 tracer bullet | 最快最简验证 embedding clustering 与 recursive hierarchy | Accepted |
| 2026-09-20 | 多 backend、完整 evaluation/privacy/UI 后移 | 不应阻塞第一条真实链路 | Accepted |
