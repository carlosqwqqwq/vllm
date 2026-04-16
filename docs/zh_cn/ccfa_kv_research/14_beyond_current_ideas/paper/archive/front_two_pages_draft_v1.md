# FalseNegativeKV Front Two Pages Draft V1

更新日期：2026-04-15

## 1. 使用说明

这份文档的目标不是做终稿排版，而是把论文前两页最核心的四块内容先拼起来：

1. working title
2. 当前最稳的 abstract
3. Introduction
4. `Background and Motivation`

当前版本严格遵守 measurement-first 口径：

- 强写问题和测量
- 清楚写 `TwinPathServe` 的设计方向
- 但不提前写 recovery 闭环已经完成

## 2. Working Title

```text
FalseNegativeKV: Contract-Aware Recovery of False-Negative Exact Hits in vLLM
```

## 3. Current Front-Matter Draft

```text
Abstract

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

1 Introduction

Exact-prefix reuse has become a foundational optimization in modern LLM serving
systems. Once multiple requests share a long system prompt, tool definition
prefix, or stable conversational context, reusing precomputed prefix state can
substantially reduce time-to-first-token and improve effective throughput.
Systems such as vLLM therefore treat prefix caching and exact reuse as a core
runtime capability rather than as an optional optimization.

However, modern serving workloads increasingly mix that shared prefix structure
with heterogeneous API contracts, observation requirements, and backend
execution paths. A request may share the same semantic prefix as a previous
request while differing only in whether it asks for `prompt_logprobs`, whether
it is routed through a reuse-friendly path or a throughput-oriented fast path,
or whether it is served under a different runtime compatibility class.
Existing runtimes typically observe only physical hits: a request either reuses
prefix state or it does not. They do not explicitly model the gap between
requests that should have reused state in principle and requests that actually
reuse state in execution.

This gap matters because many misses are not caused by the absence of reusable
state. Instead, the state exists, but cannot be consumed under the request’s
current reuse contract. Consider a family of requests with a long shared system
prompt. Under ordinary generation, later requests may obtain large exact-prefix
hits. Under `prompt_logprobs`, however, the same family can be conservatively
forced to skip prefix-cache reads, collapsing physical reuse to zero. Likewise,
when a reuse-friendly backend class and a throughput-oriented backend class are
split across execution pools, a family that is a clear exact hit in one class
may become a complete miss in the other. In both cases, the request is a
logical hit but not a physical hit.

We argue that these behaviors should not be understood as isolated feature bugs.
They expose a broader runtime pathology that current serving systems leave
largely unstructured: false-negative exact hits. A false-negative exact hit is a
request that is logically an exact reuse opportunity but fails to materialize as
a physical hit before computation begins. The central issue is not that prefix
state is unavailable, but that the runtime lacks an explicit contract relating
producer-side reusable state to consumer-side execution requirements. Without
that contract, systems fall back to conservative skip rules, backend-specific
workarounds, or ad hoc compatibility patches.

This paper takes a measurement-first step toward that problem. We formalize the
distinction among logical hits, physical hits, false-negative hits, and
bridgeable misses, and we build a request-level measurement surface in vLLM to
make these phenomena observable in real workloads. Our measurements show two
strong sources of false-negative exact hits already present in practice:
observation mismatch introduced by `prompt_logprobs`, and backend-compatibility
mismatch across reuse-oriented and throughput-oriented execution classes. These
findings motivate `TwinPathServe`, a contract-aware runtime design centered on
reuse contracts, false-negative reasoning, and bridge operators for recovering
missed exact reuse.

In summary, this paper makes the following contributions.

First, we identify false-negative exact hits as a previously unstructured
runtime pathology in LLM serving, and formalize logical hits, physical hits,
false-negative hits, and bridgeable misses as measurement-first systems
concepts.

Second, we build a request-level measurement surface in vLLM and show that
false-negative exact hits already arise from at least two strong sources in
practice: observation mismatch introduced by `prompt_logprobs`, and
backend-compatibility mismatch across reuse-oriented and throughput-oriented
execution classes.

Third, we present `TwinPathServe`, a contract-aware runtime design centered on
reuse contracts, false-negative reasoning, prefix-family routing, and bridge
operators, establishing a systematic path from measurement to recovery.

2 Background and Motivation

Today’s exact-prefix reuse mechanisms are primarily organized around physical
events: a runtime hashes prefix state, checks whether matching cache entries are
available, and reports reuse only when that state can be consumed under the
request’s current execution path. This physical-hit-centric view works well when
requests are homogeneous and the runtime contract between producer state and
consumer behavior is simple. It becomes increasingly brittle, however, once
serving workloads mix long shared prefixes with heterogeneous API contracts,
observation requirements, and backend execution classes.

The key problem is that reusable state may exist even when the runtime does not
materialize a physical hit. In our current vLLM measurements, a long-prefix
family that yields 1,088 physical-hit tokens under ordinary generation
collapses to 0 physical-hit tokens when the only change is enabling
`prompt_logprobs`. Likewise, a family that yields the same 1,088-hit reuse
behavior inside a `FLASHINFER` reuse-oriented execution class collapses to 0
when it is probed from a `FLASH_ATTN + batch_invariant` execution class. These
are not cases where reusable prefix state is missing. They are cases where the
runtime cannot consume that state under the request’s effective reuse contract.

This observation exposes a blind spot in current serving abstractions. Existing
systems optimize physical hits but rarely model logical hits: requests whose
prefix state should be reusable in principle, given what the system has already
computed elsewhere. Without that distinction, runtimes fall back to conservative
skip rules, backend-local workarounds, and per-feature compatibility patches.
What appears as a collection of unrelated misses is better understood as a
single systems problem: false-negative exact hits.

From this perspective, the central missing abstraction is a reuse contract that
connects producer-side reusable state to consumer-side execution requirements.
Once that contract is made explicit, the runtime can reason about why a logical
hit fails to become a physical hit, which misses are bridgeable, and which
execution path should be chosen to preserve reuse rather than accidentally
destroy it. This is the design motivation behind `TwinPathServe`: not merely a
faster prefix cache, but a contract-aware runtime structure for turning
measurement-backed logical evidence into systematic recovery decisions.
```

## 4. 这份组合稿的定位

这份组合稿当前适合做三件事：

1. 作为论文主文档的前两页初稿
2. 作为和导师或合作者对齐 narrative 的版本
3. 作为后续往 `Design` 和 `Evaluation` 章节扩写时的固定前缀

## 5. 当前不应怎么用

这份草稿当前还不适合直接当投稿版第一页，因为：

1. 摘要仍然是 measurement-first 版本
2. 第三条贡献仍是 design-forward 表述
3. 还没有把 `H3/H4` 的 recovery 结果写进去

因此，当前最合理的使用方式是：

- 把它当作“前两页 narrative 基底”
- 后续只增量升级结果句和第三条贡献
