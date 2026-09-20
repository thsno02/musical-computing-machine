# RFC 0003：架构与 Artifact Contract

- **状态**：Draft
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)、[RFC 0002](0002-evaluation-protocol.md)

## 摘要

本 RFC 定义的是系统边界和组件 contract，而不是绑定某个框架。

核心目标是让 LLM、embedding、clusterer、hierarchizer、privacy gate 和 UI 可以独立替换，同时保证同一个 run 可恢复、可审计、可比较。任何实现都必须通过版本化 artifact 交换数据，避免把整个 pipeline 写成只能按一种模型、一个 notebook 或一个 UI 运行的脚本。

## 1. Trust zones

系统至少划分三个 zone。

### Zone A：Raw data plane

包含：

- 原始 records / conversations；
- 直接标识符；
- actor mapping；
- 原始 metadata；
- 可能包含敏感内容的 provider request/response；
- 可逆映射与调试日志。

默认权限：最严格。不得被 explorer 或 public report 直接读取。

### Zone B：Derived analysis plane

包含：

- normalized records；
- facet values；
- embeddings；
- cluster assignments；
- representative sample IDs；
- candidate labels；
- hierarchy candidates；
- stage-level evaluation。

这些内容仍可能泄露原始信息，因此不是天然“安全区”。

### Zone C：Analyst-visible / release plane

包含通过 policy gate 的：

- cluster title / description；
- aggregate counts 和比例；
- hierarchy；
- 经过批准的时间、语言或 metadata aggregation；
- quality/privacy warnings；
- 可发布 report 和 explorer data。

从 Zone A/B 到 Zone C 必须经过明确的 `PrivacyGate`，不能靠 UI “不显示”代替数据隔离。

## 2. 逻辑 pipeline

```text
SourceAdapter
  -> Normalizer
  -> FacetExtractor
  -> RepresentationBuilder
  -> EmbeddingBackend
  -> BaseClusterer
  -> ClusterLabeler
  -> Hierarchizer
  -> PrivacyGate
  -> Evaluator
  -> Reporter / Explorer
```

实际执行允许并行、缓存和重试，但 lineage 必须保持可追踪。

## 3. 核心数据类型

以下为概念 schema；具体实现可以使用 Pydantic、dataclass、Arrow 或其他工具。

### 3.1 CanonicalRecord

```python
class CanonicalRecord:
    record_id: str
    actor_id: str | None
    timestamp: datetime | None
    content: RecordContent
    metadata: dict[str, JsonValue]
    source: SourceRef
```

约束：

- `record_id` 在 dataset version 内唯一且稳定；
- `actor_id` 必须是稳定 pseudonym，不应直接使用 email / phone / account name；
- `content` 可以是 messages 或 generic text blocks；
- metadata 字段必须在 config 中 allowlist；
- raw content 不得写入 release artifacts。

### 3.2 FacetDefinition

```python
class FacetDefinition:
    name: str
    question: str
    output_schema: JsonSchema
    representation_template: str
    extraction_policy: str
    representation_language: str
    display_language: str
    version: str
```

Facet 是一等公民，不能只是 prompt 文件中的隐含字符串。

### 3.3 FacetRecord

```python
class FacetRecord:
    record_id: str
    facet_name: str
    value: JsonValue | None
    representation_text: str | None
    confidence: float | None
    status: Literal["ok", "missing", "refused", "error", "filtered"]
    provenance: ModelCallRef
```

### 3.4 ClusterNode

```python
class ClusterNode:
    cluster_id: str
    level: int
    title: str
    description: str
    child_ids: list[str]
    member_count: int
    unique_actor_count: int | None
    representative_ids: list[str]       # Zone B only
    contrastive_ids: list[str]           # Zone B only
    centroid_ref: ArtifactRef | None
    label_provenance: ModelCallRef
    quality: dict[str, float]
    privacy_status: str
```

Release serialization 必须删除所有被 policy 禁止的字段。

### 3.5 Hierarchy

```python
class Hierarchy:
    root_ids: list[str]
    nodes: dict[str, ClusterNode]
    depth: int
    stop_reason: str
    assignment_policy: str
    version: str
```

V1 要求：

- 每个 non-root child 最多一个 parent；
- 不允许 cycle；
- 每一层节点数原则上减少；
- 所有叶节点可追溯到 base cluster；
- `unassigned` 必须显式表示，不得静默丢失。

## 4. 组件接口

## 4.1 SourceAdapter

职责：读取外部数据并生成 canonical records。

```python
class SourceAdapter(Protocol):
    def scan(self, source: SourceConfig) -> Iterator[CanonicalRecord]: ...
    def fingerprint(self, source: SourceConfig) -> DatasetFingerprint: ...
```

不负责：facet extraction、脱敏承诺、clustering。

V1 必须提供：

- canonical JSONL adapter；
- conversation JSONL example adapter；
- generic text adapter 可以是 experimental。

## 4.2 Normalizer

职责：

- schema validation；
- Unicode / whitespace normalization；
- deterministic filtering；
- metadata allowlist；
- actor pseudonymization hook；
- size/token limits；
- duplicate detection。

所有删除、截断和过滤都必须写入 reason code。

## 4.3 FacetExtractor

```python
class FacetExtractor(Protocol):
    async def extract(
        self,
        records: Sequence[CanonicalRecord],
        facet: FacetDefinition,
        context: RunContext,
    ) -> Sequence[FacetRecord]: ...
```

要求：

- structured output；
- async batching；
- per-record retry；
- idempotent cache key；
- prompt/schema/model provenance；
- 失败隔离，不能因单条 bad record 中止整个 run。

## 4.4 RepresentationBuilder

职责：从 facet value 构建 embedding 输入。

它必须显式区分：

- source-language representation；
- normalized display text；
- optional English pivot；
- concatenated multi-facet representation；
- missing-value behavior。

## 4.5 EmbeddingBackend

```python
class EmbeddingBackend(Protocol):
    async def embed(
        self,
        texts: Sequence[str],
        context: RunContext,
    ) -> EmbeddingBatch: ...
```

manifest 必须记录：

- provider / model / revision；
- dimensions；
- normalization；
- distance metric；
- batching；
- truncation；
- cache key；
- provider retention policy 的用户配置说明。

## 4.6 BaseClusterer

```python
class BaseClusterer(Protocol):
    def fit_predict(
        self,
        embeddings: EmbeddingMatrix,
        records: Sequence[FacetRecord],
        config: ClusterConfig,
    ) -> BaseClusteringResult: ...
```

输出必须包含：

- assignment；
- `unassigned` / noise；
- centroid 或 medoid；
- cluster size；
- confidence / distance（算法允许时）；
- seed；
- stop/warning；
- algorithm-specific diagnostics。

### Paper-like baseline

V1 的 paper-like baseline 应保持方法结构，而不是声称精确复制未知参数：

- facet representation；
- sentence embedding；
- large-`k` KMeans；
- minimum aggregation thresholds；
- cluster-level contrastive labeling。

精确 embedding model、`k` 和 thresholds 由 benchmark config 决定并完整记录。

### Alternative baseline

至少实现或集成一种：

- HDBSCAN / density-based；
- balanced hierarchical clustering；
- agglomerative clustering；
- topic-model baseline。

Alternative 必须使用同一 input/output contract，才能公平比较。

## 4.7 RepresentativeSampler

职责：为 labeler 选择 cluster 内代表样本和邻近反例。

策略必须版本化，例如：

- nearest-to-centroid；
- diversity-aware sampling；
- stratified by language/time；
- nearest-outside-cluster hard negatives；
- random sample with fixed seed。

不能默认把所有成员发送给 LLM。

## 4.8 ClusterLabeler

```python
class ClusterLabeler(Protocol):
    async def label(
        self,
        members: Sequence[LabelExample],
        contrastive: Sequence[LabelExample],
        policy: LabelPolicy,
        context: RunContext,
    ) -> ClusterLabel: ...
```

输出：

- concise title；
- description；
- optional distinguishing traits；
- uncertainty / warnings；
- provenance。

禁止：

- 生成成员不支持的因果解释；
- 复制 direct identifiers；
- 根据单个罕见样本命名整个 cluster；
- 把安全、意图或用户属性判断包装成事实。

## 4.9 Hierarchizer

```python
class Hierarchizer(Protocol):
    async def build(
        self,
        base_clusters: Sequence[ClusterNode],
        config: HierarchyConfig,
        context: RunContext,
    ) -> Hierarchy: ...
```

### Clio-style semantic hierarchy 参考流程

```text
current = base_clusters
level = 0

while not stop(current, level, config):
    node_embeddings = embed(title + description for node in current)
    neighborhoods = partition_for_context_window(node_embeddings)

    candidate_parents = []
    for neighborhood in neighborhoods:
        nearby_outside = nearest_nodes_outside(neighborhood)
        candidate_parents += propose_parents(
            children=neighborhood,
            contrastive=nearby_outside,
        )

    parents = deduplicate(candidate_parents)
    assignments = assign_each_child(
        children=current,
        candidate_parents=parents,
        allow_unassigned=config.allow_unassigned,
    )

    parents = relabel_from_final_children(parents, assignments)
    validate_tree(parents, assignments)
    current = parents
    level += 1
```

### Stop policy

Stop condition 必须可解释，可由以下信号组合：

- target root count；
- maximum depth；
- minimum compression ratio；
- minimum members per parent；
- parent-child quality estimate；
- duplicate rate；
- no-valid-merge；
- budget limit。

`stop_reason` 是必填 artifact。

### Assignment modes

- `paper_baseline_forced`：每个 child 必须进入某个 parent；
- `quality_first`：低置信 child 可进入 `unassigned`；
- `strict_tree`：最终所有 base clusters 必须到达 root，可在独立 repair stage 处理。

默认建议 `quality_first`；用于 paper-like ablation 时选择 forced mode。

## 4.10 PrivacyGate

```python
class PrivacyGate(Protocol):
    async def evaluate(
        self,
        candidate: ReleaseCandidate,
        policy: PrivacyPolicy,
        context: RunContext,
    ) -> PrivacyDecision: ...
```

返回：

- `allow` / `redact` / `suppress` / `manual_review`；
- reason codes；
- triggered rules；
- model/detector provenance；
- audit score；
- redacted release payload。

PrivacyGate 必须作用于实际 release serialization，而不是只生成一份旁路报告。

## 4.11 Evaluator

Evaluator 读取阶段 artifacts，而不是重新运行隐含 pipeline。

```python
class Evaluator(Protocol):
    def evaluate(self, run: RunArtifacts, benchmark: BenchmarkSpec) -> EvaluationReport: ...
```

所有 metric 都应携带：

- metric name/version；
- population/subset；
- sample count；
- confidence interval（适用时）；
- missing/excluded records；
- evaluator provenance。

## 4.12 Reporter / Explorer

Reporter 和 Explorer 只能读取 release profile 允许的 artifacts。

UI 不得：

- 从浏览器端隐藏但仍下载 raw content；
- 假定 cluster ID 永远稳定；
- 把 UMAP 坐标当成 ground-truth geometry；
- 在没有 warning 的情况下展示被 suppression 的 cluster；
- 绕过 privacy gate 加载代表样本。

## 5. Run artifact contract

建议目录：

```text
runs/<run_id>/
  run_manifest.json
  run_events.jsonl
  run_warnings.jsonl
  config.resolved.yaml
  provenance/
    code.json
    models.json
    prompts.json
    dataset.json
  private/
    records.parquet
    actor_map.enc              # optional, external key management
  derived/
    facets/<facet>.parquet
    embeddings/<facet>.f16.npy
    embeddings/<facet>.index.json
    base_assignments.parquet
    representatives.jsonl
    clusters.base.jsonl
    clusters.level-01.jsonl
    clusters.level-02.jsonl
    hierarchy.internal.json
  privacy/
    stage_audit.jsonl
    release_decisions.jsonl
    privacy_report.json
  evaluation/
    metrics.json
    annotations.jsonl          # if permitted
    evaluation_report.md
  release/
    clusters.jsonl
    hierarchy.json
    aggregates.parquet
    explorer.json
    report.md
```

实现可以调整格式，但语义 contract 必须版本化。

## 6. Run manifest 最小字段

```json
{
  "schema_version": "1.0",
  "run_id": "...",
  "created_at": "...",
  "status": "completed|partial|failed",
  "code_revision": "git-sha",
  "dataset_fingerprint": "...",
  "config_fingerprint": "...",
  "random_seeds": {},
  "models": [],
  "prompts": [],
  "stages": [],
  "artifacts": [],
  "cost": {},
  "warnings": [],
  "release_profile": "private-analysis|public-release|research-public-data"
}
```

要求：

- manifest 在 run 开始时创建、执行中增量更新；
- partial run 仍保留完整状态；
- resume 前验证 dataset/config fingerprint；
- 不允许在不留记录的情况下复用不同模型产生的 cache。

## 7. Cache 与 checkpoint

Cache key 至少包含：

- canonical input hash；
- component name/version；
- model/provider/revision；
- prompt/schema version；
- relevant config；
- representation language；
- privacy mode。

Checkpoint 必须是 stage-level 和 batch-level 可恢复的。

以下变化默认使 cache 失效：

- prompt 或 output schema 变化；
- model revision 变化；
- normalization policy 变化；
- privacy policy 变化；
- input content 变化；
- embedding dimensions/normalization 变化。

## 8. ID 与版本策略

- `record_id` 来自 source 或 deterministic mapping；
- `cluster_id` 是 run-scoped，不承诺跨 run 稳定；
- 跨 run 对比通过 cluster matching artifact 完成；
- node title 不是 ID；
- schema、component、prompt、policy 必须独立版本化；
- release artifact 应携带 `run_id` 和 `schema_version`。

## 9. 多语言策略

必须把以下概念解耦：

- **source language**：原始记录语言；
- **representation language**：用于 embedding / clustering 的文本语言；
- **display language**：cluster labels 和报告语言。

推荐初始实验矩阵：

1. multilingual embedding + source-language facet；
2. multilingual embedding + normalized Chinese/English label；
3. English pivot facet + multilingual/source display；
4. separate-by-language clustering + cross-language parent merge。

在 benchmark 结束前不把其中任意一种写死为“正确方案”。默认 display language 可设为 `zh-CN`，但必须保留原语言证据和 provenance。

## 10. Edge cases

### Tiny datasets

- 少于配置的 minimum records 时拒绝构建 hierarchy；
- 返回 reasoned report，而不是生成看似完整的三层树；
- 允许只做 facet / neighborhood exploration。

### Multi-topic records

V1 主 contract 为 single primary assignment，但必须标记 `multi_topic_suspected`。Multi-label clustering 作为后续 RFC，不在 V1 核心路径内。

### Outliers

- 不强制把所有点放入正常 cluster；
- outlier 不应直接以原始内容出现在 release report；
- 大量 outliers 是诊断信号，不应静默丢弃。

### Failed model calls

- per-record/batch error isolation；
- bounded retries；
- failure reason；
- partial run；
- downstream stage 明确处理 missing facets。

### Duplicate records

- exact duplicate 和 near-duplicate 分开报告；
- 默认不让重复内容人为放大 cluster prevalence；
- 是否 deduplicate 必须写入 config 和 report。

### Temporal drift

V1 不做实时 incremental clustering，但 artifacts 必须允许按时间切片比较。跨时间的 cluster matching 是 evaluation/reporting 功能，不保证 ID 稳定。

## 11. 实现边界

### Core library 应负责

- schemas/contracts；
- pipeline orchestration；
- adapters/interfaces；
- artifacts/provenance；
- baseline components；
- evaluation hooks；
- privacy gate enforcement。

### UI 应负责

- tree/map rendering；
- filters；
- aggregate comparison；
- warning/display states；
- shareable view state（不得包含敏感数据）。

### UI 不应负责

- 真正的 privacy enforcement；
- cluster computation；
- artifact repair；
- model/provider secrets；
- raw/derived data access policy。

## 12. 待讨论事项

1. Artifact 首选 Arrow/Parquet 还是 JSONL-first；
2. V1 是否需要统一 async runtime；
3. paper-like baseline 的 default embedding 由复现优先还是中文质量优先决定；
4. default assignment mode 是否采用 `quality_first`；
5. hierarchy candidate proposal 是否需要 deterministic non-LLM baseline；
6. release artifacts 是否允许展示经过批准的 synthetic representative examples；
7. cluster matching 是否进入 V1 contract；
8. explorer 是独立 package 还是同一 monorepo workspace。