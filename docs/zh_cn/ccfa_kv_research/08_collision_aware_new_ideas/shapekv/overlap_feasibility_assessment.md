# ShapeKV 深度评估：重合风险、可行性与论文支撑力

## 1. 先给结论

经过这一轮更窄更深的调研，`ShapeKV` 的结论需要比上一版更严格：

1. **原始版 ShapeKV 不是“零重合”。**
2. **最危险的最近邻不是 SGLang/RadixAttention，而是 `LAPS (MLSys 2025)`。**
3. **如果 ShapeKV 仅仅表述为 “short re-prefill 的 CUDA Graph clustering / graph-aware batching”，那么重合风险偏高。**
4. **如果把 ShapeKV 收窄为 “exact prefix-hit-aware execution-shape co-design for colocated vLLM, jointly optimizing residual prefill chunking, CUDA Graph dispatch, and cascade-attention eligibility”，那么仍然有可防守空间。**
5. **因此，ShapeKV 仍然可行，但必须改写 thesis，不能再用泛泛的短 prefill / graph bucket 叙事。**

更直接一点说：

- `ShapeKV` **不是已经被证明不可做**。
- 但它也**绝对不能再被表述成“应该没人做过的 prefix-aware scheduling”**。
- 当前最稳的定位，是把它定义成：**vLLM 中 exact APC 命中结果如何改变 `BatchDescriptor`、`CUDA Graph runtime mode`、`padding waste`、`cascade attention` 启用条件，并据此做联合调度。**

## 2. 这轮调研核查了哪些最近邻工作

### 2.1 Prefix-aware scheduling / locality-aware scheduling

这一条线说明：**“用 prefix 命中信息指导调度”本身早就不是新问题。**

- `LLM Query Scheduling with Prefix Reuse and Latency Constraints`（2025）把 prefix reuse 下的查询调度形式化，提出 `k-LPM` 来平衡 prefix reuse 与 TTFT 约束。它的核心是等待队列顺序与延迟约束，不涉及 CUDA Graph 或 cascade attention。
- `DLPM / D²LPM`（2025）强调 locality-aware fair scheduling，在公平性和 prefix locality 之间找平衡，也不是 graph/cascade 协同。
- `Preble`（ICLR 2024）在 distributed prompt scheduling 中按 cached token percentage 组织优先级组，并支持 vLLM / SGLang 后端。它更像 prefix-aware global/local scheduling，而不是单机执行形状联合优化。
- `SGLang v0.4` 官方博客已经把 `cache-aware load balancer` 和 `zero-overhead batch scheduler` 做成显式卖点，说明社区正在快速吃掉“prefix-aware routing + low-overhead scheduling”这条线。
- `vLLM` 自己的 issue #7883 也明确提出应当让 scheduler 更早考虑 prefix caching，否则 block allocation 和 token budget 都会被高估。

结论：

- **不能把 ShapeKV 写成 prefix-aware scheduling。**
- 这条线已经有论文、有社区实现、有 issue/RFC。

### 2.2 Prefix-aware kernel / shared-prefix attention kernel

这一条线说明：**“前缀共享能加速 attention 计算”也不是新问题。**

- `ChunkAttention`（ACL 2024）用 prefix tree 和 prefix-aware self-attention kernel 提升共享系统提示词下的 attention 效率。
- `Hydragen`（2024）把 shared prefix attention 分解成 shared-prefix 和 unique suffix 两部分，显著降低 decode 时重复读取共享 prefix KV。
- `BatchLLM`（2024/2025）不仅做 global prefix sharing 和 throughput-oriented token batching，还做 prefix-shared attention kernel 优化。
- `PAT`（2025）和 `CoDec/FlashForge`（2025）都在 decode attention 里显式利用 prefix sharing。

结论：

- **不能把 ShapeKV 写成 prefix-aware kernel。**
- 这条线更适合作为对比或互补基线。

### 2.3 最危险最近邻：LAPS（MLSys 2025）

`LAPS: A Length-Aware-Prefill LLM Serving System` 是这轮 ShapeKV 调研中最重要的新发现。

它做了三件和 ShapeKV 高度相似的事：

1. 观察到多轮对话中的 `short re-prefill` 是 memory-bound。
2. 对 short-prefill 采用 `batch waiting window + CUDA Graph-based clustering`。
3. 用 graph-aware batching 对齐到最近的 captured CUDA Graph shape，减少 padding 和 launch overhead。

更具体地说，LAPS 在论文中明确写到：

- short re-prefill 更短、更稳定，适合 CUDA Graph。
- 用 power-of-two 的 `(prompt length, batch size)` bucket 预捕获 graphs。
- 调度时把 batch 对齐到最近的 graph shape。
- 还会用等待窗口去聚合更多短请求。

这和“短 residual prefill 聚类到 graph bucket”已经非常近。

但它仍然和 refined ShapeKV 有三个关键区别：

1. **LAPS 的核心分类轴是长/短 prefill 或 re-prefill，而不是 exact prefix-cache hit。**
2. **LAPS 的系统场景是 PD/prefill instance disaggregation，而不是 colocated vLLM runtime 的 per-step residual scheduling。**
3. **LAPS 不讨论 vLLM 的 `BatchDescriptor`、`FULL/PIECEWISE` CUDA Graph runtime mode 切换、也不讨论 `cascade attention` 与 prefix hit 的协同。**

因此，LAPS 不会直接否定 ShapeKV，但它迫使 ShapeKV 必须重新收窄边界。

## 3. vLLM 本地源码是否支持 ShapeKV 的问题设定

结论是：**支持，而且问题是真实存在的。**

### 3.1 prefix cache hit 已经会改变 residual work 形状

`scheduler.py` 当前用 `num_computed_tokens` 和 `num_new_tokens` 调度请求。prefix cache hit 会直接减少 `num_computed_tokens` 之后需要实际补算的 token 数，因此 residual prefill 的长度确实会变化。

### 3.2 CUDA Graph dispatch 依赖 batch shape

`cudagraph_dispatcher.py` 与官方 CUDA Graph 文档说明，vLLM 用 `BatchDescriptor` 作为运行时 dispatch key，核心字段至少有：

- `num_tokens`
- `num_reqs`
- `uniform`
- `has_lora`

同时，vLLM CUDA Graph 文档明确指出：

- dispatcher 会根据 batch composition 在 `FULL` / `PIECEWISE` / `NONE` 间切换；
- `BatchDescriptor` 的目标就是唯一标识一个（可能 padded）的运行时 batch；
- 如果 batch uses cascade attention，则会被派发到 `PIECEWISE` 或 `NONE`。

这意味着：**prefix hit 改变 residual token 数之后，确实会改变 `BatchDescriptor`、graph mode 和 padding 行为。**

### 3.3 cascade attention 的启用仍然是启发式

`flash_attn.py` 里 `use_cascade_attention()` 仍然有静态阈值：

- common prefix `< 256` 直接关；
- 请求数 `< 8` 直接关；
- sliding window / local attention / DCP 场景直接关；
- 注释明确写着这些阈值还需要 tune。

而 `gpu_model_runner.py` 的执行顺序是：

1. 先根据 `num_common_prefix_blocks` 计算 `cascade_attn_prefix_lens`；
2. 再决定 `_determine_batch_execution_and_padding()`；
3. 再确定 cudagraph mode 与 batch descriptor。

这说明：**prefix-shared metadata、cascade attention 和 CUDA Graph dispatch 已经在同一条执行路径上，但 scheduler 并没有显式把它们联合优化。**

## 4. ShapeKV 现在到底哪里重复，哪里还不重复

### 4.1 会重复的版本

以下表述不再安全：

- “prefix-aware scheduling for vLLM”
- “cache-aware short-prefill batching”
- “CUDA Graph clustering for short re-prefills”
- “根据 prefix hit 把请求按长度聚类到 graph bucket”

因为这些表述会直接撞到：

- `LAPS`
- `k-LPM`
- `DLPM`
- `Preble`
- `SGLang v0.4`
- `vLLM issue #7883`

### 4.2 仍有防守空间的版本

下面这个版本仍然值得保留：

> ShapeKV studies how exact prefix-cache hits reshape residual prefill execution in a colocated vLLM runtime, and co-optimizes chunking, CUDA Graph dispatch, and cascade-attention eligibility using scheduler-visible hit metadata.

它的独特点有四个：

1. **exact prefix-hit-aware**
   
   不是按 prompt length 近似分组，也不是仅按 locality 路由，而是显式使用 APC hit 之后的 residual 形状。

2. **colocated vLLM runtime**
   
   不依赖 PD disaggregation，也不是外部 router，而是直接嵌入 vLLM 的单实例 continuous batching。

3. **graph + cascade joint optimization**
   
   LAPS 重点是 short-prefill 的 graph clustering；ShapeKV 要把 `BatchDescriptor`、`runtime mode`、`padding` 和 `cascade attention eligibility` 一起纳入代价函数。

4. **residual work instead of raw prompt length**
   
   这点很重要。LAPS 是 length-aware；ShapeKV 应该是 **post-cache residual-aware**。同样都是 2K prompt，如果 APC 命中 1900 个 token，问题就不是原 prompt 多长，而是 residual 100 token 应该如何对齐 graph/cascade。

## 5. ShapeKV 是否足够支撑一篇系统论文

### 5.1 可以支撑，但前提很强

要让 ShapeKV 够支撑 CCF-A/OSDI 级别论文，至少要成立下面三个命题：

1. **命题 A：APC 高命中时，vLLM baseline 仍存在显著的 graph/cascade inefficiency。**
   
   你需要测出：
   - graph replay rate 不高；
   - padding token ratio 偏高；
   - cascade attention enable rate 偏低；
   - 或者因为 batch composition 导致频繁回退到 eager / piecewise。

2. **命题 B：这些 inefficiency 不是被 LAPS/Preble/SGLang 已经解释掉的问题。**
   
   也就是要证明：
   - 不是简单的长短 prompt 混合问题；
   - 不是外部 router 的 prefix locality 问题；
   - 而是 colocated vLLM runtime 内，exact cache hit 之后 residual shape 没被利用。

3. **命题 C：联合 graph/cascade 调度能同时带来 throughput 与 tail latency 改善。**
   
   仅提升平均吞吐不够，`TTFT p99`、`throughput`、`graph replay rate`、`padding waste`、`cascade usage` 最好都能改善。

### 5.2 不够支撑的情况

如果实验发现：

- baseline 在高 APC hit 下 graph replay 已经很好；
- padding waste 很低；
- cascade attention miss 很少；
- 调度改动只能带来 3% 左右收益；
- 收益只出现在某一种特别窄的 trace；

那么 ShapeKV 不足以单独支撑 OSDI 主线，最多是一篇较小的性能优化工作。

## 6. ShapeKV 是否会有效

我现在的判断是：**很可能有效，但收益窗口高度 workload-dependent。**

### 6.1 最可能有效的场景

- 长系统提示词 / 工具说明 / 模板高度共享，但每个请求 suffix 长度不同。
- RAG/Agent/chat 多轮场景中，APC 命中后 residual prefill 很短且分布碎片化。
- vLLM 开启 `FULL_AND_PIECEWISE` 或其他 dual-mode CUDA Graph 模式。
- `cascade attention` 本该可用，但因为请求到达顺序和 residual 形状不稳定而没吃到。
- TTFT/p99 比平均吞吐更关键。

### 6.2 效果可能很差的场景

- decode 完全主导，总体瓶颈不在 residual prefill。
- prompt 本身就很短，APC 命中空间有限。
- attention backend 或模型配置让 cascade 基本不可用。
- 部署采用 PD disaggregation，prefill 已被独立实例隔离，LAPS 一类方法可能更合适。
- workload 形状天然均匀，graph replay 已经接近饱和。

## 7. 是否能在 vLLM 中实现

### 7.1 实现上是可落地的

最小原型不需要改 kernel，可以先做：

1. **观测层**
   - 记录 `APC hit ratio`
   - residual length histogram
   - chosen graph mode
   - graph replay / eager fallback reason
   - padding token ratio
   - cascade eligible / enabled / disabled reason

2. **策略层**
   - 在 scheduler 中根据 residual length 和 graph bucket 调整 `num_new_tokens`
   - 对高共享前缀请求做轻量 grouping
   - 只在严格 delay budget 内等一小段时间凑 graph-stable batch

3. **联合决策层**
   - 让 `num_common_prefix_blocks` 和 residual length 一起参与调度
   - 对“为了进 bucket 多算/少算几 token 是否划算”做 bounded cost model

### 7.2 真正难点

难点不在“能不能改代码”，而在：

- 如何做一个简单但不蠢的 cost model；
- 如何避免调度器过度复杂；
- 如何证明不是把 tail latency 通过 waiting window 偷偷换掉；
- 如何公平对比 vLLM baseline、手工 tuned graph size、以及 LAPS/Preble/SGLang 这类外部系统。

## 8. 是否能领先当前论文

### 8.1 不能说“绝对领先”

这个说法我不能帮你背书。尤其在发现 `LAPS` 之后，任何 “确保没人做过” 都不严谨。

### 8.2 可以防守的说法

截至 `2026-04-15` 这一轮检索，我没有发现**与 refined ShapeKV 完全等价**的公开工作。现有最近邻分别覆盖了：

- prefix-aware waiting-queue/order：`k-LPM`、`DLPM`、`Preble`
- cache-aware routing/load balancing：`SGLang v0.4`
- short-prefill CUDA Graph clustering：`LAPS`
- prefix-aware decode kernels：`ChunkAttention`、`Hydragen`、`PAT`、`CoDec`
- vLLM runtime hooks：`issue #7883`、`CUDA Graph docs`

但我没有找到同时满足下面四点的工作：

1. exact APC hit aware；
2. colocated vLLM runtime；
3. joint optimization of residual chunking + graph mode + cascade attention；
4. using scheduler-visible post-cache residual shape as the first-class signal。

因此，**更稳的结论是：refined ShapeKV 仍然具有 novelty space，但 novelty 已经明显比上一轮估计的小。**

## 9. 我现在对 ShapeKV 的最终判断

### 9.1 作为原始 idea

不够安全。因为会撞到 `LAPS`。

### 9.2 作为 refined idea

仍然值得继续，但 thesis 必须改成下面这种：

> ShapeKV is not a general prefix-aware scheduler. It is a cache-hit-aware execution-shape controller for colocated vLLM runtimes, which uses exact APC hit metadata to stabilize residual prefill execution across CUDA Graph dispatch and cascade-attention eligibility.

### 9.3 论文建议

如果继续做，我建议把标题和贡献往下面靠：

- 标题关键词：`cache-hit-aware`、`execution shape`、`residual prefill`、`CUDA Graph dispatch`、`cascade attention`
- 不要突出：`prefix-aware scheduling`、`short prefill clustering`、`length-aware batching`

### 9.4 风险评级

| 维度 | 评级 | 说明 |
| --- | --- | --- |
| 研究问题真实性 | 高 | vLLM 本地路径和官方文档都支持 |
| vLLM 可实现性 | 高 | 先做 scheduler/model runner 策略，不必先改 kernel |
| 优化收益可能性 | 中高 | 取决于 APC-heavy workload 与 graph/cascade 当前利用率 |
| 与已有论文重合风险 | 中高 | `LAPS` 提高了风险，但 refined thesis 仍可防守 |
| OSDI 主线潜力 | 中 | 要靠很强的 profiling 证据和跨系统对比 |

## 10. 下一步最小实验

如果你要继续 ShapeKV，下一步不要再空想了，直接做下面三组 profiling：

1. **APC-heavy colocated vLLM**
   - graph replay rate
   - graph fallback reason
   - padding token ratio
   - cascade eligible / enabled rate
   - TTFT p95/p99

2. **去掉 APC / 去掉 cascade / 去掉 CUDA Graph**
   - 形成交叉 ablation，证明三者交互不是错觉

3. **与 LAPS 式 length bucketing 的对照**
   - 用 prompt length bucket baseline
   - 对比 exact residual-hit-aware 策略
   - 证明 post-cache residual 比 raw length 更有信息量

如果第 3 组对比做不出来明显差距，ShapeKV 就很难撑住。

## 11. 关键资料

- vLLM CUDA Graphs: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM issue #7883: <https://github.com/vllm-project/vllm/issues/7883>
- LAPS: <https://arxiv.org/abs/2601.11589>
- Query Scheduling with Prefix Reuse: <https://arxiv.org/abs/2502.04677>
- DLPM: <https://arxiv.org/abs/2501.14312>
- Preble: <https://arxiv.org/abs/2407.00023>
- SGLang v0.4 blog: <https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/>
- BatchLLM: <https://arxiv.org/abs/2412.03594>
- ChunkAttention: <https://aclanthology.org/2024.acl-long.623/>
- Hydragen: <https://arxiv.org/abs/2402.05099>
- PAT: <https://arxiv.org/abs/2511.22333>
