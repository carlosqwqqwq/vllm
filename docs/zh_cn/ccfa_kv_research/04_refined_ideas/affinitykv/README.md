# AffinityKV：面向 DP 与多实例部署的 KV 感知路由

## 1. 一句话定位

`AffinityKV` 的目标是：让请求在进入 `vLLM` 之前，就优先被送到 **最可能命中已有 KV cache、且不会造成严重排队与显存压力** 的 DP rank 或 serving instance，从而提升真实部署中的 TTFT、hit rate 与系统吞吐。

## 2. 为什么这条线突然变得很重要

### 2.1 官方文档已经明确承认这个缺口

`vLLM` 官方的 DP 部署文档已经明确说过：

- 每个 DP engine 都有独立 KV cache
- prefix caching 的收益可以通过 intelligent routing 放大
- 当前 internal DP load balancing 仍主要基于 running / waiting queues
- 未来可以加入 KV-cache-aware logic

这说明问题不是想象出来的，而是官方路线已经承认但尚未闭环。

### 2.2 production-stack 也在把它当成路线图

production-stack 当前已经有：

- session-based routing
- prefix-aware routing（WIP）
- KV-cache-aware routing 的讨论

而且 roadmap 里还在继续推进：

- KV-cache-aware and prefix-aware routing
- predictive routing
- intelligent routing for agent workloads

所以这条线的一个很强特点是：**社区认可度高，落地价值强。**

## 3. 这条线到底在解决什么问题

可以把问题定义成：

> 在多 DP rank 或多实例部署中，独立实例各自维护独立 KV cache。若路由仅依据队列长度、round-robin 或简单 session sticky，则大量本可命中的请求会被送到错误实例，导致重复 prefill、TTFT 上升与缓存碎片化。如何设计一个低开销、KV 感知、可部署的路由机制，把请求尽量送到“对它最有利”的实例？

换句话说，它做的是 **把 KV 复用从单实例内，扩展到整个 serving fabric 的入口层。**

## 4. 核心设计思路

## 4.1 不是只做 prefix-aware，而是做 cache-affinity-aware

只看“是否共享某个前缀”还不够。真正的路由分数应该同时考虑：

- 前缀或上下文复用概率
- 当前实例的队列长度
- 当前实例的 KV 压力
- 该请求是否属于某个会话 / workflow
- 当前实例是否已经持有相关 remote/local KV

可以设计一个统一评分函数：

`score = reuse_gain - queue_penalty - memory_penalty - transfer_penalty`

这样才能避免“命中高，但排队更久”的反直觉路由。

### 4.2 核心机制一：轻量级 cache sketch / telemetry

router 不应该直接拿完整 token ids 去做昂贵匹配。更现实的方式是让每个实例周期性上报轻量摘要：

- hot prefixes / hot segments 的 sketch
- prefix cache usage
- active blocks 与 reusable blocks 的比例
- pending / running requests
- 最近一段时间的 hit / miss 统计

这样路由器看到的是“可比对的缓存状态摘要”，而不是底层 block 细节。

### 4.3 核心机制二：会话亲和性与 workflow 亲和性

很多线上请求天然不是独立的：

- 多轮聊天
- Agent workflow
- RAG 后续追问
- 工具调用链

如果只做 session sticky，虽然简单，但会错过“同 workflow 不同 session key”的跨请求复用；如果完全不 sticky，又会丢掉前面已经形成的 cache affinity。

更合理的设计是：

- 第一层：`session / workflow affinity`
- 第二层：`cache-aware reranking`
- 第三层：`queue / pressure guardrail`

### 4.4 核心机制三：KV 感知的溢出与回退策略

如果某个实例很适合命中 KV，但已经排队过长，路由器需要决定：

- 继续等这个实例
- 转发到次优实例
- 在 remote KV 存在时触发跨实例迁移 / 拉取

也就是说，`AffinityKV` 不只是“选最像的实例”，还需要一套 **收益与时延折中** 的回退策略。

## 5. 在 vLLM 里怎么落地

## 5.1 两种兼容路径

这条线非常灵活，可以有两种落地方式。

### 路径 A：先做 production-stack / 外部 router

优点：

- 对 `vLLM` 核心侵入小
- 最容易搭出多实例实验
- 很适合先验证 routing algorithm

需要 `vLLM` 配合的点主要是：

- 导出更细粒度的 KV telemetry
- 统一 prefix / KV 统计语义

### 路径 B：做 vLLM 内部 DP load balancer 增强

优点：

- 结果更“原生”
- 更适合把贡献写成 vLLM 主线增强

缺点：

- 会碰 API server、coordinator、engine stats 收集

## 5.2 兼容性判断

我给这条线的兼容性评估是：**高到中高**。

原因是：

- 文档已经承认 DP LB 未来可纳入 KV awareness
- `VllmConfig.needs_dp_coordinator` 已明确 coordinator 会做 queue stats 收集
- production-stack 已有 session routing 和 WIP prefix-aware routing

所以这条线更像“把现有 loose ends 做实”，而不是另起炉灶。

## 6. 论文创新点应该怎么收敛

建议把创新点控制在三条：

### 6.1 低开销 cache-affinity telemetry

核心不是暴露完整缓存，而是设计 **可用于路由决策但不会过重** 的状态摘要。

### 6.2 复用收益与排队代价联合建模

区别于只看 session sticky 或 round-robin，`AffinityKV` 的核心是联合建模：

- reuse gain
- queue delay
- memory pressure
- transfer fallback

### 6.3 可部署的分层路由策略

把：

- session/workflow affinity
- prefix/KV-aware rerank
- queue-pressure guardrail

整成一套真正能部署的入口层策略。

## 7. 这条线最适合的实验

### 7.1 部署形态

- 单机多 DP rank
- 多机多实例
- production-stack router

### 7.2 workload

- 多轮会话
- agent workflow
- prefix-heavy 客服 / 助手请求
- RAG 追问

### 7.3 baseline

- round-robin
- queue-length-only
- session sticky
- production-stack 当前 router

### 7.4 指标

- TTFT
- P95 / P99 TTFT
- prefix hit rate
- per-instance cache skew
- request throughput
- remote transfer bytes

## 8. 风险与边界

### 8.1 风险

- 需要真实多实例环境，单卡环境说服力有限
- telemetry 太弱会导致路由收益不稳定
- telemetry 太重又会引入开销

### 8.2 边界

第一版不建议上来就做：

- 很复杂的学习路由器
- 强依赖外部数据库的全局一致性系统
- 复杂请求迁移

先把轻量状态摘要 + 启发式路由做扎实。

## 9. 这条线什么时候比 TierKV 更适合

如果你们具备下面条件，我会认真考虑把它升到第一优先级：

- 有多实例或集群环境
- 更关心线上端到端部署，而不是单机内核优化
- 有多轮对话 / agent workflow / API gateway 场景
- 希望论文更偏 serving system / routing system

## 10. 结论

`AffinityKV` 是我第二轮调研后明显抬高优先级的一条线。它的优势不是“局部小修”，而是：

- 官方文档已经承认价值
- production-stack 已经公开 roadmap
- 与当前真实部署痛点高度一致

如果你们的实验环境允许做多实例或 DP 部署，它会是一个非常强的候选主线。
