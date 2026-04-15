# 第五轮：持续迭代后的三个新方向

## 1. 本轮目标

这一轮不是在 `08_collision_aware_new_ideas` 的基础上简单换名字，而是继续做“否证式”收缩：

- 继续排除已经被 2025-2026 新工作覆盖的方向；
- 继续剔除与 `ShapeKV / ElasticPageKV / ControlPlaneKV` 过度重合的表述；
- 只保留三个当前仍值得进入下一轮 profiling 或原型验证的候选。

## 2. 本轮保留下来的三个方向

| 方向 | 当前判断 | 推荐角色 | OSDI 潜力 | vLLM 落地性 | 最大风险 |
| --- | --- | --- | --- | --- | --- |
| `ResidualModeKV` | 当前最值得先验证 | 主线候选 | 中高 | 高 | 容易回退成 `LAPS` 风格的 short-prefill graph batching |
| `TypedPrefixKV` | 新颖度更强 | 主线候选 / 强备选 | 中高 | 中高 | 如果只写成 “支持 prompt embeds prefix caching” 会退化成 feature patch |
| `PromptABIKV` | 很有价值但更偏接口与系统协同 | 支撑 thesis / 次主线 | 中 | 高 | 容易被看成 prompt engineering 或 provider prompt-caching best practice |

最简洁的推荐是：

1. 先用 profiling 验证 `ResidualModeKV`
2. 同时论证 `TypedPrefixKV` 是否能把 APC 从“文本前缀优化”扩展成“异构输入统一精确复用”
3. 把 `PromptABIKV` 作为系统接口层补强，而不是最先单独主投

## 3. 这轮删除了什么

本轮继续删除或降级的方向包括：

- `lease/look-ahead eviction` 一类
  - 最近邻已经太多，包括 `k-LPM`、`PCR`、`KVFlow`、`HotPrefix/LPC` 一类工作。
- `short re-prefill graph clustering` 一类
  - 继续受到 `LAPS (MLSys 2025)` 压制。
- `pure control-plane speedup` 一类
  - 更像配套优化，不像最稳的主线 thesis。
- `security/isolation-only APC` 一类
  - `CacheSolidarity` 已经占住了多租户 prefix cache 安全这条主线。

## 4. 三个目录的阅读顺序

1. [ResidualModeKV](residualmodekv/README.md)
   - 先读。它解决的是“命中了 APC 之后，vLLM 是否还在用错误的执行模式做剩余工作”。
2. [TypedPrefixKV](typedprefixkv/README.md)
   - 第二读。它解决的是“什么才算同一个 exact prefix，以及 APC 如何覆盖 prompt embeds / multimodal / tool-rich 输入”。
3. [PromptABIKV](promptabikv/README.md)
   - 第三读。它解决的是“应用层怎样稳定地产生可命中的 exact prefix，而不是不断被模板漂移、工具顺序、动态字段击穿”。

横向比较见 [comparison.md](comparison.md)，调研路径与删选依据见 [research_log.md](research_log.md)。

## 5. 当前最稳妥的总判断

截至 `2026-04-15` 的这轮检索，我没有发现与这三个 refined thesis 完全等价的公开系统论文；但也**不能写成“确保没人做过”**。更稳妥的表述是：

> Existing works have separately optimized prefix-aware scheduling, offloaded KV hierarchies, short-prefill batching, or modular prompt reuse. The remaining gap is how vLLM-like runtimes should co-design exact-prefix semantics, cacheable input contracts, and post-hit execution modes.

这比直接宣称 `first` 更安全，也更符合目前证据强度。

## 6. 本轮核心资料

- vLLM Prefix Caching: <https://docs.vllm.ai/en/latest/design/prefix_caching/>
- vLLM CUDA Graphs: <https://docs.vllm.ai/en/latest/design/cuda_graphs/>
- vLLM Automatic Prefix Caching: <https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/>
- PagedAttention / vLLM: <https://arxiv.org/abs/2309.06180>
- LAPS: <https://arxiv.org/abs/2601.11589>
- k-LPM / prefix reuse scheduling: <https://arxiv.org/abs/2502.04677>
- PCR: <https://arxiv.org/abs/2603.23049>
- KVFlow: <https://openreview.net/pdf/2c47adb29432f99879fceb1371b72f6e97e1f3ac.pdf>
- Prompt Cache: <https://arxiv.org/abs/2311.04934>
- Serve Programs, Not Prompts: <https://yecl.org/publications/gim2025hotos.pdf>
- Don't Break the Cache: <https://arxiv.org/abs/2601.06007>
- OpenAI Prompt Caching: <https://developers.openai.com/api/docs/guides/prompt-caching>
- Anthropic Prompt Caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- vLLM issue #16016: <https://github.com/vllm-project/vllm/issues/16016>
- vLLM issue #16634: <https://github.com/vllm-project/vllm/issues/16634>
- vLLM issue #20261: <https://github.com/vllm-project/vllm/issues/20261>
- vLLM issue #25096: <https://github.com/vllm-project/vllm/issues/25096>
