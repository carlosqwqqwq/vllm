# FalseNegativeKV 中文主文稿

更新日期：2026-04-16

这个目录保存 `FalseNegativeKV / TwinPathServe` 的中文投稿体主文稿，按章节拆分维护。

与 `paper/` 目录下其他层的关系如下：

- 本目录是当前主写作入口
- `paper/supporting/` 保存 storyline 与写作策略
- `paper/archive/` 保存旧的 `draft` 类文件归档
- 这里不再使用 `draft` 命名

推荐阅读顺序如下：

1. [00_题目与摘要.md](00_题目与摘要.md)
2. [01_引言.md](01_引言.md)
3. [02_背景与动机.md](02_背景与动机.md)
4. [03_FalseNegativeExactHit的测量.md](03_FalseNegativeExactHit的测量.md)
5. [04_契约感知复用抽象.md](04_契约感知复用抽象.md)
6. [05_TwinPathServe系统设计.md](05_TwinPathServe系统设计.md)
7. [06_实验评估.md](06_实验评估.md)
8. [07_相关工作.md](07_相关工作.md)
9. [08_讨论与结论.md](08_讨论与结论.md)

这套主文稿统一遵守以下边界：

- 强写 `W1 prompt_logprobs` 的原生系统证据
- `W5` 必须区分 vanilla runtime 侧的 cross-backend logical false-negative evidence，与 `TwinPathServe` 原型侧的公开 workload 恢复证据
- 不把 `W5` 误写成在线原生 `backend_mismatch` 已完成归因
- 不虚构 `H3/H4` 尚未完成的恢复收益数字
- 当前写作轮次只关注 `paper/` 内部文稿联动，不再把实现或实验文档作为默认修改对象

与正文直接相关的 paper-only 文档入口如下：

- [../supporting/paper_packaging_playbook_20260416.md](../supporting/paper_packaging_playbook_20260416.md)
- [../supporting/paper_storyline_fnkv.md](../supporting/paper_storyline_fnkv.md)
- [../supporting/writing_strategy.md](../supporting/writing_strategy.md)
- [../supporting/full_manuscript_consistency_audit.md](../supporting/full_manuscript_consistency_audit.md)
- [../supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md](../supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md)
- [../supporting/reviewer_question_matrix.md](../supporting/reviewer_question_matrix.md)

若确实需要核对背景规格，再按需参考：

- [../../implementation/reuse_contract_spec.md](../../implementation/reuse_contract_spec.md)
- [../../implementation/realization_policy_spec.md](../../implementation/realization_policy_spec.md)
- [../../experiments/paper_evaluation_matrix.md](../../experiments/paper_evaluation_matrix.md)
