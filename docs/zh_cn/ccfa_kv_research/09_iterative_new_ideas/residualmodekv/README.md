# ResidualModeKV：命中 APC 后的执行模式重选

## 1. 一句话 thesis

`ResidualModeKV` 的核心主张不是“命中 prefix cache 越多越好”，而是：

> 对 colocated vLLM 来说，exact prefix hit 之后的剩余 prefill 可能更适合另一种执行模式；系统应当在 reuse、少量补算、graph bucket 对齐、以及 cascade-attention eligibility 之间做联合选择，而不是默认走一条固定命中路径。

## 2. 为什么这条线还值得保留

这条线之所以被保留，不是因为 “prefix-aware scheduling” 还没人做，而是因为以下组合问题目前仍没有被完全覆盖：

1. `exact APC hit` 改变的是 **post-cache residual work**，不是原始 prompt length。
2. vLLM 的 `CUDA Graph dispatch` 与 `BatchDescriptor` 依赖运行时 shape。
3. `cascade attention` 仍然依赖启发式阈值与请求组合。
4. 现有系统往往优化的是长度分桶、prefix-aware queueing、或 shared-prefix kernel；但“命中 APC 之后是否还该继续完全复用”这个决策没有被系统化。

换句话说，这个方向的切口已经不再是 `ShapeKV` 早期那种泛化版 “cache-aware shape scheduling”，而是更收窄的：

- **post-hit**
- **residual-aware**
- **mode-selection**
- **colocated vLLM**

## 3. 最近邻工作与重合风险

### 3.1 最危险的最近邻

`LAPS` 是最危险的最近邻。它已经做了：

- short re-prefill 识别；
- waiting window；
- CUDA Graph clustering；
- graph-shape 对齐与 padding 减少。

所以，下面这种说法已经不安全：

- “short residual prefill 的 graph-aware batching”
- “cache-aware graph bucket clustering”

### 3.2 其它相关工作

- `k-LPM / DLPM / Preble`
  - 重点是 prefix-aware scheduling、公平性、router 或 waiting queue 次序。
- `SGLang v0.4`
  - 重点是 cache-aware load balancing 与零开销 batch scheduler。
- `ChunkAttention / Hydragen / PAT`
  - 重点是 shared-prefix attention kernel 或 shared decode computation。

### 3.3 我们和它们的边界

`ResidualModeKV` 应该明确和这些工作拉开边界：

1. 它不是 queueing 理论论文
   - 不以等待队列重排为核心。
2. 它不是 prefix-aware kernel 论文
   - 最小原型不需要先改 kernel。
3. 它不是 short-prefill graph batching 论文
   - 重点不是 “短”，而是 “命中后 residual 的可选执行模式”。
4. 它不是 simply reuse-more
   - 关键贡献恰恰可能是：某些场景下少量补算反而更快。

## 4. 为什么它在 vLLM 中真实存在

这条问题在 vLLM 里不是假设，而是能从当前路径上直接看到：

1. `scheduler` 里已经根据 `num_computed_tokens / num_new_tokens` 决定每个 request 本轮补算多少 token。
2. `gpu_model_runner` 会在真正执行前同时处理：
   - 共享前缀长度；
   - cascade attention metadata；
   - padding；
   - CUDA Graph mode 与 batch descriptor。
3. `flash_attn` 的 cascade attention 仍然有硬阈值。

这说明：

- prefix hit 已经影响 residual shape；
- residual shape 已经影响 graph mode；
- graph mode 已经和 cascade eligibility 纠缠在一起；
- 但 scheduler 没有把它们作为同一个联合优化问题来处理。

## 5. 这条线到底新在哪里

最能防守的版本是：

> Use exact hit metadata to choose the best residual execution mode after APC, rather than blindly replaying the cache-saving path.

它的新意不在某个单点机制，而在这个系统判断：

- APC 命中了，不代表 “直接按最短残差执行” 总是最快；
- 如果补算几个 token 能进入已捕获 graph；
- 如果略微推迟 1-2 个 request 能形成更高的 cascade eligibility；
- 如果某些 residual shape 会频繁触发 eager fallback；
- 那么 “严格最大化命中长度” 不是最佳系统策略。

这和已有工作最大的不同，是把 **reuse itself** 变成了可以被优化器重新评估的对象。

## 6. 可行性判断

### 6.1 为什么我认为它可实现

最小原型不需要先动 attention kernel，可以只做三步：

1. 加 profiling
   - 记录 `APC hit ratio`
   - residual token histogram
   - graph replay / fallback
   - padding ratio
   - cascade eligible / enabled / disabled reason
2. 在 scheduler 做轻量 mode selection
   - `reuse as-is`
   - `bounded recompute`
   - `tiny delay for grouping`
3. 在执行前打通代价函数
   - 用 residual length、bucket 命中、common prefix blocks、delay budget 做决策

### 6.2 为什么我认为它可能有效

最容易有效的 workload：

- APC 命中高，但 residual 很碎；
- 长 system prompt / tool schema / few-shot 前缀共享；
- TTFT/p99 很敏感；
- batch 形状不稳定，graph replay 率不高。

### 6.3 什么时候它会失败

- decode 主导，总体瓶颈不在 residual prefill；
- baseline 下 graph replay 已经很高；
- cascade attention 基本不可能启用；
- workload 过于均匀，mode selection 没什么优化空间。

## 7. 是否足够支撑系统论文

我现在的判断是：**有希望，但前提比上一轮更苛刻。**

至少要证明三件事：

1. baseline vLLM 在 APC 高命中时仍有可观的 graph/cascade inefficiency；
2. 这些 inefficiency 不是 `LAPS` 一类 length-aware 方法已经解释掉的；
3. 我们的联合 mode selection 能同时改善：
   - `TTFT`
   - `throughput`
   - `graph replay rate`
   - `padding waste`
   - `cascade usage`

如果只能拿到很小收益，这条线更适合作为 `TypedPrefixKV` 的执行层组件，而不是独立 OSDI 主线。

## 8. 当前结论

截至这轮调研，`ResidualModeKV` 是**最值得先做 profiling 验证**的方向。它不是零重合方向，但只要 thesis 写成：

- exact hit 后的 residual mode selection
- colocated vLLM
- graph/cascade/padding 联合决策

它依然有比较清楚的防守空间。

如果需要更严格的“是否重复 / 是否足够支撑论文 / 是否能在 vLLM 中落地”的判断，请继续阅读同目录下的
`overlap_feasibility_assessment.md`。

## 9. 参考资料

- vLLM CUDA Graphs: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- LAPS: <https://arxiv.org/abs/2601.11589>
- k-LPM: <https://arxiv.org/abs/2502.04677>
- Preble: <https://arxiv.org/abs/2407.00023>
- SGLang v0.4: <https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/>
- PAT: <https://arxiv.org/abs/2511.22333>
