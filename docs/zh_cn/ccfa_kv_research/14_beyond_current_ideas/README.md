# FalseNegativeKV 主线总入口

更新日期：2026-04-16

## 1. 当前主线状态

从现在开始，这个目录只服务一条 active thesis：

- 问题名：`FalseNegativeKV`
- 系统名：`TwinPathServe`
- 当前阶段：`measurement-first`

这意味着：

- 不再并行保留第二条主 thesis
- 不再继续发散新的高层 idea
- 不再新增“多个方向横向比较”类文档

只有当 measurement gate 明确失败时，才重新开放选题。

## 2. 当前目录结构

当前目录按四类内容收口：

- [paper/README.md](paper/README.md)
  - 论文文稿目录，当前中文主文稿和相关写作材料都在这里
- [experiments/README.md](experiments/README.md)
  - 实验计划、实验报告和原始日志入口
- [implementation/README.md](implementation/README.md)
  - 实现规格、系统设计、prototype 设计和推进计划
- [ideas/README.md](ideas/README.md)
  - idea 深挖、边界判断与早期论证材料

根目录从现在开始只保留：

- 总 README
- 四个子目录入口

不再在根目录散放 paper、实验、实现和 idea 文档。

## 3. 当前最应该看的内容

如果只关心当前主线，建议按下面顺序读：

1. [master_execution_plan_20260416.md](master_execution_plan_20260416.md)
2. [implementation/false_negative_instrumentation_spec.md](implementation/false_negative_instrumentation_spec.md)
3. [implementation/reuse_contract_spec.md](implementation/reuse_contract_spec.md)
4. [implementation/realization_policy_spec.md](implementation/realization_policy_spec.md)
5. [experiments/paper_evaluation_matrix.md](experiments/paper_evaluation_matrix.md)
6. [experiments/unified_baseline_matrix_design_20260416.md](experiments/unified_baseline_matrix_design_20260416.md)
7. [implementation/twin_path_serve_minimal_architecture.md](implementation/twin_path_serve_minimal_architecture.md)
8. [paper/supporting/paper_storyline_fnkv.md](paper/supporting/paper_storyline_fnkv.md)
9. [paper/supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md](paper/supporting/prompt_logprobs_output_equivalence_audit_plan_20260416.md)
10. [paper/manuscript_cn/README.md](paper/manuscript_cn/README.md)

如果只关心当前实验证据，建议读：

1. [experiments/preexperiment_plan.md](experiments/preexperiment_plan.md)
2. [experiments/preexperiment_review_report_20260415_0910.md](experiments/preexperiment_review_report_20260415_0910.md)
3. [experiments/unified_baseline_matrix_design_20260416.md](experiments/unified_baseline_matrix_design_20260416.md)
4. [experiments/raw_logs_index.md](experiments/raw_logs_index.md)

如果只关心当前 prototype 落地，建议读：

1. [implementation/false_negative_instrumentation_spec.md](implementation/false_negative_instrumentation_spec.md)
2. [implementation/reuse_contract_spec.md](implementation/reuse_contract_spec.md)
3. [implementation/realization_policy_spec.md](implementation/realization_policy_spec.md)
4. [implementation/prompt_logprobs_selective_replay_design.md](implementation/prompt_logprobs_selective_replay_design.md)
5. [implementation/twin_path_serve_minimal_architecture.md](implementation/twin_path_serve_minimal_architecture.md)
6. [implementation/false_negative_kv_hardening_plan.md](implementation/false_negative_kv_hardening_plan.md)

## 4. 术语锁定

从现在开始，`FalseNegativeKV` 的新文档统一只使用下面这组术语：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`
- `reuse contract`
- `bridge operator`

不再发明新的同义词。

## 5. 范围锁定

当前主线的默认边界固定为：

- 单机或 colocated `vLLM`
- exact / logical-exact reuse
- 先做 measurement，再做最小 P1 recovery
- 当前阶段不引入任何用户可见 API 变更

当前明确写成 out-of-scope 的内容包括：

- PIC / arbitrary-position reuse
- hidden cache / activation restoration
- workflow-aware KV retention
- cross-model prefill sharing
- distributed cache pool

## 6. 维护约定

这个目录从现在开始只负责五件事：

1. 维护 `FalseNegativeKV` 的总入口与目录结构
2. 维护 `TwinPathServe` 的执行型规格与系统定义
3. 维护实验计划、实验报告和日志入口
4. 维护论文文稿
5. 维护 measurement gate 与降级规则

不再在这里新增：

- root 级散落文档
- 新 thesis 比较文档
- 新候选方向清单
- 新一轮高层 brainstorming

如果 measurement gate 失败，就直接在这里写明降级结论，而不是重新堆一轮元分析。
