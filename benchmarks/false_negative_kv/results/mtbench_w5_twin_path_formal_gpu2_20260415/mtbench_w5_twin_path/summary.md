# MT-Bench W5 TwinPathServe 结果摘要

## 选择结果
- 数据集: `philschmid/mt-bench`
- 原始样本数: 80
- 过阈值样本数: 80
- 最终选中请求数: 64
- 最终 family 数: 8
- prompt token 统计: min=130, mean=191.97, max=548
- 类别分布: {'coding': 8, 'extraction': 8, 'humanities': 8, 'math': 8, 'reasoning': 8, 'roleplay': 8, 'stem': 8, 'writing': 8}

## Phase 汇总
### baseline_throughput_probe
- 请求数: 64
- 平均 `physical_hit_tokens`: 93.00
- 平均 phase wall time: 0.6208s
- `route_reason_counts`: `{'forced_throughput_optimized_pool': 64}`
- `selected_pool_counts`: `{'throughput_optimized_pool': 64}`
- `false_negative_reason_counts`: `{'no_false_negative': 64}`
### reuse_reference_probe
- 请求数: 64
- 平均 `physical_hit_tokens`: 183.25
- 平均 phase wall time: 0.1474s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 64}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 64}`
- `false_negative_reason_counts`: `{'no_false_negative': 64}`
### reuse_seed
- 请求数: 64
- 平均 `physical_hit_tokens`: 93.00
- 平均 phase wall time: 0.4698s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 64}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 64}`
- `false_negative_reason_counts`: `{'no_false_negative': 64}`
### twin_path_backend_probe
- 请求数: 64
- 平均 `physical_hit_tokens`: 183.25
- 平均 phase wall time: 0.0910s
- `route_reason_counts`: `{'backend_mismatch': 64}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 64}`
- `false_negative_reason_counts`: `{'no_false_negative': 64}`
### twin_path_family_binding_probe
- 请求数: 64
- 平均 `physical_hit_tokens`: 183.25
- 平均 phase wall time: 0.0896s
- `route_reason_counts`: `{'family_binding': 64}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 64}`
- `false_negative_reason_counts`: `{'no_false_negative': 64}`

## W5 配对结论
- 配对请求数: 64
- `logical_hit_tokens`: 11728
- baseline throughput `physical_hit_tokens`: 5952
- twin-path backend `physical_hit_tokens`: 11728
- twin-path family-binding `physical_hit_tokens`: 11728
- baseline `false_negative_tokens / logical_hit_tokens`: 0.492
- twin-path backend recovery ratio: 1.000
- twin-path family-binding recovery ratio: 1.000
