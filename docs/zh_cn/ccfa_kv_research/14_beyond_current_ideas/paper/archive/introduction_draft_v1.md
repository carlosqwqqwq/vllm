# FalseNegativeKV Introduction Draft V1

更新日期：2026-04-15

## 1. 使用说明

这是一版“当前最稳”的引言草稿。

它的写作策略是：

- 把问题写厚
- 把 measurement 写实
- 把 `TwinPathServe` 写成系统回应
- 但不提前声称完整端到端恢复已经闭环

如果后续 `H3/H4` 完成，这版引言不需要推倒重写，只需要在贡献和结果预告部分升级几句即可。

## 2. English Introduction Draft

```text
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

Our work makes three contributions. First, we identify false-negative exact hits
as a previously unstructured runtime problem in LLM serving and provide a
measurement-first methodology for reasoning about logical and physical reuse.
Second, we show why a contract-aware view of reuse is more explanatory than a
collection of scattered per-feature fixes. Third, we present `TwinPathServe`,
the first runtime design in our setting that explicitly organizes reuse around
producer-consumer contracts and bridgeable mismatches, creating a path from
measurement to systematic recovery.
```

## 3. English Contributions Subsection Draft

```text
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
```

## 4. English Background and Motivation Draft

```text
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

## 5. 这版前两页为什么这样写

### 5.1 Introduction 主段落

引言主体仍然保持六段结构，因为这是当前最稳的 reviewer 进入方式：

1. prefix reuse 是基础设施
2. 现代 workload 让 reuse contract 复杂化
3. 同一 family 会从高命中掉到 0 命中
4. 这不是 feature bug，而是 runtime pathology
5. 我们先 measurement，再引出 `TwinPathServe`
6. 最后用贡献段收束

### 5.2 Contributions 小节

这里我没有直接复用 `paper_storyline_fnkv.md` 里的 full-thesis 第三条贡献，而是写成“当前可安全使用”的版本。

原因很直接：

- 现在已经能稳写的问题表征和测量贡献
- 也能稳写 `TwinPathServe` 的设计方向
- 但还不能提前声称 recovery 和端到端收益已经完整闭环

所以当前 contributions 小节强调的是：

1. identify
2. measure
3. present the design path

而不是：

1. identify
2. implement
3. win end-to-end

### 5.3 Background and Motivation 小节

这一节故意写成四段，而不是马上展开更多 taxonomy。

写法上固定遵循下面的推进顺序：

1. 先讲为什么 physical-hit-centric runtime 不够
2. 再给两个当前最强的具体例子
3. 再把它们统一提升成 `false-negative exact hit`
4. 最后落到 `reuse contract` 和 `TwinPathServe`

这样写的好处是：

- 先把问题讲具体
- 再把抽象抬起来
- 不会一上来就显得过度概念化

### 5.4 为什么在这里放具体数字

这一版 `Background and Motivation` 故意放入了当前预实验里的 `1,088 -> 0` 例子。

原因不是为了炫数字，而是为了把下面这件事写得不可回避：

- 我们讨论的不是“命中略微变差”
- 而是“逻辑 exact hit 被 runtime 直接打成 0”

这类数字放在前两页里，比纯概念描述更容易建立问题严重性。

当然，后续如果换模型、换 workload family 或换图表表达，也可以把这两个具体数字改成更抽象的说法；但当前版本保留它们更有说服力。

## 6. 后续怎么升级成 full-thesis 前两页

等 `H3/H4` 结果出来以后，这一版建议只升级四处：

1. 引言第五段最后一句
   - 从 `motivates TwinPathServe`
   - 升级为 `we implement and evaluate TwinPathServe`
2. Contributions 第三条
   - 从 `present TwinPathServe`
   - 升级为 `implement TwinPathServe with two bridge operators and show ...`
3. Background and Motivation 最后一段
   - 从 “design motivation”
   - 升级为 “validated runtime architecture”
4. 在引言或摘要里补入端到端结果预告
   - 例如 `These recovered hits translate into substantial TTFT gains`

## 7. 当前最需要警惕的点

这版前两页当前仍有两条必须守住的边界：

1. 不把 `W5` 写成已经在线原生分类成功
2. 不把 `TwinPathServe` 写成已经用完整实验闭环证明最优

只要守住这两条，这版草稿就能作为后续论文前两页的稳定基底。
