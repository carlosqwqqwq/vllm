# FalseNegativeKV Reuse Contract 规格

更新时间：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

> 把 `FalseNegativeKV / TwinPathServe` 当前主线的“统一方法层”正式写清楚。

它不负责：

- 重新发散 thesis
- 重新排重
- 替代 `false_negative_instrumentation_spec.md`
- 替代 `twin_path_serve_minimal_architecture.md`
- 承诺某个 bridge 已经实现完成

从现在开始，这份文档的固定定位是：

- `P0 instrumentation` 和 `P1 runtime` 之间的桥
- `12_combined_main_thesis` 抽象层回流到当前主线的正式落点

更具体地说，它要把旧主线里最有价值的三层：

1. `Cacheability ABI`
2. `Typed Exact Prefix Contract`
3. `Post-Hit Execution Selection`

翻译成当前主线语言：

1. `reuse contract`
2. `bridgeability / false_negative_reason / route key`
3. `TwinPathServe`

## 2. 为什么必须有 `reuse contract`

## 2.1 仅有 hash 命中，不足以构成可执行复用

当前主线的核心判断固定为：

> prefix hash 命中只说明“系统里存在一份可能相关的状态”，并不自动说明“当前请求在自己的 API 语义、观测需求、carrier 和 backend 条件下，可以合法消费这份状态”。

因此，`FalseNegativeKV` 研究的问题不是：

- cache key 没对上

而是：

- 状态已经存在
- 但 producer 与 consumer 之间缺少显式复用契约
- 结果 runtime 保守地放弃了这次复用

## 2.2 没有 `reuse contract`，整条线会退化成 patch list

如果没有统一方法层，这条线最终会被 reviewer 理解成：

1. 修 `prompt_logprobs + APC`
2. 再修 fast backend incompatibility
3. 再给它们起一个统一问题名

这不够。

必须有 `reuse contract`，因为它承担三件事：

1. 说明什么叫“逻辑上可复用”
2. 说明为什么“逻辑可复用”没有变成“物理命中”
3. 说明 runtime 在什么条件下可以桥接、何时应放弃

## 3. 一句话定义

`reuse contract` 固定定义为：

> 一个显式描述 producer state 与 consumer requirement 之间兼容关系的运行时契约层。
> 它不负责证明 prefix 身份，而负责判定当前请求是否能够直接消费既有状态、需要哪类 bridge、以及是否值得兑现这次 counterfactual exact hit。

这里有三个关键词必须锁死：

- `producer state`
- `consumer requirement`
- `realization decision`

## 4. 固定术语

从现在开始，方法层相关文档统一只使用下面这组术语，不再发明新同义词。

- `reuse contract`
- `contract violation`
- `bridgeability`
- `bridge operator`
- `realization decision`
- `prefix family`
- `producer state class`
- `consumer requirement class`

和 measurement 层对齐的术语仍然保留：

- `logical hit`
- `physical hit`
- `false-negative hit`
- `bridgeable miss`

## 5. `reuse contract` 的三层结构

`reuse contract` 必须固定分成下面三层。

## 5.1 第一层：Identity Layer

这一层只回答：

- 我们在谈的是不是“同一类可复用前缀”

这一层不回答：

- 当前请求能不能消费
- 当前请求是否应该桥接

第一层的固定对象只有：

- `prefix_family`
- `carrier_type`
- `cache_salt`
- `lora_identity`
- `mm_identity`

原则：

- Identity Layer 只负责前缀家族归属
- 不承诺消费兼容性

## 5.2 第二层：Compatibility Layer

这一层回答：

- 当前 consumer 在其 API、观测需求、backend 与 carrier 约束下，是否能直接消费 producer state

这是 `reuse contract` 的核心层。

这一层的固定对象包括：

- `producer_state_class`
- `consumer_requirement_class`
- `backend_compatibility_class`
- `observation_requirement_class`
- `false_negative_risk_class`

原则：

- Compatibility Layer 只判定“能否直接消费”
- 不负责决定“要不要付代价恢复”

## 5.3 第三层：Realization Layer

这一层回答：

- 即使不能直接消费，runtime 是否应该通过某个 bridge 去兑现这次复用

这一层的固定对象包括：

- `bridgeability_class`
- `bridge_operator`
- `realization_mode`
- `fallback_mode`

原则：

- Realization Layer 不重新定义身份
- 只在兼容性信息基础上做决策

## 6. 固定输入维度

`reuse contract` 在 P1 中只允许读下面这些主输入维度，不再增加新的一级主维度。

## 6.1 `prefix_family`

定义：

- 由稳定 prefix 指纹导出的 family 标识

作用：

- 表示“我们正在比较的是不是同一类逻辑前缀”

备注：

- 它只负责身份，不负责语义兑现。

## 6.2 `carrier_type`

固定枚举：

- `tokens`
- `embeds`
- `multimodal`

作用：

- 表示 producer / consumer 的输入载体形态

备注：

- `carrier_type` 既影响身份，也影响兼容性。

## 6.3 `api_contract_class`

固定枚举：

- `generation`
- `prompt_logprobs`
- `pooling`

作用：

- 表示当前请求对返回语义的高层要求

备注：

- P1 不扩展到更多一级枚举。

## 6.4 `backend_compatibility_class`

P1 固定枚举：

- `dual_compatible`
- `reuse_preferred`
- `throughput_preferred`
- `unknown`

作用：

- 表示当前请求在不同执行池或不同 backend 路线上兑现复用收益的相对稳定性

## 6.5 `false_negative_risk_class`

P1 固定枚举：

- `low`
- `observation_mismatch`
- `backend_mismatch`
- `carrier_mismatch`
- `unknown`

作用：

- 表示导致 logical hit 无法兑现的主风险来源

## 7. 固定中间表示

为了避免方法层继续漂移，`reuse contract` 中间层固定只产出下面四类中间表示。

## 7.1 `producer_state_class`

定义：

- 运行时当前已存在、可供复用的状态类型描述

P1 中不需要把它做成极细枚举，固定只保留下面四类：

- `prefix_kv_only`
- `prefix_kv_plus_observation_sidecar`
- `prefix_kv_backend_specific`
- `unknown`

解释：

- `prefix_kv_only`
  - 仅存在普通 prefix KV，可供标准 generation 消费
- `prefix_kv_plus_observation_sidecar`
  - 在 prefix KV 之外，还存在可支撑 observation 的最小伴生状态
- `prefix_kv_backend_specific`
  - 状态存在，但兑现依赖特定 backend / pool
- `unknown`
  - 当前无法可靠判别

## 7.2 `consumer_requirement_class`

定义：

- 当前请求为了合法消费既有状态所需的条件类型

P1 固定只保留下列四类：

- `direct_prefix_consumable`
- `needs_observation_bridge`
- `needs_backend_compatible_path`
- `not_supported_online`

解释：

- `direct_prefix_consumable`
  - 可直接读 prefix cache
- `needs_observation_bridge`
  - 需要 sidecar / replay 等观察恢复
- `needs_backend_compatible_path`
  - 需要 route 到更适合兑现复用的执行路径
- `not_supported_online`
  - 当前阶段只测量，不在线恢复

## 7.3 `bridgeability_class`

定义：

- 当前 false-negative miss 是否属于本主线第一版可恢复的 miss

P1 固定枚举：

- `direct`
- `bridgeable`
- `defer_measure_only`
- `unknown`

解释：

- `direct`
  - 已可直接兑现，无需 bridge
- `bridgeable`
  - 需要 bridge，但属于当前主线恢复范围
- `defer_measure_only`
  - 已识别出问题，但本阶段只测量不恢复
- `unknown`
  - 暂时无法可靠归类

## 7.4 `bridge_operator`

P1 固定只允许下面三种值：

- `none`
- `prompt_logprobs_selective_replay`
- `twin_pool_route`

注意：

- `twin_pool_route` 在方法语义上被视为 bridge operator。
- 它不是因为直接改写 KV，而是因为它把请求送入更可能兑现 reuse 的执行路径。

## 8. 固定输出

`reuse contract` P1 固定输出下面五个字段。

## 8.1 `contract_status`

固定枚举：

- `satisfied`
- `violated`
- `unknown`

含义：

- 当前请求是否能在不付额外代价的前提下消费既有状态

## 8.2 `contract_violation_reason`

P1 固定只允许映射到：

- `prompt_logprobs_skip`
- `backend_incompatibility`
- `carrier_incompatibility`
- `pooling_skip`
- `unknown`

要求：

- 它必须和 measurement 层 `false_negative_reason` 对齐
- 不允许在方法层再发明一套全新 taxonomy

## 8.3 `bridgeability_class`

见第 7.3 节定义。

## 8.4 `selected_bridge_operator`

见第 7.4 节定义。

## 8.5 `realization_mode`

P1 固定枚举：

- `direct_reuse`
- `reuse_with_bridge`
- `measure_only`
- `fallback_full_prefill`

说明：

- `realization_mode` 是方法层最接近 runtime 决策的一层输出

## 9. 固定决策流程

`reuse contract` 的决策流程固定为五步。

1. 构建 `prefix_family`
2. 判定 `producer_state_class`
3. 判定 `consumer_requirement_class`
4. 计算 `bridgeability_class`
5. 输出 `selected_bridge_operator` 与 `realization_mode`

这五步不允许改成：

- 先 route 再判 contract
- 先 hardcode 某个 API fix 再补 contract 标签

因为那样会再次退化成 patch path。

## 10. P1 中的固定映射规则

为了让这份规格具备执行性，P1 先把最关键的两类 mismatch 写死。

## 10.1 `prompt_logprobs` observation mismatch

输入条件：

- `api_contract_class = prompt_logprobs`
- `carrier_type = tokens`
- `false_negative_risk_class = observation_mismatch`

固定判定：

- `producer_state_class = prefix_kv_only`
- `consumer_requirement_class = needs_observation_bridge`
- `bridgeability_class = bridgeable`
- `selected_bridge_operator = prompt_logprobs_selective_replay`
- `realization_mode = reuse_with_bridge`

fallback：

- 如果 selective replay 条件不成立
  - `realization_mode = fallback_full_prefill`

## 10.2 backend mismatch

输入条件：

- `false_negative_risk_class = backend_mismatch`
- `backend_compatibility_class = reuse_preferred`

固定判定：

- `producer_state_class = prefix_kv_backend_specific`
- `consumer_requirement_class = needs_backend_compatible_path`
- `bridgeability_class = bridgeable`
- `selected_bridge_operator = twin_pool_route`
- `realization_mode = reuse_with_bridge`

fallback：

- 如果 `reuse_optimized_pool` 不可用或超载
  - `realization_mode = fallback_full_prefill`

## 10.3 carrier mismatch

输入条件：

- `false_negative_risk_class = carrier_mismatch`

P1 固定判定：

- `consumer_requirement_class = not_supported_online`
- `bridgeability_class = defer_measure_only`
- `selected_bridge_operator = none`
- `realization_mode = measure_only`

意义：

- carrier mismatch 在 P1 只负责证明问题面，不承担在线恢复义务

## 11. 与现有文档的边界

为了避免职责混乱，这份文档和其他规格的边界固定如下。

## 11.1 与 `false_negative_instrumentation_spec.md`

它负责：

- 观测字段
- JSONL schema
- request-level trace

本文负责：

- 这些字段在方法层如何被解释与聚合

## 11.2 与 `twin_path_serve_minimal_architecture.md`

它负责：

- 双池系统边界
- route / family binding / fallback

本文负责：

- route / bridge 决策在方法层的统一语义

## 11.3 与 `prompt_logprobs_selective_replay_design.md`

它负责：

- 第一个 bridge 的具体执行设计

本文负责：

- 为什么这个 bridge 属于 `reuse contract` 的自然实例

## 12. 当前最重要的约束

从现在开始，`reuse contract` 只允许服务当前 active thesis，不允许反过来把 thesis 扩大。

因此：

- 不并入 PIC
- 不并入 hidden cache 系统
- 不并入 cluster-level cache pool
- 不并入 arbitrary-position reuse
- 不并入 cross-model sharing

否则这份规格就会失去边界。

## 13. 代码落点建议

这份文档不直接要求实现，但为了后续落地，固定建议的代码承载关系如下。

- `request.py`
  - 承载 identity 层和输入侧 typed tags
- `input_processor.py`
  - 承载 route key / prefix family 派生
- `false_negative_route.py`
  - 承载 contract 输入字段和部分兼容性分类
- `kv_cache_manager.py`
  - 承载 physical hit / skip 侧观测与 direct contract outcome
- `scheduler.py`
  - 承载 `realization_mode` 与 `selected_bridge_operator` 的主决策
- `twin_path_policy.py`
  - 承载 backend 路由与 family binding 的 realization policy

这里的固定原则是：

- 不新造一整套独立控制平面文件树
- 优先把 `reuse contract` 映射到当前主线路径中的现有模块

## 14. 验收标准

这份规格真正算立住，至少要满足下面四条。

1. `prompt_logprobs_skip` 和 `backend_incompatibility` 能被同一 contract 语言解释
2. measurement 层 taxonomy 与方法层 taxonomy 一一对应
3. `TwinPathServe` 中每个 route / bridge 决策都能在 contract 输出上找到语义来源
4. reviewer 不能再自然地把整篇论文理解成两个无关 feature fix

如果这四条做不到，这份规格就不算真的把抽象层补回来。

## 15. 一句话结论

`reuse contract` 从现在开始被固定为：

> `FalseNegativeKV / TwinPathServe` 当前主线的正式方法层。
> 它把“状态存在但未兑现”的问题，从零散的 skip / route / replay 逻辑，提升成一个统一的 runtime compatibility 与 realization 决策问题。
