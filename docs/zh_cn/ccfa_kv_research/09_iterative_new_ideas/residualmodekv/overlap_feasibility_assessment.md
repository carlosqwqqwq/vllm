# ResidualModeKV 深度评估：重合风险、可行性与论文支撑力

## 1. 先给结论

这轮更严格的结论是：

1. **`ResidualModeKV` 不是“完全没人做过”的零重合方向。**
2. **它最危险的最近邻仍然是 `LAPS (2026)`，其次是 `k-LPM / DLPM / Preble / DualMap` 这一整条 prefix-aware scheduling 线。**
3. **如果把它写成 “short residual prefill 的 CUDA Graph clustering / prefix-aware batching”，那基本不安全。**
4. **如果把它收窄成 “exact APC-hit-aware post-cache execution-mode selection in colocated vLLM”，它仍然有防守空间。**
5. **它在 vLLM 中是可实现的，也很可能有效，但要撑起 OSDI 级主线，需要先用 profiling 证明一个非常具体的 runtime tradeoff：`FULL CUDA Graph` 和 `PIECEWISE / cascade attention` 在 exact hit 后可能冲突，当前 vLLM 没有显式优化这个冲突。**

更直接地说：

- 我**不能诚实地说“确保没人做过”**。
- 但截至 `2026-04-15` 这轮检索，我**没有发现与 refined ResidualModeKV 完全等价的公开系统论文**。
- 它当前的真实状态不是 “万无一失”，而是 **medium-risk but still alive**。

## 2. 本轮主要核对了哪些最近邻工作

### 2.1 LAPS：当前最危险的最近邻

`LAPS: A Length-Aware-Prefill LLM Serving System` 是 `ResidualModeKV` 当前最危险的重合源。

它已经做了三件和我们非常接近的事情：

1. 把多轮对话中的 `short re-prefill` 作为重点对象；
2. 对 short-prefill 使用 waiting window；
3. 用 `CUDA Graph-based clustering` 缓解 heterogeneous batch 带来的干扰。

这意味着，如果 `ResidualModeKV` 的论文主张只是：

- short residual 更适合某些 graph bucket；
- 等一小会儿来攒更整齐的 batch；
- 用 residual length 做 clustering；

那么它就会直接撞到 `LAPS`。

但它和 refined `ResidualModeKV` 仍有三个关键差异：

1. `LAPS` 的分类轴是 **length-aware short/long prefill**，而不是 **exact APC hit 之后的 residual**；
2. `LAPS` 主要面向 prefill-side serving，尤其是 prefill instance / PD-disaggregation 语境，而不是 colocated vLLM 单实例 runtime；
3. `LAPS` 没有把 **`FULL CUDA Graph`、`PIECEWISE CUDA Graph`、`cascade attention eligibility`、`exact hit metadata`** 作为一个联合 mode-selection 问题来写。

所以：

- `LAPS` 已经吃掉了 “short prefill + graph clustering”；
- 但还没有完全吃掉 “exact hit 后 residual mode selection”。

### 2.2 k-LPM / DLPM / Preble / DualMap：prefix-aware scheduling 线

这一条线共同说明了一件事：

> “prefix 命中信息可以参与调度决策” 本身早就不是新问题。

代表性工作包括：

- `LLM Query Scheduling with Prefix Reuse and Latency Constraints (k-LPM)`
- `DLPM / D^2LPM`
- `Preble`
- `DualMap`

它们分别聚焦：

- prefix reuse 与 TTFT 约束；
- fairness 与 locality；
- distributed/global scheduler；
- cache affinity 与 load balancing。

这类工作和 `ResidualModeKV` 的差异在于：

1. 它们主要优化 **请求排布与路由**；
2. 它们通常把 “前缀命中越多越好” 当作目标；
3. 它们没有把 **命中之后是否还该重新补算一点点 token 以换取更好的 runtime mode** 作为中心问题。

因此，`ResidualModeKV` 不能写成：

- prefix-aware scheduling；
- cache-aware routing；
- prefix locality scheduling；

否则一定会撞车。

### 2.3 BatchLLM / shared-prefix kernel 线

`BatchLLM`、`ChunkAttention`、`Hydragen`、`PAT` 这一线说明：

> “共享前缀应该改变 attention 的执行方式” 也不是新问题。

它们聚焦的是：

- shared-prefix attention kernel；
- tree-structured shared prefix runtime；
- offline grouping；
- decode-stage shared-prefix optimization。

其中最值得注意的是：

- vLLM 社区 issue #12080 已经明确讨论过 `BatchLLM` 风格的 shared-prefix grouping；
- vLLM 社区 issue #12395 还直接暴露了 `cascade attention` 的收益并不稳定。

所以 `ResidualModeKV` 也不能写成：

- shared-prefix grouping；
- better cascade attention batching；
- same-prefix requests grouping for kernel efficiency；

否则又会回到已有工作范围。

### 2.4 CacheBlend：selective recompute 不是全新概念

`CacheBlend` 非常重要，因为它说明：

> “为了更高系统效率而 selective recompute 一小部分 token” 不是一个全新的概念。

但 `CacheBlend` 的语境和我们不同：

1. 它处理的是 **RAG/non-prefix chunk fusion**；
2. 它的 selective recompute 是为了修正 reused chunk 的跨 chunk attention；
3. 它主要与 **slow-device retrieval / cache fusion** 绑定。

而 `ResidualModeKV` 的 selective recompute 是为了：

- 进入更优的 batch shape；
- 避开不利的 graph mode；
- 或改变 cascade compatibility。

所以：

- `recompute` 本身不是新；
- **post-hit runtime mode selection** 仍可能是新。

## 3. vLLM 代码路径里，这个问题到底是否真实存在

结论是：**存在，而且比我上一轮判断更具体。**

### 3.1 exact hit 已经改变 residual work

调度器里，`num_new_tokens = request.num_tokens - num_computed_tokens`。  
也就是说，APC hit 后真正参与本轮 prefill 的 residual token 数会直接变化。

这不是猜测，是 scheduler 当前路径的基本逻辑。

### 3.2 runtime mode 依赖 `BatchDescriptor`

vLLM 当前 CUDA Graph 设计文档明确说明：

- runtime dispatch 由 `runtime mode + BatchDescriptor` 驱动；
- `BatchDescriptor` 包含 `num_tokens / num_reqs / uniform / has_lora`；
- dual-mode 配置如 `FULL_AND_PIECEWISE` 会根据 batch composition 动态 dispatch。

这意味着：

- same request set but different residual lengths
- same APC hit rate but different padding
- same prefix reuse but different uniformity

都可能导致不同的 CUDA Graph runtime mode。

### 3.3 cascade attention 会强制改变 graph mode

这一点是本轮最关键的代码事实之一：

1. `gpu_model_runner` 在 `_determine_batch_execution_and_padding()` 之前先计算 `cascade_attn_prefix_lens`；
2. 调用 dispatch 时，代码里直接有 `disable_full=use_cascade_attn`；
3. vLLM 官方 CUDA Graph 文档也明确写到：
   - 如果 batch 使用 cascade attention，就会被派发到 `PIECEWISE`，否则退到 `NONE`。

这意味着一个非常具体的 tradeoff：

> exact prefix hit 可能让某个 batch 变得更适合 cascade attention；但只要启用 cascade，又可能失去 `FULL CUDA Graph` 的收益。

而当前系统并没有一个显式优化器来决定：

- 该优先吃 cascade，
- 还是优先保住 full graph，
- 还是补算一点点 token 换更稳的 batch shape。

这正是 `ResidualModeKV` 目前最有价值的切口。

### 3.4 cascade attention 本身仍然是启发式

`flash_attn.py` 当前的 `use_cascade_attention()` 仍然写着：

- common prefix `< 256` 直接禁用；
- `num_reqs < 8` 直接禁用；
- 注释明确说阈值仍需 tune。

这说明 cascade 当前不是一个“最优器”，而是一个启发式开关。

这也给 `ResidualModeKV` 留下了空间：

- 我们不是去发明一个 attention kernel；
- 而是去决定 **什么时候值得让 batch 进入 cascade 路径，什么时候不值得**。

## 4. `ResidualModeKV` 到底哪里重复，哪里还没重复

### 4.1 会重复的版本

下面这些版本都不安全：

- prefix-aware scheduling for vLLM
- cache-aware short-prefill batching
- graph clustering for short residual prefill
- APC-aware query grouping
- better cascade attention scheduling

这些表达会撞到：

- `LAPS`
- `k-LPM`
- `DLPM`
- `Preble`
- `DualMap`
- `BatchLLM`
- vLLM 社区里已有的 prefix-aware scheduling / cascade 讨论

### 4.2 还可能成立的版本

下面这个收窄版本仍然值得保留：

> ResidualModeKV studies whether exact prefix-cache hits should trigger a different post-hit execution mode in colocated vLLM, jointly considering bounded recomputation, CUDA-graph dispatch, padding, and cascade-attention eligibility.

它的防守点有四个：

1. **post-hit**
   - 不是看原始 prompt，而是看命中之后剩下的 residual。
2. **exact hit aware**
   - 不是只看 length bucket，而是看真正的 cached prefix outcome。
3. **execution mode selection**
   - 不是简单 maximize hit，而是决定 hit 之后怎么跑更快。
4. **FULL vs PIECEWISE vs cascade tradeoff**
   - 这是当前公开工作里最少被明确表述的一层。

## 5. 这个方向是否足够支撑一篇系统论文

### 5.1 能撑，但前提比之前更苛刻

要让 `ResidualModeKV` 成为 OSDI 级别主线，至少要证明以下三件事：

1. **问题显著存在**
   - baseline vLLM 在 APC high-hit workload 下，确实存在：
     - graph replay 不稳定；
     - padding waste 偏高；
     - cascade enable 不稳定；
     - 或 full graph 被过早放弃。

2. **这个问题不是 LAPS 等已有工作已经覆盖的**
   - 也就是要证明：
     - 不是简单的 short-prefill 问题；
     - 不是分布式 routing 问题；
     - 而是 colocated runtime 内 exact-hit 诱发的 mode-choice 问题。

3. **联合 mode selection 的收益足够大**
   - 最少应同时改善其中几项：
     - `TTFT p50/p99`
     - `throughput`
     - `graph replay rate`
     - `padding ratio`
     - `cascade beneficiality`

### 5.2 会撑不起来的情况

如果最终 profiling 发现：

- 开启 APC 后，baseline 的 full/piecewise dispatch 已经很合理；
- cascade 使用率虽然低，但启用也没收益；
- bounded recompute 只能带来 3%-5% 级别收益；
- 或收益只在特别窄的 trace 上成立；

那它不够当 OSDI 主线，更适合作为另一个方向的执行层优化。

## 6. 它是否有效

### 6.1 我认为它“很可能有效”的场景

最有希望的 workload 是：

- 长 system prompt / tool schema / shared template；
- APC 命中高，但每个请求 residual 长度很碎；
- batch composition 波动大；
- 低延迟交互场景，TTFT/p99 很重要；
- vLLM 运行在 `FULL_AND_PIECEWISE` 这类 dual-mode 配置下。

在这些场景里，命中 APC 之后的 “剩余 40/80/120 token” 很可能决定：

- 能不能进入某个 graph bucket；
- 能不能凑出 uniform / near-uniform；
- 会不会触发 cascade；
- 会不会因此丢掉 full graph。

### 6.2 我认为它“可能无效”的场景

- decode 完全主导；
- prompt 本身很短；
- backend 不适合 cascade；
- batch 本来就很整齐；
- 或者部署环境主要是 PD-disaggregated prefill worker，那里 `LAPS` 一类方法更合适。

## 7. 它是否能在 vLLM 中实现

结论是：**能，而且最小原型不需要改 kernel。**

### 7.1 最小实现路径

第一阶段，只做观测：

- `apc_hit_tokens`
- `residual_tokens`
- `cudagraph_mode`
- `batch_desc.num_tokens`
- `num_paddings`
- `cascade_eligible / enabled / disabled_reason`

第二阶段，做一个小决策器：

- `reuse_as_is`
- `bounded_recompute`
- `graph_aligned_chunking`
- `tiny_delay_grouping`

第三阶段，再看是否需要更复杂的 cost model。

### 7.2 为什么说“最小原型可落地”

因为当前 vLLM 已经有所有必要接口：

- scheduler 已经知道 `num_computed_tokens / num_new_tokens`
- model runner 已经决定 `cudagraph_mode / batch_desc`
- attention backend 已经暴露 cascade heuristic

也就是说，`ResidualModeKV` 更像“调度与执行协同层”的 patch，而不是从零发明新 backend。

## 8. 它是否领先业界最新论文

这个问题必须实话实说：

### 8.1 不能直接说“已经领先”

截至 `2026-04-15`，我们最多只能说：

- 我没有找到完全等价的公开论文；
- 但最危险的最近邻 `LAPS` 已经非常接近某些表述；
- 再加上 `k-LPM / DLPM / Preble / BatchLLM / CacheBlend` 的不同局部覆盖，
  所以它绝不是一个可以高枕无忧的 zero-risk novelty claim。

### 8.2 如果要“领先”，必须抓住这个最具体的点

当前最可能真正领先的点，不是：

- reuse-aware scheduling
- residual-aware batching
- short-prefill graph clustering

而是：

> exact prefix hits can flip the optimal execution mode because cascade attention and full CUDA graphs are not simultaneously optimal in current vLLM runtimes.

如果这个 tradeoff 被 prototype 明确测出来，而且改进收益明显，那它才可能真正领先当前公开工作。

## 9. 当前最终判断

本轮对 `ResidualModeKV` 的最终判断是：

1. **不是零重合方向。**
2. **最危险最近邻是 `LAPS`。**
3. **它依然没有被现有论文一比一做掉。**
4. **vLLM 中确实存在一个很具体、很系统的问题：exact hit 后的 residual shape 会影响 full/piecewise/cascade 的 mode choice，而当前 runtime 并没有联合优化它。**
5. **它值得继续，但必须立刻进入 profiling；如果 profiling 不支持，就应快速降级。**

换成最短一句话：

> ResidualModeKV 现在仍可做，但必须把创新点钉死在 “exact-hit-induced execution mode tradeoff” 上；否则很容易重新滑回已经被做过的 territory。

## 10. 下一步最小任务

1. 在 vLLM 里加 metrics，记录 APC high-hit workload 下的：
   - residual token distribution
   - graph replay / fallback
   - padding ratio
   - cascade eligible / enabled / disabled
2. 证明 full graph 与 cascade 在真实 trace 中存在 tradeoff。
3. 做一个最小决策器，只在：
   - reuse-as-is
   - bounded recompute
   - graph-aligned chunking
   三者之间切换。
4. 以 `LAPS` 和 baseline vLLM 作为首要对照组。

## 11. 资料索引

- LAPS: <https://arxiv.org/abs/2601.11589>
- LLM Query Scheduling with Prefix Reuse and Latency Constraints: <https://arxiv.org/abs/2502.04677>
- Locality-aware Fair Scheduling in LLM Serving: <https://arxiv.org/abs/2501.14312>
- Preble: <https://arxiv.org/abs/2407.00023>
- DualMap: <https://arxiv.org/abs/2602.06502>
- CacheBlend: <https://arxiv.org/abs/2405.16444>
- ChunkAttention: <https://arxiv.org/abs/2402.15220>
- PAT: <https://arxiv.org/abs/2511.22333>
- vLLM Prefix Caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM issue #7883: <https://github.com/vllm-project/vllm/issues/7883>
- vLLM issue #12080: <https://github.com/vllm-project/vllm/issues/12080>
- vLLM issue #12395: <https://github.com/vllm-project/vllm/issues/12395>
