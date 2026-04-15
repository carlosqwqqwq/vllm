# 调研过程与结果沉淀

## 1. 本轮问题

用户目标是：

- 方向：KV cache 优化。
- 平台：vLLM。
- 论文目标：系统类 CCF-A，最好接近 OSDI。
- 约束：轻量级、普适性、高吞吐、低延迟。
- 额外要求：避免与已有工作重复。

本轮不再默认前面提出的 idea 都可用，而是先按 “撞车风险” 过滤，再重新定位。

## 2. 检索关键词

主要检索方向：

- `vLLM prefix cache partial block`
- `vLLM CUDA Graph prefix cache BatchDescriptor`
- `LLM serving CUDA Graph dynamic batching prefix cache`
- `SGLang RadixAttention cache-aware scheduling`
- `Jenga hybrid KV cache vLLM`
- `Strata hierarchical KV caching LLM serving`
- `PAT prefix-aware attention`
- `LMCache vLLM KV cache offloading`
- `SpeCache transactional KV caching PagedAttention`
- `prompt logprobs prefix caching vLLM`
- `LLM serving CPU overhead scheduler prefix cache`

## 3. 已确认的关键事实

### 3.1 vLLM APC 仍是 full-block 强绑定

本地源码证据：

- `vllm/v1/core/kv_cache_manager.py`：`get_computed_blocks()` 要求 computed blocks must be full。
- `vllm/v1/core/kv_cache_utils.py`：request block hasher 在不足 full block 时 early stop，并注释 “We only hash full blocks”。
- `vllm/v1/core/block_pool.py`：cached block 被定义为 full block，并且当前为了 append-only block table 不做去重。

影响：

- 支持 ElasticPageKV 的问题存在。
- 但 vLLM APC RFC 已经讨论过 partial block 相关方向，因此不能直接主打 partial block。

### 3.2 vLLM 的 CUDA Graph 与执行形状强相关

本地源码证据：

- `vllm/v1/cudagraph_dispatcher.py`：运行时根据输入生成 cudagraph mode 和 `BatchDescriptor`。
- `vllm/v1/core/sched/scheduler.py`：调度器统一处理 chunked prefill、prefix caching、speculative decoding。
- `vllm/v1/worker/gpu_model_runner.py`：执行前同时处理 cascade attention prefix lens 与 cudagraph padding/dispatch。
- `vllm/v1/attention/backends/flash_attn.py`：cascade attention 使用固定启发式阈值，仍有调优空间。

影响：

- 支持 ShapeKV 的问题存在。
- 目前没有发现完全等价的 “APC hit -> graph/cascade shape stabilization” 论文。

### 3.3 控制面优化有空间，但论文边界危险

本地源码证据：

- `vllm/utils/hashing.py`：prefix cache hash 支持 `sha256`、`sha256_cbor`、`xxhash`、`xxhash_cbor`。
- scheduler 查询 local/external cache 后才决定 residual prefill。

影响：

- ControlPlaneKV 可以做。
- 但 vLLM V1 与 SGLang 已经强调低 CPU overhead 和 cache-aware scheduling，因此单独作为论文主线风险高。

## 4. 被排除或降级的方向

| 方向 | 处理结果 | 原因 |
| --- | --- | --- |
| SegmentKV / RAG 分段复用 | 排除 | SGLang RadixAttention、CacheBlend、EPIC、chunk-level KV reuse 太近 |
| TierKV / 多级 KV | 排除 | LMCache、Mooncake、MemServe、Strata、PCR、CALVO 太近 |
| QoSKV / 低延迟 prefix cache | 排除 | 容易退化为 scheduler/QoS，边界不干净 |
| speculative KV | 排除 | SpeCache 已经明确做 transactional KV caching under PagedAttention |
| prompt logprobs compatible APC | 降级为证据 | vLLM 明确有痛点，但 SPECS 已触及相关修改 |
| ControlPlaneKV standalone | 降级为组件 | 与 vLLM V1、SGLang 低 CPU overhead 目标高度重合 |

## 5. 最终保留方向

### 5.1 ShapeKV

保留原因：

- 问题位于 KV cache 与 GPU runtime 的交界处。
- 最近邻工作没有完全覆盖 CUDA Graph / padding / cascade attention 与 APC hit 的联合优化。
- 改动贴合 vLLM 现有 scheduler 和 model runner。
- 轻量、普适、高吞吐、低延迟四个约束都能对上。

最大待证据：

- baseline vLLM 在 APC high-hit workload 下是否真的出现 graph fallback、padding waste 或 cascade miss。

### 5.2 ElasticPageKV

保留原因：

- vLLM full-block APC 和 block-size alignment 是真实机制。
- 逻辑复用粒度与物理执行粒度解耦是系统论文常见且可讲清的抽象。

最大风险：

- 与 vLLM partial block RFC、Jenga、Strata 的叙事靠得比较近。

### 5.3 ControlPlaneKV

保留原因：

- 作为 ShapeKV 的 cache readiness fast path 很有价值。
- vLLM 中 hash、lookup、scheduler、connector 之间仍然是可优化控制面。

最大风险：

- 单独主投不够独特。

## 6. 后续最小任务清单

1. 写一个 vLLM profiling patch，记录 APC hit 后的 residual length、graph replay/fallback、padding token、cascade enable reason。
2. 构造 shared-prefix benchmark，验证 ShapeKV 的核心问题是否真实显著。
3. 做 block size sweep，判断 ElasticPageKV 是否有独立必要。
4. 做 CPU profile，判断 ControlPlaneKV 是否只是工程优化还是论文级瓶颈。
5. 如果 Step 1 支持 ShapeKV，则优先实现 graph-stable chunking 的最小原型。

## 7. 资料索引

- vLLM prefix caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM Hybrid KV Cache Manager: <https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/>
- vLLM APC RFC: <https://github.com/vllm-project/vllm/issues/2614>
- vLLM prefix-aware scheduling issue: <https://github.com/vllm-project/vllm/issues/7883>
- PagedAttention / vLLM: <https://arxiv.org/abs/2309.06180>
- SGLang / RadixAttention: <https://arxiv.org/abs/2312.07104>
- SGLang v0.4 blog: <https://lmsys.org/blog/2024-12-04-sglang-v0-4/>
- Jenga: <https://arxiv.org/abs/2503.18292>
- Strata: <https://arxiv.org/abs/2508.18572>
- PAT: <https://arxiv.org/abs/2511.22333>
- SpeCache: <https://proceedings.mlr.press/v267/jie25a.html>
- LMCache: <https://arxiv.org/abs/2510.09665>
- PCR: <https://arxiv.org/abs/2603.23049>
- CALVO: <https://arxiv.org/abs/2603.21257>
