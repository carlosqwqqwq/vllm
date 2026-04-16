# MT-Bench W1 FalseNegativeKV 结果摘要

## 选择结果
- 数据集: `philschmid/mt-bench`
- 原始样本数: 80
- 过阈值样本数: 80
- 最终选中请求数: 64
- 最终 family 数: 8
- prompt token 统计: min=130, mean=191.97, max=548
- 类别分布: {'coding': 8, 'extraction': 8, 'humanities': 8, 'math': 8, 'reasoning': 8, 'roleplay': 8, 'stem': 8, 'writing': 8}

## Phase 汇总
### control_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 183.25
- 平均 phase wall time: 0.1251s
### treatment_prompt_logprobs
- 请求数: 64
- 平均 `physical_hit_tokens`: 183.25
- 平均 `num_cached_tokens`: 0.00
- 平均 phase wall time: 1.5062s
### warmup_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 93.00
- 平均 phase wall time: 0.4604s

## W1 配对结论
- 配对请求数: 64
- 配对 family 数: 8
- `logical_hit_tokens`: 0
- treatment `physical_hit_tokens`: 11728
- `false_negative_tokens / logical_hit_tokens`: 0.000
