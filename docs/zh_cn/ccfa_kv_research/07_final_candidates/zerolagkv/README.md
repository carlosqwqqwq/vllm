# ZeroLagKV：面向异步调度的零滞后 KV 发布

## 1. 一句话定位

`ZeroLagKV` 的核心主张是：

> 现有 APC 系统大多把关注点放在“已发布 KV 怎么复用”，但在异步 serving 中，一个更隐蔽的瓶颈是“本来已经算出来、理论上可复用的 KV，还没有及时变成可见的共享状态”。

## 2. 这条线为什么会浮出来

这次调研里，一个最关键的证据来自 vLLM 自己的技术报告。

技术报告明确写了：

- 在异步调度下，如果请求 `X` 在第 `N` 步刚刚生成了 `D`
- 而共享这个前缀的新请求 `Y` 在第 `N+1` 步已经被调度
- 因为调度发生在收到 `D` 之前，`Y` 只能命中 `[A, B, C]`，而命不中刚刚生成的 `D`

也就是说：

> 异步调度会把可复用状态的“发布时刻”向后推迟至少一个 step。

这不是 cache miss，不是路由错，也不是 block size 不合适，而是**publication lag**。

## 3. 最近邻工作有哪些

最接近但又不完全一样的工作有三类。

### 3.1 provider / industrial reuse

代表：

- [TensorRT-LLM KV Cache Reuse](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)
- [TensorRT-LLM Early Reuse blog](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)

它们已经意识到：

- reuse 不能只等整个请求完全结束；
- 更早地复用 prefix state 对 TTFT 很重要。

但目前公开内容更像工业能力说明，并没有把“异步调度下的发布滞后”系统化成一个 thesis。

### 3.2 shared-prefix execution

代表：

- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)
- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)

这类工作优化的是：

- shared prefix 的 attention 执行
- prefix-aware kernel 路径

而不是：

- **同一个 serving runtime 里，KV state 何时变成其他请求可见**

### 3.3 stateful/session-aware serving

代表：

- [HCache](https://arxiv.org/abs/2410.05004)
- 一系列 session-aware / state restoration 路线

它们更关心：

- 会话状态怎么跨轮次保存
- 如何减少 replay

而不是：

- **正在运行中的 prefix state 为什么来不及被后来者看到**

## 4. 为什么我认为它的重合风险较低

我这轮没有找到一篇直接围绕下面这个命题展开的论文：

> 在异步 continuous batching 的 LLM serving 中，cache publication lag 是一个独立且可优化的系统瓶颈。

这就是 `ZeroLagKV` 目前最有价值的地方。

它和 `Early Reuse` 很接近，但还不完全一样：

- `Early Reuse` 更像“能更早复用”
- `ZeroLagKV` 更强调“异步调度和 state publication 之间存在结构性时差”

如果后续继续调研仍然找不到直接同构工作，这条线有希望成为一个干净 thesis。

## 5. 在 vLLM 里为什么可做

本地代码给了三个很强的落地点。

### 5.1 full blocks 在请求活着的时候就会被 cache

`vllm/v1/core/kv_cache_manager.py` 中，`allocate_slots()` 在请求尚未结束时就会调用 `coordinator.cache_blocks(...)`。

这说明：

- vLLM 不是“只在 free 时发布”
- 已经具备“活跃请求中间态可进入 hash table”的基础

### 5.2 cached blocks 可以在 `ref_cnt > 0` 时被 touch

`block_pool.touch()` 明确允许命中的 block 即使仍被别人引用，也继续提高 `ref_cnt`。

这说明：

- 共享中的 block 在数据结构层面不是禁区
- 真正缺的是更强的 publication / visibility protocol

### 5.3 技术报告已经承认 async scheduling 存在一 step 的注册滞后

这直接把论文 motivation 从“猜测”变成了“官方承认的限制”。

## 6. ZeroLagKV 的核心机制

### 6.1 publish-ahead protocol

不是等一个 step 完成后再被动注册，而是在：

- full block 形成
- block hash 稳定
- block 被标记为 committable

这三个条件满足时，立即把它放入可见状态。

### 6.2 follower-aware scheduling

等待队列中的请求，不再只看：

- 当前 cache hit 了多少

还要看：

- 哪些 producer request 即将在本 step 结束时发布新 block
- follower 是否值得短暂等待，换取更长的 prefix hit

### 6.3 visibility tiers

把 block 状态分成三档：

- `private`
  - 还不可见
- `published`
  - 可复用，但仍由 producer 持有
- `stable`
  - 已进入正常 APC 生命周期

这样能把“活跃请求中的共享状态”正式纳入 runtime 语义，而不是隐式行为。

## 7. 预期收益

最可能受益的场景：

- 多个请求共享长 system prompt
- 多轮 chat / agent 同一会话短时间连续到达
- bursty prefix-heavy workload

预期收益主要在：

- `TTFT`
- prefix hit 长度
- 避免重复 prefill 带来的 throughput 改善

## 8. 风险

这条线最大的风险有两个。

### 8.1 需要证明“publication lag”不是边角料

如果这个问题只在极少数 workload 上出现，论文会显得太窄。

### 8.2 需要严格 correctness protocol

必须精确定义：

- 什么时候 block 算“可发布”
- draft/spec decode token 能不能发布
- producer 更新 block 时 follower 看见的是什么版本

## 9. 我当前的判断

`ZeroLagKV` 是我这轮保留下来的三个方向里，**最像一个完整系统 thesis** 的方向之一。

它目前最吸引人的点是：

- 和经典多级/路由/粒度优化明显不同
- 和 vLLM 已知限制直接对上
- 同时满足低延迟与高吞吐目标

如果后续继续排重仍然没有撞上直接同构工作，它很有希望成为真正可冲 `OSDI/CCF-A` 的主线。
