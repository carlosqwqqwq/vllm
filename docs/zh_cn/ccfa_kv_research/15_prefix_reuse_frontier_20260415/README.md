# 前缀复用前沿调研与新 Idea 存档

更新时间：2026-04-15

本目录只做三件事：

1. 把 `2024-2026` 这轮前缀复用 / KV 复用相关论文快速收拢成一个可执行判断。
2. 给出一个比“修一个 prefix cache feature gap”更像 `CCF-A` 系统论文的问题定义。
3. 把它落回 `vLLM` 的现有代码结构，说明为什么它不是空想，而是可以在当前仓库里逐步实现的。

## 文件说明

- [literature_survey_20260415.md](literature_survey_20260415.md)
  - 前沿论文调研，覆盖 exact-prefix、position-independent、workflow-aware、security / audit 等多条近邻路线。
- [realizekv_new_idea_20260415.md](realizekv_new_idea_20260415.md)
  - 这轮最值得继续推进的新 paper-facing idea。
  - 核心建议是：把内部问题名 `FalseNegativeKV` 上升成一个更适合论文叙事的系统问题。
- [vllm_realization_map_20260415.md](vllm_realization_map_20260415.md)
  - 只读分析 `vLLM` 里现有 prefix caching / skip / route / sidecar 相关落点。
  - 给出后续文档化实现路线，但本轮不改代码。
- [osdi_master_blueprint_20260415.md](osdi_master_blueprint_20260415.md)
  - 把 `12_combined_main_thesis`、`14_beyond_current_ideas`、`15_prefix_reuse_frontier_20260415` 三套材料联动起来。
  - 目标是回答：当前主线怎样补齐为更接近 `CCF-A / OSDI` 的完整 thesis。

## 三句结论

### 1. 现在最拥挤的方向，不建议再正面撞

- “把 prefix caching 做得更灵活” 这条线，已经被 `EPIC / MPIC / KVLink / KVShare / RelayCaching` 等工作明显占住。
- “把 exact-prefix cache 管得更聪明” 这条线，已经有 `Prompt Cache / Marconi / Cake / KVFlow` 这种强系统工作。
- “把 prompt caching 做成产品或安全评测” 也已经有 `Auditing Prompt Caching`、`CacheSolidarity`、`Don't Break the Cache`。

### 2. 仍然值得打的空位，不在“有没有 cache”，而在“为什么 hit 没有兑现”

更精确地说：

> 很多请求在逻辑上已经具备 exact 或 logical-exact reuse 条件，但 runtime 因为 API 语义、观测需求、carrier、backend 或 batch 契约不匹配，没有把它兑现成 physical hit。

这和现有大部分论文都不一样。它们通常默认：

- 要么 hit 已经成立，只讨论怎么更快、更稳、更省。
- 要么 prefix 不再要求 exact，转去做 PIC、模块化拼接或 selective recompute。

而我们真正有空间的问题是：

> 当系统里明明已经“存在可复用状态”时，为什么 serving runtime 仍然把它当作 miss？

### 3. 当前最值得继续推进的新 thesis

最推荐继续推进的 paper-facing 版本不是继续停留在 `FalseNegativeKV` 这个内部名字上，而是将其收束成：

> `RealizeKV`: Realizing Counterfactual Exact Hits in LLM Serving

这里：

- `FalseNegativeKV`
  - 更适合作为内部测量语言。
- `RealizeKV`
  - 更适合作为论文叙事。
  - 它强调的是：
    - `counterfactual hit` 可测，
    - `realization gap` 可解释，
    - `bridge + route + schedule` 可系统化实现。

## 目录内建议阅读顺序

1. [literature_survey_20260415.md](literature_survey_20260415.md)
2. [realizekv_new_idea_20260415.md](realizekv_new_idea_20260415.md)
3. [vllm_realization_map_20260415.md](vllm_realization_map_20260415.md)
4. [osdi_master_blueprint_20260415.md](osdi_master_blueprint_20260415.md)

## 当前态度

这轮调研后的态度很明确：

- 不建议再发散新一批彼此平行的高层 idea。
- 应该把“latent exact hit 没被 runtime 兑现”这件事做厚。
- 如果后续要在 `vLLM` 上做实现，也应优先围绕：
  - `observation mismatch`
  - `backend mismatch`
  - `typed bridge`
  - `realization-aware scheduling`
  这四件事展开。
