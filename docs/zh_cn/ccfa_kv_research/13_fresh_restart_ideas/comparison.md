# 新三方向总对比

更新日期：2026-04-15

| 方向 | 核心问题 | 当前创新性判断 | 与最新论文重合风险 | vLLM 落地性 | 最主要风险 | 当前优先级 |
| --- | --- | --- | --- | --- | --- | --- |
| `WritePathKV` | 新生成 KV 何时值得发布、导出、offload 或写入全局层 | 高 | 中低 | 高 | 如果只剩 heuristic admission，可能被看成 policy patch | 1 |
| `ObservationSidecarKV` | prefix reuse 是否能从 generation-only 扩展到 mixed API | 中高 | 低到中 | 高 | 如果只优化 `prompt_logprobs`，容易像 feature patch | 2 |
| `DecisionSidecarKV` | prefix 共享时，能否复用 grammar / schema / sampling 决策状态 | 高 | 中 | 中高 | 更容易撞上 `SIMPLE / XGrammar 2` 一类 decision-plane 论文 | 3 |

## 1. `WritePathKV`

最核心的差异是：

- 不是继续研究 cache read path；
- 而是研究 **cache write path / publication path**。

它切的问题包括：

- 哪些块值得写出去；
- 哪些块只该本地保留；
- 哪些块的写放大大于未来收益；
- local APC、CPU offload、connector、disagg 之间怎样统一 publication policy。

为什么它更像系统论文：

- 因为它研究的是一个跨层资源决策；
- 不只是局部命中率优化。

## 2. `ObservationSidecarKV`

最核心的差异是：

- 不再把 prefix reuse 理解成“只复用 KV”；
- 而是问：对于 `prompt_logprobs`、pooling、token scoring，这些 API 所需的观测结果里，有没有极小的一部分也可以 sidecar 化复用。

为什么它有现实基础：

- `vLLM` 里 `prompt_logprobs` 和部分 pooling 任务今天确实会主动绕过 prefix cache；
- 说明 generation-only APC 语义已经不够覆盖真实服务接口。

## 3. `DecisionSidecarKV`

最核心的差异是：

- 不再停留在 data-plane state；
- 而是把复用对象延伸到 decision plane。

典型对象包括：

- grammar bitmask / schema state
- structured output validation state
- 某些 prefix-conditioned sampling 辅助状态

为什么风险更高：

- 因为最新 `SIMPLE`、`XGrammar 2` 已经把 decision plane 做热；
- 需要非常小心地把边界讲成“跨请求 prefix-conditioned decision-state reuse”，而不是一般性的 structured generation engine。

## 4. 当前推荐

如果现在只能选一条继续深挖，我的建议仍然是：

1. `WritePathKV`
2. `ObservationSidecarKV`
3. `DecisionSidecarKV`

最根本的理由是：

- 第一条最像“换问题”；
- 第二条最像“真实缺口”；
- 第三条最像“高风险高收益的备选”。 
