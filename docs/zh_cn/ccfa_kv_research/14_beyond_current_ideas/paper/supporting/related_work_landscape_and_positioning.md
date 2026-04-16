# FalseNegativeKV 相关工作全景与定位底稿

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 把 `FalseNegativeKV / TwinPathServe` 在近邻系统论文中的位置钉死，供第七章、rebuttal 和后续扩写反复复用

它不负责：

- 代替主文稿第七章
- 发明新的 thesis
- 夸大尚未完成的实验证据

这份文档的固定目标是：

1. 让 related work 不再只是堆名字
2. 让每类近邻工作都能被归到清晰的问题边界
3. 让 reviewer 最可能的“这不就是某某方向吗”能够被提前拆解

## 2. 当前最需要守住的定位

`FalseNegativeKV` 研究的不是：

- “如何让 prefix cache 更快”
- “如何让 prefix match 更灵活”
- “如何设计另一种更强的 cache medium”
- “如何把 agent / tool workflow 的 state 生命周期保得更久”

`FalseNegativeKV` 研究的是：

- 在 reference execution condition 下，某段 prefix state 明明已经存在，或者已经被证明可复用；
- 但当前 request 因为 consumer contract、observation requirement、backend compatibility 或 carrier semantics 的变化，未能把这份既有计算兑现成自己的 `physical hit`；
- 这种现象如何被测量、归因，并在 `soundness-first` 前提下被部分恢复。

因此，related work 的组织必须始终围绕下面这条界线：

> 近邻工作大多优化“已经成立且可消费的 hit”；  
> `FalseNegativeKV` 关注的是“逻辑上已经应命中，但在计算开始前没有 materialize 的 hit”。

## 3. 第一层近邻：通用 LLM serving runtime

这类工作奠定了现代 LLM serving 的基本结构，包括 continuous batching、paged KV、radix/prefix cache、CUDA graph、并发调度与开放接口。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `Orca` | OSDI 2022 | 如何通过 iteration-level scheduling 和 selective batching 提升生成式 Transformer serving 吞吐与延迟 | 是现代 autoregressive serving 的重要起点 | 它默认 request path 的生成语义已经成立，不讨论既有 prefix state 为何不能被当前 consumer 安全消费 |
| `vLLM / PagedAttention` | 2023 paper/blog | 如何通过 paged KV memory 和 continuous batching 高效管理 LLM serving | 是本文的实例系统与主要证据来源 | 它强于“如何管理与利用已经可消费的 state”，弱于“为什么 logical hit 会在执行前消失” |
| `SGLang` | 官方 blog / 代码 / 文档 | 如何通过 radix cache、前端 DSL、推理 runtime 优化 LLM application serving | 提供跨框架 capability boundary 的 supporting evidence | 它也暴露了多类 cache capability boundary，但当前并未把这些边界统一形式化为 `false-negative exact hit` |
| `DeepSpeed-FastGen` | arXiv / 官方技术报告 | 如何在生产 serving 中提升高吞吐文本生成 | 属于通用 serving baseline | 关注整体 serving path，而不是 consumer-specific reuse failure |
| `TGI` | Hugging Face 官方文档 | 如何提供工业级文本生成推理服务 | 是业界常见对比框架 | 它提供的是 serving 功能集合，不研究 logical hit 与 physical hit 的脱节 |
| `TensorRT-LLM` | NVIDIA 官方文档 | 如何通过推理编译、kernel 与内存优化提升 LLM serving 性能 | 是工业 runtime 代表 | 它优化的是执行效率与内核路径，不回答 mixed-API reuse contract 问题 |
| `Llumnix` | 近期系统工作 / preprint | 如何在 LLM serving 中做更动态的调度与资源管理 | 说明 serving 社区已把 runtime-level scheduling 当主问题 | 但它仍是 scheduler-centric，不是 contract-aware reuse-centric |

这一层工作的共同点非常重要：

- 它们把 prefix reuse 当作 runtime 优化能力来管理
- 但默认“当系统说可以 hit 时，这个 hit 就已经是可消费的”

而本文从一开始就在挑战这个默认前提。

## 4. 第二层近邻：调度、解耦、分布式 prompt-aware routing

这类工作最容易与 `TwinPathServe` 混淆，因为它们同样涉及 routing、pooling、prefill/decode 分离与 prefix-aware placement。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `Sarathi-Serve` | OSDI 2024 / arXiv | 如何通过 chunked prefills 和 stall-free scheduling 改善吞吐-延迟折中 | 与我们一样处理 runtime-level serving path | 它优化的是 prefill/decode 并发执行，不讨论 consumer contract 导致的 false-negative hit |
| `DistServe` | OSDI 2024 / arXiv | 如何通过 prefill / decode disaggregation 优化 goodput | 与我们一样处理 execution-path choice | 它研究算力路径拆分与 cluster-level placement，而不是为什么 exact-prefix reuse 没被当前 request 兑现 |
| `Splitwise` | ISCA 2024 | 如何用 phase splitting 优化 generative LLM inference | 与我们一样把 prefill 和 decode 当不同阶段处理 | 它处理的是 phase-level mapping，不是 hit realization failure |
| `Preble` | arXiv 2024 | 如何在分布式 serving 中同时优化 prompt sharing 与负载均衡 | 与我们一样关心 prefix-aware routing | 它解决的是“已知 prompt sharing 时如何更好地分布式调度”，不是“为什么 logical share 没有 materialize” |
| `Mooncake` | arXiv / 开源系统 2024-2025 | 如何以 KVCache-centric 架构做 disaggregated serving 与全局 KVCache pool | 与我们一样显式处理 KV-oriented system design | 它研究的是 KV movement、storage pool、SLO-aware scheduling，而不是 engine-local consumer compatibility |
| `Parrot` | OSDI 2024 | 如何用 semantic variable 支撑 LLM application serving | 与我们一样关注 LLM application 层的结构 | 它聚焦 application composition 与 variable-level sharing，不是 false-negative exact hit 的 runtime pathology |

这一层工作的共性是：

- 它们几乎都假定“只要 state 可以共享，就主要问题在于怎么搬、怎么排、怎么分池”

而本文强调的是：

- 在很多 mixed-API / mixed-backend 情况下，主要问题发生得更早；
- 系统根本还没进入“怎么搬、怎么排”的阶段，就已经因为 consumer-side contract mismatch 丢掉了命中。

`TwinPathServe` 虽然也有 routing 和 dual-pool 结构，但其第一性目标不是更优调度，而是：

- 把 backend compatibility 明确纳入 `reuse contract`
- 在 `soundness-first` 前提下，让本来会 false-negative 的 family 回到兼容执行面

## 5. 第三层近邻：shared-prefix execution 与 exact-prefix 命中后的加速

这类工作与本文在“共享前缀”这个表面词汇上最像，但真正研究的问题是不同的。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `Prompt Cache` | MLSys 2024 | 如何把可复用文本片段组织成模块，并在低延迟推理中复用 attention states | 与我们一样处理重复 prompt segment | 它假定模块化 segment 的 reuse contract 已经由 schema 建立，不研究 runtime 为何拒绝消费已存在状态 |
| `Hydragen` | arXiv 2024 | 当一个 batch 具有 shared prefixes 时，如何高效实现 exact attention | 与我们一样处理 shared prefix | 它研究的是 hit 已经成立之后的 kernel/execution optimization |
| `FastTree` | MLSys 2025 | 如何利用 tree-structured shared execution 提升共享前缀推理效率 | 与我们一样关心 shared structure | 它仍在“共享已被识别之后如何更快”这条线上 |
| `PAT` | arXiv 2025 | 如何通过 prefix-aware attention kernel 减少 decode 中重复访存 | 与我们一样利用 shared prefix | 它是 decode kernel 优化，不是 runtime contract 建模 |
| `Marconi` | MLSys 2025 / arXiv 2024 | 在 hybrid LLM 中如何做更合理的 prefix caching admission / eviction | 与我们一样显式讨论 prefix caching 边界 | 它解决的是 hybrid model 下“哪些 entry 更值得缓存”，仍然围绕 physical hit optimization，而不是 logical hit 为什么没被当前 consumer 用上 |

这一层工作可以帮助我们讲清楚一个关键区别：

> 这些方法研究的是“hit 之后如何更快”；  
> `FalseNegativeKV` 研究的是“为什么 hit 之前就丢了”。

因此论文必须避免滑向下面这些写法：

- “我们做了一种更好的 prefix-aware attention”
- “我们做了更灵活的 shared-prefix kernel”
- “我们优化了 reuse after hit establishment”

一旦这么写，就会直接进入 `Hydragen / Prompt Cache / PAT / FastTree` 的空间。

## 6. 第四层近邻：超越 exact prefix 的缓存与位置无关复用

这类工作是另一条非常强的近邻线，因为 reviewer 很可能会把我们的 bridge 错看成 position-independent reuse。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `EPIC` | ICML 2025 | 当上下文内容相同但不再是 exact prefix 时，如何做 position-independent caching | 与我们一样想恢复“原本难以复用”的 context reuse | 它主动放宽 match 条件，研究 non-prefix reuse；我们坚持 exact / logical-exact reuse 不变，只研究为什么 consumer 不能消费 |
| `MPIC` | arXiv 2025 | 如何在 multimodal serving 中实现位置无关上下文缓存 | 与我们一样涉及 multimodal path | 它研究 multimodal PIC，本质上仍是“让不再精确前缀的状态可复用” |
| `MEPIC` | 近期 preprint | 如何进一步扩展位置无关与多模态缓存 | 与我们一样处理复杂 carrier | 它仍属于 match-relaxation 路线 |
| `CacheBlend` | arXiv 2024 | 在 RAG 中，当可复用 chunk 不一定处于前缀位置时，如何选择性重算部分 token 并融合缓存知识 | 与我们一样含有“局部重算 + 缓存融合”的 flavor | 但它研究的是 non-prefix chunk fusion；我们的 selective replay 不是为了 position independence，而是为了补齐 observation contract |

这一层最需要主动写清的一句话是：

> 本文不是在研究“如何让不再是 prefix 的状态也能复用”；  
> 本文研究的是“当 prefix identity 已经成立时，为什么 reuse 仍然不会 materialize”。

这就是为什么：

- `selective replay` 不能被写成 PIC 的轻量版
- `TwinPathServe` 也不能被写成“更灵活的 prefix cache”

## 7. 第五层近邻：state restoration、hybrid cache 与外部 KV 层

这类工作与我们的 `sidecar`、`bridge` 和 `TwinPathServe` 最容易形成“你们是不是也在换 state medium”的误解。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `HCache` | EuroSys 2025 | 如何利用 intermediate activations 更快恢复上下文状态 | 与我们一样关心状态恢复 | 它的核心是新的 restoration medium；我们第一版不引入 hidden activation cache |
| `Apt-Serve` | arXiv 2025 | 如何通过 `KV + hidden cache` 的 hybrid cache 与调度提升 serving 效率 | 与我们一样显式讨论 hybrid state | 它研究 state medium 的扩展与 scheduler 协同；我们的 sidecar 只是 bridge 所需的最小伴生状态 |
| `LMCache` | arXiv / tech report 2025 | 如何把 KV 暴露成跨引擎、跨层级的外部缓存层，实现 offload 与跨引擎共享 | 与我们一样显式处理 KV movement 与 cache orchestration | 它研究的是 cache layer 和 control API；我们研究的是 engine-local producer / consumer contract mismatch |
| `Mooncake` | 也可归入此类 | 如何构建 KVCache-centric data plane 与跨设备 KV 池 | 与我们一样强调 KV-centric system design | 但其核心问题是 movement / storage / disaggregation，不是 false-negative exact hit |
| `Marconi` | 也与此类相邻 | 如何在 hybrid LLM 中做 cache admission / eviction | 与我们一样碰到 cache boundary | 但其焦点是 hybrid model 的 state properties，不是 mixed-API consumer usability |
| `Beyond Speedup` | ICLR 2026 preprint | 如何把 KV cache 用作采样与推理的轻量表示 | 与我们一样赋予 cached state 更多语义角色 | 它主动扩展 KV 的任务语义；我们第一版只做 correctness-preserving bridge，不把 KV cache 改造成新型表示学习介质 |

对我们最重要的约束是：

- `prompt_logprobs` sidecar 不能膨胀成另一套 hidden-state cache 系统
- 第五章与第六章必须始终坚持：sidecar 的角色是 `ObservationCompleteness` bridge，而不是新 state medium 的主张

否则 reviewer 很容易把本文重新归到 `HCache / Apt-Serve / LMCache` 这条线。

## 8. 第六层近邻：workflow continuity、agent / tool serving 与 augmented inference

这类工作解释了为什么“保住 state”在 agentic / augmented setting 中重要，但它们的核心矛盾与我们不同。

| 工作 | 来源 | 它回答的问题 | 与我们的关系 | 为什么不是同一问题 |
| --- | --- | --- | --- | --- |
| `InferCept` | ICML 2024 / arXiv | 在 augmented LLM inference 中，如何高效支持生成被工具/环境截停后的继续执行 | 与我们一样处理 mixed-API / augmented path | 它研究 intercept support 与 augmented workflow execution，不是 false-negative exact hit |
| `KVFlow` | arXiv 2025 / NeurIPS 2025 poster | 如何面向 multi-agent workflow 做 workflow-aware KV eviction 与 prefetch | 与我们一样处理 prefix cache 与 tree-structured workloads | 它研究 state lifetime 与未来工作流调度，而不是 consumer-side contract mismatch |
| `Continuum` | 近期 preprint | 如何在多轮 agent / workflow 场景中保住 continuity | 与我们一样关心 context continuation | 它关注 workflow continuity 而非 exact-prefix usability |
| `AugServe` | arXiv 2025 | 如何对 augmented LLM request 做自适应调度和 batch token 调整 | 与我们一样研究 augmented workload serving | 它聚焦 scheduling 与 queueing/SLO，而不是 reuse contract |
| `RAGServe` | arXiv 2024 | 如何在 RAG serving 中联合做调度与配置自适应 | 与我们一样处于 long-context / augmented serving | 它解决的是 quality-latency tradeoff，不是 false-negative reuse |
| `Parrot` | OSDI 2024 | 如何用 semantic variable 支撑复杂 LLM application serving | 与我们一样面对非单轮 chat 的复杂应用 | 它研究 application-level orchestration，而不是 exact-prefix hit 的实现落差 |

这一层工作提醒我们：

- 如果论文把问题写成“tool 调用回来后还想继续复用 KV”
- 或者写成“agent workflow 需要更聪明地保状态”

那么边界就会迅速滑向 `InferCept / KVFlow / Continuum / AugServe / RAGServe`。

而本文当前最稳的边界仍然是：

- exact / logical-exact reuse
- engine-local consumer compatibility
- contract-aware bridge

## 9. 第七层近邻：feature-specific capability boundary 与开源系统讨论

除了论文本身，另一个重要近邻来源是开源 serving runtime 的实现与文档。

| 来源 | 现象 | 对本文的价值 | 它不能替代什么 |
| --- | --- | --- | --- |
| `vLLM` V1 guide / 源码 | `prompt_logprobs` 请求忽略 prefix cache；pooling / backend path 存在显式 capability boundary | 是本文最强的实例系统证据 | 不能替代统一抽象与系统命名 |
| `SGLang` 代码与文档 | `input_embeds`、multimodal reranker、部分 deterministic/backend-specific path 需要关闭 radix cache | 说明 capability boundary 并非 `vLLM` 独有现象 | 不能直接证明 pathology 在两系统中完全同构 |
| 工程 issue / RFC / PR 讨论 | `prompt_logprobs + APC`、hidden-state processor、兼容性修补等 | 说明 mixed-API mismatch 已被一线开发者感知 | 仍然停留在 feature-local explanation，缺少问题类层次 |

因此，第七章可以也应该利用这些工程事实，但必须注意层级：

- 它们更像“问题存在的实现侧证据”
- 而不是“替代学术 related work”

最稳的写法是：

- 用论文工作组织主要 related work
- 用开源系统 capability boundary 作为最后一层工程语境

## 10. reviewer 最可能混淆的四种“错位对比”

### 10.1 “你们这不就是更好的 prefix cache 吗”

标准回答：

- 不是。prefix cache 工作主要研究如何提高已经成立的 `physical hit` 的利用效率；
- 本文研究的是 `logical hit -> physical hit` 之间为何会在执行前断裂。

### 10.2 “你们这不就是 PIC / 非前缀复用吗”

标准回答：

- 不是。PIC 研究的是当 prefix identity 已经不成立时，如何继续复用；
- 本文研究的是 prefix identity 已经成立甚至 state 已存在时，为什么当前 consumer 仍然不能合法消费。

### 10.3 “你们这不就是 hidden cache / hybrid cache 吗”

标准回答：

- 不是。`HCache / Apt-Serve / LMCache` 研究的是新的 state medium 或外部 cache layer；
- 本文第一版 bridge 只引入最小 sidecar，以满足 observation contract，不把它扩展成另一套 hidden-state cache 系统。

### 10.4 “你们这不就是 agent workflow 保状态吗”

标准回答：

- 不是。`InferCept / KVFlow / Continuum / AugServe / RAGServe` 研究的是 workflow continuity、augmented serving 或 quality-latency scheduling；
- 本文研究的是 exact-prefix state 的 consumer-side usability。

## 11. 第七章最稳的组织顺序

如果主文稿第七章要保持顶会系统论文的节奏，最稳的组织顺序是：

1. 通用 LLM serving runtime 与 physical-hit optimization
2. 调度、解耦与分布式 prefix-aware routing
3. shared-prefix execution 与 hit-established 加速
4. PIC / multimodal / RAG 场景下的非 exact-prefix reuse
5. state restoration / hybrid cache / 外部 KV 层
6. workflow continuity 与 augmented inference
7. feature-specific capability boundary 与工程讨论
8. 本文填补的空白：false-negative exact hit

这样组织的好处是：

- reviewer 可以看到我们不是“没读过近邻工作”
- 同时也能清楚看到，我们并不是这些方向中的任意一个旧问题换皮

## 12. 可直接复用的总定位句

如果后续相关工作、rebuttal 或答辩只保留一句话，这句最稳：

> 现有工作大多研究如何更快地利用已经可消费的 prefix/cache state、如何放宽复用匹配条件、如何设计新的状态介质，或如何在 agent/RAG workflow 中更长久地保住状态；  
> `FalseNegativeKV` 填补的空白则是：当 exact 或 logical-exact reuse 在 reference execution 下已经成立时，为什么当前 request 仍然无法把它兑现成自己的 `physical hit`。

## 13. 关键来源

- Orca: https://www.usenix.org/conference/osdi22/presentation/yu
- vLLM / PagedAttention: https://blog.vllm.ai/2023/06/20/vllm.html
- Preble: https://arxiv.org/abs/2407.00023
- Mooncake: https://github.com/kvcache-ai/Mooncake
- Prompt Cache: https://research.google/pubs/prompt-cache-modular-attention-reuse-for-low-latency-inference/
- Hydragen: https://arxiv.org/abs/2402.05099
- EPIC: https://proceedings.mlr.press/v267/hu25j.html
- CacheBlend: https://arxiv.org/abs/2405.16444
- Marconi: https://arxiv.org/abs/2411.19379
- LMCache: https://arxiv.org/abs/2510.09665
- KVFlow: https://arxiv.org/abs/2507.07400
- InferCept: https://arxiv.org/abs/2402.01869
- AugServe: https://arxiv.org/abs/2512.04013
- RAGServe: https://arxiv.org/abs/2412.10543
- SGLang 官方博客: https://lmsys.org/blog/2024-01-17-sglang/
