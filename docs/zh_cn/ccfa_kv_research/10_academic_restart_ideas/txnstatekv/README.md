# TxnStateKV：把可复用状态提升为事务对象

## 1. 一句话 thesis

`TxnStateKV` 的核心主张是：

> 在 agentic、async、tool-calling、interception-rich 的 LLM serving 中，可复用状态不是静态 cache entry，而是带有 `private / published / committed / paused / aborted` 生命周期的前缀状态对象；如果系统没有统一生命周期语义，就只能靠保守规则处理发布、暂停、恢复、回滚与共享。

补充说明：

- 这只是内部工作名。
- 截至 `2026-04-15`，更严格的重复性与可行性复审已经表明：如果对外把它直接写成“transactional KV cache”，会和最新近邻工作过近。
- 因此，这个方向后续更安全的表达应是：
  - **面向 `vLLM` 的可复用前缀状态生命周期协议**
- 详见：
  - [overlap_feasibility_assessment.md](./overlap_feasibility_assessment.md)

## 2. 学术问题定义

今天很多 serving 系统仍默认这样理解 cache：

- 某段状态要么可复用
- 要么不可复用
- 要么已缓存
- 要么未缓存

但现代 workload 已经把这个二值模型打破了。

例如：

- tool call 期间，请求会暂停，但上下文稍后还会恢复
- async scheduling 下，某些 token 已经“部分产出”但未稳定提交
- spec decode 或 branch workload 中，状态可能先发布后回滚
- multi-agent / interception workload 中，同一上下文可能被多个后续步骤继续消费

所以真正的问题不是：

- “状态有没有命中 cache”

而是：

- “状态当前处于什么生命周期阶段”
- “哪个阶段允许共享，哪个阶段只允许本地持有”
- “状态如何安全地发布、恢复、分支和回滚”

这就是 `TxnStateKV` 的核心问题。

## 3. 为什么它不是工程 patch

这条线的贡献不是：

- 再做一个更快的 publication path
- 再做一个 pause/resume 优化
- 再做一个 tool-stall offload 策略

这些都只能算机制。

`TxnStateKV` 真正的学术价值在于提出一个新的系统语义：

> reusable state should obey transactional lifecycle semantics

也就是：

- 状态何时 private
- 何时 published
- 何时 committed
- 何时 paused
- 何时 aborted

这些不应继续是分散在不同模块里的 ad hoc 规则，而应成为 runtime 的正式协议。

## 4. 最近邻工作与边界

### 4.1 最近邻

- `INFERCEPT`
  - 研究 augmented LLM 中 interception 导致的上下文暂停与恢复。
- `Tokencake`
  - 研究 multi-agent tool-call workload 下的 space/time scheduler。
- `KVFlow`
  - 研究 workflow-aware prefix cache 保留与预取。
- `TensorRT-LLM Early Reuse`
  - 研究状态更早可见、更早复用。

### 4.2 它们和 TxnStateKV 的区别

这些工作分别覆盖了 lifecycle 的某一个片段：

- pause/resume
- stall/offload
- workflow-aware prefetch
- early publication

但 `TxnStateKV` 讨论的是更统一的命题：

> Can reusable state in LLM serving be governed by a unified lifecycle protocol rather than a collection of special-case rules?

也就是说，这条线研究的是生命周期语义本身，而不是某个阶段的局部优化。

## 5. 为什么这个问题现在重要

原因很直接：LLM serving 的“状态”已经不再是静态 prefix。

过去比较典型的复用场景是：

- 多轮 chat
- 重复 system prompt
- 长文档问答

但现在越来越多请求是：

- agent 执行中断后继续
- tool call 返回后恢复
- 多分支推理
- structured output / speculative path 与真实 path 并存

在这些场景里，“状态生命周期”本身就是性能与正确性问题。

如果系统没有统一语义，就很容易：

- 过早发布导致 correctness 风险
- 过晚发布导致复用损失
- 保留过久导致显存浪费
- 回滚不清晰导致状态污染

## 6. TxnStateKV 的核心机制

### 6.1 事务状态机

第一版可以定义几个最小状态：

- `private`
  - 仅 producer 可见
- `published`
  - 可被后续请求探测到，但尚未最终稳定
- `committed`
  - 可被普通 APC / restoration 正常复用
- `paused`
  - 当前请求挂起，状态需被保留但不活跃
- `aborted`
  - 不可再共享，需要回收或回滚

### 6.2 生命周期规则

系统要明确定义：

- 什么粒度可以 publish
- 什么条件下 publish 会升级成 commit
- paused 状态如何保温、何时降级
- aborted 状态如何从全局可见集合中撤回

### 6.3 调度联动

调度器不只看：

- cache hit ratio
- token budget

还要看：

- 当前系统里有多少 `paused / published / committed` 状态
- 哪些请求值得等待一个即将 `commit` 的状态
- 哪些 paused state 应降级、迁移或释放

## 7. 为什么它能在 vLLM 中验证

这条线虽然是更一般的系统命题，但 `vLLM` 已经出现了足够多的实现信号：

- async scheduling 路径里已有 `num_output_placeholders`
- 也已有 `discard_latest_async_tokens`
- scheduler 和 request 里已经区分了多种“未完全稳定”的 token / output 状态
- `kv_cache_events` 机制说明状态发布与可见性本身已经在长出来

因此，在 `vLLM` 中可以做一个轻量 prototype：

1. 定义 full-block publish / commit 规则
2. 为 paused requests 引入显式状态
3. 给 async / speculative 路径补 rollback 语义
4. 让 scheduler 感知这些 lifecycle class

这足以验证 thesis，而不需要一开始就做一个大而全的 runtime。

## 8. 为什么它符合你的要求

### 8.1 轻量级

第一版主要是协议、元数据和 scheduler/manager 改造，不要求复杂分布式系统。

### 8.2 普适性

只要系统支持：

- prefix cache
- async scheduling
- pause/resume
- tool-rich or agentic workload

这条线就成立，不局限于单一模型或硬件。

### 8.3 高吞吐

统一 lifecycle 语义可以减少无谓重算、降低 stall 期间的显存占用，并提高状态再利用率。

### 8.4 低延迟

如果 commit/publish 更合理，TTFT 和恢复时延都可能改善。

## 9. 最大风险

最大的风险是 workload 边界。

如果只在非常 agentic、非常复杂的工作负载上才成立，那么它会显得：

- 适用面太窄
- 不够像通用 serving 问题

因此它必须证明：

1. 事务化生命周期不是 agent 特有技巧
2. 即便在更常见的 async / pause-resume / structured serving 中也成立

## 10. 当前判断

截至这轮调研，`TxnStateKV` 是一个**很像完整系统 thesis** 的方向。

它的强点在于：

- 问题边界清楚
- 抽象层次高于单点优化
- 和现有 early reuse / intercept / stall/offload 工作不同

但它的成功前提是：必须把 workload 边界讲清楚，避免被理解成“只为 agent 系统设计的一套状态机”。

进一步的严格版本见：

- [overlap_feasibility_assessment.md](./overlap_feasibility_assessment.md)

## 11. 关键资料

- INFERCEPT: <https://arxiv.org/pdf/2402.01869>
- Tokencake: <https://arxiv.org/abs/2510.18586>
- KVFlow: <https://arxiv.org/abs/2507.07400>
- TensorRT-LLM Early Reuse: <https://developer.nvidia.com/blog/5x-faster-time-to-first-token-with-nvidia-tensorrt-llm-kv-cache-early-reuse/>
- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
