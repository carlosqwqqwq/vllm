# TxnStateKV 深度复审：重复性、可行性与投稿潜力评估

## 1. 严格结论

先给结论，不绕弯：

- 截至 `2026-04-15`，我**不能诚实地说“确保没人做过”**。
- 但基于这轮检索，我**没有找到与 refined `TxnStateKV` 完全等价的公开论文**。
- 同时，我也必须把风险说清楚：`TxnStateKV` 已经不再是“低重合安全区”，而是一个**有明显近邻、但仍可能通过重新定界获得投稿空间**的方向。

更准确地说：

- 如果我们把它写成“transactional KV cache”，那会**直接撞上** `TransKV`。
- 如果我们把它写成“tool-aware pause/resume + cache retention”，那会**明显靠近** `InferCept`、`Continuum`、`Tokencake`。
- 如果我们把它写成“program-aware serving runtime”，那会**逼近** `Serve Programs, Not Prompts / Pie / ThunderAgent`。

所以这条线能否成立，关键不在于“有没有 transaction 这个词”，而在于：

> 我们能否把研究问题收窄成：`vLLM` 这类现有推理引擎内部，**可复用前缀状态何时可见、何时稳定、何时可恢复、何时必须撤销**，是否需要一个统一的生命周期协议，而不是继续依赖散落在 scheduler、streaming、KV manager、event channel 里的 ad hoc 规则。

如果用这一定义，`TxnStateKV` 仍然有研究空间。

## 2. 这轮检索到的高风险近邻

### 2.1 `TransKV`（2026-02, TechRxiv）

资料：

- [Transactional KV Caching for Speculative Decoding under Paged KV Memory](https://d197for5662m48.cloudfront.net/documents/publicationstatus/307056/preprint_pdf/197af17fbcff12ca9844d78e933bd197.pdf)

它做了什么：

- 把 speculative decoding 的 KV 写入看成事务；
- 把 `committed KV` 与 `speculative packed buffer` 分离；
- 只把 accepted tokens 提交到 paged KV 中；
- 避免 rejected draft token 污染 paged cache。

为什么危险：

- 它已经把 “transactional KV caching” 这一说法占掉了；
- 它同样谈 `committed / uncommitted`；
- 它同样强调“只在状态稳定后 commit”。

为什么又不完全相同：

- 它的对象是 **speculative decoding 的 draft window**；
- 它的核心问题是 **paged KV + speculative window 的粒度失配**；
- 它不讨论 pause/resume、tool-call stall、streaming input、external visibility、async placeholder、request/session continuation。

结论：

- **术语高风险重合**
- **机制局部重合**
- **thesis 级命题不完全相同**

这意味着我们后续**不应再用“transactional KV cache”作为主标题**，也不应把核心卖点写成“committed / uncommitted KV split”。

### 2.2 `InferCept`（2024）

资料：

- [InferCept: Efficient Intercept Support for Augmented Large Language Model Inference](https://arxiv.org/abs/2402.01869)

它做了什么：

- 面向 augmented / tool-augmented LLM；
- 当请求触发外部 API 调用时，暂停请求；
- 把已算上下文保留或换出，之后再恢复；
- 避免每次工具调用后重新从头 prefill。

与我们的关系：

- 它覆盖了 `paused -> resumed` 的生命周期片段；
- 但它没有把“状态何时 publish/commit/abort/branch”形式化为统一协议；
- 它更像 intercept-aware serving system，而不是 reusable prefix state 的 formal lifecycle。

结论：

- **问题场景近**
- **研究抽象层不同**

### 2.3 `Tokencake`（2025-10）

资料：

- [Tokencake: A KV-Cache-centric Serving Framework for LLM-based Multi-Agent Applications](https://arxiv.org/abs/2510.18586)

它做了什么：

- 面向 multi-agent + function/tool calls；
- 用 space scheduler 保护关键 agent 的 cache；
- 用 proactive offload / upload 利用 tool-call stall 时间。

与我们的关系：

- 它管的是 **agent-aware scheduling + memory management**；
- 它承认 long-running tool calls 让 cache idle；
- 但它没有把 reusable state 抽象成 `publish / commit / pause / abort / branch` 协议。

结论：

- **在 workload 上接近**
- **在系统语义上不完全重复**

### 2.4 `KVFlow`（2025-07）

资料：

- [KVFlow: Efficient Prefix Caching for Accelerating LLM-Based Multi-Agent Workflows](https://arxiv.org/abs/2507.07400)

它做了什么：

- 把 workflow 抽象成 Agent Step Graph；
- 基于 future step proximity 做 eviction；
- 做 workflow-aware prefetch。

与我们的关系：

- 它是 workflow-aware retention/prefetch；
- 但它默认 cache entry 已经存在，只是在决定“留不留、何时取回”；
- 它不讨论“这个 state 什么时候从 private 变成 globally reusable”。

结论：

- **管理层邻近**
- **状态语义层不重合**

### 2.5 `Continuum`（2025-11 / 2026-01）

资料：

- [Continuum: Efficient and Robust Multi-Turn LLM Agent Scheduling with KV Cache Time-to-Live](https://arxiv.org/abs/2511.02230)

它做了什么：

- 研究 tool pauses 下 KV 是否该继续 pin 在 GPU；
- 用 TTL 控制保留窗口；
- 与 program-level FCFS 一起减少多轮 agent workflow 的等待与重算。

与我们的关系：

- 它本质是 **pause 后 retain 多久**；
- 是 lifecycle 的一个重要阶段；
- 但它没有把 publish/commit/abort 和 visibility 统一起来。

结论：

- **局部机制强近邻**
- **完整 thesis 仍不等价**

### 2.6 `TensorRT-LLM Early Reuse`（2024, 工业系统）

资料：

- [5x Faster Time to First Token with NVIDIA TensorRT-LLM KV Cache Early Reuse](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)

它做了什么：

- 允许 system prompt 的 KV 在当前请求还没完全结束前就被复用；
- 本质上是把某些 KV 状态更早变成“可见、可共享”；
- 这是一个很强的 publication 邻居。

与我们的关系：

- 它说明“early publication / early visibility”已经不是空白点；
- 但它不是一个通用生命周期协议；
- 它主要是具体 reuse optimization，而不是 unified runtime semantics。

结论：

- **publish 机制已有工业先例**
- 这要求我们不能把贡献写成“更早发布 KV”这么窄的命题。

### 2.7 `Serve Programs, Not Prompts` / `Pie` / `ThunderAgent`（2025-2026）

资料：

- [Serve Programs, Not Prompts](https://arxiv.org/abs/2510.25412)
- [Pie: A Programmable Serving System for Emerging LLM Applications](https://arxiv.org/abs/2510.24051)
- [ThunderAgent: A Simple, Fast and Program-Aware Agentic Inference System](https://arxiv.org/abs/2602.13692)

它们做了什么：

- 不再把 serving 看成“单次 prompt completion”；
- 引入 program-aware / inferlet / workflow-aware 抽象；
- 允许外部逻辑、工具调用、资源编排与 KV 管理协同。

为什么重要：

- 这类工作是 `TxnStateKV` 的“上位威胁”；
- 如果我们把故事讲太大，就会被问：
  - 为什么不直接把它做成 programmable runtime？
  - 为什么不是 program-aware orchestration？

结论：

- `TxnStateKV` 若要成立，**必须明确自己只研究现有 `vLLM` 引擎内部的状态协议层**；
- 不做 inferlet，不做通用程序运行时，不做完整 agent orchestrator。

## 3. 哪些表述现在已经不安全

经过这轮调研，以下写法我不建议再用：

- “`TxnStateKV` 是第一个 transactional KV cache 系统”
- “我们首次提出 committed / uncommitted KV”
- “我们首次解决 pause/resume 的 KV 生命周期”
- “我们首次支持 agentic workload 的状态恢复”
- “我们提供一个 program-aware stateful serving runtime”

这些说法要么已经和已有工作直接重叠，要么非常容易引发审稿人反驳。

## 4. 目前唯一较可防守的 refined 命题

如果继续做这条线，我建议把命题收敛成：

> 在 `vLLM` 这类支持 prefix cache、async scheduling、streaming input、pause/resume 的现有推理引擎中，可复用前缀状态已经隐式存在 `private / visible / stable / resumable / revocable` 等不同阶段；问题在于这些阶段今天分散在不同模块的局部规则里。我们提出一个统一的生命周期协议，把 reusable prefix state 的发布、确认、暂停、恢复与撤销变成显式的一等语义。

这一定义有四个关键词：

- `existing runtime`
- `reusable prefix state`
- `lifecycle protocol`
- `explicit semantics`

它刻意避开了：

- speculative decoding 专题
- program-aware orchestration
- workflow-level eviction/prefetch
- 通用 agent OS

## 5. vLLM 源码里已经存在哪些“事务化胚芽”

这一部分很关键，因为它决定这条线是不是空想。

### 5.1 `Request` 里已经有“未稳定输出”语义

`vllm/v1/request.py`

- `num_output_placeholders`
- `discard_latest_async_tokens`
- `WAITING_FOR_STREAMING_REQ`
- `PREEMPTED`

这说明：

- 当前运行时并不认为“已调度输出 token”和“已稳定提交 token”是同一回事；
- async scheduling 下已经存在“先占位、后确认、必要时丢弃”的隐式两阶段语义。

### 5.2 `AsyncScheduler` 已经显式区分 confirmed vs placeholder

`vllm/v1/core/sched/async_scheduler.py`

- 调度时先增加 `num_output_placeholders`
- 输出真正回来后再扣减 placeholder
- `cache_blocks()` 使用的是：
  - `request.num_computed_tokens - request.num_output_placeholders`

这其实已经是：

- speculative / provisional state 不应直接算作 fully committed state

### 5.3 `simple_kv_offload` 也显式使用 confirmed token 边界

`vllm/v1/simple_kv_offload/manager.py`

- `confirmed_tokens = req.num_computed_tokens - req.num_output_placeholders`

这说明：

- “哪些 token 对应的 KV 已经稳定可见”在今天已经影响 cache/offload 决策；
- 只是这种语义还没有被 formalize。

### 5.4 `scheduler._update_request_as_session()` 已经做了 ad hoc commit

`vllm/v1/core/sched/scheduler.py`

- 只保留 computed output tokens；
- 丢弃 prior input chunk 的最后 sampled output token；
- 再把保留下来的 token 并入新 prompt；
- 最后重新更新 block hash。

这几步本质上已经在做：

- 哪些 token 能进入长期前缀状态；
- 哪些 token 只是暂时产生、但不能视为已稳定前缀。

换句话说，`vLLM` 已经有了**零散的 commit 语义**，只是没有统一协议名义。

### 5.5 `kv_events` 已经把“状态发布与外部可见性”做成一等机制

`vllm/distributed/kv_events.py`

- `BlockStored`
- `BlockRemoved`
- `extra_keys`
- `KVEventAggregator.get_common_events()`

`vllm/config/kv_events.py`

- 允许通过 `zmq` 向外发布 KV 事件

这说明：

- `vLLM` 里“什么时候一个 block 可以对外可见”已经不是纯内部细节；
- 发布时机、全 worker 一致性和撤销路径都已经具备原型基础。

### 5.6 `pause_generation()` 已经提供显式 pause 模式

`vllm/engine/protocol.py`

- `mode="abort" | "wait" | "keep"`

`vllm/v1/engine/async_llm.py`

- `pause_generation()`
- `resume_generation()`

这说明：

- request 生命周期本身已经支持 pause / keep / resume；
- 但 pause 期间 prefix state 的 reuse visibility 仍没有统一规则。

### 5.7 当前 prefix cache 仍然是“粗粒度稳定态”

`vllm/v1/core/kv_cache_manager.py`

- `get_computed_blocks()` 仍然要求 full block
- 全命中时还要重算最后 token

这说明：

- 当前 APC 更接近“稳定且可 hash 的 full-block state”；
- 它并没有覆盖 richer lifecycle。

## 6. 因此，这个 idea 到底可不可做

我的判断是：

- **可做**
- **但要收窄**
- **而且第一版必须轻量**

### 6.1 最小可行版本

第一版不应该试图一次性覆盖所有状态。

建议只做下面这组最小协议：

- `private`
  - 状态只属于当前 request，不可复用
- `published`
  - 状态可被系统探测，但仅允许受控复用
- `committed`
  - 状态稳定，可进入普通 prefix reuse
- `paused`
  - 请求暂停，状态需保留但不活跃
- `aborted`
  - 状态必须撤销，不再复用

`branched` 可以先不做，避免范围过大。

### 6.2 第一版实现不需要改 kernel

只改协议、元数据和 scheduler/manager 即可：

1. 在 `Request` 里显式增加 lifecycle class；
2. 在 `KVCacheManager` 里区分 published block 和 committed block；
3. 在 `AsyncScheduler` 与 streaming session 更新里，把 today 的 placeholder/confirmed 逻辑抽出来；
4. 在 `kv_events` 里补 lifecycle transition；
5. 在 reset / preemption / discard path 上增加 revoke 逻辑。

这条路线符合你的“轻量级”要求。

### 6.3 是否会有优化效果

我认为**会有，但 workload 依赖明显**。

更可能受益的场景：

- streaming input / session continuation
- tool-call pause/resume
- structured output + async scheduling
- human-in-the-loop interruption后恢复
- agentic workflow 中的短暂停顿与快速恢复

不太可能明显受益的场景：

- 纯 one-shot chat
- 没有 pause/resume 的普通 batch serving
- 只关注静态共享 system prompt 的 APC-heavy workload

所以，这条线的卖点更像：

- correctness-preserving reuse boundary
- resume latency reduction
- recompute reduction
- memory-time efficiency

而不是一开始就赌“所有 workload 都会有大吞吐提升”。

## 7. 这条线能不能支撑系统论文

### 7.1 能，但前提严格

要达到系统论文强度，必须回答三个问题：

1. 为什么现有 `vLLM` 的分散规则不够？
2. 统一 lifecycle protocol 带来的收益是否超过单点 patch？
3. 这件事是否超越 agent workload 的特殊技巧？

### 7.2 需要的证据链

至少要有四类证据：

- 语义证据
  - 当前 `vLLM` 已有 private/placeholder/confirmed/paused/revoked 的碎片化实现
- 正确性证据
  - 统一协议能避免 premature reuse、stale visibility、resume inconsistency
- 性能证据
  - TTFT / resume latency / recompute tokens / memory-time occupancy 改善
- 边界证据
  - 在非 agent 的 async / streaming / structured serving 中也成立

### 7.3 OSDI 风险判断

如果只做“pause 后把 KV 留住”，不够。

如果只做“early publish some blocks”，也不够。

如果能证明：

- 当前 `vLLM` 的 prefix state 生命周期已经碎片化；
- 这些碎片化规则导致 correctness conservatism 与性能损失；
- 一个轻量 unified protocol 能统一 pause/resume、streaming continuation、async placeholder commit；

那么它仍有希望成为一篇像 OSDI 的系统论文。

但我必须诚实地说：

- **风险中高**
- **需要 prototype 先把 thesis 证实**
- **现在还不能说“领先业界最新论文”**

更准确的说法是：

- 它**可能**领先于“单阶段机制优化”的现有工作；
- 但前提是我们把命题收窄得足够清楚，并用原型把收益做实。

## 8. 对命名与论文包装的建议

我建议现在开始避免把最终论文名字直接写成 `TxnStateKV`。

原因有二：

- `TransKV` 已经把 transactional KV 这个说法做得很显眼；
- 我们真正的卖点不是 speculative transaction，而是 prefix state lifecycle protocol。

更安全的命名方向：

- `LifecyclePrefixKV`
- `StatePublishKV`
- `PrefixState Protocol for vLLM`
- `Reusable Prefix State Lifecycle for vLLM`

内部讨论仍可保留 `TxnStateKV`，但对外投稿标题最好换掉。

## 9. 最终判断

截至 `2026-04-15`，我对这个方向的最终判断是：

- **不是零重合方向**
- **不是可以宣称“确保没人做过”的方向**
- **但仍然是目前所有候选里，最有机会在 `vLLM` 上落成“完整系统故事”的方向之一**

更具体地说：

- 它比 `ResidualModeKV` 更不容易被 `LAPS` 直接压死；
- 它比 `TypedPrefixKV` 更像完整系统 thesis；
- 它比 `PolyStateKV` 更轻量，也更贴近 `vLLM` 当前真实代码路径；
- 但它必须从“transactional KV”收窄成“prefix state lifecycle protocol”。

如果我们接受这一定义，那么它是：

- **可实现的**
- **可能有效的**
- **值得继续的**

但不是：

- 已经证明领先
- 已经证明足够发 OSDI
- 可以断言全球无人做过

## 10. 下一步最小验证任务

如果决定继续，我建议按下面顺序做，不要再发散：

1. 做 telemetry
   - 统计 `placeholder / confirmed / published / paused / revoked` 之间的转换频率。
2. 做最小协议原型
   - 先只做 `published -> committed -> paused -> aborted`。
3. 做三组 workload
   - async streaming
   - tool-call pause/resume
   - structured-output async serving
4. 做对照
   - baseline `vLLM`
   - 只保留 pause state
   - 只做 early publish
   - unified lifecycle protocol

如果这四步里，统一协议拿不到明显收益，这条线就应该尽早降级。

## 11. 参考资料

- [InferCept: Efficient Intercept Support for Augmented Large Language Model Inference](https://arxiv.org/abs/2402.01869)
- [KVFlow: Efficient Prefix Caching for Accelerating LLM-Based Multi-Agent Workflows](https://arxiv.org/abs/2507.07400)
- [Tokencake: A KV-Cache-centric Serving Framework for LLM-based Multi-Agent Applications](https://arxiv.org/abs/2510.18586)
- [Serve Programs, Not Prompts](https://arxiv.org/abs/2510.25412)
- [Pie: A Programmable Serving System for Emerging LLM Applications](https://arxiv.org/abs/2510.24051)
- [Continuum: Efficient and Robust Multi-Turn LLM Agent Scheduling with KV Cache Time-to-Live](https://arxiv.org/abs/2511.02230)
- [ThunderAgent: A Simple, Fast and Program-Aware Agentic Inference System](https://arxiv.org/abs/2602.13692)
- [Transactional KV Caching for Speculative Decoding under Paged KV Memory](https://d197for5662m48.cloudfront.net/documents/publicationstatus/307056/preprint_pdf/197af17fbcff12ca9844d78e933bd197.pdf)
- [5x Faster Time to First Token with NVIDIA TensorRT-LLM KV Cache Early Reuse](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)
