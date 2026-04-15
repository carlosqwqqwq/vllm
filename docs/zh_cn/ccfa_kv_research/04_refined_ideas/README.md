# 细化方案与对比

这部分是第二轮调研的收敛结果。目标不是“再列一批点子”，而是把真正有机会做成系统类 CCF-A 论文、并且能兼容当前 `vLLM` 演进路线的方案挑出来，再用更稳定的目录结构把它们收好。

## 1. 这一层目录为什么单独存在

前面的 `01~03` 更偏：

- 事实收集
- 社区趋势
- 初步候选方向

而这一层更偏：

- 主线方案收敛
- 方案间对比
- 重复风险与可行性评估
- 是否值得继续投入

也就是说，这里不是“idea 仓库”，而是“**已经进入论文候选阶段的方案区**”。

## 2. 这一层目录现在怎么组织

为了避免文档继续平铺，我把每个主方案都收进了自己的子目录：

- `tierkv/`
- `affinitykv/`
- `qoskv/`
- `segmentkv/`
- `heterokv/`

顶层只保留三类文档：

- 总览入口：当前这个 `README`
- 目录逻辑说明：[目录结构与阅读方式](directory_guide.md)
- 横向决策文档：[方案对比与推荐](comparison.md)

这样后面继续补充某个 idea 的“可行性评估 / OSDI 评估 / 实验草案 / related work”时，不会再把顶层挤乱。

## 3. 第二轮调研后的五条主线

### 3.1 TierKV

最推荐的主线。把 GPU prefix cache、CPU offload、LMCache/FlexKV/NIXL 统一成一个 **scheduler 协同的多级精确 KV hierarchy**，并继续升级成 `KV virtual memory` thesis。

### 3.2 AffinityKV

围绕 `DP rank / 多实例 / production-stack router` 做 **KV 感知路由与会话亲和性**。这是官方路线已经明确承认的重要空缺，而且很适合线上部署论文。

### 3.3 QoSKV

把 prefix caching 从“能命中”提升到“命中且稳定、低延迟、少污染”。它聚焦 APC 的真实线上问题：HoL blocking、死缓存、热前缀不稳定、可观测性不足。

### 3.4 SegmentKV

针对 RAG、Agent、工具调用等 **非严格前缀但高重复** workload，做分段 KV 复用，并单独评估它与现有论文的重合度和 exact 语义边界。

### 3.5 HeteroKV

面向 Hybrid LLM 的异构 KV 虚拟内存。新意很强，但实现和实验风险都更高。

## 4. 当前推荐排序

如果以“**系统类 CCF-A 论文成功率**”排序，我现在的判断是：

1. **TierKV**：最稳，问题大、系统味最强、最贴近当前 `vLLM` 主线。
2. **AffinityKV**：非常值得认真考虑，尤其适合你们如果能拿到多实例或集群环境。
3. **QoSKV**：如果你更看重低延迟和线上可用性，这条线很有现实冲击力。
4. **SegmentKV**：如果你们场景强依赖 RAG/Agent，它会非常强；否则通用性略弱。
5. **HeteroKV**：适合做 harder problem，但实现周期明显更长。

## 5. 建议怎么读

建议按下面顺序阅读：

1. 先看 [目录结构与阅读方式](directory_guide.md)，知道这一层为什么这样组织。
2. 先读 `TierKV` 子目录，把当前最值得继续打磨的主线抓牢。
3. 再看 `AffinityKV` 和 `QoSKV`，判断你们更偏“吞吐/集群”还是“时延/交互”。
4. 如果你们业务明显是 RAG/Agent，再看 `SegmentKV`。
5. 如果你们愿意打 hybrid harder problem，再看 `HeteroKV`。
6. 最后用 [方案对比与推荐](comparison.md) 做决策。

## 6. 维护原则

后续如果继续补文档，默认遵循下面两条：

- 某个 idea 的相关文档，优先继续放进该 idea 的子目录，不再回到顶层平铺。
- 顶层只保留“总览、结构说明、横向对比”这类跨 idea 文档。
