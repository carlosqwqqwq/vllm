# FalseNegativeKV 预实验首轮结果

## w1_prompt_logprobs
- 请求数: 24
- control 平均 `physical_hit_tokens`: 1072.00
- `false_negative_reason` 分布: {'no_false_negative': 16, 'prompt_logprobs_skip': 8}

## w5_backend_batch_invariant
- 请求数: 8
- control 平均 `physical_hit_tokens`: 0.00
- `false_negative_reason` 分布: {'no_false_negative': 8}

## w5_backend_control
- 请求数: 16
- control 平均 `physical_hit_tokens`: 1072.00
- `false_negative_reason` 分布: {'no_false_negative': 16}

## W1 配对结论
- 配对 family 数: 8
- 推导 `logical_hit_tokens`: 8576
- treatment `physical_hit_tokens`: 0
- 推导 `false_negative_tokens / logical_hit_tokens`: 1.000

## W5 配对结论
- 配对 family 数: 8
- baseline 推导 `logical_hit_tokens`: 8576
- batch-invariant treatment `physical_hit_tokens`: 7392
- 推导 `false_negative_tokens / logical_hit_tokens`: 0.138
- engine config disable reason: [None]

## 初步判断
- `W1` 支持 `H1` 的最小门槛。
- `W5` 支持 backend mismatch 作为主实例化方向。
