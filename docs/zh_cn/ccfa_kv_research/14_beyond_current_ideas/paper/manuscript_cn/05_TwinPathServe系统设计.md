# 5 TwinPathServe 系统设计

基于第四章的统一抽象，本章给出 `TwinPathServe` 的系统实例。该系统以 `correctness-aware reuse control plane` 为核心，面向第三章已经证实的两类 mismatch：其一是 `prompt_logprobs` 引起的 observation mismatch，其二是 backend compatibility class 切换引起的 backend mismatch。系统目标不是覆盖所有 prefix miss，也不是把任意逻辑命中都恢复成物理命中，而是在 `soundness-first` 原则下，为两类强失效提供边界清晰且可验证的恢复路径。

`TwinPathServe` 的重点不在单个 bridge，而在于把“是否恢复、恢复到哪条路径、失败时如何退回”组织为显式的运行时控制逻辑。双池执行面只是这一控制面的第一版系统实例。更根本的设计在于将 `reuse oracle -> contract gate -> realization policy -> bridge operator -> route/admission` 组织为一条可验证的控制链，用于决定哪些 reference-compatible exact reuse 可以被兑现、为何当前请求未能兑现，以及系统在何种条件下应保守拒绝恢复。

## 5.1 设计目标与非目标

`TwinPathServe` 的第一版目标非常克制。它只试图恢复两类已经被测量证明足够强、且能够共享统一恢复语言的 `false-negative exact hit`：一类来自 `prompt_logprobs` 的 observation mismatch，另一类来自 reuse-friendly backend class 与 throughput-oriented backend class 之间的 backend mismatch。前者对应第三章的 `W1`，后者对应第三章的 `W5`。系统不试图在第一版就覆盖所有 carrier mismatch、pooling partial-prefill incompatibility、hidden cache、distributed cache pool 或 arbitrary-position reuse，因为这些方向要么缺少当前正文中的强恢复证据，要么会显著扩张命题边界。

此外，`TwinPathServe` 从一开始就采用 `soundness-first` 的设计原则。它不主张“所有 logical hit 都应被恢复”，而只允许满足 `SafeReuseContract` 的请求进入 bridge 路径；一旦 identity、execution、carrier 或 observation 条件无法建立，系统就必须 fallback 到 baseline 支持路径。因此，`TwinPathServe` 的第一版目标不是最大化 bridge 覆盖面，而是在可验证的前提下把一部分 false-negative exact hit 安全兑现为物理命中。

这一定义也决定了本章的非目标。第一版明确不做三件事：第一，不做跨不兼容 backend class 的直接 state consumption；第二，不做执行中途的 request migration；第三，不把 `carrier_incompatibility` 与更广的 pooling 相关问题一并拉入主恢复面。这样做的代价是系统覆盖面更窄，但换来的好处是：边界清晰、correctness 条件可写、评估目标可证伪。

## 5.2 系统概览：控制面与执行面

`TwinPathServe` 的核心组织形式是双执行池，但双池本身只是这层控制面的第一版执行面实例，而不是抽象的全部。系统固定包含 `reuse_optimized_pool` 与 `throughput_optimized_pool` 两个执行池。前者的职责是尽可能保住 `physical hit`，承载高 false-negative 风险 family 的稳定归属，并负责执行 observation mismatch 对应的第一类 bridge；后者的职责则是保留激进 fast path 与通用吞吐优势，默认接纳低风险或冷启动 family。采用双池而非单池内按请求切换 backend，是因为本文关注的不是局部 backend 策略，而是如何让运行时在 admission 阶段就把“命中保护”与“吞吐优化”组织为显式结构。

围绕这一执行面，`TwinPathServe` 的最小控制面由五个逻辑组件构成。第一，`reuse oracle`，负责提供 compatible reference execution 下的 logical exact-hit 上界与 family 级命中证据。第二，`reuse contract registry`，负责表达 producer-state / consumer-requirement 的匹配关系，并决定哪些 mismatch 允许进入当前 bridge 面。第三，`realization policy`，负责比较 counterfactual hit 的潜在收益与 bridge 成本，并为当前请求输出 `realization_mode`、fallback 优先级与预算约束。第四，`bridge operator dispatcher`，负责在 admission 与 prefill 起点阶段把请求送入 selective replay、twin-pool route 或 baseline fallback。第五，`route/admission policy`，负责根据 `route key`、family 历史证据与当前风险类别在多个执行面中选择最可能安全兑现 reuse 的路径。双执行池则构成 P1 的执行面，负责真正承接路由后的请求并完成推理。

从请求生命周期看，`TwinPathServe` 的固定流程可以概括为五步：请求先构建 route key，再查询 family 级 `reuse oracle` 与当前 `reuse contract` 条件，随后由 `realization policy` 判定当前 miss 是否值得恢复以及适合哪类恢复模式，然后再由 dispatcher 与 admission policy 共同决定进入哪类 pool、是否触发 bridge、以及失败时如何 fallback，最后由选定的执行池完成 prefill 与 decode。该顺序反映了本文问题的发生位置，即许多命中丢失发生在“计算开始之前”；如果控制面不能在 admission 阶段显式介入，系统就只能在请求已经进入不兼容路径后被动接受命中坍塌。

## 5.3 Admission：route key 与 family 绑定

为了让双池结构具备系统意义，`TwinPathServe` 把每个请求映射为四元 route key：`prefix family`、`API contract class`、`backend compatibility class` 与 `false-negative risk class`。其中，`prefix family` 捕捉同一稳定前缀指纹下的请求归属；`API contract class` 区分 generation、`prompt_logprobs` 与 pooling 等消费语义；`backend compatibility class` 表达当前请求与两类执行池的兼容关系；`false-negative risk class` 则刻画 observation mismatch、backend mismatch、carrier mismatch 等风险来源。系统先基于这四个键与 `reuse oracle` 提供的 family 历史证据做 admission 决策，再在选定池内做常规负载均衡。

四元 route key 的意义不在于分类本身，而在于将第四章的抽象转化为可执行控制面。`prefix family` 提供“已有 state 是否已经在系统中留下可信参考”的入口；`API contract class` 把 consumer 的观察语义显式化；`backend compatibility class` 把执行类本身纳入 `reuse contract`；`false-negative risk class` 则把第三章的测量 reason 直接转化为后续设计可消费的控制信号。缺少这四类信息，系统就无法区分“未知 family 的普通 miss”和“已有 family 在当前 contract 下高概率坍塌”的请求。

P1 进一步固定采用 family binding 规则：一旦某个 `prefix family` 被证明具有高 false-negative 风险并成功进入 `reuse_optimized_pool`，后续同 family 请求默认继续沿用这一归属，只有在 pool 不可用或明显过载时才允许 fallback。这样做的目的不是追求理论最优调度，而是让第一版系统先获得可重复、可解释的命中行为。对 `prompt_logprobs` 请求，这一点尤其重要，因为错过第一次 family pinning 的代价往往就是整段前缀从高命中直接坍塌到零命中。

本文所说的 `batch-invariant` 并不意味着系统忽略 batch 形成过程，而是指同一个请求在 admission 阶段的 pool 选择只由 `route key` 与 `prefix family table` 决定，不应被同时到达的邻居请求、瞬时 co-batch 组合或局部 worker 抖动任意改写。池内 micro-batching 仍可保持动态，但 family 到 pool 的归属在一次 admission 中必须稳定；否则，`W5` 所暴露的问题就可能被系统重新制造出来。

由于第三章对 `W5` 的当前结论仍是 cross-backend logical evidence，而不是在线原生 `backend_mismatch` 分类，`TwinPathServe` 对 backend mismatch 的 admission 不能假设“当前 worker 已经直接给出后端不兼容原因”。相反，它依赖 family 级历史证据、route metadata 与 backend compatibility class 的组合判断。这也是为什么 `TwinPathServe` 首先是一种控制面设计，而不是对某个现有 skip reason 的简单转发。

## 5.4 执行面：双池职责与 `contract gate`

`reuse_optimized_pool` 与 `throughput_optimized_pool` 并不是“快池”和“慢池”的简单划分，而是两类不同的执行承诺。前者的职责是尽可能保住 `physical hit`，承载高 false-negative 风险 family 的稳定归属，并为 observation mismatch 提供 bridge 执行面；后者的职责则是保留最激进的通用吞吐路径，默认承接未知或低风险 family。对 runtime 来说，这种划分的真正意义在于：命中保护与吞吐优化不再混杂在同一局部 backend 开关里，而是被提升为 correctness-aware control plane 可以显式操控的系统级 admission 结构。

为了保证任何 bridge 都在 `SafeReuseContract` 约束下发生，`TwinPathServe` 在执行面入口处引入统一的 `contract gate`。系统先验证 `prefix family` identity 与 route metadata，再验证 backend / carrier compatibility，随后检查 observation sidecar、boundary replay 或 pool admission 的局部前提是否齐备，只有在这些检查全部通过时才允许执行 bridge。任何一步失败都不会触发模糊降级，而是直接记录 reason 并 fallback。由此，`SafeReuseContract` 不再只是第四章中的抽象术语，而成为 admission 与 prefill 起点阶段可追踪的显式判定流程。

`contract gate` 同时也是本文安全主张的实现落点。它把“状态已存在”“当前请求可能获益”“这次 bridge 仍然安全”这三件事明确区分开来。前两件事可以由 history 与 measurement 暗示，但只有在 `PrefixIdentity`、`ExecutionCompatibility`、`CarrierCompatibility`、`ObservationCompleteness` 与 `VerifiedFallback` 一起成立时，bridge 才被允许。也正因为如此，`TwinPathServe` 的执行面设计从一开始就内置了保守失败路径，而不是把 fallback 当成后期补丁。

## 5.5 Bridge I：`prompt_logprobs selective replay`

在 `TwinPathServe` 中，`prompt_logprobs selective replay` 被固定为 observation mismatch 的第一类 bridge。它并不是简单地“允许 `prompt_logprobs` 读取 prefix cache”，因为仅恢复 cache read 并不足以产生完整 prompt observation。当前 worker 路径把 prompt-logprob 的 materialization 与本次 request 的 prefill 过程绑定在一起；只要命中的前缀 token 没有在本次请求中重新 forward，对应位置的 prompt logprobs 就无从生成。因此，本文选择的最小安全设计是 `observation sidecar + boundary replay + full-prefill fallback`。这一 bridge 的安全主张也因此是条件性的：它证明的是与 `full-prefill baseline` 的 observational equivalence，而不是“任意打开 cache read 都安全”。

更具体地说，系统首先需要一个最小 prompt-observation sidecar，用于覆盖命中边界之前已经存在的 prompt observations；随后，运行时主动放弃命中边界的最后一个 cache block，并从该 block 起点 replay 到 prompt 末尾。这样做的原因在于：跨命中边界的第一项 prompt observation 依赖于边界前一 token 的 hidden state，不能单纯由 sidecar 补齐；而只要重放最后一个命中 block 而非整段 hit prefix，系统就能在 correctness 与成本之间取得最小折中。本文因此把 `selective replay` 的固定语义定义为：

```text
sidecar-covered prefix + one-block boundary replay + normal uncached suffix prefill
```

这里的 `ObservationCompleteness` 需要被写成可验证条件。对当前请求而言，sidecar 必须与其 observation schema 对齐，至少共同覆盖 prompt 长度、`num_prompt_logprobs` 请求粒度、各位置的 token identity、rank 与 logprob 字段；只要 sidecar 覆盖范围不足、字段语义不一致，或无法证明与当前请求的 observation contract 等价，系统就必须 fallback，而不能带着残缺 observation 继续执行。

这条 bridge 的适用条件也必须被写死。只有当请求属于 `prompt_logprobs` 消费语义、存在 exact-prefix 命中证据、sidecar 可用且 replay 成本在预算内时，系统才允许 selective replay。否则，请求必须留在当前兼容池内执行 full prefill，而不是以“只返回部分 observation”的方式继续运行。对 `prompt_logprobs selective replay` 而言，系统至少需要满足三个正确性条件：第一，返回的 prompt-logprob 条目数与 `full-prefill baseline` 完全一致；第二，各位置 top-k token、rank 与 logprob 数值在可接受误差范围内一致；第三，`DELTA` 输出时机不因 selective replay 而改变。

## 5.6 Bridge II：batch-invariant twin-pool route

第二类 bridge 面向 `W5` 暴露出的 backend mismatch。这里的问题不是 observation 语义缺失，而是不同 backend compatibility class 对前缀状态的消费能力不同，导致同一 family 在 reuse-friendly 执行类中表现为稳定高命中，在 throughput-oriented 执行类中却彻底失去复用。`TwinPathServe` 的应对方式不是在单 worker 内部动态切 backend，而是把“高命中风险 family 应优先进入哪类 pool”提升为 admission 级决策。系统在 family 历史证据、backend compatibility class 与 risk class 一致指向命中坍塌风险时，优先把请求送入 `reuse_optimized_pool`，而把未知或低风险 family 留在 `throughput_optimized_pool`。

这一设计的核心在于把 backend compatibility 本身纳入 `reuse contract`。由此，`TwinPathServe` 不再把 backend 视为与复用机制无关的局部优化开关，而是承认 backend class 本身会改变 exact-prefix state 的可消费性。只有在这一前提下，系统才可能解释为什么一个 family 在 A 池里是 clear exact hit，在 B 池里却完全无法获得命中，并进一步通过 twin-pool 结构把这种差异转化为可控制的 runtime 决策。

这里的安全主张同样是保守的。本文并不证明跨不兼容 backend class 直接共享 state 是安全的，恰恰相反，`TwinPathServe` 的 routing 设计正是为了避免 unsupported cross-backend consumption。对 backend mismatch 而言，bridge 的本质不是“共享一份跨池 KV”，而是“把本该在兼容执行类中消费的 family 重新送回兼容执行面”。这也是为什么该机制当前最适合作为 twin-pool route，而不是单 worker 内部的动态 backend 反复切换。

## 5.7 Failure Mode 与正确性约束

P1 明确禁止执行中途的 request migration。所有 fallback 都必须发生在 admission 或 prefill 起点阶段。一旦 `reuse_optimized_pool` 过载或不可用，请求可以回退到 `throughput_optimized_pool`，并记录 `route_fallback_reason`；一旦 `prompt_logprobs selective replay` 的前提不满足，请求必须留在当前兼容池做 full prefill，而不能降级为“只返回部分 observation”。这一约束的作用在于明确钉住 correctness 边界：P1 的目标是让逻辑命中恢复成为正确且可解释的命中，而不是以不透明的局部捷径换取不可控的语义偏差。

从系统设计角度看，P1 还固定了三条运行时不变量。第一，一个请求在一次 admission 中只能归属于一个 pool，不能在 bridge 失败后隐式漂移到另一条执行路径。第二，任何 pool 选择都必须能够被 `route key` 与 family 级历史证据解释，不能出现 trace 中不可见的人工特判。第三，所有 route 与 bridge fallback 都必须显式带原因标签，不能出现 silent fallback。只有把这三条写成不变量，第五章中的控制面设计才真正具备可验证性。

从运行时可验证性的角度看，`TwinPathServe` 至少需要保证三类 trace 可见性。第一，admission 必须暴露 family 归属、route key 和 pool 选择，使后续分析能够解释“为什么这个请求被送入这条路径”。第二，bridge 必须暴露触发条件、执行结果与 fallback 原因，使 measurement、reasoning 与 recovery 形成闭环。第三，系统必须显式记录所有 conservative failure path，而不是把回退隐藏在实现细节中。只有在这些信息可追踪的前提下，`TwinPathServe` 才是一套可验证的 runtime design，而不是几条藏在系统内部的启发式 patch。

本章的核心结论是：围绕第三章已经测量证实的两类强 mismatch，第四章中的统一抽象可以被落实为一层 `soundness-first` 的 correctness-aware reuse control plane，并在 `TwinPathServe` 中获得第一版系统实例。该系统通过 `reuse oracle`、route key、family binding、`contract gate`、selective replay、twin-pool route 与显式 fallback，把“为什么未命中”和“如何安全恢复部分命中”统一到同一控制链中。下一章将在这一设计基础上评估当前原型已经证实的结果及其边界。
