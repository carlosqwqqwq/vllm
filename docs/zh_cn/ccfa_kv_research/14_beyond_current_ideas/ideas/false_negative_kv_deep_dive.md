# FalseNegativeKV 深度复审：重复性、可行性与论文空间评估

更新日期：2026-04-15

## 1. 严格结论

先给严格结论，不绕弯：

- 截至 `2026-04-15`，我**不能诚实地说“确保没人做过”**。
- 但基于这轮公开论文与 `vLLM` 实现复核，我**没有找到一篇与 `FalseNegativeKV` 完全同构的公开系统论文**。
- 目前最接近它的工作，分别落在：
  - `prefix hit` 之后的调度 / 执行优化；
  - `exact prefix` 之外的更灵活复用；
  - hidden state / activation / workflow continuity 这类“换 state form 或延长 state lifetime”的方向。
- 只要我们把 thesis 收紧为：
  - **研究 exact 或逻辑等价 state 已经存在时，为什么 runtime 仍把它当作 miss；**
  - **研究 producer / consumer 执行契约不匹配导致的 false-negative reuse；**
  - **研究如何用轻量 bridge 恢复这些被错误丢掉的命中；**
  那么它与最新论文的边界仍然是清楚的。

更直接一点说：

> `FalseNegativeKV` 不是“再做一种 cache policy”，也不是“再做一种更灵活的 prefix match”。
> 它更像是在问：**为什么很多已经存在的可复用状态，在真实 serving runtime 里根本没有被承认为 hit？**

这是当前这条线最有价值、也最有新意的地方。

## 2. 我们到底在研究什么

### 2.1 精确定义

我建议把 `FalseNegativeKV` 定义成下面这个问题：

> 在 `vLLM` 这类 serving runtime 中，某些请求已经具有可复用的前缀状态，或者这些状态在逻辑上已经存在于系统中；但由于 API 语义、观测需求、backend 能力、batch invariance、输入载体等执行契约不匹配，runtime 不会把它们识别成可执行命中，最终触发整体重算。

这个定义里有四个关键点：

1. 研究对象是 **exact 或逻辑等价的 state**。
2. 问题发生在 **runtime contract / capability boundary**，不是 hash 不准。
3. 目标不是更灵活的近似复用，而是**恢复本来就应该成立的 reuse**。
4. 第一版机制应该是 **bridge + route + metric**，而不是大规模重写 kernel。

### 2.2 它不研究什么

为了避免和近邻工作撞题，下面这些明确**不应该**成为论文主命题：

- 不做 `position-independent caching`
- 不做 `chunk-at-arbitrary-position` 的 modular reuse
- 不做 approximate / semantic reuse
- 不做 hidden-state-first 或 activation-first cache
- 不做 cluster-level memory pool / cache pool
- 不做跨模型共享 prefill module 的训练式方案
- 不把论文写成“prompt logprobs 终于支持 APC”的单点 patch

### 2.3 它与旧三条线的关系

`FalseNegativeKV` 不是一个完全割裂的新想法，它其实是对旧三条能力缺口线的统一上升：

- `ObservationKV`
  - 对应 `API / observation contract mismatch`
- `InvariantKV`
  - 对应 `backend / execution contract mismatch`
- `EligibilityKV`
  - 对应 `cacheability decision` 仍然过于保守

单独做这三条，容易分别像：

- feature gap
- backend engineering fix
- eligibility protocol cleanup

而把三者统一成 **false-negative reuse**，就更像一个系统论文问题类。

## 3. 最近邻论文判重矩阵

下面只列与这条线最接近、且会真实影响 novelty 判断的工作。

### 3.1 调度与解耦：它们默认“命中语义已经成立”

代表工作：

- `DistServe`（OSDI 2024）
- `Sarathi-Serve`（OSDI 2024）
- `Preble`（2024）
- `Mooncake`（2024/2025）
- `Libra`（NSDI 2026）
- `TokenLake`（2025）
- `ShadowServe`（2025）

它们做的核心事情分别是：

- `DistServe`
  - prefill / decode 解耦与资源共优化
- `Sarathi-Serve`
  - chunked prefill + stall-free scheduling
- `Preble`
  - 分布式 prefix-aware routing
- `Mooncake`
  - KV-centric disaggregation 与 SLO-aware scheduler
- `Libra`
  - micro-request partition 与两级调度
- `TokenLake`
  - segment-level prefix cache pool 与 declarative cache interface
- `ShadowServe`
  - 分布式 prefix caching 中的 SmartNIC fetch path

与 `FalseNegativeKV` 的边界：

- 这些工作研究的是：
  - hit 已经成立之后，如何更好地调度、搬运、池化、抓取或跨机复用。
- `FalseNegativeKV` 研究的是：
  - **为什么很多逻辑上本应成立的 hit，根本没有 materialize。**

这里最需要小心的是 `TokenLake`。

`TokenLake` 的 `declarative cache interface` 容易和我们的 `reuse contract` 听起来相像，但两者不一样：

- `TokenLake`
  - 面向 cluster-level prefix cache pool
  - 目标是 segment-level pooling、负载均衡、去重、碎片整理
- `FalseNegativeKV`
  - 面向 engine-local producer / consumer contract
  - 目标是识别和修复 **false-negative exact hit**

因此：

- **不要把我们的卖点写成“提出一个新的 declarative cache interface”**
- 应该写成：
  - **提出一套识别与恢复 false-negative exact hits 的 runtime contract**

### 3.2 shared-prefix execution：它们优化的是“hit 之后如何更快”

代表工作：

- `Hydragen`（2024）
- `FastTree`（MLSys 2025）
- `PAT`（2025/2026）

它们的共同点：

- 已经承认存在 shared prefix
- 重点优化 attention / decode 执行
- 关注 kernel、访存或 tree-structured execution

与 `FalseNegativeKV` 的边界：

- 它们回答：
  - “共享前缀被识别之后，attention 怎么更快？”
- 我们回答：
  - “为什么很多共享前缀在 runtime 上根本没进入可执行共享状态？”

所以：

- 如果我们最后把系统做成 prefix-aware kernel，那会直接滑到它们的空间；
- 但如果我们做的是 **false-negative detection + bridge + twin-path routing**，边界仍然成立。

### 3.3 PIC / MPIC / MEPIC：它们已经占住了“比 exact prefix 更灵活”的主线

代表工作：

- `EPIC`（ICML 2025）
- `MPIC`（2025）
- `MEPIC`（2025）

它们做的事情非常明确：

- 现有 context caching 依赖 exact prefix match
- few-shot / RAG / multimodal 场景中，exact prefix 太脆弱
- 因此应该做 position-independent reuse

与 `FalseNegativeKV` 的边界：

- 它们研究：
  - **prefix 不完全一样时，如何仍然复用**
- 我们研究：
  - **prefix 已经等价甚至 state 已经存在时，为什么 consumer 还是不能用**

这组工作对我们的最大约束是：

- 论文不能写成“我们要让 prefix caching 更灵活”
- 不能把 bridge 说成 arbitrary-position reuse
- 不能把 selective replay 说成 PIC 的轻量版本

必须坚持：

- **只研究 exact / logical-exact false-negative misses**

### 3.4 hidden cache / state restoration：它们已经占住了“换 state medium”这条线

代表工作：

- `HCache`（EuroSys 2025）
- `Apt-Serve`（2025）
- `Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning`（ICLR 2026）

它们分别做了：

- `HCache`
  - 从 intermediate activations 恢复状态
- `Apt-Serve`
  - KV cache + hidden cache 的 hybrid cache
- `Beyond Speedup`
  - 把 KV cache 当 lightweight representation，用于采样与推理

与 `FalseNegativeKV` 的边界：

- 它们研究：
  - **换一种 state form**，或者让内部状态承担新的语义角色
- 我们研究：
  - **不先换 state medium，先修复“已有 state 为什么没被用上”**

这意味着：

- 如果我们的 sidecar 最终退化成 hidden cache，就会明显靠近 `Apt-Serve`
- 如果我们的论文卖点变成“KV 还能拿去做别的任务”，就会靠近 `Beyond Speedup`

因此第一版必须克制：

- sidecar 只能是 **bridge 所需的最小伴生状态**
- 不能把它扩成另一套 hidden-state cache 系统

### 3.5 workflow continuity：它们保留 state，但研究的是 agent / tool pause

代表工作：

- `InferCept`（2024）
- `KVFlow`（2025）
- `Continuum`（2025/2026）

它们研究的核心问题是：

- tool call / agent step / multi-turn pause 之后，如何保留或调度 KV
- 如何避免 agent workflow 导致的 eviction / recompute

与 `FalseNegativeKV` 的边界：

- 它们研究：
  - **state lifetime 和 workflow-aware retention**
- 我们研究：
  - **state compatibility 和 consumer-side usability**

这是相邻但不重复的一线。

不过如果我们把故事改写成：

- “tool 调用回来之后还想继续用之前的 KV”

那就会很快靠近 `InferCept / Continuum / KVFlow`。

### 3.6 跨模型共享：它们靠模块拆分或训练建立兼容性

代表工作：

- `PrefillShare`（2026）
- `SUN`（2026）

它们做的事情是：

- 通过模型因式分解与训练，让不同模型共享 prefill 或 decode 模块
- 实现跨模型的 KV / decode 复用

与 `FalseNegativeKV` 的边界：

- 它们通过 **重新设计模型模块边界** 来制造兼容性
- 我们研究的是：
  - **在不改模型或少改模型前提下，修复 runtime 本来就存在的兼容性缺口**

因此我们必须坚持：

- 第一版是 runtime thesis，不是训练 thesis

## 4. 为什么我目前认为没有“完全重复”的公开论文

### 4.1 这轮检索后最重要的判断

我目前没有找到一篇公开论文同时满足下面四点：

1. 问题定义是 **exact 或逻辑等价 state 已存在但未被认成 hit**
2. 根因定义是 **producer / consumer 执行契约不匹配**
3. 机制核心是 **bridge operators + false-negative metrics + runtime routing**
4. 落地目标是 **`vLLM` 这类通用 serving runtime**

换句话说：

- 最近邻很多；
- 但“完全同构”的公开系统论文，我目前没有检到。

### 4.2 目前最接近的公开证据，其实不是论文而是 `vLLM` 自己

`vLLM` 官方 V1 文档已经明确承认：

- `Prompt Logprobs with Prefix Caching` 仍然是 `Planned`
- 对需要 `prompt logprobs` 的请求，engine 会忽略 prefix cache 并重算整段 prompt

这很关键，因为它说明：

- 这是一个**真实存在、公开承认的 runtime gap**
- 但它目前仍然停留在 feature / RFC 层，而不是一类被正式系统化的问题

这恰好给 `FalseNegativeKV` 留出空间：

- 它不是发明一个没人见过的 bug
- 它是把多个已知孤立 gap 统一成一个系统问题类

## 5. `vLLM` 里的直接证据

下面这些不是猜测，而是当前仓库里的直接代码证据。

### 5.1 `prompt_logprobs` 默认直接跳过 prefix cache

证据：

- [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)
- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:188)
- 官方 V1 文档：
  - <https://vllm.website.cncfstack.com/usage/v1_guide.html#prompt-logprobs-with-prefix-caching>

含义：

- 请求只要需要 `prompt_logprobs`，`skip_reading_prefix_cache` 就会被置为真；
- `KVCacheManager` 看到这个标记后，连 hit 查找都不做，直接返回 `0` cached tokens；
- 官方文档也明确写了：为了得到 prompt logprobs，会忽略 prefix cache 并整段重算。

这是最标准的 **false-negative miss**：

- 逻辑上共享前缀明明存在；
- 但 consumer 需要额外 observation；
- runtime 因为缺乏 bridge，只能当成 miss。

### 5.2 `token_embed / token_classify` 默认跳过 APC

证据：

- [pooling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/pooling_params.py:130)
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:242)

含义：

- `token_embed` 和 `token_classify` 默认把 `skip_reading_prefix_cache` 置为真；
- 这说明 token-level observation tasks 已被当作“不能直接消费已有 prefix state”。

这是第二类 false-negative：

- 状态不是没有；
- 而是 task contract 不兼容。

### 5.3 pooling 对 partial prefill 的支持是不对称的

证据：

- [methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:43)
- [methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:67)
- [tokwise/methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/tokwise/methods.py:63)

含义：

- `CLS` / `MEAN` 明确不支持 partial prefill；
- tokenwise pooling 已经有 chunked hidden state accumulation 的逻辑。

这说明系统里已经存在：

- 某些 observation path 需要额外 side state；
- 但这些 side state 还是 per-request、ephemeral 的；
- 还没有被提升为 cross-request reusable object。

### 5.4 `FLASHINFER / TRITON_MLA + batch invariance` 会直接关闭 prefix caching

证据：

- [attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/attention.py:319)
- [mla_attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/mla_attention.py:375)

含义：

- 某些快后端与 `VLLM_BATCH_INVARIANT` 组合下，系统直接整体关闭 prefix caching；
- 这不是因为 prefix 不共享，而是因为当前 backend contract 还不支持。

这是第三类 false-negative：

- `backend fast-path` 和 `cache-friendly path` 之间缺乏兼容桥

### 5.5 `prompt_embeds` 已被纳入 block hash，但 token carrier 仍可能不兼容

证据：

- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)
- [gpu_input_batch.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_input_batch.py:68)

含义：

- `prompt_embeds` 已经会进入 block hash；
- 但某些路径需要 token ids，而 `prompt_embeds` 输入下 token id 可能未知。

这说明还有一类更深的 contract mismatch：

- **input carrier mismatch**

它是很有潜力的二期方向，但不建议作为第一版主贡献。

## 6. 这条线是否足够支撑论文

### 6.1 可以，但前提很苛刻

我认为它**可以支撑论文**，但前提是同时满足下面三点：

1. 不能只修 `prompt_logprobs + APC`
2. 不能只做配置 / 黑名单清理
3. 必须展示一个统一抽象，能覆盖至少两类以上 false-negative misses

也就是说，论文至少要同时覆盖：

- 一类 `observation mismatch`
- 一类 `backend mismatch`

这样它才像一个系统命题，而不是两个工程 patch。

### 6.2 我认为可防守的最小论文骨架

最小可防守骨架应当是：

1. **问题定义**
   - `logical hit`
   - `physical hit`
   - `false-negative hit`
2. **统一抽象**
   - `reuse contract`
   - `bridge operator`
3. **具体系统**
   - `TwinPathServe`
4. **至少两类实例化**
   - `prompt_logprobs` selective replay
   - batch-invariant fast-path twin-pool routing
5. **统一评测**
   - recovered hit ratio
   - TTFT / throughput / TPOT 改善

如果做不到这五步，这条线就会掉回 feature gap。

## 7. 它是否会有效、是否会有明显优化

### 7.1 从第一性原理上，答案是“有机会，而且很合理”

因为 false-negative miss 的本质是：

- 系统重算了一段**本来就已经有的前缀工作**

因此只要 bridge 成本明显低于 full recompute，理论上就会有收益。

### 7.2 我最看好的两个收益点

#### A. `prompt_logprobs` 的 selective replay

当前基线：

- 需要 `prompt_logprobs`
- 整段 prompt 重算

如果做 bridge：

- 读取已有 prefix KV
- 只 replay 生成 observation 所必需的尾部或最小范围

那么收益会很直接：

- 长共享 prompt 场景下，`TTFT` 会明显下降
- prefill 计算量会显著减少

这个点最大的优点是：

- 系统边界清晰
- correctness 容易定义
- 比较容易在 `vLLM` 内做出第一版

#### B. batch-invariant fast path 的 twin-pool route

当前基线：

- 某些 backend + batch invariance 组合下直接关闭 APC

如果做 `TwinPathServe`：

- `reuse-optimized pool`
  - 保证 cache-friendly contract
- `throughput-optimized pool`
  - 保留 fast backend

再由 scheduler 按 false-negative 风险路由：

- 共享前缀强、命中价值高的请求走 reuse pool
- 共享价值低或重吞吐请求走 throughput pool

这个点的价值更像系统论文：

- 它不是单 feature patch；
- 它是在承认 backend contract 不同的前提下，用 runtime 结构化解法恢复 hit。

### 7.3 哪些场景下收益会弱

必须诚实说，下面几种场景收益会弱：

- 工作负载本身几乎没有共享前缀
- 请求几乎不需要 `prompt_logprobs` / pooling / token tasks
- twin-pool 导致严重负载失衡
- bridge 成本接近整段重算

所以这条线不是“所有 workload 都普遍大增益”，而是：

- **在共享前缀显著、但 runtime contract 让命中失效的 workload 上，收益有望非常明显。**

## 8. 它是否能在 `vLLM` 中实现

### 8.1 结论：能，而且第一版不需要大改 kernel

我对这件事的判断是：

- **P0 可实现性：高**
- **P1 系统成型可实现性：中高**
- **P2 深化到更广泛 bridge 的可实现性：中**

### 8.2 第一版最现实的实现顺序

#### P0: instrumentation only

先不修复，只加统计：

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `false_negative_reason`
- `bridge_candidate_reason`

把 false-negative miss 从“猜测”变成“可测”。

#### P1: 先做最容易闭环的 bridge

优先级建议：

1. `prompt_logprobs`
2. `batch invariance / backend route`
3. `token_embed / token_classify`

原因：

- `prompt_logprobs`
  - 最容易证明“本该命中却没命中”
- `batch invariance`
  - 最容易体现系统价值
- token-level pooling
  - 能提供额外 generality，但当前 `vLLM` support surface 还不够整齐

#### P2: 再考虑 carrier mismatch

例如：

- `prompt_embeds`
- multimodal carrier

这类方向更前沿，但也更容易把论文带到另一条拥挤主线。

## 9. 它是否“领先业界最新论文”

### 9.1 现在能说的，只有“问题定义层面有机会领先”

截至这轮调研，我认为：

- 在**问题定义**上，`FalseNegativeKV` 比继续做 scheduling / disaggregation / PIC / hybrid cache 更有新意；
- 但在**系统结果**上，现在还不能说领先，因为还没有实验。

### 9.2 想让它看起来像 OSDI，而不是 feature patch，必须做到三件事

1. 证明这是一个**普遍存在**的问题，而不只是 `prompt_logprobs` 的特例
2. 证明统一抽象比 scattered fixes 更好
3. 证明它和最新工作是互补而不是重命名

如果能做到，审稿叙事会是：

- 过去两年大家都在研究：
  - 怎么让 hit 更多
  - 怎么让 hit 后更快
  - 怎么把 hit 搬到别的机器或别的存储层
- 我们研究的是：
  - **为什么很多 hit 根本没有被 runtime 承认**

这个角度是成立的。

## 10. 我的最终判断

截至 `2026-04-15`，我给这条线的判断如下：

| 维度 | 判断 | 说明 |
| --- | --- | --- |
| 与公开论文完全重复风险 | 中 | 最近邻很多，但未检到完全同构工作 |
| 新颖性 | 中高到高 | 前提是坚持 `false-negative exact hit` 这一定义 |
| `vLLM` 落地性 | 高 | P0/P1 都能直接落在现有代码边界上 |
| 预期效果 | 中高 | 在共享前缀强、contract mismatch 明显的 workload 上最强 |
| 工程风险 | 中 | twin-pool 和 selective replay 都需要认真控复杂度 |
| OSDI/CCF-A 潜力 | 中高 | 取决于是否把多个孤立 gap 统一成一个清晰 thesis |

我最真实的结论是：

> `FalseNegativeKV` 值得继续做，而且比我们前面很多方向更像一个新的系统问题类。
> 但它现在还不是“已经稳了的论文题目”。
> 它要真正站住，需要把单点 feature gap 提升为统一的 runtime contract thesis。

## 11. 接下来最应该做什么

下一步我建议非常明确：

1. 写 `false-negative instrumentation spec`
   - 先把问题测出来
2. 选两个最强实例化
   - `prompt_logprobs`
   - `batch-invariant fast-path`
3. 画出 `TwinPathServe` 的最小系统图
   - 不要先做大而全
4. 提前设计判重防线
   - 明确不做 PIC
   - 不做 hidden cache
   - 不做 cross-model factorization

## 12. 关键参考

- DistServe: <https://arxiv.org/abs/2401.09670>
- Sarathi-Serve: <https://arxiv.org/abs/2403.02310>
- Preble: <https://arxiv.org/abs/2407.00023>
- Mooncake: <https://arxiv.org/abs/2407.00079>
- Hydragen: <https://arxiv.org/abs/2402.05099>
- EPIC: <https://arxiv.org/abs/2410.15332>
- HCache: <https://arxiv.org/abs/2410.05004>
- MPIC: <https://arxiv.org/abs/2502.01960>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- MorphServe: <https://arxiv.org/abs/2506.02006>
- KVFlow: <https://arxiv.org/abs/2507.07400>
- TokenLake: <https://arxiv.org/abs/2508.17219>
- ShadowServe: <https://arxiv.org/abs/2509.16857>
- FastTree: <https://proceedings.mlsys.org/paper_files/paper/2025/hash/96894468eb44631a32d7ebd56f9892c7-Abstract-Conference.html>
- PAT: <https://arxiv.org/abs/2511.22333>
- Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning: <https://arxiv.org/abs/2601.20326>
- PrefillShare: <https://arxiv.org/abs/2602.12029>
- SUN: <https://arxiv.org/abs/2603.02599>
- Libra: <https://www.usenix.org/conference/nsdi26/presentation/ruan-libra>
- NSDI 2026 technical sessions: <https://www.usenix.org/conference/nsdi26/technical-sessions>
- vLLM V1 Guide: <https://vllm.website.cncfstack.com/usage/v1_guide.html#prompt-logprobs-with-prefix-caching>
