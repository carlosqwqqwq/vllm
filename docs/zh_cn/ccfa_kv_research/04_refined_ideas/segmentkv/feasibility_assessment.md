# SegmentKV：同领域重合度与可行性评估

## 1. 这份评估要回答什么

这份文档只回答三个问题：

- 业界是否已经有与 `SegmentKV` 高度相似的论文或系统
- 我们当前的 `SegmentKV` 表述是否已经和已有工作重合
- 即便不完全重复，这个方向在当前 `vLLM` 上是否真的可做、是否真的足够支撑系统类顶会论文

结论先写在前面：

- **有，而且不止一篇。**
- **如果 `SegmentKV` 仍然表述为“RAG / Agent 的非前缀分段 KV 复用 + 少量重算”，重合度已经偏高。**
- **如果坚持“精确语义”这四个字，并把目标放在 vanilla full-attention LLM + 当前 vLLM 主线，那么这个表述本身并不稳。**

我现在的判断是：

- `SegmentKV` 作为一个观察 workload 的视角仍然有价值
- 但它已经不适合直接作为“新论文主线”的默认版本
- 它更适合被改造成一个更大系统里的子机制，或者被重新定义边界后再继续

## 2. 同领域已经有哪些工作

## 2.1 Prompt Cache：模块化 prompt 复用

`Prompt Cache`（MLSys 2024）已经明确提出：

- 很多 prompt 由可复用的模块组成
- 这些模块可以不是严格前缀
- 系统可以通过 schema 显式标记这些可复用段

它的关键词几乎就是：

- reusable text segments
- prompt modules
- modular attention reuse

这和我们最初为 `SegmentKV` 写下的“segment-aware 输入表示”已经非常接近。

需要注意的是，`Prompt Cache` 的切入点更偏：

- 明确的 prompt 模块
- schema / markup
- 低延迟推理接口

也就是说，如果我们只是说“把 prompt 划分成可复用 segment”，这个点已经被占了。

参考：

- https://proceedings.mlsys.org/paper_files/paper/2024/hash/a66caa1703fe34705a4368c3014c1966-Abstract-Conference.html

## 2.2 CacheBlend：RAG 场景的非前缀 KV 复用

`CacheBlend`（EuroSys 2025）和我们当前的 `SegmentKV` 最接近。

它已经明确做了三件事：

- 复用 **非前缀** 的预计算 KV
- 针对部分 token 做 **选择性重算**
- 把这些块在推理时 **融合** 起来，同时维持与 full prefill 接近的生成质量

这和我们原本描述的：

- non-prefix segment reuse
- boundary re-materialization
- selective re-prefill

几乎是一一对位的。

因此，如果我们的论文主张还是：

> 在 RAG 中把多个可复用 segment 的 KV 拼起来，只对少量 token 重算，从而降低 TTFT

那它会被非常自然地归类为 `CacheBlend` 这条线的延伸，而不是一个全新方向。

参考：

- https://arxiv.org/abs/2405.16444
- https://www.microsoft.com/en-us/research/publication/you-only-prefill-once-combining-cached-knowledge-for-large-language-model-serving-with-cacheblend/

## 2.3 EPIC：位置无关的 context caching

`EPIC` 把问题定义得也很接近：

- 现有 context caching 依赖 exact prefix match
- few-shot / RAG 等场景里，稳定内容前面往往带着变化前缀
- 因此需要 **position-independent caching**

这实际上就是另一种表述的“segment 可组合复用”。

对我们最不利的一点是：`EPIC` 连问题命名都很强。
如果我们的故事只是“prefix cache 不够，想让检索 chunk 不受前面 prefix 影响继续复用”，那它已经把这个问题讲得很完整了。

参考：

- https://arxiv.org/abs/2410.15332

## 2.4 Cache-Craft：chunk cache 管理与小比例重算

`Cache-Craft` 进一步说明，这条线不仅有人做，而且已经开始系统化：

- 为 RAG 的 chunk 建立独立 cache
- 判断哪些 chunk cache 可以复用
- 用小比例重算修复质量
- 再把 chunk-cache 的存储、淘汰、隐藏开销一起纳入系统设计

这意味着，若我们想把 `SegmentKV` 写成系统论文，只写“chunk 复用 + 局部重算”已经不够了，因为别人已经在讨论：

- 如何识别可复用 chunk
- 如何选重算 token
- 如何管理 chunk-cache 的存储与驱逐

参考：

- https://arxiv.org/abs/2502.15734

## 2.5 KVLink：把文档独立编码后再拼接

`KVLink` 代表的是另一类更激进的做法：

- 文档的 KV 独立预计算
- 推理时直接拼接这些 KV
- 用位置编码调整、special tokens、微调来恢复效果

它和我们不同的地方在于：

- 它不再坚持严格“原语义不动”
- 它愿意引入 special tokens 和微调

但它说明了一件很重要的事实：

- 业界已经认识到“分段独立编码后再拼接”是一个正式研究问题
- 而且已经有人愿意为这个问题引入模型层面的改造

这会直接抬高审稿人对新工作的期待。

参考：

- https://arxiv.org/abs/2502.16002

## 2.6 ProphetKV：继续沿着选择性重算往前推

`ProphetKV` 是 2026 年更近的一篇工作，它本质上在说：

- “分段 KV 复用 + 少量重算”这条路线已经成立
- 真正继续打磨的方向，是怎么更聪明地选出该重算的 token

它还直接把 `CacheBlend`、`EPIC`、`KVShare` 当成比较对象。

这意味着：

- 这一方向已经形成一个明确小领域
- 新工作的门槛已经不是“想到分段复用”
- 而是“相比前人，你到底解决了哪一个仍未解决的问题”

参考：

- https://arxiv.org/abs/2602.02579

## 3. 我们的 `SegmentKV` 和已有工作到底重不重

## 3.1 已经高度重合的部分

当前版本 `SegmentKV` 的核心卖点，和已有工作重合度最高的是下面四项：

- 把 prompt 看成多个可复用 segment，而不是单一最长前缀
- 支持非前缀上下文复用
- 通过小比例重算修复拼接后的注意力偏差
- 面向 RAG / Agent / tool-use 这种组合式上下文

对应占位大致如下：

- `Prompt Cache` 已占据“模块化 prompt / segment 输入表示”
- `CacheBlend` 已占据“非前缀 KV 复用 + 选择性重算”
- `EPIC` 已占据“位置无关上下文缓存”
- `Cache-Craft` 已占据“chunk-cache 的系统管理”
- `ProphetKV` 已占据“重算 token 选择策略继续优化”

所以如果我们不改题，只是把现有想法写得更细，审稿人非常可能会问：

- 这和 `CacheBlend / EPIC / Cache-Craft` 的差别到底是什么
- 为什么这不是把已有思路搬到 `vLLM`

## 3.2 真正还没被完全占掉的点

如果硬要从 `SegmentKV` 里找还可能成立的空间，我认为只剩三类：

- **native vLLM V1 integration**
  - 不是外置 prototype，而是把分段复用真正接到 `vLLM` 的请求表示、scheduler、connector 和 continuous batching 里
- **RAG 之外的 compositional context**
  - 不只做检索 chunk，还覆盖 agent planning、tool schema、workflow state
- **严格边界条件下的 exact reuse**
  - 不是“质量基本不掉”，而是认真回答“什么时候 segment reuse 在语义上是 exact 的”

但要注意：

- 第一类更像“工程落地”，未必足够构成独立论文主张
- 第二类更像“workload 扩展”，未必足够构成核心机制创新
- 第三类最有研究味，但也是最难成立的一类

## 4. 第一性原理检查：为什么“精确分段复用”本身就很难

这是这次调研里最重要的结论。

在 vanilla causal full-attention Transformer 里，某个 token 在 chunk `B` 中的隐藏状态，通常依赖它之前所有 token。于是：

- token 在 `B` 中的第 `l` 层 hidden state
- 由 hidden state 投影得到的 `K/V`

都会受到前置 prefix `A` 的影响。

所以当 `B` 原本是独立编码的，而现在被插入到 `A + B` 这个更长上下文中时：

- `B` 的 KV **一般不再是 exact 的**
- 问题不只是 position encoding
- 更关键的是前文对每层表示的影响已经变了

这意味着：

- 对任意非前缀 chunk，直接重用它的独立 KV，在一般情形下并不精确
- “只重算边界 token”也不能从理论上保证 exact
- 如果你想恢复严格 exact，往往需要把 chunk 内大量 token 重新算一遍，收益会迅速缩水

换句话说，`SegmentKV` 目前最危险的地方不是“实现有点难”，而是：

- **它在 vanilla full-attention 模型上的 exact 语义前提本身就不牢。**

## 4.1 哪些情况下 exact 才可能成立

只有在下面这些更强约束下，`exact segment reuse` 才比较有希望成立：

- segment 本身就是同一条请求中的真实前缀
- 模型或 attention mask 明确阻断了跨 segment 相互影响
- 通过 special tokens / 训练，让 segment 的内部表示对外部前缀近似不敏感
- 实际上把几乎整个 segment 都重算，只保留少量缓存收益

其中：

- 前两种更像特殊结构，不是一般 RAG / Agent
- 第三种已经不是纯系统改造，而会进入模型训练范畴
- 第四种会削弱系统收益

这也是为什么现有论文大多会说：

- maintain quality
- negligible accuracy loss
- close to full prefill

而不是说：

- 对任意 segment reuse 保证 exact semantics

## 5. 这个方向和当前 vLLM 到底兼不兼容

## 5.1 兼容，但不是“顺手一补丁”

从架构上说，`vLLM` 不是完全不能接这个方向。
相反，它已经出现了两个很强的信号：

- 本地代码已经有 `LMCache blending` 接口
- 社区已经有人正式提出 `generalized KV cache reuse`

所以从“生态是否认这个问题”来看，答案是认的。
但从“当前主线是否已经有原生抽象承接它”来看，答案是否定的。

## 5.2 本地代码里的直接信号

当前代码里，`LMCacheConnectorV1` 已经接入了 blending 逻辑，说明 `vLLM` 生态已经在尝试：

- 非前缀 KV 复用
- 局部融合
- 和 layerwise prefix caching 协同

但与此同时，本地 `vLLM` 的核心请求路径仍然是：

- `InputProcessor` 先把输入走向 renderer / preprocess / token stream
- `Request` 核心保存的是 `prompt_token_ids`、`block_hashes`、`kv_transfer_params`

也就是说：

- 当前主线还没有一个原生的 `segment IR`
- 没有稳定表达“这个 token 属于哪个 segment、这个 segment 可否独立复用”

所以如果真要做 native `SegmentKV`，你至少要碰：

- 输入表示
- 请求元数据
- block hash / cache hit 语义
- scheduler 对局部 prefill 的调度
- 如果做 perforated reuse，还要碰 connector API 与 attention kernel

## 5.3 GitHub 社区已经把难点说得很像了

`vLLM` 的 `Generalized KV cache reuse` issue 已经把这个方向的难点写得很直白：

- 支持任意 token 子集而非只支持完整 prefix
- perforated cache
- compositional context
- special tokens for independent segments
- scheduler / connector / attention kernel 都要改

这说明：

- 这不是一个“还没人想到”的问题
- 社区已经知道这条路的吸引力
- 也已经意识到它会深入到 kernel、connector 和 scheduler 的边界

因此，如果我们把论文写成：

> 我们让 vLLM 支持 generalized non-prefix segment reuse

审稿人很容易认为这是：

- 已知方向的系统实现
- 而不是新的核心 thesis

参考：

- https://github.com/vllm-project/vllm/issues/25672

## 6. 它是否有效

## 6.1 如果问题被放宽到“质量保持得足够好”

那答案是：**有效，而且已有多篇论文证明了它在 RAG workload 上很有收益。**

这条线常见收益包括：

- 降低 TTFT
- 降低重复 prefill 计算
- 提升吞吐
- 对共享 chunk 很重的 workload 特别有效

因此，如果你把问题改写成：

> 在 RAG / Agent 组合式上下文中，如何以质量可控的方式复用非前缀 segment 的 KV

那它是成立的。

## 6.2 如果问题坚持为“严格 exact”

那答案就要保守很多：

- 对一般 full-attention 模型，这个说法并没有被现有工作充分支撑
- 而且从第一性原理上，它本来就很难在一般情况下成立

因此我不建议继续用：

- `精确分段 KV 复用`

作为默认标题，除非你先把 exact 的适用边界定义得非常窄、非常清楚。

## 7. 它是否足够支撑系统类顶会论文

我会分两个版本给判断。

## 7.1 作为当前版本的 `SegmentKV`

如果论文主张仍然是：

- RAG / Agent 的分段 KV 复用
- 非前缀 chunk 复用
- 少量 token 重算
- 接到 vLLM

那我给它的判断是：

- **可做性：中等**
- **收益概率：中高**
- **重复风险：高**
- **OSDI 级把握：偏低**

原因不是它没收益，而是：

- 太像一个拥挤方向中的又一个变体
- novelty 容易被 `CacheBlend / EPIC / Cache-Craft / ProphetKV` 吃掉
- 如果再加上“做进 vLLM”，又容易被审成“工程实现”

## 7.2 如果强行继续做，最合理的改法

如果你仍然想保留这条线，我建议至少选下面三种重定义之一：

### 方案 A：放弃“exact”，改成质量可控的 compositional context runtime

也就是明确承认：

- 我们追求的是 **high-fidelity**
- 不是任意上下文组合下的严格 exact

这样会更诚实，也更贴近现有论文范式。

问题是：

- 这仍然会和现有工作高度竞争

### 方案 B：把 exact 只定义在“受限 segment”上

例如只针对：

- tool schema
- 固定模板
- 边界可验证的 workflow state
- attention 依赖受控的结构化上下文

这样你有机会把论文从“RAG chunk 复用”转成：

- **受限组合上下文中的 exact reuse system**

这条线更新，也更像系统与接口协同设计。

### 方案 C：把 SegmentKV 降级成更大系统里的一个子机制

这是我现在最推荐的处理方式。

例如：

- 在 `TierKV` 里把它作为 `remote/local tier` 上的 compositional reuse 模块
- 在 `AffinityKV` 里把它作为 route-aware chunk locality 的配套机制

这样它不再单独承担整篇论文的新意压力，但仍然可以贡献显著实验收益。

## 8. 我的最终判断

基于截至 2026-04-14 的公开论文、vLLM 社区讨论和本地代码状态，我对 `SegmentKV` 的结论是：

- **它不是伪问题。**
- **它不是没有效果。**
- **但它当前这个表述，已经不够新，也不够稳。**

更具体地说：

- 作为 workload 观察，它是对的
- 作为系统组件，它是有价值的
- 作为独立顶会主线，它目前风险偏高

如果必须给一个明确建议，我会写成：

> 不建议把“面向 RAG 与 Agent 的精确分段 KV 复用”按当前表述直接作为主论文方案继续推进；更稳妥的做法是，要么缩窄到受限结构上的 exact reuse，要么把它作为更大 runtime 论文中的一项关键机制。

## 9. 一页式结论

### 9.1 结论打分

- 方向价值：`7.5 / 10`
- 对 RAG workload 的有效性：`8 / 10`
- 对当前 vLLM 的兼容性：`6 / 10`
- 作为严格 exact 系统的理论稳固性：`3 / 10`
- 作为独立 OSDI 论文主线的安全性：`4 / 10`

### 9.2 最终建议

- 不把当前版本 `SegmentKV` 作为默认主线
- 保留它，作为 `TierKV / AffinityKV` 的子模块候选
- 如果必须单做，就先重定义边界，不再泛化宣称“精确分段复用”
