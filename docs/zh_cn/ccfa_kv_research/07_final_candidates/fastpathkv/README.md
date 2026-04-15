# FastPathKV：把 APC 命中路径也变成可缓存对象

## 1. 一句话定位

`FastPathKV` 的核心主张是：

> 现有 KV cache 系统大多把“命中 KV 之后就省掉 prefill”当成结束，但在高命中场景下，系统仍然需要为 attention metadata、block-table 视图、dispatch 选择和输入准备付出重复开销。我们应该缓存的不只是 KV state，还应该缓存 APC hit path 的执行描述。

## 2. 这条线为什么会浮出来

这轮调研里，vLLM 自己的技术报告给了非常强的信号：

- 输入准备可能消耗毫秒级 CPU 开销
- 主要来源包括：
  - batched request bookkeeping
  - 大量 PyTorch dispatch
  - CPU-GPU synchronization

报告随后介绍了它已经做的优化：

- GPU-side input preparation
- UVA
- persistent block tables
- incremental updates

但这也反过来说明：

> 命中 KV 之后，输入准备和执行描述构建，依然是值得单独优化的一条主路径。

## 3. 最近邻工作有哪些

### 3.1 KV cache / prefix reuse 论文

代表：

- [vLLM / PagedAttention](https://arxiv.org/abs/2309.06180)
- [Prompt Cache](https://arxiv.org/abs/2311.04934)
- [ChunkAttention](https://aclanthology.org/2024.acl-long.623/)

这些工作优化的是：

- KV 怎么存
- 怎么复用 prefix
- 怎么让 attention 更快

但都没有把“APC 命中之后的执行元数据重复准备”上升成主 thesis。

### 3.2 prefix-aware kernel 路线

代表：

- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)

这类工作把 shared-prefix attention 下探到 kernel 层，已经很强，但它关注的是：

- kernel execution geometry

而不是：

- **serving runtime 在命中后仍然反复组装哪些执行状态**

### 3.3 vLLM 自己已有的局部缓存

本地代码里已经有一个重要信号：

- `gpu_model_runner.py` 已经会在同一次 `_build_attention_metadata()` 中，跨 hybrid KV groups 复用 attention metadata build 结果

这说明：

- 开发者已经意识到 metadata build 是值得缓存的
- 只是当前缓存范围还很局部，尚未上升到“跨请求、跨 step 的 APC fast path”

## 4. 为什么我认为它的重合风险较低

我这轮没有找到直接围绕下面命题展开的论文：

> prefix cache 命中后，执行描述本身应该像 KV 一样进入缓存和快路径。

这条线的独特之处在于，它不是去重新定义：

- KV 存储层次
- prefix 匹配策略
- attention kernel

而是盯住：

- **hit 之后系统仍然在为哪些重复工作付费**

这让它和大多数已知 KV 论文自然错开。

## 5. 在 vLLM 里为什么很自然

### 5.1 技术报告已经承认 input preparation 是关键开销

这给了非常扎实的 motivation。

### 5.2 worker 里已经有 persistent structures

例如：

- persistent GPU block table buffer
- incremental updates via UVA
- CUDAGraph dispatcher

这些都意味着：

- vLLM 不是每步完全从零准备
- 但它已经拥有适合做“更激进 fast path caching”的基础设施

### 5.3 已有局部 metadata caching

`gpu_model_runner.py` 里跨 hybrid groups 的 `cached_attn_metadata` 说明：

- cache execution descriptors 这件事不是异想天开
- 只是还没有被推广成独立机制

## 6. FastPathKV 的核心机制

### 6.1 cache execution descriptors with KV

每个热 prefix 除了缓存 KV block，还缓存：

- attention metadata template
- block-table view template
- dispatch class / cudagraph bucket
- 可复用的 logits / sampling side metadata skeleton

### 6.2 prefix-shaped fast path

命中新 prefix 时，不再重新走完整的 metadata build，而是：

- 读取 descriptor cache
- 只填动态 delta
- 直接进入 graph-ready 或 near-graph-ready 路径

### 6.3 descriptor validity contract

descriptor 不是无条件可复用，必须绑定：

- model / kv_cache_spec
- batch shape class
- block-table layout class
- sampling mode class

这样才能避免把 descriptor cache 做成 correctness 雷区。

## 7. 预期收益

它最可能改善的是：

- `APC hit` 场景下的 `TTFT`
- CPU 利用率
- 小请求 / 高频请求场景下的尾延迟

这条线的好处在于：

- 即使 KV 命中率已经很高，也还有优化空间
- 不需要引入分布式系统，也不依赖特殊 workload

## 8. 风险

### 8.1 需要证明命中后开销真的足够大

如果 profiling 发现元数据准备只是小头，论文价值会下降。

### 8.2 需要防止 descriptor key 爆炸

如果 descriptor cache 的 key 维度过多，缓存命中率会下降，系统反而变复杂。

## 9. 我当前的判断

在这轮保留下来的 3 个方向里，`FastPathKV` 是我目前认为：

- **最轻量级**
- **最普适**
- **最贴近 vLLM 当前主线**
- **最容易先做出有效原型**

的一条线。

它未必像 `ZeroLagKV` 那样一眼看上去就是“大论文 thesis”，但如果 profiling 能证明确实存在稳定的 post-hit overhead，这条线会非常扎实，而且很可能是三条里最稳的一条。
