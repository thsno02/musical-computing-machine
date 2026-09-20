# musical-computing-machine

面向中文读者的 **Clio-style hierarchical semantic clustering** 参考实现。

> 当前状态：**specification-first / RFC 阶段**。先把问题定义、边界、评估协议和隐私承诺讲清楚，再选择具体模型与框架。

## 一句话目标

给定一批非结构化记录（优先支持 AI 对话，也可扩展到工单、反馈、访谈和文档片段），通过可配置的：

```text
facet extraction
  -> embedding
  -> base clustering
  -> contrastive labeling
  -> bottom-up hierarchy
  -> privacy gating
  -> evaluation
```

得到可解释、可下钻、可复现、可比较，并具有明确发布边界的层次化聚类结果。

## 我们复现什么

我们复现 Clio 公开方法中可观察、可验证的系统行为：

- facet 决定语义空间；
- embedding-based base clustering；
- 同时使用 cluster 内样本与邻近反例的 contrastive labeling；
- 把 cluster 当作语义单元逐层构建 parent hierarchy；
- raw / derived / release data 的分区与 privacy gate；
- component-level 与 end-to-end evaluation；
- 稳定、版本化的 tree/map/report artifacts。

这是 **method / behavior reproduction**，不是 Anthropic 内部系统的 exact reproduction。

## 这不是

- Anthropic 产品界面或内部安全运营系统的 pixel-perfect clone；
- 把 UMAP 散点图做出来就宣称“聚类准确”；
- 把 embedding、摘要或聚类当成匿名化；
- 只在 raw points 上反复改变 `k` 就称为 semantic hierarchy；
- 通用 RAG、知识图谱、实时流处理或生产级多租户 SaaS；
- 自动执法、自动封禁或个体绩效判断工具；
- 未经实现和验证的 differential privacy / k-anonymity 保证。

## 规范与讨论

### 核心 RFC

- [RFC 0001：项目章程——我们究竟要复现什么](docs/rfcs/0001-project-charter.md)
- [RFC 0002：评估协议——“准确聚类”究竟是什么意思](docs/rfcs/0002-evaluation-protocol.md)
- [RFC 0003：架构与 Artifact Contract](docs/rfcs/0003-architecture-contracts.md)
- [RFC 0004：隐私 Threat Model 与发布边界](docs/rfcs/0004-privacy-threat-model.md)

### 调研与讨论材料

- [Clio-style 聚类系统生态与本项目定位](docs/landscape.md)
- [RFC 0001 kickoff Discussion 草稿](docs/discussions/0001-kickoff.md)
- [GitHub Discussion 中文 RFC form](.github/DISCUSSION_TEMPLATE/general.yml)

## 最重要的边界

### “层层递增”

本项目所说的 hierarchy 是：

```text
base clusters
  -> propose parents
  -> deduplicate parents
  -> assign children
  -> relabel parents from final children
  -> repeat until a recorded stop condition
```

它不是简单对 raw points 运行多组 `k`，也不是 UMAP 的 zoom level。

### “准确”

不使用一个数字概括全部质量。分别评估：

- facet faithfulness；
- embedding neighborhood quality；
- base cluster quality；
- label faithfulness / specificity / distinctiveness；
- hierarchy parent-child fit；
- end-to-end reconstruction；
- seed/model stability；
- 中文/英文 parity；
- privacy leakage；
- latency/cost。

### “隐私”

项目默认使用 `privacy-oriented` 或“具有 defense-in-depth privacy controls”的表述。

- Raw data、derived artifacts 和 analyst-visible release 必须分区；
- public release 缺少 actor evidence、policy、审计或人工 review 时应 fail closed；
- embedding 与 summary 仍属于可能敏感的 derived data；
- UI 隐藏字段不等于真正的 access control 或 release gate。

## 中文优先

- 文档、RFC、Discussion、CLI 帮助和默认报告以简体中文为主；
- 保留必要且更准确的英文术语，例如 `facet`、`embedding`、`base cluster`、`contrastive examples`、`hierarchy`、`privacy auditor`；
- 代码标识符与公共 API 使用英文；
- 中文不是翻译项，而是独立评估维度：benchmark 必须包含中文、英文和中英混合数据；
- representation language 与 display language 解耦，默认策略由实验决定。

## 建议的推进顺序

1. 接受或修改 RFC 0001–0004；
2. 固化 canonical schema、artifact contract 与 synthetic benchmark；
3. 完成 paper-like vertical slice；
4. 建立中文/英文/混合 evaluation；
5. 接入 alternative clusterer / hierarchy；
6. 再构建 explorer；
7. 用 benchmark 决定默认组件，而不是凭流行度决定。

## 参考资料

- [Anthropic: Clio / Anthropic Insights](https://www.anthropic.com/research/clio)
- [Clio paper](https://arxiv.org/abs/2412.13678)
- [OpenClio](https://github.com/Phylliida/OpenClio)
- [Kura](https://github.com/jxnl/kura)
- [anthropic-clio-impl](https://github.com/adhishthite/anthropic-clio-impl)
- [Toponymy](https://github.com/TutteInstitute/toponymy)
- [BERTopic](https://maartengr.github.io/BERTopic/)
- [TnT-LLM](https://arxiv.org/abs/2403.12173)
- [GraphRAG](https://microsoft.github.io/graphrag/)
