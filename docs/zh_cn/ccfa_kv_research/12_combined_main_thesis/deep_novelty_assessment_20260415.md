# ExactPrefixKV 深度排重与可行性评估

更新日期：2026-04-15

## 1. 结论先行

围绕当前组合主线

- `PromptABIKV`
- `TypedPrefixKV`
- `ResidualModeKV`

做了一轮更严格的公开调研与本地实现核验后，我的判断是：

1. **三条线分别单做，重复风险都偏高。**
2. **把三条线收束成一个统一系统 thesis，仍然有可防守空间。**
3. **截至 2026-04-15，我没有找到与该组合版 thesis 完全同构的公开系统论文。**
4. **但我不能保证没人做过，也不能把论文写成“确保此前无人提出”。**
5. **如果继续推进，最安全的主张不是“又一种 prompt caching 方法”，而是：**

> `ExactPrefixKV`：把 vLLM 中 exact-prefix reuse 从隐式优化升级为一个
> **可声明、可验证、可观测、可调度** 的系统契约。

这条线仍然值得往下做，但前提是要**收窄并重写贡献边界**：

- 不能把 novelty 放在 `prompt caching` 本身；
- 不能把 novelty 放在 “支持 multimodal / prompt embeds / cache_salt” 这些单点 feature 上；
- 不能把 novelty 放在一般性的 prefix-aware scheduling 上。

真正还能防守的，是下面这个组合：

1. **Cacheability ABI**
   - 解释为什么真实 workload 没有稳定地产生 exact prefix。
2. **Typed Exact Prefix Contract**
   - 在 vLLM 内统一 token / prompt embeds / multimodal / salt 的 exactness 语义。
3. **Post-Hit Execution Selection**
   - 命中后不盲目沿用固定路径，而是按 residual work 重新选择执行模式。

## 2. 公开近邻：哪些最危险

下面这些工作，是当前必须正面回应的最近邻。

### 2.1 Prompt Cache

- `Prompt Cache: Modular Attention Reuse for Low-Latency Inference`
- 2023
- 链接：<https://arxiv.org/abs/2311.04934>

它的核心是：

- 把 prompt 切成模块；
- 做结构化复用；
- 允许更灵活的 attention reuse。

它压到的点是：

- “prompt 结构可复用”
- “prompt cache 不是纯 token prefix”

但它**没有直接等价于**我们现在要讲的：

- vLLM runtime 内部的 typed exactness contract；
- 多种 carrier 的统一 hash / validation / observability；
- 命中后的 residual execution-mode re-selection。

### 2.2 OpenAI / Anthropic / Google 的工业 prompt caching

- OpenAI Prompt Caching
  - <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching
  - <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- Google Gemini Context Caching
  - <https://ai.google.dev/gemini-api/docs/caching/>

这些文档已经把很多 ABI 相关事实说得非常清楚：

- OpenAI 文档明确要求 **exact prefix match**，并写明 tools、images、schema 也必须一致；
- Anthropic 文档明确写明缓存的是 `tools -> system -> messages` 的完整前缀，并允许显式 breakpoint；
- Google 也明确建议把大且公共的内容放在 prompt 开头，并把缓存对象提升成显式 cached content。

这说明：

- **“上层输入组织决定缓存命中”** 已经不是新事实；
- **“缓存前缀需要显式边界 / scope / breakpoint”** 也不是新事实。

所以，`PromptABIKV` 单独做，极易被降格成：

- best practice 总结；
- provider 经验归纳；
- prompt engineering 指南。

### 2.3 Don’t Break the Cache

- `Don't Break the Cache`
- 2026
- 链接：<https://arxiv.org/abs/2601.06007>

这篇工作的危险之处在于，它已经把 agentic workload 下：

- tools
- schemas
- dynamic tool results
- prompt block layout
- thinking / reasoning style输入

对缓存命中的影响系统化分析了。

它压到的点是：

- “为什么真实 agent workload 下 prompt caching 不稳定”
- “如何组织输入以提高命中”

所以，如果我们只是说：

- 动态内容放后面；
- 工具定义尽量稳定；
- reasoning 输出不要破坏前缀；

那几乎没有论文空间。

### 2.4 Serve Programs, Not Prompts / Pie / ThunderAgent

- `Serve Programs, Not Prompts`
  - HotOS 2025
  - <https://yecl.org/publications/gim2025hotos.pdf>
- `Pie`
  - 2025
  - <https://arxiv.org/abs/2510.24051>
- `ThunderAgent`
  - 2026
  - <https://arxiv.org/abs/2602.13692>

这组工作的共同方向是：

- 把 serving 对象从 prompt 提升为 program / inferlet / workflow；
- 把 KV 管理交给更高层的程序结构；
- 结合 tool execution、workflow-level scheduling 或 resource management。

它们压到的点是：

- “应用层知道更多复用模式”
- “KV 管理应该变成显式接口”
- “agent / workflow 应该进入 serving runtime”

因此，我们如果把 thesis 写成：

- program-aware serving
- workflow-aware cache orchestration
- tool runtime 与 LLM runtime 一体化

就会和这组工作发生高重合。

### 2.5 Preble / DualMap 等 prefix-aware distributed scheduling

- `Preble`
  - <https://arxiv.org/abs/2407.00023>
- `DualMap`
  - <https://arxiv.org/abs/2602.06502>

这些工作主要解决：

- 分布式 cache affinity；
- shared prefix routing；
- load balancing 与 prefix reuse 的冲突。

它们说明：

- “prefix-aware scheduling” 本身已经是一条很拥挤的路；
- 如果我们只是做“让 shared prefix 请求更好地聚在一起”，新意不够。

### 2.6 LAPS 及其附近工作

- `LAPS`
  - 2026
  - <https://arxiv.org/abs/2601.11589>
- `Preble`
  - 2024
  - <https://arxiv.org/abs/2407.00023>
- `InferCept`
  - 2024
  - <https://arxiv.org/abs/2402.01869>

这组工作压到的是：

- prefix-aware scheduling；
- short re-prefill；
- waiting window；
- graph-shape 对齐；
- 对 tool / augmented inference 的 pause-resume 与 recompute。

这意味着 `ResidualModeKV` 单独做时，必须避开：

- “短 residual prefill batching”
- “prefix-aware graph clustering”
- “tool interception 下的 KV 恢复”

否则容易被认为只是既有思路的重组。

## 3. 我们的 idea 到底还有没有空间

### 3.1 有空间，但只剩“组合版系统”这个角度

单线判断如下：

1. `PromptABIKV`
   - 单独做，不稳。
   - 过于接近 provider 文档与 `Don't Break the Cache`。
2. `TypedPrefixKV`
   - 单独做，有价值，但很容易被审稿人看成 feature patch 集合。
3. `ResidualModeKV`
   - 单独做，比 `PromptABIKV` 更强，但会被 `LAPS` 等工作持续压缩空间。

因此，真正还能成立的版本，不是三选一，而是三者组合成一个闭环：

1. 上层是否真的生成了稳定前缀；
2. runtime 是否真的能统一判定这个前缀；
3. 命中以后系统是否真的兑现了性能收益。

### 3.2 目前没有找到完全同构公开论文

截至 `2026-04-15`，我没有找到一篇公开工作同时满足下面三点：

1. 以 **exact-prefix serving** 为核心，而不是 modular reuse / semantic cache / workflow execution；
2. 在 runtime 中统一覆盖：
   - token span
   - prompt embeds
   - multimodal hash / placeholder layout
   - cache scope / salt
3. 把 **post-hit residual execution selection** 当作 exact-prefix subsystem 的一部分来设计和评测。

这就是当前还能往下做的核心理由。

### 3.3 但不能写成“确保无人做过”

这个要求在学术上本来就不能严格满足。

我们最多只能严谨地写成：

> 截至 2026-04-15，我们未找到与该系统设计完全同构的公开论文或工业公开实现。

不能写成：

- first ever exact-prefix contract
- no prior work studies this
- guaranteed novel

这种表述在 rebuttal 和 camera-ready 阶段都很危险。

## 4. vLLM 中是否真能实现

结论是：**能实现，而且已经有多个天然抓手。**

### 4.1 Typed exactness 的底座已经在代码里

最关键的代码证据是：

- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:369)
  - `need_extra_keys()` 已经把 `mm_features`、`lora_request`、`cache_salt` 看作 block hash 的附加维度。
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:471)
  - `_gen_prompt_embeds_extra_hash_keys()` 已经对每个 block 的 `prompt_embeds` 做稳定 hash。
- [kv_cache_utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_utils.py:497)
  - `generate_block_hash_extra_keys()` 已经统一拼接 `LoRA + MM + cache_salt + prompt_embeds`。

这说明：

- `TypedPrefixKV` 所需的很多 carrier 已经不是概念，而是**真实进入 hash 的输入**。

对应的风险也很清楚：

- 论文**不能**再把 “给 prompt embeds / multimodal / salt 加 hash” 当成主要新意；
- 这些更适合作为系统背景和实现基础。

### 4.2 请求输入重构链条也已经存在

几个关键位置：

- [responses/utils.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/entrypoints/openai/responses/utils.py:95)
  - responses 路径会过滤旧 `system` message，并跳过 reasoning output。
- [preprocess.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/inputs/preprocess.py:161)
  - 文本、多模态、`cache_salt` 在 preprocess 阶段已分流。
- [engine.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/inputs/engine.py:107)
  - `MultiModalInput` 里已有 `mm_hashes` 与 placeholder 信息。
- [request.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/request.py:117)
  - `Request` 已缓存 per-block `prompt_embeds` hash。

这说明：

- `Cacheability ABI` 可以挂在 renderer / preprocess / request construction 链条上；
- 不是必须另起一套执行框架。

### 4.3 命中后的 mode selection 仍然有真实空白

几个关键位置：

- [kv_cache_manager.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/core/kv_cache_manager.py:176)
  - 当前核心是找 `longest cache hit`，并统一走后续分配逻辑。
- [gpu_model_runner.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/worker/gpu_model_runner.py:3593)
  - 当前运行时会 dispatch `CUDAGraphMode`，并且 `use_cascade_attn` / `has_encoder_output` 直接影响 `FULL` graph 是否可用。
- [flash_attn.py](/mnt/data3/Users/wangyang25/data/projects/KVresearch/vllm/vllm/v1/attention/backends/flash_attn.py:1043)
  - `use_cascade_attention()` 仍是启发式决策，且存在固定阈值。

这说明：

- 当前系统确实会因为 residual shape 的变化进入不同 graph / attention 路径；
- 但还没有一个显式的、prefix-hit-aware 的 mode selector。

这正是 `ResidualModeKV` 还能存在的理由。

## 5. 这条 thesis 是否有效

### 5.1 从机制上说，有效性是成立的

三层机制分别作用于不同瓶颈：

1. `Cacheability ABI`
   - 解决“为什么应用层没产出稳定前缀”；
   - 主要提升 hit-rate stability 与可诊断性。
2. `Typed Exact Prefix Contract`
   - 解决“runtime 到底如何一致地判断 exact prefix”；
   - 主要提升 workload coverage、correctness 与避免保守禁用。
3. `Post-Hit Execution Selection`
   - 解决“命中后是否真的能兑现性能收益”；
   - 主要改善 TTFT、graph replay、padding waste 与部分吞吐。

它们是顺序依赖关系，不是三篇互不相关的小工作。

### 5.2 但论文收益不能只靠“理论上应该更好”

要支撑 OSDI 级别，至少要证明：

1. 现实 workload 中存在大量 **可避免的 prefix drift**；
2. 现有 vLLM 的 typed exactness 仍然碎片化，或者虽然已有部分实现，但缺乏统一 contract / observability / policy；
3. 命中 prefix cache 后，baseline 仍存在显著的 graph / cascade / padding inefficiency；
4. 三层组合带来的收益，不只是 correctness，而是同时体现在：
   - hit rate
   - TTFT
   - p95/p99 latency
   - throughput
   - cache coverage
   - operator debugging cost

如果这些证据拿不出来，这条线会退化成：

- 很合理的工程演进；
- 但不是足够强的顶会 thesis。

## 6. 它能否领先业界最新论文

### 6.1 现在还不能说“已经领先”

截至 `2026-04-15`，只能说：

- 这条线**有机会**形成和最新工作不同的切口；
- 但还**没有证据**证明它领先于 `LAPS`、`ThunderAgent`、`DualMap` 等最新论文。

原因很简单：

- 目前还没有 prototype 数据；
- 还没有和最新 baselines 的正面对比；
- 也还没有证明三层组合的收益是叠加而不是重复。

### 6.2 想要“领先”，论文必须避开这些直接比较陷阱

如果目标是 OSDI/CCF-A 级别，必须主动避免以下比较误区：

1. 和 provider API 文档比
   - 这不叫论文比较，只能叫背景动机。
2. 和 `Prompt Cache` 比模块化复用能力
   - 我们不是 modular reuse 系统，这样比会吃亏。
3. 和 `ThunderAgent` / `Pie` 比 program-aware runtime 能力
   - 我们不是 workflow OS，这样比也会吃亏。
4. 和 `Preble` / `DualMap` 比分布式 routing
   - 我们不是分布式调度论文，这同样偏题。

更合理的比较方式是：

1. 在 **单机或 colocated vLLM** 范围内，比 exact-prefix serving 的系统闭环；
2. 证明现有系统缺少：
   - 应用到 runtime 的 cacheability contract；
   - 异构 carrier 的统一 exactness 语义；
   - 命中后的联合 mode selection；
3. 证明这三者组合后，在工具化、多模态、prompt-embed、长 system prompt 这些 workload 上更强。

## 7. 当前最值得担心的风险

### 7.1 风险一：`TypedPrefixKV` 的一部分已经进入 vLLM 代码

这是最现实的问题。

本地代码已经表明：

- multimodal hash 已经进入 prefix hash；
- `cache_salt` 已经进入 prefix hash；
- `prompt_embeds` 的 per-block hash 也已经存在。

这意味着论文不能再写成：

- “我们首次让这些输入支持 APC”

正确写法只能是：

- 现有实现是零散的；
- 缺少统一 contract、统一观测、统一 invalidation / publication 语义；
- 缺少与命中后执行路径的联动。

### 7.2 风险二：`PromptABIKV` 极易被当成经验帖

如果没有系统化证据链，它会被认为只是：

- prompt layout 建议；
- cache breaker checklist；
- tool ordering trick。

因此它必须退到“接口层和诊断层”，不能单独扛主创新。

### 7.3 风险三：`ResidualModeKV` 很可能被问到 LAPS

任何涉及：

- short residual
- graph bucket
- waiting window
- padding 对齐

的叙述，审稿人都会自然联想到 `LAPS`。

所以你必须强调：

- 不是按长度做聚类；
- 而是**在 exact hit 之后**重新评估 reuse / recompute / mode；
- 决策对象不是原始 prompt，而是 post-hit residual execution。

## 8. 是否值得继续做

结论是：**值得，但必须收缩目标，不能继续发散。**

### 8.1 当前最稳的论文定位

我认为当前最稳的写法是：

> `ExactPrefixKV` is a contract-guided exact-prefix serving substrate for
> heterogeneous vLLM inputs.

中文可以写成：

> `ExactPrefixKV` 将 vLLM 的 exact-prefix serving 从隐式缓存特性提升为一个
> 面向异构输入、具备显式契约、统一精确语义与命中后执行选择的系统底座。

### 8.2 当前不建议的定位

不建议继续写成：

- 新 prompt caching 方法；
- 新 prompt engineering 方法；
- 新 typed carrier feature；
- 新 prefix-aware scheduler；
- 新 agent runtime。

这些方向附近都已经太拥挤。

## 9. 下一步该怎么做

如果继续推进，建议按下面顺序：

1. **先做 profiling 与 tracing，而不是先写机制。**
   - 证明线上或合成真实 workload 中，prefix drift 和 post-hit inefficiency 确实存在。
2. **把论文 scope 锁死在单机 / colocated vLLM。**
   - 先别碰分布式 routing，不然会和 Preble / DualMap 纠缠。
3. **把 typed carrier 缩到最有代表性的三类。**
   - `prompt_embeds`
   - `multimodal hash + placeholder layout`
   - `cache_salt / cache scope`
4. **把 `PromptABIKV` 收缩为 observability + enforcement。**
   - 不把它写成 prompt engineering。
5. **把 `ResidualModeKV` 收缩为 post-hit selector。**
   - 不碰宏观 queueing 论文叙事。

## 10. 最终判断

截至 `2026-04-15`，我的最终判断是：

1. **这个组合版 idea 没有发现完全同构公开论文，因此仍可继续。**
2. **但它绝对不能再按三条分散小点子来讲。**
3. **它最有希望成立的版本，是“contract-guided exact-prefix serving in vLLM”。**
4. **当前最危险的不是“完全撞车”，而是被降格成 feature patch 或经验总结。**
5. **只要把边界收紧，并补上 trace + prototype + post-hit data，这条线仍然比之前那些方向更值得做。**

## 11. 参考资料

- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Preble: <https://arxiv.org/abs/2407.00023>
- InferCept: <https://arxiv.org/abs/2402.01869>
- Serve Programs, Not Prompts: <https://yecl.org/publications/gim2025hotos.pdf>
- Don’t Break the Cache: <https://arxiv.org/abs/2601.06007>
- LAPS: <https://arxiv.org/abs/2601.11589>
- ThunderAgent: <https://arxiv.org/abs/2602.13692>
- DualMap: <https://arxiv.org/abs/2602.06502>
- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- Google Gemini Context Caching: <https://ai.google.dev/gemini-api/docs/caching/>
- vLLM Prefix Caching Design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs Design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
