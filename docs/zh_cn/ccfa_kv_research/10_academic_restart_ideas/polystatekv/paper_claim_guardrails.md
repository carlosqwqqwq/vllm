# PolyStateKV：论文表述边界与 Claim Guardrails

## 1. 这份文档的目标

`PolyStateKV` 最大的现实风险，不是实现难，而是**写法失手**。

这份文档只做一件事：

> 把能写、不能写、应该怎么写，提前定死。

## 2. 最安全的一句话命题

当前最安全的一句话论文命题是：

> Modern LLM serving should not bind reusable state to a single physical form. Instead, reusable state should be treated as a first-class runtime object with unified identity and multiple materialization forms, where the central systems problem is online state-form selection under memory, recovery, and latency constraints.

对应中文可以写成：

> 现代 LLM serving 不应把可复用状态固定为单一物理表示；更合理的系统抽象是把它建模为具有统一语义身份、支持多种 materialization 形式的一等运行时对象，而系统核心问题是在内存、恢复代价与延迟约束下进行在线状态形式选择。

## 3. 绝对不要这样写

下面这些写法现在都不安全：

- “我们提出一种新的 hybrid cache”
- “我们首次把 `KV`、hidden cache 和 activation cache 统一起来”
- “我们发明了一种多层状态缓存系统”
- “我们比已有工作更全面地支持多种状态表示”
- “我们是第一个做多表示状态管理的系统”

这些表述的问题分别是：

- 容易直接撞上 `Apt-Serve`
- 容易直接撞上 `HCache / HybridServe`
- 容易把论文写成工程集成
- 容易缺少新的核心命题
- 容易过度宣称

## 4. 可以这样写

下面这些表述更安全：

- “现有系统分别优化了特定状态表示，但仍默认可复用状态应绑定到单一物理形式”
- “我们关注的是 reusable state 的对象模型，而不是某一种具体 state form”
- “本文研究统一身份下的在线 state-form selection，而不是固定 hybrid cache 设计”
- “我们的目标不是证明某一种状态表示普遍最优，而是证明单一表示默认前提在现代 workload 下并不成立”

## 5. 贡献点应该怎么写

更稳的贡献点结构建议控制在 3 条：

1. **问题与抽象**
   - 提出 reusable state polymorphism 这一系统抽象；
   - 将 serving 问题从“选择哪一种状态表示”提升为“是否应在线选择状态形式”。

2. **运行时机制**
   - 定义统一语义身份、统一成本模型和最小 online selector；
   - 给出 admission、demotion、rematerialization 的运行时规则。

3. **证据**
   - 在 `vLLM` 上实现轻量原型；
   - 证明两种状态形式的在线选择优于固定单一形式；
   - 展示 `TTFT / throughput / memory` Pareto front 的改善。

## 6. 摘要第一段应该强调什么

摘要第一段不要从 “we build a hybrid cache” 开始，而应从下面这个矛盾开始：

- 现代 serving workload 的 reuse distance、SLA 与 context length 高度异质；
- 单一状态表示无法同时兼顾 `memory / restoration cost / latency`；
- 现有系统通常只优化某一种固定表示，而没有把“状态形式选择”本身当成系统问题。

## 7. 引言里必须提前交代的边界

引言早期就要主动承认：

1. 我们**不是**在发明新的 hidden cache、activation cache 或 compressed KV form。
2. 我们**不是**在证明某一种 state form 始终更优。
3. 我们研究的是：
   - 同一 reusable state 的统一对象化表示；
   - 以及基于统一成本模型的在线 form selection。

这样可以主动切开与 `Apt-Serve / HCache / HybridServe / ShadowKV` 的命题边界。

## 8. 结果段落应该怎么写才安全

结果不要写成“全面超过所有系统”，更安全的写法是：

- 在混合 workload 下，相比固定 full `KV`，我们提升了 memory-efficiency 下的可承载并发；
- 相比固定 lighter form，我们避免了低 reuse-distance 请求的额外 rematerialization penalty；
- 因此我们改善的是整体 `TTFT/throughput/memory` Pareto front，而不是单一指标上的绝对统治。

## 9. 第一版原型的对外范围

为了保持边界清晰，第一版论文稿建议只写：

1. full `KV`
2. hidden-state checkpoint

不要在主稿中把 activation restoration 作为同等主角。  
如果后续确实实现了，可以放到：

- 扩展讨论
- 附录
- future work

## 10. 当前最终 guardrail

一句话总结：

> `PolyStateKV` 能成立，不是因为它“用了更多状态形式”，而是因为它把“状态形式选择”本身提升成了一个新的系统问题。

如果后续任何写法开始偏离这句话，就应该立刻删改。

## 11. 关键资料

- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- Fast State Restoration in LLM Serving with HCache: <https://arxiv.org/abs/2410.05004>
- Efficient LLM Inference with Activation Checkpointing and Hybrid Caching: <https://arxiv.org/abs/2501.01792>
- ShadowKV: <https://arxiv.org/abs/2410.21465>
- Cake: <https://arxiv.org/abs/2410.03065>
