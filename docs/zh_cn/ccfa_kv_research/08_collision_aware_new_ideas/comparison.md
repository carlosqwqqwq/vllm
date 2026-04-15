# 三个新方向的横向对比

## 1. 总推荐

当前最推荐顺序：

1. `ShapeKV`
2. `ElasticPageKV`
3. `ControlPlaneKV`

更准确地说，最有希望的论文主线不是三选一，而是：

> ShapeKV as the main thesis, ElasticPageKV as a granularity extension, and ControlPlaneKV as the cache-readiness fast path.

也就是：**前缀命中之后，vLLM 不应只问“命中了多少 KV”，还应问“这些命中会让 residual work 以什么形状执行，以及控制面能否及时、低开销地把这些信息交给 scheduler”。**

## 2. 关键维度对比

| 维度 | ShapeKV | ElasticPageKV | ControlPlaneKV |
| --- | --- | --- | --- |
| 核心问题 | APC hit 打碎执行形状 | 逻辑命中粒度与物理 page 粒度强绑定 | 高命中后 CPU 控制面可能成为瓶颈 |
| 最接近工作 | SGLang cache-aware scheduling、vLLM issue #7883、PAT | vLLM APC RFC、PagedAttention、Jenga、Strata | vLLM V1、SGLang v0.4、LMCache |
| 重合风险 | 中 | 中高 | 高 |
| 可防守差异 | graph/cascade/padding shape co-design | logical micro-page 与 physical macro-page 解耦 | exact verified APC fast path，不是泛泛 CPU 优化 |
| 实现难度 | 中 | 中高 | 低到中 |
| 是否需要改 kernel | 不一定 | Phase 3 可能需要 | 不需要 |
| 对 vLLM 主线适配 | 高 | 中 | 高 |
| 轻量级 | 高 | 中 | 高 |
| 普适性 | 高 | 中高 | 中 |
| 高吞吐潜力 | 高 | 中高 | 中 |
| 低延迟潜力 | 高 | 中 | 中高 |
| OSDI 潜力 | 高 | 中高 | 中低 |
| 最关键证明 | APC 高命中会降低 graph/cascade 效率 | block/page 粒度冲突是显著瓶颈 | CPU control path 在 high-hit workload 下显著占比 |

## 3. 为什么 ShapeKV 排第一

`ShapeKV` 的优势是它避开了最拥挤的 KV cache 赛道：

- 它不是新的 offload 层。
- 它不是新的 radix tree。
- 它不是新的 KV 压缩。
- 它不是 speculative KV 事务。
- 它不是单纯 partial block。

它研究的是一个更系统的问题：**cache hit 改变了 GPU runtime 的执行形状，而现有系统没有把这个变化作为一等公民。**

这很像 OSDI 喜欢的切口：一个跨层边界问题，单独优化每层都做得不错，但组合后出现新瓶颈。

## 4. 为什么 ElasticPageKV 只能排第二

`ElasticPageKV` 的概念很强，但有两个现实风险：

- vLLM APC RFC 已经讨论过 partial block / block table 相关问题。
- Strata、Jenga 等工作已经在讲 KV layout、heterogeneous page size、hierarchical cache。

所以它不能讲成 “我们第一次支持更细粒度 prefix caching”。正确讲法必须是：

> We decouple exact reuse granularity from physical execution granularity, so that vLLM can choose large pages for execution and transfer without sacrificing fine-grained prefix reuse.

如果 Phase 0 profiling 证明 block size 从 16 扩到 32/64 能明显改善 kernel/transfer，但会显著伤害 APC 命中精度，那么 ElasticPageKV 会变得很有价值。否则它可能只是一个工程优化。

## 5. 为什么 ControlPlaneKV 不建议单独主投

`ControlPlaneKV` 的最大问题是论文边界不够独特。vLLM V1、SGLang v0.4 和 LMCache 都在不同层面追求低控制面开销或 cache-aware execution。审稿人很可能会问：

> 这不就是更快的 scheduler / hash / lookup 吗？

因此它的正确定位是支撑组件：

- 为 ShapeKV 提供 cache readiness pipeline。
- 为 ElasticPageKV 提供 micro-page lookup 的低开销实现。
- 为多租户 exact prefix cache 提供 fast candidate + strong verification。

如果后续 profile 证明 high-hit workload 下 CPU control path 占 TTFT 的 30% 以上，它可以重新升级为独立论文；否则不建议主投。

## 6. 最小验证路线

### Step 1：验证 ShapeKV 是否成立

构造 shared-prefix workload：

- shared prefix 长度：512 / 2K / 8K。
- suffix 长度分布：固定、均匀、长尾。
- output 长度：1 / 16 / 64。
- 并发：低、中、高。

记录：

- APC hit ratio。
- CUDA Graph replay rate。
- graph fallback reason。
- padding token ratio。
- cascade attention enable rate。
- TTFT p99。

判定标准：

- 如果 baseline APC 高命中但 graph replay 低、padding 高或 cascade miss 多，ShapeKV 成立。
- 如果这些指标已经很好，ShapeKV 降级。

### Step 2：验证 ElasticPageKV 是否值得

做 block size sweep：

- block size：8 / 16 / 32 / 64。
- prompt shared boundary offset：0 / 1 / 4 / 8 / 15 / 31。
- 对比 full-block reusable tokens 与 exact shared tokens。

判定标准：

- 如果大 block 对执行/transfer 有收益但损害 reuse precision，ElasticPageKV 成立。
- 如果 block size 16 已经足够且 tail loss 很小，ElasticPageKV 降级。

### Step 3：验证 ControlPlaneKV 是否需要

做 CPU profile：

- hash time。
- prefix lookup time。
- scheduler step time。
- connector query time。
- Python overhead / lock / GC。

判定标准：

- 如果 high-hit short-output 下 CPU control path 是 TTFT 主因，ControlPlaneKV 升级。
- 否则仅保留 batch lookup 和 cache readiness pipeline。

## 7. 最终论文包装建议

最稳的论文题目不应叫 `ElasticPageKV` 或 `ControlPlaneKV`，而应该围绕 `ShapeKV`：

> ShapeKV: Cache-Hit-Aware Shape Stabilization for High-Throughput LLM Serving

贡献可以写成三条：

1. 揭示 exact prefix-cache hits 与 CUDA Graph / padding / cascade attention 之间的负交互。
2. 提出 shape signature 和 graph-stable residual scheduling。
3. 在 vLLM 上实现轻量原型，并证明在高复用 RAG/Agent/chat workload 下同时提升 throughput 与 p99 TTFT。

ElasticPageKV 和 ControlPlaneKV 可以作为章节或 ablation 出现：

- ElasticPageKV：granularity-aware extension。
- ControlPlaneKV：cache-readiness fast path。

这样比三条线分开投稿更安全，也更像一篇完整 CCF-A 系统论文。
