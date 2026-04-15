# LineageKV：同领域重合度与可行性评估

## 1. 这份评估要回答什么

这次评估的目标不是判断 `LineageKV` “听起来新不新”，而是回答三个更硬的问题：

- 业界和学界是不是已经在做“显式缓存边界”与“可缓存 prompt 语义”？
- 如果已经有人做了，`LineageKV` 和他们到底重不重？
- 更关键的一点：**`LineageKV` 当前版本在第一性原理上是否真的成立，尤其是它宣称的 exact 语义是否站得住？**

我先给结论：

- `LineageKV` **不是完全重复**，但它的可发表部分比最初想象得更窄。
- 如果它被定义成“显式区分哪些内容可缓存、哪些内容不可缓存”，那和 `Prompt Cache` 以及工业界的 prompt caching 语义已经明显接近。
- 如果它被定义成“跳过 reasoning / tool trace，再直接复用后面的答案 token”，那在普通 full-attention LLM 上**并不自动 exact**，这是当前版本最大的理论风险。
- 因此，**当前版本的 `LineageKV` 还不够稳**；真正还能保留下来的 thesis，必须收紧成“output-side reusable lineage materialization”，而不是“简单的 lineage 投影与重挂接”。

## 2. 同领域已经有哪些工作

## 2.1 Prompt Cache：显式声明哪些 prompt 模块可以复用

最接近 `LineageKV` 的学术工作其实不是 `HotPrefix` 或 `TierKV`，而是 `Prompt Cache`。

它的核心思想是：

- 通过 schema / prompt modules 明确哪些文本段是可复用的；
- 不同 prompt 派生自同一个 schema；
- 未被声明成模块的文本作为 uncached user text 处理。

这和 `LineageKV` 的一个表层想法非常接近：**“不是所有执行内容都该进入共享缓存”**。

更重要的是，`Prompt Cache` 已经把“显式 cacheability 语义”写成论文了，而且还支持参数化模块、函数调用对应的 prompt module 等场景。这意味着：

- 如果 `LineageKV` 只是要求上层应用标记 `public-reusable` / `private-hidden` / `ephemeral`；
- 然后让 serving 系统按照这些标记缓存或不缓存；

那么它和 `Prompt Cache` 的重合度会很高。

参考：

- [Prompt Cache, MLSys 2024 PDF](https://ingim.org/papers/gim2024prompt.pdf)

## 2.2 工业 prompt caching 产品：cache_control 和 cache breakpoints 已经存在

Anthropic 的官方文档已经明确支持：

- automatic caching
- explicit cache breakpoints
- 细粒度控制“哪些 block 被缓存”

文档甚至直接说，显式 breakpoint 适合“需要 fine-grained control over exactly what gets cached”的场景。

这对 `LineageKV` 很重要，因为它说明：

- “服务层语义参与 cacheability 决策”并不是空白；
- 至少在产品层，业界已经承认“缓存边界不该完全由底层自动猜”。

OpenAI 的官方文档也明确写到：

- prompt caching 依赖输入历史保持静态；
- 改动会把对应位置之后的 cache “bust” 掉。

所以，如果 `LineageKV` 只是把主题写成“给 prefix cache 增加显式控制语义”，那**不会足够新**。

参考：

- [Anthropic Prompt Caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [OpenAI Realtime costs guide](https://developers.openai.com/api/docs/guides/realtime-costs)

## 2.3 Agent / tool workload 里“排除动态内容”已经是公开实践

`Don't Break the Cache` 这篇 2026 年的评估论文专门研究 agentic workloads 下的 prompt caching，论文明确比较了：

- full-context caching
- system prompt only caching
- excluding dynamic tool results 的 caching strategy

而且它的结论是：

- 排除动态 tool results
- 避免 dynamic traditional function calling

往往比 naive full-context caching 更稳定。

这意味着：

- `LineageKV` 的一部分直觉，即“执行中会出现很多不该进入共享前缀的动态内容”，已经不是新发现；
- 把这些动态内容排除在缓存之外，也已经被至少一篇公开论文当作核心比较对象。

参考：

- [Don't Break the Cache, arXiv:2601.06007](https://arxiv.org/abs/2601.06007)

## 2.4 Thinking blocks / tool results 的特殊处理，工业界已经有明确语义

Anthropic 文档里有一段非常关键的说明：

- thinking blocks 不能显式标记 `cache_control`；
- 但在带 tool results 的后续请求里，它们会作为 request content 的一部分被缓存；
- 一旦用户添加了非 tool-result 的新内容，之前 thinking blocks 会被忽略。

这说明：

- “thinking content”和“最终公开上下文”之间的分离语义，工业界已经在产品行为中承认了；
- 但这种语义目前主要停留在 request-side prompt caching 层面，而不是 open-source serving runtime 的 output-side APC 机制里。

这既是 `LineageKV` 的风险，也是它剩下的机会。

参考：

- [Anthropic Prompt Caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

## 2.5 EPIC / CacheBlend / Segment reuse 工作：改变 prefix 不是一件简单小事

虽然这些工作不直接等于 `LineageKV`，但它们提供了一个重要反证：

- `Prompt Cache` 明确指出 attention states 的复用依赖位置；
- `EPIC` 之所以要专门做 PIC，是因为 exact prefix match 太严格，而 position-independent reuse 并不自然成立；
- `CacheBlend`、`Cache-Craft` 这类工作也都是通过额外机制处理“共享内容和执行上下文不完全一致”的问题。

换句话说，**一旦你改变了 prefix 的内容或结构，想直接复用原来的 KV，从来都不是一个可以轻描淡写带过的操作**。

这对 `LineageKV` 后面的 exact 分析至关重要。

参考：

- [EPIC, ICML 2025](https://proceedings.mlr.press/v267/hu25j.html)
- [Prompt Cache, MLSys 2024 PDF](https://ingim.org/papers/gim2024prompt.pdf)

## 2.6 社区 issue：最接近 `LineageKV` 的现实版本，今天仍然只是 bug / feature 讨论

`vLLM` 和 `SGLang` 社区已经各有一条非常像 `LineageKV` 动机的问题：

- [vLLM #39321](https://github.com/vllm-project/vllm/issues/39321)
- [SGLang #22373](https://github.com/sgl-project/sglang/issues/22373)

它们都指出：

- reasoning tokens 进入了 prefix hash / radix cache；
- 后续请求通常不会再带这些 tokens；
- 于是后面的答案 token 被“困”在一条未来不可达的链条后面。

这说明 `LineageKV` 的问题基础是真实存在的。

但也要反过来看：如果我们论文里只是把这件事扩写成“把 thinking tokens 从 hash chain 里跳过”，那**更像一个较大的 bug fix，而不是一篇系统论文**。

## 3. 我们的 `LineageKV` 和已有工作到底重不重

## 3.1 如果它只是“显式缓存哪些输入段”，重合度高

如果 `LineageKV` 被实现成：

- 上层给 prompt 段打标签；
- 某些段缓存，某些段不缓存；
- 或者给 cache 加断点和显式控制；

那么它和已有工作的关系会是：

- 和 `Prompt Cache` 高重合
- 和 Anthropic 的 explicit cache breakpoints 高重合
- 和 agentic prompt caching 的 engineering best practice 明显重合

这类版本最多只能说明：

- 开源 `vLLM` 还没有把产品级 cache-control 语义带进来

但这不足以支撑 `OSDI` 级别的新意。

## 3.2 如果它只是“跳过 reasoning tokens 再复用后续答案”，那甚至还不一定成立

这是我这轮调研里最关键的发现。

最初直觉上会觉得：

- 如果 `<think>` token 不会在下一轮出现；
- 那么把它们从 reusable lineage 里去掉就行；
- 后面的 `answer tokens` 仍然可以继续复用。

但对普通 full-attention 模型来说，这一步**并不自动 exact**。

原因非常直接：

- visible answer token 的 hidden state / KV state 是在 `Q + T + A_prefix` 的上下文下生成的；
- 如果下一轮真实输入只有 `Q + A`，没有 `T`；
- 那么 `A` 这些 token 在新上下文里的 K/V 一般会和原来不同。

也就是说，直接把：

- `Q -> T -> A`

改成可复用 lineage：

- `Q -> A`

并不是一次纯粹的“改 hash 链”，而是一次**改变前文条件的状态投影**。

这和 `SegmentKV` 遇到的问题非常像：**删掉中间执行内容后，后继 token 的状态不再天然可复用。**

这里我要明确说明：这一步是我根据 transformer 的因果依赖和现有论文/文档做出的推断，而不是某篇论文的原话。

## 3.3 所以，`descendant re-rooting` 作为当前表述，是不稳的

之前 `LineageKV` 里最像论文贡献的地方，是：

- 当某些 execution-only token 不进入 reusable lineage 时；
- 把后面的 answer token 重新挂到上一个 public ancestor 下。

但如果不额外做处理，这个“re-root”只是**逻辑 lineage 的重挂接**，并不保证对应的 KV state 真的还是 exact。

因此，当前版本的 `LineageKV` 最大的问题不是“有没有论文撞上来”，而是：

- **它自己的 exact 语义还没有被说圆。**

## 4. 第一性原理检查：当前版本到底还剩什么能成立

## 4.1 真实成立的是“执行语义”和“缓存语义”已经错位

这一点我认为是 `LineageKV` 最坚实的基础。

今天 `vLLM` 的现实是：

- API 层已经有 `reasoning_content`
- reasoning parser 已经能把 reasoning 从可见内容里分出来
- 但 request 对象仍然把所有 output token 放进 `_all_token_ids`
- `append_output_token_ids()` 之后直接 `update_block_hashes()`

这说明：

- 上层语义已经承认“有些内容只是执行细节，不是未来公开上下文”；
- 底层 APC 仍然把它们一视同仁。

这正是一个很好的系统问题：**serving API semantics 与 APC semantics 已经脱节。**

## 4.2 真正还能保留的 thesis，不是 projection，而是 materialization

如果还想保住 exact 语义，我认为 `LineageKV` 必须升级。

更合理的版本不是：

- 把 execution tokens 直接投影成 reusable tokens；

而是：

- 把 execution trace 和 public transcript 分开；
- 在请求结束后，按 public transcript **重新 materialize 一份 exact reusable lineage**；
- 这份 lineage 未来再用于 APC 命中。

也就是说，系统不再试图“直接拿旧 KV 改挂接”，而是：

- 使用已有 execution 结果作为信号；
- 再异步或延迟地构造一份真正可复用的 public-state cache。

这个版本虽然更重，但语义上才更稳。

## 4.3 这也意味着：当前版本并不轻量

如果为了保持 exact，需要额外做 public-lineage materialization，那么：

- `LineageKV` 就不再只是修改 hash 规则；
- 它会变成一个带有二次构建 / 后发布路径的运行时。

这仍然可能是一个好方向，但已经和最初“轻量地跳过 thinking tokens”差别很大。

## 5. 这个方向和当前 `vLLM` 到底兼不兼容

## 5.1 兼容性中高，但和最初预想不同

从代码结构上看，`vLLM` 其实很适合做这件事，因为：

- `reasoning_content` 已经在 API 层出现；
- reasoning parser 已经存在；
- request -> block_hashes -> kv_cache_manager 这条链很集中。

比较自然的改动点包括：

- `vllm/v1/request.py`
  - 不再把所有输出 token 机械地并入唯一 APC lineage
- `vllm/v1/core/kv_cache_utils.py`
  - 引入 public-lineage 的 hash 规则
- `vllm/v1/core/kv_cache_manager.py`
  - 增加 public lineage 的发布与命中逻辑
- `vllm/entrypoints/chat_utils.py`
  - 把 reasoning/tool metadata 继续向下传

## 5.2 但如果要 exact，可能需要额外执行路径

兼容性不代表代价低。

如果采用更稳的 `materialization` 版本，系统可能需要：

- 在响应完成后触发一个轻量的 lineage-publication pass；
- 或者在下一轮真正需要时，按 public transcript 做按需 rematerialization。

这就比普通 prefix cache patch 重得多。

## 6. 它是否有效

## 6.1 作为“发现问题”的 thesis，它显然有效

`LineageKV` 至少已经证明了一件事：

- 现代 reasoning / tool workload 下，“执行过的 token”和“未来会复用的前缀”不是同一回事。

这一点由：

- `vLLM` / `SGLang` 的 thinking token pollution issue
- Anthropic thinking blocks 语义
- agentic prompt caching evaluation

共同支持。

所以，作为**问题刻画**，它是成立的。

## 6.2 作为“当前版本的方法”，它还不够稳

但如果把方法写成：

- 双视图 token 流
- lineage projection
- descendant re-rooting

然后宣称 exact reuse，我认为目前还不够稳。

最大的原因不是实验还没做，而是**exact 性本身还没被论证清楚**。

## 6.3 一个更稳的可做版本

如果你还想继续打这条线，我建议把它改写成：

> `Public-Lineage Materialization for Exact Reuse under Hidden Execution Traces`

这个版本的核心不再是：

- “哪些 token 从 lineage 里删掉”

而是：

- “当 execution trace 含有 hidden/private/transient 内容时，系统如何构建一份未来真正可复用、并保持 exact 的 public transcript state”

这样它和简单 bug fix、prompt module、prompt cache breakpoints 的差异才会更清楚。

## 7. 它是否足够支撑系统类顶会论文

## 7.1 作为当前版本的 `LineageKV`

我不建议直接按当前版本往 `OSDI` 推。

理由有两点：

- input-side cacheability 这部分重合度已经不低；
- output-side exact lineage 这部分现在又还没彻底讲通。

这会导致论文两头受压：

- 一头被说“这不就是 prompt caching / cache_control”
- 另一头被问“你这个 exact 到底哪里 exact”

## 7.2 如果升级成 materialization thesis

如果把 thesis 改成：

- API / execution semantics 与 APC semantics mismatch
- 面向 hidden execution traces 的 exact public-lineage materialization

那它会更像一篇真正的系统问题：

- 问题足够新
- 和旧方案边界更清楚
- 还能解释 reasoning/tool workload 的新现象

但这个版本的工程复杂度和论证难度都会更高。

## 7.3 我的主观打分

按当前版本评估：

- `问题新颖性`: `8 / 10`
- `与现有工作重合风险`: `6.5 / 10`
- `vLLM 兼容性`: `8 / 10`
- `当前版本 exact 可信度`: `4 / 10`
- `当前版本冲 OSDI 把握`: `4.5 / 10`

如果升级成 `public-lineage materialization` 版本：

- `问题新颖性`: `8.5 / 10`
- `与现有工作重合风险`: `5.5 / 10`
- `工程复杂度`: `高`
- `OSDI 潜力`: `6.5 / 10`

## 8. 我的最终判断

我的最终判断是：

- `LineageKV` 指向的问题是真问题，而且比传统 QoS / hierarchy / routing 方向更有新意。
- 但**当前版本不够稳**，最大问题不是相关工作，而是 exact 语义本身存在缺口。
- 如果只做“跳过 thinking tokens / tool traces 的 hash”，那更像 feature patch。
- 如果要保留它作为论文方向，建议升级为：

> `LineageKV = 面向 hidden execution traces 的 exact public-lineage materialization runtime`

这个版本虽然更重，但：

- 更能和 `Prompt Cache`、Anthropic cache control、bug fix proposal 区分开；
- 也更像一个值得单独写论文的系统贡献。

## 9. 一页式结论

## 9.1 结论打分

- `是否重复`：**部分重复，但不完全重复**
- `当前版本是否站得住`：**不够稳**
- `问题是否真实`：**是**
- `当前版本是否足够支撑顶会`：**否**
- `升级后是否还有希望`：**有，但要改 thesis**

## 9.2 最终建议

如果继续做 `LineageKV`，我建议不要再用当前“projection + re-rooting”的表述，而要明确改成下面这个版本：

> 不直接复用被 hidden trace 污染过的后继状态；系统在响应完成后，构建一份 future-visible、exact、可共享的 public lineage state。

只有这样，这条线才同时满足：

- 对 `vLLM` 有真实增强价值
- 和已有 prompt caching / cache-control 工作足够区分
- 在系统论文里对 “exact” 这个词说得过去
