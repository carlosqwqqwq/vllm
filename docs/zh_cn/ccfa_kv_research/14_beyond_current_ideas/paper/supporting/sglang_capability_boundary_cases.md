# SGLang 中的 Capability Boundary 实例

更新日期：2026-04-15

## 1. 这份文档的作用

这份文档只负责一件事：

- 给“这是不是 `vLLM` 独有的工程问题，`SGLang` 有没有类似情况”这一 reviewer 问题提供可复用的证据说明

它不负责：

- 证明 `SGLang` 与本文问题完全同构
- 代替 `related work`
- 展开完整跨框架实验

这份文档的固定定位是：

- supporting evidence for generality
- not a primary evidence chapter

## 2. 证据来源

本文档基于本地拉取的官方仓库：

- 仓库路径：`/mnt/data3/Users/wangyang25/data/projects/sglang`
- 远端：`https://github.com/sgl-project/sglang.git`
- 当前提交：`ce31934ca80e87b7b0769e142e27daafc517e740`
- 提交时间：`2026-04-15 01:44:27 -0700`

这意味着下面的判断都是：

- 对当前 `main` 分支一个具体版本的源码与文档观察
- 不是对所有历史版本的泛化结论

## 3. 严格结论

先给严格结论：

> 当前 `SGLang` 主线实现中，确实也能观察到多类“state 理论上存在，但当前 consumer path / backend path 不允许直接消费”的 capability boundary。  
> 这些现象并不自动证明 `SGLang` 与本文中的 `vLLM` pathology 一一对应；  
> 但它们足以说明，本文讨论的问题并不是某个框架里单独一条 if-branch 的工程问题，而更像是现代 serving runtime 在 prefix reuse、异构 API、异构 carrier 与异构执行路径交汇处反复暴露出的共同结构。

## 4. Case 1：`input_embeds` 要求关闭 radix cache

关键证据在：

- [tokenizer_manager.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/managers/tokenizer_manager.py:712)

对应逻辑是：

- 如果请求携带 `input_embeds`
- 且没有设置 `--disable-radix-cache`
- `SGLang` 直接报错并要求关闭 radix cache

这说明：

- 在 `SGLang` 中，prefix/radix cache 并不是对所有输入 carrier 都自动成立
- `input_embeds` 会显式触发 carrier-side compatibility boundary

这类现象与本文的 `carrier mismatch` 非常接近。  
它支持的不是“`SGLang` 也有同一个 bug”，而是：

- producer state 的 identity 已建立
- 不代表当前 consumer path 仍可安全消费这份 state

## 5. Case 2：multimodal reranker 官方文档建议关闭 radix cache

关键证据在：

- [rerank_models.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/supported_models/retrieval_ranking/rerank_models.md:313)

官方文档明确写道：

- 对于多模态内容，为避免 caching issues，推荐使用 `--disable-radix-cache`

这说明：

- `SGLang` 官方文档层面已经承认
- 在某些 multimodal content path 下，radix cache 不是无条件安全的

这类现象与本文的问题边界也高度相似：

- 问题不在有没有 prefix index
- 而在当前 multimodal consumer contract 是否允许直接消费缓存 state

## 6. Case 3：某些 speculative / accelerated path 会拒绝 `return_logprob`

关键证据有三类。

第一类：

- [dflash_worker.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/speculative/dflash_worker.py:1106)

这里明确写道：

- 如果 DFLASH batch 请求 `return_logprob`
- 那么这是一个 scheduler 本来就应该拒绝的 invariant violation

第二类：

- [eagle_worker.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/speculative/eagle_worker.py:911)

第三类：

- [eagle_info.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/speculative/eagle_info.py:732)

这些路径都会在 speculative 内部把：

- `return_logprob = False`

这说明：

- `SGLang` 的某些高性能执行路径也不是对 observation 语义完全兼容
- 一旦 request 需要额外 observation，当前 accelerated path 就可能不再成立

这类现象与本文的 `observation mismatch` 不是一一对应，但结构相似：

- 不是已有 state 不存在
- 而是当前执行路径与 consumer observation requirement 不兼容

## 7. Case 4：deterministic / backend compatibility 与 radix cache 并非总兼容

关键证据在：

- [faq.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/references/faq.md:33)
- [deterministic_inference.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/advanced_features/deterministic_inference.md:19)

在 FAQ 中，`SGLang` 明确说明：

- prefix caching 是结果非完全 deterministic 的来源之一
- 若要获得更 deterministic 的结果，可以加 `--disable-radix-cache`

在 deterministic inference 文档中，`SGLang` 又进一步给出了 backend compatibility 表：

- `FlashInfer` 在 deterministic inference 下对 radix cache 是 `No`
- `FA3` 与 `Triton` 才是 `Yes`

这说明：

- radix cache 的可消费性并不只取决于 prefix identity
- 还取决于当前 correctness target 与 backend compatibility

这类现象与本文的 `backend mismatch` 非常相似：

- 不同 execution class 对同一份 state 的消费能力不同

## 8. Case 5：部分 scoring / sparse / mamba path 直接要求关闭 radix cache，或声明“理论可行但尚未支持”

关键证据在：

- [server_args.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/server_args.py:6548)
- [server_args.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/server_args.py:6568)
- [server_args.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/server_args.py:2234)
- [server_args.py](/mnt/data3/Users/wangyang25/data/projects/sglang/python/sglang/srt/server_args.py:2293)
- [server_arguments.md](/mnt/data3/Users/wangyang25/data/projects/sglang/docs/advanced_features/server_arguments.md:342)

这些位置分别表明：

1. `multi-item scoring` 要求 `--disable-radix-cache`
2. `hierarchical sparse attention` 当前要求 `--disable-radix-cache`
3. 某些 `mamba` 配置会直接禁用 radix cache
4. 某些 speculative + mamba 组合与 radix cache 明确不兼容
5. 文档中甚至直接写出：
   - branching point caching support is feasible but not implemented

这组证据很有价值，因为它表明：

- capability boundary 不只是 observation / multimodal 才会触发
- 在更复杂的 scheduling / sparse / branching 组合下，`SGLang` 也会显式承认“当前不支持”

这和本文最想表达的 thesis 很一致：

- “逻辑上可复用”不等于“当前路径安全可消费”

## 9. 这份证据能支持什么，不能支持什么

### 9.1 它能支持什么

它能支持下面这条更谨慎也更强的论断：

> 类似的 producer-state / consumer-path capability boundary 并非只在 `vLLM` 中出现；`SGLang` 的当前实现里也存在多类“identity 已建立，但当前 path 不能直接消费”的场景。因此，本文的问题切口不应被降级理解为某个单框架里的局部工程 patch。

### 9.2 它不能支持什么

它**不能**自动支持下面这些更强的 claim：

- `SGLang` 与本文的 `vLLM` pathology 完全同构
- `SGLang` 中也已经存在与 `prompt_logprobs selective replay` 一模一样的案例
- 本文已经在跨框架层面完成统一实验验证

这些都超出了当前证据边界。

## 10. 最稳的用法

这份 supporting 文档最稳的用途有两个：

1. 在 related work / discussion 中用一小段文字说明：
   - 类似 capability boundary 也出现在 `SGLang`
2. 在 rebuttal 中回答：
   - 这不是 `vLLM` 独有的工程问题

它不适合被用来：

- 替代主证据
- 在摘要中大幅外推

## 11. 一句可复用总结

如果只保留一句最该复用的话，这句应该固定为：

> `SGLang` 当前实现中也存在多类“state identity 已建立，但当前 consumer path / backend path 不能直接消费”的 capability boundary；这说明本文讨论的问题并不是单框架里的局部工程 patch，而更像是现代 serving runtime 的共同结构。
