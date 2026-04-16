# unified baseline matrix 设计方案

更新日期：2026-04-16

## 1. 这份文档的作用

这份文档只负责一件事：

- 把当前分散在 `W1`、`W5`、dedicated validation 与 formal evaluation 里的实验结果，收敛成一套 reviewer 能直接看懂的 unified baseline matrix

它不负责：

- 再定义新的 thesis
- 替代 [paper_evaluation_matrix.md](paper_evaluation_matrix.md) 的总 gate
- 虚构尚未完成的实验结果

它的职责更具体一点说，是回答下面这个问题：

- 如果 reviewer 问“为什么 unified substrate 比 best ad hoc fix 更强”，我们应该用哪一张统一矩阵来回答，而不是把若干孤立表格拼起来

## 2. 先给严格判断

截至 `2026-04-16`，当前结果已经足以说明两件事：

- `W1` 的 observation mismatch 可以被 bridge I 恢复
- `W5` 的 backend mismatch 可以被 twin-pool route 恢复

但这些结果仍然存在一个明显问题：

- 它们还没有被放进同一套 baseline、同一组指标、同一条结论链里

这意味着 reviewer 仍然可能把论文读成：

- 一个 `prompt_logprobs` 修复
- 一个 backend route 修复
- 外加一些 supporting 指标

所以，unified baseline matrix 的目标不是再跑一轮“更多数字”，而是把已有与待补结果统一到一个 reviewer 友好的比较框架里。

## 3. 固定设计原则

这轮 baseline matrix 固定遵守下面五条原则。

1. 统一 baseline 语义
   - 同一 workload 不允许今天和 vanilla 比、明天和 ad hoc fix 比，却不说明 baseline 含义变化
2. 统一指标层次
   - 每个结果都必须同时落到问题规模、方法层有效性和端到端收益三层指标
3. 统一 workload 解释
   - `W1/W2/W5` 是 recovery 主结果，`W3/W4` 是 prevalence 与 supporting 结果
4. 统一 claim 边界
   - 若某个 workload 当前只完成 prevalence，就不能把它写进 recovery 主表
5. 统一失败可见性
   - fallback rate、unknown coverage、route miss 不允许被省略

没有这五条，baseline matrix 只会变成更大的拼盘，而不会增强论文说服力。

## 4. 固定 baseline 定义

这轮统一矩阵只允许出现下面四组 baseline。

| baseline | 固定含义 | 允许出现的实现成分 |
| --- | --- | --- |
| `vanilla_vllm_v1` | 当前上游 `vLLM V1` 默认行为 | 不含任何 `FalseNegativeKV` 特化 |
| `status_quo_skip` | 当前已有 conservative skip / disable 行为显式可见后的决策基线 | 只暴露已有 skip 行为，不新增恢复 |
| `best_ad_hoc_fix` | 各 mismatch 的 best-effort 局部修补 | 允许单点修补，但不共享 `reuse contract` / `realization policy` |
| `TwinPathServe_P1` | 当前统一 measurement + contract + realization + runtime 的第一版系统 | 只允许 bridge I 与 bridge II |

固定解释：

- `best_ad_hoc_fix` 不是“我们愿意给它加多少能力就加多少能力”
- `TwinPathServe_P1` 也不是“把所有可能的修补都塞进去”
- 两边都必须保持边界可比，否则矩阵会失真

## 5. 固定 workload 与结果角色

这轮统一矩阵固定把 workload 分成两类角色。

### 5.1 recovery 主结果 workload

固定包含：

- `W1`
- `W2`
- `W5`

这三类 workload 必须出现在 unified recovery matrix 中，因为它们直接回答：

- unified substrate 是否比 scattered fixes 更强
- 至少两类 mismatch 是否已被同一机制框架恢复

### 5.2 prevalence / supporting workload

固定包含：

- `W3`
- `W4`

这两类 workload 当前不强行进入 recovery 主表，但必须出现在 prevalence 表或 appendix supporting 表中，用来支撑：

- 问题面并不局限于 token-only chat workload

## 6. 统一矩阵不等于一张表

为了既保持统一，又避免主表过于臃肿，这轮 unified baseline matrix 固定拆成三张关联表，而不是强行塞进一张超大总表。

### 6.1 表 `A`：problem prevalence matrix

作用：

- 回答问题规模与现象分布

固定 workload：

- `W1`
- `W2`
- `W3`
- `W4`
- `W5`

固定 baseline：

- `vanilla_vllm_v1`
- `status_quo_skip`

固定指标：

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `false_negative_tokens / logical_hit_tokens`
- `false_negative_reason` breakdown
- `unknown_false_negative_tokens / false_negative_tokens`

### 6.2 表 `B`：unified recovery matrix

作用：

- 回答 unified substrate 与 best ad hoc fix 的正面对比

固定 workload：

- `W1`
- `W2`
- `W5`

固定 baseline：

- `status_quo_skip`
- `best_ad_hoc_fix`
- `TwinPathServe_P1`

固定指标：

- `contract_status` coverage
- `bridgeability_class` coverage
- `selected_bridge_operator`
- `realization_efficiency`
- `bridge_cost_per_realized_token`
- `median/p95/p99 TTFT`
- steady-state throughput
- route fallback rate

### 6.3 表 `C`：correctness and failure appendix matrix

作用：

- 回答 reviewer 对 correctness、fallback 与失败分类的追问

固定 workload：

- `W1` 的 bridge I output-equivalence 审计切片
- `W5` 的 route/fallback 切片

固定 baseline：

- `full_prefill_baseline`
- `best_ad_hoc_fix`
- `TwinPathServe_P1`

固定指标：

- `request_output_equivalent_ratio`
- `fallback_interface_equivalent_ratio`
- `schema_mismatch`
- `token_set_mismatch`
- `rank_mismatch`
- `numeric_drift`
- `fallback_semantic_mismatch`

## 7. recovery 主表的固定写法

为了避免不同 workload 的表头反复变化，`B` 表固定按下面这个列顺序展开。

| workload | baseline | `false_negative_rate` | `contract_coverage` | `bridgeability_coverage` | `realization_efficiency` | `bridge_cost_per_realized_token` | `median TTFT` | `p95 TTFT` | throughput | fallback rate |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

固定解释：

- `false_negative_rate` 负责说明这个 workload 为什么值得恢复
- `contract_coverage` 与 `bridgeability_coverage` 负责说明统一方法层是否真的在工作
- `realization_efficiency` 与 `bridge_cost_per_realized_token` 负责说明恢复是否值得
- `TTFT / throughput / fallback rate` 负责说明恢复是否在系统层站得住

只要这组列固定下来，reviewer 就很难再说“你们每个 case 都挑了最有利的指标”。

## 8. `best_ad_hoc_fix` 的固定边界

这一点必须写死，否则 unified matrix 很容易被 baseline 污染。

`best_ad_hoc_fix` 固定只允许包含下面两类局部修补：

1. `prompt_logprobs` 的单点修补
2. backend mismatch 的单点 fallback / family pinning

它固定不允许包含：

- 共享的 `reuse contract`
- 共享的 `realization policy`
- 统一的 bridge cost 记账
- 面向多个 mismatch 的统一 route/admission 逻辑

原因是：

- 一旦把这些都加进去，它就不再是 ad hoc baseline，而是在偷用 `TwinPathServe_P1` 的结构性贡献

## 9. `TwinPathServe_P1` 的固定边界

为了保证矩阵可比，`TwinPathServe_P1` 也必须自我约束。

它固定只允许包含：

- request-level measurement surface
- `reuse contract`
- `realization policy`
- `prompt_logprobs selective replay`
- batch-invariant twin-pool route

它固定不允许提前包含：

- `W3/W4` 的新 feature 支持
- distributed cache pool
- PIC / arbitrary-position reuse
- hidden cache / activation restoration

否则主表比较的就不是 “unified substrate vs scattered fixes”，而是“功能更多的一方自然更强”。

## 10. 统一结果写法需要的最小日志字段

若未来代码推进没有统一留下面这些字段，baseline matrix 就无法按同一口径聚合。

### 10.1 问题规模层

必须有：

- `logical_hit_tokens`
- `physical_hit_tokens`
- `false_negative_tokens`
- `false_negative_reason`

### 10.2 方法层

必须有：

- `contract_status`
- `contract_violation_reason`
- `bridgeability_class`
- `selected_bridge_operator`

### 10.3 兑现层

必须有：

- `realization_mode`
- `counterfactual_hit_tokens`
- `recovered_physical_hit_tokens`
- `realization_efficiency`
- `bridge_cost_proxy`

### 10.4 系统层

必须有：

- `selected_pool`
- `route_reason`
- `fallback_reason`
- `TTFT`
- throughput 相关聚合字段

## 11. 固定 go / no-go 解释规则

这份矩阵不是为了让所有 workload 都显得成功，而是为了能诚实地给出 go / no-go 判断。

### 11.1 可以维持当前 thesis 强度的条件

下面三条至少要满足两条：

1. `W1` 在 unified recovery matrix 中相对 `best_ad_hoc_fix` 仍然保持正的 `realization_efficiency` 优势或相当性能但更好统一解释力
2. `W5` 在 unified recovery matrix 中相对 `best_ad_hoc_fix` 保持正的 route recovery 优势，且 fallback rate 可接受
3. `W2` 至少有一个公开 workload 子集被纳入同一矩阵并获得正收益

### 11.2 必须降级 claim 的条件

满足下面任意一条，就必须降低论文主张强度：

1. 只有 `W1` 能稳定进 recovery 主表
2. `best_ad_hoc_fix` 在 `W1/W5` 都不差于 `TwinPathServe_P1`，且统一解释层没有显著优势
3. `realization_efficiency` 无法在多个 workload 上进入稳定统计
4. fallback rate 太高，导致主表收益主要来自少量幸运 slice

## 12. 这套矩阵如何映射到论文主表

最推荐的论文组织方式如下：

1. 主文第一个结果图：
   - 用 `A` 表回答 false-negative prevalence
2. 主文主结果表：
   - 用 `B` 表回答 unified substrate vs ad hoc baseline
3. Appendix 或补充主表：
   - 用 `C` 表回答 correctness、fallback 与 failure breakdown

这比“每个机制一张表”更强，因为它直接把论文从 patch story 推成 system comparison story。

## 13. 最推荐的执行顺序

最推荐的推进顺序如下：

1. 先把 `W1` 和 `W5` 按统一列头重新聚合一次
2. 再把 `W2` 的最小公开 workload 子集接进同一表
3. 再把 `W3/W4` 的 prevalence 表补齐
4. 最后把 correctness appendix matrix 与主表对齐

原因是：

- `W1/W5` 已经有最强证据，最适合先形成模板
- `W2` 一旦进表，论文的一般性会明显增强
- `W3/W4` 当前更适合作为问题面 supporting，而不是强行追求 P1 recovery

## 14. 一句话结论

unified baseline matrix 的真正作用，不是让表格变得更大，而是让 reviewer 看到：我们比较的不是两个零散 patch，而是一套统一的方法层、兑现层和运行时控制面，是否真的在同一组 workload 上比 scattered fixes 更有解释力、更有恢复能力、而且代价可控。
