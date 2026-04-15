# FrontierKV：重合度与可行性评估

## 1. 先给结论

这轮调研后，我对 `FrontierKV` 的判断比前一版更收敛，也更保守：

- `FrontierKV` 抓到的**确实是真问题**。
  - 当前 exact APC 的复用效率，确实会被固定 block size、full-block-only 共享、block-aligned 重算这些约束卡住。
- 但按当前表述，`FrontierKV` **已经不够“没人做过”**。
  - 我没有找到一篇和它**完全相同**的论文，但已经找到多条非常接近的工作线，尤其是 `TensorRT-LLM` 的 `Partial Reuse`、`MEPIC` 的 page-aligned clean/dirty boundary、`ContiguousKV` 的 granularity-aligned 设计。
- 所以，如果目标是“确保没人做过”并且冲 `OSDI/CCF-A`，我**不建议把当前版本的 FrontierKV 直接当主线**。
  - 更准确的说法是：它有真实价值，也能在 `vLLM` 里做出来，但**现在的 thesis 不够新，也不够大**。

一句话结论：

> `FrontierKV` 当前版本更像一条“值得做的 runtime 优化线”，还不像一条“足够稳、足够新、足够大”的 `OSDI` 主线。

## 2. FrontierKV 到底想解决什么

`FrontierKV` 的核心观察并没有错：

- `vLLM` 的 prefix caching 只缓存 full blocks；
- cache hit 以 block 为单位；
- 即使只差一个 token，也可能因为 block 对齐而多重算一整块；
- 大 block 提高 kernel/metadata 效率，但会降低 hit granularity；
- 小 block 提高 hit granularity，但会增加 metadata 和运行时管理成本。

所以，这条线真正盯住的是：

> exact APC 的性能上限，并不只由“有没有 prefix cache”决定，还由“prefix cache 的块几何形态”决定。

这个命题本身成立，但问题在于：**业界已经开始围绕同一个命题发力了**。

## 3. 目前最接近的论文与系统

我把最接近的相关工作分成三组。

### 3.1 exact prefix caching 系统：块大小本身已经被承认为 tradeoff

代表：

- [vLLM Prefix Caching 设计文档](https://docs.vllm.ai/en/latest/design/prefix_caching/)
- [TensorRT-LLM KV Cache Reuse](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)
- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)

这些工作的共同点是：

- 都承认 exact prefix reuse 以 block/chunk/page 为基本单位；
- 都承认块大小影响命中率和运行时开销；
- 都围绕 prefix tree / radix tree / paged block 来做共享。

其中最危险的是 `TensorRT-LLM` 的最新文档。它已经明确写了两件事：

1. `tokens_per_block` 是命中率和效率之间的 tradeoff；
2. 系统支持 `Partial Reuse`，当“只有部分 token 命中”时，可以复制匹配 token 到新块里继续复用。

这意味着：**“块边界会造成 false miss，因此需要更细的边界复用”这个洞见，工业界已经显式实现了。**

### 3.2 非前缀 chunk/PIC 系统：page alignment 与 clean/dirty boundary 已经被讲过

代表：

- [EPIC, ICML 2025](https://proceedings.mlr.press/v267/hu25j.html)
- [Cache-Craft, SIGMOD 2025](https://arxiv.org/abs/2502.15734)
- [ContiguousKV, arXiv 2026](https://ar5iv.labs.arxiv.org/html/2601.13631v1)
- [MEPIC, arXiv 2025](https://ar5iv.labs.arxiv.org/html/2512.16822v1)

这些工作不是 exact APC，而是 PIC / chunk reuse / re-prefill，但它们已经覆盖了几个对 FrontierKV 很危险的点：

- `ContiguousKV`
  - 把“算法语义的粒度”和“系统 I/O/缓存粒度”不一致，明确上升为系统瓶颈；
  - 核心贡献就是 granularity alignment。
- `MEPIC`
  - 直接讲 `page-aligned canonical chunk placement`；
  - 直接讲 `dirty block boundary`；
  - 甚至直接讲“把 selective recomputation 从 token-level 移到 block-level，只让第一块变脏，后面的块保持可共享”。

这和 `FrontierKV` 的边界主张已经高度邻近了。虽然任务场景不同：

- `MEPIC` 解决的是 PIC；
- `FrontierKV` 想解决的是 exact APC；

但如果 `FrontierKV` 仍然只说“边界附近需要更细粒度、稳定后再粗化”，评审很容易认为你只是把 `MEPIC/ContiguousKV` 的 granularity logic 迁移到 exact APC。

### 3.3 prefix-aware attention / kernel 路线：前缀边界已经不只在 scheduler 侧被优化

代表：

- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)
- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)

这条线说明一件事：

> prefix reuse 的瓶颈不再只被认为是“有没有 cache 命中”，而已经被进一步下探到 kernel execution 和 tile packing。

这会削弱 `FrontierKV` 的“高度”。因为如果你只在 cache manager 层做块粒度调整，而不能解释为什么它优于 kernel-side / tree-side / partial-reuse-side 的既有改进，论文主张会显得偏窄。

## 4. 和我们 idea 的真实重合度

我把重合度分成三档。

| 工作 | 相关性 | 为什么像 | 为什么还不完全一样 |
| --- | --- | --- | --- |
| TensorRT-LLM Partial Reuse | 很高 | 都在处理 partial match / block boundary / copy-based reuse | 目前公开的是工业文档，不是完整论文；也没有看到“frontier-only 细化 + 稳定后 coalesce”的完整 thesis |
| MEPIC | 很高 | page-aligned layout、clean/dirty boundary、block-level recompute、与 paged cache 集成 | 目标是 PIC，不是 exact APC |
| ContiguousKV | 高 | 直接把 granularity mismatch 上升成系统问题 | 主要场景是 offloaded re-prefill，不是 GPU-resident exact APC |
| ChunkAttention | 中高 | chunk 化、prefix tree、prefix-aware runtime | chunk size 是固定的，重点更偏 kernel 与 prefix tree |
| EPIC / Cache-Craft | 中 | 都在讨论 reuse granularity 和 selective recomputation | 解决的是 non-prefix reuse，不是 exact APC |
| vLLM / TensorRT 基线 prefix cache | 中 | 都承认 full-block-only 共享和 block size tradeoff | 还没有形成 frontier-local adaptive geometry 的论文级表述 |

我的最终判断是：

- **没有找到一篇与 FrontierKV 完全同构的论文**；
- 但也**绝对不能再说“这个方向没人做”**；
- 更准确的说法应该是：
  - `FrontierKV` 的“完整组合”我暂时没找到现成论文；
  - 但它的关键部件和核心洞见，已经被不同工作分别覆盖了。

因此，如果你的要求是“确保没有人做过”，那么以目前证据，我**无法给出这个保证**。

## 5. vLLM 当前实现给了我们什么，也拿走了什么

这是决定可行性的关键。

### 5.1 证明这个问题真实存在的本地证据

本地 `vLLM` 代码里，已经能直接看到 FrontierKV 想攻击的瓶颈：

- `docs/design/prefix_caching.md` 明确写了：`We only cache full blocks.`
- `vllm/v1/core/kv_cache_manager.py` 明确写了：computed blocks 必须是 full blocks。
- 同一个文件还明确写了：当所有 token 都 hit cache 时，仍然需要为了 logits 重算最后一个 token，而这个过程**可能因为 block-size 对齐重算整块**。

这说明：

- fixed block geometry 的确在 exact APC 中引入了额外重算；
- 这不是拍脑袋出来的问题，而是当前主线实现里的真实限制。

### 5.2 证明 vLLM 已经不再是“纯单粒度系统”的本地证据

但与此同时，本地代码也表明：

- `block_pool.py` 已经支持 `hash_block_size` 与实际 `block_size` 不同；
- `kv_cache_utils.py` 已经支持从更细 hash granularity 向更粗 block granularity 做 lazy conversion；
- `gpu_model_runner.py` 和 `block_table.py` 已经支持 `kernel_block_size` 小于 `kv_manager block_size`，也就是**分配粒度和执行粒度已经开始解耦**；
- `kv_cache_manager.py` 甚至直接写了，未来如果不同 group 有不同 block size，当前抽象会被打破。

这几件事合起来意味着：

> vLLM 并不是完全没有多粒度基础设施。相反，它已经有一部分“hash 粒度 / 逻辑块粒度 / kernel 粒度”解耦的雏形。

这既是机会，也是坏消息。

机会在于：

- 我们不是从零开始；
- 原型能做出来。

坏消息在于：

- “我们提出多粒度 block”本身，不再是新点；
- 评审很可能会问：这和 vLLM 已有的 `hash_block_size + kernel_block_size` 到底差在哪？

## 6. 这个 idea 在 vLLM 里能做吗

结论是：**能做，但不是低成本 patch。**

### 6.1 为什么说能做

因为它已经具备三块必要基础：

1. `hash_block_size`
   - 说明 block hash 的计算粒度已经不是硬编码死的。
2. `block_size != kernel_block_size`
   - 说明执行粒度和管理粒度已经允许不同。
3. `coordinator / manager / block_table` 已经分层
   - 说明你有机会只动 cache metadata 和 block mapping，而不是完全重写 attention kernel。

### 6.2 真正要补的能力

如果真的做 `FrontierKV`，至少需要补下面几件事：

- **frontier-local logical subblocking**
  - 不是全局缩小 block，而是只在复用边界附近暴露更细的逻辑子块。
- **bidirectional reblocking**
  - 当前本地实现主要支持从细 hash 向粗 block 合并；
  - 但 `FrontierKV` 需要真正的在线 split / coalesce，而不只是静态 scale-up。
- **frontier-aware publication**
  - 不是所有计算出来的 block 都按同样规则进入可复用状态；
  - 边界区和稳定 body 需要不同的“发布粒度”。
- **mixed-granularity lookup and eviction**
  - 命中查找、引用计数、free queue、eviction policy 都要理解 body/frontier 混合状态。

### 6.3 最大的实现风险

最大风险不在 kernel，而在 metadata correctness：

- partial reuse 之后 block 的哈希和父链怎么定义；
- split / coalesce 后如何保持 refcount 正确；
- mixed-granularity block 如何与 free queue / eviction 协同；
- 如何避免把命中提升建立在大量 copy 上，结果吞掉收益。

所以，`FrontierKV` 是“可做”，但不是“轻改几处配置就能发论文”。

## 7. 这个 idea 会有效吗

我的判断是：**会有效，但收益窗口没有想象中大，而且 workload 依赖明显。**

### 7.1 最可能受益的场景

- 多个请求共享大部分前缀，但边界经常落在 block 内部；
- 多轮对话每轮只追加少量 token；
- 模板前缀稳定，但末尾 few-shot / metadata / user delta 经常只差几十个 token；
- 长 prompt 下，block-aligned extra recompute 对 TTFT 变得可见。

在这些场景里，`FrontierKV` 能减少：

- coarse block 带来的 false miss；
- “只差一个 token 却重算一整块”的额外代价；
- 把全局 block size 调得很小才能换命中率的尴尬。

### 7.2 收益不会很大的场景

- 请求本来就大多是 full-block exact hit；
- prompt 很短；
- decode 比 TTFT 更主导；
- workload 的重用不是 exact prefix，而是 RAG / PIC 类型复用。

在这些场景里，`FrontierKV` 的边际收益可能不够大，甚至容易被 metadata/copy 开销抵掉。

### 7.3 我对效果的总体判断

如果原型做得好，这条线大概率能拿到：

- 一部分 `TTFT` 改善；
- 一部分 prefix hit granularity 改善；
- 某些 prefix-heavy 场景下更稳定的吞吐。

但我不认为它会像 `TierKV` 或 `SessionKV` 那样自然打出一个“全局系统级数量级提升”的故事。

## 8. 它能领先最新论文吗

如果“最新论文”指 **peer-reviewed 论文**，我的判断是：

- **不容易稳稳领先**。

原因不是因为它没价值，而是：

- `MEPIC` 和 `ContiguousKV` 已经把 granularity/alignment 讲得很深；
- `ChunkAttention` 和 `PAT` 已经把 prefix-aware 优化推进到 kernel 层；
- `TensorRT-LLM` 已经在工业系统里公开了 `Partial Reuse`。

如果“领先”指的是“提出一个更聚焦 exact APC、并能在 vLLM 主线里更自然落地的版本”，那还有空间；但这个空间已经很窄了。

因此，我的真实判断是：

- `FrontierKV` 可以做到**和现有工作错位**；
- 但很难做到**明显领先所有最新工作**；
- 更无法负责任地说“确保没有人做过”。

## 9. 它够不够支撑 OSDI / CCF-A

以当前版本，我给出的判断是：**不够稳。**

原因有三个：

1. **新意不够干净**
   - 核心洞见已经被 `TensorRT-LLM Partial Reuse`、`MEPIC`、`ContiguousKV` 部分覆盖。
2. **thesis 不够大**
   - 现在更像一个 runtime geometry optimization，不像一个新的 serving abstraction。
3. **收益窗口有限**
   - 它更容易成为“一个很聪明的优化”，而不是“一个改变系统设计边界的机制”。

如果目标是 `ATC / EuroSys` 风格论文，这条线还有救；如果目标是把它作为目前最核心的 `OSDI` 主线，我不推荐。

## 10. 最后判断

我的最终建议非常明确：

- **不要再把当前版本的 FrontierKV 当作默认主线。**
- 它最适合的角色是：
  - 作为更大系统里的一个子机制；
  - 或者作为 `vLLM` runtime 优化论文的一个核心组件；
  - 但不适合作为“确保没被做过、还能稳冲 OSDI”的主 thesis。

如果一定要保住这条线，唯一还有希望的升级方向是：

> 不再把论文主张写成“边界自适应块粒度”，而改写成“exact APC 中 publication granularity、storage granularity、execution granularity 三者解耦的运行时原理”。

只有把 thesis 抬到这个层次，它才有机会重新拉开和 `Partial Reuse / MEPIC / ContiguousKV` 的差距。

## 11. 我这轮用到的关键来源

- [vLLM Prefix Caching 设计文档](https://docs.vllm.ai/en/latest/design/prefix_caching/)
- [TensorRT-LLM KV Cache Reuse](https://nvidia.github.io/TensorRT-LLM/advanced/kv-cache-reuse.html)
- [TensorRT-LLM KV Cache System](https://nvidia.github.io/TensorRT-LLM/latest/features/kvcache.html)
- [ChunkAttention, ACL 2024](https://aclanthology.org/2024.acl-long.623/)
- [EPIC, ICML 2025](https://proceedings.mlr.press/v267/hu25j.html)
- [Cache-Craft, SIGMOD 2025](https://arxiv.org/abs/2502.15734)
- [ContiguousKV, arXiv 2026](https://ar5iv.labs.arxiv.org/html/2601.13631v1)
- [MEPIC, arXiv 2025](https://ar5iv.labs.arxiv.org/html/2512.16822v1)
- [PAT, ASPLOS 2026](https://arxiv.org/abs/2511.22333)
