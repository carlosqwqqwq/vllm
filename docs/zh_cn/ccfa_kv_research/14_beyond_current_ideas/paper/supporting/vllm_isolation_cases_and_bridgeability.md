# vLLM 复用隔离实例与 Bridgeability 说明

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 把“哪些逻辑上可复用的 exact-prefix state 在 `vLLM` 中被主动隔离”这件事，整理成一个可复用的 reviewer note / discussion note

它不负责：

- 重新定义 `FalseNegativeKV` thesis
- 重新展开完整排重
- 重新设计 `TwinPathServe`

它面向的使用场景很明确：

1. 给 reviewer 解释“为什么不是 hash 没打通”
2. 给正文、引言、discussion 或 rebuttal 提供可复用的 case 库
3. 帮助内部统一口径：哪些 case 是 `P1` 可安全恢复的，哪些只用于证明问题存在

从现在开始，这份文档的固定任务是：

- 给出 `vLLM` 中“复用隔离”的源码级证据
- 解释每个隔离为什么存在
- 解释在 `reuse contract` 理论下，哪些隔离是 bridgeable 的

## 2. 一个总判断

先给最重要的总判断：

> 在 `vLLM` 中，很多“没有复用”的 exact-prefix state，并不是因为系统找不到前缀，也不是因为 hash 天然失效；
> 更常见的原因是：系统在 request 语义、backend capability 或 input carrier 这些 consumer-side boundary 上，主动做了保守隔离。

因此，本文讨论的问题不能写成：

- cache key 没打通
- 某个 prefix 明明在 hash 表里但没被拿出来
- prefix caching 偶尔失效

更准确的写法应该是：

- 语义等价的前缀计算，已经在参考执行条件下被证明可复用
- 但当前请求由于 `reuse contract` 不满足，没能把这部分既有计算兑现为自己的物理命中收益

这里有三层概念必须分清：

1. `identity`
   - 这份前缀 state 是不是“同一个东西”
   - 主要由 hash / cache key 回答
2. `legality`
   - 当前 consumer 在自己的语义和正确性约束下，能不能合法消费这份 state
   - 这是 `reuse contract` 要回答的问题
3. `recoverability`
   - 如果当前不能直接消费，是否能通过一个有限、可验证的 bridge 来恢复复用
   - 这是 `bridge operator` 要回答的问题

这三层里：

- hash 只解决第一层
- reviewer 的质疑通常集中在第二层
- `FalseNegativeKV` 真正要贡献的是第二层和第三层

## 3. 什么叫“被 vLLM 隔离了”

这份文档里，“被 `vLLM` 隔离了”不是笼统说“miss 了”，而是指下面三类源码里可直接看到的行为。

### 3.1 request-level isolation

即当前请求被显式标记为：

- `skip_reading_prefix_cache = True`

这意味着：

- 即便 prefix 在逻辑上已存在
- 当前请求也不会去读 prefix cache

这是最直接、最典型的隔离方式。

### 3.2 backend-level isolation

即当前执行类直接把：

- `enable_prefix_caching = False`

这不是单个请求的 miss，而是整个 backend compatibility class 不允许消费 prefix cache。

### 3.3 consumer-level isolation

即 consumer 侧直接声明：

- 不兼容
- 不支持 partial prefill
- 需要 token ids / prompt observation 等额外能力

这类 case 最能说明：

- 问题不在 hash
- 问题在 consumer-side contract 没被显式桥接

## 4. 五个核心 case 一览

| Case | 隔离形态 | `vLLM` 当前为什么不复用 | 我们的理论分类 | 当前是否适合作为 `P1` bridge | 当前最稳角色 |
| --- | --- | --- | --- | --- | --- |
| `prompt_logprobs` | request-level isolation | 需要 prompt-level observation，裸 KV 不足以直接产出完整 prompt logprobs | `observation mismatch` | 是 | 主案例 |
| `FLASHINFER/TRITON_MLA + batch invariance` | backend-level isolation | 当前 backend class 不支持 prefix caching | `backend mismatch` | 是 | 主案例 |
| `token_embed` / `token_classify` | request-level isolation | pooling / token task 的输出语义不等于 generation 语义 | `task/observation mismatch` | 否 | supporting case |
| `CLS/MEAN` pooling + partial prefill | consumer-level isolation | consumer 没有 partial state composition rule | `partial-prefill mismatch` | 否 | supporting case |
| `prompt_embeds` + `prompt_logprobs` / token-id-sensitive task | consumer-level isolation | hash 可定位 identity，但 carrier 不能满足 token-id / observation contract | `carrier mismatch` | 否 | 边界论证 case |

这张表的用途是：

- 让 reviewer 先看到“不是所有隔离都要被去掉”
- 让内部写作时明确区分：
  - 哪些是 `P1` 强 claim
  - 哪些只是证明 thesis 不是单点 bug

## 5. Case 1：`prompt_logprobs` 被 request-level 隔离

### 5.1 隔离点在哪里

第一处关键证据在：

- [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)

这里的默认逻辑是：

- 如果 `skip_reading_prefix_cache` 尚未显式指定
- 只要 `prompt_logprobs is not None`
- 就把 `skip_reading_prefix_cache` 置为真

源码注释写得非常直接：

- `the output of prompt logprobs may less than n_prompt_tokens`
- `we need to skip reading cache at this request`

第二处证据在：

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:242)

这里 `KVCacheManager` 明确写道：

- 如果 `request.skip_reading_prefix_cache`
- 就直接跳过 prefix cache lookup

也就是说，`vLLM` 在这个 case 上做的不是“查了没中”，而是“连查都不查”。

### 5.2 `vLLM` 为什么这样隔离

`vLLM` 的这个保护并不是拍脑袋定的，而是来自 worker 端 prompt-logprob 的 materialization 语义。

关键证据在：

- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:5056)

这里 prompt logprobs 的生成起点直接绑定：

- `request.num_computed_tokens`

并且后续会：

- 从本次 request 对应的 `hidden_states` 切片
- 重新算 logits
- 再 gather 对应 token 的 prompt logprobs

这意味着：

1. prompt logprobs 不是一个现成附着在 KV 上的字段
2. 它依赖当前 request 重新 forward 出来的 prompt hidden states
3. 如果前缀因为 prefix hit 被直接跳过计算，那么对应位置的 prompt observation 就不会被 materialize

所以这里的隔离，本质上是在保护：

- prompt-level output semantics
- `prompt_logprobs` 的完整性与对齐关系

### 5.3 为什么这不是 hash 问题

因为这里的核心矛盾根本不在“这个 prefix 能不能被定位”，而在：

- 现有 prefix KV 是否足以支持当前 consumer 所需的 prompt observation

换句话说：

- hash 只能回答“是不是同一个前缀”
- 不能回答“当前请求是否已经具备返回完整 prompt logprobs 的能力”

### 5.4 为什么在我们的理论下它是 bridgeable 的

这是当前最干净、最适合 `P1` 恢复的 case。

原因是：

1. producer state 已经非常明确
   - prefix KV 已经存在
2. consumer 缺的能力也非常明确
   - prompt-level observation
3. 缺口可以由一个有限、可验证的 bridge 补齐

这个 bridge 不是“打开一个开关”，而是：

1. 对命中边界之前的 prefix，消费最小 `observation sidecar`
2. 主动放弃命中边界的最后一个 cache block
3. 从这个 block 起点做 `boundary replay`
4. 如果 sidecar 不存在或 replay 前提不满足，就 fallback 到 full prefill

这里的关键不是“我们反对 `vLLM` 的保护逻辑”，而是：

> `vLLM` 正确识别到了“裸 KV 不足以支持 prompt_logprobs”；
> 我们进一步指出，这个 gap 可以通过 `KV + minimal observation sidecar + boundary replay` 被安全桥接。

### 5.5 这个 case 在论文里该怎么写

最稳的写法：

- `prompt_logprobs` exposes an observation mismatch
- current `vLLM` conservatively skips prefix-cache reads because prompt observations are materialized only for tokens recomputed in the current request
- a contract-aware bridge can recover most exact-prefix reuse without violating prompt-logprob semantics

最不该写的说法：

- `we simply enable APC for prompt_logprobs`
- `vLLM mistakenly disables prefix cache`
- `this is only a missing feature flag`

## 6. Case 2：`FLASHINFER/TRITON_MLA + batch invariance` 被 backend-level 隔离

### 6.1 隔离点在哪里

关键证据在：

- [attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/attention.py:319)
- [mla_attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/mla_attention.py:377)

这两处逻辑都非常明确：

- 如果开启 prefix caching
- 同时启用 `VLLM_BATCH_INVARIANT`
- 且 backend 是 `FLASHINFER` 或 `TRITON_MLA`
- 系统就直接把 `enable_prefix_caching = False`

同时还会设置：

- `prefix_caching_disable_reason = "backend_incompatibility"`

### 6.2 `vLLM` 为什么这样隔离

这里的注释已经给出最重要的信息：

- `prefix caching + batch invariance is currently not supported`

也就是说，这不是：

- prefix identity 不可判定
- cache key 不稳定
- hit 查找逻辑出 bug

而是：

- 当前 backend execution class 根本不支持 prefix caching 这条能力组合

这里 `vLLM` 保护的是：

- backend capability boundary
- correctness / supportedness boundary

### 6.3 为什么这也不是 hash 问题

因为在这个 case 里，系统甚至不是“查表失败”，而是从更早一层就把 prefix caching 整体关掉了。

这清楚说明：

- hash 不是充分条件
- 就算 prefix identity 可以确定
- 当前 backend 仍可能不允许消费这份 state

### 6.4 为什么在我们的理论下它仍然是 bridgeable 的

这里的 bridge 不是：

- 在不支持的 backend 上强行打开 prefix caching

而是：

- 把 backend compatibility class 显式纳入 `reuse contract`
- 让不同 family 的请求进入适合自己的执行池

更具体地说：

1. `throughput_optimized_pool`
   - 保留 aggressive fast path
   - 不要求它无条件支持 prefix reuse
2. `reuse_optimized_pool`
   - 采用 cache-friendly backend / config
   - 承担高 false-negative 风险 family
3. route 决策
   - 不再假设所有 backend class 对 exact-prefix state 具有同样的消费能力

因此这个 case 的论文意义在于：

> 我们不是要取消 backend-level isolation，而是要把它从“隐式 backend 实现细节”提升为“显式 runtime contract + routing decision”。

### 6.5 这个 case 在论文里该怎么写

最稳的写法：

- cross-backend compatibility classes can induce logical false-negative exact hits
- `vLLM` already exposes this as a backend support boundary
- a twin-pool runtime can recover reuse by routing compatible families to a reuse-friendly pool instead of forcing unsupported backends to consume the cache

最不该写的说法：

- `we make FLASHINFER fully support prefix caching under batch invariance`
- `backend mismatch is just a temporary implementation bug`

## 7. Case 3：`token_embed` / `token_classify` 被 request-level 隔离

### 7.1 隔离点在哪里

关键证据在：

- [pooling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/pooling_params.py:130)

这里默认逻辑是：

- 对 `token_embed`
- 对 `token_classify`
- 如果没有显式指定，就把 `skip_reading_prefix_cache = True`

### 7.2 `vLLM` 为什么这样隔离

源码注释给出的理由是：

- pooling 的输出可能少于 `n_prompt_tokens`
- 因此当前 request 需要跳过 cache read

这说明系统在保护的不是“prefix 查找正确性”，而是：

- token-level / pooling-style output semantics

也就是说，这类 task 的 consumer 需求并不是：

- “继续生成后续 token”

而是：

- “返回 token-level embedding / classification / pooling result”

这和 generation 的消费语义不是一回事。

### 7.3 为什么这类 case 对我们的 thesis 仍然重要

它的重要性不在于它已经适合做 `P1` 实现，而在于它证明：

- `prompt_logprobs` 不是单点 feature bug
- request-level isolation 在 mixed-API workload 中是更普遍的现象

换句话说，这类 case 支撑的是“问题面”而不是“当前恢复面”。

### 7.4 为什么在我们的理论下它原则上也可被桥接

如果某类 pooling / token task 的输出对 prefix 部分是可缓存、可组合的，那么理论上可以：

1. 对 exact-hit prefix 发布最小 sidecar
2. 对 suffix / boundary 部分继续 fresh compute
3. 在 output 层把 prefix sidecar 和 suffix 结果组合起来

但当前我们没有足够证据证明：

- 哪些 pooling operator 具备稳定、统一的组合规则
- 这种 bridge 是否值得进入第一版系统范围

因此最稳妥的定位是：

- 这是一个 supporting case
- 它证明 `reuse contract` 缺口具有普遍性
- 但不应在当前论文中写成已恢复的主结果

## 8. Case 4：`CLS/MEAN` pooling 不支持 partial prefill，而 tokenwise pooling 已展示出局部组合能力

### 8.1 隔离点在哪里

关键证据在：

- [seqwise/methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:43)
- [seqwise/methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/seqwise/methods.py:67)

这里 `CLSPool` 和 `MeanPool` 都直接断言：

- `partial prefill not supported`

但在另一路 tokenwise pooling 中：

- [tokwise/methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/tokwise/methods.py:63)

可以看到它已经支持：

1. 先缓存 chunked hidden states
2. prefill 结束后再拼接

### 8.2 这个对比为什么重要

它说明：

- “能不能消费分段 state”不是一个 prefix caching 天然决定的事实
- 而是 consumer 有没有 composition rule 的问题

也就是说：

- `CLS/MEAN` 当前不能消费 partial prefill
- 并不等于“所有 pooling 都不可能消费 partial state”

tokenwise pooling 已经证明：

- 只要定义了显式组合规则
- 分段状态是可以被组合成最终输出的

### 8.3 为什么这个 case 对 reviewer 很有说服力

因为它把问题从“prefix cache 是否命中”进一步推进到了：

- consumer operator 是否有稳定的 state composition contract

这正是 `reuse contract` 想显式化的东西。

### 8.4 为什么在我们的理论下它有桥接空间

这里甚至可以给出非常具体的直觉例子：

1. `CLS pooling`
   - 如果输出只依赖某个固定位置的 hidden state
   - 那么 exact-hit prefix 中的该位置向量理论上可以 sidecar 化
2. `MEAN pooling`
   - 如果输出等于 token hidden states 的平均值
   - 那么 prefix 部分未必需要保留全部 hidden states
   - 只要保留 prefix 的 `sum` 和 `count`
   - 再与 suffix 的 `sum/count` 组合，也能恢复最终结果

这说明缺失的不是数学上的可组合性，而是：

- runtime 没有为这些 consumer 提供统一的 bridge/operator contract

### 8.5 为什么当前不应把它写成主贡献

因为我们还没有：

- 为这类 pooling task 设计统一、稳定的 bridge 接口
- 证明其 correctness / cost model

所以当前最稳写法是：

- 它是一个极强的 supporting case
- 用来证明 false-negative isolation 不是单点问题
- 不应被包装成当前系统已恢复的主结果

## 9. Case 5：`prompt_embeds` 已进入 hash，但 carrier 仍可能不兼容

### 9.1 关键证据在哪里

第一处关键证据在：

- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)

这里 `prompt_embeds` 已经进入 block hash 的 extra keys 计算路径。

这件事非常重要，因为它直接说明：

- 在 `prompt_embeds` 场景下，系统并不是完全没有做 identity matching

第二处关键证据在：

- [render/serving.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/serve/render/serving.py:335)

这里 API 层直接拒绝：

- `prompt_logprobs` + `prompt_embeds`

第三处关键证据在：

- [tokwise/methods.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/pooler/tokwise/methods.py:85)

`StepPool` 明确声明：

- `requires_token_ids=True`

### 9.2 这个 case 为什么最适合回答“为什么不能只靠 hash”

因为它几乎构成了一个完整的反例链：

1. `prompt_embeds` 已经被纳入 hash
2. 说明 prefix identity 可以被定位
3. 但某些 consumer 仍不能合法消费这份 state
4. 原因是 consumer 需要 token ids 或 token-level observation，而 carrier 只有 embeds

所以这个 case 可以直接支持下面这句话：

> hash 是必要条件，但不是充分条件。

### 9.3 `vLLM` 在这里保护的是什么

保护的是：

- carrier compatibility
- token-id-sensitive consumer requirement

也就是说，即便 state identity 已经匹配，系统仍然要问：

- 当前 consumer 是否拿到了自己完成语义所需的载体信息

### 9.4 为什么这个 case 当前不应进入 `P1`

理论上，这类 case 可以通过更强的 carrier-side contract 来桥接，比如：

1. 发布 token-id provenance
2. 发布 carrier-aligned observation sidecar
3. 限制到某些 token-free consumer

但当前这些能力都还没有被稳定定义。

因此最稳妥的定位是：

- 这是一个边界论证 case
- 它最重要的价值是回答 reviewer 的 hash 质疑
- 不应被写成当前系统已解决的贡献点

## 10. 从这五个 case 提炼出来的统一结论

这五个 case 加在一起，要支撑的不是：

- `vLLM` 有很多 bug
- prefix cache 支持还不够全
- 只要多打通几个 feature 就好了

它们共同支撑的是下面更强、更统一的判断：

> 现代 LLM serving runtime 中，exact-prefix reuse 不是一个“命中了就赚，没命中就算了”的纯机会主义优化；
> 很多逻辑上已经成立的命中，会在 computation 开始前，因为 consumer-side contract 不满足，而被 runtime 主动隔离。

因此：

- `logical hit`
  - 说的是“在参考执行条件下，这份 state 已被证明可复用”
- `physical hit`
  - 说的是“当前请求真正兑现出来的复用”
- `false-negative hit`
  - 说的是“前者与后者之间，因 contract mismatch 丢失掉的那部分收益”

## 11. 给 reviewer 的固定回答模板

### 11.1 回答“那你得是在安全的前提下复用”

可以直接回答：

> 我们同意。`vLLM` 当前隔离这些路径并不是偶然，而是为了保护 output semantics、backend capability 和 carrier compatibility。我们的观点不是取消这些保护，而是区分其中哪些隔离只是因为缺少 bridge，因此可以在 correctness-preserving 的前提下恢复；若 bridge 前提不满足，请求仍应 fallback 到原始 full prefill 或原始执行路径。

### 11.2 回答“如果真的在 vLLM 中专门被隔离的话，肯定是有原因的”

可以直接回答：

> 是的，这正是本文的问题起点。`prompt_logprobs`、backend incompatibility、pooling partial-prefill 不支持、以及 carrier mismatch 都表明：`vLLM` 已经隐式暴露出多种 consumer-side contract boundary。本文不是否认这些原因，而是把这些原因从 scattered checks 提升为显式的 `reuse contract` 与 `bridge operator` 语言。

### 11.3 回答“要不可以直接用 hash，不存在找不到前缀的事”

可以直接回答：

> hash 只解决 prefix identity，不解决当前请求是否合法消费这份 state。`prompt_embeds` 已经进入 block hash，但 `prompt_logprobs` 仍与 `prompt_embeds` 不兼容，这恰好说明 identity matching 是必要条件，但不是充分条件。本文讨论的不是“找不到前缀”，而是“前缀 state 已在参考执行条件下被证明可复用，但当前 consumer contract 下不能直接兑现”。

## 12. 当前写作和实现上的固定取舍

从现在开始，这些 case 在论文中的角色应固定如下：

### 12.1 `P1` 强 claim

- `prompt_logprobs`
- `backend incompatibility` / twin-pool routing

### 12.2 supporting evidence

- `token_embed` / `token_classify`
- `CLS/MEAN` pooling 与 partial prefill

### 12.3 边界论证

- `prompt_embeds` / carrier mismatch

这样做的好处是：

1. 问题面足够广，不会被理解成单点 patch
2. 主系统恢复面仍然足够克制，不会过度承诺
3. reviewer 的三类典型质疑都能被源码级 case 直接回应

## 13. 一句可复用总结

如果只保留一句最该复用的话，这句应该固定为：

> 在 `vLLM` 中，很多未复用的 exact-prefix state 不是“找不到”，而是 runtime 为了保证 consumer 语义、backend 能力或 carrier 正确性而主动隔离；`FalseNegativeKV` 的目标不是去掉这些保护，而是把其中可 bridge 的隔离，转化为 contract-aware 的安全复用。
