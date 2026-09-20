# RFC 0003：V1 单一路径与最小 Artifact Contract

- **状态**：Accepted for V1
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)、[RFC 0002](0002-evaluation-protocol.md)

## 0. 决定

V1 不建设通用 pipeline framework。

实现先采用一条顺序执行、容易阅读和调试的 Python 路径：

```text
load
  -> extract_request
  -> embed_requests
  -> cluster_leaves
  -> label_leaves
  -> build_parents
  -> render_artifacts
```

只有当第二种 backend、第二种 schema 或第二种算法真实出现时，才提取 interface。

## 1. 一条命令

目标调用形态：

```bash
uv run mcm run \
  --input data/demo.zh.jsonl \
  --config configs/tracer.yaml \
  --output runs/demo
```

具体 CLI 名称可以调整，但必须保持：

- 输入、配置和输出位置显式；
- 一条命令跑完整链路；
- 失败返回非零 exit code；
- 中间 artifacts 保留下来，方便定位最早失败阶段。

## 2. 最小输入 contract

V1 只接受 conversation JSONL：

```json
{
  "id": "conv-001",
  "messages": [
    {"role": "user", "content": "帮我写一个 Python 脚本整理 CSV"},
    {"role": "assistant", "content": "..."}
  ],
  "gold_leaf": "csv-data-processing",
  "gold_parent": "programming"
}
```

规则：

- `id` 必填、稳定、唯一；
- `messages` 必填；
- `gold_leaf` / `gold_parent` 仅 demo benchmark 使用，真实输入可以省略；
- V1 不支持 generic text、工单、数据库 adapter 或任意 metadata schema；
- 输入 validation 只检查当前 pipeline 真正需要的字段。

## 3. 最小配置

示例：

```yaml
seed: 42

facet:
  name: request
  language: zh-CN

models:
  llm: one-working-model
  embedding: one-working-embedding-model

clustering:
  leaf_k: 8
  hierarchy_k: [3, 1]

sampling:
  representatives: 5
  contrastive: 3

output:
  include_raw_text_in_report: false
```

V1 不实现 config inheritance、provider registry、dynamic model routing 或自动 tuning。

## 4. 顺序 pipeline

### 4.1 Load

读取 JSONL，检查：

- JSON 可解析；
- `id` 唯一；
- `messages` 非空；
- 至少存在一条 user message。

输出内存中的 records 和 input fingerprint。

### 4.2 Extract request facet

固定问题：

> 用户希望助手完成什么？请用一句简洁中文概括，不要复述无关细节。

每条记录输出：

```json
{
  "id": "conv-001",
  "request": "编写脚本整理 CSV 数据",
  "status": "ok"
}
```

V1 可以顺序或简单 batch 调用。单条失败可以重试一次；仍失败则整个 demo run 标记失败，不实现复杂隔离队列。

### 4.3 Embed requests

只对 `request` 文本生成 embedding。

记录：

- embedding model；
- dimensions；
- normalization（如有）；
- distance metric；
- 输入顺序和 record IDs。

V1 不建立 `EmbeddingBackend` protocol。

### 4.4 Cluster leaves

使用固定 seed 的 KMeans：

```python
assignments = KMeans(
    n_clusters=config.leaf_k,
    random_state=config.seed,
).fit_predict(embeddings)
```

V1 不自动选择 `k`，也不处理 noise / `unassigned`。显式 `leaf_k` 是 tracer bullet 的有意简化。

### 4.5 Select examples

对每个 leaf cluster：

- representatives：选择离 centroid 最近的 `N` 个 request facets；
- contrastive examples：选择离该 centroid 最近、但属于其他 cluster 的 `M` 个 request facets。

只把 facet 文本发送给 labeler，不默认发送完整对话。

### 4.6 Label leaves

固定 prompt 输入：

- cluster 内 representative facets；
- cluster 外 contrastive facets；
- 输出语言和格式要求。

输出：

```json
{
  "node_id": "leaf-03",
  "level": 0,
  "title": "处理表格与 CSV 数据",
  "description": "用户要求编写或修改程序来读取、清洗、转换和导出表格数据。",
  "member_count": 17
}
```

V1 不实现多模型 voting、label repair 或独立 judge。

### 4.7 Build hierarchy

对当前层每个 node 构造：

```text
node_text = title + "。" + description
```

然后对 node texts embedding，并按 `hierarchy_k` 逐层运行 KMeans。

对于 `leaf_k: 8`、`hierarchy_k: [3, 1]`：

```text
8 leaf nodes
  -> 3 parent nodes
  -> 1 root node
```

每个 parent 使用最终 children 的标题、描述和数量重新生成 `title + description`。

V1 hierarchy 的必要性质：

- 每个 child 恰好一个 parent；
- node count 逐层减少；
- parent 从最终 children relabel；
- 所有 leaf 都能到达 root；
- 没有 cycle。

V1 不实现动态 parent proposal、candidate dedup、`unassigned`、repair、DAG 或自动 stop policy。它验证的是“把 cluster node 当作新语义单元继续上卷”这一条主链路。

### 4.8 Render

生成 machine-readable artifacts 和中文 report。

Report 至少包含：

- 数据集与配置摘要；
- base clustering metrics；
- hierarchy tree；
- 每个 node 的 title、description 和 count；
- 轻量人工 review 表；
- runtime / token / cost；
- 已知限制。

## 5. 最小 artifact contract

```text
runs/<run_id>/
  run.json
  facets.jsonl
  embeddings.npy
  embedding_index.json
  leaf_assignments.jsonl
  nodes.jsonl
  hierarchy.json
  metrics.json
  report.md
```

### 5.1 `run.json`

```json
{
  "run_id": "demo",
  "status": "completed|failed|failed_sanity",
  "code_revision": "git-sha",
  "input_fingerprint": "sha256:...",
  "config": {},
  "models": {},
  "seed": 42,
  "stages": [],
  "runtime_seconds": 0,
  "usage": {},
  "warnings": []
}
```

### 5.2 `facets.jsonl`

每条记录只包含：

- `id`；
- `request`；
- `status`；
- 可选 usage/error metadata。

### 5.3 `leaf_assignments.jsonl`

```json
{"id":"conv-001","leaf_id":"leaf-03","distance":0.42}
```

### 5.4 `nodes.jsonl`

每行一个 leaf / parent / root node：

```json
{
  "node_id": "parent-01",
  "level": 1,
  "title": "编程与数据处理",
  "description": "...",
  "child_ids": ["leaf-01", "leaf-03"],
  "member_count": 34
}
```

### 5.5 `hierarchy.json`

```json
{
  "root_ids": ["root-00"],
  "levels": [
    ["leaf-00", "leaf-01"],
    ["parent-00", "parent-01"],
    ["root-00"]
  ],
  "edges": [
    {"child": "leaf-01", "parent": "parent-00"}
  ]
}
```

V1 schemas 可以变化，但每个 breaking change 应与代码一起提交，不需要先建设 schema registry。

## 6. 最小代码组织

建议从小开始：

```text
src/mcm/
  cli.py
  pipeline.py
  prompts.py
  artifacts.py
  metrics.py

tests/
  test_smoke.py
  test_hierarchy_invariants.py
```

允许把多个 stage 写在同一个 `pipeline.py` 中。不要为每个 stage 创建抽象基类、DI container 或 plugin registry。

## 7. V1 不实现的工程能力

- 通用 adapters；
- backend protocols；
- database / queue；
- async job orchestration；
- cache 与 checkpoint；
- resume；
- distributed execution；
- observability platform；
- stable artifact versioning；
- browser UI；
- public API server；
- deployment manifests。

如果一次 demo run 已经慢到影响迭代，可以加入最小本地 cache；但它必须由实际痛点驱动，而不是因为完整系统“通常应该有”。

## 8. 演进触发条件

只有出现以下具体事件才增加 abstraction：

- 第二个输入 schema出现：提取 `SourceAdapter`；
- 第二个 LLM / embedding provider出现：提取 backend interface；
- 第二个 clusterer出现：统一 result contract；
- 重跑成本成为瓶颈：增加 cache/checkpoint；
- 真实私有数据需求出现：启动完整 privacy/release RFC；
- JSON/Markdown 不足以理解结果：构建 explorer。

在此之前，最简单、最透明的实现优先。
