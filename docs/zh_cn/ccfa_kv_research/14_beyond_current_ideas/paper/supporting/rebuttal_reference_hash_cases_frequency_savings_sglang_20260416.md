# Rebuttal 参考：Hash、隔离原因、出现频率、收益上界与 SGLang 对照

更新日期：2026-04-16

## 1. 这份文档回答什么

这份文档集中回答审稿人与导师最容易追问的七个问题：

1. 既然 prefix KV 已经存在，为什么没有被复用？
2. KV cache 不是靠 hash 吗，为什么还会出现这种情况？
3. 如果 `vLLM` 专门把这些路径隔离掉，肯定有原因；为什么这些 case 仍然可能被安全复用？
4. 请给出具体 case，而不是抽象概念。
5. 这种情况出现的频率到底有多高？
6. 即便全部优化，理论上到底能节省多少开销？
7. `SGLang` 中有没有类似现象？

这份文档不负责：

- 替代正文
- 把当前证据边界夸大成“所有 case 都已闭环”
- 把 supporting evidence 写成主实验

## 2. 先给最短结论

最短结论可以直接写成三句：

1. 本文讨论的问题不是“hash 找不到 prefix”，而是“hash 只能建立 identity，不能保证当前 consumer 在自己的语义与正确性约束下合法消费这份 state”。
2. `vLLM` 中确实存在多类“state 已存在但当前路径主动不消费”的显式隔离，这些隔离有其正确性原因；但其中一部分隔离并不是不可逾越的边界，而是可以在显式 `SafeReuseContract` 下被 bridge。
3. 从当前测量看，这类现象在目标 workload 上并不稀薄。尤其在长上下文与异构 backend 路径中，token-weighted 损失很大；如果能够在安全边界内恢复，会直接转化为显著的 prefill 节省和延迟收益。

## 3. 为什么“已经存在但没被复用”

### 3.1 必须先拆开三层概念

围绕“为什么已经存在但没被复用”，最容易混淆的是下面三层概念：

1. `identity`
   同一个 prefix 是否已经被定位到。
   这主要由 hash、prefix key 或 family identity 回答。
2. `legality`
   当前 consumer 在自己的 API 语义、backend compatibility 和 input carrier 约束下，能否合法消费这份 state。
   这由 `reuse contract` 回答。
3. `recoverability`
   如果当前不能直接消费，系统是否可以通过一个有限、可验证且可回退的 bridge 恢复复用。
   这由 `bridge operator` 回答。

`FalseNegativeKV` 讨论的问题不在第一层，而在第二层和第三层。也就是说，问题不是“有没有这份 state”，而是“当前请求是否允许直接使用它”，以及“如果不允许直接使用，是否能在安全条件下恢复”。

### 3.2 hash 只能回答“是不是同一个前缀”

hash 的作用是：

- 把当前请求的 prefix identity 对应到已有缓存条目

但它不能回答：

- 当前请求是否需要额外 observation
- 当前 backend class 是否允许消费 prefix cache
- 当前 input carrier 是否满足 token-id-sensitive consumer 的语义要求
- 当前执行路径是否要求更严格的确定性或其他 correctness target

因此，“hash 能找到 prefix”与“当前请求能安全消费 prefix KV”是两件不同的事。

### 3.3 在 `vLLM` 里，很多 case 连 lookup 都不会做

最直接的证据是 [sampling_params.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/sampling_params.py:431)：

- 当 `skip_reading_prefix_cache is None` 时，
- 只要 `prompt_logprobs is not None`，
- 就把 `skip_reading_prefix_cache` 置为真。

对应源码注释非常清楚：

- prefix caching 打开时，`prompt logprobs` 的输出可能少于 prompt token 数
- 因此当前请求需要跳过 prefix cache read

进一步的执行点在 [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:194)：

- 如果 request 被标记为 `skip_reading_prefix_cache`
- 且没有显式忽略这个标记
- 直接返回空命中，不做 prefix cache hit 查找

这说明 `prompt_logprobs` 这个 case 不是“查了但没找到”，而是“出于 consumer 语义原因，系统主动不查”。

## 4. 如果 `vLLM` 专门隔离，为什么还会有可 bridge 的 case

### 4.1 隔离有原因，这一点必须承认

如果 `vLLM` 把一条路径专门隔离，通常都不是随意而为，而是在保护某个 correctness boundary。本文不能把这些隔离写成：

- 错误实现
- 丢了一个 feature flag
- hash 没打通

更准确的说法是：

- 现有运行时正确识别到了“裸 KV 不足以满足当前 consumer contract”
- 但它没有继续区分“绝对不可复用”和“可在明确条件下被 bridge”的 case

### 4.2 `prompt_logprobs`：隔离原因是 observation completeness

在 [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:5082) 中，`prompt_logprobs` 的 materialization 明确依赖本次 request 的：

- `hidden_states`
- `request.num_computed_tokens`
- 当前 request 的 `prompt_token_ids`

这说明：

1. prompt logprobs 不是天然附着在 KV 上的现成字段
2. 如果 prefix 因为 cache hit 被跳过计算，相应位置的 prompt observation 不会自动被产出
3. 因而 `vLLM` 的保守 skip 在语义上是合理的

但这并不意味着该 case 不可复用。它只说明：

- 不能直接把“已有 KV”当成“完整 prompt observation”

只要 bridge 能补齐 observation 缺口，这个 case 仍然可以被安全恢复。当前第一类 bridge 的做法是：

1. 用最小 `observation sidecar` 覆盖命中边界之前已经存在的 prompt observation
2. 主动放弃最后一个命中 block
3. 从命中边界做 `boundary replay`
4. 任何前提不满足时 fallback 到 `full prefill`

因此，这不是“推翻了 `vLLM` 的保护逻辑”，而是：

> `vLLM` 正确识别到了“裸 KV 不足以支持 prompt logprobs”；  
> 本文进一步指出，这个 gap 可以通过 `KV + minimal observation sidecar + boundary replay` 被安全 bridge。

### 4.3 backend mismatch：隔离原因是 execution compatibility

在 [attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/attention.py:319) 和 [mla_attention.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/model_executor/layers/attention/mla_attention.py:375) 中，`vLLM` 明确写出：

- `prefix caching + batch invariance` 对 `FLASHINFER` 和 `TRITON_MLA` 当前不支持
- 因而在这些 backend class 下禁用 prefix caching

这同样说明：

- 问题不是 prefix identity 没建立
- 而是不同 execution class 对同一份 state 的消费能力不同

对于这类 case，本文并不主张：

- 跨不兼容 backend 直接共享同一份 KV

第二类 bridge 的安全主张更保守：

- 不是“让不兼容 backend 也直接吃这份 state”
- 而是“把本应在兼容执行类中消费的 family，重新路由回兼容执行面”

因此，第二类 bridge 的恢复不依赖不安全消费，而依赖：

1. family 级 compatible-hit evidence
2. backend compatibility class
3. route/admission 时的保守决策

### 4.4 不是所有隔离都要桥接

这一点必须在 rebuttal 中主动说清。并不是所有被隔离的 case 都应该被恢复。当前最稳的区分方式是：

1. `P1` 主恢复面
   - `prompt_logprobs` observation mismatch
   - backend compatibility mismatch
2. supporting case
   - pooling / token task mismatch
   - partial-prefill mismatch
   - carrier mismatch

也就是说：

- 有隔离原因，不等于绝对不可 bridge
- 但有隔离原因，也绝不等于“都应该强行复用”

## 5. 具体 case：为什么它们可以被复用

### 5.1 Case A：`prompt_logprobs`

`vLLM` 当前行为：

- request 级 `skip_reading_prefix_cache = True`
- prefix lookup 直接跳过

为什么当前不直接复用：

- 裸 KV 不能直接产出完整 prompt observation

为什么它仍然可能被安全复用：

- 现有缺口不是 identity 缺口，而是 observation 缺口
- 这个缺口可以由最小 sidecar + boundary replay 补齐
- fallback 条件可以被写成显式契约

安全前提：

1. `PrefixIdentity` 成立
2. sidecar 覆盖范围与 observation schema 对齐
3. replay 起点与预算满足要求
4. 不满足时必须回退到 `full-prefill baseline`

### 5.2 Case B：backend mismatch

`vLLM` 当前行为：

- 某些 backend class 直接禁用 prefix caching

为什么当前不直接复用：

- 当前执行类不保证能合法消费这份 prefix state

为什么它仍然可能被安全恢复：

- 不是要求当前不兼容 backend 直接消费 state
- 而是把 family 送回 reference-compatible execution class

安全前提：

1. family 级 logical-hit evidence 已建立
2. backend compatibility class 可判定
3. 路由发生在 admission 阶段
4. 无法验证时回退到 baseline pool

### 5.3 Case C：`prompt_embeds` / multimodal / token-id-sensitive task

这类 case 更适合作为 supporting case，而不是 `P1` 主恢复面。

例如在 [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:5105) 中：

- 如果 `request.prompt_token_ids is None`
- `prompt_logprobs` 直接不兼容 `prompt embeddings`

这类 case 说明：

- hash 能建立 prefix identity
- 但 carrier 不满足 consumer 所需的 token-id / observation contract

当前它们更适合承担的角色是：

- 证明问题不是单点 feature bug
- 证明“identity 已建立”仍不足以推出“当前可安全消费”

## 6. 出现频率怎么回答

### 6.1 先区分三种“频率”

审稿人追问“出现频率高不高”时，必须先区分三种量：

1. request-level prevalence
   多少请求会遇到这类 false-negative
2. family-level prevalence
   多少共享前缀 family 会稳定暴露这类 false-negative
3. token-weighted prevalence
   这些 false-negative 吞掉了多少本应兑现的 prefix tokens

对 prefix reuse 问题而言，第三个量往往比第一个量更重要。原因很简单：

- 即便某类请求在总请求数中占比不高
- 只要它们携带很长的共享 prefix
- 它们就可能吞掉大量 prefill 计算

因此，如果只看“请求比例”，很容易低估问题的重要性。

### 6.2 当前最强的频率证据

#### `MT-Bench W1`

来自 [formal_experiment_report_20260415_gpu2.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/experiments/formal_experiment_report_20260415_gpu2.md:67)：

- 原始样本数：`80`
- 最终选中请求数：`64`
- family 数：`8`
- `logical_hit_tokens = 11728`
- treatment `physical_hit_tokens = 0`
- `false_negative_tokens / logical_hit_tokens = 1.000`

这说明在当前 `W1` 构造下：

- 选中的 repeated `prompt_logprobs` 请求族中，false-negative 不是偶发现象
- 而是条件化出现率极高的系统性现象

#### `LongBench W1`

来自 [03_FalseNegativeExactHit的测量.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/paper/manuscript_cn/03_FalseNegativeExactHit的测量.md:55)：

- 请求数：`64`
- family 数：`32`
- 平均 prompt 长度：`10191.84`
- `logical_hit_tokens = 483504`
- treatment `physical_hit_tokens = 0`
- `false_negative_tokens / logical_hit_tokens = 1.000`

这条证据比 `MT-Bench W1` 更重要，因为它直接说明：

- 在长上下文 workload 上，即便 request-level 频率仍需放回真实流量中估计，
- token-weighted 频率已经非常高

#### `MT-Bench W5`

来自 [formal_experiment_report_20260415_gpu2.md](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/docs/zh_cn/ccfa_kv_research/14_beyond_current_ideas/experiments/formal_experiment_report_20260415_gpu2.md:117)：

- 原始样本数：`80`
- 最终选中请求数：`64`
- family 数：`8`
- `logical_hit_tokens = 11728`
- baseline throughput `physical_hit_tokens = 5952`
- baseline `false_negative_tokens / logical_hit_tokens = 0.492`

这说明 backend mismatch 不一定总表现为“0 命中”，它也可能表现为：

- reference-compatible path 中可以拿到完整 hit
- throughput-oriented path 中只拿到退化后的部分 hit

因此，第二类问题的频率不能只按“完全归零”来定义，而应按：

- `logical hit` 与 baseline `physical hit` 的差值

### 6.3 怎样诚实回答“如果频率低，会不会只是工程问题”

这类问题的最稳回答是：

1. 如果某类 false-negative 在真实流量中 request-level 占比很低
2. 且 token-weighted 占比也很低
3. 且 bridge 代价高于 recoverable gain

那么它确实更像局部工程优化，而不是系统论文问题。

但当前证据说明，至少在 `W1` 和 `W5` 这两类核心 workload 上，情况不是这样：

1. 它们不是一次性的 lookup bug
2. 它们不是少数请求上的噪声 miss
3. 它们在 token 维度上吞掉了大量可复用 prefix
4. 两类问题共享同一类 consumer-side contract structure

因此，是否构成 systems problem，关键不在“是否 100% 普遍”，而在：

- 是否存在统一的运行时结构
- 是否在 token-weighted 维度上足够显著
- 是否可以由统一控制面而不是散乱 patch 来处理

## 7. 理论上能节省多少开销

### 7.1 最稳的理论上界写法

对请求 `i`，定义：

- `B_i`：bridgeable false-negative tokens
- `alpha_i`：恢复机制的 realization efficiency
- `c_prefill(i)`：该请求在当前模型与 backend 下的单位 prefill token 成本

则理论可节省的 prefill 开销上界可写为：

```text
SavedPrefillCost <= Σ_i alpha_i * B_i * c_prefill(i)
```

如果只做 token-level 代理，而不引入具体 kernel 常数，那么更稳的写法是：

```text
SavedPrefillTokens <= Σ_i alpha_i * B_i
```

相对 prefill 节省比例可以写成：

```text
RelativePrefillSaving
  <= (Σ_i alpha_i * B_i) / (Σ_i prompt_tokens_i)
```

如果要把它转换为端到端时延上界，则还要乘以 prefill 在该 workload 上对总时延的占比 `beta`：

```text
RelativeLatencySaving
  <= beta * RelativePrefillSaving
```

其中：

- 长上下文 workload 的 `beta` 往往较大
- 短 prompt 或 decode-heavy workload 的 `beta` 会较小

### 7.2 用 `LongBench W1` 做一个直接量化

当前已有数字：

- `logical_hit_tokens = 483504`
- 请求数 `64`
- 平均 prompt 长度 `10191.84`
- `prompt_logprobs selective replay` 在 formal validation 中的 `effective_reuse / physical_hit = 1072 / 1088 = 98.5%`

由此可得：

1. 平均每个请求被吞掉的 logical-hit token：

```text
483504 / 64 = 7554.75 tokens / request
```

2. 相对于平均 prompt 长度的 token-weighted 损失比例：

```text
7554.75 / 10191.84 = 74.13%
```

3. 如果用当前第一类 bridge 的 98.5% realization efficiency 做乐观但仍受约束的代理，则有效可回收的 prefill token 比例约为：

```text
74.13% * 98.5% = 73.04%
```

这组数字的含义非常直接：

- 在这个长上下文子负载中，
- 即便不把收益夸大成完整端到端 latency gain，
- 单从 prefill token 角度看，理论上可回收的工作量也已经非常大

### 7.3 用 `MT-Bench W5` 做一个直接量化

当前已有数字：

- `logical_hit_tokens = 11728`
- baseline throughput `physical_hit_tokens = 5952`

则 baseline 中被 backend mismatch 吞掉的比例为：

```text
(11728 - 5952) / 11728 = 49.25%
```

如果看“恢复后相对 baseline 已兑现 hit 的增长”，则有：

```text
(11728 - 5952) / 5952 = 97.04%
```

这说明：

- 在目标 backend mismatch family 上，
- baseline 并不是“找不到前缀”
- 而是只兑现了约一半 logically available 的 prefix hit

如果把这部分恢复回来，节省的就不是边角 token，而是接近 baseline 已兑现量本身的一倍。

### 7.4 为什么不能只看 request 比例

如果某类请求：

- 占总请求数不高
- 但平均 prompt 极长
- 且共享 prefix 很长

那么它在 token-weighted 维度上仍可能主导 prefill 成本。`LongBench W1` 正是这种情况。  
因此，rebuttal 中最稳的说法不是：

- “这类请求占比多少”

而是：

- “这类 false-negative 在 token 维度上吞掉了多少本应兑现的 prefill”

这才是 prefix reuse 论文真正该报告的成本代理。

## 8. 这是不是只是工程问题

### 8.1 什么时候它只是工程问题

如果满足下面三条，那么它更像工程问题：

1. case 稀薄，只在极少数 feature 上出现
2. 吞掉的 prefix token 很少
3. 没有共享的运行时结构，只能逐个 patch 修

### 8.2 为什么当前证据不支持把它简单降级成工程问题

当前最强证据至少说明：

1. `prompt_logprobs` 不是 hash bug，而是 request-level observation mismatch
2. backend mismatch 不是 lookup failure，而是 execution compatibility mismatch
3. 两类现象共享“identity 已建立，但 current consumer contract 不允许直接消费”的结构
4. 在 `W1` 长上下文负载上，token-weighted 损失规模很大
5. 在 `W5` 上，baseline throughput path 丢掉了约 `49.25%` 的逻辑前缀收益

因此，当前更合理的结论是：

- 这不是一个“前缀偶尔没命中”的局部工程 bug
- 而是一类由现代 mixed-API / mixed-backend runtime 系统性暴露出来的 consumer-side runtime pathology

但同样必须诚实地承认：

- 若后续在更大规模流量中，token-weighted prevalence 明显偏低，
- 或 bridge gain 无法覆盖 bridge cost，
- 这项工作就应被重新降级为 feature-local optimization

也就是说，问题能否支撑顶会论文，最终仍取决于：

1. 多 workload prevalence
2. token-weighted cost concentration
3. 恢复收益与系统代价的统一闭环

## 9. `SGLang` 中有没有类似情况

### 9.1 严格结论

有，而且不是只有一处。  
但更准确地说，`SGLang` 当前实现中也存在多类：

- state identity 已建立
- 但当前 consumer path / backend path 不允许直接消费

的 capability boundary。

这并不自动证明 `SGLang` 与本文的 `vLLM` pathology 完全同构，但足以说明：

- 这不是 `vLLM` 独有的一条工程 if-branch
- 而更像现代 serving runtime 的共同结构

### 9.2 具体证据

#### Case 1：`input_embeds` 要求关闭 radix cache

在 [tokenizer_manager.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/managers/tokenizer_manager.py:712) 中：

- 如果请求携带 `input_embeds`
- 且没有设置 `--disable-radix-cache`
- `SGLang` 直接报错，要求关闭 radix cache

这说明：

- carrier 改变后，cache identity 并不自动意味着当前 path 可安全消费这份 state

#### Case 2：multimodal reranker 文档建议关闭 radix cache

在 [rerank_models.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/supported_models/retrieval_ranking/rerank_models.md:313) 中，官方文档明确写道：

- 对多模态内容，推荐使用 `--disable-radix-cache` 以避免 caching issues

这说明：

- multimodal consumer contract 与 radix cache 之间也存在显式边界

#### Case 3：确定性推理与 radix cache 的兼容性不是无条件成立

在 [faq.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/references/faq.md:33) 中，官方 FAQ 明确把 prefix caching 列为 nondeterminism 来源之一，并建议：

- 若希望结果更 deterministic，可加 `--disable-radix-cache`

同时在 [deterministic_inference.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/advanced_features/deterministic_inference.md:25) 的兼容表中：

- `FlashInfer` 在 deterministic inference 下对 `Radix Cache` 的兼容性是 `No`
- `FA3` 与 `Triton` 才是 `Yes`

这进一步说明：

- 相同 prefix state 的可消费性会因 correctness target 与 backend class 而变化

### 9.3 这能支持什么，不能支持什么

它能支持的最稳结论是：

> 类似的 producer-state / consumer-path capability boundary 并非只在 `vLLM` 中出现；`SGLang` 当前实现里也存在多类“identity 已建立，但当前 path 不能直接消费”的场景。因此，本文的问题不应被降级为单框架内的局部工程 patch。

它不能直接支持的更强结论是：

- `SGLang` 已经存在与 `prompt_logprobs selective replay` 完全一一对应的 case
- 本文已经完成跨框架统一实验验证

## 10. 一段可直接复用的长回答

下面这段话可以直接作为长版 rebuttal：

> 我们并不主张这里的问题是“hash 找不到 prefix”。相反，本文的核心观察恰恰是：在许多 case 中，prefix identity 已经建立，甚至参考执行已经证明这段前缀可以稳定复用，但当前 request 仍然无法把这份既有计算兑现为自己的 `physical hit`。原因在于 hash 只解决 identity，而不解决 legality。当前 consumer 是否可以安全消费这份 state，还取决于其 observation requirement、backend compatibility、carrier 语义以及 correctness target。`vLLM` 中已经存在多类显式隔离来保护这些边界，例如 `prompt_logprobs` 会直接设置 `skip_reading_prefix_cache`，某些 backend class 会在 batch invariance 条件下禁用 prefix caching。这些隔离是有原因的，因此本文并不把它们描述为 bug；本文的主张是，其中一部分隔离并不是绝对不可逾越的边界，而是可以在显式 `SafeReuseContract` 下被 bridge。例如，`prompt_logprobs` 的问题不是“prefix KV 不存在”，而是“裸 KV 不足以生成完整 prompt observation”，因此可以通过最小 sidecar 与 boundary replay 在安全条件下恢复；backend mismatch 的问题也不是“查不到前缀”，而是“当前 execution class 不兼容”，因此本文并不跨不兼容 backend 直接共享 state，而是将 family 保守地路由回兼容执行面。从当前证据看，这类现象在目标 workload 上并不稀薄：在 `MT-Bench W1` 和 `LongBench W1` 中，条件化的 false-negative ratio 为 `1.000`；尤其在 `LongBench W1` 上，平均每个请求损失 `7554.75` 个本应兑现的 prefix tokens，占平均 prompt 长度的约 `74.13%`。在 `MT-Bench W5` 中，baseline throughput path 也丢掉了约 `49.25%` 的 logical hit。因而，是否值得研究，关键不只在 request-level 频率，而在 token-weighted 损失是否集中。至少在当前长上下文与异构 backend 两类主案例上，这种损失已经足够大，并且不是单点 feature bug，而是具有统一 consumer-side contract 结构的运行时问题。类似的 capability boundary 在 `SGLang` 中也能观察到，例如 `input_embeds` 需要关闭 radix cache、多模态 reranker 官方建议关闭 radix cache，以及 deterministic inference 下部分 backend 与 radix cache 不兼容。这说明本文的问题更像现代 serving runtime 的共同结构，而不是 `vLLM` 独有的局部工程问题。`

## 11. 最后必须守住的边界

这份文档最后要守住三条边界：

1. 不把问题写成 hash failure
2. 不把安全恢复写成“已有 state 一律可复用”
3. 不把 `SGLang` supporting evidence 写成跨框架主实验

只要这三条边界守住，这组问题就能被比较稳地答住。
