# FrontierKV：面向 exact APC 的边界自适应块粒度运行时

## 1. 一句话定位

`FrontierKV` 的核心主张是：

> 当前 `vLLM` 的 prefix cache 被固定 block size 绑死了。大 block 更高效，但更难共享；小 block 更容易命中，但元数据和运行时开销更高。真正需要细粒度的，不是整个 prefix，而是 prefix 的复用边界本身。

## 2. 为什么这条线值得单独拎出来

我们前面的很多 idea 都默认“page / block granularity 是给定的”，但其实这可能正是 exact prefix reuse 的硬约束之一。

从当前系统现状看：

- `vLLM` 里只有 full blocks 才能共享；
- 当所有 token hit cache 时，最后一个 token 仍然需要重新计算，甚至可能触发整块重算；
- `TensorRT-LLM` 官方文档也明确把 `tokens_per_block` 当成命中率和效率的 tradeoff。

这说明 block size 不是一个细节，而是**cache reuse 和 kernel efficiency 之间的硬冲突**。

## 3. 核心洞见

不是整个 prefix 都需要细粒度。真正需要细粒度的，是两个地方：

- `reuse frontier`
  - 已缓存前缀和新 token 的交界面
- `evolving tail`
  - 刚刚变成热门、还不稳定的共享段

而已经非常稳定、几乎总是整段复用的 prefix body，其实没必要一直用细粒度 block 表示。

因此 `FrontierKV` 的设计不是“把 block size 全部做成动态”，而是：

- **prefix body 用大块**
- **reuse frontier 用小块**
- **当 frontier 稳定后，再把小块在线并回大块**

## 4. 系统设计

## 4.1 两层 block geometry

一条 prefix 在缓存里被拆成两部分：

- `body blocks`
  - 大块，追求 kernel 友好和低元数据开销
- `frontier blocks`
  - 小块，追求更高的复用粒度和更低的边界重算

这两类 block 仍然保持 exact 语义，只是物理表示不同。

## 4.2 frontier splitting

当一个请求首次把某段 prefix 推到“可能被共享”的边界时，系统不再立即用统一大块固化它，而是：

- 在边界附近先采用更细粒度 block
- 观察其命中模式和稳定性
- 如果后续确认这段边界经常以相近位置被复用，再把它合并为更大的 body block

这让 block size 从一次性静态选择，变成一个在线演化过程。

## 4.3 online coalescing

边界长期稳定后，持续使用小块会浪费元数据、增大调度和遍历成本。

因此 `FrontierKV` 需要一个 `coalescing` 过程：

- 在不改变逻辑 token span 的前提下
- 把若干连续小块并成大块
- 同时保留对历史命中统计的摘要

这样才能把“命中粒度”和“运行时开销”同时顾住。

## 4.4 frontier-aware indexing

当前 APC 的 block hash 和 longest-hit 路径更适合固定块大小。

`FrontierKV` 需要新的索引结构：

- 对 body block，仍然使用较粗粒度索引；
- 对 frontier block，维护更密集的边界索引；
- 命中时优先快速定位 body，再在 frontier 上精细对齐。

这让系统既不需要全局细粒度索引，也不用为所有 tokens 付出小块管理成本。

## 5. 为什么它和已有工作不一样

和 `TierKV / HotPrefix / CachedAttention` 不同：

- 它不先讨论“块放在哪一层”；
- 它先讨论“块应该长什么样”。

和 `HeteroKV` 不同：

- `HeteroKV` 的 block 差异来自模型结构异构性；
- `FrontierKV` 的 block 差异来自 prefix reuse frontier 的动态特征。

和 `ContiguousKV` 不同：

- `ContiguousKV` 更偏 offload / re-prefill 的 I/O granularity；
- `FrontierKV` 更偏在线 APC 命中边界的计算粒度。

## 6. 为什么它能增强 vLLM

如果做成，`vLLM` 在 APC 上会得到三类直接收益：

- 更少的边界重算
- 更好的 shared-prefix hit granularity
- 在不把所有 block 都做小的前提下，减少 coarse blocks 带来的 false miss

这条线最吸引人的一点是：它不是在靠 workload prediction 或 external cache 取胜，而是在**重写 exact APC 的基本几何形态**。

## 7. 在 vLLM 里怎么落地

可能的改动点包括：

- `vllm/v1/core/kv_cache_interface.py`
  - 扩展 block metadata，支持 body/frontier class
- `vllm/v1/core/kv_cache_utils.py`
  - 新的 hash / span / merge 规则
- `vllm/v1/core/kv_cache_manager.py`
  - longest-hit 与 allocate/free 逻辑要感知 frontier
- `vllm/v1/worker/*`
  - attention backend 或 cache view 需要适配 mixed block classes

## 8. 论文价值与风险

### 8.1 价值

- 明显不同于上一轮的多级、路由、QoS、segment、hybrid 方案
- 问题很“底层”，有硬核系统味
- 很容易用当前 `vLLM` 的 APC 设计缺口来做 motivation

### 8.2 风险

- 实现不轻，需要动到 cache interface 和 worker 路径
- 论文必须证明收益来自“frontier-aware geometry”，而不是仅仅因为 block 变小了
- 如果实验只得到中等收益，评审可能会觉得 thesis 不够大

## 9. 我的判断

`FrontierKV` 是三个新方案里最偏 runtime / cache geometry 的一个。它和前一轮方案重合最少之一，也很有工程含量，但论文的“高度”取决于你能不能把它讲成：

- **exact APC 的几何形态本身就是系统瓶颈**

如果这个 thesis 讲稳了，它会是一条很扎实的系统方向。
