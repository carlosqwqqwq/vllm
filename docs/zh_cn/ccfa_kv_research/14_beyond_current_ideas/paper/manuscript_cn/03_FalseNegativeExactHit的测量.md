# 3 False-Negative Exact Hit 的测量

本章在请求级别测量 `false-negative exact hit`，并评估现有证据是否足以支撑后续抽象与系统设计。具体而言，本章界定研究问题与测量范围，给出观测面与核心度量，说明受控工作负载与配对方法，并报告两类代表性失效的测量结果及其有效性边界。

## 3.1 研究问题与测量范围

本文并不把所有 prefix miss 都视为同一个问题。本章关心的是另一类更具系统意义的 miss：请求在逻辑上已经构成 exact reuse，却没有在运行时兑现为真实的物理命中。围绕这一现象，本章聚焦回答以下两个研究问题。

`RQ1`：在共享前缀、backend class 与缓存池保持不变的前提下，额外 observation 需求是否会把原本稳定的 exact reuse 直接打成零命中？

`RQ2`：当同一 `prompt family` 已在某个 backend compatibility class 中证明可以稳定复用时，将请求放入另一类 backend compatibility class，是否会诱发强 logical false-negative？

这两个问题分别对应当前证据最清晰的两类失效来源。`RQ1` 对应 `W1 prompt_logprobs`，该场景已经具备在线可见的 skip reason；`RQ2` 对应 `W5`，该场景当前仍主要表现为 cross-backend logical evidence，但已经足以揭示 backend compatibility 与 exact reuse 之间并非彼此独立。对这两个问题的测量结果构成后续统一抽象与系统设计的经验基础。

本章同时固定采用第二章已经给出的四个操作性条件来界定 `false-negative exact hit`：存在参考执行条件；当前请求与参考请求共享相同的可复用前缀；当前变化的是消费条件而不是前缀本身；当前请求的实际命中显著低于参考请求。若任一条件不成立，例如前缀此前从未被计算过、family 对齐失败或上下文已被截断，本章都将其视为普通 miss，而不是本文讨论的问题。

## 3.2 request-level 观测面与核心度量

为保证结论建立在请求级证据之上，而不是建立在汇总后的不可追溯统计上，本文在 `vLLM` 中引入了一个最小 request-level 观测面。该观测面仅记录三类信息：请求输入侧的 typed tags、prefix lookup 的真实结果，以及 lookup 被跳过或整体关闭时的原因。其设计原则是“只做观测，不改策略”，即 P0 measurement 不引入新的 cache policy、scheduler policy 或 bridge 逻辑，只负责把问题从概念转化为可量化对象。

本章统一使用五个核心测量量：`logical_hit_tokens`、`physical_hit_tokens`、`false_negative_tokens`、`bridgeable_tokens` 和 `false_negative_reason`。其中，`physical_hit_tokens` 直接绑定到 prefix lookup 的真实返回值；`logical_hit_tokens` 在当前阶段允许通过 paired-request 方法离线推导，但本文明确不把它写成脱离执行条件的客观 ground truth。更准确地说，它是围绕 compatible reference execution 构造的 `counterfactual reuse oracle`：对同一 `prompt family` 选择一个已经在兼容条件下被系统实际命中的参考请求，并把其命中量视为 treatment 请求在该参考条件下本应可兑现的 reuse 上界；`false_negative_tokens` 则定义为：

```text
false_negative_tokens = max(logical_hit_tokens - physical_hit_tokens, 0)
```

这一差值定义使“命中为何没有发生”能够脱离具体 feature 行为被统一建模。`bridgeable_tokens` 则表示这些差值中理论上可以被当前研究路线恢复的那一部分；在 P0 阶段，它并不要求 runtime 在线输出精确值，但必须能够通过 reason class 和 request-level trace 支持离线聚合。

为了让后续章节能够围绕统一 language 展开，本章还固定采用一组一级 `false_negative_reason` taxonomy：`prompt_logprobs_skip`、`pooling_skip`、`backend_incompatibility`、`partial_prefill_incompatibility`、`carrier_incompatibility`、`no_false_negative` 与 `unknown`。其中，`prompt_logprobs_skip` 已经能够在当前 `vLLM` 路径中被在线观测到，因此可以直接进入后续 bridge 设计；`backend_incompatibility` 当前则仍停留在离线逻辑配对层面，这一点将在 `RQ2` 中明确说明。

## 3.3 受控工作负载、数据收集与配对方法

本章的正式测量基于单机 `vLLM` 环境，模型为 `Llama-2-7B-Chat`。我们使用三组受控工作负载来回答前述两个研究问题。第一组 `W1 prompt_logprobs` 维持同一 `prompt family`、同一 backend class 和同一缓存池不变，仅在 treatment 请求中开启 `prompt_logprobs=1`，用于回答 `RQ1`。第二组 `W5 backend control` 固定在 reuse-friendly backend class 中建立参考线，用于确认某个 family 在该执行类下确实可以稳定获得 exact-prefix reuse。第三组 `W5 backend batch-invariant` 则把相同 family 送入另一类 backend compatibility class 中进行探测，用于回答 `RQ2`。

这三组工作负载都采用 family-level pairing，而不是把所有请求混在一起直接求平均。`false-negative exact hit` 的定义本身依赖“逻辑上应复用”的参照物；如果不在 family 级别建立 reference/treatment 对应关系，就无法把“没有命中”与“本就没有共享前缀”区分开。具体到 `W1`，我们从 `control_generation` 中提取每个 family 的 `physical_hit_tokens` 作为 treatment 请求的 `logical_hit_tokens`；具体到 `W5`，我们则从 control backend class 中提取每个 family 的命中量作为跨池逻辑参考，再与 probe backend class 中的请求做配对。

本文并不声称参考路径的 `physical_hit_tokens` 构成任意 runtime 设置下绝对可兑现的理论最优值；它只是同一 `prefix family` 在兼容参考执行条件下已经被系统真实兑现过的 reuse 上界。因此，后文所有 `false_negative ratio` 都应理解为“相对 compatible reference execution 的丢失比例”，而不是脱离执行条件的全局客观损失。这一点对于 `W5` 尤其关键，因为该场景本身就在研究 backend compatibility class 会如何改变可兑现 reuse 的边界。

为保证结果的构造效度，本章对原始产物做了三类一致性检查。第一，文件完整性检查确认每个场景都同时具备输入侧 trace、lookup trace 与 request outputs。第二，请求对齐检查确认 manifest、lookup 和 outputs 在请求级一一对应，且 `request_outputs.num_cached_tokens` 与 `lookup.physical_hit_tokens` 完全一致。第三，family 唯一性检查确认所有场景中 family 在共享文本开始前就完成分叉，从而排除不同 family 之间前缀串扰污染结论的可能。当前正式结果均通过了这三类检查。

最后说明一个与结果解释直接相关的细节。最终共享前缀长度约为 `1098` tokens，但 control 请求中的稳定命中是 `1088`，不是 `1098`。这并不表示测量有误，而是因为当前 `vLLM` 的 prefix hit 以 block 为粒度，且实验配置中的 `block_size = 16`。在这种情况下，只有完整对齐的整块 token 才能被计入 `physical_hit_tokens`，因此 `1098` token 的共享前缀在当前配置下只能兑现为 `68` 个 block，也即 `1088` 个物理命中 token。后续所有“`1088 -> 0`”的结果都应在这一粒度约束下理解。

## 3.4 `RQ1`：额外 observation 需求是否会把稳定命中打成零命中

`RQ1` 的答案是肯定的。`W1` 中的 control 组与 treatment 组在共享前缀、backend class 和缓存池上完全一致，唯一变化是 treatment 请求开启了 `prompt_logprobs=1`。在 control 组中，每个 family 的普通 generation 请求稳定获得平均 `1088` 个 `physical_hit_tokens`；在 treatment 组中，所有请求的 `physical_hit_tokens` 全部为 `0`。按 family 配对后得到：

```text
logical_hit_tokens = 8704
physical_hit_tokens = 0
false_negative_tokens / logical_hit_tokens = 1.000
```

该结果表明，`prompt_logprobs` 并非轻微降低命中率，而是将已经稳定发生的 exact-prefix reuse 在运行前完全抹去。与此同时，这一现象并不依赖纯离线推断。在当前 trace 中，treatment 请求已经被在线标记为 `false_negative_reason = prompt_logprobs_skip`，同时 `bridge_candidate = true`。因此，`W1` 同时满足三个条件：问题幅度显著，存在明确的在线原因标签，并且已经进入当前研究主线的 bridgeable 范围。

这一现象也并非只出现在 `MT-Bench` 这类相对短的共享前缀设置中。我们在公开 `LongBench W1` 长上下文负载上进行了同样的正式测量：筛选后共有 `64` 个请求、`32` 个 family，平均 prompt 长度达到 `10191.84` tokens。在该设置下，仅增加 `prompt_logprobs` 需求，就会把 `control_generation` 中可兑现的 `483504` 个 `logical_hit_tokens` 全部压成 `0` 个 `physical_hit_tokens`，对应的 `false_negative_tokens / logical_hit_tokens` 仍为 `1.000`。这说明 `W1` 不是局限于单一小型 benchmark 的偶发现象，而是已经在公开长上下文负载上放大为可观测的系统性损失。

从系统视角看，`RQ1` 进一步说明，`prompt_logprobs` 的问题并不适合被理解为单个 feature 的局部缺口。更准确地说，observation requirement 改变了 consumer 侧契约，而系统缺少一套 `reuse contract` 去连接“已有 prefix KV”和“所需 prompt observation”。因此，`W1` 更适合作为 `observation mismatch` 的典型实例，而不是孤立的 feature bug。

## 3.5 `RQ2`：backend compatibility class 是否会诱发强 logical false-negative

`RQ2` 的答案同样是肯定的，但其结论边界需要更谨慎地表述。`W5` 的参考池固定在 reuse-friendly backend compatibility class 中，同一批 family 的 control 请求稳定获得平均 `1088` 个 `physical_hit_tokens`；而探测池则把相同 family 送入另一类 backend compatibility class 中，结果所有 probe 请求的 `physical_hit_tokens` 都为 `0`。按 family 配对后，同样得到：

```text
logical_hit_tokens = 8704
physical_hit_tokens = 0
false_negative_tokens / logical_hit_tokens = 1.000
```

这一结果说明，一个 family 在 reference backend class 中已经被系统证明可以 exact reuse，但一旦落入另一类 backend compatibility class，原本清晰的逻辑命中机会就可能在运行前完全丢失。从问题表征角度看，这已经足够强，可以支撑“backend compatibility class mismatch 会诱发强 logical false-negative”这一论断。

但与 `RQ1` 不同，当前探测池内部仍把这些请求记为 `no_false_negative`。这并非测量错误，而是因为现有 runtime 站在单池视角下只能观察到“本池未命中”，无法同时看到另一执行类中已经存在的逻辑参考状态。因此，本章对 `RQ2` 的结论限定为：`W5` 证明了 cross-backend logical false-negative 的存在，并为 twin-pool 结构提供了设计依据，但尚不能被表述为“runtime 已经在线完成 `backend_mismatch` 分类”。后续第五、六章中出现的 `route_reason = backend_mismatch`，应理解为控制面结合 compatible-reference evidence 与 backend metadata 给出的诊断标签，而不是 vanilla runtime 的原生 skip reason。

## 3.6 有效性威胁与当前局限

从内部效度看，本章最大的早期风险来自 prompt 过长和 family 串扰。前者会触发模型上下文长度校验，使结果失效；后者会让不同 family 在 probe 批次中相互借到 prefix hit，从而低估真实的 false-negative 比例。当前正式结果已经显式修复了这两个问题，并通过 family 唯一性检查加以验证。另一个内部效度相关细节是 block 粒度造成的 `1098 -> 1088` 差异；本章已将其解释为当前 `block_size = 16` 下的正常行为，而不是测量异常。

从构造效度看，当前最大的限制在于 `logical_hit_tokens` 仍然是 derived metric，而非 runtime 在线直接输出的字段。这一点在 `W1` 中影响较小，因为 treatment 的 `false_negative_reason` 已能在线观测；但在 `W5` 中，它直接导致当前证据只能停留在 cross-backend logical false-negative，而不能上升为在线原生 `backend_mismatch` 分类。更具体地说，`logical_hit_tokens` 目前是“兼容参考执行下的反事实 oracle”，而非完全独立于执行条件的客观 ground truth。因此，正文需要清楚区分“已经在线可见的原因”“通过 family 配对得到的逻辑证据”以及“相对 compatible reference execution 的损失上界”。

从外部效度看，当前证据已经覆盖单机 `vLLM`、单模型下的两类公开负载与一类受控路由场景：`MT-Bench W1`、`LongBench W1` 以及 `W5` 的 family-level backend 探测。这样的选择已经比早期版本更接近真实公开工作负载，但仍不足以支持对所有大模型推理框架的无条件外推。本文因此有意识地把本章的结论锚定在 `vLLM` 这一实例系统上，而把更广泛的 workload family、更多公开 benchmark 与跨框架一般性讨论留给后续章节与 supporting 证据。

## 3.7 本章小结

本章表明，`false-negative exact hit` 可以通过 request-level 证据被稳定识别。通过 `RQ1`，本文证明 `prompt_logprobs` 可以在共享前缀、backend class 和缓存池完全不变的情况下，把稳定的 exact-prefix reuse 直接打成零命中；这一现象不仅在 `MT-Bench` 中成立，也已经在 `LongBench` 长上下文公开负载中得到验证，并具备在线 reason 标签。通过 `RQ2`，本文进一步证明 backend compatibility class 的切换同样能够诱发强 logical false-negative，尽管当前在线归因尚未闭环。

这组结果具有两个直接含义。第一，`FalseNegativeKV` 并不是凭空构造的问题类，而是在现有推理框架中已经真实存在、且规模显著的运行时现象。第二，后续分析应围绕已经被测量证实的两类 mismatch，构建统一的契约感知抽象与最小恢复机制。下一章将在这一基础上说明，为什么这些看似来自不同路径的失效可以被纳入同一套 `reuse contract` 语言加以理解。
