# 第七轮调研日志：从能力缺口反推论文候选

## 1. 这一轮的问题

前面几轮不断撞上已有工作，说明继续从“看起来像系统论文的大词”发散已经不高效了。

所以这一轮改成：

> 先找 `vLLM` 里今天哪些 prefix cache 能力面被真实关闭或保守跳过，再从这些缺口里反推更干净的研究方向。

## 2. 本轮重点核对的源码信号

### 2.1 观测型 API 会主动跳过 prefix cache

- `vllm/sampling_params.py`
  - `prompt_logprobs` 默认令 `skip_reading_prefix_cache=True`
- `vllm/pooling_params.py`
  - 多类 pooling task 默认跳过 prefix cache
- `vllm/v1/core/kv_cache_manager.py`
  - 注释直接写明 prompt logprobs / all pooling 请求会跳过 APC 查询

### 2.2 batch invariance / fast backend 与 APC 存在冲突

- `vllm/model_executor/layers/attention/attention.py`
  - `FLASHINFER` / `TRITON_MLA` 与 `VLLM_BATCH_INVARIANT` 同时打开时会关闭 prefix cache
- 多处 backend / MoE / quantization 路径都额外处理 batch invariance

### 2.3 APC 启停高度依赖静态黑名单

- `vllm/config/model.py`
  - pooling + bidirectional attention 不支持 prefix cache
  - hybrid / attention_free / encoder_decoder 不支持 prefix cache
- 一些模型文件中还直接拒绝 `'all' prefix caching`

## 3. 本轮看过的关键外部资料

- [vLLM RFC: Prompt logprobs + APC compatibility](https://github.com/vllm-project/vllm/issues/13414)
- [Beyond Speedup – Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)
- [ShadowServe](https://arxiv.org/abs/2509.16857)
- [Efficient Multi-Adapter LLM Serving via Cross-Model KV-Cache Reuse with Activated LoRA](https://arxiv.org/abs/2512.17910)

## 4. 本轮明确删除的候选

- routing / affinity / replica warming
  - 附近工作已太多，且容易与分布式 prefix caching 论文重合
- hybrid state / general state unification
  - 太容易回到 `PolyStateKV` / `TxnStateKV` 那类高抽象方向
- adapter reuse
  - 已被最新 aLoRA serving 工作显著占位
- distributed fetching / offload
  - 已有 `ShadowServe` 等强近邻

## 5. 最终留下的三个方向

- `ObservationKV`
- `InvariantKV`
- `EligibilityKV`

它们分别对应：

- data/observation gap
- execution/backend gap
- enablement/protocol gap

## 6. 当前策略

短期内不再继续大规模发散。

如果下一步要做原型，优先顺序是：

1. `ObservationKV`
2. `InvariantKV`
3. `EligibilityKV`
