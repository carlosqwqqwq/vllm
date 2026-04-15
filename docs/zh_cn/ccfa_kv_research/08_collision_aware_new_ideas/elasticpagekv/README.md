# ElasticPageKV：逻辑微块与物理大页解耦的精确 KV Cache

## 1. 一句话 thesis

`ElasticPageKV` 的核心 thesis 是：**vLLM 当前把 prefix 命中粒度、hash 粒度、物理 KV page 粒度、attention kernel 粒度和 KV transfer 粒度绑定得过紧；下一代 KV cache 应该把逻辑复用粒度与物理执行粒度解耦，在保持 exact semantics 的同时减少尾部重算、页内浪费和 block-size 选择两难。**

这不是简单的 “partial block caching”。如果只写成 partial block，重合风险非常高。它必须被定义为：

- `logical micro-page`：用于精确 hash、精确命中、细粒度复用和尾部状态表达。
- `physical macro-page`：用于 GPU KV 分配、block table、attention kernel、DMA/offload transfer。
- `mapping layer`：负责从 micro-page hit 翻译到 macro-page offset、mask、tail recompute 和 promotion。

## 2. vLLM 当前证据

vLLM 当前 prefix cache 的基础假设仍然偏向 “full block first”：

- `vllm/v1/core/kv_cache_manager.py` 的 `get_computed_blocks()` 明确写着 computed blocks must be full，并且全命中时仍要重算最后 token；注释还说明 block-size alignment 可能导致整块重算。
- `vllm/v1/core/kv_cache_utils.py` 的 request block hasher 在不足一个完整 block 时 early stop，并注释 “We only hash full blocks”。
- `vllm/v1/core/block_pool.py` 把 cached block 定义为 full block；同时为了 block table append-only，当前不去重已经缓存的相同 block。
- vLLM 官方 prefix caching 文档也把 KV blocks 的 hash 作为 cache key，重点是 full block 级别的自动前缀缓存。

这说明一个基本事实：**APC 的复用语义和物理 block/page 语义仍然强绑定**。当 block size 为 16 时，这个损失可能只是若干 token；但如果为了 kernel、transfer、offload 或 hybrid KV 选择更大的 page，逻辑复用粒度也被迫变粗，问题会被放大。

## 3. 最近邻工作与重合分析

| 工作线 | 代表工作 | 做了什么 | 与 ElasticPageKV 的关系 | 重合风险 |
| --- | --- | --- | --- | --- |
| Paged KV memory | PagedAttention / vLLM | 用分页式 KV 管理降低内存碎片，提升 batching 能力 | 基础平台；但它没有把逻辑命中粒度和物理页粒度系统性解耦 | 中 |
| vLLM APC 与 RFC | vLLM prefix caching docs、APC RFC issue | full block hash、prefix block table、曾讨论 complete / partial block table | 如果我们只做 partial block table，会高度重合 | 高 |
| Hybrid / heterogeneous KV | Jenga、vLLM Hybrid KV Cache Manager | 面向不同 attention layer 的异构 KV 分配，改善模型级内存利用 | 关注 layer/group 之间的异构；不是 prefix hit micro-page 与 physical page 的解耦 | 中 |
| 分层/远端 KV | LMCache、Strata、PCR、CALVO、MemServe | 关注 GPU/CPU/SSD/远端 KV 存取、布局转换、预取和替换 | Strata 已经涉及 CPU/GPU 布局解耦，必须避开 “hierarchical layout” 叙事 | 中高 |
| 非连续上下文复用 | CacheBlend、EPIC、RAG chunk cache | 复用 RAG chunk 或位置无关上下文 | 本方向只做 exact prefix cache 的粒度解耦，不解决非前缀拼接 | 低到中 |

## 4. 是否重复

原始表述 “支持 partial block / sub-block prefix caching” **重复风险高**，因为 vLLM 社区早已有相关 RFC 和讨论。

 refined 之后可以防守的差异是：

1. 不是把 block table 改成更小 block，而是同时保留 `logical micro-page` 与 `physical macro-page`。
2. 不是只提升命中率，而是解决 block-size 在 `hash granularity`、`allocator page size`、`kernel page size`、`offload transfer size` 之间的系统性冲突。
3. 不要求 attention kernel 一开始支持任意微块，第一阶段可通过 micro-hit + tail recompute + macro-page promotion 落地。
4. 可以与 ShapeKV 联动，让 micro-page 命中结果服务于 CUDA Graph bucket 和 cascade attention 形状选择。

因此结论是：**ElasticPageKV 不是最干净的独立新方向，但 refined thesis 仍有空间。要避免撞车，论文必须主打“多粒度解耦”，不能主打“partial block”。**

## 5. 为什么可能有效

它的收益不来自单个 token 级小修小补，而来自三个放大效应：

1. **更大物理 page 的可行性**：物理 page 可以为 kernel/transfer 选择更大粒度，而逻辑命中仍保持细粒度。
2. **尾部重算减少**：当 prompt 共享长度不刚好落在 block 边界时，micro-page 可减少重算窗口。
3. **替换与迁移更准确**：热 micro-page 可以被保留或 promote，冷 macro-page 的未命中部分不必被错误视作同等价值。

最可能有效的 workload：

- RAG/Agent 中有长系统提示词、工具描述、模板、历史摘要，后缀少量变化。
- 多租户共享模型但 prefix 长度分布不规则。
- KV connector / offload 需要较大 transfer chunk，但 APC 希望小粒度匹配。
- block size sweep 后发现大 block 提升 kernel 或 transfer 效率，却明显伤害 prefix hit tail。

## 6. vLLM 实现路径

### Phase 0：先证明问题存在

新增 profiling，不改语义：

- 记录 `exact shared tokens` 与 `full-block reusable tokens` 的差值。
- 记录每次 APC 命中后因 block 对齐导致的 recompute tokens。
- 做 block size sweep，观察 TTFT、prefill recompute、hit ratio、KV memory waste、offload transfer time。

### Phase 1：只做 logical micro-index

最小侵入实现：

- 保留现有物理 block table。
- 为 request 额外维护 micro-page hash sidecar。
- 命中时返回 `micro_hit_len`，但实际复用仍回退到 full physical block；剩余 tail 通过重算补齐。
- 用这一步验证 “细粒度 hit 信息” 是否能改善调度和统计，即使暂不改 kernel。

### Phase 2：micro-to-macro mapping

核心系统实现：

- 为每个 macro-page 保存 micro-page bitmap、offset 和 reference metadata。
- allocator 仍分配 macro-page，但 cache lookup 可以返回 page offset。
- 对完整覆盖的 macro-page 走现有 fast path，对部分覆盖的 page 决策 tail recompute 或 page promotion。

### Phase 3：可选 kernel/connector 支持

高风险增强：

- attention metadata 支持 page offset / validity mask。
- KV connector 传输 macro-page，同时传输 micro-page validity map。
- 与 ShapeKV 的 graph bucket 对齐策略联动。

## 7. 实验设计

必须做的实验：

- `block_size` sweep：8/16/32/64 token。
- prefix 长度偏移实验：共享前缀分别落在 block 边界前后 1、4、8、15 token。
- long prompt + short output：TTFT 和 p99 TTFT。
- high reuse RAG/Agent trace：hit ratio、recompute tokens、KV memory pressure。
- offload/connector 场景：macro transfer size 与 micro hit granularity 的 tradeoff。

关键 ablation：

- 只有 micro-index，无 macro-page promotion。
- 只有 macro-page promotion，无 micro-index。
- 不同 micro-page size：1/2/4/8 token。
- 是否与 ShapeKV 的 graph-aware scheduling 联动。

## 8. 发表性判断

| 维度 | 判断 |
| --- | --- |
| 轻量级 | 中。Phase 1 轻，Phase 3 不轻 |
| 普适性 | 中高。适用于 exact prefix cache，但不覆盖非前缀 RAG 拼接 |
| 高吞吐 | 条件成立。需要证明更大 physical page 带来的执行收益超过 metadata 成本 |
| 低延迟 | 条件成立。主要体现在减少 TTFT 和 tail recompute |
| vLLM 兼容性 | 中。可分阶段兼容，最终 kernel 支持风险较高 |
| OSDI 潜力 | 中高，但必须避开 partial block 叙事 |

最终建议：**不要把 ElasticPageKV 单独作为第一主线，除非 Phase 0 profiling 证明 block/page 粒度冲突是 vLLM APC 的显著瓶颈。它更适合与 ShapeKV 合并，形成“granularity-aware + shape-aware exact KV runtime”。**

## 9. 关键资料

- PagedAttention / vLLM: <https://arxiv.org/abs/2309.06180>
- vLLM prefix caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM APC RFC: <https://github.com/vllm-project/vllm/issues/2614>
- vLLM Hybrid KV Cache Manager: <https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/>
- Jenga: <https://arxiv.org/abs/2503.18292>
- Strata: <https://arxiv.org/abs/2508.18572>
