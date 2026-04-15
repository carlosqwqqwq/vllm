# TierKV：同领域重合度与可行性评估

## 1. 这份评估回答什么

这份文档集中回答三个问题：

- 业界是否已经有与 `TierKV` 高度相关的论文、系统或工业方案
- 我们当前版本的 `TierKV` 是否已经与这些工作发生明显重合
- 即便不完全重复，`TierKV` 在当前 `vLLM` 上是否真的可做、是否真的足够支撑系统类顶会论文

先给结论：

- **有，而且这个方向已经明显拥挤。**
- **如果 `TierKV` 仍然表述为“GPU / CPU / Remote 的多级精确 KV cache + 预取 + 放置”，重合度已经不低。**
- **但和 `SegmentKV` 不同，`TierKV` 不是被直接否掉，而是需要升级 thesis。**

我现在的判断是：

- `TierKV` 仍然是最值得继续打磨的主线之一
- 但它的默认版本已经不足以作为“新论文主张”
- 真正还能站住脚的空间，在于把它从“分层 cache 功能”提升成 **vLLM 的 exact KV virtual memory runtime**

## 2. 同领域已经有哪些工作

## 2.1 LMCache：企业级 KV cache layer

`LMCache` 是这条线上最直接、最不能回避的工作。

它已经明确提出：

- 把 KV cache 从 GPU 中抽出来
- 支持跨 query、跨 engine 的共享
- 同时支持 offloading 与 prefill-decode disaggregation
- 提供跨 GPU / CPU / storage / network 的控制接口

它的官方架构文档还直接把系统写成：

- GPU tier
- CPU DRAM tier
- local storage tier
- remote storage tier

并明确说明：

- overflow KV 可以从 GPU 下沉到 CPU
- CPU 可以异步写回磁盘或远端
- 热数据可以预取回 CPU
- 需要复用时再迁回 GPU

也就是说，**“多级 KV hierarchy” 这件事本身已经被 LMCache 清楚讲过了。**

参考：

- https://arxiv.org/abs/2510.09665
- https://docs.lmcache.ai/developer_guide/architecture.html

## 2.2 HiCache：SGLang 的三级层次缓存

`HiCache` 更进一步，因为它不是论文概念，而是已经写进主流 serving engine 的正式设计文档。

它明确把：

- GPU memory 作为 `L1`
- host memory 作为 `L2`
- distributed storage 作为 `L3`

而且还有：

- 元数据树 `HiRadixTree`
- local match
- prefetch from L3
- write-back policies
- CPU-to-GPU transfer overlap
- GPU-assisted I/O
- 与 PD disaggregation 集成
- 统一的 L3 backend 接口

这说明如果我们把 `TierKV` 表达成：

> 类似 CPU L1/L2/L3 的多级 KV cache 系统

那基本已经撞在 `HiCache` 正面了。

参考：

- https://docs.sglang.io/advanced_features/hicache_design.html

## 2.3 Strata：分层 context caching + cache-aware scheduling

`Strata` 是当前对 `TierKV` 威胁最大的论文之一。

它的关键词几乎就是：

- hierarchical context caching
- GPU-assisted I/O
- decoupled GPU / CPU memory layouts
- cache-aware request scheduling

它还明确指出一个和我们很像的系统矛盾：

- 现有系统在长上下文下会被 cache loading 拖慢
- 仅有分层存储还不够，还需要 scheduler 感知 I/O 延迟

换句话说，**“多级缓存 + 调度器协同 + I/O 优化” 这个组合，Strata 已经立起来了。**

如果我们现在继续用原版 `TierKV` 表述，很容易被审稿人理解成：

- 一个与 `Strata` 同方向、但换到 `vLLM` 上的版本

参考：

- https://arxiv.org/abs/2508.18572

## 2.4 HotPrefix：热度感知的 GPU / Host KV 调度

`HotPrefix` 说明这条线不仅有系统层，还有更细的 cache policy 竞争。

它聚焦 prefix-sharing workload，已经做了：

- 动态 hotness tracking
- selective KV cache admission
- 热前缀从 GPU 下沉到 host memory
- 高热度节点再优先提升回 GPU
- 预取 host memory 中可能复用的 KV cache

这意味着：

- 如果我们说 `TierKV` 的核心是“基于复用价值的放置、驱逐和提升”
- 那这个点本身也已经有人系统性做过

参考：

- https://doi.org/10.1145/3749168

## 2.5 TensorRT-LLM：工业级优先级驱动二级 KV 系统

`TensorRT-LLM` 的官方文档已经公开展示了一个非常成熟的工业设计：

- primary / secondary pools
- prioritized LRU
- retention policy
- host offloading
- partial reuse
- limited attention window aware pools

特别关键的是，它不仅支持 host offloading，还允许：

- 为 token range 分配 priority
- 设置 secondary offload threshold
- 让某些块只在 GPU 保留、不值得下沉到 host

这说明工业界已经把：

- 多级存储
- 优先级
- 保留策略

结合成一套产品化 KV 管理机制了。

因此，**“加优先级的多级 KV cache” 也不是空白地带。**

参考：

- https://nvidia.github.io/TensorRT-LLM/latest/features/kvcache.html

## 2.6 Mooncake 与 DualPath：远端 KV 与解耦调度

`Mooncake` 和 `DualPath` 分别说明了远端 tier 这条线也已经很深入。

`Mooncake` 的重点是：

- KVCache-centric 的 disaggregated serving
- 利用 CPU / DRAM / SSD 作为分离式 KV cache
- 用一个 KV-centric scheduler 在吞吐与延迟之间做平衡

`DualPath` 的重点是：

- agentic inference 下，瓶颈已经变成 KV-Cache storage I/O
- 不只是“放在哪”，还要重新设计 “从哪里到哪里加载”
- scheduler 需要全局地平衡 prefill / decode path

这说明如果我们想把 `TierKV` 往 remote tier 和 PD disaggregation 继续扩，它当然是重要问题，但也已经有强前作在这里。

参考：

- https://arxiv.org/abs/2407.00079
- https://arxiv.org/abs/2602.21548

## 2.7 InfiniGen：与 offloading 协同的动态 KV 管理

`InfiniGen` 是 `OSDI 2024` 工作，虽然它更偏“只预取必要 KV 条目”，不属于严格 exact reuse 的多级层次系统，但它明确证明：

- KV offloading 不只是容量问题
- 真正的关键是 **runtime 如何和 offloading 协同**
- 如果没有更强的运行时逻辑，仅仅把 KV 搬去 host memory 还不够

这一点对 `TierKV` 很重要，因为它说明：

- 顶会审稿人已经见过“动态 KV 管理”这件事
- 你的论文必须比“做一个更好的 offloading 层”讲得更深

参考：

- https://arxiv.org/abs/2406.19707

## 3. 我们的 `TierKV` 和已有工作到底重不重

## 3.1 已经明显重合的部分

当前版本 `TierKV` 的核心卖点，和已有工作重合度最高的是下面四项：

- GPU / CPU / Storage / Remote 的分层 KV cache
- 预取与写回
- 基于热度或价值的放置 / 驱逐 / 提升
- 调度器感知 KV 层级状态

对应占位大致如下：

- `LMCache` 已占据“通用多级 KV layer”
- `HiCache` 已占据“L1/L2/L3 hierarchical cache runtime”
- `Strata` 已占据“hierarchy + cache-aware scheduling + I/O optimization”
- `HotPrefix` 已占据“热度感知的 GPU/Host cache scheduling”
- `TensorRT-LLM` 已占据“工业级 priority-based secondary cache”
- `Mooncake / DualPath` 已占据“remote/disaggregated KV scheduling”

所以如果不改题，只是把现在的 `TierKV` 实现得更完整，审稿人非常可能会问：

- 这和 `LMCache / HiCache / Strata` 的差别到底是什么
- 为什么这不是“把已有的 hierarchical cache 做到 vLLM 上”

## 3.2 连“exact”也不构成差异化

这是一个非常关键、但容易被忽略的点。

当前版本 `TierKV` 用了“精确语义”或 `exact reuse` 这个表述，但在多级 offload / cache reuse 这条线上：

- `LMCache`
- `HiCache`
- `TensorRT-LLM` 的 host offload
- `Mooncake` 的 KV disaggregation

本质上也都是在复用 **原本已计算出来的 KV**，而不是近似压缩、量化或删 token。

所以：

- **exact** 在这条线上更像是默认前提，而不是贡献点

如果论文里把“保持 exact semantics”当作主 novelty，会显得不够有力。

## 4. 第一性原理检查：当前 `TierKV` 还剩什么真正没被占掉

如果硬要从 `TierKV` 里找还可能成立的空间，我认为只剩下面几类，而且它们必须一起出现才够强。

## 4.1 统一 control plane，而不是又一层 cache

现有工作虽然都很强，但它们往往各自专注于：

- 某个独立 cache layer
- 某个 serving engine
- 某种 workload
- 某种 data path

而当前 `vLLM` 的真实痛点是：

- `prefix cache`
- `SimpleCPUOffload`
- `LMCache/FlexKV`
- `NIXL/Mooncake`
- `Hybrid KV manager`

这些能力已经都在主线里，但它们还没有形成一个统一的 **control plane**。

这意味着如果论文命题变成：

> 如何把 vLLM 中分裂的 KV 生命周期、位置状态、异步迁移与调度接口统一为同一套 exact KV virtual memory control plane

这个点还有价值。

## 4.2 correctness 和一致性还没有被系统讲透

本地 `vLLM` 社区已经暴露出一个非常关键的问题：

- `SimpleCPUOffload` 有真实的 TOCTOU race

这说明多级 KV 的难点并不只是：

- 哪个块该留在 GPU

还包括：

- 命中判断后如何 pin
- allocator 何时能复用 block
- transfer 与 scheduling 如何协调
- preemption / request finish 时如何清理状态

我目前没有看到现有工作把“**exact KV hierarchy 的 correctness protocol**”作为核心 thesis 来系统展开。

如果你能把这件事讲清楚，它会比单纯的 offloading policy 更像系统论文。

## 4.3 hybrid-aware 多级管理仍然是缺口

`vLLM` 社区已经公开承认：

- hybrid model 需要 layer-specific KV dtype / layout / block size

这说明当前多级 KV 系统在真实异构模型上还远没收敛。

现有多级 KV 工作中，很多设计默认：

- block 粒度相对统一
- pool 假设较刚性
- 不需要同时兼容 hybrid groups、connector 和 runtime offload

如果你能把 `TierKV` 做成：

- group-aware
- heterogeneous-layout-aware
- exact reuse preserving

那它会比“普通多级 cache”更强。

但也要诚实：这会把实现难度显著拉高。

## 4.4 vLLM 缺的不是 backend，而是 scheduler-owned KV VM

`vLLM` 当前已经不缺 backend：

- SimpleCPUOffload
- LMCache
- FlexKV
- Mooncake
- NIXL

真正缺的是：

- scheduler 作为 **唯一权威者**
- 对 block residence、prefetch budget、transfer deadline、pinning state 有统一视图

所以我认为 `TierKV` 真正可打的 thesis 不是：

- one more hierarchical cache

而是：

- **scheduler-owned exact KV virtual memory for vLLM**

## 5. 这个方向和当前 vLLM 到底兼不兼容

## 5.1 兼容，而且接缝非常自然

和 `SegmentKV` 不同，`TierKV` 在当前 `vLLM` 上的兼容性是高的。

原因很简单：

- `V1 scheduler` 已经是主线
- `KVConnectorFactory` 已经存在
- `SimpleCPUOffloadConnector` 已经是主线实现
- `FlexKVConnectorV1 / LMCacheConnectorV1 / MooncakeConnector / NixlConnector` 都已经挂上统一入口
- `Hybrid KV cache manager` 也已经开始进入主线问题域

所以从架构演进上看，`TierKV` 不是逆势重写，而是在顺势整合。

## 5.2 但它不是“低成本补丁”

虽然兼容性高，但这条线不会是小改动。

如果要把它做成真正有论文价值的版本，你至少会碰到：

- scheduler
- kv_cache_manager
- kv_cache_coordinator
- connector metadata
- offload manager
- metrics / observability

并且必须同时设计：

- 命中查询语义
- pin / reserve / release 协议
- transfer completion 回传
- block residence metadata

也就是说：

- **它可做，但不便宜。**

## 5.3 真实代码状态支持这个判断

当前 `vLLM` 的 GitHub 信号和本地代码信号其实非常一致：

- `SimpleCPUOffload` 明确复用了 `BlockPool` 和 `KVCacheCoordinator`
- `FlexKV` 已经被作为 multi-level cache backend 接入
- 社区对 hybrid model 的 KV 抽象仍不满意
- Simple offload 路径已经暴露 correctness race

这组事实组合起来，刚好说明：

- 路径是对的
- 但体系还没有完成

## 6. 它是否有效

## 6.1 作为“多级 KV”本身，它显然有效

这一点其实已经被现有工作广泛证明了：

- `LMCache`
- `HiCache`
- `Strata`
- `Mooncake`
- `HotPrefix`
- `TensorRT-LLM`

都说明了：

- 更多层级的 KV 容量
- 更聪明的放置与预取
- 与 scheduler / I/O 协同

确实能带来：

- 更低 TTFT
- 更高 throughput
- 更高 cache hit ratio
- 更好的长上下文与多轮会话表现

所以如果问题只是：

> 多级 exact KV 系统有没有用

答案是：**非常有用，而且已经被反复证明。**

## 6.2 作为“当前版本的 TierKV 论文主张”，它不够稳

问题不在于它无效，而在于：

- **它太容易被归类为已有路线的又一个实现。**

如果论文主张仍然是：

- 统一 GPU / CPU / Remote KV
- 做收益感知放置
- 做预取与回写

那我认为 novelty 安全性不够。

## 6.3 作为“升级版 TierKV”，它仍然有机会

如果把论文问题重新定义成下面这个版本，`TierKV` 仍然是有机会的：

> 在 `vLLM` 这类连续批处理推理引擎中，KV reuse、CPU offload、外部 KV connector 与 PD transfer 已经并存，但它们缺乏统一的状态语义、调度接口与一致性协议。如何把这些分裂路径提升为一套 scheduler-owned exact KV virtual memory system，并在 hybrid-aware、多 backend、异步迁移场景下同时保证 correctness 与性能？

这个版本和现有工作的差别会更清楚：

- 不再只是“分层 cache”
- 而是“统一运行时语义 + 正确性 + 调度协同”

## 7. 它是否足够支撑系统类顶会论文

我会分两个版本给判断。

## 7.1 作为当前版本的 `TierKV`

如果仍然按当前默认表述推进，我给它的判断是：

- **可做性：高**
- **收益概率：高**
- **重复风险：中高**
- **OSDI 级把握：中低**

原因是：

- 做出来大概率会有效
- 但容易被归类为：
  - `LMCache / HiCache / Strata` 同线工作
  - vLLM 上的更完整工程实现

## 7.2 如果要把它推到更像 OSDI

我建议至少补齐下面四个升级点。

### 升级点 A：把标题从 “hierarchy” 提升为 “KV virtual memory”

原因很简单：

- `hierarchical cache` 已经被很多系统讲过了
- `exact KV virtual memory runtime` 才更像新的系统主张

### 升级点 B：把 novelty 压到 control plane 和 correctness 上

也就是明确说：

- 我们的核心不是存储层级本身
- 而是统一 block residence / transfer / pinning / visibility / completion 的控制平面

### 升级点 C：把 hybrid-aware 支持变成正式贡献

否则它和 `HiCache / LMCache` 的差别仍然不够大。

### 升级点 D：要证明广泛收益，而不是单场景收益

实验至少要覆盖：

- 单机 GPU + CPU
- GPU + CPU + Remote
- 长上下文
- 多轮会话
- PD / external KV
- 最好再补 hybrid model

## 8. 我的最终判断

基于截至 2026-04-14 的论文、工业方案、vLLM 社区讨论和本地代码状态，我对 `TierKV` 的结论是：

- **它不是空想。**
- **它一定有效。**
- **但它的“默认版本”已经不够新。**

更具体地说：

- 作为工程方向，它非常合理
- 作为性能优化，它大概率有效
- 作为论文主线，它必须主动升级 thesis 才安全

如果必须给一个明确建议，我会写成：

> 不建议把“面向 vLLM 的多级精确 KV 系统”按当前表述直接作为主论文版本继续推进；更稳妥的做法是，把它重新定义为 `scheduler-owned exact KV virtual memory for vLLM`，把 novelty 聚焦到统一 control plane、异步迁移正确性和 hybrid-aware 运行时协同上。

## 9. 一页式结论

### 9.1 结论打分

- 方向价值：`9 / 10`
- 对当前 vLLM 的兼容性：`9 / 10`
- 作为性能优化的有效性：`9 / 10`
- 当前版本的 novelty 安全性：`5 / 10`
- 升级为更强 thesis 后的系统论文潜力：`7.5 / 10`

### 9.2 最终建议

- 保留 `TierKV`，但不要继续沿着默认版本直接写论文
- 立即把问题重写成 `KV virtual memory` thesis
- novelty 不再押“多级”本身，而押：
  - 统一 control plane
  - correctness protocol
  - hybrid-aware scheduler-coordination
