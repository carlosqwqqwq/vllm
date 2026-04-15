# TypedPrefixKV：面向异构输入的精确前缀语义

## 1. 一句话 thesis

`TypedPrefixKV` 的核心主张是：

> vLLM 的 APC 不应该只被理解为“文本 token 前缀复用”，而应该被提升为“异构输入上的精确前缀语义系统”，统一覆盖 token prompt、prompt embeds、多模态输入、tool-rich prompt、以及 cache scope。

## 2. 这条线是怎么冒出来的

这条线不是凭空想出来的，而是被 vLLM 社区最近的问题逼出来的：

1. `prompt embeds + prefix caching` 在社区里曾被明确指出存在断点，issue #25096 甚至直接建议把 prompt embeds 的 canonical representation 加入 block hash；但从当前代码看，这个子问题已经开始被实现，不能再单独作为论文主创新。
2. 多模态 prefix caching 在 issue #20261 里暴露出视觉输入与缓存键之间的正确性问题。
3. `cache salting` 的 RFC #16016 表明 vLLM 已经在面对“同一请求中不同片段是否具有不同共享域”的问题。
4. raw prompt / chat template 绕过问题 #16634 说明“到底什么才算同一个 prompt”并不只是 tokenizer 问题。

这些信号放在一起说明：**精确前缀复用的边界已经从纯文本 token 扩展到了 typed input semantics。**

## 3. 最近邻工作与重合风险

### 3.1 相关但不等价的工作

- `Prompt Cache`
  - 核心是 modular / structured prompt reuse，允许更灵活的模块级复用。
- `Serve Programs, Not Prompts`
  - 核心是把 prompt-serving 升级成 program-serving 架构。
- `Don't Break the Cache`
  - 核心是跨 provider 的 prompt caching 评测与实践建议。
- `OpenAI / Anthropic` prompt caching 文档
  - 明确要求 exact-prefix match，并强调 tools、images、schema 都会影响缓存命中。

### 3.2 我们和它们的边界

`TypedPrefixKV` 不应该写成：

- “更聪明的 modular prompt reuse”
- “更通用的 prompt programming framework”
- “provider prompt caching best practices”

它的边界应该是：

1. **exact reuse，不做 approximate reuse**
   - 不是 semantic cache，也不是 fuzzy match。
2. **runtime-side typed exactness**
   - 讨论的是 vLLM 内部如何表达、哈希、验证、发布一个可复用前缀。
3. **heterogeneous inputs under one cache contract**
   - 不只是 token；还包括 embeds、mm hashes、tool/schema/salt 等缓存相关输入。

换句话说，它真正新的是：

> 给 APC 建立一个跨输入载体的一致“精确语义”，而不是继续让不同输入路径各自决定能不能缓存、怎么 hash、什么时候失效。

## 4. 为什么它对 vLLM 是真实问题

本地代码与社区现状已经能看到几个关键事实：

1. `inputs/engine.py` 里，`TokensInput`、`EmbedsInput`、`MultiModalInput` 本来就是并列输入类型。
2. `input_processor.py` 会把 multimodal hashes、placeholders、prompt embeds、cache_salt 一起带入 `EngineCoreRequest`。
3. `kv_cache_utils.py` 的 block hash 依然主要是 full-block token 语义，只在特定情况下拼入额外 key。
4. 社区 issue #25096 已经明确把“prompt embeds 的 canonical representation 加入 hash”作为后续工作。

这说明 vLLM 现在实际上处于一种“输入已经异构，缓存语义仍然偏 token/block”的状态。

## 5. 这条线的系统价值

如果只把它当“补 feature”，价值会很弱；但如果把它写成下面三个层次，它就更像系统论文：

### 5.1 统一语义层

定义 typed prefix 的组成：

- token span
- prompt embeds fingerprint
- multimodal placeholder layout
- multimodal content hash
- tool/schema fragment
- cache scope / salt

### 5.2 统一哈希与验证层

解决几个实际问题：

- 哪些字段进入 block hash；
- 哪些字段只进入 prefix scope；
- 哪些字段必须按顺序累积；
- 哪些字段需要 canonical serialization；
- 如何在不牺牲 correctness 的前提下保留高吞吐。

### 5.3 统一发布与失效层

让 APC 的可复用边界不再分散在多个特例里：

- prompt embeds 支持与否；
- 多模态是否安全；
- tool/schema 是否等价；
- cache salt 如何影响共享域；
- template bypass 是否仍能保证 cache correctness。

## 6. 为什么它可能有效

这条线的收益不是只体现在一个 benchmark 上，而是体现在 APC 的覆盖面上：

1. 让本来被禁用或不稳定的输入路径获得 APC；
2. 降低多模态 / prompt embeds / tool-rich workloads 的错误 miss；
3. 避免 correctness bug 导致的保守禁用；
4. 把“可缓存”从文本专属能力扩展成统一 runtime 能力。

更现实一点说，这条线可能不会像论文里那种“单点带来 2x 吞吐”，但它有机会带来更难替代的贡献：

- 让 APC 进入更多 workload；
- 让 vLLM 的 prefix cache 从 feature 变成 subsystem。

## 7. 可行性判断

### 7.1 为什么我认为可实现

最小落地步骤比较清楚：

1. 把 typed prefix 明确成内部 schema；
2. 为 prompt embeds、mm hashes、cache salt 定义 canonical fingerprint；
3. 调整 block-hash extra keys 的组织方式；
4. 在支持路径上补 correctness test 和 hit/miss observability；
5. 最后再考虑对 scheduler 暴露 typed hit metadata。

### 7.2 最大风险

最大的风险不是实现难，而是**论文包装难**：

- 如果只做 “让 prompt embeds 支持 prefix caching”，会像 feature patch。
- 如果只修一个 multimodal bug，会像 correctness fix。

所以它必须被讲成：

> a unified exact-prefix semantics for heterogeneous vLLM inputs

只有这样，才能和单点 issue 修复拉开层级。

## 8. 是否足够支撑系统论文

我认为它**有可能成为比 `ResidualModeKV` 更新颖的主线**，但前提是要把证据链补齐：

1. 证明异构输入已经是主流 workload，而不是边缘场景；
2. 证明现有 APC 语义分裂确实带来：
   - 错误 miss
   - correctness 风险
   - 功能保守禁用
   - 性能浪费
3. 证明统一 typed prefix contract 后，收益不仅是 correctness，而是：
   - 更高 hit rate
   - 更广 workload coverage
   - 更低 TTFT
   - 更少 ad hoc exclusions

如果这三点都能成立，它的论文新颖度可能比 `ResidualModeKV` 更强。

## 9. 当前结论

截至这轮调研，`TypedPrefixKV` 是一个**有价值但单独主投风险偏高**的方向。它最值得警惕的，不只是论文撞车，更是被审稿人降格成一组 feature/best-effort patch。

因此，最好的推进方式不是一上来做所有输入类型，而是：

1. 先选 `prompt embeds + multimodal hashes + cache salt` 三个最能说明问题的 typed carriers；
2. 用这三个点证明“需要统一语义层”；
3. 再决定是否扩大到更多 carrier。

如果需要更严格的“是否重复 / 是否足够支撑论文 / 是否能在 vLLM 中落地”的判断，请继续阅读同目录下的
`overlap_feasibility_assessment.md`。

## 10. 参考资料

- vLLM Automatic Prefix Caching: <https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/>
- vLLM issue #25096: <https://github.com/vllm-project/vllm/issues/25096>
- vLLM issue #20261: <https://github.com/vllm-project/vllm/issues/20261>
- vLLM issue #16016: <https://github.com/vllm-project/vllm/issues/16016>
- vLLM issue #16634: <https://github.com/vllm-project/vllm/issues/16634>
- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Serve Programs, Not Prompts: <https://yecl.org/publications/gim2025hotos.pdf>
- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
