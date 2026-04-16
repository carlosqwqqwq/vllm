# WritePathKV 深度复审：重复性、可行性与投稿潜力评估

更新日期：2026-04-15

## 1. 严格结论

先给结论，不绕弯：

- 截至 `2026-04-15`，我**不能诚实地说“确保没人做过”**。
- 基于这轮公开资料检索，我**没有找到一篇与 refined `WritePathKV` 完全同构的公开系统论文**。
- 但我同样不能把它继续当成“高创新、低重合”的安全方向。

更准确地说：

- 如果把 `WritePathKV` 写成“更好的 KV admission / offload / export policy”，它已经**明显进入高重合区**。
- 如果把它写成“减少 write amplification 的多级 KV 管理”，也会**明显靠近** `HotPrefix / Marconi / CachedAttention / LMCache / TensorRT-LLM` 一线。
- 如果把它收敛成“`vLLM` 统一运行时里的跨层 state publication semantics”，它**仍有一定空间**，但创新性已经没有初版判断里那么稳。

所以当前最诚实的结论是：

> 原始版 `WritePathKV` 不适合作为“几乎没人做过”的安全主线。
> 它最多只能在**重写 thesis**后，作为一个“可实现、可能有效、但论文创新性需要再证明”的候选方向。

## 2. 这轮检索到的高风险近邻

### 2.1 `HotPrefix`：最危险的直接近邻

资料：

- [HotPrefix: Hotness-Aware KV Cache Scheduling for Efficient Prefix Sharing in LLM Inference Systems](https://cs.nju.edu.cn/tianchen/lunwen/2026/sigmod26-liyuhang.pdf)

它做了什么：

- 动态追踪 prefix hotness；
- 对从 GPU 驱逐出去的 KV 做 **Selective KV Cache Admission**；
- 只把高价值 prefix 留在 CPU memory；
- 再把高 hotness prefix 从 CPU 提升回 GPU。

为什么危险：

- 它已经明确占住了“**GPU/CPU 双层 KV + selective admission + promotion**”这条叙事；
- 它不是一般性的 eviction 调优，而是系统化地讨论了：
  - 哪些 KV 值得进入 host memory；
  - 哪些 KV 直接丢弃；
  - 哪些 KV 值得被重新提升；
- 这与原始版 `WritePathKV` 里“哪些 KV 值得发布/导出/offload”高度相近。

严格判断：

- **如果我们的故事还停留在 GPU/CPU 两层 admission 与 write-path 价值判断，重合风险很高。**

### 2.2 `Marconi`：generic admission/evaluation 问题已经有人讲了

资料：

- [Marconi: Prefix Caching Admission and Eviction for Hybrid Models Serving](https://proceedings.mlsys.org/paper_files/paper/2025/file/d0f45180142e0fd6dd03123a6ec7651e-Paper-Conference.pdf)

它做了什么：

- 把 prefix cache entry 的 admission 和 eviction 变成显式问题；
- 用 reuse likelihood、compute saving 与 memory footprint 做决策；
- 目标是提高 end-to-end latency 与 throughput。

为什么危险：

- 它已经把“**cache entry 值不值得留**”这个 generic 问题讲成了完整系统论文；
- 如果我们只是在说：
  - 未来复用价值减去写成本；
  - 低价值 entry 不值得写出去；
  - 需要更智能的 admission；
  审稿人很容易问：这和 `Marconi` 的本质差别是什么？

严格判断：

- **如果 `WritePathKV` 的主要贡献只是一个 value-aware admission policy，它不够新。**

### 2.3 `TensorRT-LLM`：工业界已经把写出控制和事件暴露做出来了

资料：

- [Introducing New KV Cache Reuse Optimizations in NVIDIA TensorRT-LLM](https://developer.nvidia.com/blog/introducing-new-kv-cache-reuse-optimizations-in-nvidia-tensorrt-llm/)

它做了什么：

- 提供 priority-based eviction；
- 提供 KV cache 事件接口；
- 当 block 被存储、删除或更新时发出事件；
- 这些事件还能支撑 KV-aware routing 和 scheduling。

为什么重要：

- 这说明工业系统已经不再把 KV cache 看成“黑箱里的自动副作用”；
- 它们已经在暴露：
  - 写入事件；
  - 生命周期事件；
  - 外部调度可见性。

严格判断：

- **如果我们只是强调“把写路径变成显式一等对象”，这个卖点本身已经不够。**

### 2.4 `CachedAttention`：多级层次化 KV 与异步保存并不是空白

资料：

- [CachedAttention: Efficient Memory Management for Multi-Turn Inference of LLMs](https://www.usenix.org/system/files/atc24-wang-jiahao.pdf)

它做了什么：

- 层次化管理多轮会话 KV；
- 支持异步保存与按需取回；
- 让 scheduler 与 fetch / eviction 协同。

为什么危险：

- 它已经说明“**多级 KV、异步 save、调度协同**”不是新命题；
- 如果 `WritePathKV` 只是把“写出去”和“以后再取回来”重新包一层叙事，仍然会显得太近。

### 2.5 `LMCache`：多层 KV 已经被工程化为独立 substrate

资料：

- [LMCache Tech Report](https://arxiv.org/abs/2510.09665)
- [LMCache Architecture](https://docs.lmcache.ai/getting_started/architecture.html)

它做了什么：

- 把 KV 视为独立的多层缓存层；
- 提供 locate、move、pin、compress 等控制能力；
- 支持从 CPU 到 disk / remote 的异步写入。

为什么重要：

- 这意味着“KV 是需要专门管理的数据对象”这层抽象，工业界和社区已经接受了；
- `WritePathKV` 不能再靠“把 KV 写路径显式化”本身立题。

### 2.6 `SafeKV`：public/private visibility 这条线也不是空白

资料：

- [SafeKV: Safe KV-Cache Sharing in LLM Serving](https://openreview.net/pdf?id=jhDsbd5eXL)

它做了什么：

- 讨论安全的 KV 共享；
- 区分可共享与不可共享状态；
- 在异构层级里维护共享与隔离语义。

为什么重要：

- 原始版 `WritePathKV` 里那些 `LOCAL_ONLY / EXPORTABLE / DROP_AFTER_USE` 分类，
  已经不再是“完全没被碰过”的空间；
- 即便 `SafeKV` 的主目标是安全，不是吞吐，它仍然说明**state visibility class** 这类语言已经被占用了一部分。

### 2.7 `BanaServe` 与 `ContiguousKV`：不是直接重复，但会继续压缩我们的空间

资料：

- [BanaServe: Efficient Serving of Disaggregated LLMs via Cache-Aware Global Scheduling](https://arxiv.org/abs/2510.13223)
- [ContiguousKV: Accelerating LLM Prefill with Granularity-Aligned KV Cache Management](https://arxiv.org/abs/2601.13631)

它们分别强调：

- `BanaServe`：disaggregated serving、global store、cache-aware global scheduling；
- `ContiguousKV`：persistent prefix KV offloading、I/O granularity、re-prefill。

它们为什么重要：

- 它们虽然不直接讲“publication at birth”，但都在不断压缩“多层 KV write/read management”这块剩余空间；
- 这要求我们必须把 thesis 讲得比“更好的 offload / export”更窄、更硬。

## 3. 哪些表述现在已经不安全

经过这轮调研，下面这些说法我都不建议再用：

- “`WritePathKV` 是第一个研究 KV write path 的系统”
- “我们首次提出 KV publication policy”
- “我们首次把 KV 写路径变成一等对象”
- “我们首次解决多级 KV 的写放大问题”
- “我们首次决定哪些 KV 应该 offload/export/store”
- “我们提供第一个多层 KV publication runtime”

这些写法要么已经与近邻工作直接重叠，要么非常容易被审稿人一句话击穿。

## 4. 还剩下什么相对可防守的 refined thesis

如果还要继续保留这条线，我建议只能保留下面这个更窄的版本：

> 在 `vLLM` 这类同时具备 local APC、CPU offload、KV connector 与 P/D disaggregation 的统一运行时里，新生成 KV 从“当前请求私有执行状态”过渡到“跨层可复用状态”的过程，今天仍然主要由 eager caching、eviction-time offload 和 connector hooks 隐式触发。我们研究的不是一般性的 admission/eviction，而是**state externalization / publication 的时机、可见性和层级边界**。

这一定义故意强调四件事：

- `birth-time or finalize-time publication`，不是单纯 eviction-time 保留；
- `visibility transition`，不是只做 cache replacement；
- `cross-layer externalization`，不是单一 GPU/CPU 二层策略；
- `existing vLLM runtime`，不是另起炉灶做一个全新 serving substrate。

### 4.1 这个 refined 版本和 `HotPrefix` 的边界

可以尝试这样区分：

- `HotPrefix`
  - 主要研究 GPU 与 host 之间的 hotness-aware admission / promotion；
  - 核心触发点更接近 eviction / promotion。
- refined `WritePathKV`
  - 研究的是一个状态从“刚被 finalize”到“是否该 externalize”的生命周期；
  - 重点是 local / offload / connector / disagg 的统一 publication boundary。

但我要强调：

- **这个边界不是天然牢固的**；
- 它必须靠非常强的 problem characterization 和 implementation evidence 才能站住。

### 4.2 这个 refined 版本和 `Marconi` 的边界

可以尝试这样区分：

- `Marconi`
  - 主要是 value-aware cache admission / eviction。
- refined `WritePathKV`
  - 不只是问“值不值得保留”；
  - 而是问“状态何时从 private 变成 cross-tier visible / exportable”。

但同样要强调：

- **如果最后实现出来只是一个分数函数去决定 store / skip-store，这个边界会立刻塌掉。**

## 5. vLLM 里为什么它仍然可实现

尽管论文空间收窄了，但实现空间是存在的，而且很真实。

### 5.1 本地 finalize/cache path 已经是显式逻辑

`vllm/v1/core/kv_cache_manager.py`

这里已经有：

- `get_computed_blocks()`：区分本地 prefix hit；
- `allocate_slots(..., num_external_computed_tokens=..., delay_cache_blocks=...)`；
- 只对 finalized tokens 执行 `cache_blocks(...)`。

这说明：

- `vLLM` 已经承认“什么时候 cache、哪些 token 可 commit”是运行时问题；
- 不是所有 KV 都必须立刻进入可复用状态。

### 5.2 CPU offload 写路径是显式 store pipeline

`vllm/v1/simple_kv_offload/manager.py`

这里已经有：

- `StoreRequestState`
- `update_state_after_alloc(...)`
- `prepare_store_specs(...)`
- `build_connector_meta(...)`
- `store_event / load_event`

这说明：

- CPU offload 不是一个不可控的后台副作用；
- 它本身已经有事件、元数据和显式写出流程。

### 5.3 connector/export 路径也已经有生命周期钩子

`vllm/v1/worker/kv_connector_model_runner_mixin.py`

这里已经有：

- `bind_connector_metadata(...)`
- `start_load_kv(...)`
- `wait_for_save()`
- `clear_connector_metadata()`
- `get_kv_connector_stats()` 和 `get_kv_connector_kv_cache_events()`

这说明：

- connector/export 已经具备“先绑定元数据，再开始 transfer，再等待 save 完成”的生命周期；
- publication semantics 不是凭空假设。

### 5.4 scheduler 已经在统一 local/external 命中与 async load

`vllm/v1/core/sched/scheduler.py`

这里已经统一处理：

- local prefix hit；
- connector external hit；
- `load_kv_async`；
- `delay_cache_blocks`；
- `num_external_computed_tokens`。

这意味着：

- 如果要做第一版原型，最佳落点不是 attention kernel；
- 而是 scheduler + kv manager + offload/connector manager。

## 6. 从实现角度，最保守也最现实的原型是什么

如果真的要做，我建议不要第一步就碰 local APC 全局语义，而是按下面三阶段推进。

### 6.1 P0：只做 instrumentation，不做 policy

先回答四个问题：

1. 今天到底写出了多少后来根本没复用的 block？
2. CPU offload 与 connector export 的字节流量里，有多少是“低价值写流量”？
3. publication latency、reuse lag、write amplification 分别是多少？
4. 这些问题是否会真实拖慢 TTFT、throughput 或尾延迟？

如果这一步没有观察到强 pathology，整条线就不值得继续。

### 6.2 P1：只 gate external publication，不动 local APC

第一版最稳的做法是：

- 本地 APC 仍照旧缓存 finalized blocks；
- 只对外层写路径做 gating：
  - CPU store
  - connector export
  - disagg transfer publication

这样做的好处是：

- 改动小；
- 风险低；
- 更容易把贡献讲成“cross-tier externalization policy”，而不是“又改了一次 prefix cache”。

### 6.3 P2：引入显式 publication state

如果 P1 真的有效，再升级为显式状态机，例如：

- `PRIVATE_LOCAL`
- `LOCAL_REUSABLE`
- `EXPORT_DEFERRED`
- `EXPORT_ELIGIBLE`
- `EXPORTED`
- `DROP_AFTER_USE`

但这里要非常克制：

- 这个状态机不是为了把系统复杂化；
- 而是为了让论文的抽象层高于简单 heuristic。

## 7. 这条线要成立，实验上必须证明什么

如果后续真做实验，最关键的证据不是“cache hit 提高了多少”，而是下面这些问题。

### 7.1 新的 problem characterization 必须成立

必须证明：

- 在 `vLLM` 的多层 KV 路径里，存在显著的 `cross-tier write amplification`；
- 相当一部分 external writes 从未产生复用价值；
- 这些无效写流量会显著消耗：
  - PCIe / NVLink / network bandwidth；
  - CPU memory bandwidth；
  - shared cache capacity；
  - connector/save pipeline 时间。

### 7.2 简单 admission baseline 不够

必须证明：

- 单纯 LRU / priority / hotness / reuse-score 不足以解释 publication timing；
- 真正关键的是“何时 externalize，而不是只看要不要保留”。

否则评审会直接说：

- 这不就是另一个 `HotPrefix / Marconi` 变体吗？

### 7.3 优化收益必须覆盖真实主路径

必须至少看到下面几类收益中的两类以上：

- external write bytes 明显下降；
- useless-write ratio 明显下降；
- TTFT 改善；
- throughput 改善；
- tail latency 改善；
- shared-tier pollution 下降。

## 8. 目前我对论文价值的判断

我把“论文上值不值得押注”和“工程上能不能做”分开讲。

### 8.1 工程落地性

- **高**

原因：

- `vLLM` 已经有现成写路径钩子；
- 第一版可以只改 scheduler / kv manager / offload/connector metadata；
- 不需要碰 attention kernel，也不需要改模型本身。

### 8.2 论文创新性

- **原始版：中低**
- **refined 版：中等，最多中高**

原因：

- `HotPrefix` 把 selective admission 讲得很完整；
- `Marconi` 把 admission/evaluation 讲得很直接；
- `TensorRT-LLM` 把工业界对 KV 事件与生命周期的显式控制提前暴露出来了；
- `LMCache` 和 `CachedAttention` 把多层管理 substrate 做得不再陌生。

### 8.3 OSDI/顶会安全性

- **当前不够安全**

这句话我必须讲清楚：

- `WritePathKV` 不是“完全不能发”；
- 但它已经不是那种“只要做出来就明显新”的方向；
- 如果要冲 `OSDI` 这类顶会，它现在最缺的不是实现抓手，而是**一个足够尖锐、足够可防守的新 pathology**。

## 9. 当前建议：go / no-go

我的建议很明确：

- **原始版 `WritePathKV`：No-Go**
- **refined `WritePathKV`：仅在做出强 characterization 之后再决定是否继续**

更具体一点：

1. 不要再把它写成 generic write-path optimization。
2. 如果要继续，只做 `P0 instrumentation`，先验证 multi-tier publication mismatch 是否真的大到值得发论文。
3. 如果 `P0` 结果不够尖锐，这条线就应该尽快放弃，不要继续堆 policy。

## 10. 参考资料

- HotPrefix: <https://cs.nju.edu.cn/tianchen/lunwen/2026/sigmod26-liyuhang.pdf>
- Marconi: <https://proceedings.mlsys.org/paper_files/paper/2025/file/d0f45180142e0fd6dd03123a6ec7651e-Paper-Conference.pdf>
- TensorRT-LLM KV Reuse Optimizations: <https://developer.nvidia.com/blog/introducing-new-kv-cache-reuse-optimizations-in-nvidia-tensorrt-llm/>
- CachedAttention: <https://www.usenix.org/system/files/atc24-wang-jiahao.pdf>
- LMCache Tech Report: <https://arxiv.org/abs/2510.09665>
- LMCache Architecture: <https://docs.lmcache.ai/getting_started/architecture.html>
- SafeKV: <https://openreview.net/pdf?id=jhDsbd5eXL>
- BanaServe: <https://arxiv.org/abs/2510.13223>
- ContiguousKV: <https://arxiv.org/abs/2601.13631>
