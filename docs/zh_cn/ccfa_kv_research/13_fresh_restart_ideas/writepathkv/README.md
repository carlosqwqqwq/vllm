# WritePathKV：把 KV 的写路径与发布路径变成一等优化对象

更新日期：2026-04-15

> 复审说明：下面这份 `README` 保留的是第一版构思。
> 截至 `2026-04-15` 的严格重复性与可行性判断，请以 [`overlap_feasibility_assessment.md`](./overlap_feasibility_assessment.md) 为准。

## 1. 一句话 thesis

`WritePathKV` 的核心主张是：

> 在 `vLLM` 逐步演进为“本地 APC + CPU offload + KV connector + P/D disaggregation”统一运行时时，
> 系统真正越来越贵的不只是读路径，而是**新生成状态的写路径与发布路径**；
> 因此问题不该只问“如何命中 cache”，还应问“哪些 KV 值得发布、何时发布、发布到哪一层”。

## 2. 为什么这是一个新问题

过去大多数 KV cache 论文更关心：

- 如何命中已有 cache
- 如何迁移已有 cache
- 如何在不同 worker 间复用已有 cache

但在 `vLLM` 当前这类多层 KV 栈里，还存在另一个越来越重要的问题：

- 新生成状态默认会被 eager 地写入本地 prefix cache；
- 也可能进入 CPU offload；
- 也可能通过 connector/export 路径进入外部层；
- 在 P/D disaggregation 下，还可能经历额外 transfer。

这意味着系统今天越来越容易遇到：

1. **write amplification**
   - 生成一个本来只会本地消费一次的状态，却被写进多层。
2. **cache pollution**
   - 导出了未来几乎不会被复用的 block，占掉共享层容量。
3. **publication mismatch**
   - 有些状态发布过早，污染全局层；
   - 有些状态发布过晚，又错过复用机会。

这和“read path 命中率不够高”不是同一个问题。

## 3. 为什么它和最近论文边界更清楚

这条线最值得继续的原因，是它和最近两年的强工作边界相对更干净。

### 3.1 与 `LMCache` 的边界

- `LMCache`
  - 更关注外部 KV layer、connector、KV 传输和 orchestration。
- `WritePathKV`
  - 研究的是：
  - **一个新状态到底应不应该被发布/写出**
  - 以及 publication/admission policy 本身。

也就是说：

- `LMCache` 更像“怎么搬”
- `WritePathKV` 更像“值不值得搬”

### 3.2 与 `BanaServe` 的边界

- `BanaServe`
  - 更关注 disaggregated serving 下的 global store、迁移与负载平衡。
- `WritePathKV`
  - 不研究全局资源调度；
  - 研究的是跨 local cache / offload / connector 的 **写路径价值判断**。

### 3.3 与 `HCache` 的边界

- `HCache`
  - 更关注状态恢复。
- `WritePathKV`
  - 更关注状态发布与导出。

### 3.4 当前结论

截至 `2026-04-15`，我**没有找到一篇公开系统论文把“KV write path / publication path admission”作为主命题系统化写清楚**。

不能说“确保没人做过”，但可以说：

- **这条线比 read-path exact-prefix reuse 更不拥挤。**

## 4. 为什么它能在 vLLM 中落地

这条线不是纸上命题，`vLLM` 当前已经有明显实现抓手。

### 4.1 本地 cache 写路径

`vllm/v1/core/kv_cache_manager.py`

这里已经显式处理：

- 本地 prefix-cached tokens
- connector-cached external tokens
- `delay_cache_blocks`

说明“状态何时进入 cache”本身已经是 runtime 决策点。

### 4.2 CPU offload 写路径

`vllm/v1/simple_kv_offload/manager.py`

这里已经显式存在：

- `StoreRequestState`
- `prepare_store`
- `complete_store`

说明 offload 不是单纯被动结果，而是有显式 store 流程。

### 4.3 connector / export 写路径

`vllm/v1/worker/kv_connector_model_runner_mixin.py`

这里已经存在：

- `wait_for_save`
- `bind_connector_metadata`
- connector stats / events

说明外部层导出本身就是一等路径。

### 4.4 为什么这比读路径更适合成为新主线

因为 `vLLM` 现在已经不是单层本地 APC。

当系统拥有：

- local APC
- CPU offload
- external connector
- P/D disaggregation

之后，“是不是写出去”本身就成了跨层资源问题。

## 5. 可能的核心机制

第一版不需要做复杂学习器，可以很克制。

### 5.1 publication class

给新生成状态分几类：

- `LOCAL_ONLY`
  - 只保留在本地 prefix cache
- `LOCAL_AND_OFFLOAD`
  - 本地保留并允许 CPU store
- `EXPORTABLE`
  - 可发布到 connector / 外部层
- `DEFERRED`
  - 先不发布，等待复用信号
- `DROP_AFTER_USE`
  - 不值得进入共享层

### 5.2 publication signal

第一版可用的信号包括：

- 前缀是否重复出现
- 请求是否属于共享模板类 workload
- residual 长度与总前缀长度
- 是否多次被本地命中
- 是否处于高压 cache 环境
- export/offload 的写成本

### 5.3 目标函数

目标不是最大化“写进去的块数”，而是最大化：

- future reuse value - write cost - pollution cost

这就是这条线的核心学术抽象。

## 6. 为什么它符合你们的要求

### 6.1 轻量级

第一版主要改：

- `kv_cache_manager`
- `simple_kv_offload`
- connector metadata / scheduler

不需要改模型，不需要改 attention kernel。

### 6.2 普适性

只要系统里存在：

- 本地 cache
- offload
- connector/export

这条线就成立，不依赖特定模型架构。

### 6.3 高吞吐

减少无效导出和写放大，本质上是在回收：

- PCIe / 网络带宽
- CPU 存储带宽
- 共享 cache 容量

这些都会回到吞吐上。

### 6.4 低延迟

当 publication policy 更合理时：

- 真正高价值状态更容易留下来；
- 低价值状态不会挤掉高价值状态；
- 也不会因为过多写出而拖慢请求路径。

## 7. 最关键的实验应该是什么

如果这条线往下做，最关键的不是“命中率提升多少”，而是：

1. 当前系统写出了多少最终没有复用的 block
2. 导出/offload 的写流量里有多少是低价值流量
3. 不同 publication policy 下：
   - reuse value
   - write amplification
   - pollution
   - TTFT / throughput

也就是说，这条线最强的证据应该来自：

- **problem characterization of the write path**

而不是又一张 prefix hit rate 图。

## 8. 最大风险

最大的风险是：

- 如果最后只做成一个简单 admission heuristic，
- reviewer 会说这只是 cache policy patch。

所以它必须坚持两个边界：

1. 研究对象是 **publication/write path**
2. 贡献不是“更好 heuristic”，而是：
   - 明确定义 publication class
   - 明确写路径的价值模型
   - 在多层 KV 栈里统一处理

## 9. 当前判断

截至 `2026-04-15`：

- **创新性：高**
- **与最新论文重合风险：中低**
- **vLLM 落地性：高**
- **轻量原型可行性：高**
- **当前三条里的主推荐程度：最高**

## 10. 参考资料

- LMCache: <https://arxiv.org/abs/2510.09665>
- BanaServe: <https://arxiv.org/abs/2510.13223>
- HCache: <https://chenyoumin1993.github.io/papers/eurosys25-hcache.pdf>
- vLLM Prefix Caching Design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
