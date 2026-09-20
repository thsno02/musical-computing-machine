# RFC 0004：V1 数据与安全边界

- **状态**：Accepted for V1
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)、[RFC 0003](0003-architecture-contracts.md)

## 0. 决定

V1 不实现完整的 privacy-preserving analysis system。

为了最快验证 embedding clustering 与 recursive hierarchy，V1 采用更窄的边界：

> 只处理 synthetic 或明确可公开、非敏感的 demo 数据；只输出聚合 cluster labels、counts 和 hierarchy；不声称匿名化，不支持私有生产数据或公开发布敏感分析结果。

完整 threat model、actor thresholds、privacy auditor、release profiles 和 access-control integration 全部后移。

## 1. 为什么这样切

Clio 原始工作把隐私视为核心目标之一，但当前仓库的 V1 核心问题是：

- 中文 facet embedding 能否聚出有意义的 groups；
- cluster label 能否准确区分；
- cluster nodes 能否逐层形成较粗 hierarchy。

在没有第一条可用聚类链路之前实现完整 privacy system，会显著增加工程面积，却不能直接回答这些问题。

这不意味着隐私不重要，而是 V1 通过限制数据和使用方式来降低风险，而不是假装已经实现完整保护。

## 2. V1 允许的数据

只允许：

- 仓库生成的 synthetic conversations；
- 明确许可用于研究和再发布的公开数据；
- 维护者确认不包含真实 secrets、账号凭证或敏感个人信息的数据。

## 3. V1 禁止的数据

不得用 V1 pipeline 处理：

- 客户、员工或真实用户的私有对话；
- 医疗、法律、财务、教育记录等敏感数据；
- 内部工单、商业机密和未公开 source code；
- 包含 access token、API key、密码或 private URL 的内容；
- 需要法律意义匿名化或正式合规保证的数据。

如果真实私有数据成为需求，必须先进入 post-V1 privacy design，而不是直接把 demo pipeline 指向生产数据。

## 4. V1 不作出的声明

不得把 V1 描述为：

- anonymous / anonymized；
- privacy-preserving；
- differential privacy；
- k-anonymous；
- embedding 后不可还原；
- 可安全处理任意真实用户数据；
- 与 Anthropic 内部 Clio/Insights 具有相同隐私保证。

推荐措辞：

> 当前 tracer bullet 仅在 synthetic / public demo data 上验证聚类与 hierarchy 方法，不提供生产隐私保证。

## 5. 最小输出边界

### 5.1 `report.md`

默认只包含：

- cluster title；
- cluster description；
- aggregate count；
- parent-child structure；
- metrics 和 warnings。

默认不包含：

- 原始 conversation；
- 完整 user message；
- record ID 列表；
- representative / contrastive examples；
- embeddings；
- model raw response。

### 5.2 中间 artifacts

`facets.jsonl`、`embeddings.npy` 和 assignments 只作为本地开发 artifacts。它们不得被描述为安全可公开的数据。

### 5.3 小 cluster

为了避免 demo report 被单个样本支配，默认只在 report 中展示：

```text
member_count >= 5
```

较小 cluster 仍可存在于本地 debug artifacts，但在 report 中标记为 suppressed。这个数字是 demo heuristic，不是隐私保证。

## 6. 最小文本检查

在生成 `report.md` 前，对所有 title / description 执行简单 detector：

- email；
- phone-like number；
- API key / token-like string；
- URL query token；
- 明显长引用。

命中时：

- 不发布对应文本到 report；
- 在 warnings 中记录；
- 由维护者检查 prompt 或 demo data。

V1 不建设通用 PII detector framework，也不保证 detector recall。

## 7. Provider 边界

- synthetic / public demo data 可以发送到 V1 选定 provider；
- model/provider 名称必须写入 `run.json`；
- 不得把 API key、完整 request body 或 model response 写入 git；
- `.env`、本地 cache 和 run artifacts 必须在 `.gitignore` 中处理；
- 不对 provider retention、training 或 region 做统一保证。

使用者有责任确认其选择的 provider 适用于 demo data。

## 8. 公开仓库边界

可以提交：

- synthetic input；
- prompts；
- config；
- aggregate hierarchy；
- demo report；
- metrics；
- 不含秘密的日志片段。

不得提交：

- credentials；
- 私有数据；
- 未经许可的对话；
- 含 raw provider payload 的 traces；
- 可识别真实个人的 examples。

任何计划公开的 demo report 在提交前进行一次人工通读。

## 9. V1 的 fail-closed 条件

下列情况不得生成“可公开”的 report：

- 输入数据来源或许可不清楚；
- 数据包含真实敏感信息；
- generated title / description 命中 token、email、phone 或长引用 detector；
- report 意外包含 raw messages；
- 维护者无法解释某个罕见 cluster 的来源；
- run artifacts 中发现 credential。

命令仍可保留本地 debug artifacts，但必须标记 `report_status: blocked`。

## 10. Post-V1 才讨论的 privacy 能力

只有真实私有数据或 public product 成为明确需求后，再设计：

- raw / derived / release trust zones；
- actor pseudonymization；
- minimum records / unique actors；
- single-actor dominance；
- metadata intersection；
- privacy-aware facet extraction；
- independent privacy auditor；
- public/private release profiles；
- access control、retention 和 deletion；
- provider policy contract；
- reconstruction / membership risk；
- formal privacy mechanisms。

这些能力应由独立 RFC 和真实 deployment assumptions 驱动，不能由 V1 的 demo heuristics 推出。

## 11. V1 安全检查表

运行前：

- [ ] 数据是 synthetic 或明确许可的 public data；
- [ ] 数据不包含 credentials 或真实敏感信息；
- [ ] provider 只接收允许发送的 demo data。

运行后：

- [ ] report 不包含 raw conversations；
- [ ] 小 cluster 已 suppress；
- [ ] generated text detector 未发现高风险内容；
- [ ] public artifacts 已人工通读；
- [ ] 文档没有声称匿名化或正式隐私保证。
