# ZeroLagKV：重合度与可行性评估

## 1. 先给结论

这轮深调研后，我对 `ZeroLagKV` 的判断比最初更精确，也更审慎：

- `ZeroLagKV` 抓到的是**真问题**。
  - 在异步 `continuous batching` 中，`schedule()` 和 `update_from_output()` 之间存在时间差，这会让已经“应该变成可复用状态”的新前缀晚一个 step 才被后来者看到。
- 但按宽泛版本来讲，`ZeroLagKV` **不能说“没人做过”**。
  - 工业界已经有很接近的能力：`TensorRT-LLM Early Reuse` 明确支持在前一个请求仍在生成时就开始复用其 KV；
  - 学术界也已经把“completed requests 的 KV cache”当成默认模型来研究，比如 `ATC 2025` 的 `KVCache Cache in the Wild`。
- 所以，`ZeroLagKV` 真正还能站住的版本，**不是泛化的 early reuse**，而是：

> **异步调度下，decode-evolved prefix 的 publication lag 与可见性协议。**

一句话结论：

> `ZeroLagKV` 仍然是当前几个候选里最像系统 thesis 的方向之一，但它不能再宽泛地写成“更早复用 KV”，而必须收紧为“async continuous batching 下的新前缀发布滞后”。 

## 2. ZeroLagKV 真正要解决什么

这轮调研最大的修正是：

- `ZeroLagKV` 不是在解决“静态 prompt 能不能更早复用”；
- 它真正解决的是“**一个请求在本 step 新形成的可共享前缀，为什么在下一次调度时还不可见**”。

这个区别非常关键。

如果用更精确的表述：

- 对于 **已知 prompt token**，系统通常在请求结束前就已经知道其 token 序列和 block hash；
- 对于 **新生成的 decode token / 新形成的会话 continuation prefix**，其内容只有在模型输出回来后才确定；
- 在 async 调度里，下一次 `schedule()` 可以发生在上一次 `update_from_output()` 之前；
- 于是 follower request 看到的是“旧前缀”，而不是“刚刚演化出的新前缀”。

所以，`ZeroLagKV` 的核心命题不是“早发布一切”，而是：

> **对 decode-evolved prefix，publication timing 本身就是性能边界。**

## 3. 业界最接近的工作线

我把最近邻工作分成四组。

### 3.1 工业系统：已经有非常接近的 Early Reuse

最危险的近邻是：

- [TensorRT-LLM Early Reuse blog](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)
- [TensorRT-LLM KV Cache Reuse docs](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)

它们已经公开表达了两件对 `ZeroLagKV` 很危险的事实：

1. 传统 prefix/KV reuse 常常要求前一个请求完成后才能复用；
2. TensorRT-LLM 已经支持在“前一个请求仍在生成时”开始复用其 KV。

这意味着：

- 如果 `ZeroLagKV` 还只是讲“我们要更早复用 active request 的 KV”，那已经和工业界强重合了。

### 3.2 学术界：completed-request KV cache 已是默认研究模型

代表：

- [KVCache Cache in the Wild, USENIX ATC 2025](https://ipads.se.sjtu.edu.cn/_media/publications/kvcache-atc25.pdf)

这篇论文在引言里直接写到：

- 现有 serving 系统一般缓存 **completed requests** 的 KV；
- 它把这类机制统称为 `KV$ cache / prefix cache / prompt cache`。

这对 `ZeroLagKV` 很重要，因为它说明：

> “完成后再进入可复用态”已经是学界默认前提，而不是一个必须如此的真理。

这恰好给了 `ZeroLagKV` 明确的问题切口。

### 3.3 prefix-aware execution / scheduler：优化的是命中后的执行，不是 state visibility

代表：

- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)
- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)
- [Preble, SOSP 2024 workshop / arXiv](https://arxiv.org/abs/2407.00023)

这些工作很接近，但主问题不同：

- `ChunkAttention` / `PAT`
  - 优化 shared-prefix 的 attention 执行与 kernel 路径
- `Preble`
  - 更关注 prefix-aware routing / scheduling / locality

而 `ZeroLagKV` 要强调的是：

- **prefix state 何时变成其他请求可见**

这和“命中后怎么跑更快”不是同一个问题。

### 3.4 stateful / session-aware serving：保存状态，但不研究 in-flight visibility

代表：

- [HCache](https://arxiv.org/abs/2410.05004)
- 一系列 session-aware / state restoration 路线

这些工作主要回答：

- 会话状态怎么跨轮次保存
- 如何减少 replay

而 `ZeroLagKV` 关注的是：

- **正在运行中的状态为什么还来不及被后来者看到**

## 4. 和我们 idea 的真实重合度

| 工作 | 相关性 | 为什么像 | 为什么还不完全一样 |
| --- | --- | --- | --- |
| TensorRT-LLM Early Reuse | 很高 | 都在强调 active request 的 KV 可以早于完成时被别人复用 | 公开形式更像工业能力说明，没有围绕 async scheduling 中的 publication lag 建立完整 thesis |
| TensorRT-LLM KV Cache Reuse docs | 很高 | 直接承认传统算法通常要等前一个请求终止后再复用 | 没有把 decode-evolved prefix 的 step-level lag 抽象成问题 |
| KVCache Cache in the Wild | 高 | 明确把 completed-request KV cache 当成默认模型来研究 | 重点在工作负载特征和 eviction，不在活跃请求可见性 |
| Preble | 中高 | 也在做 prefix-aware scheduling 和 locality | 不研究 state publication timing |
| ChunkAttention / PAT | 中 | 都优化 shared-prefix 场景 | 关注 kernel/execution，不关注 publication protocol |
| HCache 等 stateful serving | 中 | 都和会话状态有关 | 不研究 in-flight state visibility |

我的最终判断是：

- **没有找到一篇和 ZeroLagKV 完全同构的正式论文；**
- 但也**绝对不能再说“这个方向没人做过”**；
- 最准确的说法应该是：
  - `ZeroLagKV` 的“async publication lag as thesis”我暂时没找到现成 peer-reviewed work；
  - 但“active/in-flight reuse”本身，工业界已经非常接近，甚至已经有公开实现。

因此，如果你的要求是“确保没有人做过”，那么以目前证据，我**不能给出这个保证**。

## 5. vLLM 当前实现到底做到了哪里

这是这条线最关键的部分。

### 5.1 vLLM 已经不是“只在请求结束后发布”

本地代码明确说明：

- `kv_cache_manager.allocate_slots()` 在请求尚未结束时就会调用 `self.coordinator.cache_blocks(...)`；
- `cache_blocks()` 提交的是已经“committable”的 full blocks；
- `block_pool.touch()` 明确允许 `ref_cnt > 0` 的 block 被别的请求继续共享。

这说明一个非常重要的事实：

> vLLM 已经具备某种程度的“活跃请求中间态可复用”能力。

所以，`ZeroLagKV` 不能再把 thesis 讲成：

- “让活跃请求的 KV 也能共享”

因为这在本地实现层面已经部分成立了。

### 5.2 真正还没解决的是 decode-evolved prefix

异步调度路径里，本地代码给出了更关键的证据：

- `engine/core.py` 的 async batch queue 允许先执行新的 `scheduler.schedule()`，再等待上一轮 `future.result()`；
- `AsyncScheduler._update_after_schedule()` 会先给运行中的请求增加 output placeholders；
- 真正把新 token 写回请求并调用 `kv_cache_manager.cache_blocks(...)`，是在 `AsyncScheduler._update_request_with_output()`；
- 也就是说，**新生成 token 对应的新前缀，要等 output update 后才进入 cache visibility**。

这意味着：

- 对于静态 prompt 前缀，vLLM 可能已经部分提前发布；
- 对于新 decode token 演化出的 prefix，仍然存在真实的 publication lag。

这正是 `ZeroLagKV` 还站得住的核心空隙。

### 5.3 这条线在 vLLM 上为什么依然自然

因为它不需要完全推翻现有实现，而是沿着已有语义继续推进：

- `cache_blocks()` 已经存在；
- block 已经允许被 active producer 和 follower 同时引用；
- async scheduling 的时间差已经存在；
- 请求对象已经有 `num_output_placeholders` 这类“尚未完成但已预占位”的状态。

换句话说：

> vLLM 已经有了 `ZeroLagKV` 所需的 70% 基础设施，剩下的是把“何时可见”这件事正式化。

## 6. 这条线在 vLLM 里能做吗

结论是：**能做，而且比想象中更顺着当前架构。**

### 6.1 可能的核心机制

如果真的实现，至少需要这三块：

1. **decode-aware publish protocol**
   - 区分 prompt-known full blocks 与 decode-evolved blocks；
   - 后者需要在输出 ready 之后立即进入 published state，而不是等下轮普通更新。
2. **visibility tiers**
   - `private / published / stable`
   - 让 follower 能安全读取 producer 仍持有的块。
3. **follower-aware scheduling**
   - scheduler 在候选 follower request 上考虑“是否值得等一个即将发布的新前缀”。

### 6.2 最可能改动的位置

- `vllm/v1/core/sched/async_scheduler.py`
  - 让 `_update_request_with_output()` 的发布行为显式化
- `vllm/v1/engine/core.py`
  - 在 async batch queue 中暴露更明确的 publication point
- `vllm/v1/core/kv_cache_manager.py`
  - 增加 visibility state / publish API
- `vllm/v1/core/block_pool.py`
  - 明确 published-but-active block 的生命周期

### 6.3 最大实现风险

最大的风险不是“做不出来”，而是：

- **边界会不会太窄**

因为经过这轮调研后，`ZeroLagKV` 真正剩下的创新空间已经从“所有早发布”缩到了：

- **decode-evolved prefix**
- **async queue 中的 step-level lag**

如果这个窗口太窄，评审会觉得：

- 只是把一个异步实现细节抬成论文。

## 7. 它会有效吗

我的判断是：**会有效，但 workload 依赖性比最初想象更强。**

### 7.1 最可能受益的场景

- 多轮 chat，同一会话短时间连续 follow-up；
- agent / tool-use 中父请求刚输出，子请求马上接着来；
- bursty prefix-heavy workload，且 follower request 依赖 producer 刚生成出的少量 continuation token。

这些场景的共同点是：

- follower 真正想复用的，不只是静态 system prompt；
- 而是 producer **刚刚多生成出来的那一点新前缀**。

### 7.2 不太会有大收益的场景

- 单轮独立请求；
- 主要共享的是固定 system prompt；
- follower 到达时间很晚，producer 早就完成；
- 低并发或非 async scheduling。

在这些场景里：

- `ZeroLagKV` 的价值会被现有 prefix cache 或普通 session cache 覆盖掉。

## 8. 它够不够领先最新论文

如果“最新论文”指 peer-reviewed academic papers，我的判断是：

- **它仍然有一定错位空间**；
- 因为我暂时没找到把 `async publication lag` 本身写成 thesis 的论文。

但如果把“业界最新进展”也算进去，就不能说它领先得很稳：

- `TensorRT-LLM Early Reuse` 已经是非常强的现实对照；
- 它至少证明了“active request 早复用”不是空白地带。

因此，最准确的说法应该是：

- `ZeroLagKV` 有机会在**学术问题定义**上领先；
- 但在**工业能力层面**，已经有非常接近的公开实现。

## 9. 它够不够支撑 OSDI / CCF-A

以当前版本，我给出的判断是：**有潜力，但必须进一步压窄和拔高。**

和 `FastPathKV` 相比，它更像一篇大论文，因为：

- 它抓的是一个 runtime principle：
  - state visibility timing matters
- 而不是一个纯工程优化点。

但它也有三个明确风险：

1. **工业重合风险高**
   - `Early Reuse` 太接近。
2. **scope 变窄后可能不够“大”**
   - 如果只剩 decode-evolved prefix 的一 step lag，评审可能觉得命题太细。
3. **需要 correctness story**
   - published block 的版本语义、draft token 边界、spec decode 兼容性都要讲清楚。

所以我的当前结论是：

- 比 `FastPathKV` 更像 `OSDI thesis`
- 但还没有到“可以放心说这条线稳赢”的程度

## 10. 最后判断

我的最终建议是：

- **保留 ZeroLagKV，但必须重写问题定义。**

更稳的写法不是：

- “active request 的 KV 更早复用”

而是：

> **async continuous batching 下 decode-evolved prefix 的 publication lag**：请求在 step `N` 新形成的可共享前缀，在 step `N+1` 的调度决策中仍不可见，从而造成结构性重复 prefill 与额外时延。

只有把 thesis 定位到这里，`ZeroLagKV` 才能：

- 和 TensorRT 的通用 early reuse 拉开一点差距；
- 和 completed-request KV cache 文献形成清晰对照；
- 在 vLLM 上找到直接、自然、可验证的代码落点。

## 11. 这轮用到的关键来源

- [TensorRT-LLM Early Reuse blog](https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/)
- [TensorRT-LLM KV Cache Reuse docs](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)
- [KVCache Cache in the Wild, USENIX ATC 2025](https://ipads.se.sjtu.edu.cn/_media/publications/kvcache-atc25.pdf)
- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)
- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)
- [Preble](https://arxiv.org/abs/2407.00023)
- [HCache](https://arxiv.org/abs/2410.05004)
