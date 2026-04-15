# ShapeKV：前缀命中感知的 CUDA Graph / Cascade Attention 形状稳定调度

## 1. 一句话 thesis

`ShapeKV` 的核心 thesis 是：**prefix cache hit 不只是减少 prefill token，它还会改变 batch shape、CUDA Graph dispatch key、padding waste、cascade attention 触发条件和 kernel path；vLLM 需要一个 cache-hit-aware execution-shape scheduler，在保持 exact KV semantics 的同时最大化 graph replay、减少 padding 和降低 TTFT。**

这是当前三个方向里我最推荐的主线。

## 2. 第一性原理

vLLM 里的 APC 命中会把一个请求从 “需要 prefill N 个 token” 改成 “只需要补算 R 个 token”。这看似总是好事，但在现代 LLM serving runtime 中，真实成本不是只由 token 数决定，还由执行形状决定：

- CUDA Graph 需要稳定的 batch descriptor 和捕获尺寸。
- padding 会让少量 residual token 被扩展到更大的 capture size。
- cascade attention 只有在 shared prefix 足够长、请求数足够多、attention 变体支持时才值得启用。
- chunked prefill 和 token budget 会把 residual work 切成不同形状。
- speculative decoding 会改变 decode query length，进一步影响 graph bucket。

因此，一个 cache hit 可能出现两种相反结果：

1. 节省了 prefill 计算。
2. 打碎了 batch shape，导致 graph fallback、padding waste 或 cascade miss。

`ShapeKV` 要优化的是二者的总和，而不是单独最大化 hit ratio。

## 3. vLLM 当前证据

本地源码显示这个切口是真实存在的：

- `vllm/v1/cudagraph_dispatcher.py` 中 `CudagraphDispatcher` 把 valid cudagraph dispatch keys 作为运行时 graph replay 的事实来源，并根据输入生成 `BatchDescriptor`。
- `vllm/v1/core/sched/scheduler.py` 的调度逻辑以 `num_computed_tokens`、`num_tokens_with_spec` 和 token budget 为中心，明确覆盖 chunked prefill、prefix caching、speculative decoding。
- `vllm/v1/worker/gpu_model_runner.py` 在执行前先计算 `cascade_attn_prefix_lens`，再调用 `_determine_batch_execution_and_padding()` 决定 cudagraph mode、batch descriptor 和 padding。
- `vllm/v1/attention/backends/flash_attn.py` 的 cascade attention 启发式包含固定阈值，例如 common prefix 小于 256 token 或请求数小于 8 时直接关闭，注释中也写着这些阈值仍需调优。

这些路径说明：vLLM 已经具备 CUDA Graph、cascade attention、prefix caching、chunked prefill 等组件，但它们的协同仍主要是启发式和分层决策。

## 4. 最近邻工作与重合分析

重要补充：在后续更深一轮调研中，我确认 `LAPS (MLSys 2025)` 是 `ShapeKV` 最危险的最近邻工作。详见 [overlap_feasibility_assessment.md](overlap_feasibility_assessment.md)。

| 工作线 | 代表工作 | 做了什么 | 与 ShapeKV 的关系 | 重合风险 |
| --- | --- | --- | --- | --- |
| Prefix-aware scheduling | vLLM GitHub issue #7883、SGLang cache-aware scheduling | 根据 prefix cache 状态调整调度和路由 | 最近邻。必须说明我们优化的是 graph/cascade execution shape，不是普通命中率或等待队列顺序 | 中 |
| Short re-prefill + CUDA Graph batching | LAPS (MLSys 2025) | 用 waiting window 和 CUDA Graph clustering 优化 short prefill/re-prefill | 最危险最近邻；如果 ShapeKV 只讲短 residual prefill 聚类到 graph bucket，会明显重合 | 中高 |
| Radix prefix cache | SGLang / RadixAttention | 用 radix tree 管理 KV cache，并结合 cache-aware scheduling | 关注 prefix index 与 structured LM programs；不是 vLLM CUDA Graph / Cascade Attention shape co-design | 中 |
| Prefix-aware kernels | PAT、ChunkAttention | 改 attention kernel 或 decode 阶段访存路径 | 互补；ShapeKV 在 scheduler/runtime 层选择更好的执行形状 | 低到中 |
| CUDA Graph runtime | vLLM CUDA Graph docs、Dynamo/CUDAGraph 相关工程 | 捕获并 replay 稳定形状以降低开销 | 现有文档没有把 APC hit 作为 graph shape 输入 | 低 |
| KV offload / disaggregation | LMCache、Strata、PCR、CALVO | 优化远端 KV 加载、预取、替换 | 主要是数据移动；ShapeKV 关注本地执行形状 | 低 |
| Fair scheduling / QoS | DLPM 等 LLM serving scheduler | 优化 latency fairness 或 prefill/decode 干扰 | 目标不同；可以作为 baseline，但不是同一贡献 | 低 |

## 5. 是否重复

没有发现与 refined `ShapeKV` 完全等价的公开论文，但最接近的已经不止两类。危险最近邻包括：

1. **vLLM 社区的 prefix-caching-aware scheduling issue**：它指出调度器应该考虑 prefix cache 命中，但主要讨论 token budget、can allocate 和避免把很长 prompt 挤出调度队列。
2. **SGLang 的 cache-aware scheduling / RadixAttention**：它强在 prefix tree、共享前缀复用和 structured program 执行，但不是围绕 vLLM 的 CUDA Graph replay、BatchDescriptor、padding 和 cascade attention 共同优化。
3. **LAPS**：它已经把短 re-prefill、waiting window、CUDA Graph-based clustering 做得很深；如果 ShapeKV 只停留在 “short residual prefill graph batching”，会明显重合。

所以 `ShapeKV` 的防守边界应写成：

> Prior work uses prefix-cache information to decide what to reuse or which request to schedule. ShapeKV studies how exact prefix-cache hits reshape the GPU execution batch, and co-optimizes residual prefill scheduling with CUDA Graph replay and cascade-attention eligibility.

不能写 “第一个 prefix-aware scheduler”，也不能写 “第一个 short-prefill CUDA Graph batching”。更稳妥的说法是：**第一个系统研究 exact prefix hit 如何在 colocated vLLM runtime 中改变 graph-stable execution shape 与 cascade-attention eligibility 的工作。**

## 6. 方法设计

### 6.1 Shape signature

为每个 request 或 request group 生成 `shape signature`：

- `hit_len`：prefix cache 命中的 token 数。
- `residual_len`：还需实际 prefill 的 token 数。
- `block_aligned_residual`：是否落在 block 边界。
- `graph_bucket`：预计落入哪个 CUDA Graph capture size。
- `padding_cost`：为 replay graph 需要填充的 token 数。
- `cascade_eligibility`：当前 common prefix、query 数、attention variant 是否满足 cascade attention。
- `spec_decode_width`：speculative decoding 下 decode query width。

### 6.2 Cache-hit-aware batch forming

调度器不只看 token budget，而是把请求分成几类：

- 同一 graph bucket 的 residual prefill。
- 同一 shared prefix group 且 cascade attention 可能收益高的请求。
- residual 太小但会导致 graph fallback 的请求。
- 命中很高但 tail 不对齐、可能值得延后或合并的请求。

### 6.3 Graph-stable chunking

对 chunked prefill 的 `num_new_tokens` 做轻量调整：

- 如果少算几个 token 会落入已捕获 graph bucket，则截断 chunk。
- 如果多算几个 token 能进入更优 bucket，且额外 compute 小于 graph replay 收益，则允许 bounded recompute。
- 如果 batch padding waste 过高，则拆成两个 bucket。

### 6.4 Cascade-aware grouping

把 `num_common_prefix_blocks` 从被动 metadata 变成调度输入：

- 优先合并 shared prefix 足够长且 query 数接近 cascade threshold 的请求。
- 对 256 token / 8 requests 这类硬阈值做 cost-model 替代。
- 在 DBO/microbatching 与 cascade attention 冲突时，选择收益更高的路径。

### 6.5 Cost model

粗略代价函数：

```text
benefit =
  saved_prefill_compute
  + graph_replay_benefit
  + cascade_attention_benefit
  - padding_waste
  - bounded_extra_recompute
  - scheduling_delay_penalty
```

它必须足够简单，避免过度工程化。第一版可以用离线 profiling 表 + 在线常数阈值。

## 7. vLLM 实现路径

### Phase 0：profiling 验证

先不改调度，只记录：

- APC hit ratio 与 residual length 分布。
- CUDA Graph replay / fallback 比例。
- 每 batch padding tokens。
- cascade attention 是否启用以及未启用原因。
- TTFT、TPOT、p95/p99 latency。

### Phase 1：shape-aware token budget

在 scheduler 中加入轻量函数：

- 输入：`num_computed_tokens`、`num_new_tokens`、capture sizes、token budget。
- 输出：调整后的 `num_new_tokens`。
- 目标：减少 graph fallback 和 padding waste。

### Phase 2：shape-aware request grouping

在 waiting/running request 选择阶段加入 grouping：

- 同 graph bucket 优先。
- 同 common prefix group 优先。
- 对过小 residual request 做 short delay，但设置严格 delay budget，保证低延迟。

### Phase 3：cascade-aware policy

替换固定 cascade threshold：

- 用 profiling 建表估计 cascade 收益。
- 对不同模型、head 数、KV head 数和 GPU 做自适应。
- 保留保守 fallback，保证 correctness。

## 8. 为什么可能有效

`ShapeKV` 的收益来自一个经常被忽略的事实：**APC 命中率高不等于 runtime 效率高**。

它可能在这些场景显著有效：

- 多请求共享长系统提示词，但每个请求后缀长度不同。
- RAG/Agent 请求短输出、TTFT 敏感，APC 命中后 residual prefill 很碎。
- CUDA Graph capture sizes 离散，baseline 经常落入 fallback 或过度 padding。
- shared prefix 足够长，但请求到达顺序使 cascade attention 未被触发。

如果 profiling 发现 vLLM baseline 在 APC workload 下 graph replay 率已经很高、padding waste 很低、cascade miss 很少，那么这个方向就不成立。因此它的第一步必须是测量，而不是直接实现复杂调度器。

## 9. 实验设计

核心实验：

- 合成 workload：固定 shared prefix，改变 suffix length 分布和并发度。
- 真实 workload：ShareGPT、多轮 agent trace、RAG trace。
- 模型：Llama/Qwen/Mistral 系列，至少覆盖 GQA 和不同 context length。
- 指标：TTFT、TPOT、throughput、p99 latency、graph replay rate、padding token ratio、cascade enable rate、APC hit ratio。

对比基线：

- vLLM V1 baseline。
- vLLM baseline + APC。
- vLLM baseline + 手动 tuned capture sizes。
- SGLang 类 cache-aware scheduling，如果可复现则作为外部系统对比。
- PAT/ChunkAttention 作为 kernel 层互补基线，而不是直接替代。

消融：

- 只做 graph-stable chunking。
- 只做 request grouping。
- 只做 cascade-aware grouping。
- 不同 delay budget。
- 不同 cost-model 精度。

## 10. 发表性判断

| 维度 | 判断 |
| --- | --- |
| 轻量级 | 高。主要改 scheduler / metadata / policy，不改模型 |
| 普适性 | 高。所有 exact prefix hit workload 都可能受益 |
| 高吞吐 | 高。提高 graph replay 和降低 padding waste |
| 低延迟 | 高。目标直接是 TTFT / p99 |
| vLLM 兼容性 | 高。贴合现有 CUDA Graph、scheduler、cascade attention 路径 |
| OSDI 潜力 | 高，但取决于 profiling 能否证明交互足够显著 |

最终建议：**ShapeKV 是当前最值得主攻的方向。它的关键不是把 scheduler 写复杂，而是用强实验展示：KV cache 命中之后，系统瓶颈从“是否复用”转移到“如何让复用后的 residual work 以稳定形状执行”。**

## 11. 关键资料

- vLLM CUDA Graphs design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM prefix caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM prefix-caching-aware scheduling issue: <https://github.com/vllm-project/vllm/issues/7883>
- SGLang / RadixAttention: <https://arxiv.org/abs/2312.07104>
- SGLang v0.4 blog: <https://lmsys.org/blog/2024-12-04-sglang-v0-4/>
- PAT: <https://arxiv.org/abs/2511.22333>
- DLPM scheduler: <https://arxiv.org/abs/2501.14312>
