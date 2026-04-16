# FalseNegativeKV 实验目录

更新日期：2026-04-16

这个目录统一收纳 `FalseNegativeKV` 当前主线的实验计划、实验报告和实验日志入口。

## 当前内容

- [formal_experiment_report_20260415_gpu2.md](formal_experiment_report_20260415_gpu2.md)
  - 当前最新的正式实验报告，覆盖 `W1 prompt_logprobs selective replay`
    与 `W5 TwinPathServe backend mismatch` 的 GPU2 结果
- [paper_evaluation_matrix.md](paper_evaluation_matrix.md)
  - 论文级评测矩阵、`H1-H4`、workload family、baseline 和 go / no-go gate
- [unified_baseline_matrix_design_20260416.md](unified_baseline_matrix_design_20260416.md)
  - 把 `W1/W2/W5` 的 recovery 结果、`W3/W4` 的 prevalence 结果和 `best_ad_hoc_fix` / `TwinPathServe_P1` 的统一比较，收敛成 reviewer 可直接阅读的 baseline matrix 设计
- [public_benchmark_evaluation_protocol.md](public_benchmark_evaluation_protocol.md)
  - 公开 benchmark 优先的 workload 绑定协议、允许的最小重放方式和 benchmark shortlist
- [benchmark_instantiation_plan.md](benchmark_instantiation_plan.md)
  - 逐个公开 benchmark 的实例化计划、family 构造规则、仓库支持现状和脚本排期
- [preexperiment_plan.md](preexperiment_plan.md)
  - 第一轮预实验计划、场景设计、门槛与 kill criteria
- [preexperiment_review_report_20260415_0910.md](preexperiment_review_report_20260415_0910.md)
  - 当前唯一有效的预实验复核报告
- [raw_logs_index.md](raw_logs_index.md)
  - 原始实验结果目录、脚本入口和日志位置索引

## 维护约定

- 研究叙事中引用的正式实验报告统一放在这个目录
- 原始 JSONL traces、manifest 和 request outputs 继续保留在 `benchmarks/false_negative_kv/results/`
- 如果后续新增新一轮实验，优先在这里补计划、报告与日志索引，而不是再把报告散落到别处
