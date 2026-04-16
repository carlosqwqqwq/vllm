# FalseNegativeKV Paper Evaluation Matrix

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档不再讨论要不要换 thesis。

它只负责把 `FalseNegativeKV` 变成一个 **可判死活的 measurement-first 计划**，并锁定：

- 四条必须被证伪或证成的假设
- 五类必须覆盖的 workload family
- 四组固定 baseline
- 明确的 go / no-go gate
- 明确的降级条件

如果这份矩阵最后不能给出清晰门槛，`FalseNegativeKV` 就不应继续被当作主 thesis。

## 2. 固定范围

本轮评测范围固定为：

- 单机或 colocated `vLLM`
- exact / logical-exact reuse
- 先做 measurement，再做最小 P1 recovery
- 不修改模型
- 不引入分布式 cache pool
- 不讨论 PIC / arbitrary-position reuse
- 不讨论 hidden cache / activation restoration
- 不讨论 workflow-aware KV retention
- 不讨论 cross-model prefill sharing
- 论文主结果优先来自公开 benchmark / 公开数据集；synthetic workload 只允许用于 smoke、debug 或 appendix

所有 P0/P1 结果都必须建立在
[../implementation/false_negative_instrumentation_spec.md](../implementation/false_negative_instrumentation_spec.md)
定义的 request-level JSONL traces 之上。

公开 benchmark 的固定绑定协议见：
[public_benchmark_evaluation_protocol.md](public_benchmark_evaluation_protocol.md)

## 3. 固定 workload family

本轮 workload family 固定为五类，不再删改：

| 编号 | workload family | 代表场景 | 重点 reason class | 作用 |
| --- | --- | --- | --- | --- |
| `W1` | 长 system prompt 多轮 chat | 长系统提示词、共享历史、suffix 轻微变化 | `prompt_logprobs_skip`、`backend_incompatibility` | 验证最基础的 logical hit 是否常被 conservative contract 吃掉 |
| `W2` | tool-rich / agentic | tools 定义固定、responses API、多轮 tool result 注入 | `prompt_logprobs_skip`、`backend_incompatibility`、`carrier_incompatibility` | 验证最接近生产 agent workload 的 false-negative 面 |
| `W3` | `prompt_embeds` | 相同 embedding prefix，多请求 suffix 变化 | `carrier_incompatibility` | 证明 false-negative 不是纯 token-only 问题 |
| `W4` | multimodal | 图像不变、文本 suffix 变化、placeholder 布局控制 | `carrier_incompatibility`、`partial_prefill_incompatibility` | 证明异构 carrier 下 logical hit 也可能丢失 |
| `W5` | APC 高命中但 residual 碎片化 | 长共享前缀，残余 prefill 短且形状波动 | `backend_incompatibility` | 专门验证 backend mismatch 与 twin-pool route |

固定解释：

- `W1` 和 `W2` 是第一批 recovery 必须出效果的主战场。
- `W3` 和 `W4` 在 P0 中必须测，但在 P1 中只承担“证明问题面更广”的职责，不要求第一版 bridge 全覆盖。
- `W5` 是 `TwinPathServe` 必须拿下的强实例化 workload；如果 `W5` 不成立，backend mismatch 的主张会明显变弱。

## 3.1 公开 benchmark 绑定要求

从现在开始，五类 workload family 在主结果中优先绑定到下面这组公开 benchmark：

| workload family | 主公开 benchmark | 备选公开 benchmark | 固定要求 |
| --- | --- | --- | --- |
| `W1` | `LongBench / LongBench v2` | `MT-Bench` | 主结果不得主要依赖手工长 prompt |
| `W2` | `BFCL` | `ToolBench` | 主结果必须使用公开 tool schema |
| `W3` | `MTEB` | `BEIR` | 至少完成公开数据上的 prevalence 测量 |
| `W4` | `MMBench` | `VLMEvalKit` 支持的公开多模态 benchmark | 不允许主表使用合成图像数据 |
| `W5` | `LongBench / LongBench v2` | `InfiniteBench` 真实子集 | backend mismatch 主结果不得主要依赖 synthetic retrieval 子集 |

若主结果无法落到这组公开 benchmark 上，必须在论文里明确降级 claim，而不能维持同等强度的主张。

## 4. 固定 baseline matrix

本轮只允许使用下面四组 baseline，不再新增第五组主 baseline：

| baseline | 固定含义 | 在矩阵中的角色 |
| --- | --- | --- |
| vanilla `vLLM V1` | 当前上游 `vLLM V1` 默认行为，不加任何 `FalseNegativeKV` 特化 | 端到端性能和命中行为的总基线 |
| status quo `skip_reading_prefix_cache` | 当前已有 conservative skip / disable 行为显式暴露后的决策基线 | 量化 false-negative miss 到底由现有 skip 决策造成多少 |
| ad hoc per-feature fix | 各 mismatch 单独修补的 best-effort patch，不共享 `reuse contract` / `bridge operator` | 检验 unified substrate 是否优于 scattered fixes |
| `FalseNegativeKV` P1 prototype | 带统一 measurement、`reuse contract`、`bridge operator` 和 `TwinPathServe` 的第一版系统 | 主方案 |

其中：

- `ad hoc per-feature fix` 固定只允许包含两个局部修补：
  - `prompt_logprobs` 的单点修补
  - backend mismatch 的单点 fallback / pinning
- `FalseNegativeKV` P1 prototype 固定只允许包含两个 bridge：
  - `prompt_logprobs` selective replay
  - batch-invariant fast-path twin-pool route

不允许把 `prompt_embeds` 或 multimodal 的新 feature 支持偷偷塞进 P1，以免 thesis 再次发散。

## 5. 固定核心指标

所有假设统一只围绕下面三层指标展开：

### 5.1 问题规模指标

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `bridgeable_tokens`
- `false_negative_reason`
- `false_negative_tokens / logical_hit_tokens`
- `bridgeable_tokens / false_negative_tokens`
- `unknown_false_negative_tokens / false_negative_tokens`

### 5.2 方法层有效性指标

- `contract_status` coverage
- `contract_violation_reason` coverage
- `bridgeability_class` coverage
- `selected_bridge_operator` coverage
- `unknown_contract_outcomes / false_negative_tokens`

固定解释：

- 这组指标服务于 `reuse contract` 本身，而不是端到端速度。
- 如果方法层分类解释不了主 workload，上面的 TTFT 数字就没有论文说服力。

### 5.3 端到端结果指标

- `TTFT`
- `p95 TTFT`
- `p99 TTFT`
- steady-state throughput
- `physical_hit_tokens` recovery
- `bridge_success_ratio`
- route fallback rate
- `realization_efficiency`
- `bridge_cost_per_realized_token`

固定要求：

- 所有 prevalence 结论必须先基于 request-level JSONL trace，再聚合成 family 结果。
- 所有端到端结果必须同时报告 warm-cache steady-state 和 cold-to-warm 过渡段。
- 只报平均值不够；`TTFT` 至少要有 `median/p95/p99`。
- `realization_efficiency` 固定定义为：
  - `recovered_physical_hit_tokens / counterfactual_hit_tokens`
- `bridge_cost_per_realized_token` 在 P1 中允许用近似代理指标：
  - `replay_tokens / recovered_physical_hit_tokens`
  - 或 route 带来的额外 fallback / queueing 代理成本

## 6. 假设矩阵

## 6.1 `H1`：false-negative misses 在真实 workload 中足够普遍

### 固定问题

真实 workload 中，是否存在足够多的 `logical hit` 没有变成 `physical hit`？

### 必测 workload

- `W1`
- `W2`
- `W3`
- `W4`
- `W5`

### 对比 baseline

- vanilla `vLLM V1`
- status quo `skip_reading_prefix_cache`

### 主指标

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `false_negative_tokens / logical_hit_tokens`
- `false_negative_reason` breakdown
- `unknown_false_negative_tokens / false_negative_tokens`

### 通过门槛

`H1` 只有在同时满足下面三条时才算成立：

1. 至少两个 workload family 中，`false_negative_tokens / logical_hit_tokens >= 10%`
2. 这两个 family 里至少一个不是纯 `prompt_logprobs` 单点 slice
3. 在所有达到门槛的 family 中，`unknown_false_negative_tokens / false_negative_tokens <= 20%`

### kill criteria

满足下面任意一条，就把 `H1` 判为失败：

1. 五个 family 里只有一个达到 `10%`
2. 所有 family 都低于 `5%`
3. 达到门槛的 family 中，`unknown` 占比超过 `30%`

### 解释

`H1` 的核心不是“找一个极端例子”，而是证明 false-negative miss 是 workload-level 现象，而不是 RFC 级边角 bug。

## 6.2 `H2`：统一 substrate 比 scattered fixes 更有解释力

### 固定问题

`FalseNegativeKV` 是否真的比“把每个不兼容路径单独修掉”更强，而不是只是给 patch collection 起了一个名字？

### 必测 workload

- `W1`
- `W2`
- `W5`

补充观测 workload：

- `W3`
- `W4`

### 对比 baseline

- status quo `skip_reading_prefix_cache`
- ad hoc per-feature fix
- `FalseNegativeKV` P1 prototype

### 主指标

- `false_negative_reason` coverage
- `contract_status` coverage
- `bridgeability_class` coverage
- `unknown_false_negative_tokens / false_negative_tokens`
- `bridgeable_tokens / false_negative_tokens`
- `TTFT` relative to best ad hoc fix
- throughput relative to best ad hoc fix

### 通过门槛

`H2` 只有在同时满足下面四条时才算成立：

1. 同一套 `reuse contract` + `false_negative_reason` taxonomy 能解释至少三个 family 中 `>= 80%` 的 false-negative tokens
2. 同一套 `reuse contract` 输出在至少三个 family 中达到 `contract_status` 和 `bridgeability_class` 双 coverage `>= 80%`
3. `FalseNegativeKV` P1 在 `prompt_logprobs_skip` 和 `backend_incompatibility` 两类 mismatch 上都给出正收益
4. 相对 best ad hoc fix，`FalseNegativeKV` P1 的 `median TTFT` 不得差于 `3%`
5. 相对 best ad hoc fix，`FalseNegativeKV` P1 的 throughput 回退不得超过 `5%`

### kill criteria

满足下面任意一条，就把 `H2` 判为失败：

1. `prompt_logprobs_skip` 和 `backend_incompatibility` 需要完全不同的控制面，无法共享 `reuse contract` / `bridge operator`
2. best ad hoc fix 在所有正收益 workload 上都不差于 `FalseNegativeKV` P1，且没有可见的统一解释收益
3. `unknown` 在高 prevalence family 中仍超过 `20%`

### 解释

`H2` 是整篇论文是否会被打成 “why not fix separately” 的关键。如果这条不成立，就不应再写 unified substrate 的论文故事。

## 6.3 `H3`：至少两类 mismatch 可被同一机制框架恢复

### 固定问题

同一套 `reuse contract` / `bridge operator` 机制，是否真的能恢复两种不同来源的 false-negative hit？

### 必测 workload

- `W1`
- `W2`
- `W5`

### 对比 baseline

- status quo `skip_reading_prefix_cache`
- ad hoc per-feature fix
- `FalseNegativeKV` P1 prototype

### 主指标

- recovered `physical_hit_tokens`
- recovered `false_negative_tokens`
- `bridge_success_ratio`
- `TTFT`
- `p95 TTFT`
- steady-state throughput

### 固定恢复目标

本轮只允许验证下面两类 mismatch：

1. `prompt_logprobs_skip`
   - 由 `prompt_logprobs` selective replay 恢复
2. `backend_incompatibility`
   - 由 batch-invariant fast-path twin-pool route 恢复

### 通过门槛

`H3` 只有在同时满足下面四条时才算成立：

1. `prompt_logprobs` selective replay 在 `W1` 或 `W2` 中至少一个 workload 上恢复 `>= 50%` 的 baseline `false_negative_tokens`
2. 上述 workload 上 `TTFT` 改善 `>= 15%`
3. twin-pool route 在 `W5` 上实现 `TTFT >= 10%` 改善，或 recovered `physical_hit_tokens >= 50%`
4. 两类机制都满足 throughput 回退 `<= 5%`

### kill criteria

满足下面任意一条，就把 `H3` 判为失败：

1. 只有一类 mismatch 有正收益
2. 任一类恢复都依赖手工 workload pinning，而不是统一 router / bridge path
3. 任一类 workload 出现 throughput 回退 `> 5%`

### 解释

`H3` 直接决定论文能不能叫“统一系统 + 双实例化”。如果最终只剩一类 mismatch 有用，`FalseNegativeKV` 就应降级为 feature paper / patch line。

## 6.4 `H4`：恢复后有显著端到端收益

### 固定问题

即便 false-negative miss 存在，恢复它们之后是否真的能带来 reviewer 无法忽略的端到端收益？

### 必测 workload

- `W1`
- `W2`
- `W5`

补充观测 workload：

- `W3`
- `W4`

### 对比 baseline

- vanilla `vLLM V1`
- ad hoc per-feature fix
- `FalseNegativeKV` P1 prototype

### 主指标

- `TTFT`
- `p95/p99 TTFT`
- steady-state throughput
- `physical_hit_tokens`
- `bridge_success_ratio`
- `realization_efficiency`
- `bridge_cost_per_realized_token`

### 通过门槛

`H4` 只有在同时满足下面四条时才算成立：

1. 至少一个强实例化达到 `TTFT >= 20%` 改善
2. aggregate throughput 回退不得超过 `5%`
3. 至少两类 mismatch 都有正收益
4. 至少三个 workload family 的 `p95 TTFT` 不差于 vanilla `vLLM V1`
5. 至少一个主战场 workload 上 `realization_efficiency >= 50%`

### kill criteria

满足下面任意一条，就把 `H4` 判为失败：

1. 只有一个很窄的 microbenchmark 有收益
2. aggregate throughput 回退超过 `5%`
3. 提升只出现在 average `TTFT`，但 `p95/p99` 没有改善
4. `realization_efficiency` 长期低于 `30%`，说明 bridge 多数没有把 counterfactual hit 真正兑现出来

### 解释

`H4` 不是锦上添花，而是论文是否站得住的底线。没有明显端到端收益，`FalseNegativeKV` 最多只能算 measurement 或 compatibility work。

## 7. 固定评测矩阵

下面这张表把假设、workload、baseline 和结论角色绑定起来：

| workload | H1 | H2 | H3 | H4 | 必测 baseline | 备注 |
| --- | --- | --- | --- | --- | --- | --- |
| `W1` 长 system prompt 多轮 chat | 必测 | 必测 | 必测 | 必测 | 4 组全测 | `prompt_logprobs` selective replay 主战场之一 |
| `W2` tool-rich / agentic | 必测 | 必测 | 必测 | 必测 | 4 组全测 | 最接近生产 workload |
| `W3` `prompt_embeds` | 必测 | 补充观测 | 不要求 P1 恢复 | 不要求强收益 | 至少测前 2 组 + P1 观测 | 用于证明问题面更广 |
| `W4` multimodal | 必测 | 补充观测 | 不要求 P1 恢复 | 不要求强收益 | 至少测前 2 组 + P1 观测 | 用于证明 carrier mismatch 真实存在 |
| `W5` APC 高命中但 residual 碎片化 | 必测 | 必测 | 必测 | 必测 | 4 组全测 | twin-pool route 主战场 |

## 8. 固定 go / no-go gate

只要最终结果不满足下面四条，`FalseNegativeKV` 就不能继续作为主 thesis：

1. 至少两个 workload family 中，`false_negative_tokens / logical_hit_tokens >= 10%`
2. 至少一个强实例化达到 `TTFT >= 20%` 改善
3. 吞吐回退不得超过 `5%`
4. 至少两类 mismatch 都有正收益，否则降级为 patch 线

## 9. 固定降级规则

### 9.1 降级为 patch line

满足下面任意一条，就降级：

- 只能证明 `prompt_logprobs_skip` 一类 mismatch 有效
- 只能证明 backend mismatch 一类有效
- unified substrate 没有明显优于 ad hoc per-feature fix

### 9.2 停止作为主 thesis

满足下面任意一条，就停止扩写论文叙事：

- `H1` 失败
- `H2` 失败
- `H4` 失败

## 10. 固定产出物

这份矩阵对应的论文主结果至少要产出下面四张表或图：

1. prevalence table
   - 五类 workload 的 `logical hit / physical hit / false-negative hit`
2. reason breakdown figure
   - `false_negative_reason` 分布
3. two-bridge comparison table
   - ad hoc fix vs `FalseNegativeKV` P1
4. end-to-end result figure
   - `TTFT / p95 / throughput / realization_efficiency`

如果最后连这四个结果都凑不齐，就说明 measurement-first 阶段还没把 thesis 做厚。
