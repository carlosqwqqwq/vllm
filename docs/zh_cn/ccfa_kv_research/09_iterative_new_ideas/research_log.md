# 第五轮调研日志：为什么最后留下这三个

## 1. 本轮问题

用户要求继续寻找新的 idea，并且持续迭代搜索和思考，最后只保留三个。

与上一轮相比，这一轮的筛选标准更严格：

1. 不能和前面被淘汰的方向本质同构；
2. 不能只是在已有工作边上做一个小变体；
3. 必须还能映射回 vLLM 当前代码路径；
4. 必须满足轻量级、普适性、高吞吐、低延迟这四个约束中的大部分；
5. 必须诚实标注“重合风险”而不是宣称绝对首创。

## 2. 这轮重点检索了什么

### 2.1 调度 / residual / graph 相关

- `LAPS`
- `k-LPM`
- `DLPM`
- `Preble`
- `SGLang v0.4`

目的：

- 判断 `ShapeKV` 收窄之后还能不能继续活下来；
- 判断是否能从中抽出更干净的 `ResidualModeKV`。

### 2.2 eviction / queue / look-ahead 相关

- `PCR`
- `KVFlow`
- `HotPrefix / LPC`

目的：

- 判断 “lease / look-ahead retention” 是否还能作为新主线。

结论：

- 这条线太拥挤，应继续删除。

### 2.3 prompt structure / exact cacheability 相关

- `Prompt Cache`
- `Serve Programs, Not Prompts`
- `Don't Break the Cache`
- OpenAI Prompt Caching docs
- Anthropic Prompt Caching docs

目的：

- 判断 prompt stability / cacheability contract 是否只是经验总结；
- 判断它能不能被系统化成 `PromptABIKV`。

### 2.4 vLLM 社区输入语义相关

- issue #16016 `cache salting`
- issue #16634 `raw prompt bypassing template rendering`
- issue #20261 `multimodal visual input and prefix caching`
- issue #25096 `prompt embeds + prefix caching`

目的：

- 判断“异构输入上的精确前缀语义”是不是一个真实且尚未被统一抽象的问题。

结论：

- 是真实问题；
- 但必须写成 unified semantics，而不是几个 feature patch 的集合。

## 3. 被排除的候选

### 3.1 LeaseKV / look-ahead retention 一类

排除原因：

- 已有 `PCR`、`KVFlow`、`k-LPM`、`HotPrefix/LPC` 等强邻居；
- 很难再讲出一个足够干净的新 thesis；
- 很容易被审稿人看成 eviction heuristic 的新变体。

### 3.2 单独强化 ShapeKV 的旧表述

排除原因：

- “short re-prefill + graph clustering” 太接近 `LAPS`；
- 必须进一步改写成 `ResidualModeKV` 才能保留。

### 3.3 单独 control-plane fast path

排除原因：

- 更像配套优化；
- 主线论文的边界不够强。

### 3.4 security / isolation only

排除原因：

- `CacheSolidarity` 已经把多租户 APC side-channel 主线占住了；
- 不完全贴合当前“提升推理效率”的核心目标。

## 4. 为什么最后剩下这三个

### 4.1 `ResidualModeKV`

保留理由：

- 最贴近 vLLM 当前 runtime；
- 最容易做 profiling；
- 最容易较快得到 yes/no 证据。

### 4.2 `TypedPrefixKV`

保留理由：

- 新颖度更强；
- vLLM 社区已有多条 issue 指向同一个根问题；
- 有机会把 APC 从 feature 升级成异构输入上的统一 subsystem。

### 4.3 `PromptABIKV`

保留理由：

- 能解释很多线上 hit 不稳定的根因；
- 和 provider prompt caching 的真实使用约束高度一致；
- 很适合作为另外两条线的系统接口层。

## 5. 当前总判断

如果只看“先做哪个最划算”，答案是：

1. `ResidualModeKV`
2. `TypedPrefixKV`
3. `PromptABIKV`

如果只看“哪条线的 novelty ceiling 更高”，答案更接近：

1. `TypedPrefixKV`
2. `ResidualModeKV`
3. `PromptABIKV`

## 6. 下一轮最小任务

1. 给 `ResidualModeKV` 设计 profiling patch
2. 给 `TypedPrefixKV` 列出最小 typed carriers:
   - prompt embeds
   - multimodal hashes
   - cache salt
3. 给 `PromptABIKV` 设计 miss-breaker trace 分类

## 7. 本轮资料索引

- LAPS: <https://arxiv.org/abs/2601.11589>
- k-LPM: <https://arxiv.org/abs/2502.04677>
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
