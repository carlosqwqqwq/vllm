# LongBench W1 FalseNegativeKV 结果摘要

## 选择结果
- 数据集: `THUDM/LongBench`
- `data.zip`: `/data/infinigen/accuracy/LongBench/LongBench/longbench_data/data.zip`
- 可用 task 数: 34
- 选中 task: ['qmsum']
- 原始样本数: 200
- 模型长度过滤后样本数: 191
- 超过模型长度样本数: 9
- 过阈值样本数: 190
- 最终选中请求数: 4
- 最终 family 数: 2
- prompt token 统计: min=3161, mean=15155.96, max=29094
- 语言分布: {'en': 191}
- 选中 task 分布: {'qmsum': 4}
- 模型上下文上限: 32768
- prompt token budget: 32767

## Phase 汇总
### control_generation
- 请求数: 4
- 平均 `physical_hit_tokens`: 11848.00
- 平均 `num_cached_tokens`: 11848.00
- 平均 phase wall time: 0.0985s
- `false_negative_reason_counts`: `{'no_false_negative': 4}`
### treatment_prompt_logprobs
- 请求数: 4
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 0.00
- 平均 phase wall time: 4.9276s
- `false_negative_reason_counts`: `{'prompt_logprobs_skip': 4}`
### warmup_generation
- 请求数: 4
- 平均 `physical_hit_tokens`: 5928.00
- 平均 `num_cached_tokens`: 5928.00
- 平均 phase wall time: 2.0591s
- `false_negative_reason_counts`: `{'no_false_negative': 4}`

## W1 配对结论
- 配对请求数: 4
- 配对 family 数: 2
- 配对 task 分布: {'qmsum': 4}
- `logical_hit_tokens`: 47392
- treatment `physical_hit_tokens`: 0
- `false_negative_tokens / logical_hit_tokens`: 1.000
