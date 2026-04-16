# FalseNegativeKV 论文目录

更新日期：2026-04-16

## 1. 这个目录的作用

这个目录只负责一件事：

- 把 `FalseNegativeKV` 从执行型规格，推进到可投稿论文的写作材料

它不负责：

- 再发散新 thesis
- 再重写 measurement spec
- 再做高层 brainstorming

从现在开始：

- `14_beyond_current_ideas/` 根目录负责 thesis 规格、评测矩阵和系统边界
- `paper/` 子目录负责论文主文稿、supporting 文档和归档草稿

## 2. 当前主入口

当前论文写作的主入口已经切换为：

- [manuscript_cn/README.md](manuscript_cn/README.md)
  - 按章节拆分的中文主文稿，作为当前最成熟的正文版本
- [supporting/paper_packaging_playbook_20260416.md](supporting/paper_packaging_playbook_20260416.md)
  - 当前 paper-only 写作模式的一页式包装手册，固定题目、摘要、引言、贡献与 reviewer 第一眼风险
- [supporting/paper_storyline_fnkv.md](supporting/paper_storyline_fnkv.md)
  - reviewer 导向的 paper storyline 锁定稿
- [supporting/writing_strategy.md](supporting/writing_strategy.md)
  - 当前最稳的标题策略、claim 边界与学术写法约束

旧的 `draft` 类文件保留为归档参考，不再作为默认主入口。

从现在开始，`paper/` 内部默认采用 `paper-only` 工作模式：

- 只改主文稿与 supporting 写作材料
- 不再把 `implementation/` 与 `experiments/` 文档当作默认写作入口
- 如需引用外部规格，只作为背景参考，而不作为当前写作工作流的一部分

## 3. 当前写作姿态

当前最稳的写作姿态不是“完整系统已经闭环取胜”，而是：

- `measurement-backed problem paper`
- `design-forward system thesis`
- `contract-and-realization-forward system thesis`

更具体地说：

1. `W1 prompt_logprobs` 已经是强证据，可以直接写成问题表征的主例子
2. `W5 backend mismatch` 在 vanilla runtime 侧仍应稳妥写成
   `cross-backend logical false negative`，
   但在 `TwinPathServe` 原型侧已经有 `MT-Bench W5` 公开 workload 的恢复证据
3. `TwinPathServe` 现在可以作为主设计和目标系统来写
4. 当前已经有两条主机制的 formal 结果，但摘要和引言仍不能提前声称：
   - 在线统一恢复已经完整成立
   - 多 benchmark 端到端收益已经闭环证明

## 4. 当前目录结构

- [manuscript_cn/README.md](manuscript_cn/README.md)
  - 中文投稿体主文稿目录，按章节拆分维护
- [supporting/paper_storyline_fnkv.md](supporting/paper_storyline_fnkv.md)
  - 目标 full-thesis 版本的标题、摘要骨架、五张主图与 reviewer rebuttal
- [supporting/writing_strategy.md](supporting/writing_strategy.md)
  - 当前最稳的论文定位、标题策略、claim 边界和升级条件
- [supporting/paper_packaging_playbook_20260416.md](supporting/paper_packaging_playbook_20260416.md)
  - 只聚焦论文包装与写作的一页式手册，固定题目、摘要、引言、贡献与 reviewer 第一眼会抓的几个点
- [supporting/vllm_isolation_cases_and_bridgeability.md](supporting/vllm_isolation_cases_and_bridgeability.md)
  - `vLLM` 中“复用隔离”实例的源码级说明，固定回答 reviewer 关于安全复用、隔离原因与 `hash` 充分性的质疑
- [supporting/safe_reuse_soundness_note.md](supporting/safe_reuse_soundness_note.md)
  - `safe reuse` 的 `soundness` 说明，固定回答 reviewer 关于“不安全复用是否会影响输出、理论能否证明安全”的质疑
- [supporting/sglang_capability_boundary_cases.md](supporting/sglang_capability_boundary_cases.md)
  - `SGLang` 中 capability boundary 的实例说明，固定回答“这是不是 `vLLM` 独有工程问题”的质疑
- [supporting/reviewer_question_matrix.md](supporting/reviewer_question_matrix.md)
  - reviewer 高频问题与主文稿/ supporting 覆盖关系的矩阵，用于后续文稿 hardening
- [supporting/reviewer_critique_analysis_20260415.md](supporting/reviewer_critique_analysis_20260415.md)
  - 对一份高强度系统审稿意见的逐条分析、当前成立问题与后续补强优先级
- [supporting/reviewer_driven_elevation_strategy_20260415.md](supporting/reviewer_driven_elevation_strategy_20260415.md)
  - 在承认 reviewer 有效批评的前提下，如何提标而不降格、扩大范围与抬高贡献的执行方案
- [supporting/osdi_elevation_linkage_plan_20260415.md](supporting/osdi_elevation_linkage_plan_20260415.md)
  - 说明当前方向为什么单独还不够、以及如何把 `measurement -> contract -> realization -> runtime` 联成可支撑 `CCF-A / OSDI` 工作量的完整 thesis
- [supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md](supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md)
  - bridge I 的 correctness 审计方案，专门回答 `prompt_logprobs selective replay` 是否已经完成 output-equivalence 审计
- [supporting/reviewer_revision_log_20260415.md](supporting/reviewer_revision_log_20260415.md)
  - 本轮围绕 reviewer 核心质疑所做的主文稿修订记录，说明已经改了什么、还缺什么
- [supporting/submission_readiness_and_next_steps.md](supporting/submission_readiness_and_next_steps.md)
  - 当前文稿已能回答什么、仍有哪些问题必须靠后续实验补齐的准备度说明
- [supporting/related_work_landscape_and_positioning.md](supporting/related_work_landscape_and_positioning.md)
  - 相关工作全景、近邻工作分类和与 `FalseNegativeKV` 的定位边界，供第七章与 rebuttal 复用
- [supporting/full_manuscript_consistency_audit.md](supporting/full_manuscript_consistency_audit.md)
  - `00-08` 主文稿的一致性审计、统计口径约定与当前仍未闭环的证据缺口
- [archive/README.md](archive/README.md)
  - 旧的 draft 类文件归档说明
- [../implementation/prompt_logprobs_selective_replay_design.md](../implementation/prompt_logprobs_selective_replay_design.md)
  - 第一个 bridge 的实现设计稿，用来约束摘要/引言中对 `prompt_logprobs selective replay` 的写法和 prototype 边界

## 5. 推荐阅读顺序

如果要继续写论文，按下面顺序读：

1. [manuscript_cn/README.md](manuscript_cn/README.md)
2. [supporting/paper_packaging_playbook_20260416.md](supporting/paper_packaging_playbook_20260416.md)
3. [supporting/writing_strategy.md](supporting/writing_strategy.md)
4. [supporting/paper_storyline_fnkv.md](supporting/paper_storyline_fnkv.md)
5. [supporting/full_manuscript_consistency_audit.md](supporting/full_manuscript_consistency_audit.md)
6. [supporting/reviewer_question_matrix.md](supporting/reviewer_question_matrix.md)
7. [supporting/reviewer_revision_log_20260415.md](supporting/reviewer_revision_log_20260415.md)

如果只做论文包装与写作，到这里为止即可。

如果确实需要补充背景规格，再按需参考：

1. [../implementation/reuse_contract_spec.md](../implementation/reuse_contract_spec.md)
2. [../implementation/realization_policy_spec.md](../implementation/realization_policy_spec.md)
3. [../experiments/paper_evaluation_matrix.md](../experiments/paper_evaluation_matrix.md)
4. [../implementation/prompt_logprobs_selective_replay_design.md](../implementation/prompt_logprobs_selective_replay_design.md)

## 6. 当前写作原则

从现在开始，paper 草稿统一遵守下面几条原则：

1. 先把问题写厚，再把系统写强，不反过来
2. `W1` 可以强写成 runtime pathology 的直接证据
3. `W5` 必须区分 vanilla runtime 侧的 logical evidence，与 `TwinPathServe` 原型侧的公开 workload 恢复证据
4. 在没有多 benchmark 和完整 baseline matrix 之前，不提前写“系统收益已完整闭环”
5. 标题、摘要、引言都要围绕 `false-negative exact hit`，而不是围绕单个 feature patch
6. 当前轮次默认只修改 `paper/` 内的文稿与 supporting，不再扩散到实现或实验文档

## 7. 当前判断

如果今天就开始写论文，最合理的路径不是直接写终稿，而是：

1. 先固定 claim 边界
2. 先写一版稳的摘要和引言
3. 让后续实现和实验去“填充已有叙事”
4. 避免后面因为证据不够而重写整篇论文的 framing
