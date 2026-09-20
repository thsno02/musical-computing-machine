# musical-computing-machine

面向中文读者的 **Clio-style hierarchical semantic clustering** 实验项目。

> **V1 是 tracer bullet，不是完备系统。** 目标是用最少代码、最少组件和最小数据集，尽快跑通一条真实的端到端路径，验证“facet embedding 能否形成有用聚类，以及 cluster 能否逐层抽象成可读 hierarchy”。

## V1 要验证的唯一问题

给定一小批中文 AI 对话，能否通过一条固定 pipeline：

```text
conversation JSONL
  -> request facet
  -> embedding
  -> base clustering
  -> cluster labeling
  -> cluster-node embedding
  -> fewer parent clusters
  -> parent labeling
  -> hierarchy.json + report.md
```

得到一棵至少两层、能够人工理解和追溯的聚类树？

这条路径必须使用真实的 facet extraction、embedding、clustering 和 labeling；可以很窄、很朴素，甚至包含硬编码配置，但不能用 mock 结果绕过核心风险。

## V1 的单一路径

- **输入**：仓库内置的 `100–300` 条中文 synthetic / 明确可公开对话；
- **Facet**：只提取一个 `request` facet；
- **模型栈**：只支持一套能工作的 LLM 与 embedding 配置；
- **Base clustering**：只实现一种确定性 baseline，例如固定 `k` 的 KMeans；
- **Labeling**：给模型少量 cluster 内代表样本和邻近 cluster 的 contrastive examples；
- **Hierarchy**：把 `title + description` 重新 embedding，按显式的 `k` schedule 聚成更少的 parent nodes，并重新命名；
- **输出**：JSON artifacts 和中文 Markdown report；
- **运行方式**：一条命令从输入跑到最终 hierarchy。

V1 不解决自动选 `k`、算法最优性、通用 schema、任意 provider、任意数据规模或生产部署。

## V1 完成条件

V1 完成只看下面几件事：

1. 一条命令能在内置中文数据集上完整运行；
2. 每条记录都能追溯到 leaf cluster 和 ancestor path；
3. hierarchy 至少包含 leaf 和 parent 两个语义层级，节点数量逐层减少且没有 cycle；
4. 在已知主题的 demo 数据上通过最低 clustering sanity check；
5. leaf / parent labels 经一次轻量人工检查，大部分是准确、具体且彼此可区分的；
6. 保存实际配置、模型名、seed、耗时和成本，便于复盘；
7. report 默认不包含原始对话文本。

详细口径见 [RFC 0002](docs/rfcs/0002-evaluation-protocol.md)。这些只是 tracer bullet 的 smoke criteria，不是产品质量保证。

## V1 明确不做

- 多 provider、可插拔 backend 或稳定公共 API；
- alternative clusterer、自动调参和大规模 benchmark；
- 中文/英文/多语言 parity；
- cache、checkpoint、resume、分布式执行和 `10^5+` 扩展性；
- UMAP、完整 explorer 或 Anthropic UI 的复刻；
- 私有生产数据接入、public-release pipeline 或正式隐私保证；
- differential privacy、k-anonymity 或“embedding 等于匿名化”的声明；
- 实时流处理、multi-tenant SaaS、RAG、knowledge graph 或自动处置。

这些内容只有在 tracer bullet 暴露出真实需求后，才进入后续版本。

## 为什么这样切

最需要尽快获得证据的，不是框架是否优雅，而是：

- 中文 `request` facet 的 embedding 邻域是否有意义；
- base clusters 是否能被清楚命名；
- 对 cluster nodes 再聚类时，语义是否真的逐层变粗；
- contrastive examples 是否改善标签区分度；
- 哪一步最先失真，以及值得把工程复杂度花在哪里。

因此，V1 允许单体 pipeline、固定模型和显式 `k`。只有出现第二种实现需求时，才抽象 interface；只有有了第一份真实结果时，才设计完整 evaluation、privacy 和 explorer。

## 文档

- [RFC 0001：V1 Tracer Bullet 项目章程](docs/rfcs/0001-project-charter.md)
- [RFC 0002：V1 最小验收协议](docs/rfcs/0002-evaluation-protocol.md)
- [RFC 0003：V1 单一路径与最小 Artifact Contract](docs/rfcs/0003-architecture-contracts.md)
- [RFC 0004：V1 数据与安全边界](docs/rfcs/0004-privacy-threat-model.md)
- [Kickoff / Decision Discussion](docs/discussions/0001-kickoff.md)
- [生态与相邻系统调研](docs/landscape.md)——研究材料，不构成 V1 scope。

## 语言约定

- 文档、Discussion、CLI 帮助和默认报告以简体中文为主；
- 保留 `facet`、`embedding`、`cluster`、`contrastive examples`、`hierarchy` 等更准确的英文术语；
- 代码标识符使用英文；
- V1 只保证中文 demo。多语言能力在 tracer bullet 之后再决定。

## 参考资料

- [Anthropic: Clio / Anthropic Insights](https://www.anthropic.com/research/clio)
- [Clio paper](https://arxiv.org/abs/2412.13678)
- [OpenClio](https://github.com/Phylliida/OpenClio)
- [Kura](https://github.com/jxnl/kura)
- [anthropic-clio-impl](https://github.com/adhishthite/anthropic-clio-impl)
