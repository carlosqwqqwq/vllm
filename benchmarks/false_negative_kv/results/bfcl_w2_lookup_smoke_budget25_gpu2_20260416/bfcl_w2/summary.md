# BFCL W2 FalseNegativeKV 结果摘要

## 选择结果
- 数据集: `gorilla-llm/Berkeley-Function-Calling-Leaderboard`
- 选中子集: `['multi_turn_base']`
- 原始行数: 200
- 原始请求数: 745
- 超出上下文预算请求数: 0
- 过阈值请求数: 745
- 最终选中请求数: 4
- 最终 family 数: 2
- 模型上下文上限: 32768
- prompt token 统计: min=9796, mean=9881.50, max=9963
- tool catalog 大小统计: min=32, mean=32.00, max=32
- 选中 subset 分布: {'multi_turn_base': 4}
- 选中 turn 分布: {'0': 2, '1': 2}
- 选中 tool class 分布: {'GorillaFileSystem': 4, 'TwitterAPI': 4}

## Phase 汇总
### control_generation
- 请求数: 4
- 平均 `physical_hit_tokens`: 9876.00
- 平均 `num_cached_tokens`: 9876.00
- 平均 phase wall time: 0.0867s
### treatment_prompt_logprobs
- 请求数: 4
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 0.00
- 平均 phase wall time: 3.9658s
### warmup_generation
- 请求数: 4
- 平均 `physical_hit_tokens`: 7296.00
- 平均 `num_cached_tokens`: 7296.00
- 平均 phase wall time: 0.9196s

## W2 配对结论
- 配对请求数: 4
- 配对 family 数: 2
- `logical_hit_tokens`: 39504
- treatment `physical_hit_tokens`: 0
- `false_negative_tokens / logical_hit_tokens`: 1.000
