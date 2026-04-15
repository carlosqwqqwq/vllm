# PolyStateKV：相关工作矩阵

## 1. 这份矩阵要回答什么

这份文档不重复罗列论文摘要，而只回答一个更关键的问题：

> `PolyStateKV` 与最容易被审稿人联想到的近邻工作，到底是不是同一命题。

判断标准不是关键词像不像，而是看它们分别在回答什么系统问题。

## 2. 最近邻工作矩阵

| 工作 | 核心问题 | 主要状态形式 | 是否固定到特定 form | 是否做在线 form selection | 是否强调统一语义身份 | 主要收益目标 | 与 PolyStateKV 的关系 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `HCache` | 如何用更低 I/O / 计算代价恢复已被驱逐的状态 | intermediate activations | 是 | 否 | 否 | 降 TTFT、降 storage cost | 它发明一种更优 restoration medium；`PolyStateKV` 研究的是 form 选择本身 |
| `Apt-Serve` | 如何通过 `KV + hidden cache` 提升 effective throughput | `KV` + hidden cache | 是 | 部分调度在线，但 form 集合固定 | 否 | 提升 effective throughput | 它是特定 hybrid cache + scheduling；`PolyStateKV` 不绑定到固定二元组合 |
| `HybridServe` | 在 host-memory offloading 下，如何结合 activation checkpointing 与 hybrid caching 平衡重算与参数加载 | `KV` + activation cache | 是 | 主要是 ratio tuning，不是统一对象级 form selection | 否 | 提升 offloading 场景吞吐 | 它进一步压缩了“activation + hybrid cache”叙事空间，因此 `PolyStateKV` 不能再写成 activation/KV hybrid |
| `ShadowKV` | 如何重构长上下文下的 `KV` 物理表达与访问路径 | low-rank key + offloaded value | 是 | 否 | 否 | 降显存、提长上下文吞吐 | 它优化某一种表示；`PolyStateKV` 研究多表示对象与表示选择 |
| `Cake` | prefix reuse 下如何协同 compute 与 load 完成恢复 | restored `KV` | 是 | 否 | 否 | 降 TTFT、用满 compute + I/O | 它解决恢复路径调度；`PolyStateKV` 解决状态形式选择语义 |
| `PolyStateKV` | reusable state 是否应成为统一身份的多表示对象，以及 runtime 如何在线选择 form | `KV` + lighter checkpoint form（第一版只做两种） | 否 | 是 | 是 | 优化 `TTFT/throughput/memory` 的 Pareto front | 当前主线 |

## 3. 对照后的关键结论

### 3.1 `PolyStateKV` 不能写成 “another hybrid cache”

如果对外表述停留在：

- `KV + hidden cache`
- `KV + activation cache`
- 根据压力在两种状态间切换

那审稿人完全可以把它归入：

- `Apt-Serve`
- `HybridServe`
- `HCache`

这时即使实现不同，论文命题也很难成立。

### 3.2 `PolyStateKV` 的真正边界是 “对象模型 + 在线选择”

`PolyStateKV` 想保住学术性，必须牢牢站在下面两点上：

1. 同一个 reusable state 具有**统一语义身份**
2. runtime 基于统一成本模型做**在线 form selection**

这两点一旦丢掉，它就会退化成已有工作的重述。

### 3.3 第一版必须主动回避 activation-heavy 叙事

考虑到 `HCache` 和 `HybridServe` 都已经把 activation 相关叙事做得很强，`PolyStateKV` 第一版原型更稳妥的范围应该是：

1. full `KV`
2. hidden-state checkpoint

而不是：

1. full `KV`
2. hidden checkpoint
3. activation restoration

后者虽然更“完整”，但论文边界会更危险，且原型成本更高。

## 4. 现在最安全的 related-work 定位句

下面这句是当前更安全的 related-work 定位：

> Existing systems have separately proposed better restoration media, fixed hybrid cache designs, compressed KV representations, or compute/load coordinated recovery. `PolyStateKV` asks a different systems question: whether reusable state itself should be treated as a first-class runtime object with multiple materialization forms and online form selection under a unified identity and cost model.

## 5. 当前判断

截至 `2026-04-15`，我对 related work 的最终判断是：

- `PolyStateKV` 周围的近邻已经非常强；
- 但只要它坚持“统一对象模型 + 在线 form selection”这条边界，它仍然**没有被现成公开工作完全覆盖**；
- 如果它回退成 “hybrid cache with two or more state forms”，那它就不再安全。

## 6. 关键资料

- HCache: <https://arxiv.org/abs/2410.05004>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- Efficient LLM Inference with Activation Checkpointing and Hybrid Caching (`HybridServe`): <https://arxiv.org/abs/2501.01792>
- ShadowKV: <https://arxiv.org/abs/2410.21465>
- Cake: <https://arxiv.org/abs/2410.03065>
