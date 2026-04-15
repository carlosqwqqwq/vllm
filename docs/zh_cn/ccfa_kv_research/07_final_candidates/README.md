# 第三轮收敛后的最终候选

## 1. 为什么还要再来一轮

前两轮调研已经把一个事实讲得很清楚：

- 多级 KV hierarchy、external KV、prefix-aware routing、热前缀保留、非前缀分段复用、hybrid KV、边界自适应粒度

这些方向都不是坏方向，但都已经能找到很强的近邻工作。继续沿着这些主线推进，论文很容易落成“在既有路线上的又一个优化”。

所以这第三轮的目标只有一个：

> 不再问“还能不能做 KV 优化”，而是问“还有哪些关键系统约束，被今天的工作默认接受了，但还没有被当成论文主张写出来”。

## 2. 这一轮怎么筛选

我把筛选标准进一步收紧成五条：

1. **必须仍然是 KV cache 优化**
   - 不能完全跑成通用 routing 或纯模型改造。
2. **必须能直接作用到 vLLM**
   - 最好能映射到现有 `scheduler / kv_cache_manager / worker / serve` 路径。
3. **必须偏轻量级**
   - 不依赖大规模外部系统或复杂分布式基础设施。
4. **必须同时服务高吞吐和低延迟**
   - 不能只优化命中率，必须能落到 `TTFT / P99 / throughput`。
5. **必须尽量避开现有强工作**
   - 如果明显已经有人系统性做过，就不再保留。

## 3. 最后保留下来的三个方向

- [ZeroLagKV](zerolagkv/README.md)
  - 把“KV state 何时可发布”变成系统主问题，重点解决异步调度下的 cache publication lag。
- [FastPathKV](fastpathkv/README.md)
  - 不再只缓存 KV 本身，而是把 APC hit 路径上的执行元数据、block-table 视图和 dispatch 描述一并快路径化。
- [PublicStateKV](publicstatekv/README.md)
  - 把可复用的公开会话状态和不应共享的私有执行状态分开，服务 reasoning / tool-use / agent workload。

## 4. 这三条线为什么比前面更干净

- `ZeroLagKV`
  - 不是多级 KV，不是粒度优化，不是路由。
  - 它盯住的是 `cache publication timing`。
- `FastPathKV`
  - 不是“多缓存一点”，而是“命中了之后别再重复准备执行状态”。
  - 它盯住的是 `post-hit overhead`。
- `PublicStateKV`
  - 不是普通 prefix policy，也不是 session replay。
  - 它盯住的是 `shareable state boundary`。

## 5. 当前总判断

如果按“最符合你的约束”排序，我目前的判断是：

1. `FastPathKV`
2. `ZeroLagKV`
3. `PublicStateKV`

这个排序和上一轮不一样，原因是我把“轻量级、普适性、能直接增强 vLLM”放得更靠前了。

- `FastPathKV` 最轻、最通用、最贴近当前 vLLM worker 路径。
- `ZeroLagKV` 非常像一篇干净的系统 thesis，但需要更强的 scheduler 论证。
- `PublicStateKV` 新意最大，但 exact 边界和系统改造都更重。

## 6. 一个必须坦率说明的边界

我这轮检索后，能给出的最严格说法是：

- **我没有找到与这三个方向完全同构的现成论文。**
- 但我**不能负责任地保证“绝对没人做过”**。

更准确的表达应该是：

> 在当前检索到的顶会论文、预印本、工业系统文档和 vLLM 本地实现约束下，这三条线是我目前认为重合风险最低、同时又最符合你需求的三个候选。
