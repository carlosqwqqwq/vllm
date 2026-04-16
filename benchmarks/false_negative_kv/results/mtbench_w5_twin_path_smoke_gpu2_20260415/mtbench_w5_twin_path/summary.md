# MT-Bench W5 TwinPathServe 结果摘要

## 选择结果
- 数据集: `philschmid/mt-bench`
- 原始样本数: 80
- 过阈值样本数: 80
- 最终选中请求数: 4
- 最终 family 数: 2
- prompt token 统计: min=130, mean=191.97, max=548
- 类别分布: {'roleplay': 2, 'writing': 2}

## Phase 汇总
### baseline_throughput_probe
- 请求数: 4
- 平均 `physical_hit_tokens`: 64.00
- 平均 phase wall time: 0.1001s
- `route_reason_counts`: `{'forced_throughput_optimized_pool': 4}`
- `selected_pool_counts`: `{'throughput_optimized_pool': 4}`
- `false_negative_reason_counts`: `{'no_false_negative': 4}`
### reuse_reference_probe
- 请求数: 4
- 平均 `physical_hit_tokens`: 148.00
- 平均 phase wall time: 0.0352s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 4}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 4}`
- `false_negative_reason_counts`: `{'no_false_negative': 4}`
### reuse_seed
- 请求数: 4
- 平均 `physical_hit_tokens`: 64.00
- 平均 phase wall time: 0.0845s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 4}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 4}`
- `false_negative_reason_counts`: `{'no_false_negative': 4}`
### twin_path_backend_probe
- 请求数: 4
- 平均 `physical_hit_tokens`: 148.00
- 平均 phase wall time: 0.0425s
- `route_reason_counts`: `{'backend_mismatch': 4}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 4}`
- `false_negative_reason_counts`: `{'no_false_negative': 4}`
### twin_path_family_binding_probe
- 请求数: 4
- 平均 `physical_hit_tokens`: 148.00
- 平均 phase wall time: 0.0407s
- `route_reason_counts`: `{'family_binding': 4}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 4}`
- `false_negative_reason_counts`: `{'no_false_negative': 4}`

## W5 配对结论
- 配对请求数: 4
- `logical_hit_tokens`: 592
- baseline throughput `physical_hit_tokens`: 256
- twin-path backend `physical_hit_tokens`: 592
- twin-path family-binding `physical_hit_tokens`: 592
- baseline `false_negative_tokens / logical_hit_tokens`: 0.568
- twin-path backend recovery ratio: 1.000
- twin-path family-binding recovery ratio: 1.000
