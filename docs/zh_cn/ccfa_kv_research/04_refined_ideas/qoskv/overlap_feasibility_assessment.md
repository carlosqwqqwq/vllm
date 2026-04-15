# QoSKV：同领域重合度与可行性评估

## 1. 这份评估要回答什么

截至 `2026-04-14`，`QoSKV` 这条线已经不能只靠“prefix cache 很重要”来立题了。真正需要回答的是三个更尖锐的问题：

- 业界和学界是不是已经把“热前缀保留、前缀感知调度、前缀层次缓存”做得差不多了？
- 如果这些点都已经有人做过，我们现在说的 `QoSKV` 还剩下什么没有被占掉？
- 这条线在当前 `vLLM` 上到底是不是一个真实、有效、可验证的问题，而不是把几个 feature request 拼在一起？

我这次评估的结论先放在前面：

- `QoSKV` **不是空方向**，因为 `vLLM` 社区已经出现了一组非常一致的真实问题：`HoL blocking`、前缀污染、热前缀不稳定、状态不可观测、prefix hit 低于预期。
- 但 `QoSKV` **也不是空白方向**。如果把它简单定义成“hot prefix retention + prefetch + cache-aware scheduling”，那和已有工作重合度已经很高。
- 这条线仍然可以成立，但前提是把 thesis 收紧成：**让 automatic prefix caching 在低延迟交互场景下成为 latency-safe、pollution-aware、observable 的运行时机制**。

## 2. 同领域已经有哪些工作

## 2.1 CachedAttention：多轮会话 + 分层 KV + scheduler-aware fetching

`CachedAttention` 已经把“多轮对话里的历史 KV 持久化下来，再做层次缓存和异步预取”这条线讲得很完整了。论文明确写到：

- 它面向 multi-turn conversations；
- 它有 hierarchical KV caching system；
- 它有 layer-wise pre-loading、asynchronous saving；
- 它还有 scheduler-aware fetching and eviction。

这意味着，如果我们把 `QoSKV` 讲成“前缀缓存要做分层保留和调度协同”，那这个说法本身并不新。

参考：

- [CachedAttention, arXiv:2403.19708](https://arxiv.org/abs/2403.19708)

## 2.2 HotPrefix：热度感知的 admission / eviction / promotion

`HotPrefix` 和当前 `QoSKV` 的表面重合度非常高。它的三个核心机制是：

- `Dynamic Hotness Tracking`
- `Selective KV Cache Admission`
- `Hotness Promotion`

而这三件事，几乎正对应我们最初对 `QoSKV` 的三条直觉：

- 热前缀识别
- 保护高价值前缀
- 把重要前缀从慢层搬回快层

更要命的是，它已经把这些点系统化实现了，而且还直接和 `vLLM with prefix sharing` 做了对比。因此，**如果 QoSKV 只停留在“热度感知 APC”这个层面，重复风险很高**。

参考：

- [HotPrefix, SIGMOD 2025 / DOI](https://doi.org/10.1145/3749168)
- [HotPrefix PDF](https://cs.nju.edu.cn/tianchen/lunwen/2026/sigmod26-liyuhang.pdf)

## 2.3 Prefix-aware scheduling：QoS 与 prefix reuse 的冲突已经被单独拿出来研究

`LLM Query Scheduling with Prefix Reuse and Latency Constraints` 这篇工作非常关键，因为它几乎直接戳中了 `QoSKV` 的主张核心。论文明确指出：

- 现有 `FCFS` 和 `LPM` 在 prefix reuse 下对 latency constraints 有局限；
- prefix reuse 下的调度问题在 `TTFT` 约束下是困难的；
- 需要在 prefix reuse 和 fairness 之间做平衡。

这说明：

- “prefix reuse 不等于低时延”这个洞见已经被论文化了；
- 如果我们只说“前缀缓存会拖累 TTFT，所以要做更公平的调度”，这个命题本身不新。

参考：

- [LLM Query Scheduling with Prefix Reuse and Latency Constraints, arXiv:2502.04677](https://arxiv.org/abs/2502.04677)

## 2.4 Preble / KVFlow / Continuum：prefix-aware scheduling 已经从单机扩展到工作流与 agent

这一组工作进一步压缩了 `QoSKV` 的 novelty 空间：

- `Preble` 做的是 distributed prompt scheduling，明确把 prefix reuse 和 load balancing 联合优化。
- `KVFlow` 做的是 workflow-aware prefix cache management，用步骤图预测未来复用，并做预取和 eviction。
- `Continuum` 做的是 multi-turn agent scheduling with KV cache time-to-live，通过 tool-aware TTL pinning 来保护多轮连续性。

它们共同说明：**prefix reuse 的问题已经从“缓存命中”演化到“调度与生命周期管理”**，而且这些工作并不局限在一个小 patch 上。

参考：

- [Preble, arXiv:2407.00023](https://arxiv.org/abs/2407.00023)
- [KVFlow, arXiv:2507.07400](https://arxiv.org/abs/2507.07400)
- [Continuum, arXiv:2511.02230](https://arxiv.org/abs/2511.02230)

## 2.5 KVCache Cache in the Wild：真实生产 trace 已经证明 eviction policy 是 workload-dependent

这篇 `USENIX ATC 2025` 工作的重要价值不在于某个具体机制，而在于它给 `QoSKV` 带来一个现实判断：

- KV cache reuses 在真实云 workload 里是 skewed 的；
- reuse time 和 probability 会随着请求类别而变化；
- eviction policy 是高度 workload-dependent 的。

这等于从实证层面确认了：**简单 LRU 并不天然适合线上 prefix reuse**。所以 `QoSKV` 在“真实问题是否存在”这一点上，其实是被强化了；但同时也说明，**做一个 workload-aware policy 本身已经不是新鲜事**。

参考：

- [KVCache Cache in the Wild, arXiv:2506.02634](https://arxiv.org/abs/2506.02634)

## 2.6 Bidaw / ContiguousKV / TensorRT-LLM：工业界已经在做交互式 KV 的时延保护

这条线提醒我们，不要把 `QoSKV` 误写成“只有开源框架才在考虑交互式 latency”。

- `Bidaw` 面向 interactive LLM serving，强调 bidirectional computation-storage awareness，并且专门减少 I/O-induced request blocking。
- `ContiguousKV` 面向 persistent prefix KV cache offloading，核心目标是降低 re-prefill 阶段的 I/O 气泡与读取放大。
- `TensorRT-LLM` 官方文档已经明确支持 `KV cache reuse`，并指出 host offload、LRU 限制、以及 retention / priority 的必要性。

这说明：

- 前缀缓存在交互式场景中的“阻塞、保留、慢层回收”问题已经被工业界正面处理；
- `QoSKV` 若只是提出“给热前缀更高优先级”或者“支持 pinned prefixes”，很难形成足够强的新意。

参考：

- [Bidaw, FAST 2026 PDF](https://www.usenix.org/system/files/fast26-hu-shipeng.pdf)
- [ContiguousKV, arXiv:2601.13631](https://arxiv.org/abs/2601.13631)
- [TensorRT-LLM KV cache reuse docs](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)

## 3. 我们的 `QoSKV` 和已有工作到底重不重

## 3.1 已经明显重合的部分

如果 `QoSKV` 保持当前最直观的定义，即：

- 前缀分类
- 热前缀保留
- cache-aware scheduling
- 适度预取
- 多加一些指标

那它和现有工作会有以下重合：

- 和 `HotPrefix` 重合在 `hotness-aware admission / retention / promotion`
- 和 `CachedAttention`、`Bidaw`、`ContiguousKV` 重合在 `scheduler-aware fetching / eviction / preloading`
- 和 `LLM Query Scheduling...`、`Preble`、`KVFlow`、`Continuum` 重合在 `prefix-aware scheduling + fairness / latency`
- 和 `TensorRT-LLM`、社区的 `pinned prefix` 诉求重合在 `priority retention`

也就是说，**QoSKV 不能再作为“prefix cache policy 大杂烩”来讲**。如果这么讲，评审很容易一句话打掉：这些点分别都有人做过了。

## 3.2 还没被完全占掉的部分

我认为 `QoSKV` 还剩下的、并且最值得抓住的空隙，不是“某一个 policy”，而是下面这个系统命题：

> Automatic Prefix Caching 在交互式低延迟场景下，不仅会带来 hit，也可能悄悄引入 `HoL blocking`、死缓存竞争、热前缀抖动和观测盲区。系统需要一个 latency-safe 的 APC runtime，把 prefix reuse 收益稳定转化为用户可见的低延迟。

这个命题的独特点在于：

- 它不把焦点放在“如何进一步提高 hit ratio”；
- 它把焦点放在“为什么 hit 了却仍然可能高 TTFT / 高 P99”；
- 它试图统一 `pollution control`、`QoS-aware scheduling`、`state observability` 三件事。

我目前没有看到哪篇论文正好把这三者在 `vLLM` 这种 APC 语义里绑定成一个完整 thesis。现有工作大多更偏：

- 热度优化
- 分层 offload
- 分布式 prompt routing
- agent workflow 预测

而不是**“交互式 APC 的 QoS 正确性”**。

## 4. 第一性原理检查：当前 `vLLM` 到底有没有这个问题

## 4.1 代码层面，当前 APC 更像“命中导向”而不是“QoS 导向”

从本地代码看，`vLLM` 当前 APC 的主线语义非常清晰：

- `Request.append_output_token_ids()` 会把输出 token 继续并入 `all_token_ids`，随后重新更新 block hashes；
- `KVCacheManager.get_computed_blocks()` 的核心职责是找 longest cache hit；
- prefix cache stats 目前主要是 `queries / hits` 这类基本计数。

这套设计非常适合回答“有没有命中”“命中了多少 token”，但不擅长回答：

- 哪些 cached blocks 已经几乎不可能再命中；
- 命中为什么没有转化成更好的 TTFT；
- 是 active requests 造成压力，还是 idle prefix cache 在挤占容量；
- 某个 latency regression 是调度问题、污染问题，还是 retention 问题。

换句话说，**当前 APC 的语义中心更接近 cache efficiency，而不是 service QoS**。

## 4.2 社区已经出现了典型的“命中了，但用户没受益”现象

这是我认为 `QoSKV` 成立的最重要事实基础。

`vLLM` 社区里至少已经有几类非常具体的问题：

- [Issue #37308](https://github.com/vllm-project/vllm/issues/37308)
  在 asymmetric batches 下，prefix caching 会出现严重 `HoL blocking`，小请求在共享前缀的情况下仍被大请求拖住，TTFT 回归到 `14x-147x`。
- [Issue #38194](https://github.com/vllm-project/vllm/issues/38194)
  真实高并发多轮聊天 workload 里，用户报告 prefix cache hit 明显低于其他 stack。
- [Issue #38988](https://github.com/vllm-project/vllm/issues/38988)
  某些模型上的 prefix caching 行为和收益不稳定，说明线上预期并不稳。

这三类现象合起来说明：

- prefix cache 的“逻辑命中”不是最终目标；
- 用户真正感知的是 `TTFT / P99 TTFT / tail latency`；
- prefix cache 需要被当成在线系统中的 `QoS-sensitive mechanism` 来重新审视。

## 4.3 pollution 与 observability 的缺口也是实证存在的

另外两类社区信号也很关键：

- [Issue #39321](https://github.com/vllm-project/vllm/issues/39321)
  reasoning model 的 thinking tokens 会沿着 hash chain 进入 APC；如果后续 prompt 不再携带这些 token，它们就会变成几乎不可达的缓存链条，继续和真正有价值的块竞争 eviction。
- [Issue #33434](https://github.com/vllm-project/vllm/issues/33434)
  生产环境明确需要区分 active blocks 和 prefix-cached blocks，但当前指标层不够细。
- [Issue #23083](https://github.com/vllm-project/vllm/issues/23083)
  社区直接提出了 persistent / pinned prefixes 的需求。

这说明 `QoSKV` 想做的三件事：

- 污染控制
- 热前缀保留
- APC 状态可观测

都不是拍脑袋想出来的，而是已经被真实用户需求“点名”过。

## 5. 这个方向和当前 `vLLM` 到底兼不兼容

## 5.1 兼容性高，而且是顺着现有 APC 主线往前走

我给 `QoSKV` 的兼容性评估是：**高**。

原因有三个：

- 它不需要引入新的 attention backend，也不要求改模型结构；
- 它主要修改的是 `APC` 的 cacheability、retention、admission 和 metrics 语义；
- 它和当前 `vLLM V1` 的 request / scheduler / kv_cache_manager 结构是顺向兼容的。

比较自然的改动点包括：

- `vllm/v1/request.py`
  - 标记哪些 output tokens 不应进入共享 APC 链；
- `vllm/v1/core/kv_cache_manager.py`
  - 维护 prefix class、dead/unreachable hints、soft reservation、useful-hit stats；
- `vllm/v1/core/sched/scheduler.py`
  - 对短请求增加 rescue path 或 quota，避免共享前缀场景下的无声 HoL；
- `vllm/v1/metrics/*`
  - 暴露 active vs cached、dead prefix ratio、hot prefix quota usage、cache pressure。

## 5.2 但它不能只是几个 feature request 的并集

兼容不等于自动成立。

如果实现只是：

- 加一个 pin prefix flag
- 再补几个 metrics
- 顺手调一下 LRU

那它更像一组工程 patch，而不像一篇系统论文。

`QoSKV` 想成立，必须把这些零散动作组织成一个统一论点：

> APC 需要从“cache hit optimization”升级为“latency-safe interactive runtime”。

也就是说，机制之间必须有清晰协同关系：

- `pollution control` 负责减少无效竞争；
- `retention / reservation` 负责保护关键热前缀；
- `QoS-aware scheduling` 负责防止共享前缀场景下的时延劣化；
- `observability` 负责让系统对回归可解释、可调优。

## 6. 它是否有效

## 6.1 从第一性原理看，这条线大概率有效

我对“是否有效”的判断是：**有效，而且很可能很容易测出收益**。

理由并不复杂：

- 真实 workload 的 prefix reuse 是高度偏斜的，说明“保护少数高价值前缀”本身就有收益空间；
- `FCFS / LPM / LRU` 已被论文和社区案例反复证明，不足以兼顾 fairness 与 tail latency；
- reasoning / interactive workload 里的“不可复用 token”会占据链式哈希空间，污染 eviction 决策；
- 当前 vLLM 指标体系无法区分 APC 的“有效命中”和“有害驻留”，这让线上回归很难被及时解释。

换句话说，`QoSKV` 的很多收益不是来自发明一个很神奇的新算法，而是来自**把本来互相打架的局部策略放回一个服务 QoS 目标里统一处理**。

## 6.2 但有效不等于 automatically 足够发论文

这条线最容易陷入的误区是：

- 实验上能得到不错的 `TTFT / P99` 改善；
- 但论文贡献被拆开看，每一项都像现有系统里已经出现过的技巧。

所以 `QoSKV` 的有效性要靠“闭环”来证明，而不是靠单点提升：

1. 先证明 `APC` 在交互式场景下会产生可重复的 silent QoS regression。
2. 再证明 regression 不是单一原因，而是由 `pollution + unfair scheduling + unstable retention + blindness in metrics` 共同造成。
3. 最后证明单独修其中一个点收益有限，只有把它们合成一个 runtime 才能稳定改善 `TTFT / P99 / useful-hit ratio`。

如果这三个层次都做出来，论文说服力会强很多。

## 6.3 最低限度需要哪些实验才算站住

我认为至少要有四类实验：

- `asymmetric burst`：
  - 复现共享前缀下大请求拖住小请求的 `HoL` 问题；
- `reasoning multi-turn`：
  - 证明 thinking tokens 或类似不可复用输出会形成 dead / low-utility cache；
- `shared system prompt under high concurrency`：
  - 证明热前缀保护确实能带来更稳定的 `TTFT / P99`；
- `observability-driven diagnosis`：
  - 用新增指标说明收益来自哪里，而不是只给一个总吞吐或平均时延。

如果没有这些实验，只拿一个 benchmark 上的 latency 降低来讲，论文会显得很虚。

## 7. 它是否足够支撑系统类论文

## 7.1 作为“当前版本的 QoSKV”，还不够稳到 OSDI

这是我最坦率的判断。

如果 `QoSKV` 还是当前这个宽泛表述：

- 前缀分类
- 热前缀保留
- 轻量调度
- 多加监控

那它更像一篇：

- 强工程论文
- 或者一篇 `ATC / EuroSys / SoCC` 风格的系统优化工作

但**还不够稳到 `OSDI`**。原因不是它无效，而是：

- novelty 容易被拆散；
- 很多局部点都能在相近论文里找到影子；
- thesis 还不够尖锐。

## 7.2 如果要把它提升到更像顶会的版本，题目必须更窄

我认为可行的升级方向不是“把功能再堆多”，而是把中心论点收紧成：

> `QoSKV` is a latency-safe exact APC runtime for interactive LLM serving.

围绕这个标题，论文贡献可以压成三条：

- 我们首次系统性刻画 APC 在 interactive serving 中的 silent QoS failure modes；
- 我们提出一套 latency-safe APC runtime，将 pollution control、prefix retention、QoS-aware scheduling、state observability 联合起来；
- 我们在 vLLM 上证明：仅优化 hit ratio 并不够，只有把 APC 作为 QoS-sensitive runtime 管理，才能稳定改善 `P99 TTFT` 与 tail latency。

这样它就不再像一个“policy bucket”，而更像一个有清晰 thesis 的系统问题。

## 7.3 我对它的主观打分

以当前版本来评估，我会给出下面这组分数：

- `真实问题强度`：`8.5 / 10`
- `与当前 vLLM 的兼容性`：`9 / 10`
- `实验上打出收益的概率`：`8 / 10`
- `和现有工作的重合风险`：`7.5 / 10`
- `作为当前版本冲 OSDI 的把握`：`4.5 / 10`
- `如果收紧成 latency-safe APC runtime 之后的把握`：`6.5 / 10`

这里最关键的一点是：**问题强，不代表论文 automatically 强**。`QoSKV` 当前最缺的是一个更锋利、更难被已有工作直接覆盖的 thesis。

## 8. 我的最终判断

我的最终判断是：

- `QoSKV` **有真实问题基础**，而且比 `SegmentKV` 那类方向更容易在当前 `vLLM` 上快速打出实验结果。
- 但 `QoSKV` **已经不是“谁先想到谁赢”的空白赛道**。它周围已经有 `HotPrefix`、`CachedAttention`、`Preble`、`KVFlow`、`Continuum`、`KVCache Cache in the Wild`、`Bidaw` 这类工作包围。
- 因此，`QoSKV` 若想继续成立，就不能只做“更聪明的 APC policy”，而必须做成一个更明确的 thesis：
  - **面向低延迟交互场景的 latency-safe APC runtime**
  - 核心目标是减少 `silent QoS regression`
  - 核心指标是 `P99 TTFT / tail latency / useful-hit ratio`

如果按这个版本推进，我认为它：

- 对 `vLLM` 是自然兼容的；
- 在工程上是可做的；
- 在实验上大概率是有效的；
- 但要冲 `OSDI`，还需要更强的 failure characterization 和更硬的统一机制故事。

## 9. 一页式结论

## 9.1 结论打分

- `是否重复`：**部分重复，不能按原始宽泛版本继续讲**
- `是否兼容 vLLM`：**兼容性高**
- `是否大概率有效`：**是**
- `是否已经足够支撑顶会`：**当前版本不够，收紧 thesis 后才有机会**

## 9.2 最终建议

如果你想保留 `QoSKV`，我建议不要再把它表述成“前缀缓存优化”这五个字，而是改成下面这个更窄、更有系统味道的版本：

> `QoSKV: A Latency-Safe Exact APC Runtime for Interactive LLM Serving`

也就是把问题从“缓存怎么更聪明”升级成：

- `APC` 为什么会在 interactive serving 中悄悄伤害 `TTFT / P99`
- 系统如何在不改变 exact prefix reuse 语义的前提下，让 APC 变得 `pollution-aware`、`fairness-aware`、`observable`

如果后续要继续做，我建议下一步直接写它的升级版提案，而不是继续扩散相关工作列表。
