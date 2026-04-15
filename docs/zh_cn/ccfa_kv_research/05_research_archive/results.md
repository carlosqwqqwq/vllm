# 调研结果与判断

## 1. 截至本轮调研，对 vLLM 的总体判断

截至 **2026-04-14**，我对 `vLLM` 在 KV cache 上的总体判断是：

> `vLLM` 已经从“单机 PagedAttention + APC”的阶段，演进到了“面向 Hybrid KV、多级 KV、外部 KV 与分布式部署”的阶段；当前真正缺少的，不再是单点功能，而是更统一、更稳定、更可调度的 KV 运行时系统。

这意味着：

- 现在的研究重点不应再是“再做一个基础缓存”
- 更有价值的是把已有能力统一起来，或补完当前体系缺口

## 2. 对当前 vLLM 现状的结论

### 2.1 已经形成的能力

当前主线已经具备：

- `V1 scheduler`
- hash-based prefix caching
- `Hybrid KV Cache Manager`
- `SimpleCPUOffload`
- `LMCache/FlexKV/NIXL/Mooncake` 等 connector
- disaggregated prefill / external KV transfer 相关基础设施

### 2.2 尚未闭环的核心问题

调研后我认为最关键的几个缺口是：

1. **多级 KV 能力分裂**
   - GPU、本地 offload、remote connector 还没有真正统一成一个系统
2. **多实例 / DP 下缺少 KV 感知路由**
   - 官方文档自己承认这是重要方向
3. **prefix cache 收益不稳定**
   - 有 HoL、污染、可观测性不足、热前缀保留不足等问题
4. **RAG / Agent workload 的非严格前缀复用不足**
   - APC 假设不总适合现代 workflow
5. **Hybrid 模型的 KV 抽象仍偏刚性**
   - group-specific dtype/layout/block size 仍是明确缺口

## 3. 对 GitHub 社区进展的结论

从 release、PR、issue 和 production-stack roadmap 看，社区最近一年的主线非常清楚：

### 3.1 社区正在把 vLLM 推向多级 KV 与外部 KV 系统

明显信号包括：

- `LMCacheConnectorV1`
- `FlexKVConnectorV1`
- `SimpleCPUOffloadConnector`
- `multiple KV cache groups`
- `kv connector metadata` 扩展

这说明“KV 不再只在本地 GPU 上”已经是官方认可方向。

### 3.2 社区正在意识到 routing 与 KV 复用的联系

明显信号包括：

- `data_parallel_deployment.md` 明确提到 KV-cache-aware logic
- production-stack 已有 session routing
- prefix-aware routing 被公开标成 WIP
- roadmap 明确写了 KV-cache-aware and prefix-aware routing

这说明“入口层路由”已经进入主线视野。

### 3.3 社区已经进入 APC 的第二阶段问题

问题不再只是“开不开 prefix cache”，而是：

- prefix hit 为什么低
- APC 为什么会污染缓存
- APC 为什么会带来 HoL blocking
- APC 为什么难监控

这说明 prefix cache 已经从 feature 进入 runtime quality 问题。

## 4. 最终保留的五条候选主线

## 4.1 TierKV

### 定位

多级精确 KV hierarchy + scheduler 协同。

### 为什么保留

- 最贴近 `vLLM` 当前主线演进
- 最像系统论文
- 最容易同时打 throughput 和 latency

### 当前排序

**第 1**

## 4.2 AffinityKV

### 定位

DP / 多实例 / router 层的 KV 感知路由。

### 为什么保留

- 官方文档已明确承认价值
- production-stack 有明确 roadmap
- 更贴近真实线上 serving

### 当前排序

**第 2**

## 4.3 QoSKV

### 定位

围绕 APC 的低时延、低污染、可观测和热前缀保留。

### 为什么保留

- 社区问题密集
- 兼容现有 vLLM 很自然
- 容易快速打出 P99 与 TTFT 收益

### 当前排序

**第 3**

## 4.4 SegmentKV

### 定位

面向 RAG / Agent / tool-use 的精确分段复用。

### 为什么保留

- 很符合现代 workload
- 对 prefix caching 的边界切得很准

### 当前排序

**第 4**

## 4.5 HeteroKV

### 定位

面向 hybrid model 的异构 KV 虚拟内存。

### 为什么保留

- 新意最强
- 与社区 RFC / local code 注释高度一致

### 当前排序

**第 5**

不是因为方向弱，而是因为第一版实现与验证成本最高。

## 5. 为什么最后推荐 TierKV

我最后把 `TierKV` 放在第一位，原因不是“它最炫”，而是它同时满足五个条件：

1. **问题足够大**
2. **兼容当前 vLLM**
3. **证据链最完整**
4. **实验空间最充足**
5. **论文故事最闭环**

它的故事线非常清楚：

- 当前 `vLLM` 已经有多条 KV 能力
- 但这些能力没有统一成系统
- 结果就是收益不稳定、状态分散、策略碎片化
- 新系统通过统一抽象、调度协同和一致性协议，把这些能力整合成一个多级 KV hierarchy

这就是很典型、也很稳的系统论文结构。

## 6. 为什么 AffinityKV 这轮优先级上升

这轮调研里，我对 `AffinityKV` 的看法比之前更积极，主要因为：

- 官方 DP 文档直接写了 KV-aware load balancing 的价值
- production-stack 已经有相关 RFC 与 roadmap
- 它非常贴近真实线上部署

如果你们实验环境允许做多实例或 DP 集群，这条线完全可能成为与 `TierKV` 并列的强主线。

## 7. 为什么 QoSKV 也值得认真考虑

如果目标不是优先追求“系统范围最大”，而是更看重：

- TTFT
- P99
- 交互稳定性
- APC 的真实收益

那 `QoSKV` 其实很务实，而且更容易先做出一版结果。

它的风险在于：必须讲成一个统一系统问题，而不是零散 patch。

## 8. 本轮调研最终给出的排序

综合排序如下：

1. `TierKV`
2. `AffinityKV`
3. `QoSKV`
4. `SegmentKV`
5. `HeteroKV`

## 9. 如果后续要继续推进，最合理的路线

### 9.1 最稳路线

直接选 `TierKV`，然后在实现或论文中吸收：

- `QoSKV` 的热前缀保留、cache state metrics
- `AffinityKV` 的外部扩展实验

### 9.2 更偏线上部署的路线

选 `AffinityKV` 做主线，再吸收：

- `QoSKV` 的可观测性
- `SegmentKV` 的 workflow / context metadata

### 9.3 更偏低延迟的路线

选 `QoSKV` 做主线，集中火力把 APC 的副作用讲透。

## 10. 给后人的一句建议

这轮调研最核心的结果不是“只剩一个正确答案”，而是：

- `vLLM` 的 KV 方向已经进入系统整合阶段
- 候选主线应该优先围绕“统一系统、部署协同、runtime 质量”展开

如果后面社区再往前推进，排序可以变化，但这条判断大概率仍成立。
