# `prompt_logprobs selective replay` output-equivalence 审计方案

更新日期：2026-04-16

## 1. 这份文档只解决什么问题

这份文档只解决一个当前最关键、也最容易被 reviewer 继续追打的问题：

- `prompt_logprobs selective replay` 虽然已经证明了恢复收益与时延收益，但是否已经证明它不会改变用户可见输出

它不负责：

- 再解释 `FalseNegativeKV` 的总 thesis
- 替代第五章系统设计
- 虚构尚未完成的 output-equivalence 结果

换言之，这份文档的目标不是“再写一遍 bridge I 为什么有用”，而是把 bridge I 缺的那块 correctness 审计补成一个可直接执行的实验方案。

## 2. 先给严格判断

截至 `2026-04-16`，`prompt_logprobs selective replay` 已经能够稳定支撑下面这些主张：

- `W1` 的 false-negative hit 可以在真实 runtime 中被恢复成高 `effective reuse`
- 这条 bridge 已经带来显著 phase wall time 改善
- sidecar、boundary replay 与 replay budget 都是必要条件

但它还不能直接支撑下面这个更强的主张：

- 在统一 public runner 上，bridge I 已经完成完整 output-equivalence 审计

因此，下一步最该补的不是再跑更多吞吐数字，而是补一套足够让 reviewer 信服的 output-equivalence audit。

## 3. 审计对象与固定边界

当前审计对象固定为：

- `prompt_logprobs selective replay`

当前不纳入这份审计主线的对象包括：

- `TwinPathServe` 的 twin-pool route
- `carrier_incompatibility` 的未来 bridge
- cross-framework 对照

原因很简单：

- 第二类 bridge 的主要问题是 route correctness，不是 prompt observation correctness
- reviewer 当前最容易卡住的 correctness 缺口集中在 bridge I
- 如果第一类 bridge 的 output-equivalence 都没有做扎实，继续横向扩面只会分散精力

## 4. 这份审计要回答的四个问题

这轮审计固定回答四个问题，不再发散。

1. `ObservationCompleteness`
   - selective replay 返回的 `prompt_logprobs` 字段是否完整，是否与 `full-prefill baseline` 保持相同位置数和相同 schema
2. `TokenAndRankEquivalence`
   - 每个 prompt position 的 top-k token、token id 与 rank 是否与 baseline 对齐
3. `NumericalEquivalence`
   - 在 token/rank 对齐的前提下，各位置 logprob 数值是否只存在可接受的数值误差
4. `FallbackEquivalence`
   - sidecar 不可用、replay budget 不足或任一契约条件失败时，fallback 路径是否仍保持与 baseline 一致的接口与输出语义

如果这四个问题没有被同时回答，bridge I 仍然只能算“恢复有效”，还不能算“恢复且可审计”。

## 5. 固定对照组

本轮审计固定只使用下面三组对照：

| 组别 | 固定含义 | 在审计中的角色 |
| --- | --- | --- |
| `full_prefill_baseline` | 不使用 selective replay，按当前支持路径完整 prefill 并返回 `prompt_logprobs` | correctness ground truth |
| `status_quo_skip` | 当前默认 conservative 行为，保留 `prompt_logprobs` 但不兑现 prefix reuse | 问题背景，不参与 output-equivalence 判定 |
| `selective_replay_on` | 开启 sidecar + boundary replay 的 bridge I 原型 | 主比较对象 |

需要明确：

- 这份审计的主比较永远是 `full_prefill_baseline` 对 `selective_replay_on`
- `status_quo_skip` 的价值在于说明为什么存在 false-negative miss，不在于提供 correctness ground truth

## 6. 固定 workload 切片

为了避免只在最容易成功的 case 上做审计，这轮固定拆成四类 workload 切片。

### 6.1 `A` 类：dedicated validation 切片

目的：

- 先在最可控条件下验证接口完整性和 token/rank 一致性

固定做法：

- 同一 `prompt family` 下顺序发送两次请求
- 第一次建立 sidecar
- 第二次作为 probe，比对 `full_prefill_baseline` 与 `selective_replay_on`

### 6.2 `B` 类：public `W1` 切片

目的：

- 证明 output-equivalence 审计不是只在手工构造 slice 上成立

固定绑定：

- `MT-Bench W1`
- `LongBench W1`

固定要求：

- 至少覆盖短上下文与长上下文两种公开负载
- 不允许只报 dedicated validation 结果而把 public slice 留空

### 6.3 `C` 类：boundary-sensitive 切片

目的：

- 直接审计最可能出错的位置，也就是命中边界附近的 prompt observation

固定覆盖：

- `num_prompt_logprobs = 1`
- `num_prompt_logprobs = 5`
- `num_prompt_logprobs = 10`
- 命中前缀很长、uncached suffix 很短
- 命中前缀较短、replay 占比高

### 6.4 `D` 类：fallback 切片

目的：

- 证明系统在不满足契约条件时会正确回退，而不是返回“部分正确”的 observation

固定覆盖：

- sidecar 缺失
- sidecar key 不匹配
- replay budget 不足
- `contract gate` 主动拒绝 bridge

## 7. 必须保存的实验产物

如果后续实验没有保存下面这些产物，这轮 output-equivalence 审计就不能算完整。

### 7.1 request 级 manifest

至少包含：

- `request_id`
- `prefix_family_id`
- `num_prompt_logprobs`
- `prompt_token_count`
- `physical_hit_tokens`
- `effective_reuse_tokens`
- `selected_bridge_operator`
- `realization_mode`
- `sidecar_key_ready`
- `replay_budget`
- `replay_tokens`
- `fallback_reason`

### 7.2 response 级原始输出

至少保存：

- 原始 response JSON
- 每个 prompt position 的返回 schema
- 每个 position 的 top-k token / rank / logprob

### 7.3 对齐后的审计视图

为避免后处理时再引入歧义，审计阶段必须额外导出一份 position-aligned 的规范化视图。每个 prompt position 至少包含：

- `position_index`
- `observed_token_id`
- `topk_token_ids`
- `topk_ranks`
- `topk_logprobs`

这份对齐视图不是用户接口，而是 correctness audit 的唯一对比口径。

## 8. 固定指标

本轮 output-equivalence 审计统一使用下面九个指标。

| 指标 | 固定含义 |
| --- | --- |
| `field_completeness_rate` | 返回字段完整率，检查是否缺少 `prompt_logprobs` 必要字段 |
| `position_alignment_rate` | 返回 position 数与 prompt token 数是否完全对齐 |
| `topk_token_exact_match_rate` | baseline 与 bridge 在各 position 的 top-k token id 完全一致比例 |
| `rank_exact_match_rate` | baseline 与 bridge 在各 position 的 token rank 完全一致比例 |
| `logprob_mae` | 对齐 token 上的平均绝对误差 |
| `logprob_p99_abs_diff` | 对齐 token 上绝对误差的 `p99` |
| `logprob_max_abs_diff` | 对齐 token 上的最大绝对误差 |
| `request_output_equivalent_ratio` | 满足“字段完整 + 位置对齐 + token/rank 一致 + 数值误差在门槛内”的请求占比 |
| `fallback_interface_equivalent_ratio` | fallback 请求在接口与输出语义上等同于 baseline 的比例 |

补充要求：

- 所有指标都必须同时按 request 粒度和按 prompt position 粒度聚合
- 所有 public slice 都必须给出 `p50 / p95 / p99` 或等价分位口径

## 9. 固定失败分类

为避免所有不一致都被塞进一个模糊的“diff”里，这轮审计固定把失败分成五类。

1. `schema_mismatch`
   - 字段缺失、字段类型错误、position 数不一致
2. `token_set_mismatch`
   - top-k token id 集合不一致
3. `rank_mismatch`
   - token 相同但 rank 顺序不一致
4. `numeric_drift`
   - token 与 rank 对齐，但 logprob 数值误差超过容差
5. `fallback_semantic_mismatch`
   - 本应回退的请求没有走 baseline 语义，或返回了部分 observation

只有把失败先按这五类拆开，后续 reviewer 才能清楚知道我们到底是“哪里不等价”。

## 10. 固定通过门槛

这轮审计的通过门槛必须足够严格，否则文稿里仍然不能强写 output-equivalence。

### 10.1 硬门槛

下面四条必须全部成立：

1. `field_completeness_rate = 100%`
2. `position_alignment_rate = 100%`
3. `fallback_interface_equivalent_ratio = 100%`
4. `schema_mismatch` 与 `fallback_semantic_mismatch` 计数必须为 `0`

### 10.2 token/rank 门槛

下面两条必须全部成立：

1. `topk_token_exact_match_rate = 100%`
2. `rank_exact_match_rate = 100%`

解释：

- 对 bridge I 而言，token/rank 不一致不是“小数值抖动”，而是用户可见输出变化
- 因此不能用“平均误差很小”去掩盖 token/rank 的实际偏差

### 10.3 数值容差门槛

在 token/rank 已完全对齐的前提下，数值层固定采用两层门槛：

1. 主门槛：
   - `logprob_p99_abs_diff <= 1e-4`
2. 审计门槛：
   - `logprob_max_abs_diff <= 1e-3`

如果超过主门槛但未超过审计门槛，可以继续做 failure analysis，但不能把结果直接写成“已完成 output-equivalence 审计”。

## 11. 固定论文写法映射

这轮审计不是为了堆 appendix，而是为了把论文里最容易被抓的 correctness 口径写稳。

如果通过门槛成立，主文稿可以稳妥写成：

- bridge I 已完成接口完整性、token/rank 一致性与数值容差审计
- 在 fallback 条件不满足时，系统保持与 baseline 等价的接口语义

如果门槛没有成立，主文稿只能写成：

- bridge I 已证明恢复收益与显著时间收益
- output-equivalence audit 仍在进行中

不允许出现第三种含糊写法。

## 12. 最推荐的执行顺序

最推荐的顺序如下：

1. 先在 `A` 类 dedicated validation 上打通完整审计流水
2. 再把同一审计口径接进 `MT-Bench W1`
3. 再接进 `LongBench W1`
4. 最后集中补 `D` 类 fallback 切片

原因是：

- 如果 dedicated validation 连 position-aligned diff 都对不齐，public slice 只会把问题放大
- 如果 fallback 切片最后才做，能避免过早被一些尚未整理好的 failure mode 打断主干推进

## 13. 一句话结论

`prompt_logprobs selective replay` 现在最该补的不是更多性能图，而是一套足以回答“输出有没有变”这一问题的严格审计。只要这份审计做扎实，bridge I 就能从“方向正确、收益显著”推进到“收益显著且 correctness 可审计”。
