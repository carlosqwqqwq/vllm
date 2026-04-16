# RealizeKV：把 Counterfactual Exact Hits 兑现成 Physical Hits

更新时间：2026-04-15

## 1. 严格结论

如果目标是 `CCF-A` 系统论文，我现在最推荐继续推进的，不是再发散一个新的“cache 结构”或“更灵活复用”想法，而是把当前内部主线 `FalseNegativeKV` 升格成一个更适合论文叙事的新版本：

> `RealizeKV: Realizing Counterfactual Exact Hits in LLM Serving`

更直白地说：

- `FalseNegativeKV`
  - 适合内部测量和排查。
- `RealizeKV`
  - 适合 paper-facing 叙事。
  - 它强调的是：
    - 可复用状态其实已经存在，
    - 但 runtime 没把它兑现成 hit，
    - 我们提出一套可测、可桥接、可调度的 substrate 去把它兑现出来。

## 2. 为什么要从 FalseNegativeKV 升格成 RealizeKV

`FalseNegativeKV` 这个名字有两个问题：

1. 太像 bug taxonomy。
2. reviewer 很容易误读成“修一堆 APC feature gaps”。

而 `RealizeKV` 更强，因为它把问题写成一个正向系统目标：

> 不是“哪些请求被错误地 miss 了”，而是“如何让 counterfactual exact hits 稳定地 materialize 成 physical hits”。

这会自然带出三个系统问题：

1. 如何测量 latent hit？
2. 如何最小代价地 bridge？
3. 如何在 serving runtime 里选择何时兑现、何时放弃？

## 3. 核心问题定义

### 3.1 四个基本量

我们建议把论文里的核心量固定成下面四个：

#### physical_hit_tokens

- 当前 runtime 真正读到并使用的 prefix cache token 数。

#### counterfactual_hit_tokens

- 如果忽略某个 runtime contract 限制，系统本可以命中的 token 数。
- 这里的“忽略”不是修改模型语义，而是做 counterfactual probe。

#### realization_gap

- `counterfactual_hit_tokens - physical_hit_tokens`
- 这个量越大，说明 latent reuse 越多但没有兑现。

#### bridge_cost

- 为了把 latent hit 变成可执行 hit，需要额外付出的代价。
- 典型包括：
  - boundary replay
  - selective recompute
  - sidecar state
  - route 到兼容执行池
  - 额外调度等待或 KV load

### 3.2 一句话目标

`RealizeKV` 的目标不是最大化 raw cache hit，而是：

> 最大化 `realized_gain = realized_hit_benefit - bridge_cost`

因此它不是“能复用就全复用”，而是一个有代价模型的 realization system。

## 4. 新 idea 的系统骨架

## 4.1 Counterfactual Hit Profiler

作用：

- 对每个请求测量 physical hit。
- 同时估计在 relaxed contract 下的 counterfactual hit。

要点：

- profiler 必须足够轻量，不能把线上路径变成双倍查表成本。
- 第一版可以只对部分 mismatch 类别启用：
  - `observation mismatch`
  - `backend mismatch`

为什么它重要：

- 没有 counterfactual 测量，整篇论文就会退化成 patch collection。
- 有了 counterfactual hit，问题规模就能被量化。

## 4.2 Typed Reuse Contract

核心思想：

- 把“这个请求为什么不能直接读 prefix cache”从布尔判断，提升成 typed contract。

最小维度建议如下：

- `api_contract_class`
  - `generation / prompt_logprobs / pooling / classify / embed`
- `carrier_type`
  - `tokens / embeds / multimodal`
- `backend_compatibility_class`
  - `reuse_preferred / dual_compatible / unknown`
- `false_negative_risk_class`
  - `observation_mismatch / backend_mismatch / carrier_mismatch / low`

为什么这层必要：

- 没有 typed contract，就无法统一 `prompt_logprobs`、pooling、backend incompatibility 这些看似无关的 miss。
- 有了 typed contract，才能把 scattered fixes 上升成系统 substrate。

## 4.3 Bridge Planner

Bridge Planner 是整篇论文的关键，不然 `RealizeKV` 就只是一个 profiler。

第一版只建议保留四种桥接动作：

### 1. zero-bridge

- 直接复用，什么都不补。

### 2. boundary replay

- 复用大部分 prefix，仅重算末尾一小段。
- 最适合 `prompt_logprobs` 这种 observation contract mismatch。

### 3. observation sidecar

- 缓存少量伴生观测状态，而不是再引入一整套 hidden-state cache。
- 必须严格限制为 bridge 所需最小状态。

### 4. compatible-route

- 当同一 prefix family 在不同执行池上的兑现能力不同，允许 route 到更适合 realization 的池。

第一版明确不做：

- arbitrary-position KV stitching
- semantic reuse
- 通用 hidden cache substrate
- cluster-level cache pool

## 4.4 Realization-Aware Scheduler

这层决定 `RealizeKV` 是否真像系统论文。

调度器不再只看：

- token budget
- queue order
- cache hit rate

而要额外看：

- counterfactual hit 是否足够大
- bridge 成本是否值得
- 是否会拖累 batch throughput
- 是否值得把一个 prefix family 绑定到特定 realization-friendly pool

一个最朴素但足够论文化的策略是：

> 仅当 `realized_gain > threshold` 时，才执行 bridge 或 route。

这样可以自然带出：

- latency-friendly mode
- throughput-friendly mode
- family binding / cold-start handling

## 5. 为什么这个方向还有论文空间

## 5.1 它不等于 Prompt Cache

`Prompt Cache` 的重点是显式模块与位置正确性。

`RealizeKV` 的重点是：

- 可复用状态已经存在，
- 但 runtime 由于契约不匹配，没有把它兑现成 hit。

## 5.2 它不等于 Marconi / Cake / KVFlow

这些工作主要优化：

- admission / eviction
- load vs compute
- workflow-aware prefetch

它们默认 hit 资格已经成立。

`RealizeKV` 则研究：

- hit 为什么没 materialize。

## 5.3 它不等于 EPIC / MPIC / KVLink

这些工作研究：

- 不再要求 exact prefix。
- 或者将模块、文档、多模态段在不同位置下复用。

`RealizeKV` 则坚持：

- 只研究 exact / logical-exact reuse。
- 不触碰 arbitrary-position reuse。

## 5.4 它不等于 KVShare / RelayCaching

这些工作虽然也有 selective recompute / bridge 的味道，但它们主要解决的是：

- 位置偏移
- phase transfer
- 近似一致性

我们的目标不同：

- 不是“让不一致的状态尽量可复用”，
- 而是“让已经一致或逻辑等价的状态不再被 runtime 错杀”。

## 6. 论文最推荐的 scope

如果你们真要冲 `CCF-A`，scope 一定要克制。

我最推荐的第一版 scope 是：

### 只做两类 mismatch

#### observation mismatch

- 典型例子：
  - `prompt_logprobs`
  - 部分 pooling / classify / token-level observation

#### backend mismatch

- 典型例子：
  - `batch invariance + FLASHINFER/TRITON_MLA`
  - 同一 prefix family 在不同后端路线上兑现能力不同

### 只做 exact / logical-exact reuse

- 不扩到 PIC
- 不扩到 semantic reuse
- 不扩到跨模型训练式共享

### 只做最小 bridge

- boundary replay
- small sidecar
- route / family binding

这样 scope 才够像系统论文，而不是一堆松散小技巧。

## 7. 为什么它适合 vLLM

`vLLM` 特别适合这条线，原因有三：

### 1. prefix caching 已经是主流程基础设施

- 不是外挂 feature，而是 `v1 core` 的关键路径。

### 2. 代码里已经有明确的“跳过读 prefix cache”与“禁用 APC”机制

- 这意味着系统里天然存在“可测的 realization gap”。

### 3. 本仓库里已经出现了这条线的脚手架

从当前只读代码看，已经存在：

- `false_negative_route`
- `prompt_logprobs_sidecar`
- `twin_path_policy`

这非常重要，说明：

- 你们不是从零开始想空 idea；
- 而是已经具备一个可以被论文化的 P0 substrate 雏形。

## 8. 推荐标题

最推荐的标题模板：

1. `RealizeKV: Realizing Counterfactual Exact Hits in LLM Serving`
2. `RealizeKV: Turning Counterfactual Prefix Hits into Physical Reuse in vLLM`
3. `Beyond Cache Misses: Realizing Counterfactual Exact Reuse in LLM Serving`

如果需要保留内部主线名，也可以副标题化：

4. `RealizeKV: A Runtime Substrate for False-Negative Exact Reuse in vLLM`

## 9. 当前建议

我的建议很明确：

- 对内继续沿用 `FalseNegativeKV` 做测量和工程沟通。
- 对外把论文叙事切换到 `RealizeKV`。
- 第一版就围绕：
  - `counterfactual hit`
  - `bridge`
  - `realization-aware scheduling`
  这三件事写。

这比继续发散“再来一个 cache 新结构”更有胜率。
