# PromptABIKV：为 exact-prefix serving 建立稳定输入契约

## 1. 一句话 thesis

`PromptABIKV` 的核心主张是：

> 很多 APC miss 并不是 runtime 不够强，而是上层应用并没有稳定地产生同一个 exact prefix；因此 vLLM 需要一个面向 chat / tools / agent / multimodal 应用的 cacheability ABI，让“可命中前缀”成为显式、可诊断、可维护的系统契约。

## 2. 为什么它值得单独保留

这条线之所以没有被直接并入 `TypedPrefixKV`，是因为它解决的是另一层问题：

- `TypedPrefixKV` 关心的是 runtime 内部如何定义 exact prefix；
- `PromptABIKV` 关心的是应用层如何持续生成这个 exact prefix。

这不是空想。外部证据已经很明确：

1. OpenAI 文档明确写到 exact prefix match 只在静态内容放前面、动态内容放后面时才稳定，并且 tools、images、schema 都必须保持一致。
2. Anthropic 文档明确写到 cache prefixes 有 `tools -> system -> messages` 的层级顺序，动态内容位置和 breakpoint 布局会影响命中。
3. `Don't Break the Cache` 这篇 2026 年工作也证明了：agentic workload 下，动态 tool results、traditional function calling、prompt block 摆放都会显著影响缓存收益。
4. vLLM issue #16634 说明 raw prompt / chat template 渲染路径本身就可能改变最终输入格式。

所以，这条线的真实问题不是“怎么写 prompt 更优雅”，而是：

> 在 exact-prefix serving 时代，prompt 组织方式本身已经成为系统接口问题。

## 3. 最近邻工作与重合风险

### 3.1 最近邻

- provider prompt-caching docs
  - 会给出大量 best practices。
- `Don't Break the Cache`
  - 会给出跨 provider 的 agentic workload 评测结论。
- `Prompt Cache`
  - 会把 prompt 组织成可复用模块。
- `Serve Programs, Not Prompts`
  - 会从更高层把 serving 对象升级成程序。

### 3.2 我们必须避免的表述

`PromptABIKV` 不应该写成：

- “prompt caching 最佳实践总结”
- “如何写更利于缓存的 prompt”
- “更好的 tool ordering guideline”

那样会退化成经验帖或工程指南。

### 3.3 仍然可防守的边界

它应该写成一个**接口与运行时协同系统**：

1. runtime 暴露 cache-breaker observability；
2. renderer / template / tool schema / multimodal wrapper 遵循统一 ABI；
3. request 在进入 tokenizer 之前就能被标注为：
   - stable prefix
   - dynamic suffix
   - cache scope
   - breaker reason
4. vLLM 对 ABI 合规请求提供更稳定的 APC 行为与诊断。

它的重点不是“教用户写 prompt”，而是“让 prefix cache 从隐式行为变成显式接口契约”。

## 4. 为什么它在 vLLM 上可落地

vLLM 已经有几个很适合承接 ABI 的位置：

1. renderer / preprocess 路径
   - chat template、raw prompt、multimodal placeholder 都在这里决定最终输入。
2. input processor
   - 在这里还能看到 prompt embeds、mm hashes、cache_salt 等元数据。
3. request construction / scheduling
   - 可以在这里注入 cacheability diagnostics。

因此，`PromptABIKV` 不需要改模型，也不一定需要改 kernel。它更像：

- 一个 cacheability contract
- 一个 diagnostics subsystem
- 再加少量 renderer/input-side canonicalization

## 5. 为什么它可能有效

它的收益逻辑是非常“系统”的：

1. 提高稳定 hit rate
   - 不是靠更复杂的 cache index，而是减少无意义的格式漂移。
2. 降低 TTFT 波动
   - 命中更稳定，尾延迟会更可控。
3. 降低应用侧与 serving 侧的调试成本
   - 当 cache miss 时，系统能告诉你是哪个字段、哪段模板、哪组工具顺序把缓存打碎了。
4. 让 agent / RAG / tool-rich workload 真正能持续吃到 APC
   - 而不是“文档说能命中，线上总是偶发 miss”。

## 6. 最大风险

这条线最大的风险不是技术，而是**会不会被审稿人理解成 prompt engineering**。

所以，它必须坚持三个边界：

1. 不讨论 prompt 质量，只讨论 cacheability contract；
2. 不做 semantic cache，只做 exact-prefix stability；
3. 不只是 guideline，而是 runtime-observable、可验证、可 enforced 的 ABI。

## 7. 是否足够支撑系统论文

单独作为 OSDI 主线，我认为它的风险高于 `ResidualModeKV` 和 `TypedPrefixKV`。但它有两个非常重要的价值：

1. 它是现实 workload 命中不稳定的根因之一；
2. 它可以成为另一个主线的接口层贡献。

所以更稳妥的定位是：

- 作为 `TypedPrefixKV` 的外部接口层；
- 或作为 `ResidualModeKV` 的 cache-stability 配套系统；
- 如果后续 trace 证明线上 miss 大量来自模板漂移，再考虑升格为独立主线。

## 8. 当前结论

截至这轮调研，`PromptABIKV` 不适合立刻排到第一优先级，但它是一个**非常值得留下来的支撑方向**。它和传统“prompt engineering”最大的不同，是它要求 vLLM runtime 对 cacheability 给出可观测、可归因、可 enforced 的系统承诺。

## 9. 参考资料

- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>
- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Serve Programs, Not Prompts: <https://yecl.org/publications/gim2025hotos.pdf>
- vLLM issue #16634: <https://github.com/vllm-project/vllm/issues/16634>
- vLLM issue #16016: <https://github.com/vllm-project/vllm/issues/16016>
