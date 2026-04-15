# PolyStateKV：把可复用状态提升为多表示对象

## 1. 一句话 thesis

`PolyStateKV` 的核心主张是：

> 现代 LLM serving 不应该把“可复用状态”固定理解为单一 `KV cache`，而应把它建模为可在多种状态表示之间切换的一等对象，由 runtime 根据复用距离、内存预算、恢复带宽与延迟目标动态选择状态形态。

## 2. 学术问题定义

现有系统大多默认一个前提：

- 可复用状态要么是完整 `KV`
- 要么是某一种替代品
- 然后围绕这一种表示做内存、offload、恢复或调度优化

但随着 workload 越来越复杂，这个前提开始变得不自然：

- 热状态适合保留完整 `KV`
- 中温状态也许更适合保留较轻的 hidden-state checkpoint
- 冷状态可能更适合保留 activation-based restoration 信息
- 某些长上下文路径则更适合压缩或分解状态表示

因此，真正值得研究的问题不再是：

- “哪一种状态表示最好”

而是：

- “可复用状态是否应被抽象为可变表示对象”
- “runtime 是否应动态决定状态表示，而不是静态绑定一种 cache 形态”

这就是 `PolyStateKV` 的学术问题。

## 3. 为什么它更像论文而不是工程 patch

这条线不是：

- 再做一个新的 KV offload
- 再做一个新的 hidden cache
- 再做一个新的 low-rank cache

它的核心不是某一种机制，而是新的系统抽象：

> reusable state form polymorphism

也就是：同一个“可复用上下文状态”，在不同温度、距离和资源条件下，可以被 materialize 成不同表示，而 runtime 负责在这些表示之间切换。

如果这个抽象成立，那么很多今天看起来分裂的工作，其实都只是这个抽象下的特例。

## 4. 最近邻工作与边界

### 4.1 最近邻

- `HCache`
  - 从 intermediate activations 恢复状态，以降低 TTFT 和存储成本。
- `Apt-Serve`
  - 结合 `KV cache` 和 hidden cache，提高 batch size 与 effective throughput。
- `HybridServe`
  - 结合 activation checkpointing 与 hybrid caching，平衡重算与 host-memory offloading 开销。
- `ShadowKV`
  - 通过 low-rank key + offloaded value 的方式降低长上下文推理中的 memory footprint。
- `Cake`
  - 协同 compute 与 load 完成 `KV` 恢复，优化 prefix reuse 下的 TTFT。

### 4.2 它们和 PolyStateKV 的区别

这些工作都很强，但它们的共同特点是：

- 选择了一种特定状态表示
- 或围绕某种固定 hybrid 设计 / 固定恢复路径优化服务系统

`PolyStateKV` 的边界则更高一层：

> We do not advocate one better state form. We ask whether reusable state itself should be a polymorphic runtime object.

也就是说，`PolyStateKV` 研究的是：

- 哪些状态形式应该共存
- 什么时候切换表示
- 切换代价如何建模
- 哪种请求 / 前缀 / 生命周期该绑定到哪种表示

它和 `HCache / Apt-Serve / HybridServe / ShadowKV / Cake` 不是同一命题。

## 5. 为什么这个问题现在值得提出

近两年相关系统已经在不同侧面暴露出一个信号：

1. 单纯保存 `KV` 会面临巨大的内存压力。
2. 单纯恢复 `KV` 又会带来大量 IO 或重算。
3. 仅靠一种状态表示很难同时兼顾：
   - 高吞吐
   - 低 TTFT
   - 高并发
   - 长上下文
4. 不同 workload 对状态表示的偏好并不相同：
   - chat / RAG
   - long context QA
   - tool-use / agent
   - pause/resume 式 workload

这些现象说明：**状态表示本身已经成为 serving runtime 的调度维度。**

## 6. PolyStateKV 的核心机制

### 6.1 状态对象抽象

为每个 reusable context state 定义统一对象：

- `semantic identity`
  - 它表示的是哪段上下文状态
- `current form`
  - 当前以哪种物理表示存在
- `recovery cost`
  - 从当前表示恢复到可执行状态的成本
- `space cost`
  - 当前表示的存储开销
- `reuse horizon`
  - 预期多快会再次被用到
- `sla class`
  - 该状态所服务请求的延迟敏感等级

### 6.2 多表示集合

从长期研究角度，`PolyStateKV` 可以容纳多种状态表示；但从第一版论文与原型边界看，必须主动收紧。

第一版建议只保留两种形式：

- full `KV`
- hidden-state checkpoint

activation-based restoration state 可以作为后续扩展，但不建议在第一版主结果里与前两者并列。

论文重点不是“表示越多越好”，而是证明：

- 单表示系统会在某些区域系统性吃亏
- 少量多表示已足以改善整体 Pareto front

### 6.3 形式切换策略

runtime 决定何时：

- 保持 full `KV`
- 降级为 lighter state
- 升级回 full `KV`

粗略决策依据：

- 最近复用概率
- 内存压力
- 恢复带宽
- 预计 TTFT penalty
- workload SLO

## 7. 为什么它能在 vLLM 中验证

`vLLM` 不是这条 thesis 的全部，但它是一个很好的原型载体，因为它已经具备若干必要部件：

- 本地 `KV cache`
- `KV offload`
- `connector / event` 机制
- `scheduler`
- async / chunked prefill 路径

因此，在 `vLLM` 中可以做一个轻量 prototype：

1. 热状态保留 full `KV`
2. 次热状态保留 lighter checkpoint
3. 在请求到来时，根据恢复代价和内存压力决定是否 rematerialize

这足以验证论文命题，而不需要一开始就把所有状态表示全部实现出来。

## 8. 为什么它满足你的需求

### 8.1 轻量级

它不要求额外分布式控制平面，也不要求改模型。

### 8.2 普适性

它不依赖某一种特定 workload，只要系统存在“状态重用但内存受限”的矛盾，这条线就成立。

### 8.3 高吞吐

通过更合理地配置状态形态，可以在相同显存下容纳更多活跃请求或更多可复用上下文。

### 8.4 低延迟

相比统一 offload 或统一 recompute，它有机会在 TTFT 上取得更优折中。

## 9. 最大风险

最大的风险不是撞车，而是**问题太大**。

如果不控制边界，它会迅速膨胀成：

- 一个“大而全”的多级状态管理系统
- 一个过于复杂的 cache taxonomy
- 一个 prototype 很难做完的系统

因此它必须坚持两个边界：

1. 第一版只做 2 种状态形式
2. 重点证明“多表示抽象比单表示抽象更对”，而不是把所有优化都堆上去

## 10. 当前判断

截至这轮调研，`PolyStateKV` 是当前三个方向里**最像新的系统论文命题**的一条。

它的价值不在于某个单点机制，而在于它有机会把今天分散在：

- `KV cache`
- hidden cache
- activation restoration
- compressed state

这些工作背后的共性，统一到一个更高层的 reusable-state 抽象中。

如果后续继续排重后仍然成立，它很可能是这一轮最值得主攻的主线。

## 11. 关键资料

- HCache: <https://arxiv.org/abs/2410.05004>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- Efficient LLM Inference with Activation Checkpointing and Hybrid Caching (`HybridServe`): <https://arxiv.org/abs/2501.01792>
- ShadowKV: <https://arxiv.org/abs/2410.21465>
- Cake: <https://arxiv.org/abs/2410.03065>
- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM Hybrid KV Cache Manager: <https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/>

## 12. 本轮补充文档

- [排重与论文可行性评估](overlap_feasibility_assessment.md)
  这份补充文档重点回答三件事：
  `PolyStateKV` 为什么不能写成宽泛 hybrid cache；
  它和 `HCache / Apt-Serve / ShadowKV / Cake` 的真正边界；
  以及在 `vLLM` 中最合理的第一版 prototype 应该长什么样。
- [相关工作矩阵](related_work_matrix.md)
  把 `HCache / Apt-Serve / HybridServe / ShadowKV / Cake` 放到同一张表里，对比它们与 `PolyStateKV` 是否真的是同一命题。
- [论文表述边界与 Claim Guardrails](paper_claim_guardrails.md)
  固定哪些话绝对不能写、哪些一句话命题最安全，以及摘要/引言/贡献点应该怎样表达。
- [实验矩阵与判死标准](experiment_matrix.md)
  固定第一版原型的基线、workload、指标、消融与 kill criteria，避免后续实验继续漂移。
- [vLLM 最小原型设计](prototype_design.md)
  把 `PolyStateKV` 第一版在 `vLLM` 里的实际落点钉到模块级别，明确哪些模块需要改、统一状态对象怎么落，以及 selector 最小应该如何实现。
- [Profiling 与实验执行计划](profiling_plan.md)
  把实验矩阵继续下钻成可执行 trace、benchmark/wrapper 脚本、指标采集点、结果目录结构与 baseline 运行方案。
