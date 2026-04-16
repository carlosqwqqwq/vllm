# FalseNegativeKV Abstract Drafts

更新日期：2026-04-15

## 1. 使用说明

这份文档保留两版摘要：

- Draft A
  - 当前最稳的版本
  - 适用于我们只有 measurement 证据、还没有完整 `H3/H4` 闭环的时候
- Draft B
  - 目标版本
  - 只在 recovery prototype 和端到端结果出来之后使用

当前默认使用 Draft A。

## 2. Draft A：当前最稳的摘要

### 2.1 写作意图

这版摘要的目标是：

- 强写问题
- 强写 measurement
- 写出 `TwinPathServe` 的系统方向
- 但不提前声称完整恢复结果已经成立

### 2.2 English Abstract Draft A

```text
Exact-prefix reuse has become a core optimization in modern LLM serving
runtimes, yet today’s workloads increasingly mix long shared prompts with
heterogeneous API contracts, observation requirements, and backend fast paths.
As a result, many requests that are logical exact hits do not become physical
hits at runtime: the reusable state exists, but the runtime lacks an explicit
contract that connects producer state to consumer-side reuse requirements. We
identify this phenomenon as false-negative exact hits and present a
measurement-first methodology that formalizes logical hits, physical hits,
false-negative hits, and bridgeable misses in vLLM. Using request-level traces,
we show that false-negative exact hits arise from at least two strong sources in
practice: observation mismatch introduced by `prompt_logprobs` requests, and
backend-compatibility mismatch across reuse-friendly and throughput-oriented
execution paths. Based on these findings, we present TwinPathServe, a
contract-aware runtime design centered on reuse contracts, false-negative
reasoning, and bridge operators for recovering missed exact reuse. Our results
argue that exact-prefix reuse should be treated as a contract-aware runtime
capability rather than a collection of scattered compatibility patches.
```

## 3. Draft B：full-thesis 目标摘要

### 3.1 使用前提

只有同时满足下面条件后，才允许使用这版摘要：

1. `prompt_logprobs selective replay` 已实现并有正收益
2. `TwinPathServe` 双池 route 已实现并有正收益
3. `H3/H4` 已形成稳定结果

### 3.2 English Abstract Draft B

```text
Exact-prefix reuse is a foundational optimization in modern LLM serving, but
current runtimes optimize only physical cache hits and fail to model logical
hits that are lost before computation begins. In mixed-API, tool-rich, and
backend-heterogeneous workloads, reusable prefix state often exists but cannot
be consumed because the runtime lacks an explicit reuse contract between
producer state and consumer requirements. We identify this phenomenon as
false-negative exact hits, and present a measurement-first framework that
formalizes logical hits, physical hits, false-negative hits, and bridgeable
misses in vLLM. We design TwinPathServe, a contract-aware reuse runtime built
around reuse contracts and bridge operators, and instantiate it with two
mechanisms: `prompt_logprobs` selective replay and batch-invariant twin-pool
routing. Across representative workloads, we show that false-negative misses
are prevalent, that at least two mismatch classes can be recovered within the
same framework, and that the recovered hits translate into significant TTFT
improvements with controlled throughput regression. Our results suggest that
exact reuse should be treated as a contract-aware runtime capability rather than
as a collection of per-feature fixes.
```

## 4. 当前推荐版本为什么是 Draft A

当前更推荐 Draft A，而不是 Draft B，原因很具体：

1. Draft A 已经完整表达了问题、测量和系统方向
2. Draft A 不要求我们现在就拿出恢复收益曲线
3. Draft B 的第四句和第五句默认 `H3/H4` 已经成立
4. 现在直接使用 Draft B，会把 `W5` 的证据写得过于超前

## 5. 当前最需要保留的关键词

无论选哪版摘要，下面这组关键词都建议保留：

- `logical hit`
- `physical hit`
- `false-negative exact hit`
- `reuse contract`
- `bridgeable miss`
- `TwinPathServe`

这组词决定了论文像一篇“系统问题论文”，而不是“兼容性 patch 论文”。

## 6. 之后如何升级

后续升级摘要时，建议只改下面三处，不重写整篇：

1. 把第四句从“we present TwinPathServe”升级为“we implement TwinPathServe”
2. 在第五句补入两类 mechanism 的 recovery 结果
3. 把“argue that”升级为“show that”
