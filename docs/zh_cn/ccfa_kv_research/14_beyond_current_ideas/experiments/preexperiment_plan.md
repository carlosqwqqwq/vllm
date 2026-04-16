# FalseNegativeKV 预实验计划

## 目标

这轮预实验只回答一个问题：`false-negative exact hit` 在当前 `vLLM`
里是否真实、稳定、且值得继续投入做恢复机制。

当前不追求完整系统收益，只追求先把问题测实。

## 预实验假设

### H1: `prompt_logprobs` 会制造稳定的 false-negative miss

- 同一 `prompt family` 下，普通 generation 请求应形成较高物理命中。
- 同一 `prompt family` 下，开启 `prompt_logprobs` 后，命中会显著下降。
- 这类下降不是语义不相同造成，而是 `API contract mismatch` 造成。

### H2: `backend mismatch` 会制造稳定的 false-negative miss

- 同一 `prompt family` 下，normal backend 应形成较高物理命中。
- 同一 `prompt family` 下，`VLLM_BATCH_INVARIANT=1` 时，prefix reuse 会退化。
- 这类退化不是 workload 变化造成，而是 `backend compatibility class`
 变化造成。

### H3: 这两类 miss 都属于后续可恢复的 `bridgeable miss`

- `prompt_logprobs` 属于 `api_contract_mismatch`
- `batch invariant` 属于 `backend_mismatch`
- 两类都应在 trace 中被标为 `bridge_candidate=true`

## 场景

### W1: `prompt_logprobs`

- 对照组：普通 generation
- 实验组：`prompt_logprobs=1`
- family 数量：默认 `8`
- 每个 family 两个请求：
  - 第一条 warm request
  - 第二条 probe request

### W5-control: backend 正常路径

- 对照组：`VLLM_BATCH_INVARIANT=0`
- backend：`FLASHINFER`
- 目标：建立正常 prefix reuse 参考线

### W5-batch-invariant: backend mismatch

- 实验组：`VLLM_BATCH_INVARIANT=1`
- backend：`FLASH_ATTN`
- 不在本池做 warmup，只做 probe
- 目标：近似测量同一 `prefix family` 路由到不同 backend compatibility
  class 后的 cross-pool false negative

## 运行环境

- 仓库：`/data/projects/KVresearch/vllm`
- Python 环境：`/opt/conda/envs/vllm_longbench/bin/python`
- GPU：`CUDA_VISIBLE_DEVICES=2`
- `PYTHONPATH=/data/projects/KVresearch/vllm`
- 模型默认值：`/data/models_copy/llama/llama-2-7b-chat`

## 固定产物

每个场景必须产出：

- `scenario_metadata.json`
- `manifest.jsonl`
- `request_outputs.jsonl`
- `traces/frontend_input.jsonl`
- `traces/engine_0_lookup.jsonl`
- `traces/engine_0_config.jsonl`

聚合产物：

- `summary.md`

## 核心指标

- `physical_hit_tokens`
- `false_negative_reason`
- `bridge_candidate`
- `bridgeable_class`
- `prompt_tokens`
- `api_contract_class`

派生指标：

- `false_negative_ratio = 1 - physical_hit_tokens / prompt_tokens`
- `bridgeable_ratio = bridgeable_false_negative_tokens / prompt_tokens`

## 通过门槛

这轮预实验只要满足以下条件，就进入下一轮恢复机制实现：

1. `W1` 中 `prompt_logprobs` 相比普通 generation 出现明显更高的
   `false_negative_ratio`
2. `W5-batch-invariant` 相比 `W5-control` 出现明显更高的
   `false_negative_ratio`
3. 两类 miss 在 trace 中都能稳定归因为各自的 mismatch class
4. 两类 miss 都能被解释为 `bridgeable miss`，而不是随机噪声

## kill criteria

出现以下任一情况，则不继续把它当主 thesis 推进：

1. `W1` 与 `W5` 都无法稳定复现 false-negative miss
2. false-negative 比例过低，无法支撑后续系统收益
3. 归因字段不稳定，无法把 miss 明确分为可恢复的 mismatch class
4. 同一 workload family 内结果高度漂移，无法形成可重复结论

## 执行顺序

1. 先跑 `W1` 冒烟，确认 trace 字段齐全
2. 再跑 `W5-control`
3. 再跑 `W5-batch-invariant`
4. 最后运行 `analyze_preexperiment.py` 生成聚合结论

## 后续衔接

如果预实验成立，下一步不是继续加 workload，而是进入两个最小恢复实例：

1. `prompt_logprobs selective replay`
2. `backend mismatch twin-pool fast path`
