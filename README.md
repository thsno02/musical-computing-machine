# musical-computing-machine

面向中文读者的 **Clio-style hierarchical semantic clustering** 参考实现。

> 当前状态：**specification-first / RFC 阶段**。先把问题定义、边界、评估协议和隐私承诺讲清楚，再选择具体模型与框架。

## 一句话目标

给定一批非结构化记录（优先支持 AI 对话，也可扩展到工单、反馈、文档等），通过可配置的 **facet extraction → embedding → base clustering → contrastive labeling → bottom-up hierarchy → privacy gating → evaluation** 流程，得到可解释、可下钻、可复现实验的层次化聚类结果。

## 这不是

- Anthropic 产品界面或内部安全运营系统的像素级克隆；
- 把 UMAP 散点图做出来就宣称“聚类准确”；
- 把 embedding 当成匿名化或正式隐私保证；
- 通用 RAG、知识图谱、实时流处理或生产级多租户 SaaS。

## 项目规范

项目的第一份规范是：

- [RFC 0001：复现 Clio 的问题定义、项目边界与验收标准](docs/rfcs/0001-project-charter.md)

RFC 0001 规定了：

- “复现”的对象究竟是什么；
- v1 的 in-scope / out-of-scope；
- 什么叫“准确聚类”；
- hierarchy、privacy、中文支持与可复现性的验收口径；
- paper baseline、alternative baseline 与阶段里程碑。

## 语言约定

- 文档、讨论、报告与示例以中文为主；
- 保留必要且更准确的英文术语，例如 `facet`、`embedding`、`base cluster`、`contrastive examples`、`hierarchy`、`privacy auditor`；
- 代码标识符与公共 API 优先使用英文，避免中英双份 API；
- 中文不是文档翻译项，而是独立的评估维度：必须包含中文、英文和中英混合数据的质量报告。

## 参考资料

- [Anthropic: Clio / Anthropic Insights](https://www.anthropic.com/research/clio)
- [Clio paper](https://arxiv.org/abs/2412.13678)
- [OpenClio](https://github.com/Phylliida/OpenClio)
- [Kura](https://github.com/jxnl/kura)
- [anthropic-clio-impl](https://github.com/adhishthite/anthropic-clio-impl)

## 当前协作方式

在实现代码之前，优先通过 RFC / GitHub Discussion 讨论以下问题：

1. 需求与非目标；
2. 数据与输出 contract；
3. 评估数据集、指标和人工标注 rubric；
4. 隐私 threat model 与默认发布边界；
5. 组件接口和 baseline；
6. 只有在上述内容落定后，才进入实现与性能优化。
