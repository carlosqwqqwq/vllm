# 前缀复用前沿论文调研

更新时间：2026-04-15

## 1. 筛选口径

这份调研只纳入与下面问题直接相关的论文：

- `prompt caching / prefix caching / context caching / KV cache reuse`
- 面向 `LLM serving` 或 `LLM inference`
- 优先 `2024-2026`
- 优先一手论文源，主要采用 arXiv 条目和论文原文摘要

不纳入的内容：

- 单纯的 KV 压缩，不涉及跨请求复用
- 纯训练方法，不讨论 serving 复用
- 只谈 agent memory / semantic cache，而不谈 KV / prefix reuse

## 2. 先给判断

这轮文献看下来，前沿已经明显分成四类：

### A. exact-prefix 复用已经成立，重点做系统管理

代表工作：

- `Prompt Cache`
- `Marconi`
- `Cake`
- `KVFlow`

它们主要回答：

- exact prefix 如何暴露给系统
- cache entry 如何 admission / eviction / prefetch
- cache 是算还是载
- workflow 场景下如何更好地保留前缀

### B. 不满足 exact prefix，转去做更灵活的 reuse

代表工作：

- `EPIC`
- `MPIC`
- `KVLink`
- `EFIM`
- `KVShare`
- `RelayCaching`

它们主要回答：

- prefix 不完全一样时如何仍然复用
- 文档、模态、suffix、decode state 是否能经过变换后复用
- selective recompute 如何弥补位置或上下文不一致

### C. prompt caching 的安全、审计与 agentic 产品化

代表工作：

- `Auditing Prompt Caching in Language Model APIs`
- `CacheSolidarity`
- `Don't Break the Cache`

它们说明一件事：

- prompt caching 已经不再只是系统优化技巧，而是已经进入多租户、安全、agentic 真实工作负载层面。

### D. 对我们最重要的空位

几乎没有论文专门系统化研究下面这个问题：

> exact 或 logical-exact 的 reuse 条件其实已经存在，但 runtime 因为 API / backend / observation / carrier 契约不匹配，没有把它兑现成 physical hit。

这正是我们最应该打的地方。

---

## 3. 重点论文逐篇判断

### 3.1 Prompt Cache: Modular Attention Reuse for Low-Latency Inference

- 论文：`Prompt Cache: Modular Attention Reuse for Low-Latency Inference`
- 发布时间：`2023-11-07`
- 发表信息：`MLSys 2024`
- 链接：<https://arxiv.org/abs/2311.04934>
- 关键点：
  - 把可复用 prompt 片段显式建模成 prompt modules。
  - 通过 schema 和位置约束保证 attention state 的正确复用。
  - 目标是低延迟 TTFT。
- 对我们的意义：
  - 它是“显式模块化复用”的代表作。
  - 但它默认 reusable module 已经被显式声明，且命中语义已经成立。
  - 它不回答“系统里已有状态为何没被承认成 hit”。

### 3.2 Marconi: Prefix Caching for the Era of Hybrid LLMs

- 论文：`Marconi: Prefix Caching for the Era of Hybrid LLMs`
- 发布时间：`2024-11-28`
- 发表信息：`MLSys 2025`
- 链接：<https://arxiv.org/abs/2411.19379>
- 关键点：
  - 面向 hybrid models（attention + recurrent / SSM）的 APC。
  - 指出 hybrid 模型里 partial overlap 很难 rollback，因此 exact-match 约束更强。
  - 核心贡献在 admission / eviction，平衡复用收益与内存占用。
- 对我们的意义：
  - 它非常强，也非常相关。
  - 但它重点是“exact hit 已成立之后，怎么管理 cache 更聪明”。
  - 它没有研究 runtime contract 导致的 false-negative exact hit。

### 3.3 Cake: Compute Or Load KV Cache? Why Not Both?

- 论文：`Compute Or Load KV Cache? Why Not Both?`
- 发布时间：`2024-10-04`
- 链接：<https://arxiv.org/abs/2410.03065>
- 关键点：
  - 关注长上下文场景中 prefix cache 的 I/O 瓶颈。
  - 核心思想是并行利用计算资源和 I/O 资源。
  - 动态决定哪些 token 计算、哪些 token 读取。
- 对我们的意义：
  - 这是“hit 已存在，如何把 load / compute 协同做快”的代表。
  - 它优化的是兑现路径的代价，而不是命中资格本身。

### 3.4 KVFlow: Efficient Prefix Caching for Accelerating LLM-Based Multi-Agent Workflows

- 论文：`KVFlow: Efficient Prefix Caching for Accelerating LLM-Based Multi-Agent Workflows`
- 发布时间：`2025-07-10`
- 链接：<https://arxiv.org/abs/2507.07400>
- 关键点：
  - 针对 multi-agent workflow 的 prefix cache 管理。
  - 用 workflow graph 预测未来 agent 激活顺序。
  - 做更细粒度 eviction 和 fully overlapped prefetch。
- 对我们的意义：
  - 它说明 agentic workload 已经是 prefix caching 的强场景。
  - 但它仍然假设可复用前缀已经被系统识别出来。

### 3.5 EPIC: Efficient Position-Independent Caching for Serving Large Language Models

- 论文：`EPIC: Efficient Position-Independent Caching for Serving Large Language Models`
- 发布时间：`2024-10-20`
- 链接：<https://arxiv.org/abs/2410.15332>
- 关键点：
  - 从 exact prefix 推进到 PIC。
  - 目标是在 varying prefixes 下仍复用不可变文档段。
  - 核心难点是 attention sink，提出 `LegoLink` 降低精度损失。
- 对我们的意义：
  - 这是最强近邻之一。
  - 如果我们把故事写成“更灵活复用”或“prefix 不用 exact 也能复用”，很容易撞上它。
  - 所以我们必须坚持：只研究 exact / logical-exact latent hits。

### 3.6 MPIC: Position-Independent Multimodal Context Caching System for Efficient MLLM Serving

- 论文：`MPIC: Position-Independent Multimodal Context Caching System for Efficient MLLM Serving`
- 发布时间：`2025-02-04`
- 链接：<https://arxiv.org/abs/2502.01960>
- 关键点：
  - 把 PIC 扩展到多模态。
  - 强调 interleaved text-image 和 multimodal RAG 下 exact-prefix 太脆弱。
  - 用 integrated reuse + recompute 降低精度损失。
- 对我们的意义：
  - 它占住了“多模态 + 非 exact 位置复用”这条线。
  - 我们不能把论文写成通用 multimodal flexible reuse。

### 3.7 KVLink: Accelerating Large Language Models via Efficient KV Cache Reuse

- 论文：`KVLink: Accelerating Large Language Models via Efficient KV Cache Reuse`
- 发布时间：`2025-02-21`
- 链接：<https://arxiv.org/abs/2502.16002>
- 关键点：
  - 将 document KV 独立预编码，再在推理时拼接。
  - 通过位置嵌入调整和 special tokens 恢复跨文档 self-attention。
  - 主要面向 RAG / 文档复用。
- 对我们的意义：
  - 这是“预切分文档 + 推理期拼接”的代表。
  - 本质上是更强的模块化 / 结构化 reuse。
  - 不属于我们要打的 runtime false-negative 问题。

### 3.8 EFIM: Efficient Serving of LLMs for Infilling Tasks with Improved KV Cache Reuse

- 论文：`EFIM: Efficient Serving of LLMs for Infilling Tasks with Improved KV Cache Reuse`
- 发布时间：`2025-05-28`
- 发表信息：`Euro-Par 2025 Oral`
- 链接：<https://arxiv.org/abs/2505.21889>
- 关键点：
  - 面向 infilling，把 FIM prompt 格式改造成更利于 KV 复用的格式。
  - 说明 prompt format 本身会决定 KV 是否能持续重用。
  - 还引入 fragment tokenization 训练来缓解生成问题。
- 对我们的意义：
  - 这类工作说明“输入形式导致复用失败”是重要问题。
  - 但 EFIM 是 prompt format / model capability 改造，不是 runtime contract restore。

### 3.9 KVShare: An LLM Service System with Efficient and Effective Multi-Tenant KV Cache Reuse

- 论文：`KVShare: An LLM Service System with Efficient and Effective Multi-Tenant KV Cache Reuse`
- 发布时间：`2025-03-17`
- 链接：<https://arxiv.org/abs/2503.16525>
- 关键点：
  - 多租户 KV 复用。
  - 提出 Dual-Stage High Deviation selective recompute。
  - 结合 cache-aware scheduler，兼顾 TTFT、吞吐和精度。
- 对我们的意义：
  - 这是 selective recompute 的强近邻。
  - 但它主要面对跨请求复用误差和 decode 偏移，不是“系统明明可以 exact hit 却跳过读 cache”。

### 3.10 RelayCaching: Accelerating LLM Collaboration via Decoding KV Cache Reuse

- 论文：`RelayCaching: Accelerating LLM Collaboration via Decoding KV Cache Reuse`
- 发布时间：`2026-02-28`
- 链接：<https://arxiv.org/abs/2603.13289>
- 关键点：
  - 在 multi-agent collaboration 中复用前一 agent 的 decoding-phase KV。
  - 观察到 decode 和 prefill 之间对相同内容的 KV 高度一致，只在局部层 / token 上有偏差。
  - 用 selective recomputation 修正稀疏偏差。
- 对我们的意义：
  - 它非常接近“局部修补后兑现复用”。
  - 但它研究的是 phase transfer（decode -> prefill）的一致性，不是 runtime API / backend contract false negatives。

### 3.11 Auditing Prompt Caching in Language Model APIs

- 论文：`Auditing Prompt Caching in Language Model APIs`
- 发布时间：`2025-02-11`
- 发表信息：`ICML 2025`
- 链接：<https://arxiv.org/abs/2502.07776>
- 关键点：
  - 通过 timing audit 识别真实 API 提供商是否共享 prompt cache。
  - 证明 prompt caching 会形成跨用户 timing side channel。
- 对我们的意义：
  - 它不是 serving 优化论文，但非常重要。
  - 说明 prompt caching 已经进入“产品级可观察行为”。
  - 如果我们后面做多池路由或共享策略，要考虑可观测性与隔离性。

### 3.12 CacheSolidarity: Preventing Prefix Caching Side Channels in Multi-tenant LLM Serving Systems

- 论文：`CacheSolidarity: Preventing Prefix Caching Side Channels in Multi-tenant LLM Serving Systems`
- 发布时间：`2026-03-11`
- 链接：<https://arxiv.org/abs/2603.10726>
- 关键点：
  - 面向 APC side channel。
  - 不再一刀切关闭 APC，而是选择性隔离 prefix。
- 对我们的意义：
  - 说明 prefix caching 研究已经进入安全-性能折中层面。
  - 如果我们做 family binding / route binding，要预留 tenant-aware 扩展空间。

### 3.13 Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks

- 论文：`Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks`
- 发布时间：`2026-01-09`
- 链接：<https://arxiv.org/abs/2601.06007>
- 关键点：
  - 不是新 caching 机制，而是系统性评估 agentic workloads 中 prompt caching 的真实收益。
  - 指出 naive full-context caching 未必总是最优。
  - 说明 dynamic tool results、prompt block placement、策略选择都会影响收益。
- 对我们的意义：
  - 它进一步证明 agentic workloads 是非常适合做 prefix reuse 的战场。
  - 也说明我们后续实验必须纳入工具调用与长 session，不然很容易显得 workload 太理想化。

---

## 4. 文献总体结论

### 4.1 已经明显拥挤的路线

- `position-independent / modular / document-level reuse`
- `cache admission / eviction / prefetch / compute-load overlap`
- `multi-agent workflow-aware prefix cache management`
- `multi-tenant prompt caching security`

### 4.2 仍然相对空的路线

下面这个问题，在公开论文里仍然没有看到被系统化占住：

> 当 exact 或 logical-exact 的可复用状态已经存在时，为什么 serving runtime 还会因为执行契约不匹配而把它落成 miss？

这类问题有三个特征：

1. 它不是 hash 冲突问题。
2. 它不是非 exact prefix 问题。
3. 它不是“hit 之后怎么调度”问题。

这正是我们后面文档里提出 `RealizeKV` 的原因。

---

## 5. 对我们选题的直接约束

### 必须坚持的边界

- 只打 `exact / logical-exact` reuse
- 不把论文写成 PIC
- 不把论文写成分布式 cache pool
- 不把论文写成纯安全论文
- 不把论文写成“prompt_logprobs 终于支持 prefix cache”的 feature patch

### 最值得借力的论文 insight

- 从 `Prompt Cache` 借“显式暴露 reuse contract”的思路
- 从 `Marconi` 借“exact-hit 也需要系统级 policy”的视角
- 从 `Cake` 借“realization 需要把 compute / load / schedule 统一考虑”的视角
- 从 `KVShare / RelayCaching` 借“局部 replay / bridge 可以比全量重算更优”的经验
- 从 `Don't Break the Cache` 借“agentic workload 是主战场”的 workload 设计意识

---

## 6. 论文链接汇总

- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Marconi: <https://arxiv.org/abs/2411.19379>
- Cake: <https://arxiv.org/abs/2410.03065>
- KVFlow: <https://arxiv.org/abs/2507.07400>
- EPIC: <https://arxiv.org/abs/2410.15332>
- MPIC: <https://arxiv.org/abs/2502.01960>
- KVLink: <https://arxiv.org/abs/2502.16002>
- EFIM: <https://arxiv.org/abs/2505.21889>
- KVShare: <https://arxiv.org/abs/2503.16525>
- RelayCaching: <https://arxiv.org/abs/2603.13289>
- Auditing Prompt Caching in Language Model APIs: <https://arxiv.org/abs/2502.07776>
- CacheSolidarity: <https://arxiv.org/abs/2603.10726>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>
