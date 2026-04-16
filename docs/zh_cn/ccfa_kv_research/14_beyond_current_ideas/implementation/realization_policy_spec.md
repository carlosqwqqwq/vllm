# FalseNegativeKV Realization Policy 规格

更新时间：2026-04-15

## 1. 这份文档的作用

这份文档只负责回答一个问题：

> 即使某次请求存在 `counterfactual exact hit`，runtime 什么时候应该真的去兑现它，什么时候应该放弃？

它不负责：

- 重新定义 `reuse contract`
- 重新定义 `TwinPathServe` 组件
- 承诺某个 online cost model 已经实现

从现在开始，这份文档的固定定位是：

- `reuse_contract_spec.md` 的下一层
- `TwinPathServe` 真正从 “route + bridge” 抬升到 “realization-aware runtime” 的最小决策规格

## 2. 为什么必须有 `realization policy`

## 2.1 恢复 `logical hit` 不等于恢复收益

当前主线最容易犯的错误，是把论文写成：

- 只要恢复了更多 `physical_hit_tokens`，系统就成立

这不够。

因为真实 runtime 里，bridge 会带来额外代价，例如：

- replay 额外 token
- sidecar 额外状态
- route 到 `reuse_optimized_pool` 的潜在排队
- fallback 带来的调度扰动

因此必须固定一个判断：

> `FalseNegativeKV / TwinPathServe` 的目标不是最大化 raw recovered hit，而是在受控代价下最大化 realized gain。

## 2.2 没有这一层，论文仍然像 compatibility patch

如果没有 `realization policy`，reviewer 很容易说：

- 你们只是做了两种 recovery
- 为什么不永远开着
- 为什么不见到 logical hit 就强制桥接

所以必须显式写出：

- 什么情况下值得 bridge
- 什么情况下不值得
- 什么时候 route 到 reuse pool
- 什么时候宁可 full prefill

## 3. 一句话定义

`realization policy` 固定定义为：

> 在 `reuse contract` 已经判定存在 `bridgeable miss` 的前提下，负责比较 `bridge cost` 与 `realized gain`，并输出最终 `realization_mode` 的运行时决策层。

这个定义里的三个关键词锁死为：

- `bridgeable miss`
- `bridge cost`
- `realized gain`

## 4. 固定术语

从现在开始，这一层统一使用下面这些术语：

- `counterfactual_hit_tokens`
- `recovered_physical_hit_tokens`
- `bridge_cost`
- `realized_gain`
- `realization_efficiency`
- `realization_mode`
- `fallback_mode`

对应含义如下。

### 4.1 `counterfactual_hit_tokens`

- 在 relaxed contract 条件下，本可以命中的 token 数。

### 4.2 `recovered_physical_hit_tokens`

- 通过当前 bridge / route 实际恢复出来的 `physical_hit_tokens`。

### 4.3 `bridge_cost`

- 为恢复这部分 hit 额外付出的代价。

### 4.4 `realized_gain`

- 恢复收益减去桥接代价后的净收益。

### 4.5 `realization_efficiency`

- `recovered_physical_hit_tokens / counterfactual_hit_tokens`

它衡量的是：

- 这次 bridge 到底把多少 latent hit 真正兑现了出来。

## 5. 固定决策目标

P1 的目标固定不是最优控制，而是下面这个保守版本：

> 只在 `bridge_cost` 足够低、`realization_efficiency` 足够高、且不会明显拖垮 throughput 的情况下，才执行 bridge 或 route。

这意味着：

- P1 不追求所有 `logical hit` 都被兑现
- P1 追求“高把握、可解释、可复现”的兑现

## 6. 固定输入

`realization policy` P1 只允许读取下面这些输入。

## 6.1 合同层输入

来自 `reuse contract` 的固定输入：

- `contract_status`
- `contract_violation_reason`
- `bridgeability_class`
- `selected_bridge_operator`
- `consumer_requirement_class`

## 6.2 测量层输入

来自 instrumentation / counterfactual probe 的固定输入：

- `physical_hit_tokens`
- `counterfactual_hit_tokens`
- `false_negative_tokens`

## 6.3 执行层输入

来自 runtime 的固定输入：

- `pool_load_class`
- `route_fallback_risk`
- `replay_token_budget`
- `sidecar_ready`

P1 不允许再加新的一级主输入。

## 7. 固定输出

P1 固定只输出下面四类结果。

## 7.1 `realization_mode`

固定枚举：

- `direct_reuse`
- `reuse_with_bridge`
- `reuse_with_route`
- `measure_only`
- `fallback_full_prefill`

## 7.2 `fallback_mode`

固定枚举：

- `none`
- `bridge_not_worth_it`
- `bridge_not_ready`
- `reuse_pool_unavailable`
- `safety_fallback`

## 7.3 `realization_efficiency`

这是运行后结果指标，不是先验判定。

P1 必须在 trace 和聚合层都支持输出它。

## 7.4 `bridge_cost_proxy`

P1 允许用近似代理，而不是精确统一成本函数。

固定允许的代理有：

- `replay_tokens`
- `sidecar_bytes`
- `route_fallback_count`
- `reuse_pool_queue_delay_proxy`

## 8. `bridge cost` 的固定近似

P1 不强求一个复杂精确 cost model，但必须先固定最小近似。

## 8.1 replay 类 bridge

对 `prompt_logprobs_selective_replay`，`bridge_cost` 固定近似为：

- `replay_tokens`

必要时可补充：

- `replay_tokens / prompt_tokens`

## 8.2 route 类 bridge

对 `twin_pool_route`，`bridge_cost` 固定近似为：

- `reuse_pool_queue_delay_proxy`
- `route_fallback_count`

说明：

- route 类 bridge 的成本更多体现在路径和负载，而不是额外 token 计算。

## 9. P1 中的固定决策规则

为了保证可执行性，P1 先把规则写死，不做复杂学习策略。

## 9.1 direct reuse

触发条件：

- `contract_status = satisfied`

固定输出：

- `realization_mode = direct_reuse`
- `fallback_mode = none`

## 9.2 observation bridge

触发条件：

- `bridgeability_class = bridgeable`
- `selected_bridge_operator = prompt_logprobs_selective_replay`
- `sidecar_ready = true`
- `replay_token_budget` 满足

固定输出：

- `realization_mode = reuse_with_bridge`
- `fallback_mode = none`

否则：

- `realization_mode = fallback_full_prefill`
- `fallback_mode = bridge_not_ready` 或 `bridge_not_worth_it`

## 9.3 backend route

触发条件：

- `bridgeability_class = bridgeable`
- `selected_bridge_operator = twin_pool_route`
- `pool_load_class != overloaded`

固定输出：

- `realization_mode = reuse_with_route`
- `fallback_mode = none`

否则：

- `realization_mode = fallback_full_prefill`
- `fallback_mode = reuse_pool_unavailable`

## 9.4 measure-only

触发条件：

- `bridgeability_class = defer_measure_only`

固定输出：

- `realization_mode = measure_only`
- `fallback_mode = none`

适用对象：

- `carrier_mismatch`
- 当前阶段未恢复的其它 mismatch

## 10. 与 `TwinPathServe` 的关系

为了避免职责混乱，这里明确锁定：

- `reuse contract`
  - 判定“是否可直接消费，是否属于 bridgeable miss”
- `realization policy`
  - 判定“是否值得 bridge / route”
- `TwinPathServe`
  - 真正执行 route、bridge 与 fallback

换句话说：

- `reuse contract` 是方法层语义
- `realization policy` 是方法层决策
- `TwinPathServe` 是 runtime 落地

## 11. 与评测矩阵的关系

从现在开始，评测层必须至少支持下面两个 realization 指标：

- `realization_efficiency`
- `bridge_cost_per_realized_token`

固定定义：

- `realization_efficiency = recovered_physical_hit_tokens / counterfactual_hit_tokens`
- `bridge_cost_per_realized_token`
  - replay 型场景用 `replay_tokens / recovered_physical_hit_tokens`
  - route 型场景用 route 代理成本 / `recovered_physical_hit_tokens`

如果后续论文里只有：

- `TTFT`
- throughput

而没有 realization 指标，那么这条线就还没有真正从 compatibility fix 抬升成 runtime thesis。

## 12. 当前明确不做的事

P1 里不做：

- 学习式 controller
- 全局最优调度器
- cluster-level route optimizer
- 多桥同时搜索
- 对每个请求做高代价在线 counterfactual 模拟

P1 只需要一个：

- 保守、可解释、规则化的 realization policy

## 13. 验收标准

这份规格成立的最低条件固定为：

1. `prompt_logprobs` bridge 和 backend route 都能在同一 policy 语言下解释
2. 每个 `realization_mode` 都能映射到明确 runtime 行为
3. 每种 fallback 都能被 trace 出来
4. 评测矩阵中正式纳入 `realization_efficiency` 和 `bridge_cost` 代理指标

如果这四条不成立，当前主线仍然只能叫“恢复型 patch”，还不能叫“realization-aware runtime”。

## 14. 一句话结论

`realization policy` 从现在开始被固定为：

> `FalseNegativeKV / TwinPathServe` 当前主线的决策层。
> 它负责在 `reuse contract` 已经识别出 `bridgeable miss` 后，决定这次 counterfactual exact hit 是否值得被兑现，以及该以哪种模式兑现。
