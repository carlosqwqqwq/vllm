# ObservationKV 深挖评估：重复性、可行性、有效性与论文潜力

## 1. 先给结论

截至 `2026-04-15` 的这轮公开检索与源码核对，我对 `ObservationKV` 的判断是：

- **这不是一个已经被公开系统论文完整做掉的命题**；
- **但它也不能再写成“支持 prompt_logprobs + APC”**，因为 `vLLM` 社区已经在 `2025-03-08` 合并了保守兼容方案；
- **真正还没被系统化完成的空白**是：如何让 prefix cache 从“只服务纯生成”升级成“服务混合 observation API 的统一执行基座”，并且避免当前的 near-full recomputation；
- **在 vLLM 上落地是可行的**，而且第一版不需要改训练，不需要新硬件，也不一定需要重写 attention kernel；
- **论文潜力是中等偏上**，但前提是命题要拔高到 mixed-API serving substrate，而不是停留在 feature patch。

更坦率一点说：

- 我**不能保证**“全世界绝对没人做过”；
- 但以当前公开论文、GitHub 社区讨论、以及 vLLM 主线实现来看，`ObservationKV` 仍然存在一段**可防守的新空间**。

## 2. 我们到底在研究什么

这里先把 `ObservationKV` 的命题收紧，否则很容易和已有工作重合。

### 2.1 不够好的表述

下面这些写法都太危险：

- “让 `prompt_logprobs` 支持 APC”
- “给 pooling / scoring 也加 prefix cache”
- “缓存 hidden states 以复用更多 API”

原因很简单：

- 第一条已经被 vLLM 社区做过保守兼容；
- 第二条太像功能补齐；
- 第三条又容易和 hidden-states API 扩展、通用 processor 接口混在一起。

### 2.2 更可防守的精确定义

我建议把 `ObservationKV` 定义为：

> 在统一 LLM serving 系统中，prefix cache 不应只缓存“足够继续 decode 的状态”，还应支持一类带精确语义的 observation 请求，使系统可以在命中共享前缀后，以比全量 prefill 重算更低的代价，恢复 prompt logprobs、token-level observation、scoring / pooling 所需的观测结果。

这一定义有三个关键点：

1. **目标是 mixed-API serving，而不是单一 API 修补。**
2. **坚持 exact semantics，不做近似值替代。**
3. **核心矛盾是“KV 可复用，但 observation 结果不可直接从 KV 读出”。**

## 3. 目前业界最接近的工作是什么

### 3.1 最近邻一：vLLM 的 `prompt_logprobs + APC` RFC 与已合并 PR

最相关的公开工程记录是：

- [vLLM RFC: Prompt logprobs + APC compatibility #13414](https://github.com/vllm-project/vllm/issues/13414)
- [vLLM PR: [V1] Prompt logprobs + APC compatibility; prompt logprobs reqs cannot fill APC #13949](https://github.com/vllm-project/vllm/pull/13949)

它们的重要信息不是“这个问题存在”，而是“社区已经选了一个保守但不彻底的解法”：

- RFC 明确指出：APC 命中后，现有缓存里没有 prompt logprobs，难点在于如何恢复 top-k prompt logprobs；
- RFC 给出的三个候选方案里，最激进的是“利用缓存 KV 进行重算”，最保守的是直接绕过 APC；
- 最终合并的 PR 采用的实际策略是：
  - **APC 可以在实例级开启；**
  - **但 `prompt_logprobs` 请求自己不能命中 prefix cache；**
  - **它们会做 near-full recomputation，并把自身产生的 KV 提供给其他普通请求复用。**

这说明：

- 该问题**确实不是空想**；
- 但主线社区当前只把它处理成**兼容性补洞**，还没有把它上升为混合 API 的统一缓存语义问题。

### 3.2 最近邻二：hidden states processor / prompt hidden states 社区方向

第二组重要信号是：

- [vLLM RFC: Hidden states processor #12249](https://github.com/vllm-project/vllm/issues/12249)
- [vLLM RFC: Support Returning Prompt Hidden States #24288](https://github.com/vllm-project/vllm/issues/24288)

这两条说明社区已经意识到：

- 生成、embedding、classification、reward、prompt hidden states 这些需求正在汇合；
- 现有“生成 runner”和“pooling runner”的边界越来越不自然；
- 用户希望在一次调用里获得更多 observation，而不是再发第二次请求。

但这条线和 `ObservationKV` **还不是一回事**。

它们更像：

- 如何暴露 hidden states；
- 如何设计统一 processor 接口；
- 如何让一个模型支持更多输出。

而 `ObservationKV` 关心的是：

- **这些 observation 请求命中共享前缀后，系统怎样避免全量重算。**

所以它们是**强相关社区背景**，但不是同题替代品。

### 3.3 最近邻三：把 KV cache 用于“不是纯 decode”的论文

在公开论文里，我目前找到最接近的概念邻居是：

- [Beyond Speedup – Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)

这篇工作的价值在于：

- 它证明了 KV cache 不一定只是 decode 加速副产品；
- KV 还可以作为轻量表示，被用于采样与推理相关流程。

但它与 `ObservationKV` 仍有明显边界：

- 它不是在做 `vLLM` 这类 serving runtime 的 mixed-API 执行语义；
- 它讨论的是“KV 还能拿来做什么”，不是“命中 APC 后如何恢复 prompt logprobs / pooling / scoring 观测结果”。

所以它更适合作为**概念支持**，而不是直接重合项。

### 3.4 最近邻四：TGI 对 prefill logprobs 的工业取舍

工业侧也有强信号：

- [Hugging Face TGI v3 overview](https://huggingface.co/docs/text-generation-inference/en/conceptual/chunking)

这个文档的价值不在于提供相同机制，而在于说明：

- 预填阶段的 logits / logprobs 很贵；
- TGI 默认把这类能力关掉；
- 如果开启 `--enable-prefill-logprobs`，会牺牲 prompt size 上限。

这恰好支持了 `ObservationKV` 的问题设定：

- observation 不是白送的；
- 系统如果没有单独优化语义，很容易在“更丰富输出”和“吞吐/容量”之间硬碰硬。

### 3.5 最近邻五：其他 prefix cache 系统论文

如果把视野拉大到 prefix cache 论文，像：

- [Preble: Efficient Distributed Prompt Scheduling for LLM Serving](https://arxiv.org/abs/2407.00023)
- [ShadowServe: Interference-Free KV Cache Fetching for Distributed Prefix Caching](https://arxiv.org/abs/2509.16857)

它们研究的是：

- 路由与调度；
- 分布式缓存获取；
- 网络/SmartNIC/跨副本 fetch；
- 多 GPU / 多副本场景下的 prefix sharing。

这些工作与 `ObservationKV` **不构成直接重复**，因为它们默认复用对象仍是“可继续推理的 KV 状态”，而不是“带精确 observation 语义的请求恢复机制”。

## 4. vLLM 目前到底是什么状况

这部分只说本仓库当前代码里能直接验证的事实。

### 4.1 `prompt_logprobs` 仍然默认绕过 prefix cache 读取

在 `vllm/sampling_params.py:431-435`：

- 如果请求带 `prompt_logprobs`，默认会令 `skip_reading_prefix_cache=True`；
- 注释直接写出原因：开启 prefix cache 时，prompt logprobs 的输出可能少于 `n_prompt_tokens`，因此要跳过读缓存。

这意味着当前语义仍然是：

- **KV 不是不能复用；**
- **而是 observation 输出和“已命中的缓存块”之间没有被统一建模。**

### 4.2 `token_embed` / `token_classify` 等 observation-heavy pooling 也会跳过

在 `vllm/pooling_params.py:130-137`：

- `token_embed` 与 `token_classify` 默认会把 `skip_reading_prefix_cache` 置为 `True`；
- 原因同样写得很直接：pooling 输出可能少于 prompt token 数，不能简单按 APC 命中块来回填结果。

这里要注意一个边界：

- 不是**所有** pooling 都不支持 prefix caching；
- 测试已经表明，某些 causal attention + LAST/ALL pooling 的模型可以支持；
- 真正卡住的是**更细粒度、与 token 对齐更复杂、或者需要额外元数据的 observation 类输出**。

### 4.3 `kv_cache_manager` 里的系统级策略仍然是“直接跳过命中”

在 `vllm/v1/core/kv_cache_manager.py:188-192`：

- 注释明确写出：如果请求需要 `prompt logprobs` 或 all-pooling，就跳过 prefix cache 命中查找。

也就是说，系统级策略并不是：

- “先命中，再想办法补 observation”

而是：

- “只要 observation 语义复杂，就别走这条 fast path”

这正是 `ObservationKV` 要打穿的主矛盾。

### 4.4 观测结果当前是从 `hidden_states -> logits/pooler` 路径里算出来的

在 `vllm/v1/worker/gpu_model_runner.py` 里可以看到两条关键路径：

- `3145-3147`：pooling 结果来自 `model.pooler(hidden_states=..., pooling_metadata=...)`
- `3464-3468`、`5014-5095`：prompt logprobs 来自 `_get_prompt_logprobs_dict(...)`
  - 先切出 prompt 对应的 `hidden_states`
  - 再调用 `model.compute_logits(...)`
  - 然后 `sampler.gather_logprobs(...)`

这说明一个非常重要的事实：

- **ObservationKV 不能被误解为“多存一点 KV 就够了”。**

因为：

- prompt logprobs 需要 logits；
- pooling / classify / score 需要 hidden states 和任务特定 processor；
- 这些都不是从现有 APC block 里直接读出来的现成产物。

### 4.5 vLLM 已经部分支持某些 pooling 场景，但不是我们要解决的全部

测试 `tests/test_config.py:778-844` 说明：

- causal attention + LAST/ALL pooling 的部分模型支持 prefix caching；
- bidirectional attention 模型一般不支持；
- LAST/STEP 组合也存在不支持情况。

这进一步说明：

- `ObservationKV` 不能写成“让所有 pooling 都用 prefix cache”；
- 更精确的命题应该是：**让当前由于 observation 恢复困难而被强制绕开的那一类请求，获得 exact 语义下的更优复用路径。**

## 5. 我们的 idea 是否已经重复

### 5.1 如果命题写得太窄，重复风险高

下面三种写法，我认为都已经不够新：

1. “prompt_logprobs 与 APC 兼容”
2. “返回 prompt hidden states”
3. “给更多 pooling 任务统一 hidden-states processor”

因为这些点要么已经 merge，要么已经有明确 RFC 在推进。

### 5.2 如果命题写成 mixed-API observation substrate，重复风险明显下降

如果我们明确主张：

- 研究对象不是单个 API；
- 研究问题是“共享前缀命中后，如何在 exact semantics 下恢复 observation 输出”；
- 研究贡献包含请求分类、执行计划、状态恢复、结果拼装、以及调度影响；

那它与现有公开工作之间仍然有清楚边界。

我现在给出的重复性判断是：

- **与公开论文直接重复：低到中低**
- **与 GitHub 社区工程方向局部相邻：中高**
- **被审稿人误判为 feature completion 的风险：高**

这三个判断并不矛盾。

真正的危险不是“已经有同题论文”，而是：

- **如果你只做一个 API，会看起来像补洞；**
- **如果你把问题提升为 mixed-API cache semantics，就有机会变成系统命题。**

## 6. 这个方向是否足够支撑，是否有效

我认为它是**足够支撑的**，但必须满足下面三个条件。

### 6.1 条件一：必须坚持精确语义，不做模糊近似

如果 `ObservationKV` 做成：

- 命中 APC 后，拿近似 logits 或近似 hidden states 来糊 observation 输出；

那会非常危险：

- 正确性边界不清楚；
- 更像模型近似优化；
- 也更容易被现有近似 KV/稀疏 attention 论文吸走。

更稳妥的方向是：

- **exact observation semantics**
- **replay only where needed**
- **cache only what is safe to reuse**

### 6.2 条件二：必须做成统一执行计划，而不是单点 if-else

我建议第一版机制拆成三层：

1. **Observation class**
   - 把请求分成：
     - pure generation
     - prompt-logprob observation
     - token-aligned observation
     - pooled / scored observation
2. **Recovery plan**
   - 对每类请求选择：
     - pure reuse
     - reuse + selective replay
     - reuse + lightweight checkpoint materialization
     - full recompute
3. **Executor support**
   - 在已有 APC 命中后，执行受控 replay，而不是直接把请求整个打回无缓存路径。

只有这样，论文才能从“参数兼容”变成“系统执行语义”。

### 6.3 条件三：必须覆盖至少两类 observation API

如果只覆盖 `prompt_logprobs`：

- 工程上很有价值；
- 论文上说服力偏弱。

更合理的组合是：

- `prompt_logprobs`
- 至少一种 token-level observation（如 `token_embed` 或 `token_classify`）
- 再选一种 scoring / pooling 场景作为第三类工作负载

这样才能证明：

- 你解决的是“观测类请求的统一 cache gap”，而不是一个参数特判。

## 7. 在 vLLM 中到底能不能实现

我的判断是：**能，而且第一版就能做出像样 prototype。**

### 7.1 为什么说实现上可行

因为关键部件都已经存在，只是还没有被组织成新的语义路径：

- APC 命中查询已存在；
- chunked prefill 已存在；
- `_get_prompt_logprobs_dict` 已经能按 chunk 回填 observation；
- pooling runner 已经能基于 `hidden_states` 与 `pooling_metadata` 输出结果；
- 当前系统只是把 observation-heavy 请求提早打入“禁止 APC 命中”的分支。

所以第一版更像是：

- **重写执行策略**
- 而不是
- **重写整个模型执行栈**

### 7.2 我认为最可行的第一版机制

我更看好下面这个版本，而不是上来就缓存大量 hidden states。

#### 方案 A：Selective Observer Replay

核心思路：

- APC 命中后，不把命中前缀视为“完全不可观测”；
- 对 observation 请求，只在必要范围内做 replay；
- replay 的目标不是重新生成全部 KV，而是恢复 observation 所需的最小状态。

对 `prompt_logprobs`：

- 可以复用命中的 KV 作为 attention 上下文；
- 只为需要输出 logprob 的 token 重新走 logits 路径；
- 避免当前 near-full recomputation 的保守实现。

对 token-level pooling / classify：

- 可以只对 observation 覆盖的 token 区间做补算；
- 与 `pooling_metadata` 结合输出最终结果。

#### 方案 B：Observation Checkpoints

如果 selective replay 还不够稳，可以增加少量检查点：

- 不缓存完整 prompt hidden states；
- 只缓存恢复某类 observation 所需的轻量检查点；
- 例如块边界上的小型中间表示、token range descriptors、或可验证的 replay anchors。

这一层的价值是：

- 减少 replay 长度；
- 控制额外显存成本；
- 给不同 observation class 提供统一接口。

### 7.3 为什么我不建议第一版就“缓存完整 hidden states”

因为这会很快踩中三个问题：

- 显存/CPU 内存开销太大；
- 序列化与输出成本高；
- 更容易和“返回 prompt hidden states”社区方向重合。

第一版最好坚持：

- **KV 仍然是主缓存对象**
- **observation 只通过 replay 或轻量 checkpoint 恢复**

这会更轻量，也更符合你的约束。

## 8. 这个方向是否真的会有效

我认为**有明确生效空间**，但不是所有场景都赚。

### 8.1 最可能显著受益的场景

1. **长共享前缀 + 多轮 prompt scoring / prompt logprobs**
   - 比如 rerank、评测、agent prompt 诊断、偏好比较。
2. **统一 API 网关中的混合请求**
   - 同一套服务同时承载 generate、score、embed、classify。
3. **RAG / Agent 工作负载**
   - 大段 system prompt、工具说明、知识片段高度共享；
   - 但请求输出不总是纯文本生成。
4. **长上下文研究型接口**
   - 用户需要 prompt-level logits、hidden-state derived outputs、或 token-level 观测。

### 8.2 收益指标应该怎么看

这类工作不能只看单一吞吐。

我建议至少看：

- observation 请求的 TTFT
- end-to-end latency
- 同 GPU 上 mixed workload 的 aggregate throughput
- APC hit rate 不变时的有效重算 token 数
- 单请求额外显存/CPU 占用

如果机制成立，应该看到：

- observation 请求的全量 prefill 重算显著下降；
- 与纯生成请求共存时，总体吞吐不再被 observation 请求严重拖累；
- 不需要关闭 APC 或开第二套服务才能兼容混合 API。

### 8.3 哪些场景未必有效

需要主动承认三类弱点：

1. **短 prompt 场景**
   - replay 优化空间很小；
   - 调度和 bookkeeping 成本可能吃掉收益。
2. **完全不共享前缀的工作负载**
   - 没有 APC 命中，自然没有 ObservationKV 收益。
3. **encoder-only / bidirectional 模型**
   - 现有 prefix caching 支持本来就弱；
   - 第一版论文最好不要把这些场景当主战场。

## 9. 它够不够 OSDI / CCF-A

### 9.1 只做单点兼容，不够

如果最后结果只是：

- “我们让 `prompt_logprobs` 比以前快了”

那我认为**不够 OSDI**，甚至高水平系统会也不稳。

因为审稿人很可能会问：

- 这不就是 vLLM 的一个缺失特性吗？
- 为什么不是工程 patch？
- 为什么不是再加一个 flag？

### 9.2 做成下面这个结构，才有系统论文味道

如果我们把论文组织成：

1. **问题定义**
   - 现有 prefix cache 只服务 generation-only fast path；
   - mixed-API serving 中 observation 请求被迫退化到 full recompute。
2. **抽象**
   - 定义 observation class、exact semantics、recovery plan。
3. **机制**
   - APC hit 后的 selective replay / checkpoint / stitching。
4. **系统实现**
   - 嵌入 vLLM scheduler、kv manager、worker runner。
5. **评估**
   - 多类 API、混合负载、长共享前缀、与现有保守基线比较。

那它就开始有系统论文的骨架了。

### 9.3 我当前对投稿潜力的评级

截至目前，我给 `ObservationKV` 的评级是：

- **工程可行性：高**
- **vLLM 落地性：高**
- **轻量级程度：高**
- **普适性：中高**
- **高吞吐/低延迟潜力：中高**
- **与已公开论文直接重合风险：中低**
- **被看成功能补丁的风险：高**
- **如果故事讲好后的 OSDI 潜力：中**

这不是“最稳必赢”的 OSDI 方向，但它是一个**真实缺口很强、实现门槛适中、可以比较快做出 prototype** 的方向。

## 10. 我建议如何改造这个 idea，才能更像论文

### 10.1 不要叫“prompt_logprobs cache”

这会直接把命题做窄。

更好的主张应该是：

- `ObservationKV`
- mixed-API observation-aware prefix caching
- exact observation recovery on cache hits

### 10.2 明确把现有 vLLM 实现当作 baseline，而不是 novelty

必须明确承认：

- vLLM 已经通过 `#13949` 解决了实例级兼容性；
- 但它本质上还是**保守绕过 APC hit + near-full recomputation**；
- 我们的目标不是“第一次支持”，而是“第一次系统化降低 exact observation 的恢复代价”。

### 10.3 第一版只打最强三类 workload

我建议别贪多，主打：

1. `prompt_logprobs`
2. 一种 token-level observation
3. 一种 score / pooling workload

这样：

- story 足够统一；
- 实现范围还可控；
- 更能体现轻量与普适。

## 11. 下一步最值得做的验证

如果后面要把它继续打磨成主线 idea，我建议最先做下面几项最小验证：

1. **测当前 vLLM 基线**
   - 对共享长前缀的 `prompt_logprobs` 请求，量化 near-full recomputation 的真实代价。
2. **做一个最小 selective replay prototype**
   - 先只支持 `prompt_logprobs`；
   - 验证能否在 exact 输出不变前提下减少重算 token。
3. **扩展到第二类 observation**
   - 例如 `token_embed` 或 `token_classify`；
   - 验证统一 recovery plan 是否真的能复用。
4. **跑 mixed-workload 实验**
   - generate + prompt_logprobs + score 同时跑；
   - 证明这不是单 API 特殊优化。

## 12. 最终判断

如果你问我一句最简短的结论，我会这样回答：

> `ObservationKV` 不是“绝对没人碰过”的真空地带，但它仍然没有被现有公开论文和 vLLM 主线实现完整覆盖。只要我们把问题定义成 mixed-API serving 中的 exact observation recovery，而不是单 API feature patch，它依然是一个可做、能在 vLLM 上落地、也有机会长成 CCF-A 系统论文的方向。

## 13. 参考链接

- [vLLM RFC: Prompt logprobs + APC compatibility #13414](https://github.com/vllm-project/vllm/issues/13414)
- [vLLM PR: [V1] Prompt logprobs + APC compatibility; prompt logprobs reqs cannot fill APC #13949](https://github.com/vllm-project/vllm/pull/13949)
- [vLLM RFC: Hidden states processor #12249](https://github.com/vllm-project/vllm/issues/12249)
- [vLLM RFC: Support Returning Prompt Hidden States #24288](https://github.com/vllm-project/vllm/issues/24288)
- [vLLM Issue: Add efficient interface for evaluating probabilities of fixed prompt-completion pairs #5234](https://github.com/vllm-project/vllm/issues/5234)
- [Beyond Speedup – Utilizing KV Cache for Sampling and Reasoning](https://arxiv.org/abs/2601.20326)
- [Hugging Face TGI v3 overview](https://huggingface.co/docs/text-generation-inference/en/conceptual/chunking)
- [Preble: Efficient Distributed Prompt Scheduling for LLM Serving](https://arxiv.org/abs/2407.00023)
- [ShadowServe: Interference-Free KV Cache Fetching for Distributed Prefix Caching](https://arxiv.org/abs/2509.16857)
