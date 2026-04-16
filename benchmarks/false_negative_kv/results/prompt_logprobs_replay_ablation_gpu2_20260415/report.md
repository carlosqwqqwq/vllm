# Prompt Logprobs Selective Replay 消融报告

## 结果总表
### baseline_off
- 目录: `/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_off_gpu2_20260415_210200`
- 开关: `selective_replay=False`, `sidecar=False`, `boundary_replay=False`, `max_replay_tokens=256`
- probe `physical_hit_tokens`: 0
- probe `effective_reuse_tokens`: 0
- probe 平均 `effective_reuse_tokens`: 0.00
- probe `prompt_logprobs_use_sidecar` 请求数: 0
- probe `sidecar_key_ready` 请求数: 0
- probe 平均 phase wall time: 0.7521s
- probe `false_negative_reason` 分布: {'prompt_logprobs_skip': 8}
- 相对 `baseline_off` 的 phase 时间变化: 0.0%

### sidecar_only
- 目录: `/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_sidecar_only_gpu2_20260415_210620`
- 开关: `selective_replay=False`, `sidecar=True`, `boundary_replay=False`, `max_replay_tokens=256`
- probe `physical_hit_tokens`: 0
- probe `effective_reuse_tokens`: 0
- probe 平均 `effective_reuse_tokens`: 0.00
- probe `prompt_logprobs_use_sidecar` 请求数: 0
- probe `sidecar_key_ready` 请求数: 0
- probe 平均 phase wall time: 0.7954s
- probe `false_negative_reason` 分布: {'prompt_logprobs_skip': 8}
- 相对 `baseline_off` 的 phase 时间变化: 5.8%

### full_replay_budget_25
- 目录: `/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_budget25_gpu2_20260415_210705`
- 开关: `selective_replay=True`, `sidecar=True`, `boundary_replay=True`, `max_replay_tokens=25`
- probe `physical_hit_tokens`: 8704
- probe `effective_reuse_tokens`: 0
- probe 平均 `effective_reuse_tokens`: 0.00
- probe `prompt_logprobs_use_sidecar` 请求数: 0
- probe `sidecar_key_ready` 请求数: 8
- probe 平均 phase wall time: 0.7543s
- probe `false_negative_reason` 分布: {'no_false_negative': 8}
- 相对 `baseline_off` 的 phase 时间变化: 0.3%

### full_replay_budget_256
- 目录: `/data/projects/KVresearch/vllm/benchmarks/false_negative_kv/results/prompt_logprobs_replay_validation_on_gpu2_20260415_210240`
- 开关: `selective_replay=True`, `sidecar=True`, `boundary_replay=True`, `max_replay_tokens=256`
- probe `physical_hit_tokens`: 8704
- probe `effective_reuse_tokens`: 8576
- probe 平均 `effective_reuse_tokens`: 1072.00
- probe `prompt_logprobs_use_sidecar` 请求数: 8
- probe `sidecar_key_ready` 请求数: 8
- probe 平均 phase wall time: 0.2021s
- probe `false_negative_reason` 分布: {'no_false_negative': 8}
- 相对 `baseline_off` 的 phase 时间变化: -73.1%

## 判读
- `sidecar_only` 没有形成有效复用，说明该配置不足以让 bridge 落地。
- `sidecar_only` 相对 baseline 的 probe phase 时间变化为 5.8%。
- `full_replay_budget_25` 没有形成有效复用，说明该配置不足以让 bridge 落地。
- `full_replay_budget_25` 相对 baseline 的 probe phase 时间变化为 0.3%。
- `full_replay_budget_256` 恢复了有效复用，说明该配置满足 bridge 的必要条件。
- `full_replay_budget_256` 相对 baseline 的 probe phase 时间变化为 -73.1%。

