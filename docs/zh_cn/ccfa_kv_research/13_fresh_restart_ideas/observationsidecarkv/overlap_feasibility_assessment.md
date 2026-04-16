# ObservationSidecarKV 深度复审：重复性、可行性与投稿潜力评估

更新日期：2026-04-15

## 1. 严格结论

先给结论，不绕弯：

- 截至 `2026-04-15`，我**不能诚实地说“确保没人做过”**。
- 基于这轮检索，我**没有找到一篇与 refined `ObservationSidecarKV` 完全同构的公开系统论文**。
- 但我同样不能把原始版 `ObservationSidecarKV` 继续当成“低重合、安全创新”的方向。

更准确地说：

- 如果把它写成“让 `prompt_logprobs` 兼容 APC”，那会**直接退化成 feature patch**；
- 如果把它写成“缓存 hidden states / hidden cache 来服务 mixed API”，那会**明显靠近** `Apt-Serve` 与 `HCache`；
- 如果把它写成“把 KV 当 embedding/representation 来做下游任务”，那又会**明显靠近** `Beyond Speedup / KV-Embedding`；
- 如果把它收敛成“**exact-prefix 场景下的 typed observation sidecar / observation contract cache**”，它仍然有研究空间，而且比 `WritePathKV` 更可救。

所以最诚实的判断是：

> 原始版 `ObservationSidecarKV` 不能直接拿去当顶会主 thesis。
> 但它在**重写 thesis**之后，仍然可能成为一个可做、可发、并且确实能在 `vLLM` 落地的候选方向。

## 2. 这轮检索到的高风险近邻

### 2.1 `prompt_logprobs + APC` 已经是公开讨论过的工程问题

资料：

- [vLLM RFC: Prompt logprobs + APC compatibility](https://github.com/vllm-project/vllm/issues/13414)

它说明了什么：

- `prompt_logprobs` 与 APC 的冲突已经被明确提出；
- 核心困难也已经被说得很清楚：
  - APC 命中后，prefix token 会被视为“已计算”；
  - 但这些 token 的 prompt logprobs 并没有被缓存下来；
  - 于是系统需要在“recompute / cache logprobs / bypass APC”之间取舍。

为什么危险：

- 这意味着如果我们的故事只是“prompt logprobs 也应该支持 APC”，那不是新论文问题，而是**已知工程缺口**。

### 2.2 `Apt-Serve`：一旦 sidecar 变成 hidden cache，重合就会很高

资料：

- [Apt-Serve: Adaptive Request Scheduling on Hybrid Cache for Scalable LLM Inference Serving](https://arxiv.org/abs/2504.07494)

它做了什么：

- 提出 hybrid cache；
- 组合 KV cache 与 **memory-efficient hidden cache**；
- 用可复用的 input hidden state vectors 提升并发与吞吐。

为什么危险：

- 如果 `ObservationSidecarKV` 最后需要缓存大块 hidden states 或 hidden vectors 来服务 pooling / token tasks，
  那它会明显落入 `Apt-Serve` 的叙事范围；
- 那时你很难继续把它讲成“极小 sidecar”，而更像“另一种 hidden cache”。

严格判断：

- **sidecar 必须是 finalized observation outputs，而不能退化成 hidden cache。**

### 2.3 `HCache`：中间激活恢复也会压缩空间

资料：

- [Fast State Restoration in LLM Serving with HCache](https://arxiv.org/abs/2410.05004)

它做了什么：

- 不从原 token 重算，也不单纯依赖 KV offload；
- 而是从 intermediate activations 恢复状态；
- 目标是加速 state restoration。

为什么危险：

- 如果 `ObservationSidecarKV` 往“中间层激活 sidecar”方向走，
  就会越来越像 state restoration，而不是 mixed-API observation reuse；
- 这会把问题重新拖回系统已有主线。

### 2.4 `Beyond Speedup / KV-Embedding`：KV 作为表示复用已经有人讲了

资料：

- [Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)

它做了什么：

- 把 KV cache 看成 lightweight representation；
- 用于 sampling / reasoning；
- 明确提出不必总是重算或存 full hidden states。

为什么重要：

- 这说明“把 KV 或内部状态当表示来做别的事”已经不是空白；
- 如果我们把 `ObservationSidecarKV` 讲成“KV 不只用于 decode，也能用于 embedding/scoring”，
  那会与它高度相近。

严格判断：

- **我们的卖点不能是‘内部状态还能干别的’；必须是 exact-prefix mixed-API serving 的 typed observation reuse。**

### 2.5 `vLLM` 自己已经有一个 observation-cache 特例：late interaction query cache

资料：

- `vllm/v1/worker/gpu/pool/late_interaction_runner.py`

它已经做了什么：

- 缓存 query token embeddings；
- 用 `query_key` 做 worker-side cache lookup；
- 再对 document 请求做 late-interaction scoring。

为什么重要：

- 这说明“缓存 observation outputs 而不是只缓存 KV”在 `vLLM` 自己内部已经不是完全陌生的东西；
- 如果 `ObservationSidecarKV` 只是把这个特例扩写成更大概念，但没有更强的统一抽象，审稿人会问：
  - 这是不是只是把已有 special case 泛化了一下？

### 2.6 prompt embeds / mixed API compatibility 也已经在社区议程里

资料：

- [vLLM Issue: Prefix Caching support when Prompt Embeds is enabled](https://github.com/vllm-project/vllm/issues/25096)

为什么重要：

- 这说明社区已经不再把 prefix reuse 只看成“纯 token prompt 的生成优化”；
- mixed API / prompt embeds / auxiliary outputs 的 APC 兼容性，正在逐步变成公开工程议题。

结论：

- **问题是真实的，但这也意味着“只是指出兼容性缺口”不够新。**

## 3. 本地代码复审后的关键修正

这里有一个很重要的修正：原始版 `README` 把 mixed API 的现状说得有点过宽。

### 3.1 不是所有 pooling 都在绕过 APC

从 `vllm/pooling_params.py` 看：

- 默认强制 `skip_reading_prefix_cache=True` 的是：
  - `token_embed`
  - `token_classify`
- 其它 pooling task 默认并**不会**统一跳过 APC。

这意味着：

- “pooling 整体不兼容 APC”这个说法不准确；
- 真正的系统缺口更集中在：
  - `prompt_logprobs`
  - token-level observation tasks
  - 部分 partial-prefill 场景

### 3.2 `CLS/MEAN` 不是完全不支持，但 partial prefill 有硬约束

从 `seqwise/methods.py` 看：

- `CLSPool` 和 `MeanPool` 显式要求：
  - `partial prefill not supported`
- `LastPool` 没有这个断言。

这说明：

- seq-wise pooling 并不是“完全不能复用”；
- 但它也没有被纳入一个更完整的 typed mixed-API reuse 设计。

### 3.3 tokenwise pooling 已经有 per-request sidecar 胚芽

从 `tokwise/methods.py` 和 `v1/pool/metadata.py` 看：

- chunked prefill 下已经会把 `hidden_states_cache` 暂存在 `PoolingStates`；
- prefill 完成后再拼接并送给 pooler head。

这说明：

- 系统内部已经承认某些 observation path 需要额外 side state；
- 只是它今天还是 **per-request ephemeral state**，不是 **cross-request prefix-aligned reusable sidecar**。

### 3.4 `prompt_logprobs` 路径已经有 CPU staging side state

从 `gpu_model_runner.py` 看：

- `num_prompt_logprobs`
- `in_progress_prompt_logprobs_cpu`
- `_get_prompt_logprobs_dict(...)`

已经形成了比较完整的 prompt-logprob 生产路径。

这说明：

- `prompt_logprobs` 并不是不可处理；
- 只是这些状态今天没有被 APC/prefix reuse 纳入。

## 4. 哪些表述现在已经不安全

经过这轮调研，下面这些说法我不建议再用：

- “`ObservationSidecarKV` 是第一个让 mixed API 复用 prefix cache 的系统”
- “我们首次支持 prompt_logprobs + APC”
- “我们首次缓存 observation sidecar”
- “我们首次把 hidden states 之外的输出纳入 prefix cache”
- “我们首次让 pooling / scoring / logprobs 统一复用”

这些表述要么已经落入已知工程问题，要么会撞上 hidden cache / representation reuse / local special-case prior art。

## 5. 目前唯一较可防守的 refined thesis

如果继续做这条线，我建议把 thesis 收敛成下面这个版本：

> 在 `vLLM` 这类 exact-prefix serving runtime 中，可复用对象今天主要被定义为 KV state；但真实服务 API 还会请求一类“prefix-deterministic observations”，例如 prompt logprobs、token-level classification scores、token embeddings、部分 pooled outputs。我们提出一个 **typed observation sidecar / observation contract cache**，把这些 deterministic observation outputs 作为 prefix cache 的伴生对象来管理，并在 contract-compatible 时直接复用、在 contract-incompatible 时执行 selective replay 或 full recompute。

这里的关键词不是 “sidecar” 本身，而是：

- `typed`
- `prefix-deterministic`
- `finalized observation outputs`
- `contract-compatible reuse`

### 5.1 为什么一定要强调 `typed observation contract`

因为不同 API 的输出契约本来就不同，例如：

- `prompt_logprobs`: `top-k`
- `token_embed`: `dimensions`, `use_activation`
- `token_classify`: `use_activation`
- step pooling: `step_tag_id`, `returned_token_ids`

如果不把这些参数显式纳入 reuse contract：

- 就无法保证 exact semantics；
- 也无法解释为什么某些请求可以 sidecar hit、某些必须 selective replay。

### 5.2 为什么一定要强调“存 final outputs，不存 hidden states”

因为一旦你开始存 hidden states：

- 论文边界会迅速靠近 `Apt-Serve` 和 `HCache`；
- sidecar 就不再轻量；
- 也更难讲出“普适 mixed API serving substrate”。

而如果存的是最终 observation outputs：

- `prompt_logprobs`: 每 token top-k 结果，远小于 full hidden states；
- `token_classify`: 每 token logits/score，通常远小于 KV；
- `seqwise embed/classify`: 每请求一小段输出向量，体积更小；
- `late interaction`: query token embeddings 已经说明这种做法在工程上能成立。

## 6. vLLM 里为什么它仍然可实现

### 6.1 请求和运行时已经显式区分 mixed API

`Request`、`SamplingParams`、`PoolingParams` 和 `gpu_model_runner` 当前已经显式携带：

- `prompt_logprobs`
- pooling task
- `dimensions`
- `use_activation`
- `late_interaction_params`

这说明：

- mixed API 不是假设中的未来需求；
- 而是现在就存在的运行时异构接口。

### 6.2 现有 prefix cache 跳过逻辑已经是天然插点

`Request.get_skip_reading_prefix_cache()` 和 `kv_cache_manager.get_computed_blocks()` 已经形成了清晰的决策点：

- 命中 APC
- 跳过 APC

这让第一版 prototype 很容易插入：

- `contract hit -> reuse sidecar`
- `contract miss but prefix hit -> selective replay`
- `no usable cache -> full recompute`

### 6.3 `prompt_logprobs` 路径已经有完整的生产与 CPU staging 逻辑

`_get_prompt_logprobs_dict(...)` 说明：

- prompt-logprob 结果已经按 request 管起来；
- 也已经有 chunked prefill 对应的 incremental accumulation；
- 差的是跨请求 prefix reuse，不是功能从零开始写。

### 6.4 pooling 路径已经有 metadata/state 基础设施

`PoolingMetadata`、`PoolingStates`、`hidden_states_cache`、`pooler heads` 已经提供了：

- task metadata
- per-request transient state
- finalized outputs 生产点

这意味着：

- 你不需要先去改 attention kernel；
- 也不需要先改模型结构；
- 最小实现可以落在 `request / scheduler / model_runner / pooler output cache` 这一层。

### 6.5 `late_interaction_runner` 反而是很好的工程证据

它说明：

- `vLLM` 团队自己已经接受“有些 observation outputs 值得被缓存并跨请求复用”；
- 我们真正要做的是：
  - 把这种 special-case feature 提升为统一 typed reuse substrate，
  - 而不是重复造一个 query cache。

## 7. 最保守也最现实的原型该怎么做

### 7.1 P0：只做 characterization 和 instrumentation

先回答四个问题：

1. `prompt_logprobs`、`token_embed`、`token_classify`、late interaction 各自有多少 shared-prefix 重算？
2. 这些 API 的 observation outputs 大小，相对 KV 的比例是多少？
3. contract fragmentation 有多严重？
   - `top-k`
   - `dimensions`
   - `use_activation`
   - `step_tag_id / returned_token_ids`
4. mixed batch 下，这些 observation requests 对 TTFT/throughput 造成了多大拖累？

如果这一步看不到强 pathology，这条线就不该继续。

### 7.2 P1：不要先碰 hidden states，先做 finalized output sidecar

第一版建议只覆盖三类：

1. `prompt_logprobs`
2. `token_classify`
3. `token_embed`

为什么是这三类：

- 它们今天确实更接近 APC 缺口；
- 输出契约明确；
- sidecar 可以直接存 finalized outputs；
- 不需要为了论文新意退化成 hidden cache。

### 7.3 `prompt_logprobs` 需要 contract-compatible reuse，而不是一把梭

这里最大难点不是能不能存，而是：

- 以后来的请求可能要求更大的 `k`。

所以第一版最合理的机制不是“总能命中”，而是：

- 若 `requested_k <= cached_k`：直接 sidecar hit
- 若 `requested_k > cached_k`：selective replay
- 若 prefix 本身不命中：full recompute

这正是 `observation contract cache` 相对 feature patch 更像论文的地方。

### 7.4 seqwise pooling 不该当成第一目标

原因很简单：

- 有些 seqwise pooling 现在已经能走 APC；
- 真正没覆盖的点更多是 partial prefill 和 task-typed outputs；
- 如果把这部分当主目标，问题会显得不够尖锐。

### 7.5 把 late interaction 纳入统一框架，但不要把它当主卖点

最好的做法是：

- 用它作为“已有局部 observation cache”的例子；
- 然后展示 unified substrate 如何覆盖：
  - prompt-logprob sidecar
  - token output sidecar
  - late-interaction special case

而不是反过来把整篇论文讲成 late-interaction cache。

## 8. 这条线要成立，实验上必须证明什么

### 8.1 必须先证明 mixed API 的重算浪费是真问题

必须量化：

- shared-prefix observation requests 中，多少比例今天被迫 full recompute；
- 这些请求造成了多少额外 prefill FLOPs；
- 在 mixed workload 下，它们是否拖累了 generation requests 的 TTFT 和 throughput。

### 8.2 必须证明 sidecar 真的是“小”的

必须画出：

- `sidecar bytes / KV bytes`
- `sidecar bytes / hidden-cache bytes`

否则审稿人会直接问：

- 你这不就是换个名字存一份 hidden states 吗？

### 8.3 必须证明 contract-aware fallback 是必要的

如果实验里看不出：

- parameter mismatch 很常见；
- selective replay 比 full recompute 更值；

那整篇论文就很容易被打成：

- “缓存一个 API 输出”的 feature patch。

### 8.4 必须覆盖多个 API family

至少要覆盖：

- `prompt_logprobs`
- `token_classify`
- `token_embed`
- 一种已有 special-case observation cache（如 late interaction）

否则 unified substrate 的故事站不住。

## 9. 目前我对论文价值的判断

### 9.1 原始版 `ObservationSidecarKV`

- **创新性：中等**
- **重合风险：中等**
- **顶会安全性：一般**

原因：

- 问题真实；
- 但故事太宽；
- 且部分子问题已是公开工程议题，另一些子问题又会撞到 hidden cache / representation reuse。

### 9.2 refined `typed observation contract cache`

- **创新性：中高**
- **重合风险：中**
- **vLLM 落地性：高**
- **顶会安全性：中等偏上**

原因：

- 它把“支持某个 API”提升成“typed reuse substrate”；
- 它把 generic hidden cache 排除出去；
- 它有清晰的 exact semantics / compatibility / fallback 抽象；
- 也有很强的 `vLLM` 本地实现证据。

但我要强调：

- 这依然**不是“稳稳没人做过”的方向**；
- 它是否能冲到 `OSDI` 级别，取决于你能否把 problem characterization 和 typed-contract 抽象做得足够硬。

## 10. 当前建议：go / no-go

我的建议是：

- **原始版 `ObservationSidecarKV`：No-Go**
- **refined `typed observation contract cache`：Go/Maybe**

更具体地说：

1. 不要继续沿着“又给 APC 支持一个 API”这个方向写。
2. 不要把 sidecar 做成 hidden cache。
3. 把贡献收敛成：
   - exact-prefix
   - typed observation contracts
   - finalized output sidecars
   - contract-aware fallback
4. 先做 `P0 characterization`；如果 mixed API 重算和 contract fragmentation 不够尖锐，就立刻止损。

## 11. 参考资料

- vLLM RFC: Prompt logprobs + APC compatibility: <https://github.com/vllm-project/vllm/issues/13414>
- vLLM Feature Request: Prompt Embeds + Prefix Caching: <https://github.com/vllm-project/vllm/issues/25096>
- Apt-Serve: <https://arxiv.org/abs/2504.07494>
- HCache: <https://arxiv.org/abs/2410.05004>
- Beyond Speedup / KV-Embedding: <https://arxiv.org/abs/2601.20326>
