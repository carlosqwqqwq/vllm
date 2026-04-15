# 最终候选对比与推荐

## 1. 总结论

如果把你的目标重新写成一句话：

> 在 `vLLM` 上做一个轻量级、普适性强、能同时提升吞吐和时延、且尽量不和已有工作重合的 `KV cache` 系统论文方向。

那么这轮调研后，我最终推荐保留的 3 个方向是：

1. `FastPathKV`
2. `ZeroLagKV`
3. `PublicStateKV`

## 2. 横向对比

| 方向 | 核心问题 | 最近邻工作 | 重合风险 | vLLM 落地性 | 轻量级 | 普适性 | 吞吐/时延收益 | OSDI 潜力 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| FastPathKV | APC 命中后仍有大量输入准备与 metadata 开销 | vLLM 自己的 input preparation 优化、PAT/ChunkAttention 的 kernel 优化 | 低 | 高 | 高 | 高 | 中高 | 中高 |
| ZeroLagKV | 异步调度让可复用 KV 的发布滞后于请求到达 | TensorRT-LLM Early/Partial Reuse、vLLM async scheduling limitation | 低到中 | 中高 | 中高 | 高 | 高 | 高 |
| PublicStateKV | 公共会话状态与私有执行状态混在一起，破坏 agent/reasoning 的 cacheability | Prompt Cache、Don't Break the Cache、provider prompt caching docs | 中 | 中 | 中 | 中高 | 中高 | 高 |

## 3. 为什么最终是这个排序

### 3.1 FastPathKV 放第一

因为它最符合你的四个硬约束：

- 轻量级
- 普适性
- 高吞吐
- 低延迟

并且它直接贴在 `vLLM` 当前主线的 worker/input preparation 路径上，不需要引入外部系统、也不依赖特定 workload。

它和现有工作的边界也更干净：

- 现有工作主要缓存 **KV state**
- 它要缓存的是 **命中后仍会反复准备的执行描述**

### 3.2 ZeroLagKV 放第二

它的 thesis 非常像系统论文：

- 现有系统关注“state 被发布之后如何复用”
- 它关注“为什么最该复用的 state 没有及时变成可复用”

这条线对 `TTFT` 和 prefix-heavy 并发 workload 很有吸引力，也和 `vLLM` 的异步调度文档限制直接对上。

之所以没有排第一，不是因为方向不强，而是因为：

- 调度论证需要更扎实
- 实现复杂度高于 `FastPathKV`

### 3.3 PublicStateKV 放第三

它在概念上其实最新，也最像未来 agent/reasoning 工作负载的方向。

但它的风险也最大：

- exact 边界更难讲清
- 系统改造范围会从引擎层扩展到会话/服务层
- 很容易被误解成 prompt engineering 或 session replay

所以我保留它，但不把它放在最稳的第一位。

## 4. 如果只能选一条

### 4.1 最稳选择

选 `FastPathKV`。

这是我现在认为：

- 最容易讲清楚
- 最容易做出原型
- 最符合“轻量级 + 普适性”
- 又最不容易直接撞上现成论文

的一条线。

### 4.2 最像大论文的选择

选 `ZeroLagKV`。

如果后续检索继续支持“publication lag 基本没被系统性写过”，它有机会成为最像 `OSDI` thesis 的方向。

### 4.3 最前沿但风险最高的选择

选 `PublicStateKV`。

如果你判断未来 agent/reasoning workload 是核心主战场，这条线可能最有前瞻性，但实现与论证成本明显更高。
