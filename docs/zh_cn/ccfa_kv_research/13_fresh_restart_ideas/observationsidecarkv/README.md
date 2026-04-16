# ObservationSidecarKV：为 mixed API 复用增加极小观测 sidecar

更新日期：2026-04-15

> 复审说明：下面这份 `README` 保留的是第一版构思。
> 截至 `2026-04-15` 的严格重复性与可行性判断，请以 [`overlap_feasibility_assessment.md`](./overlap_feasibility_assessment.md) 为准。

## 1. 一句话 thesis

`ObservationSidecarKV` 的核心主张是：

> 现有 prefix cache 只缓存 KV，导致 `prompt_logprobs`、pooling、token-level scoring 等 mixed API 请求即使共享长前缀，也往往需要跳过 APC 或近乎整段重算；
> 更合理的做法不是只缓存 KV，而是缓存一个**极小的 observation sidecar**，让 prefix reuse 从 generation-only 能力扩展为 mixed-API serving substrate。

## 2. 为什么这是一个真实缺口

这条线不是概念发散，而是来自 `vLLM` 当前真实存在的接口裂缝。

### 2.1 `prompt_logprobs`

`vllm/sampling_params.py`

当前默认逻辑会在 `prompt_logprobs is not None` 时触发：

- `skip_reading_prefix_cache = True`

这说明：

- 前缀不是不能复用；
- 而是系统今天只会复用 KV，不会复用这些请求需要的观测侧结果。

### 2.2 pooling / scoring

`vllm/pooling_params.py` 与 pooling runner 路径里也存在类似保守策略。

这说明：

- 现有 APC 语义仍然偏 generation-only；
- mixed API 请求在真实共享前缀下仍无法充分受益。

### 2.3 这和旧 `ObservationKV` 的区别

旧思路更像：

- “让更多 API 兼容 APC”

现在的新表述更强一些：

- 不只讲 compatibility；
- 而是把复用对象从 `KV only` 扩展成：
  - `KV + minimal observation sidecar`

## 3. 为什么它和最近论文不完全重合

### 3.1 与 vLLM 现有兼容路径的边界

社区已经有：

- `prompt_logprobs + APC compatibility`

但那更多是保守兼容，并不是：

- 让 mixed API 真正享受 prefix reuse 的统一机制。

### 3.2 与 `Beyond Speedup` 的边界

- `Beyond Speedup`
  - 讨论 KV cache 不只可用于加速 decode，也可用于 sampling / reasoning。
- `ObservationSidecarKV`
  - 研究的是 serving runtime 中：
  - **如何在 exact prefix reuse 场景下恢复 mixed API 所需观测结果**

### 3.3 当前结论

截至 `2026-04-15`，我**没有找到一篇公开系统论文明确把“mixed-API observation sidecar reuse”作为主问题系统化写清楚**。

不能说这是零重合方向，但它比：

- exact-prefix contract
- typed prefix hash
- prefix-aware scheduling

这些更不拥挤。

## 4. 为什么它能在 vLLM 中实现

### 4.1 请求侧已经区分 mixed API

`Request` 和 `gpu_model_runner` 当前已经携带：

- `prompt_logprobs`
- pooling params
- late interaction / token classify / token embed 等路径

这说明 mixed API 不是未来需求，而是现在就有的请求类型。

### 4.2 现有系统已经保留了部分观测侧状态

例如：

- `num_prompt_logprobs`
- `in_progress_prompt_logprobs`

说明系统并非完全没有 observation-side state，只是这些状态还没有被纳入 prefix reuse 设计。

### 4.3 最小原型可以很克制

第一版不需要缓存完整 hidden states。

可以先做：

1. prefix hit 后的 selective replay
2. 极小 observation checkpoint
3. API-aware reuse plan

这就足够验证 thesis。

## 5. 可能的机制

### 5.1 sidecar object

第一版 sidecar 不应太大，建议只包含：

- 观测恢复所需的最小辅助状态
- 与请求类型相关的恢复计划
- 与 prefix token range 对齐的 metadata

### 5.2 三种执行模式

第一版可以把 mixed-API request 分成：

- `REUSE_ONLY`
  - 纯生成请求，沿用 APC
- `REUSE_PLUS_SELECTIVE_REPLAY`
  - 命中 APC，但只对少量 token 做受控 replay
- `FULL_RECOMPUTE`
  - sidecar 不够或请求语义不支持

### 5.3 最好的论文叙事

这条线最好的表述不是：

- 支持 `prompt_logprobs`

而是：

- **让 prefix reuse 从 generation-only optimization 变成 mixed-API serving substrate**

## 6. 为什么它符合你们的要求

### 6.1 轻量级

第一版主要改：

- request / sampling / pooling 相关路径
- prefill replay 与 observation extraction 路径

不需要改模型训练，也不需要先改 kernel。

### 6.2 普适性

只要服务同时承载：

- generation
- prompt scoring
- pooling
- token-level observation

这条线就成立。

### 6.3 高吞吐 / 低延迟

在长共享前缀 workload 下，mixed API 的整段重算会很贵。

如果能把 full recompute 降成：

- reuse + selective replay

那么 TTFT 和整体吞吐都可能改善。

## 7. 最关键的实验是什么

这条线最关键的实验不是只看一个 API。

最强的证据应当是：

1. generation-only baseline
2. prompt_logprobs workload
3. pooling / token scoring workload
4. mixed batch workload

然后展示：

- 当前 generation-only APC 语义造成了多少重算；
- sidecar 机制能回收多少这部分浪费；
- 是否保持 exact observation semantics。

## 8. 最大风险

最大风险非常明确：

- **很容易被看成 API feature patch。**

因此，必须坚持三个边界：

1. 不把它写成“又支持了一个参数”
2. 不把它写成 prompt_logprobs 单点优化
3. 强调 unified mixed-API reuse substrate

## 9. 当前判断

截至 `2026-04-15`：

- **创新性：中高**
- **与最新论文重合风险：低到中**
- **vLLM 落地性：高**
- **优化效果把握：中高**
- **当前三条里的推荐程度：第二**

## 10. 参考资料

- Beyond Speedup -- Utilizing KV Cache for Sampling and Reasoning: <https://arxiv.org/abs/2601.20326>
- vLLM PR #13949: <https://github.com/vllm-project/vllm/pull/13949>
