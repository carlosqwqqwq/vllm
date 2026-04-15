# InvariantKV：面向异构后端的批不变精确前缀缓存

## 1. 一句话 thesis

`InvariantKV` 的核心主张是：

> prefix cache 的 exact semantics 不应因为执行后端、kernel 选择、batch shape 或 batch-invariant 模式发生漂移；否则系统只能在“更快后端”和“更强缓存复用”之间二选一。

## 2. 为什么它是一个真实缺口

`vLLM` 当前源码已经明确暴露出这个问题：

- 当 `VLLM_BATCH_INVARIANT=1` 且后端为 `FLASHINFER` / `TRITON_MLA` 时，会主动关闭 prefix caching；
- 多处 MoE / attention backend / quantization 路径都额外处理 batch invariance；
- 某些模型和路径只能支持部分 prefix caching，而不能支持 `'all' prefix caching`。

这些信号说明：

- `vLLM` 已经在努力追求 batch-invariant / backend-stable 执行；
- 但 APC 语义在这些快路径上还不够“可移植”；
- 结果是系统只能保守禁用 APC，而不是稳定启用。

## 3. 最近邻工作与重复风险

截至这轮检索，我**没有找到一篇公开系统论文专门研究“batch-invariant APC / backend-invariant prefix caching”**。

它与现有主流近邻的区别比较清楚：

- 与 `LAPS`、`BatchLLM`、`SGLang` 不同
  - 它不是 prefix-aware scheduling。
- 与 `ChunkAttention`、`FlashForge` 不同
  - 它不是 prefix-aware attention kernel。
- 与 `ElasticPageKV` 这类粒度方向不同
  - 它不是 page/block 粒度优化。

它更像一个新的系统命题：

> how to preserve exact cache reuse semantics across heterogeneous fast execution paths

### 当前判断

- **重复风险：低到中**
- **实现风险：中高**
- **学术独立性：较强**

## 4. 我认为可防守的 refined 命题

> 现有推理引擎为了获得更高吞吐，会在不同 attention backend、不同 batch shape、不同 reduction 路径之间切换。但 prefix cache 的复用语义往往默认这些路径对 exact prefix 是完全可移植的，一旦不满足，系统就只能整体禁用 APC。我们提出一种 backend-invariant 的 prefix cache 机制，在保持 fast path 的同时维持 exact reuse semantics。

## 5. 可能的机制设计

第一版可以只做轻量版本：

1. `backend-stability descriptor`
   - 显式记录某条执行路径是否满足 APC 所需稳定性。
2. `portable hit certificate`
   - block hit 不只取决于 token hash，也取决于 backend-stable 条件。
3. `selective fallback`
   - 对不稳定子路径做局部回退，而不是整体关闭 APC。

更强版本再继续做：

- canonical reduction order
- deterministic routing metadata for sparse kernels
- backend-aware cacheability matrix

## 6. 为什么它符合你的要求

### 6.1 轻量级

- 第一版主要是 metadata、guard 和 selective fallback；
- 不要求分布式新系统，也不要求改模型。

### 6.2 普适性

- 适用于所有追求 batch invariance / fast backend portability 的 `vLLM` 部署；
- 不是某个 workload 专属 trick。

### 6.3 高吞吐 / 低延迟

- 一旦能把当前被保守关闭的 APC 恢复到快后端上，收益可能很直接；
- 同时也减少“换后端就失去 prefix cache”的配置折损。

## 7. 最大风险

最大的风险是：

- 它可能太像“高级工程修复”；
- 以及真正做深时会滑向 kernel-heavy 工作。

因此要让它更像论文，必须证明：

1. 这是一个普遍存在的系统矛盾，而不是单后端 bug；
2. 统一机制比逐模型逐后端打补丁更优；
3. 收益能覆盖多种 backend / model family。

## 8. 当前判断

截至 `2026-04-15`：

- **可行性：中高**
- **vLLM 落地性：高**
- **优化效果把握：中高**
- **与公开论文撞车风险：低到中**
- **OSDI 潜力：中高**

如果只从“新颖性 + 贴近 vLLM 真实缺口”看，`InvariantKV` 是这轮里很值得保留的一条线。
