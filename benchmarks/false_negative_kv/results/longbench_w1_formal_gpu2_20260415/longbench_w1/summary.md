# LongBench W1 FalseNegativeKV 结果摘要

## 选择结果
- 数据集: `THUDM/LongBench`
- `data.zip`: `/data/infinigen/accuracy/LongBench/LongBench/longbench_data/data.zip`
- 可用 task 数: 34
- 选中 task: ['qmsum', 'narrativeqa', 'qasper', 'multifieldqa_zh', 'multifieldqa_en']
- 原始样本数: 950
- 模型长度过滤后样本数: 826
- 超过模型长度样本数: 124
- 过阈值样本数: 713
- 最终选中请求数: 64
- 最终 family 数: 32
- prompt token 统计: min=1472, mean=10191.84, max=29094
- 语言分布: {'en': 626, 'zh': 200}
- 选中 task 分布: {'qmsum': 64}
- 模型上下文上限: 32768
- prompt token budget: 32767

## Phase 汇总
### control_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 7554.75
- 平均 `num_cached_tokens`: 7554.75
- 平均 phase wall time: 44.1222s
- `false_negative_reason_counts`: `{'no_false_negative': 64}`
### treatment_prompt_logprobs
- 请求数: 64
- 平均 `physical_hit_tokens`: 0.00
- 平均 `num_cached_tokens`: 0.00
- 平均 phase wall time: 94.8756s
- `false_negative_reason_counts`: `{'prompt_logprobs_skip': 64}`
### warmup_generation
- 请求数: 64
- 平均 `physical_hit_tokens`: 7553.75
- 平均 `num_cached_tokens`: 7553.75
- 平均 phase wall time: 43.9454s
- `false_negative_reason_counts`: `{'no_false_negative': 64}`

## W1 配对结论
- 配对请求数: 64
- 配对 family 数: 32
- 配对 task 分布: {'qmsum': 64}
- `logical_hit_tokens`: 483504
- treatment `physical_hit_tokens`: 0
- `false_negative_tokens / logical_hit_tokens`: 1.000
