# FalseNegativeKV 提标方案：审稿人攻击点、贡献加厚与可投稿门槛

更新日期：2026-04-15

## 1. 执行结论

如果按现在的形态直接投稿，`FalseNegativeKV` **不够稳**。

最危险的问题不是“完全没学术性”，而是：

- 它现在更像一个很好的**问题雏形**；
- 还不像一个已经长成的 **OSDI/CCF-A 级 thesis**。

更严格地说，当前版本最大的风险是：

- 容易被看成若干 `vLLM` compatibility gap 的统一命名；
- 容易被打成 feature patch，而不是新系统问题；
- 容易被 reviewer 质疑“为什么不分别修掉，而要写成论文”。

因此，下一阶段不能继续扩写概念，而要把这条线**强行做厚**，做到下面这件事：

> 不只是指出 `false-negative reuse` 存在，而是证明它是一个普遍、可测、可统一建模、可用单一 runtime substrate 修复、并且能稳定带来端到端收益的系统瓶颈。

如果做不到这一层，就不该把它继续当主 thesis。

## 2. 先把问题说清：现在为什么还不够

### 2.1 现在这条线已经有的东西

目前我们已经有：

- 一个相对新的问题定义：
  - `logical hit` 没有变成 `physical hit`
- 一组 `vLLM` 直接证据：
  - `prompt_logprobs` 跳过 APC
  - token-level pooling 跳过 APC
  - `batch invariance` 与快后端会关闭 APC
- 一个初步的机制草图：
  - `reuse contract`
  - `bridge operators`
  - `TwinPathServe`

这说明它已经不是空想。

### 2.2 现在还缺什么

它真正缺的是**可投稿厚度**，主要缺四层：

1. 缺“问题规模证明”
   - 还没有证明 false-negative miss 在真实 workload 中足够普遍。
2. 缺“统一抽象的不可替代性”
   - 还没有证明这不是一堆 case-by-case patch。
3. 缺“至少两类非平凡机制”
   - 现在还停留在机制草图，没有形成强实例组合。
4. 缺“闭嘴级实验”
   - 没有证据说明收益足够大，也没有证据说明 generality 足够强。

换句话说：

- 现在更像“正确的问题”
- 还不像“足够强的论文”

## 3. 审稿人最可能怎么攻击

下面这些攻击点，不是理论上的，而是我认为 reviewer 真会写在 review 里的。

## 3.1 攻击一：这不就是几个 `vLLM` feature gap 的拼盘吗

典型表述会是：

- `prompt_logprobs + prefix cache` 是已知 RFC；
- batch invariance 关闭 APC 也像 backend support gap；
- 这些不就是不同 feature 的不兼容吗？

为什么危险：

- 一旦 reviewer 这样理解，`FalseNegativeKV` 就从“新问题类”退化成“几处工程缺口的统一命名”。

如何反击：

- 必须证明这些现象共享同一种根因：
  - **producer / consumer contract mismatch**
- 必须证明统一 substrate 可以同时解释并修复多类现象。

## 3.2 攻击二：为什么不能分别修？为什么要写成论文？

这是最难的一击。

典型表述会是：

- 给 `prompt_logprobs` 单独加缓存不就好了？
- 给 `FLASHINFER` 这类路径单独加 fallback 不就好了？
- 为何需要一个新的系统抽象？

如何反击：

- 要证明 scattered fixes 的问题：
  - 每来一种 API / backend / input carrier 都要堆新黑名单或新特判；
  - 不同特判之间无法统一统计、统一解释、统一调度。
- 要证明 unified substrate 的价值：
  - 统一判定 `logical hit`
  - 统一表达 `bridgeability`
  - 统一记录 `false-negative reason`
  - 统一做 `route / bridge / fallback`

## 3.3 攻击三：这是不是太依赖 `vLLM`，缺少一般性

典型表述会是：

- 你们是不是只发现了 `vLLM V1` 的一些实现空白？
- 换个 runtime 这些问题还存在吗？

如何反击：

- 论文里不能把 `vLLM` 写成问题本体，只能把它写成实例系统。
- 要把问题抽象成：
  - 任何存在 shared-state reuse、异构 API、异构 backend、异构 execution path 的 serving runtime，都可能出现 false-negative exact hits。

也就是说：

- `vLLM` 提供代码证据
- 但 thesis 必须是 runtime-general 的

## 3.4 攻击四：贡献是不是太少，只是一个 patch 加一个 scheduler

典型表述会是：

- 一边是 `prompt_logprobs` selective replay
- 一边是双池路由
- 这两件事真的构成一个统一论文吗？

如何反击：

- 必须把贡献组织成一个“分层包”：
  - 问题表征
  - 统一抽象
  - 两类机制实例化
  - 统一评价框架

缺任何一层，都容易散。

## 3.5 攻击五：收益是不是只在一个边角 workload 上成立

如果实验只在：

- 很长共享系统提示词
- 又恰好要 `prompt_logprobs`

这种窄 workload 上才有效，reviewer 会直接说：

- 这是 niche optimization，不是通用系统问题。

如何反击：

- 至少要证明：
  - 这不是单 workload 偶然成立；
  - 至少两类不同 mismatch 的收益都成立；
  - 端到端收益不只是一项指标好看。

## 4. 我们必须把贡献做厚到什么程度

我建议把这条线强行提升为一个 **四层贡献包**。

## 4.1 第一层：问题贡献

这层的目标不是修系统，而是**重新定义问题**。

论文必须明确提出并形式化下面四个概念：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`

这层的价值是：

- 把零散工程症状上升成一类 runtime pathology

如果连这层都站不住，整篇论文就不成立。

## 4.2 第二层：抽象贡献

这层是论文的核心。

必须提出一个统一抽象，而不是“这里加个缓存，那里加个 route”。

我建议抽象至少包含：

- `reuse contract`
  - producer 产出什么 state
  - consumer 需要什么 observation / capability
- `bridgeability lattice`
  - full hit
  - bridgeable miss
  - non-bridgeable miss
- `false-negative metrics`
  - 用统一指标量化损失与恢复

如果没有这一层，论文就会被看成两个 patch 的组合。

## 4.3 第三层：系统贡献

必须至少有两个**不同类型**的强实例化，而且都挂在同一 substrate 上。

我建议第一版只保留两个：

1. `prompt_logprobs` selective replay
2. `TwinPathServe` 双池路由

为什么是这两个：

- 一个代表 `observation mismatch`
- 一个代表 `backend mismatch`

它们刚好能证明：

- 这不是单 API 特例
- 也不是单 backend 特例

## 4.4 第四层：实证贡献

这一层要证明的不是“能 work”，而是：

- 问题足够普遍
- 机制足够 general
- 收益足够明显

必须回答四个问题：

1. false-negative misses 到底有多常见？
2. 其中有多少是可桥接的？
3. 恢复这些 misses 后，收益有多大？
4. 这些收益是否能跨 API / backend / workload 成立？

## 5. 论文主线该怎么改写，才能更像顶会

## 5.1 问题名可以叫 `FalseNegativeKV`

这个名字适合拿来定义问题类，因为它直观、尖锐、带判断力。

但论文题目本身，未必应该直接叫这个。

## 5.2 论文应该突出“contract-aware recovery”

比起直接写成：

- `FalseNegativeKV`

更像系统论文的写法可能是：

- `ContractKV`
- `TwinPathServe`
- `Contract-Aware Reuse Recovery`

原因很简单：

- `FalseNegativeKV` 说明“问题是什么”
- 但论文题目更应该说明“我们提出了什么系统”

## 5.3 更强的主 thesis 表述

我建议把最强版本的 thesis 写成：

> 现有 LLM serving runtime 主要把 prefix reuse 建模为“是否命中”的二值问题，却忽略了大量 exact 或逻辑等价状态虽然已经存在，但因 producer / consumer 契约不匹配而无法被消费。我们提出一种 contract-aware reuse substrate，将这类 false-negative exact hits 显式识别、分层归类，并通过 bridge operators 与 twin-path execution 恢复原本被 runtime 错误丢失的复用机会。

这个版本比“有些 cache 没用上”强很多，因为它同时包含：

- 新问题
- 新抽象
- 新系统

## 6. 最小可投稿标准

如果要把这条线当成主 thesis，我建议内部先设一个严格门槛。

## 6.1 范围门槛

下面这三件事必须同时成立：

1. 至少覆盖两类独立 mismatch：
   - `observation`
   - `backend`
2. 至少有一个统一 substrate：
   - 不是两个互不相关 patch
3. 至少有一套统一指标：
   - 能量化 false-negative misses 与 recovered hits

少一项，都不够。

## 6.2 结果门槛

我建议用下面这些内部 gate 作为止损标准。

这些不是论文里必须写成硬阈值，但我们自己必须用它们判断该不该继续。

### Gate A：问题规模

在至少两个 realistic target workloads 上，必须观察到：

- `false-negative tokens / logical-hit tokens` 不低于 `10%`

如果低于这个量级，这条线大概率太边缘。

### Gate B：恢复空间

在至少一类核心 mismatch 上，必须有明显 bridge 空间：

- `bridgeable false-negative tokens` 不能只是极少数尾部 token

如果桥接空间太小，论文会被打成“学术上成立，系统上不重要”。

### Gate C：端到端效果

至少要看到：

- `TTFT` 改善达到 `20%+`
- 吞吐不明显下降，最好还有 `10%+` 的提升

如果只是 `5%-8%` 量级的小优化，很难支撑主 thesis。

### Gate D：通用性

收益必须跨至少两个维度成立：

- 两类 mismatch
- 两类 workload
- 至少两个 backend 或执行模式

如果只在一个点上成立，论文会被打成 niche optimization。

## 7. 强实验应该怎么设计

如果真按顶会标准推进，实验必须分成四组，不能只跑端到端。

## 7.1 组一：问题普遍性测量

目标：

- 证明 false-negative miss 是可重复、可量化、不是偶发现象

至少要测：

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `false_negative_reason`

输出应该是：

- API 维度
- backend 维度
- workload 维度

的矩阵，而不是单个数字。

## 7.2 组二：机制可恢复性

目标：

- 证明这些 misses 不是“看起来存在，但实际上无法恢复”

至少要测：

- bridge 成功率
- bridge 额外开销
- recovered hit length

## 7.3 组三：端到端系统收益

至少要汇报：

- `TTFT`
- `throughput`
- `TPOT`
- `p95/p99 latency`

并区分：

- no-share baseline
- vanilla APC
- APC + ad hoc fix
- `FalseNegativeKV / TwinPathServe`

## 7.4 组四：消融实验

必须拆开证明：

- 只有 instrumentation 没有 bridge，不够
- 只有 `prompt_logprobs` bridge，不够
- 只有 twin-pool route，不够
- 二者叠加才体现统一 thesis 的价值

## 8. 现在不该再碰的危险方向

如果想保持 novelty，下面这些方向不要重新滑回去。

## 8.1 不要把论文写成 PIC

不能说：

- 我们让 prefix reuse 更灵活
- 我们支持 non-prefix reuse

这会直接靠近 `EPIC / MPIC / MEPIC`。

## 8.2 不要把论文写成 hidden cache

不能说：

- 我们为 observations 维护另一类 hidden cache

这会靠近 `Apt-Serve / HCache`。

## 8.3 不要把论文写成 workflow-aware retention

不能把主线改成：

- tool 调用回来如何继续复用旧 KV

这会靠近 `InferCept / Continuum / KVFlow`。

## 8.4 不要把论文写成单功能补丁

尤其不要写成：

- 首次支持 `prompt_logprobs + APC`

这在 `vLLM` 文档和 RFC 语境里太像 feature gap。

## 9. 最合理的推进顺序

如果要按更高标准做，我建议顺序不要乱。

## 9.1 P0：先测，不先修

先做：

- `false-negative instrumentation spec`

把问题从“概念成立”推进到“规模成立”。

## 9.2 P1：只做一个最强 bridge

优先做：

- `prompt_logprobs` selective replay

原因：

- 代码边界清晰
- correctness 最好定义
- 也最容易形成第一张好看的收益图

## 9.3 P2：再做一个不同类型的机制

接着做：

- `TwinPathServe`

原因：

- 它代表的是另一类 mismatch
- 能把论文从 feature patch 拉回系统论文

## 9.4 P3：只有过了 gate，才继续写论文叙事

如果：

- 问题规模不大
- recoverable fraction 不高
- 端到端收益不明显

那就应该及时止损，不要继续堆文档。

## 10. 我的最终建议

如果你要我按更高标准继续推进，我的建议很明确：

1. 保留 `FalseNegativeKV`，但只把它当**问题名**。
2. 后续论文主 artifact 应该突出：
   - `reuse contract`
   - `bridgeability`
   - `TwinPathServe`
3. 先把“问题规模证明”做出来，再谈主 thesis 稳不稳。

更直接的说法是：

> 现在这条线还没有强到可以自信投稿。
> 但它有机会被做强。
> 能不能做成，取决于我们能否把它从“好问题”推进成“有厚度的贡献包”。

## 11. 接下来最应该产出的文档

按优先级排序，我建议下一步依次写：

1. `false_negative instrumentation spec`
2. `prompt_logprobs selective replay design`
3. `TwinPathServe minimal architecture`
4. `paper evaluation matrix`

如果这四份东西写不出来，或者写出来后发现数据很弱，这条线就不该继续当主线。

## 12. 关键参考

- DistServe: <https://arxiv.org/abs/2401.09670>
- Sarathi-Serve: <https://arxiv.org/abs/2403.02310>
- Preble: <https://arxiv.org/abs/2407.00023>
- Mooncake: <https://arxiv.org/abs/2407.00079>
- EPIC: <https://arxiv.org/abs/2410.15332>
- MPIC: <https://arxiv.org/abs/2502.01960>
- HCache: <https://arxiv.org/abs/2410.05004>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- KVFlow: <https://arxiv.org/abs/2507.07400>
- TokenLake: <https://arxiv.org/abs/2508.17219>
- ShadowServe: <https://arxiv.org/abs/2509.16857>
- Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning: <https://arxiv.org/abs/2601.20326>
- PrefillShare: <https://arxiv.org/abs/2602.12029>
- SUN: <https://arxiv.org/abs/2603.02599>
- Libra: <https://www.usenix.org/conference/nsdi26/presentation/ruan-libra>
- vLLM V1 Guide: <https://vllm.website.cncfstack.com/usage/v1_guide.html#prompt-logprobs-with-prefix-caching>
