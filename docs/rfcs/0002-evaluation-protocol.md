# RFC 0002：评估协议——“准确聚类”究竟是什么意思

- **状态**：Draft
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)

## 摘要

本项目拒绝用一个数字概括系统质量。

Clio-style pipeline 至少包含 facet extraction、semantic representation、base clustering、cluster labeling、hierarchy、privacy gating 和 end-to-end reconstruction。每个阶段可能单独失败，也可能彼此抵消；因此，“准确聚类”必须被定义为一组可测量、可诊断的 properties，而不是某个截图、silhouette score 或 LLM 主观评价。

本 RFC 定义 benchmark 分层、指标、人工 rubric、ablation、stability protocol 和 V1 provisional gates。

## 1. 评估原则

### 1.1 指标必须对应决策

每个指标必须回答一个明确问题：

- facet 是否保留了分析任务需要的语义？
- embedding 的局部邻域是否符合人类判断？
- base cluster 是否内部一致、彼此可区分？
- label 是否准确、具体且能区分邻居？
- hierarchy 的 parent 是否真正概括 children？
- 结果对随机种子、模型和采样是否稳定？
- 中文与英文是否存在系统性质量差距？
- analyst-visible output 是否泄露被禁止的信息？
- 质量提升是否值得增加的调用成本和延迟？

### 1.2 无 ground truth 时，不伪造 ground truth

对真实开放世界数据，通常不存在唯一正确 taxonomy。

此时应报告：

- 多位标注者的一致性；
- 多种可接受划分；
- pairwise / local judgments；
- stability；
- cluster usefulness；
- 失败案例。

不得把某位研究者写出的 taxonomy 当作宇宙真相。

### 1.3 论文数字不是默认验收线

公开论文中的结果依赖特定数据、模型、prompt、sampling、threshold 和 evaluation design。除非这些条件足够一致，否则只能作为背景参考，不能把某个 reported accuracy 直接当作本项目“复现成功”的门槛。

### 1.4 外部 metric 与人类判断并用

- 仅使用 intrinsic metric，可能奖励几何上紧凑但语义上无用的 cluster；
- 仅使用 LLM-as-judge，可能受到同源模型偏好和 prompt 敏感性影响；
- 仅使用人工评审，成本高且难以持续运行。

因此，V1 采用 deterministic metrics、人工标注、independent judge 和 error analysis 的组合。

## 2. Benchmark 分层

### 2.1 Layer A：受控 synthetic benchmark

目的：为 end-to-end accuracy、hierarchy、multilingual parity 和 privacy canary 提供已知 ground truth。

每条 synthetic record 至少具有：

- `leaf_topic_id`；
- `parent_topic_ids`；
- `language`；
- `difficulty`；
- `multi_topic` 标记；
- `style_variant`；
- `contains_canary` 与 canary 类型；
- `actor_id`；
- 生成器版本和 seed。

必须覆盖：

- 中文；
- 英文；
- 中英混合；
- 同义改写；
- 长短文本；
- 相邻但不同的 intent；
- 单条记录含多个主题；
- 极少数主题；
- 无法归类 / outlier；
- 直接标识符、准标识符和稀有事件描述；
- 数据分布不均衡。

Synthetic benchmark 不能替代真实数据，只负责回答“系统是否能恢复我们明确植入的结构”。

### 2.2 Layer B：人工 gold set

目的：在非模板化语言上评估实际语义质量。

建议 V1 建立 `300–500` 条记录的版本化 gold set：

- 来源必须允许研究与再发布，或只发布脱敏后的 annotation；
- 至少 `100` 条中文、`100` 条英文、`100` 条中英混合或多语言样本；
- 每条记录由至少两位标注者独立判断；
- disagreement 由第三位 adjudicator 处理，且保留原始 disagreement；
- 不要求唯一 taxonomy，优先标注 pairwise relation、primary request、acceptable parents 和 label quality。

### 2.3 Layer C：public exploratory corpus

目的：观察开放世界行为、规模、成本和未知 pattern。

例如使用许可合适的公开对话或反馈数据。该层主要报告：

- cluster distribution；
- label/hierarchy 人工抽检；
- stability；
- runtime/cost；
- privacy warnings；
- qualitative error taxonomy。

不得在该层凭空计算“真实准确率”。

### 2.4 Layer D：用户私有数据

目的：验证真实部署价值。

该层结果默认不进入公开 benchmark；必须记录：

- 数据使用授权；
- provider data path；
- retention；
- 访问角色；
- 哪些 artifact 可发布；
- 哪些评估只能在受控环境中运行。

## 3. 阶段级指标

## 3.1 Facet extraction

### 人工 rubric

每个 facet 输出按以下维度评分：

1. **Faithfulness**：是否被原始记录支持；
2. **Coverage**：是否包含完成分析所需的主要信息；
3. **Focus**：是否只回答 facet 问题；
4. **Abstraction**：是否避免无意义细节，同时没有过度抽象；
5. **Privacy compliance**：是否移除 policy 禁止的信息；
6. **Language compliance**：是否符合 representation/display language 策略。

推荐 `0/1/2` 三级评分，并同时报告 pass rate 和平均分。

### 自动指标

- schema validity；
- missing / refusal / parse-error rate；
- 长度分布；
- direct identifier detector hit rate；
- canary retention rate；
- 与 gold primary intent 的 exact / semantic match；
- 重复运行的一致性。

## 3.2 Embedding / neighborhood quality

Embedding 本身不直接给出“聚类准确率”，但可以评估局部邻域：

- `Recall@k`：同一 gold leaf / acceptable pair 是否进入邻居；
- `MRR`；
- pairwise AUC；
- hard-negative retrieval accuracy；
- language-pair retrieval parity；
- translation consistency：同一内容的中英版本是否互为近邻；
- hubness 和 duplicate concentration。

必须至少比较：

- raw text embedding；
- selected facet embedding；
- 一个 multilingual embedding baseline；
- 任何计划作为默认实现的 embedding。

## 3.3 Base clustering

在有 ground truth 的 benchmark 上：

- B-cubed precision / recall / F1；
- Adjusted Rand Index（ARI）；
- Normalized Mutual Information（NMI）；
- pairwise precision / recall / F1；
- coverage；
- noise / unassigned rate；
- cluster size distribution；
- small-cluster recall；
- minority-language recall。

在开放世界数据上：

- 人工 coherence；
- nearest-cluster confusion；
- duplicate-cluster rate；
- outlier appropriateness；
- size imbalance；
- 跨 seed co-assignment stability。

`silhouette`、Davies–Bouldin 等几何指标可以报告，但不得单独决定模型优劣。

## 3.4 Cluster labeling

Label 需要独立于 cluster assignment 评估。

每个 label/description 按以下 rubric：

1. **Faithfulness**：由 cluster 内样本支持；
2. **Coverage**：覆盖主要共同点；
3. **Specificity**：足够具体，不是“其他问题”“技术相关”；
4. **Distinctiveness**：能与最近邻 cluster 区分；
5. **Conciseness**：没有不必要修饰；
6. **No hallucinated causality**：不添加样本不支持的因果或动机；
7. **Privacy compliance**；
8. **Locale quality**：中文表达自然、术语稳定。

额外指标：

- sibling title semantic similarity；
- duplicate / near-duplicate title rate；
- label-to-member retrieval；
- label-to-neighbor margin；
- with-vs-without contrastive examples 的 blind preference。

## 3.5 Hierarchy

### 有已知 tree 时

- hierarchical precision / recall / F1；
- ancestor F1；
- dendrogram purity；
- leaf-to-ancestor path accuracy；
- depth error；
- parent branching factor distribution。

### 开放世界时

每条 parent-child edge 评估：

- child 是否属于 parent；
- parent 是否完整覆盖 child 的核心含义；
- parent 是否过宽或过窄；
- siblings 是否处于相似 abstraction level；
- siblings 是否重复；
- 是否存在 orphan / forced bad assignment。

系统级指标：

- parent-child fit pass rate；
- sibling near-duplicate rate；
- layer compression ratio；
- unassigned rate；
- monotonic node-count violations；
- depth distribution；
- stop reason 分布。

## 3.6 End-to-end reconstruction

在 synthetic benchmark 上，把输出 cluster/hierarchy 映射回已知 taxonomy，报告：

- overall accuracy；
- macro-F1；
- weighted-F1；
- per-leaf precision/recall；
- top-level distribution 的 Jensen–Shannon divergence；
- minority-topic recall；
- concerning / rare / multilingual subsets；
- unmapped / ambiguous rate。

映射过程必须保存，并区分：

- 自动 semantic mapping；
- 人工 adjudication；
- 多对一与一对多映射；
- 无法映射的 cluster。

## 3.7 Stability

每个候选默认配置至少运行三个 seed，并在可承受时跨 model/provider 运行。

报告：

- AMI / ARI across runs；
- pairwise co-assignment agreement；
- cluster matching 后的 label consistency；
- top-level distribution variance；
- hierarchy edge stability；
- result churn：新增少量数据后已有 records 的重分配比例。

“看起来合理但每次完全不同”不是稳定系统。

## 3.8 Multilingual quality

至少分组报告：

- 简体中文；
- 英文；
- 中英混合；
- 其他语言（样本足够时）。

指标包括：

- facet pass rate；
- neighborhood Recall@k；
- B-cubed F1；
- label quality；
- hierarchy fit；
- end-to-end macro-F1；
- privacy error；
- 每条记录成本与 token。

还需运行 translation invariance test：同一语义的中文、英文和平行改写是否获得相近 facet、邻域和 ancestor path。

## 3.9 Privacy evaluation

详细 threat model 见 RFC 0004。质量协议至少包含：

- direct identifier detector recall；
- synthetic canary extraction / retention / release rate；
- rare-topic disclosure test；
- minimum records / unique actors gate coverage；
- privacy auditor recall，特别是高风险输出；
- false positive rate；
- analyst-visible artifacts 的人工抽检；
- provider/log/cache 中敏感内容路径审计。

隐私结果必须按 pipeline stage 报告，不能只审最终页面。

## 3.10 效率与成本

每次 benchmark 记录：

- wall-clock time；
- records / second；
- peak memory；
- LLM calls、tokens 和重试；
- embedding calls / dimensions；
- cache hit rate；
- estimated and actual cost（可得时）；
- failed / partial records；
- resume 后重复工作量。

质量报告应给出 Pareto comparison，而不是把“最贵”自动等同于“最好”。

## 4. V1 provisional gates

下列门槛用于推动可重复工程，不是永久标准，也不是对论文结果的直接复刻。每次调整必须在 RFC / decision log 中记录。

| 维度 | V1 provisional gate |
|---|---:|
| Facet faithfulness pass rate | `>= 90%` |
| Facet schema success | `>= 99%`，失败必须可重试/隔离 |
| Synthetic base clustering B-cubed F1 | `>= 0.80` |
| Cluster label rubric pass rate | `>= 85%` |
| Parent-child fit pass rate | `>= 90%` |
| Sibling near-duplicate rate | `<= 10%` |
| Synthetic top-level reconstruction accuracy | `>= 85%` |
| Synthetic end-to-end macro-F1 | `>= 0.80` |
| 中文与英文 macro-F1 绝对差 | `<= 5` percentage points |
| 三个 seed 的 cluster AMI | `>= 0.70` |
| 高风险 privacy examples 的 auditor recall | `>= 98%` |
| Analyst-visible benchmark 中 direct identifiers | `0` |
| Run provenance fields | `100%` 完整 |

### Gate 解释

- 不通过某个 gate 不一定意味着实现不可合并，但必须标记为 experimental，并附 failure analysis；
- privacy gate 不得以“已知问题”方式绕过 public-release；
- 小样本 confidence interval 必须报告；
- 同一 benchmark 上反复调参必须记录，避免无意中把 test set 当 training set。

## 5. Ablation matrix

V1 至少完成以下 ablation：

| 问题 | A | B |
|---|---|---|
| Representation | raw text | selected facet |
| Label context | in-cluster only | in-cluster + contrastive |
| Base clusterer | paper-like KMeans | alternative clusterer |
| Assignment | forced | allow `unassigned` |
| Hierarchy | geometry-only linkage | semantic propose/assign/relabel |
| Representation language | source language | English pivot / normalized language |
| Privacy prompt | off | on |
| Auditor | off | on |

每个 ablation 至少报告 quality、stability、privacy、latency 和 cost。

## 6. 人工评审协议

### 6.1 Blind review

- 隐藏系统名称、模型名称和配置；
- 随机化候选顺序；
- 对 label/hierarchy 使用 pairwise preference + absolute rubric；
- 允许“二者都可接受”“二者都不可接受”；
- 标注者不得只看 title，必须查看规定数量的成员与邻居。

### 6.2 Inter-annotator agreement

根据任务类型报告：

- Cohen’s kappa / Fleiss’ kappa；
- Krippendorff’s alpha；
- pairwise agreement；
- disagreement distribution。

若一致性低，应优先检查 rubric 是否含糊，而不是把分歧全部归咎于标注者。

### 6.3 Error taxonomy

每次 release 至少抽样记录：

- facet omission；
- facet hallucination；
- semantic neighbor failure；
- over-splitting；
- under-splitting；
- forced bad assignment；
- vague label；
- duplicate siblings；
- wrong abstraction level；
- multilingual drift；
- privacy leakage；
- evaluator disagreement。

## 7. LLM-as-judge 约束

允许使用 LLM judge，但必须：

- 与被评系统尽量使用不同 model family；
- 固定并版本化 rubric/prompt；
- 在人工 gold subset 上校准；
- 报告 judge 的 confusion matrix；
- 对高风险 privacy examples 优先优化 recall；
- 不把 judge 分数包装成客观真理；
- 对模型升级执行 regression suite。

## 8. 结果发布格式

每份 evaluation report 至少包含：

1. 数据集版本与样本构成；
2. 完整 pipeline config；
3. model/provider 与 prompt/schema versions；
4. primary metrics 与 confidence interval；
5. 分语言、分难度、分主题结果；
6. stability；
7. privacy；
8. latency/cost；
9. ablation；
10. 失败案例；
11. 已知 limitation；
12. 与上一个版本的 regression / improvement。

## 9. 不接受的成功证明

以下证据单独出现时不足以宣称“准确”：

- 一张漂亮的 UMAP；
- 几个 cherry-picked cluster；
- 一个高 silhouette score；
- LLM 对自身输出的笼统好评；
- cluster title 看起来像 taxonomy；
- 在英文小样本上运行成功；
- 没发现 PII 就宣称 privacy-preserving；
- 与论文某个百分比数字相近但评估协议不同。

## 10. 待讨论事项

1. V1 gold set 的公开数据来源与许可；
2. 是否将 provisional gates 设为 CI warning 还是 release blocker；
3. 中文标注 rubric 是否需要语言学/领域双角色评审；
4. stability 的主指标使用 AMI 还是 co-assignment agreement；
5. 对 multi-topic records，主任务是 single-primary assignment 还是 multi-label evaluation；
6. privacy auditor 的独立 gold set 如何构造；
7. 哪些 benchmark artifacts 可以公开，哪些只能公开聚合指标。