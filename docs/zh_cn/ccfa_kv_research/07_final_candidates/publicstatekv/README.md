# PublicStateKV：面向 Agent 与 Reasoning 的公私状态分离缓存

## 1. 一句话定位

`PublicStateKV` 的核心主张是：

> 在现代 agent / reasoning workload 中，serving 系统把“未来应该共享的公共会话状态”和“只服务当前执行过程的私有状态”混在了一起，结果是缓存收益不稳定、状态边界不清晰。系统应该只把可公开复用的 state 变成共享 KV，而不是盲目缓存整个执行历史。

## 2. 这条线为什么会浮出来

这次检索里，一个很重要的外部信号来自：

- [Prompt Cache](https://arxiv.org/abs/2311.04934)
- [Don't Break the Cache](https://arxiv.org/abs/2601.06007)
- OpenAI / Anthropic / Gemini 等 provider 的 prompt caching 文档与实践

它们共同揭示了一个现实：

- prompt caching 的收益高度依赖“哪些内容被放在 cacheable prefix 里”
- 动态 tool results、function calling、agent execution 中间态，都会显著影响收益

但现有大部分实践仍停留在：

- prompt engineering
- block control
- 人工把动态内容往后放

这说明系统层面其实还缺一个问题定义：

> 哪些状态天然属于未来可复用的公共会话状态，哪些状态只是当前执行的私有副产品，不应该进入共享缓存。

## 3. 最近邻工作有哪些

### 3.1 Prompt Cache / modular reuse

代表：

- [Prompt Cache, MLSys 2024](https://arxiv.org/abs/2311.04934)

它的价值在于：

- 让结构化 prompt 片段更容易被复用

但它更像：

- 模块化 attention reuse

而不是：

- **agent/reasoning runtime 的公私状态边界**

### 3.2 provider prompt caching practices

代表：

- [OpenAI Prompt Caching Guide](https://platform.openai.com/docs/guides/prompt-caching)
- [Don't Break the Cache](https://arxiv.org/abs/2601.06007)

这些资料强调：

- 静态内容放前面
- 动态内容放后面
- 排除 tool results 往往更好

但它们没有把这个问题提升成：

- 一个 server-side state management 问题

### 3.3 session/stateful serving

已有一些 stateful/session-aware 路线在做：

- 保存会话状态
- 减少多轮 replay

但它们大多默认：

- 历史状态整体都是可继承对象

而 `PublicStateKV` 要强调的是：

- **不是所有状态都该继承，更不是所有状态都该共享**

## 4. 为什么我认为它还有新意

我这轮没有找到一篇直接把下面这个命题系统化的论文：

> 对 agent/reasoning serving 而言，cacheability 的核心不是“怎么更聪明地命中整个上下文”，而是“如何把 public dialogue state 从 private execution state 中精确分离出来”。

这条线最吸引人的地方在于：

- 它不把问题退化成 prompt engineering
- 也不把问题退化成 session replay
- 而是把“什么 state 可以进入共享缓存”提升成运行时语义

## 5. 在 vLLM 里为什么可做

它不一定主要改 `kv_cache_manager`，更可能落在：

- OpenAI-compatible server
- request/session state management
- chat template / conversation rendering
- 引擎与会话层的 state boundary

这意味着：

- 它不是最短路径 patch
- 但它确实可以作为 vLLM 之上的第一方增强方案

## 6. PublicStateKV 的核心机制

### 6.1 public/private state split

把会话状态分成两类：

- `public state`
  - 将来其他轮次或其他请求应当继承的公开对话状态
- `private state`
  - 只服务当前推理过程的 thinking、tool trace、scratchpad、ephemeral intermediate state

### 6.2 exact public-state materialization

一次 agent/reasoning 轮次完成后，不直接把完整执行历史扔进共享缓存，而是：

- 根据公开 transcript
- 精确重建一个 future-shareable state
- 仅缓存这份 public state 对应的 KV

### 6.3 cache policy by visibility, not recency

块是否应该进入共享 APC，不再只由：

- 是不是 prefix
- 最近用没用过

决定，而要先看：

- 它代表的是 public state 还是 private state

## 7. 预期收益

最可能受益的场景：

- reasoning-heavy chat
- tool-use agent
- 多轮研究/写作/搜索类会话

收益主要体现在：

- 更稳定的 cache hit
- 避免动态 execution traces 污染共享前缀
- 在 agent workload 上更稳定的 `TTFT`

## 8. 风险

### 8.1 exact 边界必须讲清楚

最大风险是：如果 public-state materialization 不是 exact，它就会变成语义近似复用，这会大幅增加论文风险。

### 8.2 系统边界更高

这条线会跨：

- engine
- API server
- session abstraction

范围明显大于 `FastPathKV`。

## 9. 我当前的判断

`PublicStateKV` 是这轮最终保留下来的三个方向里：

- **最前沿**
- **最贴近未来 agent/reasoning workload**
- **但也最需要谨慎定义 exact 边界**

的一条线。

如果你要的是“最稳最轻最容易落地”，它不是第一名；但如果你要的是“可能最有前瞻性的大 thesis”，它值得认真保留。
