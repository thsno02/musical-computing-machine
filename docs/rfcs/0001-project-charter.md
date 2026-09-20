# RFC 0001：项目章程——我们究竟要复现什么

- **状态**：Draft
- **日期**：2026-09-20
- **讨论入口**：待建立 GitHub Discussion
- **相关 RFC**：
  - [RFC 0002：评估协议](0002-evaluation-protocol.md)
  - [RFC 0003：架构与 artifact contract](0003-architecture-contracts.md)
  - [RFC 0004：隐私 threat model](0004-privacy-threat-model.md)

## 摘要

本项目不是复刻 Anthropic 的某个网页，也不是把 `embedding + KMeans + UMAP` 拼起来。

本项目要实现并验证的是一种 **Clio-style、bottom-up、facet-conditioned 的层次语义聚类系统**：先把每条原始记录转换为面向分析问题的语义表示，再形成可解释的 base clusters；利用 cluster 内样本和邻近 cluster 的 contrastive examples 生成有区分度的标签；随后把 cluster 当作新的语义单元，逐层生成、去重、分配和重命名 parent clusters；最后只发布通过质量与隐私 gate 的聚合结果。

面向中文读者，项目文档与默认报告使用中文；实现、公共 API 与必要术语使用英文。

## 1. 一句话目标

> 给定一批非结构化记录，产出一棵可解释、可下钻、可复现、可评估，并具有明确隐私发布边界的层次化语义聚类树。

V1 优先处理 AI 对话，但组件接口不得与“聊天消息”强耦合，后续可以通过 adapter 支持工单、用户反馈、访谈、文档片段等记录。

## 2. “复现”的含义

### 2.1 我们复现什么

我们复现公开方法中可观察、可验证的系统行为：

1. **Facet extraction**：从原始记录抽取面向分析目标的简洁语义 facet；
2. **Semantic representation**：对选定 facet 生成 embedding，而不是默认直接 embed 全量原文；
3. **Base clustering**：发现底层、相对具体的语义群组；
4. **Contrastive labeling**：标签生成必须同时看到 cluster 内代表样本和邻近 cluster 的反例；
5. **Bottom-up hierarchy**：以已命名 cluster 为语义单元，逐层构建 parent clusters；
6. **Privacy gating**：分析者可见结果与原始记录之间存在明确隔离和发布 gate；
7. **Evaluation**：每个阶段都有独立指标，端到端结果也有可复现实验；
8. **Exploration artifacts**：输出稳定的 tree/map/report 数据 contract，UI 只是其消费者。

### 2.2 我们不复现什么

公开资料没有给出的 Anthropic 内部实现、数据和运营流程不在复现承诺中，包括但不限于：

- 内部数据集、标注数据与真实用户流量；
- 未公开的精确 `k`、内部阈值、prompt 版本、模型路由与人工审核流程；
- 内部权限系统、合规流程、安全调查工具和账号级联动；
- Anthropic Insights / Clio 产品界面的视觉与交互细节；
- 论文中某个数字结果的机械复刻。

因此，本项目应使用“**method reproduction / behavior reproduction**”，而不是“exact reproduction”。任何与论文结果的比较都必须标明数据、模型、prompt、阈值和评估口径是否一致。

## 3. 核心不变量

下面六条是项目的 identity。删除其中任意一条，都应明确说明这是简化版 baseline，而不是完整的 Clio-style pipeline。

### 3.1 Facet 决定语义空间

“相似”不是数据的固有属性，而是分析问题的函数。

同一批对话可以按 `request`、`task`、`language`、`failure_mode` 或其他 facet 得到完全不同的结构。系统必须显式记录：

- facet 的问题定义；
- extraction prompt / schema / model；
- representation text；
- 缺失值与不确定性；
- display language 与 representation language。

默认不得把“直接 embed 全量原始文本”当作唯一或隐含方案。它只能作为对照实验。

### 3.2 Base cluster 是可解释的语义单元

Base clustering 不是只追求低维图上“看起来分开”，而是要形成足够具体、内部一致、外部可区分的语义单元。

系统必须允许：

- outlier / noise；
- cluster 大小不均衡；
-无法稳定归类的记录进入 `unassigned`；
- 在 paper-baseline 模式下启用强制分配，以便做 apples-to-apples ablation。

### 3.3 标签必须有 contrastive context

只向 LLM 提供 cluster 内样本，容易产生宽泛、重复的标签。

标签生成器至少要接收：

- cluster 内代表样本；
- 与该 cluster 最近但不属于它的 contrastive samples；
- cluster 大小与可选的统计摘要；
- 标签语言和格式约束。

标签质量必须单独评估，不能用“LLM 生成得很流畅”代替验证。

### 3.4 “层层递增”是语义层级，不是多档 `k`

本项目对 hierarchy 的定义是：

1. 以 base clusters 的名称、描述和统计作为当前层节点；
2. 在局部 neighborhood 中提出候选 parent concepts；
3. 对候选 parent 去重；
4. 将每个 child 分配给最合适的 parent，或在允许时标为 `unassigned`；
5. 根据最终 children 重新命名和描述 parent；
6. 重复以上过程，直到满足停止条件。

每一层节点数原则上单调减少；V1 中每个 child 最多有一个 parent，形成 tree，而不是 DAG。

以下做法不等同于上述 hierarchy：

- 对全部 raw points 分别运行 `k=1000/100/10`；
- 把 UMAP 的 zoom level 当作语义层级；
- 仅根据 centroid 做 agglomerative linkage，且不重新生成语义标签；
- 先写一个固定 ontology，再把样本塞进去。

这些方法可以作为 baseline，但必须显式命名。

### 3.5 隐私是发布流程，不是 embedding 的副作用

Embedding、摘要、聚类本身都不等于匿名化。

系统必须区分：

- **raw data plane**：原始记录和直接标识符；
- **derived analysis plane**：facet、embedding、assignment、候选标签；
- **analyst-visible / release plane**：通过质量和隐私 gate 的聚合结果。

“privacy-preserving”不是默认事实。除非部署场景、访问控制、阈值、审计和评估均有证据支持，项目应优先使用“privacy-oriented”或“具有隐私防护层”的表述。

### 3.6 每次运行必须可追溯

每个 run 至少记录：

- 输入数据 fingerprint；
- config、随机种子与代码版本；
- 每个 model/provider 的明确标识；
- prompt/schema 版本；
- embedding 与 clustering 参数；
- 阶段耗时、调用量、token 与可得的成本；
- 质量、稳定性与隐私评估；
- warning、失败和 stop reason。

没有这些元数据的结果只能视为 exploratory demo，不能作为可比较实验。

## 4. V1 用户与用例

### 4.1 主要用户

- 想理解大量 AI 对话/反馈中“用户在做什么”的产品与研究团队；
- 想研究 semantic clustering、taxonomy induction 和 hierarchical labeling 的工程/ML 读者；
- 需要中文、英文和中英混合数据 benchmark 的开发者。

### 4.2 主要用例

- 发现未知的 request / task 模式；
- 从具体 pattern 向上浏览较粗的主题；
- 比较时间段、语言、产品版本或其他 facet distribution；
- 评估 embedding、clusterer、labeler、hierarchizer 的替代实现；
- 生成可供人工研究的聚合报告。

### 4.3 非主要用户

V1 不是为下列需求设计：

- 逐条检索、阅读或标注原始对话；
- 实时客服路由；
- 在线推荐；
- 账号级风险处置；
- 法律意义上的匿名化认证；
- 通用企业 BI 平台。

## 5. V1 范围

### 5.1 In scope

- Batch pipeline；
- AI conversation JSONL 的 canonical schema 与 adapter；
- 中文、英文、中英混合数据；
- 可插拔 LLM、embedding backend 和 clusterer；
- 一个 paper-like baseline；
- 至少一个非 paper alternative baseline；
- base clustering、contrastive labeling、bottom-up hierarchy；
- tree/report artifacts；
- 可选的轻量 explorer；
- 阶段级与端到端 evaluation；
- privacy threat model、release modes 与 audit artifacts；
- cache、checkpoint、resume 和 run manifest。

### 5.2 Out of scope

下列内容不应阻塞 V1：

- Anthropic UI 的 pixel-perfect clone；
- 实时或 sub-minute streaming；
- 生产级 multi-tenancy、billing、RBAC 与 SaaS 运维；
- 多模态输入；
- 训练 foundation model 或 embedding model；
- 通用 RAG、knowledge graph 或 agent memory；
- 通用搜索引擎；
- 自动执法、自动封禁、自动绩效判断等 high-impact action；
- 形式化 differential privacy、k-anonymity 或其他未经实现与证明的保证；
- 为所有行业建立通用 ontology；
- 用单一“准确率”掩盖多个阶段的误差。

### 5.3 数据规模承诺

V1 的工程目标是：

- 本地 smoke test：`10²` 级记录；
- 标准 benchmark：`10³–10⁴` 级记录；
- 扩展 benchmark：至少一次 `10⁵` 级记录运行。

在完成内存、成本、失败恢复和质量测试前，不承诺“可扩展到 millions”。

## 6. 输入与输出 contract 概览

### 6.1 Canonical input

最小记录：

```json
{
  "record_id": "stable-id",
  "actor_id": "optional-stable-pseudonym",
  "timestamp": "optional-ISO-8601",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ],
  "metadata": {}
}
```

- `record_id` 必须稳定且唯一；
- `actor_id` 可选，但没有它就不能声称执行了 unique-user / unique-actor aggregation；
- adapter 可以接收其他 schema，但必须转换到 canonical record；
- 原始内容不得默认进入可发布 artifact。

### 6.2 主要输出

- run manifest；
- facet records；
- embedding metadata；
- base assignments；
- cluster labels 与代表性统计；
- hierarchy tree；
- privacy report；
- evaluation report；
- human-readable report；
- 可选 map coordinates。

精确文件 contract 见 RFC 0003。

## 7. 中文优先的含义

“中文优先”不是把 README 翻译成中文，而是：

1. 文档、RFC、CLI 帮助和默认报告以简体中文为主；
2. 专业术语保留更准确的英文写法；
3. code identifier 和公共 API 使用英文；
4. 测试集必须覆盖中文、英文和中英混合；
5. 报告必须展示分语言质量，而不是只给总体平均值；
6. representation language 与 display language 解耦；
7. 中文 cluster label 必须评估信息密度、歧义和同义词重复问题。

## 8. 成功标准

V1 只有同时满足以下条件才算完成：

- end-to-end run 可从 canonical JSONL 生成 hierarchy 和 report；
- 所有核心阶段具有可替换 interface；
- paper-like baseline 可以复现方法结构；
- 至少一个 alternative baseline 可比较；
- RFC 0002 中的 provisional quality gates 有完整报告；
- 中文、英文和中英混合数据分别报告结果；
- privacy release mode 不输出被禁止的 artifacts；
- run manifest 足以复现实验；
- failure、outlier、tiny dataset 和 provider error 有明确行为；
- 文档不宣称未被实验支持的隐私或准确性结论。

## 9. 里程碑

### M0：Specification

- 接受本 RFC；
- 接受 evaluation、architecture、privacy RFC；
- 建立 synthetic benchmark schema；
- 固化 terminology。

### M1：Vertical slice

- `100–1,000` 条 synthetic records；
- 一种 facet；
- 一种 embedding；
- paper-like KMeans baseline；
- base labels；
- 两层 hierarchy；
- 完整 run artifacts。

### M2：Evaluation-first

- gold set 与标注 rubric；
- 阶段级 metrics；
- 中文/英文/混合报告；
- seeds/models 的 stability run；
- privacy canary benchmark。

### M3：Alternative implementations

- density-based 或 balanced clustering baseline；
- alternative hierarchy；
- ablation：raw text vs facet、with/without contrastive examples、forced vs unassigned；
- 成本与质量 trade-off 报告。

### M4：Explorer and release

- tree/map explorer；
- public-release artifact profile；
- `10⁵` 级扩展运行；
- 可复现 demo 与版本化报告。

## 10. 明确的开放问题

在进入 M1 前，需要在 Discussion 中形成记录：

1. V1 是否只接受 conversation input，还是同时提供 generic text adapter？
2. 默认 representation language 是原语言、英文 pivot，还是 multilingual embedding + 原语言摘要？
3. 默认是否允许 `unassigned`，并把 forced assignment 仅留给 paper baseline？
4. hierarchy 默认深度由目标节点数、最小压缩比、语义质量还是组合 stop policy 决定？
5. V1 是否必须包含 `public-release` mode，还是先完成 `private-analysis`？
6. 哪些 provisional metrics 可以作为 merge/release gate？
7. explorer 是否属于 V1 必交付，还是 artifact contract 稳定后再做？

## 11. 决策原则

遇到实现选择时，按以下顺序取舍：

1. 是否保持 core invariants；
2. 是否可评估；
3. 是否可复现；
4. 是否有清楚的隐私边界；
5. 是否对中文数据成立；
6. 是否可替换而不污染其他阶段；
7. 最后才是 UI 完整度和算法新颖性。

本项目可以使用两年前没有的模型、embedding、clusterer 和工程框架，但不能因为组件更新而丢失上述系统行为。