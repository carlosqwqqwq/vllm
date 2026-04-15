# 第四轮：撞车感知后的三个新方向

## 1. 本轮结论

这一轮的目标不是继续给旧方案换名字，而是在前几轮已经发现大量重合之后，重新筛出三个仍然值得进入原型验证的方向。

结论先放在前面：

| 方向 | 当前判断 | 推荐角色 | OSDI 潜力 | vLLM 落地性 | 最大风险 |
| --- | --- | --- | --- | --- | --- |
| `ShapeKV` | 最值得主攻 | 主线论文方向 | 高 | 高 | 需要证明 APC 会显著扰乱 CUDA Graph / Cascade Attention 形状 |
| `ElasticPageKV` | 条件成立时很强 | 备选主线或与 ShapeKV 合并 | 中高 | 中 | 容易被误解为 partial block caching 或 vLLM 旧 RFC 的延伸 |
| `ControlPlaneKV` | 不建议单独主投 | 支撑组件 | 中低 | 高 | 与 vLLM V1 / SGLang 的低 CPU overhead 目标重合 |

最严格的推荐是：**主攻 `ShapeKV`，把 `ElasticPageKV` 作为可选增强，把 `ControlPlaneKV` 作为控制面支撑组件**。如果后续 profiling 证明 APC 高命中时 CUDA Graph replay 率、padding waste、cascade attention 使用率显著变差，`ShapeKV` 会是当前三者里最干净、最贴近 vLLM、最容易讲成系统论文的方向。

## 2. 为什么删除上一批自然想法

这些方向已经不再作为主线：

| 被删除方向 | 删除原因 |
| --- | --- |
| RAG / Agent 分段 KV 复用 | 容易撞 `SGLang RadixAttention`、`CacheBlend`、`EPIC`、chunk-level context caching |
| 多级 KV offload / 分层缓存 | 容易撞 `LMCache`、`Mooncake`、`MemServe`、`Strata`、`PCR`、`CALVO` |
| speculative decoding KV 事务缓存 | 已有 `SpeCache` 明确做 transactional KV caching under PagedAttention |
| prompt logprobs 与 prefix cache 兼容 | vLLM 明确存在限制，但 `SPECS` 已经提到为 vLLM 做过相关绕过式修改，不能作为干净主线 |
| 单纯 QoS / prefix-aware scheduling | vLLM GitHub issue 与 SGLang 已经覆盖相当一部分，需要更细的系统切口 |

## 3. 三个目录的阅读顺序

1. [ShapeKV](shapekv/README.md)：先读。它从 “KV hit 改变执行形状” 切入，是当前最推荐的主线。
2. [ElasticPageKV](elasticpagekv/README.md)：第二读。它从 “逻辑复用粒度与物理页粒度解耦” 切入，但要严防与 partial block / virtual memory work 重合。
3. [ControlPlaneKV](controlplanekv/README.md)：最后读。它更适合作为前两个方向的支撑优化，而不是独立 OSDI thesis。

横向比较见 [comparison.md](comparison.md)，调研过程与删除理由见 [research_log.md](research_log.md)。

## 4. 当前不能过度宣称的点

截至本轮检索，我没有发现与三个 refined thesis 完全等价的公开系统论文。但是，**不能写“确保没人做过”**。更稳妥的论文表述应是：

> Existing systems optimize KV allocation, KV transfer, prefix-index structures, or prefix-aware scheduling separately. Our work studies how exact prefix-cache hits should be co-designed with execution shape, cache granularity, and control-plane overhead in vLLM-like LLM serving runtimes.

这句话比 “first” 更安全，也更容易在 OSDI 审稿中防守。

## 5. 本轮使用的关键资料

- vLLM prefix caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM Hybrid KV Cache Manager: <https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/>
- PagedAttention / vLLM paper: <https://arxiv.org/abs/2309.06180>
- SGLang / RadixAttention paper: <https://arxiv.org/abs/2312.07104>
- SGLang v0.4 blog: <https://lmsys.org/blog/2024-12-04-sglang-v0-4/>
- Jenga hybrid KV cache paper: <https://arxiv.org/abs/2503.18292>
- Strata hierarchical KV caching: <https://arxiv.org/abs/2508.18572>
- PAT prefix-aware attention: <https://arxiv.org/abs/2511.22333>
- SpeCache: <https://proceedings.mlr.press/v267/jie25a.html>
- vLLM APC RFC / prefix scheduling discussions: <https://github.com/vllm-project/vllm/issues/2614>, <https://github.com/vllm-project/vllm/issues/7883>
