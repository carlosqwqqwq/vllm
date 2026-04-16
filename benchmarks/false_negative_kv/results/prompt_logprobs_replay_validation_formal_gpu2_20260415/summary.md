# Prompt Logprobs Selective Replay Validation

## 配置
- 模型: `/data/models_copy/llama/llama-2-7b-chat`
- family 数: 32
- `enable_selective_replay`: `True`
- `enable_sidecar`: `True`
- `enable_boundary_replay`: `True`
- `max_prompt_logprobs_replay_tokens`: 256

## Phase 汇总
### seed_prompt_logprobs
- 请求数: 32
- 平均 `physical_hit_tokens`: 0.00
- 平均 `effective_reuse_tokens`: 0.00
- 平均 phase wall time: 2.8841s
- `prompt_logprobs_use_sidecar` 请求数: 0
- `sidecar_key_ready` 请求数: 0
- `false_negative_reason` 分布: {'prompt_logprobs_skip': 32}
### probe_prompt_logprobs
- 请求数: 32
- 平均 `physical_hit_tokens`: 1088.00
- 平均 `effective_reuse_tokens`: 1072.00
- 平均 phase wall time: 0.9765s
- `prompt_logprobs_use_sidecar` 请求数: 32
- `sidecar_key_ready` 请求数: 32
- `false_negative_reason` 分布: {'no_false_negative': 32}

## Probe 配对结论
- 配对请求数: 32
- seed `physical_hit_tokens`: 0
- probe `physical_hit_tokens`: 34816
- probe `effective_reuse_tokens`: 34304

