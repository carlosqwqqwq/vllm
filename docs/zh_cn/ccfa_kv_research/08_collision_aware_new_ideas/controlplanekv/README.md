# ControlPlaneKV：高命中场景下的精确 Prefix Cache 控制面

## 1. 一句话 thesis

`ControlPlaneKV` 的原始 thesis 是：**当 prefix cache 命中率足够高时，GPU prefill 成本被跳过，系统瓶颈会转移到 CPU 控制面，包括 block hash、prefix lookup、scheduler metadata、锁竞争、KV event 和请求准备；因此需要一个低开销、批量化、可验证的 exact prefix-cache control plane。**

深入调研后的判断是：**这个方向不适合作为独立 OSDI 主线，但很适合作为 ShapeKV / ElasticPageKV 的支撑组件。**

原因很直接：vLLM V1 和 SGLang 都已经把低 CPU overhead、cache-aware scheduling、prefix cache 加速作为重要目标。如果我们只说 “降低 prefix cache 控制面开销”，重合风险太高。

## 2. vLLM 当前证据

vLLM 的控制面确实存在可优化空间：

- `vllm/utils/hashing.py` 支持 `sha256`、`sha256_cbor`、`xxhash`、`xxhash_cbor` 等 hash function，说明 prefix caching hash 本身已经是可配置的性能/安全 tradeoff。
- `vllm/v1/core/kv_cache_utils.py` 按 block 计算 request block hash，且只 hash full blocks。
- `vllm/v1/request.py` 与 scheduler 通过 request metadata 传递 cache salt、block hashes、num computed tokens 等状态。
- `vllm/v1/core/sched/scheduler.py` 在调度新请求时会先查询本地 cache，再查询外部 KV connector，并把 local/external cached tokens 写入 prefill stats。

这些路径说明 APC 的命中不是一个纯 GPU 行为，而是穿过 tokenizer/request/hash/index/scheduler/connector 的控制面流程。

## 3. 最近邻工作与重合分析

| 工作线 | 代表工作 | 做了什么 | 与 ControlPlaneKV 的关系 | 重合风险 |
| --- | --- | --- | --- | --- |
| 低 CPU overhead engine | vLLM V1、SGLang v0.4 | 强调更低 CPU overhead、更快 scheduler、更好的 prefix cache 集成 | 直接重合，需要避免泛泛而谈 CPU overhead | 高 |
| Prefix cache index | vLLM APC、SGLang RadixAttention | block hash / radix tree / cache-aware scheduling | 如果只做更快 lookup，会撞车 | 高 |
| KV transfer control | LMCache、PCR、CALVO | 控制远端 KV 加载、预取、替换和跨节点路由 | 关注远端 KV，不是本地 exact prefix index；但控制面叙事相近 | 中 |
| 安全 hash / cache salt | vLLM docs、provider prompt caching security discussions | 通过 hash/salt 隔离租户，避免错误复用或侧信道 | 与我们的 “fast candidate + strong verification” 可结合 | 中 |
| 通用 scheduler | DLPM、prefill/decode disaggregation schedulers | 优化延迟、公平或 goodput | 目标不同，但会被审稿人视为调度类 baseline | 中 |

## 4. 是否重复

如果原始论文题目是：

> Faster prefix cache control plane for vLLM.

那么重复风险很高，不建议主投。

可防守的 refined version 应该收窄为：

> Exact and verifiable prefix-cache fast path for high-hit LLM serving, combining batched lookup, rolling candidate fingerprints, lazy strong-hash verification, and scheduler-visible cache readiness without sacrificing correctness or tenant isolation.

这个 refined version 的边界是：

1. 不替代 vLLM V1 的整体低 CPU overhead，而是专门研究 APC 高命中路径。
2. 不牺牲 exactness。fast hash 只做候选，强 hash / canonical hash 负责验证。
3. 不只优化 hash 函数，而是把 hash future、batch lookup、cache readiness 和 scheduler 连接起来。
4. 不独立承诺大收益，而是服务于 ShapeKV 的 shape-aware scheduling 和 ElasticPageKV 的 micro-index。

## 5. 方法设计

### 5.1 Rolling candidate fingerprint

为 token prefix 维护增量 fingerprint：

- tokenizer 产生 token 时同步更新 rolling hash。
- request 到达 scheduler 前，已有候选 prefix fingerprint。
- fast fingerprint 只用于定位候选 cache entries，不直接决定复用。

### 5.2 Lazy strong verification

为了兼顾安全与性能：

- 候选命中用 `xxhash` 或 rolling hash 快速定位。
- 真正复用前，用 `sha256_cbor` 或 canonical hash 批量验证。
- 多租户场景把 `cache_salt` 纳入 namespace。
- 验证失败走 baseline，不允许 silent incorrect reuse。

### 5.3 Batched prefix lookup

把 per-request lookup 改成 per-scheduler-step lookup：

- scheduler 每轮收集 waiting requests 的 hash-ready prefix。
- 对相同 parent hash / same namespace 的请求批量查 index。
- 返回 cache-ready 状态和候选 hit length，供 ShapeKV grouping 使用。

### 5.4 Cache-readiness pipeline

把 prefix cache lookup 从同步阻塞变成 pipeline：

- `TOKENIZED`：token 已就绪。
- `HASH_READY`：候选 fingerprint 已就绪。
- `VERIFIED_READY`：强验证完成。
- `KV_READY`：本地或外部 KV 可读。

这可以让 scheduler 提前知道哪些 request 即将可复用，避免等到调度瞬间才查询 cache。

## 6. 为什么可能有效

这个方向只有在以下条件成立时才有效：

- workload 是 high prefix-hit + short output。
- GPU prefill 已经被 APC 大量跳过。
- CPU profile 显示 hash / lookup / metadata / scheduler 占 TTFT 的比例明显上升。
- batch lookup 能减少 Python overhead 或 lock contention。
- strong verification 的额外成本可以被批量化或异步化隐藏。

如果 profiling 发现 vLLM V1 在目标 workload 下 CPU overhead 已经接近不可见，那么 `ControlPlaneKV` 不能独立成文，只能作为工程优化。

## 7. vLLM 实现路径

### Phase 0：CPU profile

先测：

- hash generation time。
- prefix lookup time。
- scheduler step time。
- connector local/external cache query time。
- Python GC 与 lock contention。

### Phase 1：batch lookup

最小改动：

- 不改 hash 算法。
- 在 scheduler step 中收集多个 request 的 block hashes。
- 对 KV cache coordinator 做批量 query wrapper。
- 输出 batched hit stats。

### Phase 2：fast candidate + strong verification

中等改动：

- 增加 fast fingerprint 字段。
- prefix index 支持 candidate bucket。
- 强验证仍用现有 hash function，保证 correctness。

### Phase 3：与 ShapeKV 联动

把 `VERIFIED_READY` 和 `shape signature` 一起交给 scheduler：

- shape-aware grouping 不再被同步 lookup 阻塞。
- cache-ready request 可优先组成 graph-stable batch。

## 8. 实验设计

核心实验：

- high-hit short-output benchmark：长系统 prompt + 少量变化 + 短回答。
- QPS sweep：低/中/高并发下 CPU profile。
- hash algo sweep：sha256、sha256_cbor、xxhash、xxhash_cbor。
- salt / multi-tenant test：验证隔离开销。
- batch size sweep：per-request lookup 与 batched lookup 对比。

必须报告：

- TTFT p50/p95/p99。
- scheduler step time。
- hash/lookup 占 CPU 时间比例。
- cache hit correctness。
- verification fallback rate。
- 对 ShapeKV 的 graph replay rate 是否有帮助。

## 9. 发表性判断

| 维度 | 判断 |
| --- | --- |
| 轻量级 | 高 |
| 普适性 | 中。只在 high-hit short-output 下强 |
| 高吞吐 | 中。取决于 CPU 是否真瓶颈 |
| 低延迟 | 中高。若 CPU control path 占比高，则 TTFT 会改善 |
| vLLM 兼容性 | 高 |
| OSDI 潜力 | 中低。单独主投风险大，作为组件更合理 |

最终建议：**ControlPlaneKV 不应作为独立主线。它应该服务于 ShapeKV，负责让 cache readiness 更早、更便宜、更可验证地暴露给 scheduler。**

## 10. 关键资料

- vLLM V1 blog: <https://vllm.ai/blog/v1-alpha-release>
- vLLM prefix caching design: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs design: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- SGLang v0.4 blog: <https://lmsys.org/blog/2024-12-04-sglang-v0-4/>
- SGLang / RadixAttention: <https://arxiv.org/abs/2312.07104>
- LMCache: <https://arxiv.org/abs/2510.09665>
- PCR: <https://arxiv.org/abs/2603.23049>
- CALVO: <https://arxiv.org/abs/2603.21257>
