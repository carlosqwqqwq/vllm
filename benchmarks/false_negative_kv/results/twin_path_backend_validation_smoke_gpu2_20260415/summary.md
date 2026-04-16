# TwinPathServe Backend Mismatch Validation

## 配置
- 模型: `/data/models_copy/llama/llama-2-7b-chat`
- family 数: 2
- reuse pool backend: `FLASHINFER`
- throughput pool backend: `FLASH_ATTN`
- throughput pool `batch_invariant`: `True`
- 每池 `gpu_memory_utilization`: 0.24

## Phase 汇总
### reuse_seed
- 请求数: 2
- 平均 `physical_hit_tokens`: 0.00
- 平均 phase wall time: 0.1841s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 2}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 2}`
- `false_negative_reason_counts`: `{'no_false_negative': 2}`

### reuse_reference_probe
- 请求数: 2
- 平均 `physical_hit_tokens`: 1088.00
- 平均 phase wall time: 0.0477s
- `route_reason_counts`: `{'forced_reuse_optimized_pool': 2}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 2}`
- `false_negative_reason_counts`: `{'no_false_negative': 2}`

### baseline_throughput_probe
- 请求数: 2
- 平均 `physical_hit_tokens`: 0.00
- 平均 phase wall time: 0.2411s
- `route_reason_counts`: `{'forced_throughput_optimized_pool': 2}`
- `selected_pool_counts`: `{'throughput_optimized_pool': 2}`
- `false_negative_reason_counts`: `{'no_false_negative': 2}`

### twin_path_backend_probe
- 请求数: 2
- 平均 `physical_hit_tokens`: 1088.00
- 平均 phase wall time: 0.0881s
- `route_reason_counts`: `{'backend_mismatch': 2}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 2}`
- `false_negative_reason_counts`: `{'no_false_negative': 2}`

### twin_path_family_binding_probe
- 请求数: 2
- 平均 `physical_hit_tokens`: 1088.00
- 平均 phase wall time: 0.0725s
- `route_reason_counts`: `{'family_binding': 2}`
- `selected_pool_counts`: `{'reuse_optimized_pool': 2}`
- `false_negative_reason_counts`: `{'no_false_negative': 2}`

## 核心对比
- baseline throughput probe 平均 `physical_hit_tokens`: 0.00
- twin-path backend probe 平均 `physical_hit_tokens`: 1088.00
- twin-path family-binding probe 平均 `physical_hit_tokens`: 1088.00
- baseline throughput probe 平均 phase wall time: 0.2411s
- twin-path backend probe 平均 phase wall time: 0.0881s
- twin-path family-binding probe 平均 phase wall time: 0.0725s
