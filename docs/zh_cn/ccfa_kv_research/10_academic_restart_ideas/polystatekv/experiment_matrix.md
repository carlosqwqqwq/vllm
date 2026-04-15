# PolyStateKV：实验矩阵与判死标准

## 1. 实验要证明什么

`PolyStateKV` 第一版不需要证明“状态形式越多越好”，只需要证明下面这个核心命题：

> 在相同 memory budget 下，把 reusable state 固定为单一形式，不如允许 runtime 在两种状态形式之间做在线选择。

第一版建议只使用两种状态形式：

1. full `KV`
2. hidden-state checkpoint

## 2. 核心假设

### H1

不同请求的 reuse distance、SLA 敏感度和上下文长度差异，会导致**固定单一状态形式**在部分区域系统性吃亏。

### H2

在线 `state-form selection` 能在不显著扩大系统复杂度的前提下，改善 `TTFT / throughput / memory` 的 Pareto front。

### H3

这种收益不是只来自“多存一点状态”，而是来自：

- 对高 reuse-probability 请求保留 full `KV`
- 对低 reuse-probability 但仍值得保温的状态降级为 lighter form

## 3. 基线设计

第一版不必强求完整复现外部论文系统，更稳妥的做法是构造**命题对照基线**：

1. `Static-KV`
   - 所有 reusable state 一律保存为 full `KV`
2. `Static-Hidden`
   - 所有可降级状态一律保存为 hidden-state checkpoint
3. `Threshold-Demotion`
   - 用固定阈值做 form 切换，不做在线成本比较
4. `PolyStateKV`
   - 使用统一成本模型与 online selector

如果后续实现资源允许，可以追加：

5. `Apt-Serve-like fixed hybrid`
   - 不是全文复现，而是用“固定 form ratio / 固定 hybrid placement”模拟固定 hybrid cache 思路

## 4. Workload 分层

第一版至少覆盖三类 workload：

### 4.1 短上下文高并发 chat / RAG

用途：

- 验证在常见服务场景下，`PolyStateKV` 不会因为 selector 开销而失去收益

关注指标：

- `TTFT p50/p95/p99`
- requests/sec
- selector overhead

### 4.2 中长上下文、reuse distance 混合

用途：

- 这是最可能体现 `PolyStateKV` 优势的主场景

关注指标：

- peak GPU memory
- active reusable contexts
- throughput under fixed memory budget

### 4.3 长上下文但 reuse 呈分层分布

用途：

- 验证单一 full `KV` 与单一 lighter form 各自的短板是否真的存在

关注指标：

- rematerialization latency
- storage footprint
- TTFT degradation under pressure

## 5. 核心指标

### 5.1 端到端指标

- `TTFT p50 / p95 / p99`
- steady-state throughput
- effective throughput under SLO

### 5.2 资源指标

- peak GPU memory
- CPU / host memory footprint
- number of retained reusable states

### 5.3 机制指标

- demotion rate
- rematerialization count
- average rematerialization cost
- selector decision latency
- wrong-form decisions

这里的 `wrong-form decisions` 可定义为：

- 明显应保留 full `KV` 却被降级
- 或明显应降级却继续占用 full `KV`

## 6. 消融实验

至少应包含以下消融：

1. 无 selector，仅固定单一形式
2. 固定阈值 selector
3. 去掉 `reuse horizon`
4. 去掉 `sla class`
5. 去掉 memory pressure signal

这样才能证明收益来自“统一成本模型下的在线选择”，而不是某个偶然 heuristic。

## 7. 结果叙事应该验证的命题

实验不是为了证明 `PolyStateKV` 在所有指标上绝对最强，而是为了验证三个更精确的结论：

1. 固定 full `KV` 在显存压力下会过早丢失可复用状态承载能力
2. 固定 lighter form 会在短 reuse distance 场景下引入额外 TTFT penalty
3. 在线 form selection 可以把两类极端的缺点同时压低

## 8. 判死标准

如果出现下面任一情况，我建议直接停止把 `PolyStateKV` 当主线推进：

1. 主流 workload 下最优策略几乎总是 `Static-KV`
2. 主流 workload 下最优策略几乎总是 `Static-Hidden`
3. `PolyStateKV` 相比固定阈值 selector 没有稳定优势
4. selector 自身开销明显抵消收益
5. 收益只存在于非常特殊的长上下文角落场景

## 9. 与论文主张的对应关系

实验矩阵必须和论文主张一一对应：

- 论文说“单一形式默认前提不成立”
  - 就必须有 `Static-KV` 与 `Static-Hidden` 的失败案例
- 论文说“统一成本模型有意义”
  - 就必须有 fixed-threshold 对照
- 论文说“vLLM 上可落地”
  - 就必须记录 selector overhead 与实现复杂度边界

## 10. 当前建议

第一版实验最重要的不是追求大而全，而是尽快回答下面这个生死问题：

> 在现实 serving workload 下，是否真的存在一块足够大的区域，使得在线选择两种状态形式显著优于固定单一形式？

只要这个问题回答不了，`PolyStateKV` 的论文主张就站不住。

## 11. 关键资料

- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- Fast State Restoration in LLM Serving with HCache: <https://arxiv.org/abs/2410.05004>
- Efficient LLM Inference with Activation Checkpointing and Hybrid Caching: <https://arxiv.org/abs/2501.01792>
- ShadowKV: <https://arxiv.org/abs/2410.21465>
- Cake: <https://arxiv.org/abs/2410.03065>
