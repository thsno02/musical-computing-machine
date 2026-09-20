# Clio-style 聚类系统生态与本项目定位

- **调研日期**：2026-09-20
- **目的**：理解 Clio 之后已有的实现与相邻方法，决定“借什么、不借什么”，而不是重复造一个没有评估标准的 demo。

## 1. 先说结论

本项目不应盲目 fork 某一个现有实现，也不应把“用了更新的 embedding / LLM”当作创新。

更有价值的定位是：

> 建立一个中文优先、provider-agnostic、evaluation-first 的 Clio-style reference system，用明确 component/artifact contract 比较不同实现，并把 hierarchy 与 privacy 的边界讲清楚。

可借鉴的方向：

- 从 OpenClio 借鉴 local-first、静态 explorer 和 facet 可配置性；
- 从 Kura 借鉴清晰的 procedural API、cache/checkpoint 与 pipeline 拆分；
- 从 anthropic-clio-impl 借鉴 run artifacts、strict/resume、privacy/eval 阶段化；
- 从 Toponymy 借鉴 balanced hierarchy、topic naming 和可缩放 semantic map；
- 把 BERTopic、agglomerative、HDBSCAN 等作为可替换 baseline；
- 把 GraphRAG、TnT-LLM 视作相邻问题，而不是本项目默认架构。

## 2. 原始参考：Anthropic Clio / Anthropic Insights

### 核心路径

```text
raw conversations
  -> facet extraction
  -> facet embeddings
  -> large-k base clustering
  -> contrastive cluster labels
  -> bottom-up semantic hierarchy
  -> privacy thresholds/auditor
  -> tree/map exploration
```

### 真正值得复现的部分

- 分析目标由 facet 显式定义；
- bottom-up discovery，而不是先写死 taxonomy；
- cluster label 使用邻近反例来提高区分度；
- hierarchy 不是简单多档 `k`，而是 parent proposal、dedup、assignment、relabel 的迭代；
- 原始数据与 analyst-visible aggregate 之间有 defense-in-depth；
- component-level 与 end-to-end evaluation 并存。

### 不可假装复现的部分

- 未公开的精确 `k`、阈值、prompt、内部数据与审核流程；
- 内部权限和账号级安全调查；
- 产品 UI 的完整行为；
- 特定论文数字在不同数据/模型上的原样复现。

## 3. OpenClio

项目：`Phylliida/OpenClio`

### 观察到的设计

- 使用本地 LLM/VLLM 和 SentenceTransformers；
- 支持 facet 配置；
- 输出静态网站；
- 提供 hierarchy/tree、conversation view 和 UMAP；
- 前端按需加载分块压缩数据；
- URL hash 保存视图状态；
- 可处理 conversation 之外的数据，只要提供适当 facet/prompt。

### 值得借鉴

- local-first 和 provider independence；
- explorer 与 pipeline 的可运行完整度；
- facet 作为可扩展 abstraction；
- 静态 artifact 可以脱离服务端部署。

### 不直接等同于本项目目标

- 有 UI 不代表 quality/privacy contract 已完整；
- 静态网站如果包含 raw data chunks，需要单独定义 public/private release 边界；
- 需要补齐统一 benchmark、中文质量和多实现 comparison；
- 项目是否安全取决于部署和输出 policy，不能由 local model 自动推出。

## 4. Kura

项目：`jxnl/kura`

### 观察到的设计

- Python procedural API；
- conversation summarization；
- base clustering（README 示例为 MiniBatch KMeans）；
- meta-cluster / hierarchy；
- HDBUMAP projection；
- cache 与 JSONL checkpoint；
- terminal visualization；
- 面向 batch analytics，不以 real-time 为主。

### 值得借鉴

- 组件化 API；
- 对工程用户友好的 pipeline；
- cache/checkpoint；
- 便于独立替换 summarizer、clusterer 和 meta-clusterer；
- 适合作为 baseline adapter 的候选。

### 不直接等同于本项目目标

- 需要验证 hierarchy 是否满足本项目定义的 propose/dedup/assign/relabel contract；
- 需要独立评估 contrastive label 的必要性；
- README 中的产品价值叙事不能代替 benchmark；
- “privacy-first”需要 threat model、release gate 和 stage-level evidence。

## 5. anthropic-clio-impl / ant-clio

项目：`adhishthite/anthropic-clio-impl`

### 观察到的设计

- terminal-first CLI；
- input validation；
- facet extraction、embeddings、base clustering、hierarchy；
- privacy audit/gating；
- synthetic evaluation；
- run manifest、events、metrics、warnings；
- resume 与 fingerprint guard；
- 支持 KMeans、HDBSCAN 和 hybrid strategy；
- 可选 Next.js UI。

### 值得借鉴

- run-oriented architecture；
- artifact provenance；
- strict/fail-on-warning 语义；
- resume 前检查输入/config drift；
- privacy/evaluation 不是 UI 后置功能；
- 多 clustering strategy。

### 不直接等同于本项目目标

- 默认 provider/model 选择不应成为本项目 contract；
- 需要中文、多语言 parity 和独立 evaluation rubric；
- 需要区分“实现了 auditor”与“满足 public-release”；
- 仍需通过统一 benchmark 验证每种 cluster strategy。

## 6. Toponymy

项目：`TutteInstitute/toponymy`

### 核心问题

Toponymy 面向 large text collections 的 topic naming 与层次化探索，强调：

- balanced hierarchical clustering；
- LLM topic naming；
- layered topic structures；
- scalable semantic maps / DataMapPlot。

### 与本项目的关系

它是很强的相邻实现：比“只做 flat clustering”更接近我们需要的层次化产品形态，也提供了 hierarchy 和 naming 的替代实现。

### 边界

- 本项目的核心输入 abstraction 是 facet-conditioned records；
- Clio-style contrastive label 和 privacy release gate 仍需单独验证；
- Toponymy 可作为 alternative hierarchy/naming backend，而不是自动替代整个 pipeline。

## 7. BERTopic

### 核心路径

典型 BERTopic 使用 transformer embeddings、降维、density-based clustering 和 class-based TF-IDF 生成 topics，并支持 topic hierarchy。

### 值得作为 baseline 的原因

- 成熟、广泛使用；
- 容易建立 flat topic 与 hierarchical topic baseline；
- 可提供 keyword-oriented explanation；
- 可与 facet embedding 共用输入进行公平比较。

### 为什么不等同于 Clio-style system

- 默认目标是 topic modeling，不是完整的 facet→contrastive label→semantic hierarchy→privacy gate；
- hierarchy 通常基于 topic representation 的 linkage，不必然包含 LLM parent proposal 和 relabel；
- 不默认解决 analyst-visible release boundary。

## 8. TnT-LLM

### 核心问题

TnT-LLM 关注 text mining at scale：从未标注文本中生成 taxonomy，并进一步进行 label assignment/classification。

### 与本项目的关系

它适合作为 taxonomy induction 的相邻 baseline，尤其可回答：

- 先诱导 taxonomy 再分类，是否比持续 bottom-up clustering 更稳定？
- 固定 taxonomy 对时间比较是否更友好？

### 边界

本项目默认优先发现数据中的 bottom-up structure，而不是先固化一个长期 classifier taxonomy。两者可以组合，但需要独立 RFC 定义 taxonomy freeze、version migration 和 unknown class。

## 9. GraphRAG

### 核心路径

GraphRAG 从文档抽取 entities/relations，构建图，进行 hierarchical community detection，并生成 community reports 支持 retrieval 和 global questions。

### 值得借鉴

- 多层 community reports；
- global-to-local navigation；
- provenance；
- 面向大规模语料的分层 summary。

### 为什么是相邻问题而非默认方案

- 核心对象是 entity-relation graph，而本项目核心对象是 facet-conditioned records；
- 目标主要是 retrieval/answering，不是估计行为/请求 pattern 的 prevalence；
- graph extraction 引入另一组错误和成本；
- 对 conversation clustering 而言，图并非必要前提。

## 10. 关键设计分歧矩阵

| 系统/方法 | 输入 abstraction | Base grouping | Labeling | Hierarchy | Privacy release gate | 本项目角色 |
|---|---|---|---|---|---|---|
| Clio / Insights | facets from conversations | embedding + clustering | LLM + contrastive examples | semantic bottom-up | 核心组成 | 参考方法 |
| OpenClio | configurable facets | embedding clustering | LLM | tree | 需按部署验证 | 可运行实现参考 |
| Kura | summaries | KMeans/default alternatives | LLM | meta-cluster | 需补 threat model | API/baseline adapter |
| ant-clio | facets | KMeans/HDBSCAN/hybrid | LLM | multi-level | 有 audit/gating 阶段 | run architecture 参考 |
| Toponymy | text/topic representations | balanced hierarchy | LLM topic names | layered | 非核心 | alternative backend |
| BERTopic | text embeddings | density/topic modeling | c-TF-IDF/optional LLM | linkage | 非核心 | classical baseline |
| TnT-LLM | raw text | taxonomy induction + classification | taxonomy labels | taxonomy | 非核心 | taxonomy baseline |
| GraphRAG | entity-relation graph | graph communities | community reports | Leiden hierarchy | 非核心 | 相邻系统 |

## 11. 本项目的差异化主张

不是“又一个 Clio clone”，而是以下四点组合：

### 11.1 Specification-first

先定义什么叫 facet、cluster、hierarchy、release artifact，再写实现。

### 11.2 Evaluation-first

每个 backend 都在同一 benchmark 和 artifact contract 下比较；不靠 demo screenshot 决胜。

### 11.3 Chinese-first

中文、英文、中英混合是 primary benchmark，不是最后补一组翻译样例。

### 11.4 Boundary-first

明确区分：

- clustering vs taxonomy classification；
- semantic hierarchy vs repeated raw-point clustering；
- exploration UI vs quality evidence；
- privacy-oriented controls vs formal privacy guarantee；
- aggregate research vs individual enforcement。

## 12. 建议的 V1 implementation sequence

1. 先实现 contracts、synthetic benchmark 和 paper-like vertical slice；
2. 把 KMeans + contrastive label + two-level semantic hierarchy 跑通；
3. 加入中文/英文/混合 evaluation；
4. 接入一个 alternative clusterer；
5. 接入 Toponymy/BERTopic/Kura 中至少一个 adapter 或对照实现；
6. 再做 explorer；
7. 最后决定默认组件，而不是一开始凭流行度决定。

## 13. 参考链接

- Anthropic Clio / Anthropic Insights：<https://www.anthropic.com/research/clio>
- Clio paper：<https://arxiv.org/abs/2412.13678>
- OpenClio：<https://github.com/Phylliida/OpenClio>
- Kura：<https://github.com/jxnl/kura>
- anthropic-clio-impl：<https://github.com/adhishthite/anthropic-clio-impl>
- Toponymy：<https://github.com/TutteInstitute/toponymy>
- BERTopic：<https://maartengr.github.io/BERTopic/>
- TnT-LLM：<https://arxiv.org/abs/2403.12173>
- GraphRAG：<https://microsoft.github.io/graphrag/>
