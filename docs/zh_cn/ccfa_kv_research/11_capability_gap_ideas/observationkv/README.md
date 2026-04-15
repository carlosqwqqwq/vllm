# ObservationKV：面向混合 API 的观测友好前缀缓存

## 1. 一句话 thesis

`ObservationKV` 的核心主张是：

> prefix cache 不应只服务“纯生成请求”；在 `prompt_logprobs`、`pooling`、`token-level scoring` 这类需要更多观测输出的 API 上，`vLLM` 也应尽可能复用前缀状态，而不是因为“缓存里没有观测信息”就整段重算。

## 2. 为什么它是一个真实缺口

`vLLM` 当前实现里，prefix cache 会被以下条件主动绕过：

- `prompt_logprobs` 请求默认跳过 prefix cache；
- 多种 pooling 任务默认跳过 prefix cache；
- 根因不是前缀不能复用，而是**APC 只缓存了 KV，没有缓存或可恢复这些请求所需的观测结果**。

源码里对应的直接证据：

- `vllm/sampling_params.py`
  - `prompt_logprobs` 默认令 `skip_reading_prefix_cache=True`
- `vllm/pooling_params.py`
  - `token_embed` / `token_classify` 等 pooling 任务默认跳过 prefix cache
- `vllm/v1/core/kv_cache_manager.py`
  - 注释明确写出：prompt logprobs 或 all-pooling 请求会跳过 prefix cache 查询

这说明：

- 现有 APC 语义是“跳过 prefill 计算”
- 但很多真实 API 需要的不是只有 KV，而是还需要某些**可观测结果**

## 3. 最近邻工作与重复风险

截至这轮检索，我**没有找到与 `ObservationKV` 完全等价的公开系统论文**。

但这里有一个关键更新：

- `vLLM` 已经在 `2025-03-08` 合并了 [PR #13949](https://github.com/vllm-project/vllm/pull/13949)，让 `prompt_logprobs` 与开启 APC 的实例“可以共存”；
- 不过它采取的是**保守兼容**：
  - `prompt_logprobs` 请求自身仍然**不能命中 prefix cache**
  - 实现方式是 **near-full recomputation**
  - 这些请求产生的 KV 仍可被其他普通请求复用

这意味着：

- `ObservationKV` 不能再写成“第一次支持 prompt_logprobs + APC”；
- 更合理的命题应当是：
  - **把 prefix cache 从 generation-only optimization 提升成 mixed-API observation substrate**
  - **系统化减少 exact observation 的恢复代价**

最近邻更多是工程信号而不是已定型论文：

- [vLLM RFC: Prompt logprobs + APC compatibility](https://github.com/vllm-project/vllm/issues/13414)
- [vLLM PR: [V1] Prompt logprobs + APC compatibility; prompt logprobs reqs cannot fill APC](https://github.com/vllm-project/vllm/pull/13949)
  - 社区已经明确承认这是一个真实问题。
- [Beyond Speedup – Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)
  - 它说明 KV cache 不只可以用于加速 decode，也可以作为轻量表示被二次利用。

但它们与 `ObservationKV` 仍有明显差别：

- vLLM RFC 是单点兼容性问题，不是系统命题；
- `Beyond Speedup` 研究的是把 KV 当作表示用于推理/采样，不是 serving runtime 里让 prefix cache 与混合 API 共存。

### 当前判断

- **重复风险：中低**
- **工程信号：强**
- **学术表达风险：中**

最大风险是它容易被误解成“把 APC 支持再补几个 API 参数”。

因此，如果要把它做成论文，命题不能写成：

- “支持 prompt_logprobs + APC”

而应写成：

- “把 prefix cache 从 generation-only optimization 提升成 mixed-API serving substrate”

## 4. 我认为可防守的 refined 命题

> 在现代推理服务中，请求不再只有文本生成，还包括 prompt scoring、token-level observation、embedding / classify / pooling 等混合 API。现有 prefix cache 只缓存 KV，导致带观测需求的请求无法享受缓存收益。我们提出一种观测友好前缀缓存机制，在保持 exact semantics 的前提下，把 KV reuse 与 observation extraction 解耦。

## 5. 可能的机制设计

第一版不需要走到“大而全”，可以只做下面三件事：

1. `observer replay`
   - 命中 APC 后，不重跑整段 prefill，只对需要观测结果的 token 做受控 replay。
2. `observation checkpoints`
   - 对高价值位置缓存轻量观测检查点，而不是缓存完整 hidden states。
3. `API-aware cache plan`
   - scheduler / executor 根据请求类型选择：
     - pure reuse
     - reuse + selective replay
     - full recompute

## 6. 为什么它符合你的要求

### 6.1 轻量级

- 第一版主要改：
  - `sampling_params.py`
  - `pooling_params.py`
  - `kv_cache_manager.py`
  - model executor 的 prefill replay 路径
- 不需要改模型训练，不要求新的外部系统。

### 6.2 普适性

- 适用于 generation + scoring + pooling 的混合服务场景；
- 尤其适合统一 OpenAI-compatible API 的 serving 平台。

### 6.3 高吞吐 / 低延迟

- 在长共享前缀 + 多种 API 混合的线上场景里，可以减少整段 prefill 重算；
- 对 TTFT 与整体 GPU 占用都可能有收益。

## 7. 最大风险

最大的风险不是实现，而是**论文包装**。

如果实验只证明：

- prompt_logprobs 更快了

那很容易被审稿人看成 feature patch。

要把它撑成系统论文，必须证明：

- 这是一类普遍存在的 mixed-API serving 矛盾；
- 现有 APC 的 generation-only 语义已经落后于真实 API 需求；
- 统一机制能在多类 API 上成立，而不是只对一个参数成立。

## 8. 当前判断

截至 `2026-04-15`：

- **可行性：高**
- **vLLM 落地性：高**
- **优化效果把握：中高**
- **与公开论文撞车风险：中低**
- **OSDI 潜力：中**

它不是当前最“宏大”的方向，但却是当前最像**真实缺口 + 轻量实现 + 明显收益**的方向之一。

## 9. 关键资料

- [vLLM RFC: Prompt logprobs + APC compatibility](https://github.com/vllm-project/vllm/issues/13414)
- [vLLM PR: [V1] Prompt logprobs + APC compatibility; prompt logprobs reqs cannot fill APC](https://github.com/vllm-project/vllm/pull/13949)
- [Beyond Speedup – Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)
- [ObservationKV 深挖评估](./deep_dive.md)
