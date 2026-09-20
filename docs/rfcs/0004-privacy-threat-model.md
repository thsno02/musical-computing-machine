# RFC 0004：隐私 Threat Model 与发布边界

- **状态**：Draft
- **日期**：2026-09-20
- **依赖**：[RFC 0001](0001-project-charter.md)、[RFC 0002](0002-evaluation-protocol.md)、[RFC 0003](0003-architecture-contracts.md)

## 摘要

本项目处理的原始记录可能包含个人信息、商业机密、敏感经历和稀有事件。摘要、embedding、聚类和层次结构都不能自动消除这些风险。

本 RFC 明确：

- 项目不提供默认或形式化隐私保证；
- “privacy-preserving”不是只靠 prompt 就能获得的标签；
- 必须定义威胁、部署模式、信任边界、发布 artifact 和失败行为；
- public-release mode 的 gate 必须真正控制序列化输出，而不是生成一份可被忽略的审计报告。

## 1. 安全目标

在授权的分析任务中，系统应尽量让人看到**群体层面的 pattern**，而不是某个个体的原话、身份或罕见经历。

具体目标：

1. 减少原始数据暴露面；
2. 阻止 direct identifiers 进入 analyst-visible output；
3. 抑制过小群体、单一 actor 或稀有事件形成的可识别 cluster；
4. 限制 metadata intersection；
5. 记录第三方 provider、cache、log 和 artifact 的数据路径；
6. 对 release candidate 执行 deterministic rules、detectors、model audit 和人工抽检；
7. 默认 fail closed：无法确认符合 policy 时不发布。

## 2. 非目标与不作出的保证

V1 不声称实现：

- differential privacy；
- k-anonymity、l-diversity 或 t-closeness 的形式化保证；
- 对任意攻击者的不可重识别；
- 法律意义上的匿名化；
- 所有类型 PII 的完美检测；
- 对恶意内部人员的完整防护；
- 通过 embedding 或摘要实现的不可逆转换；
- 账号级调查、风控处置或自动执法。

若未来实现形式化机制，必须通过独立 RFC 定义 threat assumptions、参数、composition 和验证方法。

## 3. 资产

需要保护的资产包括：

- 原始记录内容；
- direct identifiers；
- actor mapping；
- 准标识符和 metadata；
- facet summaries；
- embeddings；
- cluster assignments；
- representative/contrastive sample IDs；
- model prompts/responses；
- cache、checkpoint、logs；
- private evaluation annotations；
- release 前候选标签；
- API keys、credentials 和 encryption keys。

尤其要注意：derived artifacts 仍可能包含敏感信息，不能因为不再是原文就降低访问控制。

## 4. Threat actors

### 4.1 未授权外部访问者

可能读取公开仓库、静态网站、报告、对象存储、错误暴露的 artifact 或共享 URL。

### 4.2 具有部分权限的 analyst

可访问聚合结果，但不应因此获得 raw records、actor identity 或低频 group 详情。

### 4.3 过度权限的内部用户

可能通过 cluster filter、时间切片和 metadata 组合缩小到个人。

### 4.4 第三方 model/provider

可能接收 raw records 或 derived text；风险取决于服务条款、retention、training policy、region 和部署方式。

### 4.5 恶意或提示注入型数据

原始记录可能包含指令，试图让 facet extractor / labeler 输出秘密、忽略格式或复述 PII。

### 4.6 公开数据关联攻击者

即使 cluster label 不含 direct identifier，也可能结合公开事件、时间、地区和职业重识别个人。

## 5. 主要威胁

## 5.1 Direct identifier leakage

例如姓名、邮箱、电话、地址、账号、证件号、精确 URL token、私有 repository 名称等进入：

- facet；
- cluster title/description；
- representative examples；
- report；
- logs；
- browser payload。

## 5.2 Rare-topic disclosure

一个只有少量记录的 cluster 可能准确描述某位用户的罕见事件，即使没有姓名也可被识别。

## 5.3 Single-actor dominance

某个 cluster 虽有很多 records，但都来自一个 actor。只检查 record count 不足以保护群体隐私。

## 5.4 Metadata intersection

语言、时间、地区、组织、产品版本等多个聚合 filter 组合后，可能将群体缩小为一人。

## 5.5 Label memorization / quotation

LLM 可能复制某条记录的独特措辞、代码、机密项目名或完整句子，并用它命名 cluster。

## 5.6 Embedding and vector-store exposure

Embeddings 不是匿名数据。攻击者可能通过 inversion、membership inference、nearest-neighbor probing 或已知文本匹配获得信息。

## 5.7 Provider retention and training

把 raw records 发送到外部 API 可能超出数据使用授权，或触发不符合部署要求的 retention/training/region 行为。

## 5.8 Log/cache spill

异常、debug print、trace、telemetry、notebook output 和 cache key 可能复制原始文本。

## 5.9 Prompt injection from records

模型可能把 record 中的文本误当作系统指令，输出其他 records、隐藏字段或绕过 privacy policy。

## 5.10 Reconstruction from repeated queries

允许 analyst 任意切片或反复改变 threshold，可能通过差分比较推断某条记录是否存在。

## 5.11 Misuse of aggregate analysis

即使没有直接泄露，系统也可能被用于不透明监控、绩效判断或对个体采取高影响行动。

## 6. 部署与发布模式

每个 run 必须选择一个 `release_profile`，不得隐含推断。

## 6.1 `research-public-data`

适用：

- 数据已合法公开，且许可允许分析；
- 仍需考虑数据中第三方信息和二次伤害；
- 可以在受控 UI 中查看原始 public examples，但必须显式开启。

要求：

- 标明来源与许可；
- 不因“公开”就跳过 PII/rare-topic audit；
- release report 默认仍只包含聚合结果；
- 原始 examples 与聚合 explorer 分开。

## 6.2 `private-analysis`

适用：

- 授权团队在受控环境分析自有数据；
- analyst 可看到聚合结果；
- 少数受权人员可能另行访问 raw records。

要求：

- 最小权限；
- raw/derived/release zone 分离；
- provider 路径审批；
- retention 与 deletion policy；
- audit log；
- 禁止从 explorer 无限制下钻至 raw records；
- cluster suppression 和 privacy warnings。

## 6.3 `public-release`

适用：

- 结果将进入公开网页、论文附件、公开 object storage 或对不受信任用户开放的 API。

这是最严格模式。要求：

- 默认不发布 raw records；
- 默认不发布 facet-level records；
- 默认不发布 embeddings；
- 默认不发布 representative/contrastive examples 或 IDs；
- 必须同时满足 minimum records 和 minimum unique actors；
- 必须执行 metadata intersection policy；
- 所有 text fields 通过 privacy audit；
- release payload 由 gate 生成；
- 未通过、无法评估或缺少 actor evidence 的 cluster 被 suppress；
- 公开前进行人工抽样 review；
- 发布 manifest 记录 policy version。

## 7. Actor identity 边界

`actor_id` 是执行 unique-actor threshold 的必要信息，但它本身也敏感。

要求：

- 输入层尽早把真实账号映射为稳定 pseudonym；
- mapping 与分析 artifacts 分开存储；
- release artifacts 不含 actor ID；
- 只有 record count、没有稳定 actor ID 时，不得声称满足 unique-user/actor aggregation；
- public-release mode 在缺少 actor evidence 时应 fail closed，或采用更保守、经批准的替代 policy 并明确披露 limitation。

## 8. Defense in depth

## 8.1 Layer 0：数据治理与最小化

在模型调用前明确：

- 数据使用授权和目的；
- 必需字段；
- retention；
- region；
- processor/provider；
- 谁可以访问 raw/derived/release data；
- 删除请求与 incident process。

不需要的 metadata 不应进入 pipeline。

## 8.2 Layer 1：本地 deterministic preprocessing

在可能时先本地执行：

- direct identifier detectors；
- secret/token detectors；
- metadata allowlist；
- actor pseudonymization；
- exact duplicate handling；
- file/URL/attachment policy；
- oversized content truncation with reason；
- known-canary tagging for benchmark。

Detector miss 不能由 prompt magically 修复，因此后续仍需多层 audit。

## 8.3 Layer 2：Privacy-aware facet extraction

Facet prompt/schema 应：

- 明确禁止 direct identifiers 和不必要 proper nouns；
- 要求抽象共同 request/task；
- 限制长度；
- 使用 structured output；
- 把 record content 放在明确 data delimiter 中；
- 声明 record 内指令不可信；
- 对拒答、异常和疑似注入单独标记。

Facet 输出仍留在 Zone B。

## 8.4 Layer 3：Aggregation thresholds

任何候选 release node 至少检查：

- record count；
- unique actor count；
- single-actor share；
- dominant-record / duplicate share；
- time-window concentration；
- metadata cell size；
- rare-topic risk；
- descendant suppression consistency。

本 RFC 不写死通用“安全数字”。阈值必须由 deployment policy 配置、版本化并通过 benchmark。示例 config 只能标为 demo，不得宣传为安全保证。

## 8.5 Layer 4：Contrastive but privacy-minimized labeling

Labeler 优先看到 facet representations，而不是 raw records；sample 数量受限；代表样本先经过 filter。

Label policy：

- 描述共同 pattern，不复述个例；
- 禁止 direct identifiers；
- 禁止引用式输出；
- 避免罕见 proper noun；
- 不从单条样本推断整个群体属性；
- 对不确定 cluster 生成 generic-but-useful 描述或 suppress，而非猜测。

## 8.6 Layer 5：Automated release audit

组合使用：

- deterministic PII/secret detectors；
- pattern and entropy detectors；
- proper-noun / quotation checks；
- rare-topic heuristics；
- independent privacy auditor model；
- policy engine；
- canary tests。

Auditor 输出必须驱动 gate decision，不能只是 dashboard 指标。

## 8.7 Layer 6：Human review

Public release 前：

- 审查高风险 cluster；
- 随机抽检低风险 cluster；
- 检查 filter combinations；
- 检查 browser payload；
- 检查 report、logs 和 static assets；
- 记录 reviewer、日期和决定。

## 8.8 Layer 7：访问控制与运营防护

包括：

- least privilege；
- encryption in transit/at rest；
- environment separation；
- secret management；
- audit logging；
- retention/deletion；
- download controls；
- incident response；
- provider contract/config verification。

这些能力可能超出开源 library 本身，但部署文档必须明确它们不由算法替代。

## 9. Release policy contract

概念配置：

```yaml
release_profile: public-release

artifacts:
  allow_raw_records: false
  allow_facet_records: false
  allow_embeddings: false
  allow_representative_examples: false

aggregation:
  min_records: REQUIRED_BY_DEPLOYMENT
  min_unique_actors: REQUIRED_BY_DEPLOYMENT
  max_single_actor_share: REQUIRED_BY_DEPLOYMENT
  metadata_dimensions_allowlist: []

text_audit:
  direct_identifier_detector: required
  secret_detector: required
  privacy_auditor: required
  manual_review: required

failure_mode: fail_closed
```

配置 parser 不应接受 `REQUIRED_BY_DEPLOYMENT` 进入实际 run；这是文档占位符，用来阻止未经思考的默认值。

## 10. Gate decision

每个 release candidate 产生：

```json
{
  "candidate_id": "cluster-id",
  "decision": "allow|redact|suppress|manual_review",
  "policy_version": "...",
  "checks": [],
  "reason_codes": [],
  "auditor_scores": {},
  "released_fields": [],
  "redacted_fields": [],
  "review": null
}
```

### Fail-closed 条件

至少包括：

- 缺失 policy；
- public-release 缺少必要 actor evidence；
- detector/auditor 执行失败；
- aggregation threshold 未满足；
- text audit 高风险；
- release serializer schema mismatch；
- lineage/provenance 缺失；
- manual review 被要求但未完成。

## 11. Metadata 与切片策略

即使每个 cluster 本身足够大，任意 filter 也可能产生小 cell。

要求：

- metadata dimension allowlist；
- 对每个 filter combination 重算 release eligibility；
- 不返回被 suppression cell 的精确 count；
- 限制高维组合和自由查询；
- 时间 bucket 有 minimum granularity；
- 防止通过相邻查询做 differencing；
- explorer 必须显示 suppression，而不是把缺失误画成零。

## 12. Logs、traces 与缓存

默认：

- 不记录 raw prompt body；
- error message 不回显完整 record；
- trace 使用 record hash/ID；
- cache 处于 Zone B；
- crash dump 和 notebook output 按敏感数据处理；
- telemetry opt-in；
- debug mode 明确警告并禁止用于 public-release workflow；
- 删除 run 时覆盖或删除关联 cache/index，具体能力写入 adapter 文档。

## 13. 第三方 model/provider

每个 provider adapter 必须允许记录：

- endpoint/region；
- model/revision；
- retention setting；
- training/data-use setting；
- enterprise/no-retention contract 是否由部署方确认；
- encryption；
- batch/file API 是否产生额外存储；
- request logging；
- deletion mechanism。

Library 不能替用户做法律或供应商尽调，但必须让这些信息可见并阻止“provider 不重要”的隐含假设。

## 14. Prompt injection 防护

- record content 始终视为 untrusted data；
- system/developer policy 与 record data 使用清晰 delimiter；
- extractor/labeler 不提供读取其他 records 或外部 secrets 的工具；
- structured output validation；
- 拒绝 record 要求的角色切换；
- benchmark 注入样本；
- 对异常长、含 system-like tokens 或 encoded content 的 records 标记；
- 不把 prompt injection detector 当作唯一防线。

## 15. Misuse boundary

本项目输出应支持群体研究，不应作为单独依据对个人采取高影响行动。

明确禁止把核心库宣传为：

- 自动发现“问题员工”；
- 根据私人对话自动处分用户；
- 无需人工复核的执法/风控系统；
- 证明某个个体有特定敏感属性；
- 绕过数据授权的监控工具。

若部署方把聚合分析与个体账号联动，这是本项目 core scope 之外的高风险扩展，需要独立治理、权限和审查。

## 16. 隐私评估

V1 至少包含：

- direct identifier gold set；
- synthetic canaries；
- rare event samples；
- single-actor dominated clusters；
- metadata intersection cases；
- quotation/memorization cases；
- prompt injection cases；
- multilingual PII；
- detector/auditor failure simulation；
- browser payload inspection。

报告：

- stage-by-stage leakage；
- detector precision/recall；
- auditor confusion matrix；
- gate allow/redact/suppress/manual-review distribution；
- public-release payload violations；
- false positive impact；
- unresolved risks。

## 17. 文档措辞规则

在没有完整部署证据时：

推荐：

- “privacy-oriented pipeline”；
- “具有 defense-in-depth privacy controls”；
- “在给定 policy 和 benchmark 下减少泄露”；
- “public-release profile 抑制未通过 gate 的 cluster”。

避免：

- “匿名”；
- “绝对安全”；
- “embedding 后无法还原”；
- “符合所有隐私法规”；
- “与 Clio 相同的隐私保证”；
- “只要人数达到 N 就安全”。

## 18. 待讨论事项

1. Public-release 是否必须进入 V1；
2. actor evidence 缺失时是否完全禁止 public release；
3. 默认是否只支持 local/self-hosted model，还是 provider adapters 同等支持；
4. 哪些 metadata dimensions 可以进入 explorer；
5. 是否允许发布 synthetic representative examples；
6. privacy auditor 的 model independence 要求；
7. policy/detector 失败是否在所有 profile 下 fail closed；
8. 是否为 release artifacts 增加签名与 immutable manifest；
9. 如何定义和测试 differencing attack 的 query budget；
10. 项目何时可以在限定语境中使用“privacy-preserving”表述。