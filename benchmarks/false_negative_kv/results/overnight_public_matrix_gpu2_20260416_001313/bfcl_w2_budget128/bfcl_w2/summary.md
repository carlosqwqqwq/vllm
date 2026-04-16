# BFCL W2 FalseNegativeKV 结果摘要

## 选择结果
- 数据集: `gorilla-llm/Berkeley-Function-Calling-Leaderboard`
- 选中子集: `['multi_turn_base', 'multi_turn_composite', 'multi_turn_long_context', 'multi_turn_miss_func', 'multi_turn_miss_param']`
- 原始行数: 1000
- 原始请求数: 4528
- 超出上下文预算请求数: 0
- 过阈值请求数: 4528
- 最终选中请求数: 64
- 最终 family 数: 32
- 模型上下文上限: 32768
- prompt token 统计: min=5331, mean=8681.84, max=12243
- tool catalog 大小统计: min=18, mean=28.94, max=36
- 选中 subset 分布: {'multi_turn_base': 61, 'multi_turn_composite': 3}
- 选中 turn 分布: {'0': 16, '1': 14, '2': 14, '3': 12, '4': 8}
- 选中 tool class 分布: {'GorillaFileSystem': 48, 'MathAPI': 10, 'MessageAPI': 18, 'TicketAPI': 10, 'TwitterAPI': 14, 'VehicleControlAPI': 16}

## Phase 汇总
### control_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 8673.50
- 平均 `num_cached_tokens`: 8673.50
- 平均 phase wall time: 0.4764s
### treatment_prompt_logprobs
- 请求数: 64
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 0.00
- 平均 phase wall time: 47.1557s
### warmup_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 7801.00
- 平均 `num_cached_tokens`: 7801.00
- 平均 phase wall time: 4.2949s

## W2 配对结论
- 配对请求数: 64
- 配对 family 数: 32
- `logical_hit_tokens`: 555104
- treatment `physical_hit_tokens`: 0
- `false_negative_tokens / logical_hit_tokens`: 1.000
